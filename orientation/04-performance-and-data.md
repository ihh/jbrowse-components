# 04 — Performance and Data

**What this covers:** how JBrowse 2 actually moves bytes — adapter patterns
(BAM, CRAM, VCF/VCF-tabix, BigWig, etc.) on top of `generic-filehandle2`
byte-range fetches; how the block-based renderer schedules work and caches it;
indexed vs un-indexed code paths and when feature-density gating kicks in; what
gets large at high zoom-out; embedded-mode tradeoffs (main-thread RPC, smaller
core plugin set).

---

## File I/O foundation

All file access goes through `openLocation` in
`packages/core/src/util/io/index.ts:46`. It dispatches on the `FileLocation`
discriminator and returns a `GenericFilehandle` from
[`generic-filehandle2`](https://www.npmjs.com/package/generic-filehandle2):

| Location kind | Returned filehandle |
| --- | --- |
| `UriLocation` (`{uri, locationType:'UriLocation'}`) | `RemoteFileWithRangeCache` (or an InternetAccount-decorated fetcher) |
| `LocalPathLocation` (`{localPath}`) — Node/Electron only | `LocalFile` |
| `BlobLocation` (`{blobId}`) — drag-dropped File preserved in-memory | `BlobFile` |
| `FileHandleLocation` (`{handleId}`) — FS Access API persisted handle | `BlobFile` over the cached `File` |

`GenericFilehandle.read(buffer, offset, length, position)` is the byte-range
primitive every parser uses. No code in this repo downloads whole BAMs/VCFs.

### `RemoteFileWithRangeCache`

`packages/core/src/util/io/RemoteFileWithRangeCache.ts:1-6` configures it with:

- `MAX_CACHE_ENTRIES = 2000` — LRU of fetched chunks (FIFO eviction).
- `CHUNK_SIZE = 256 * 1024` — request alignment.
- `MAX_CONCURRENT = 20` — global concurrency cap; further requests queue.

Each cache entry holds a `Uint8Array` of one 256 KB chunk. The wrapper keys by
url + chunk index so multiple adapters reading the same BAM/BAI/CRAI share
bytes. The cache is per-thread (one instance on main, one on each worker).

`clearCache()` is exported and called on factory reset.

### `RpcMethodType.augmentLocationObjects`

Before any `UriLocation` is sent to a worker, it is walked
(`packages/core/src/pluggableElementTypes/RpcMethodType.ts:172-212`) and
attached `internetAccountPreAuthorization` carrying the live OAuth token / HTTP
basic credentials so the worker's filehandle can sign requests without round-
tripping back to the main thread. `FileHandleLocation` arguments are converted
to `BlobLocation` against a shared blob map; this is how drag-dropped files
reach the worker.

---

## Format-specific adapter patterns

All adapters extend `BaseFeatureDataAdapter` (`packages/core/src/data_adapters/BaseAdapter/BaseFeatureDataAdapter.ts:20`).
Per-format wrappers around `@gmod/*` parsers:

```
plugins/alignments/src/BamAdapter/BamAdapter.ts          → @gmod/bam (BamFile)
plugins/alignments/src/CramAdapter/CramAdapter.ts        → @gmod/cram
plugins/alignments/src/HtsgetBamAdapter/                 → @gmod/bam over htsget
plugins/alignments/src/SNPCoverageAdapter/               → wraps BAM, derives SNP coverage
plugins/variants/src/VcfTabixAdapter/                    → @gmod/tabix + @gmod/vcf
plugins/variants/src/VcfAdapter/                         → unindexed (loads full)
plugins/variants/src/SplitVcfTabixAdapter/               → multiple tabix files
plugins/variants/src/PlinkLDAdapter/                     → custom LD format
plugins/wiggle/src/BigWigAdapter/                        → @gmod/bbi (BigWig)
plugins/wiggle/src/MultiWiggleAdapter/                   → fan-out across many BigWigs
plugins/bed/src/BigBedAdapter/                           → @gmod/bbi (BigBed)
plugins/bed/src/BedTabixAdapter/ + BedAdapter/           → bed text, tabix-indexed or in-memory
plugins/gff3/src/Gff3TabixAdapter/ + Gff3Adapter/        → @gmod/tabix + custom parsing
plugins/gtf/src/GtfAdapter/                              → text GTF
plugins/hic/src/HicAdapter/                              → @gmod/hic
plugins/sequence/src/IndexedFastaAdapter/                → @gmod/indexedfasta
plugins/sequence/src/BgzipFastaAdapter/                  → bgzipped indexed FASTA
plugins/sequence/src/TwoBitAdapter/                      → @gmod/twobit
plugins/sequence/src/ChromSizesAdapter/                  → tsv chrom sizes
plugins/comparative-adapters/src/PAFAdapter/             → text PAF (synteny)
plugins/arc/src/ArcAdapter/                              → text arc file
```

### Indexed path (BAM example)

`plugins/alignments/src/BamAdapter/BamAdapter.ts:29-49` wires:

```ts
new BamFile({
  bamFilehandle: openLocation(bamLocation, this.pluginManager),
  baiFilehandle: !csi ? openLocation(idxLoc, …) : undefined,
  csiFilehandle:  csi ? openLocation(idxLoc, …) : undefined,
  recordClass: BamSlightlyLazyFeature,
})
```

`@gmod/bam` reads the BAI/CSI index (resolved through the same `openLocation`
chain) once at `setup()` time, then for each `getFeatures(region)` call it
fetches only the BGZF blocks covering that region. `BamSlightlyLazyFeature`
(`plugins/alignments/src/BamAdapter/BamSlightlyLazyFeature.ts`) defers CIGAR /
MD parsing until the renderer asks for it.

Setup is cached via `setupP ??=` (`BamAdapter.ts:77`). On failure, both
`setupP` and `configureResult` are cleared so a retry rebuilds them.

### Unindexed path (VCF, GFF3, GTF, BED)

`plugins/variants/src/VcfAdapter`, `plugins/gff3/src/Gff3Adapter`, etc., load
the entire file into memory on first `getFeatures` call. They use
`@jbrowse/core/util/parseLineByLine.ts` to stream the decompressed text and
build an in-memory `IntervalTree` (`util/IntervalTree.ts`). Subsequent
`getFeatures` calls do an interval-overlap query against the in-memory tree.
This is fine for ≤ tens of MB; the adapter will OOM on whole-genome VCFs.

### BigWig (quantitative)

`plugins/wiggle/src/BigWigAdapter/` wraps `@gmod/bbi`. For a region it picks
the appropriate zoom level (BigWig stores precomputed summaries at multiple
resolutions): coarse summaries at high zoom-out, full per-bp data at high zoom-
in. The adapter exposes `getMultiRegionFeatureDensityStats` so the wiggle
display can pick render scales (min/max) before drawing.

### Tabix (compressed text + .tbi)

Tabix-backed adapters (`VcfTabixAdapter`, `BedTabixAdapter`,
`Gff3TabixAdapter`) all instantiate `new TabixIndexedFile({filehandle, tbiFilehandle, csiFilehandle})`
from `@gmod/tabix`. Tabix performs the BGZF / index dance to byte-range-fetch
only the bins overlapping the queried region. Each `getFeatures` is therefore
proportional to the *output* region, not the file size.

---

## Block-based rendering

### Block computation

`packages/core/src/util/calculateStaticBlocks.ts:23` builds a `BlockSet` for a
1D view (LGV, dotplot axis, comparative view):

- Slices the visible bp range into fixed-width blocks (default `width = 800`
  px in the call site).
- Emits `ContentBlock`s for visible regions, `InterRegionPaddingBlock`s
  between displayed regions, and `ElidedBlock`s for compressed
  hidden-region runs.
- Adds `extra` blocks on each side as a render-ahead buffer (typical: 1).
- Adjacent `ElidedBlock`s coalesce in `BlockSet.push`
  (`util/blockTypes.ts:11-21`) so a huge stretch of skipped regions becomes a
  single block.

`calculateDynamicBlocks.ts` is the variant used when block boundaries should
follow region boundaries rather than fixed pixel grid (used in some embedded
modes).

### BlockState

Each block of each display has a `BlockState` MST instance keyed by `blockKey`
(`plugins/linear-genome-view/src/BaseLinearDisplay/models/serverSideRenderedBlock.ts:101`).
Its lifecycle:

```
BaseLinearDisplay autorun:
  current set of static blocks → for each block.key not in display.blockState:
     addBlock(key, block)   → creates BlockState
  for each existing key not in new set:
     deleteBlock(key)       → BlockState afterDetach disposes the reaction
```

(See `BaseLinearDisplay/model.ts:678-686`.)

Inside `BlockState.afterAttach`
(`serverSideRenderedBlock.ts:265-285`), a `makeAbortableReaction`
(`packages/core/src/util/makeAbortableReaction.ts`) starts:

- Tracks `renderBlockData(self)` — gathers `renderProps`, `adapterConfig`,
  `regions`, `blockKey`, `sessionId`, etc.
- Debounced by `display.renderDelay` (LGV default 50 ms, see
  `BaseLinearDisplay/model.ts:142`).
- On data change, kicks `renderBlockEffect` which calls
  `rendererType.renderInClient(rpcManager, args)` →
  `rpcManager.call(sessionId, 'CoreRender', args)`.
- On completion sets `block.data` (the imageData/reactElement) and `block.renderingComponent`.
- On abort or unmount disposes the reaction and frees the bitmap.

`reloadFlag` on a BlockState is bumped to force a re-render without changing
the key (`renderArgs.reloadFlag`, line 363) — used after `displayedRegions`
change or after a config edit.

### Block cache / adapter cache

There are two caches:

- **Block cache:** the `blockState` map on each display. It is per-display,
  per-session; reseating the same `blockKey` reuses any cached layout/bitmap.
- **Adapter cache:** `packages/core/src/data_adapters/dataAdapterCache.ts:62`
  maps adapter config hashes → `Promise<AdapterCacheEntry>` and a set of
  active `sessionIds`. Two tracks pointing at the same BAM share a single
  adapter; an adapter survives as long as any session references it.
  `freeAdapterResources({sessionId})` (line 96) removes the session from each
  entry; entries with zero sessions are dropped.

The hash key is computed by `adapterConfigCacheKey` (`data_adapters/util.ts`),
which uses a stable JSON.stringify of the snapshot, so adding/removing
optional defaults does not invalidate.

### Layout & hit testing

For "box"-style renderers (gene tracks, alignments), the renderer returns a
serialized layout map (feature id → bounding box). On the main thread the
display rebuilds the layout for click hit-testing:
`BaseLinearDisplay.getFeatureByID(blockKey, id)` (model.ts:281) and
`getFeatureByCoord(blockKey, x, y)` (line 274) consult
`block.layout.getByCoord(x,y)` /`getByID`. The layout object on the main thread
is independent from the worker's drawing layout — the worker draws once, the
main thread maintains an `RBush`-style index for clicks.

---

## Feature density gating

Tracks must guard against "too many features per pixel". The
`FeatureDensityMixin` (`plugins/linear-genome-view/src/BaseLinearDisplay/models/FeatureDensityMixin.tsx:23`)
pre-flights every render:

```
featureDensityStatsP   = rpcManager.call('CoreGetFeatureDensityStats', …)
featureDensityStats    = { featureDensity, bytes, fetchSizeLimit, … }
featureDensity * view.bpPerPx → estimated features per pixel
```

If `featureDensity * bpPerPx` exceeds the configured limit (e.g. BAM tracks
default to a few thousand features per request), the display shows a
`TooLargeMessage` (`BaseLinearDisplay/components/`) and refuses to render
until the user zooms in or manually overrides with `setUserBpPerPxLimit`.

This is also where `BamAdapter.configSchema.fetchSizeLimit`
(`plugins/alignments/src/BamAdapter/configSchema.ts:51`) comes into play —
adapter pre-computes the byte size of the BGZF chunks for the query region and
short-circuits if it exceeds the limit.

Quantitative tracks have their own gate: `WiggleGetGlobalQuantitativeStats`
and `WiggleGetMultiRegionQuantitativeStats` RPC methods
(`plugins/wiggle/src/WiggleRPC/`) return min/max/sum/mean used to pick the
y-axis and decide whether per-bp data or a coarser BigWig zoom is appropriate.

---

## Memory at high zoom-out

Hot spots and what they actually cost:

1. **Static blocks across the whole genome.** For a multi-chromosome
   `displayedRegions`, the number of blocks scales with
   `total_bp / (bpPerPx * 800)`. At 1 bp/px and the whole human genome that is
   ~3.7M blocks. `ElidedBlock` coalescing keeps that bounded — only the
   visible window is `ContentBlock`. The view's `bpPerPx` setter clamps to
   `view.maxBpPerPx` based on the smallest reasonable block size.
2. **Adapter feature streams.** `BaseFeatureDataAdapter.getFeatures` returns
   an Observable. The renderer collects features for *the block region only*
   (typically 800 px ≈ ~1 Mbp at 1 bp/px). At more zoomed-out levels the
   `FeatureDensityMixin` short-circuits before fetch.
3. **ImageBitmaps in blockState.** Each rendered block holds an `ImageBitmap`
   (size = `block.width * height * 4` bytes). With ~20 visible blocks per
   track and a 100-px-tall track that is ~6 MB per track. Closing a display
   disposes the bitmaps via the reaction disposer.
4. **`displayedRegions`.** Stored as `types.frozen<Region[]>` so the whole
   list is a single JS array; "whole genome" view at 100k contigs ≈ MB of
   metadata. Generally fine.
5. **`RemoteFileWithRangeCache`.** Hard cap of 2000 × 256 KB ≈ 500 MB upper
   bound per thread (FIFO eviction). In practice usage is much lower because
   each adapter only touches a few index chunks.
6. **TextSearch.** Trix indices are loaded fully into memory by
   `TrixTextSearchAdapter` (`plugins/trix/src/`) when first searched.
7. **`BaseLinearDisplay.featureDensityStatsP`.** Cached on the display until
   the displayed regions change; cheap.

Avoid: storing serialized features on the main thread for the whole genome.
The renderer returns image bitmaps + a feature layout, not raw features.
Hover-detail RPCs (`CoreGetFeatureDetails`) fetch the single feature again on
demand.

---

## Embedded-mode tradeoffs

The embedded React products (`products/jbrowse-react-linear-genome-view`,
`-circular-genome-view`, `-app`) differ from `jbrowse-web` in:

- **Plugin set.** Each has a smaller `corePlugins.ts` — e.g. the embedded LGV
  ships LGV + the format plugins, but omits CircularView, SVInspector, etc.
  Bundle size in embedded mode is consequently a few hundred KB rather than
  several MB.
- **Default RPC driver.** Configurable. The embedded products accept a
  `makeWorkerInstance: () => Worker` option
  (`products/jbrowse-react-linear-genome-view/src/createViewState.ts:38`); if
  omitted, the product falls back to `MainThreadRpcDriver`. Embedding pages
  that don't want worker bundling (Vite users, plain `<script>` UMD users)
  can run all RPC on the main thread at the cost of UI jank during heavy
  rendering.
- **Session model.** Embedded LGV's session is small —
  `packages/embedded-core/src/` provides only what the view needs (no
  drawer widgets, no dialog queue). `createViewState` returns a single LGV
  state model rooted in a stripped session.
- **No persistent storage.** Embedded mode does not write to IndexedDB; the
  consumer wires `onChange` and persists snapshots themselves
  (`createViewState.ts:36`).
- **No menus.** Without `RootAppMenuMixin`, plugin `configure()` calls that
  branch on `isAbstractMenuManager(rootModel)` silently skip themselves.

When debugging embedded performance:

- Bundle audit: webpack bundles in `products/jbrowse-react-*/dist`. The
  `webpack.config.*` files are the source of truth for chunk splits.
- If using `MainThreadRpcDriver`, all rendering happens during the React
  render cycle — large block sets visibly stutter. Switch to a worker via
  `makeWorkerInstance` for production embedding.
- `react-i18next` is not used; there's no i18n bundle cost.
- `mobx-react`'s `enableStaticRendering(true)` is invoked only inside the
  worker entry; on the embedded main thread observers reactively update as
  usual.

### Building the worker in embedded mode

`makeWorkerInstance.ts` in each embedded product returns a Worker via
`new Worker(new URL('./rpcWorker', import.meta.url))`. With Webpack 5 / Vite 4+
this produces a separate worker chunk. Vite users typically need
`?worker` query suffix instead; see `products/jbrowse-react-linear-genome-view/src/makeWorkerInstance.ts`
for the canonical wiring and `component_tests/lgv-vite/` for a working Vite
example.
