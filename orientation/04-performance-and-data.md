# 04 — Performance and Data

**What this covers:** how JBrowse 2 actually moves bytes — adapter patterns
(BAM, CRAM, htsget-BAM, VCF tabix vs in-memory, BigWig, etc.) on top of
`generic-filehandle2` byte-range fetches; how the block-based renderer schedules
work, caches it, and indexes feature rectangles for hit testing; indexed vs
in-memory loading paths and the per-adapter `fetchSizeLimit` gates that ride
on top; what gets large at high zoom-out; embedded-mode tradeoffs
(main-thread RPC, smaller core plugin set).

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
primitive every indexed parser uses. No indexed code path in this repo
downloads whole BAMs/CRAMs/VCFs.

### `RemoteFileWithRangeCache`

`packages/core/src/util/io/RemoteFileWithRangeCache.ts:1-6`:

- `MAX_CACHE_ENTRIES = 2000` — LRU of fetched chunks (FIFO eviction).
- `CHUNK_SIZE = 256 * 1024` — request alignment.
- `MAX_CONCURRENT = 20` — global concurrency cap; further requests queue.

Each cache entry is one 256 KB `Uint8Array`, keyed by `url+chunkIndex` so
multiple adapters reading the same BAM/BAI/CRAM/CRAI share bytes. The cache is
per-thread (one on main, one on each worker). `clearCache()` is exported for
factory-reset paths.

### `RpcMethodType.augmentLocationObjects`

Before any `UriLocation` is sent to a worker, it is walked
(`packages/core/src/pluggableElementTypes/RpcMethodType.ts:172-212`) and
attached `internetAccountPreAuthorization` carrying the live OAuth token / HTTP
basic credentials so the worker's filehandle can sign requests without a
round-trip. `FileHandleLocation` arguments are converted in-place to
`BlobLocation` against a shared blob map (RpcMethodType.ts:21-71); this is how
drag-dropped files reach the worker.

---

## Adapter taxonomy

All feature adapters extend `BaseFeatureDataAdapter`
(`packages/core/src/data_adapters/BaseAdapter/BaseFeatureDataAdapter.ts:20`).
Indexed adapters are NOT all BAM-shaped: each format has its own I/O graph and
its own definition of "byte cost for a region".

```
plugins/alignments/src/BamAdapter/BamAdapter.ts          @gmod/bam (BamFile + BAI/CSI)
plugins/alignments/src/CramAdapter/CramAdapter.ts        @gmod/cram (IndexedCramFile + CraiIndex + seqFetch)
plugins/alignments/src/HtsgetBamAdapter/HtsgetBamAdapter.ts  @gmod/bam (HtsgetFile)
plugins/alignments/src/SNPCoverageAdapter/               wraps a BAM/CRAM via sub-adapter
plugins/variants/src/VcfTabixAdapter/                    @gmod/tabix + @gmod/vcf
plugins/variants/src/VcfAdapter/                         unindexed text VCF, in-memory IntervalTree
plugins/variants/src/PlinkLDAdapter/                     custom binary LD format
plugins/wiggle/src/BigWigAdapter/                        @gmod/bbi (BigWig zoom levels)
plugins/wiggle/src/MultiWiggleAdapter/                   fan-out across many BigWigs via sub-adapter
plugins/bed/src/BigBedAdapter/                           @gmod/bbi (BigBed)
plugins/bed/src/BedTabixAdapter/ + BedAdapter/           bed text, tabix-indexed or in-memory
plugins/gff3/src/Gff3TabixAdapter/ + Gff3Adapter/        tabix-indexed or in-memory
plugins/gtf/src/GtfAdapter/                              text GTF, in-memory
plugins/hic/src/HicAdapter/                              @gmod/hic
plugins/sequence/src/{IndexedFasta,BgzipFasta,TwoBit}Adapter/   sequence formats
plugins/comparative-adapters/src/PAFAdapter/             synteny pairwise alignments
```

---

## BAM (block-level random access)

`plugins/alignments/src/BamAdapter/BamAdapter.ts:29-49` wires two file handles
into `@gmod/bam`:

```ts
new BamFile({
  bamFilehandle: openLocation(bamLocation, this.pluginManager),
  baiFilehandle: !csi ? openLocation(idxLoc, ...) : undefined,
  csiFilehandle:  csi ? openLocation(idxLoc, ...) : undefined,
  recordClass: BamSlightlyLazyFeature,
})
```

The index (BAI or CSI) is read once at `setup()` time; thereafter
`bam.getRecordsForRange(refName, start, end)` translates the query into BGZF
virtual offsets and pulls *only* the BGZF blocks that contain matching reads.
`BamSlightlyLazyFeature` (`BamAdapter/BamSlightlyLazyFeature.ts`) defers CIGAR
/ MD parsing until the renderer reads those attributes.

Setup memoization (`BamAdapter.ts:77-93`):
```ts
this.setupP ??= updateStatus('Downloading index', statusCallback, async () => {
  ...
}).catch(e => { this.setupP = undefined; this.configureResult = undefined; throw e })
```

`getMultiRegionFeatureDensityStats` (BamAdapter.ts:167-182) pre-flights every
query by asking the index for the BGZF byte estimate:

```ts
if (bam.index) {
  const bytes = await bam.estimatedBytesForRegions(regions)   // BGZF block bytes
  const fetchSizeLimit = this.getConf('fetchSizeLimit')        // default 5_000_000
  return { bytes, fetchSizeLimit }
}
```

This is what powers the "too large to render" gate in the LGV alignments
display — the index, not the adapter, knows how many BGZF chunks would be
touched and thus how many bytes will fly. `BamAdapter` reference-sequence
fetching is optional and lazy via an attached `BaseSequenceAdapter` sub-adapter
(BamAdapter.ts:51-68); it is consulted only for records missing the `MD` tag,
and one batched `getSequence` call covers the whole region (BamAdapter.ts:128-141).

---

## CRAM (slice-level random access, reference-required)

CRAM differs from BAM in three structural ways. Every adapter detail below is
from `plugins/alignments/src/CramAdapter/CramAdapter.ts`:

### 1. Three filehandles, not two

The `.crai` index does not carry sequence — CRAM slices are reference-
compressed and must dereference the assembly's FASTA at decode time. The
configuration only has two slots (`cramLocation`, `craiLocation` in
`CramAdapter/configSchema.ts:24,38`), but at runtime CRAM needs a third I/O
channel: the reference. JBrowse passes it via a `seqFetch` callback wired to
a sub-adapter at construct time
(`CramAdapter.ts:48-73`):

```ts
this.configureResult = {
  cram: new IndexedCramFile({
    cramFilehandle: openLocation(cramLocation, this.pluginManager),
    index: new CraiIndex({
      filehandle: openLocation(craiLocation, this.pluginManager),
    }),
    seqFetch: async (seqId, start, end) => {
      const sequenceAdapter = await this.getSequenceAdapter()
      if (!sequenceAdapter) throw new Error('no sequenceAdapter available')
      const refName = this.refIdToOriginalName(seqId) || this.refIdToName(seqId)
      return (await sequenceAdapter.getSequence({ refName, start: start-1, end })) ?? ''
    },
    checkSequenceMD5: false,
  }),
}
```

The sub-adapter (`CramAdapter.ts:79-91`) is obtained through
`this.getSubAdapter(this.sequenceAdapterConfig)`. `sequenceAdapterConfig` is
set by the track-level wiring (`BaseAdapter.setSequenceAdapterConfig`,
`BaseAdapter.ts:42-46`) from the assembly's sequence config. Note
`checkSequenceMD5: false` (line 72) — `@gmod/cram` *can* verify each slice's
referenced sequence by MD5 against the FASTA, but JBrowse disables that check.
Mismatched assemblies will therefore decode silently to garbled reads instead
of erroring; choosing the right assembly is the user's job.

### 2. Slice-level pre-flight, not BGZF-level

The BAM pre-flight uses BGZF chunks (the parallel BAI structure). CRAM stores
data in *containers* of *slices*, and CRAI rows give per-slice byte offsets.
`CramAdapter.bytesForRegions` (CramAdapter.ts:229-242):

```ts
const blockResults = await Promise.all(
  regions.map(region => {
    const chrId = this.refNameToId(region.refName)
    return chrId !== undefined
      ? cram.index.getEntriesForRange(chrId, region.start, region.end)
      : Promise.resolve([{ sliceBytes: 0 }])
  }),
)
return sum(blockResults.flat().map(a => a.sliceBytes))
```

`getMultiRegionFeatureDensityStats` (CramAdapter.ts:216-223) returns the same
`{ bytes, fetchSizeLimit }` shape as the BAM adapter — the public contract for
the FeatureDensityMixin is identical — but the unit is `sliceBytes` (a sum of
CRAM slice sizes), and the default cap is lower:

| Adapter | Default `fetchSizeLimit` | Source |
| --- | --- | --- |
| BamAdapter | 5_000_000 bytes | `plugins/alignments/src/BamAdapter/configSchema.ts:51` |
| CramAdapter | 3_000_000 bytes | `plugins/alignments/src/CramAdapter/configSchema.ts:18` |

The asymmetry reflects decode cost: CRAM slice bytes expand more during
decompression and reference-substitution than BGZF block bytes do.

### 3. Same `Observable<Feature>` contract; slightly different feature shape

`CramAdapter.getFeatures` (CramAdapter.ts:131-211) returns
`ObservableCreate<Feature>` exactly like every other feature adapter; the
display code is agnostic. Inside the observable:

- `cram.getRecordsForRange(refId, start, end)` — fetches and decodes the
  matching slices, returning an array of records (not a stream).
- Each record is wrapped in `CramSlightlyLazyFeature` (parallel to
  `BamSlightlyLazyFeature`, deferring expensive accessors).
- Reads with `record.readLength > 5_000` are additionally interned in
  `ultraLongFeatureCache: QuickLRU<number, Feature>({ maxSize: 500 })`
  (CramAdapter.ts:36-38, 194-202) — repeated long-read renders across blocks
  reuse the same Feature instance and avoid re-decoding.
- Filter predicates (`flagInclude`, `flagExclude`, `tagFilter`, `readName`)
  are applied client-side per record (CramAdapter.ts:175-192) using shared
  helpers from `plugins/alignments/src/shared/util.ts`.
- On error inside `getRecordsForRange`, `setupP` and `configureResult` are
  cleared so a reload rebuilds the index state (CramAdapter.ts:161-164).
  This is the same self-healing pattern as BamAdapter.

So: same Observable contract, *different* I/O graph (three handles vs two,
slice-level instead of block-level random access, mandatory reference sub-
adapter), *similar* density-stats contract with format-specific units.

---

## htsget-BAM (server-side range proxy)

`plugins/alignments/src/HtsgetBamAdapter/HtsgetBamAdapter.ts:1-22` is a
subclass of `BamAdapter` that replaces only the `configure()` hook:

```ts
new HtsgetFile({                // from @gmod/bam
  baseUrl: this.getConf('htsgetBase'),
  trackId: this.getConf('htsgetTrackId'),
  recordClass: BamSlightlyLazyFeature,
}) as unknown as BamFile<BamSlightlyLazyFeature>
```

The configSchema (`HtsgetBamAdapter/configSchema.ts`) has only `htsgetBase`
and `htsgetTrackId` slots. No local index handle exists — the htsget endpoint
performs the BAI lookup server-side. Concretely:

- No `bamFilehandle` / `baiFilehandle`; the `HtsgetFile` does not call
  `openLocation`. Outbound requests therefore do not flow through
  `RemoteFileWithRangeCache` and are not byte-range chunked by JBrowse.
- No client-side index. `BamAdapter.getMultiRegionFeatureDensityStats`
  short-circuits on `if (bam.index)` (BamAdapter.ts:173) and falls back to
  `super.getMultiRegionFeatureDensityStats` (line 181) — i.e. the generic
  `BaseFeatureDataAdapter` density heuristic
  (`BaseFeatureDataAdapter.ts:181-189`) which samples `getFeatures` from the
  first region. The htsget path therefore has no precise byte pre-flight.
- Authentication: htsget tickets are returned by the htsget server as a list
  of URLs (often pre-signed S3 / GCS) plus headers. `@gmod/bam`'s
  `HtsgetFile` handles ticket retrieval and the per-chunk fetches itself; the
  configured headers are negotiated by the htsget protocol response, not by
  a JBrowse `InternetAccount`. If a deployment needs auth *to* the htsget
  endpoint, that's a separate concern — wrap the URL with an
  `InternetAccount` only at the htsget GET level, which `HtsgetFile` does
  not currently do.
- Chunk semantics: htsget chunks correspond to the server-chosen byte ranges
  returned in the ticket, not to BGZF blocks aligned to JBrowse's 256 KB
  request grid. Concurrency and caching are entirely server-driven.

In practice: HtsgetBamAdapter inherits the BAM Observable contract and all of
BamAdapter's record-processing (including the MD-tag reference-fetch path),
but loses the local index pre-flight and the local byte-range cache.

---

## VCF (tabix-indexed vs unindexed)

`plugins/variants/src/VcfTabixAdapter/` wraps `@gmod/tabix`'s
`TabixIndexedFile` (BGZF data + .tbi or .csi index). Each `getFeatures` fetches
only the bins overlapping the region; cost is proportional to the *output*
region, not file size. The `fetchSizeLimit` gate is the generic interval-based
heuristic (BaseFeatureDataAdapter density stats).

`plugins/variants/src/VcfAdapter/VcfAdapter.ts:1-90` is the unindexed path. It
loads the entire file once into memory:

```ts
const buffer = await fetchAndMaybeUnzip(loc, opts)                  // line 43
const { header, featureMap } = parseVcfBuffer(buffer, statusCallback)
const parser = new VcfParser({ header })
// featureMap : Record<refName, lines: string[]>
```

For each refName the adapter lazily builds a `Record<string, IntervalTree<Feature>>`
(`VcfAdapter.ts:51-68`) using the vendored interval tree at
`packages/core/src/util/IntervalTree.ts:1-9` — a red-black-tree-backed
implementation copied and trimmed from `@flatten-js/interval-tree`. Per-refName
trees are materialized on first query and cached on `this.calculatedIntervalTreeMap`,
so the first feature query for a chromosome pays the parse cost; subsequent
queries on the same chromosome are tree lookups.

This path is fine for small datasets. The actual ceiling is set by the
following observable factors, not by a configured limit:

- The whole file is held as a single `ArrayBuffer` after `fetchAndMaybeUnzip`,
  so available browser memory bounds it. There is no per-line byte cap in
  the adapter.
- Every line is parsed into a `VcfFeature` (line 54) on demand and the entire
  refName's features end up in the tree once any feature on that refName is
  queried.
- There is no `getMultiRegionFeatureDensityStats` override — the generic
  base-class heuristic applies, which samples real `getFeatures` and so will
  finish only after the parse completes.

In short: the unindexed VCF adapter holds the *whole decompressed file* +
*one parsed Feature per record on touched refNames* in memory simultaneously.
There is no specific "tens of MB" threshold in the code; the failure mode is
"setup takes ages then memory pressure". For anything beyond hand-curated
demo VCFs, use `VcfTabixAdapter` or `SplitVcfTabixAdapter`.

---

## BigWig (multi-resolution numeric)

`plugins/wiggle/src/BigWigAdapter/` wraps `@gmod/bbi`. BigWig embeds
precomputed summary tracks at multiple zoom levels in the same file. For each
region the adapter picks the lowest-resolution zoom level whose bin width is
smaller than the visible `bpPerPx`, fetching summaries instead of raw values
at zoomed-out scales. It exposes `getMultiRegionFeatureDensityStats` and
`getMultiRegionQuantitativeStats` so the wiggle display can compute min/max
for the y-axis before drawing (see also the
`WiggleGetGlobalQuantitativeStats` / `WiggleGetMultiRegionQuantitativeStats`
RPC methods in `plugins/wiggle/src/WiggleRPC/`).

---

## Block-based rendering

### Block computation

`packages/core/src/util/calculateStaticBlocks.ts:23` builds a `BlockSet` for a
1D view:

- Slices the visible bp range into fixed-width blocks (typical width 800 px).
- Emits `ContentBlock` for visible regions, `InterRegionPaddingBlock` between
  displayed regions, and `ElidedBlock` for compressed hidden-region runs.
- Adds buffer blocks on each side as render-ahead.
- Adjacent `ElidedBlock`s coalesce in `BlockSet.push`
  (`util/blockTypes.ts:11-21`).

`calculateDynamicBlocks.ts` is the alternative used when block edges should
align to region boundaries rather than the pixel grid.

### BlockState lifecycle

Each visible block of each display has a `BlockState` MST instance keyed by
`blockKey` (`plugins/linear-genome-view/src/BaseLinearDisplay/models/serverSideRenderedBlock.ts:101`).
A `BaseLinearDisplay` autorun adds/removes block keys
(`BaseLinearDisplay/model.ts:678-686`). On attach, `BlockState` starts a
`makeAbortableReaction` (`serverSideRenderedBlock.ts:265-285`) that:

1. Reads `renderBlockData(self)` — collects `renderProps`, `adapterConfig`,
   `regions` (with `seqAdapterRefName`), `sessionId`, etc.
2. Debounces by `display.renderDelay` (LGV default 50 ms,
   `BaseLinearDisplay/model.ts:142`).
3. On change, calls `renderBlockEffect` → `rendererType.renderInClient(rpcManager, args)`
   → `rpcManager.call(sessionId, 'CoreRender', args)`.
4. Stores the result on `block.data`; `reloadFlag` bumps re-trigger renders
   without changing the key.

On detach (display closed, block scrolled out), the reaction disposes,
canceling any in-flight RPC via `StopToken`.

### Two caches

- **Block cache:** the per-display `blockState` map. Reusing the same key
  reuses the cached layout/bitmap.
- **Adapter cache:** `packages/core/src/data_adapters/dataAdapterCache.ts:62`
  maps adapter config hashes → `Promise<AdapterCacheEntry>` keyed by
  `adapterConfigCacheKey(snapshot)` (`data_adapters/util.ts`). Cross-track
  adapter sharing happens here: two tracks pointing at the same BAM share
  the same `BamAdapter` instance, so they share `setupP` and the `samHeader`.
  Entries with no remaining `sessionIds` are evicted by
  `freeAdapterResources({sessionId})` (line 96).

### Hit-test index (Flatbush, not RBush)

For "box"-style renderers (gene tracks, alignments), the renderer returns a
serialized layout map (feature id → `[leftPx, topPx, rightPx, bottomPx]`). On
the main thread the display reconstructs the layout via
`PrecomputedLayout` (`packages/core/src/util/layouts/PrecomputedLayout.ts:17`):

```ts
class PrecomputedLayout<T> implements BaseLayout<T> {
  private rectangles: Map<string, RectTuple>      // featureId → [l,t,r,b]
  private index?: Flatbush                         // static R-tree
  ...
  private buildIndex() {
    this.index = new Flatbush(this.rectangles.size)
    for (const [name, rect] of this.rectangles) {
      this.index.add(rect[0], rect[1], rect[2], rect[3])
      this.indexData.push({ name, rect })
    }
    this.index.finish()
  }
  getByCoord(x, y) {
    const results = this.index!.search(x, y, x+1, y+1)
    return results.length ? this.indexData[results[0]!]?.name : undefined
  }
}
```

`Flatbush` is a vendored copy of [mourner/flatbush](https://github.com/mourner/flatbush)
at `packages/core/src/util/flatbush/index.ts:1-4` (ISC; non-recursive sort
modifications by Colin Diesh). It is a *static* packed Hilbert R-tree using
flat typed arrays — built once via `add`/`finish`, queried via `search` —
specifically designed for the "we have a fixed bag of rectangles for this
block" use case. JBrowse 2 does **not** use the `rbush` npm package
anywhere (`grep -rn 'RBush\|rbush' packages plugins → no matches`).

`BaseLinearDisplay.getFeatureByID(blockKey, id)`
(`BaseLinearDisplay/model.ts:281`) and `getFeatureByCoord(blockKey, x, y)`
(line 274) delegate to `block.layout.getByCoord(x,y)` / `getByID`.

The on-worker layout used *during* rendering is a different layout class —
`GranularRectLayout` (`packages/core/src/util/layouts/GranularRectLayout.ts:224`)
inside a `MultiLayout` (`MultiLayout.ts`), packed via row-by-row best-fit (see
also `PileupLayout.ts` used for read pile-ups). The renderer serializes the
final rectangles map and the main thread builds the Flatbush from it. So
"layout on the worker, hit-test on the main thread" are different data
structures connected by the rectangles map.

---

## Feature density gating

`plugins/linear-genome-view/src/BaseLinearDisplay/models/FeatureDensityMixin.tsx:23`
runs before every render:

```ts
featureDensityStatsP = rpcManager.call('CoreGetFeatureDensityStats', …)
featureDensityStats  = { featureDensity, bytes, fetchSizeLimit, … }
// regionTooLarge ←  bytes > fetchSizeLimit  (when adapter provides bytes)
//             OR    featureDensity * bpPerPx exceeds user/heuristic threshold
```

Adapter contract: `getMultiRegionFeatureDensityStats(regions, opts)` may return
either feature-count stats (default base-class behavior,
`BaseFeatureDataAdapter.ts:181-189`) or `{ bytes, fetchSizeLimit }` for adapters
that can compute byte cost from their index — BAM (BGZF chunks) and CRAM
(slice bytes) both do this. Quantitative tracks have their own paths via the
wiggle RPC methods.

When the display sees `regionTooLarge`, it renders `TooLargeMessage` with a
"Force load" affordance; clicking it sets `userBpPerPxLimit` on the display
and the autorun re-fires.

---

## Memory at high zoom-out

Hot spots and what they actually cost:

1. **Static blocks across the whole genome.** `ElidedBlock` coalescing keeps
   the BlockSet bounded — visible regions become `ContentBlock`s, hidden runs
   become a single `ElidedBlock`. The view's `bpPerPx` setter clamps to
   `view.maxBpPerPx` based on the smallest reasonable block size.
2. **Adapter feature streams.** `BaseFeatureDataAdapter.getFeatures` returns
   an Observable. Renderers collect features for *the block region only*
   (~800 px). At wider zooms the FeatureDensityMixin short-circuits before
   the fetch, so the adapter is never asked for unfeasibly many features.
3. **ImageBitmaps in `blockState`.** Each rendered block holds an
   `ImageBitmap` of `block.width * height * 4` bytes. ~20 visible blocks ×
   100 px height ≈ ~6 MB per track. Disposed when the reaction disposes.
4. **PrecomputedLayout + Flatbush per block.** Per-block, you carry the
   features' rectangles map plus the Flatbush typed arrays. Flatbush is
   compact (six 32-bit fields per rect inside ArrayBuffers) so this is
   typically smaller than the bitmap.
5. **`displayedRegions`.** `types.frozen<Region[]>` — for whole-genome
   views with many contigs this is a few hundred KB at worst.
6. **`RemoteFileWithRangeCache`.** Hard cap of 2000 × 256 KB ≈ 500 MB upper
   bound per thread (FIFO eviction). Practical usage is much lower.
7. **`CramAdapter.ultraLongFeatureCache`** — `QuickLRU` of up to 500
   long-read Features per CRAM adapter.
8. **VcfAdapter (unindexed)** — whole decompressed file + all parsed
   `VcfFeature`s for any touched refName, retained for the session.

Avoid: storing serialized features on the main thread for the whole genome.
The renderer returns image bitmaps + a rectangles map; hover-detail RPCs
(`CoreGetFeatureDetails`) re-fetch single features on demand.

---

## Embedded-mode tradeoffs

The embedded React products (`products/jbrowse-react-linear-genome-view`,
`-circular-genome-view`, `-app`) differ from `jbrowse-web` in:

- **Plugin set.** Each has a smaller `corePlugins.ts` — e.g. embedded LGV
  ships LGV + the format plugins but omits CircularView, SVInspector, etc.
  Bundle size is consequently a few hundred KB rather than several MB.
- **Default RPC driver.** Configurable. The embedded products accept
  `makeWorkerInstance: () => Worker`
  (`products/jbrowse-react-linear-genome-view/src/createViewState.ts:38`); if
  omitted, the product falls back to `MainThreadRpcDriver`. Embedding pages
  that don't want worker bundling can run all RPC on the main thread at the
  cost of UI jank during heavy rendering.
- **Session model.** Embedded LGV's session lives in
  `packages/embedded-core/` and includes only what the view needs (no
  drawer widgets, no dialog queue). `createViewState` returns a single LGV
  rooted in a stripped session.
- **No persistent storage.** Embedded mode does not write to IndexedDB;
  consumers wire `onChange` and persist snapshots themselves
  (`createViewState.ts:36`).
- **No menus.** Without `RootAppMenuMixin`, plugin `configure()` calls that
  branch on `isAbstractMenuManager(rootModel)` silently skip themselves.

When debugging embedded performance:

- Bundle audit: webpack output in `products/jbrowse-react-*/dist`. The
  `webpack.config.*` files are the source of truth for chunk splits.
- If using `MainThreadRpcDriver`, all rendering happens during the React
  render cycle — large block sets visibly stutter. Switch to a worker via
  `makeWorkerInstance` for production embedding.
- `mobx-react`'s `enableStaticRendering(true)` is invoked only in the
  worker entry; on the embedded main thread observers reactively update.

### Building the worker in embedded mode

`makeWorkerInstance.ts` in each embedded product returns a Worker via
`new Worker(new URL('./rpcWorker', import.meta.url))`. Webpack 5 / Vite 4+
produce a separate worker chunk. Vite users typically need the `?worker`
query suffix; see
`products/jbrowse-react-linear-genome-view/src/makeWorkerInstance.ts` and the
working Vite example at `component_tests/lgv-vite/`.
