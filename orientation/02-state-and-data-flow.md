# 02 — State and Data Flow

**What this covers:** how state lives in this codebase using
`@jbrowse/mobx-state-tree`; the root/session/view/track/display nesting; the
volatile-vs-persistent split; the configuration system (slots, schemas,
snapshots, JEXL callbacks); the RPC boundary between the main thread and web
worker; and the end-to-end path a single block of pixels travels from a view
back to a renderer call.

---

## MST flavor

All MST imports come from `@jbrowse/mobx-state-tree` (a fork that lives in this
monorepo's lockfile). The API surface used in this codebase is the standard MST
one — `types.model`, `.props`, `.volatile`, `.views`, `.actions`, `types.compose`,
`addDisposer`, `getSnapshot`, `getParent`, `getRoot`, `getEnv`, `flow`, etc. The
fork mainly fixes typings; do not assume any other deviations.

`mobx` is the autoresponse engine. Components are wrapped in `observer(…)` from
`mobx-react`. Side-effects use `autorun` + `addDisposer` (see e.g.
`plugins/linear-genome-view/src/BaseLinearDisplay/models/serverSideRenderedBlock.ts:271`),
not `useEffect`, for MST-driven reactions.

---

## The state tree

```
RootModel  (per product, factory in products/<product>/src/rootModel/)
 ├── jbrowse        JBrowseConfig MST node — the persisted config.json
 │   ├── configuration         RootConfiguration ConfigurationSchema
 │   ├── assemblies            array of assembly ConfigurationSchemas
 │   ├── tracks                array of track ConfigurationSchemas
 │   ├── internetAccounts      array of internetAccount ConfigurationSchemas
 │   ├── connections           array of connection ConfigurationSchemas
 │   └── plugins               PluginDefinition[]
 ├── session        the live session (volatile-mostly, replaceable)
 │   ├── id, name
 │   ├── views[]               array of view models (LGV, CGV, …)
 │   ├── widgets               map<id, widgetStateModel>
 │   ├── activeWidgets         map<id, safeReference→widget>
 │   ├── queueOfDialogs        volatile [Component, props] stack
 │   ├── sessionTracks         per-session track configs (not persisted to config.json)
 │   ├── sessionConnections    per-session connections
 │   ├── connectionInstances   live connection models
 │   ├── temporaryAssemblies   transient assemblies
 │   ├── selection             volatile global "selected thing"
 │   └── hovered               volatile global "hovered thing"
 ├── assemblyManager           assemblyManager (`packages/core/src/assemblyManager`)
 ├── rpcManager                RpcManager (`packages/core/src/rpc/RpcManager.ts`)
 ├── textSearchManager         TextSearchManager
 ├── internetAccounts[]        live InternetAccount instances
 ├── adminMode
 ├── history                   undo/redo (HistoryManagementMixin)
 └── menus                     mutable menu spec (RootAppMenuMixin)
```

Cross-references:

- Root model factory: `products/jbrowse-web/src/rootModel/rootModel.ts:88`
  composes `BaseRootModelFactory`, `InternetAccountsRootModelMixin`,
  `HistoryManagementMixin`, `RootAppMenuMixin`.
- Base session: `packages/product-core/src/Session/BaseSession.ts:17` — only
  `id`, `name`, `margin`, `selection`, `hovered`.
- Composed jbrowse-web session mixins:
  `packages/web-core/src/BaseWebSession/` composes `ConnectionManagementSession`,
  `DrawerWidgetSessionMixin` (`Session/DrawerWidgets.ts:23`),
  `DialogQueueSessionMixin` (`Session/DialogQueue.ts:13`),
  `MultipleViewsSessionMixin`, `ReferenceManagementSessionMixin`,
  `SessionTracksManagementMixin`, `ThemeManagementMixin`,
  `TracksManagementSessionMixin`.

### View → Track → Display nesting

`LinearGenomeView.tracks` is `types.array(pluginManager.pluggableMstType('track', 'stateModel'))`
(`plugins/linear-genome-view/src/LinearGenomeView/model.ts:141`). The track
state model has a `displays` array of pluggable display state models. So:

```
session.views[0]   = LinearGenomeView instance
  .tracks[i]       = e.g. AlignmentsTrack
    .displays[j]   = e.g. LinearAlignmentsDisplay
      .blockState  = map<blockKey, BlockState>
        .features  = layout result
        .data      = serialized rendered image data / react element
```

Track lookup helpers (`packages/core/src/util/index.ts`):
`getContainingView`, `getContainingTrack`, `getContainingDisplay`, `getSession`,
`getEnv`. These walk parents via `getParent`.

---

## Volatile vs persistent

MST distinguishes serializable `props` from volatile (per-instance, non-serializable)
state. Conventions in this repo:

- **Persistent (`types.model({...})`):** anything saved to a session snapshot
  or to `config.json`. IDs, offsets/zoom level (LGV `offsetPx`, `bpPerPx`),
  displayed regions, track configurations, widget configurations.
- **Volatile (`.volatile(self => ({...}))`):** anything that is too big to
  serialize, holds non-cloneable values (DOM measurements, Promises, file
  handles, ImageBitmaps), or is meaningful only during a session. Naming
  convention: prefix volatile properties that "shadow" a persistent value
  with `volatile`, e.g. `volatileWidth`, `volatileError`, `volatileGuides`
  in `LinearGenomeView/model.ts:242-305`. Getters then take the volatile
  value if defined and fall back to a persistent default.
- **Frozen blobs:** for large structured data that must persist but never be
  navigated (e.g. `displayedRegions`), the codebase uses
  `types.frozen<Region[]>()` (`LinearGenomeView/model.ts:135`).

`.preProcessSnapshot` and `.postProcessSnapshot` are used heavily on
configuration schemas to compact snapshots — only non-default keys are written
(`packages/core/src/configuration/configurationSchema.ts:209-235`). This keeps
saved sessions small and human-editable.

`getSnapshot(model)` returns the serializable subtree. Volatile fields are
omitted. URL/IndexedDB session sharing relies on this.

Time travel & undo: `packages/core/src/util/TimeTraveller.ts` provides patch
recording; `HistoryManagementMixin` wires it into the root model.

---

## Configuration system

`ConfigurationSchema` is *not* a plain MST type — it is a higher-level helper
that builds an MST model from a slot specification. Schema definition lives in
`packages/core/src/configuration/configurationSchema.ts`.

### Authoring a schema

```ts
// plugins/alignments/src/BamAdapter/configSchema.ts
ConfigurationSchema(
  'BamAdapter',
  {
    bamLocation: { type: 'fileLocation', defaultValue: {...} },
    index: ConfigurationSchema('BamIndex', {
      indexType: {
        type: 'stringEnum',
        model: types.enumeration('IndexType', ['BAI', 'CSI']),
        defaultValue: 'BAI',
      },
      location: { type: 'fileLocation', defaultValue: {...} },
    }),
    fetchSizeLimit: { type: 'number', defaultValue: 5_000_000 },
  },
  {
    explicitlyTyped: true,
    preProcessSnapshot: snap => snap.uri ? { ...snap, bamLocation: {…} } : snap,
  },
)
```

Slot `type` strings recognised (`configurationSlot.ts:14-24`):
`stringArray, stringArrayMap, numberMap, boolean, color, integer, number,
string, text, fileLocation, frozen`. Plus the union `stringEnum` when a
`model: types.enumeration(...)` is supplied.

### Schema-builder options (`ConfigurationSchemaOptions`)

`configurationSchema.ts:49-66`:

- `explicitlyTyped` — emit a `type` discriminator slot (required for
  pluggable subtype unions).
- `explicitIdentifier` / `implicitIdentifier` — name of the identifier slot
  (e.g. `trackId`, `displayId`, `assemblyName`).
- `baseConfiguration` — extend another ConfigurationSchema's slots/options;
  e.g. every track config uses
  `createBaseTrackConfig(pluginManager)` (`packages/core/src/pluggableElementTypes/models/baseTrackConfig.ts:22`).
- `preProcessSnapshot` — back-compat shims; e.g. allow `{uri:'x.bam'}` to
  expand to a full BamAdapter config.
- `actions` / `views` / `extend` — attach MST actions/views/extensions to the
  schema model.

### Reading a schema

```ts
import { readConfObject, getConf } from '@jbrowse/core/configuration'
readConfObject(configModel, 'bamLocation')          // single slot
readConfObject(configModel, ['index', 'location'])  // nested
readConfObject(configModel, 'color', { feature })   // pass callback args (JEXL)
getConf(stateTreeNode, 'name')                      // shorthand via parent.configuration
```

`readConfObject` is defined in `packages/core/src/configuration/util.ts:50`. It
walks the path, calls `.getValue(args)` if the slot is a config slot, else
returns the nested schema/value. JEXL callback slots — strings of the form
`jexl:…` — are compiled lazily and run in the slot's `getValue`.

### Snapshot/restore

`getSnapshot(rootModel.jbrowse)` produces the saved `config.json`. The post-
processor in `configurationSchema.ts:210-234` strips any property equal to its
default and any empty object/array, *and* returns `{}` whenever every slot
matches the defaults — making snapshots minimal and diffable. The reverse is
`Model.create(snap, { pluginManager })`; passing the `pluginManager` in the
environment is how config callbacks reach the plugin registry (see
`packages/core/src/data_adapters/dataAdapterCache.ts:39`).

### Plugin-level configuration

A Plugin can contribute config in three places (`packages/core/src/Plugin.ts`):

- `configurationSchema`           → `configuration.<PluginName>.<slot>`
- `configurationSchemaUnnamespaced` → `configuration.<slot>` (flat)
- `rootConfigurationSchema(pm)`  → spread into the root `JBrowseConfig`

Collected and merged by `PluginManager.pluginConfigurationNamespacedSchemas()` /
`pluginConfigurationUnnamespacedSchemas()` /
`pluginConfigurationRootSchemas()` (PluginManager.ts:176-210).

---

## RPC boundaries

```
main thread (UI)                      web worker (rpcWorker.ts)
─────────────────                     ─────────────────────────
React + observer components           PluginManager (mirror)
session/view/track state              data adapter cache
rpcManager.call(sessionId, name, ─►   RpcServer (librpc.ts)
  args, opts)                         ├─ CoreGetFeatures
                                      ├─ CoreRender
                                      ├─ CoreGetFeatureDensityStats
                                      ├─ CoreGetSequence
                                      ├─ CoreFreeResources
                                      ├─ CoreGetMetadata
                                      ├─ CoreGetFileInfo
                                      ├─ CoreGetRegions
                                      ├─ CoreGetRefNames
                                      └─ plugin-registered RpcMethodTypes
```

### `RpcManager` (`packages/core/src/rpc/RpcManager.ts:30`)

Holds `driverObjects: Map<string,DriverClass>` and `driverFactories`. Two
drivers are registered by default at construction (RpcManager.ts:46-71):

- `MainThreadRpcDriver` — runs RPC methods inline on the main thread.
  Default for `react-linear-genome-view` and tests.
  Implementation: `packages/core/src/rpc/MainThreadRpcDriver.ts`.
- `WebWorkerRpcDriver` — sends messages to a pool of workers via
  `RpcClient` (`packages/core/src/util/librpc.ts`).
  Implementation: `packages/core/src/rpc/WebWorkerRpcDriver.ts`.

Driver selection happens at call time: `args.rpcDriverName ||
readConfObject(this.mainConfiguration, 'defaultDriver')` (RpcManager.ts:108).
jbrowse-web defaults to `WebWorkerRpcDriver`
(`products/jbrowse-web/src/createPluginManager.ts:65-69`).

### Worker bootstrap

- Main side: `makeWorkerInstance.ts:7` returns `new Worker(new URL('./rpcWorker', import.meta.url))`.
- Worker side: `products/jbrowse-web/src/rpcWorker.ts` calls
  `initializeWorker(corePlugins, { fetchESM })`
  (`packages/product-core/src/rpcWorker.ts:78`). The worker waits for a
  `'config'` postMessage carrying `{ plugins: PluginDefinition[], windowHref }`
  (rpcWorker.ts:23-32), then `new PluginManager(...).createPluggableElements().configure()`
  and starts an `RpcServer`. Crucially, the worker installs the same plugins
  the main thread does, so `getAdapter(pluginManager, sessionId, snapshot)` on
  the worker can instantiate any registered AdapterType.

### What runs where

| Concern | Main | Worker |
| --- | --- | --- |
| React render / MST mutations | ✓ | (mobx-react `enableStaticRendering(true)` on worker, `rpcWorker.ts:7`) |
| Data adapter `getFeatures` (BAM/CRAM/VCF/BigWig) | for `MainThreadRpcDriver` only | ✓ in worker mode |
| Renderer `render()` producing ImageBitmap / SVG | for `MainThreadRpcDriver` only | ✓ in worker mode |
| Adapter cache (`dataAdapterCache.ts`) | per-thread; both have one | ✓ |
| TextSearchManager queries | ✓ (CoreTextSearch RPC available) | ✓ |
| File handle access (`File`/`Blob` from drag-drop) | ✓ — transferred as `BlobMap` | accessed via `setBlobMap` on RPC arg deserialize |

### Serialization details

- `BaseRpcDriver.filterArgs` strips uncloneable values (functions, Errors)
  before postMessage (`BaseRpcDriver.ts:73`).
- `RpcMethodType.serializeArguments` walks args, converting `FileHandleLocation`
  references into `BlobLocation` entries against a worker-side blob map
  (`packages/core/src/pluggableElementTypes/RpcMethodType.ts:21-71`), and
  augments `UriLocation` entries with `internetAccountPreAuthorization` if an
  InternetAccount applies.
- `coreRpcMethods.ts:1-11` registers all `Core*` methods. Plugins add their
  own with `pluginManager.addRpcMethod` (e.g. wiggle's
  `WiggleGetGlobalQuantitativeStats`, alignments' Pileup RPC methods).

---

## Track render request flow

End-to-end path for filling one block of one display:

```
1. LinearGenomeView changes offsetPx/bpPerPx
   └─ calculateStaticBlocks(model) builds a BlockSet
      (packages/core/src/util/calculateStaticBlocks.ts:23)

2. BaseLinearDisplay autorun reacts to staticBlocks
   └─ for each block key, it sets a BlockState in display.blockState
      (BaseLinearDisplay/models/serverSideRenderedBlock.ts)

3. BlockState.afterAttach kicks off makeAbortableReaction
   └─ name: "<display.id>/<locString> rendering" (line 276)
      delay: display.renderDelay
      data fn: renderBlockData(self)           (line 295)
      effect fn: renderBlockEffect             (effects RPC + setRendered)

4. renderBlockData collects:
     rendererType   = display.rendererType    (a RendererType instance)
     rpcManager     = session.rpcManager
     renderProps    = display.renderProps()   (config, theme, regions, …)
     renderingProps = display.renderingProps?.()
     adapterConfig  = display.adapterConfig
     sessionId      = getRpcSessionId(display)
     trackInstanceId= parentTrack.id
     blockKey       = self.key
     regions        = [self.region with seqAdapterRefName]

5. renderBlockEffect → rendererType.renderInClient(rpcManager, args)
   └─ defaults to rpcManager.call(sessionId, 'CoreRender', args)
      (ServerSideRendererType.ts; methods/CoreRender.ts)

6. CoreRender on the worker:
   a. getAdapter(pluginManager, sessionId, adapterConfig)
      (data_adapters/dataAdapterCache.ts:62) — instantiates AdapterType class
   b. rendererType.getFeatures(region, opts) on the adapter
      (FeatureRendererType.ts; e.g. BamAdapter.getFeatures returns Observable)
   c. rendererType.render({features, regions, config, …})
      (subclass-specific: PileupRenderer.makeImageData,
       XYPlotRenderer.makeImageData, etc.)
   d. result is `{ imageData: ImageBitmap, height, width, … }` — Transferable
      so it crosses the worker boundary by reference, not copy.

7. Main thread BlockState.setRendered(result)
   └─ The block's React component (BaseLinearDisplay → BlockMsg) now reads
      block.data and either draws the ImageBitmap to a canvas or renders the
      returned `reactElement`.

8. On scroll out / display close, BaseLinearDisplay calls freeResources
   → CoreFreeResources RPC → dataAdapter.freeResources(region) +
      cache eviction (dataAdapterCache.ts:freeAdapterResources).
```

Cancellation uses `StopToken` (`packages/core/src/util/stopToken.ts`) propagated
through the RPC and checked inside long-running operations (BamAdapter's
feature stream calls `checkStopToken` between chunks).

### Renderer subclasses to know

- `RendererType` (`pluggableElementTypes/renderers/RendererType.tsx:21`) —
  base; for entirely main-thread renderers, override `render(props)` to return
  `{ reactElement }`.
- `ServerSideRendererType` (`renderers/ServerSideRendererType.ts:69`) — when
  rendering should run on the worker; serializes config, deserializes args,
  produces transferable images, supports SVG export.
- `FeatureRendererType` (`renderers/FeatureRendererType.ts:51`) — extends
  ServerSide and additionally calls the adapter to fetch features for the
  block before invoking `render`.
- `BoxRendererType` (`renderers/BoxRendererType.ts`) — adds a
  `LayoutSession` that returns feature positions for hit-testing on the main
  thread.
- `CircularChordRendererType` (`renderers/CircularChordRendererType.tsx`) —
  CGV-specific.

### Block lifecycle invariants

- A BlockState entry is keyed by a stable `blockKey` (assembly+location+
  reload-flag tuple); reload requests bump `reloadFlag` so the same key gets a
  fresh `makeAbortableReaction` round.
- `display.renderDelay` (BaseLinearDisplay) debounces the autorun and prevents
  storm-of-RPCs while scrubbing/zooming.
- The reaction's effect uses `addDisposer` so closing a display cancels
  pending RPCs cleanly.
