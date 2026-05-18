# 03 — Extension Points

**What this covers:** the practical skeletons for writing a JBrowse 2 plugin —
a plugin class, then a minimal AdapterType, ViewType, TrackType, DisplayType,
RendererType, and WidgetType — each using real package imports the way the
existing plugins in this repo do. Also: ConfigurationSchema authoring patterns,
extension points (`addToExtensionPoint`), and pitfalls observed in this codebase
(MST identifier collisions, async init, worker symmetry, SSR concerns).

---

## Imports cheat sheet

```ts
import Plugin from '@jbrowse/core/Plugin'
import type PluginManager from '@jbrowse/core/PluginManager'

import {
  AdapterType,
  ViewType,
  TrackType,
  DisplayType,
  WidgetType,
  RendererType,
  RpcMethodType,
  ConnectionType,
  TextSearchAdapterType,
  InternetAccountType,
  GlyphType,
  AddTrackWorkflowType,
  createBaseTrackModel,
} from '@jbrowse/core/pluggableElementTypes'

import {
  BaseAdapter,
  BaseFeatureDataAdapter,
  BaseSequenceAdapter,
} from '@jbrowse/core/data_adapters/BaseAdapter'

import {
  ConfigurationSchema,
  ConfigurationReference,
  readConfObject,
  getConf,
} from '@jbrowse/core/configuration'

import {
  BaseViewModel,
  BaseDisplay,
  createBaseTrackConfig,
} from '@jbrowse/core/pluggableElementTypes/models'

import { types } from '@jbrowse/mobx-state-tree'
import { lazy } from 'react'
```

---

## Plugin skeleton

```ts
// src/index.ts
import Plugin from '@jbrowse/core/Plugin'
import { ConfigurationSchema } from '@jbrowse/core/configuration'
import { isAbstractMenuManager } from '@jbrowse/core/util'

import MyAdapterF from './MyAdapter/index.ts'
import MyTrackF from './MyTrack/index.ts'
import MyDisplayF from './MyDisplay/index.ts'
import MyRendererF from './MyRenderer/index.ts'
import MyViewF from './MyView/index.ts'
import MyWidgetF from './MyWidget/index.ts'

import type PluginManager from '@jbrowse/core/PluginManager'
import type { AbstractSessionModel } from '@jbrowse/core/util'

export default class MyPlugin extends Plugin {
  name = 'MyPlugin'
  version = '0.1.0'

  // Optional: namespaced plugin-level config slots.
  // Becomes config.configuration.MyPlugin.someOption.
  configurationSchema = ConfigurationSchema('MyPlugin', {
    someOption: { type: 'boolean', defaultValue: true },
  })

  install(pluginManager: PluginManager) {
    MyAdapterF(pluginManager)
    MyRendererF(pluginManager)   // before displays/tracks (phased scheduler)
    MyDisplayF(pluginManager)
    MyTrackF(pluginManager)
    MyViewF(pluginManager)
    MyWidgetF(pluginManager)
  }

  configure(pluginManager: PluginManager) {
    if (isAbstractMenuManager(pluginManager.rootModel)) {
      pluginManager.rootModel.appendToSubMenu(['Add'], {
        label: 'My view',
        onClick: (session: AbstractSessionModel) => {
          session.addView('MyView', {})
        },
      })
    }
  }
}
```

The order of `MyXxxF` calls inside `install` does not strictly matter — the
`PhasedScheduler` (`packages/core/src/PluginManager.ts:110`) reorders by element
group (glyph → renderer → adapter → text search adapter → display → track →
connection → view → widget → rpc method → internet account → add track
workflow). Within a group it preserves call order.

Use `version = …` so future-you can detect plugin upgrades; `name` doubles as
the namespace for `configurationSchema`.

### Patterns to follow

- Every sub-folder exposes a single default-exported function of the form
  `XxxF(pluginManager: PluginManager) { … }`. The naming convention `XxxF` is
  consistent across `plugins/alignments/src/index.ts:5-19` and similar.
- Wrap `ReactComponent` and `getAdapterClass` in `lazy(() => import('…'))` so
  the build can code-split. The Plugin file should not import any heavy React
  components directly.

---

## Custom AdapterType

```ts
// src/MyAdapter/index.ts
import AdapterType from '@jbrowse/core/pluggableElementTypes/AdapterType'
import configSchema from './configSchema.ts'
import type PluginManager from '@jbrowse/core/PluginManager'

export default function MyAdapterF(pluginManager: PluginManager) {
  pluginManager.addAdapterType(() => new AdapterType({
    name: 'MyAdapter',
    displayName: 'My file format',
    configSchema,
    adapterCapabilities: ['getFeatures', 'getRefNames'],
    getAdapterClass: () => import('./MyAdapter.ts').then(r => r.default),
  }))
}
```

```ts
// src/MyAdapter/configSchema.ts
import { ConfigurationSchema } from '@jbrowse/core/configuration'

/** #config MyAdapter */
export default ConfigurationSchema(
  'MyAdapter',
  {
    fileLocation: {
      type: 'fileLocation',
      defaultValue: { uri: '/path/to/my.tsv', locationType: 'UriLocation' },
    },
  },
  { explicitlyTyped: true },
)
```

```ts
// src/MyAdapter/MyAdapter.ts
import { BaseFeatureDataAdapter } from '@jbrowse/core/data_adapters/BaseAdapter'
import { ObservableCreate } from '@jbrowse/core/util/rxjs'
import { openLocation } from '@jbrowse/core/util/io'
import SimpleFeature from '@jbrowse/core/util/simpleFeature'

import type { BaseOptions } from '@jbrowse/core/data_adapters/BaseAdapter'
import type { Feature } from '@jbrowse/core/util'
import type { Region } from '@jbrowse/core/util/types'

export default class MyAdapter extends BaseFeatureDataAdapter {
  private setupP?: Promise<{ refNames: string[] }>

  async getRefNames(_opts?: BaseOptions): Promise<string[]> {
    return (await this.setup()).refNames
  }

  getFeatures(region: Region, _opts?: BaseOptions) {
    return ObservableCreate<Feature>(async observer => {
      const fh = openLocation(this.getConf('fileLocation'), this.pluginManager)
      // parse file, emit features that overlap [region.start, region.end]
      observer.complete()
    })
  }

  freeResources(_region: Region) { /* close file handles etc. */ }

  private async setup() {
    this.setupP ??= (async () => ({ refNames: [/* ... */] }))()
    return this.setupP
  }
}
```

Base classes available (`packages/core/src/data_adapters/BaseAdapter/`):
- `BaseAdapter` — minimum (id, getConf, freeResources). For non-feature data
  (e.g. cytoband, refname alias).
- `BaseFeatureDataAdapter` — adds abstract `getRefNames(opts)` and
  `getFeatures(region, opts): Observable<Feature>`. Used by every BAM/VCF/BED
  adapter.
- `BaseSequenceAdapter` — extends BaseFeatureDataAdapter with `getRegions` and
  expects features carrying `seq`.
- `BaseRefNameAliasAdapter` — `getRefNameAliases()`.
- `RegionsAdapter` — `getRegions()` only.

Use `openLocation(this.getConf('fileLocation'), this.pluginManager)`
(`packages/core/src/util/io/index.ts:46`) — it resolves the right
GenericFilehandle from a `UriLocation` / `LocalPathLocation` / `BlobLocation` /
`FileHandleLocation` and applies any matching InternetAccount auth. `getConf`
forwarded by `BaseAdapter.getConf` runs through `readConfObject` so JEXL
callbacks evaluate.

Sub-adapters: use `this.getSubAdapter(otherAdapterConfig)` (passed by
`dataAdapterCache.ts:48`). The cache de-dupes adapters by config snapshot so
two tracks pointing at the same BAM share an instance.

---

## Custom ViewType

```ts
// src/MyView/index.ts
import { lazy } from 'react'
import { ViewType } from '@jbrowse/core/pluggableElementTypes'
import { stateModelFactory } from './model.ts'
import type PluginManager from '@jbrowse/core/PluginManager'

export default function MyViewF(pluginManager: PluginManager) {
  pluginManager.addViewType(() => new ViewType({
    name: 'MyView',
    displayName: 'My view',
    stateModel: stateModelFactory(pluginManager),
    ReactComponent: lazy(() => import('./components/MyView.tsx')),
  }))
}
```

```ts
// src/MyView/model.ts
import { BaseViewModel } from '@jbrowse/core/pluggableElementTypes/models'
import { ElementId } from '@jbrowse/core/util/types/mst'
import { types } from '@jbrowse/mobx-state-tree'
import type PluginManager from '@jbrowse/core/PluginManager'

export function stateModelFactory(pluginManager: PluginManager) {
  return types
    .compose(
      'MyView',
      BaseViewModel,
      types.model({
        id: ElementId,
        type: types.literal('MyView'),
        tracks: types.array(pluginManager.pluggableMstType('track', 'stateModel')),
      }),
    )
    .volatile(() => ({
      volatileWidth: undefined as number | undefined,
    }))
    .views(self => ({
      get width() { return self.volatileWidth ?? 800 },
    }))
    .actions(self => ({
      setWidth(n: number) { self.volatileWidth = n },
    }))
}
```

Always:
- Compose `BaseViewModel` (`packages/core/src/pluggableElementTypes/models/BaseViewModel.ts`)
  — it provides `displayName`, `setDisplayName`, `setWidth`, menus.
- Use `ElementId` (`packages/core/src/util/types/mst.ts`) — an MST identifier
  that auto-generates a unique ID. Don't use `types.identifier` or `types.string`
  for the `id` slot.
- Set `type: types.literal('MyView')` (must match the registered name).
- Use `pluginManager.pluggableMstType('track', 'stateModel')` — the union of all
  registered track state models. Lets a session snapshot route tracks into the
  correct subtype (`PluginManager.ts:381-401`).
- `extendedName` on a ViewType lets sibling display types automatically attach.
  See `ViewType.ts:36-41` and `PluginManager.addViewType` (PluginManager.ts:549).

The container product reads `getViewType(typeName).ReactComponent` and renders
it with `{model: viewState}`.

---

## Custom TrackType

Tracks are thin: a configuration schema + a base state model. Compatible
DisplayTypes attach automatically.

```ts
// src/MyTrack/index.ts
import { TrackType, createBaseTrackModel } from '@jbrowse/core/pluggableElementTypes'
import configSchemaF from './configSchema.ts'
import type PluginManager from '@jbrowse/core/PluginManager'

export default function MyTrackF(pm: PluginManager) {
  pm.addTrackType(() => {
    const configSchema = configSchemaF(pm)
    return new TrackType({
      name: 'MyTrack',
      displayName: 'My track',
      configSchema,
      stateModel: createBaseTrackModel(pm, 'MyTrack', configSchema),
    })
  })
}
```

```ts
// src/MyTrack/configSchema.ts
import { ConfigurationSchema } from '@jbrowse/core/configuration'
import { createBaseTrackConfig } from '@jbrowse/core/pluggableElementTypes'
import type PluginManager from '@jbrowse/core/PluginManager'

export default (pm: PluginManager) =>
  ConfigurationSchema(
    'MyTrack',
    {},
    {
      baseConfiguration: createBaseTrackConfig(pm),
      explicitIdentifier: 'trackId',
    },
  )
```

Key points:

- `createBaseTrackConfig(pm)` provides `name`, `assemblyNames`, `description`,
  `category`, `displays`, `metadata`, plus the JEXL color callbacks
  (`packages/core/src/pluggableElementTypes/models/baseTrackConfig.ts:22`).
- `createBaseTrackModel(pm, name, configSchema)` constructs an MST state model
  matching that config and exposes `displays`, `configuration`, `setSelected`,
  menu items, save-file actions
  (`packages/core/src/pluggableElementTypes/models/BaseTrackModel.ts:60`).
- `explicitIdentifier: 'trackId'` — required for tracks since `tracks[]` is an
  MST array; the identifier slot is what makes config-by-id resolution work
  (`resolveIdentifier`).
- Don't add behavior to TrackType. Behavior lives in DisplayTypes.

---

## Custom DisplayType

```ts
// src/MyDisplay/index.ts
import { lazy } from 'react'
import { DisplayType } from '@jbrowse/core/pluggableElementTypes'
import configSchemaF from './configSchema.ts'
import modelFactory from './model.ts'
import type PluginManager from '@jbrowse/core/PluginManager'

export default function MyDisplayF(pm: PluginManager) {
  pm.addDisplayType(() => {
    const configSchema = configSchemaF(pm)
    return new DisplayType({
      name: 'MyLinearMyTrackDisplay',
      displayName: 'My display',
      trackType: 'MyTrack',                // attaches to MyTrack
      viewType: 'LinearGenomeView',        // shown only inside LGV
      configSchema,
      stateModel: modelFactory(configSchema),
      ReactComponent: lazy(
        () => import('./components/MyDisplayComponent.tsx'),
      ),
    })
  })
}
```

Display state model patterns:

```ts
// src/MyDisplay/model.ts
import { types } from '@jbrowse/mobx-state-tree'
import { BaseDisplay } from '@jbrowse/core/pluggableElementTypes/models'
import { ConfigurationReference } from '@jbrowse/core/configuration'

export default function modelFactory(configSchema: any) {
  return types
    .compose(
      'MyDisplay',
      BaseDisplay,
      types.model({
        type: types.literal('MyDisplay'),
        configuration: ConfigurationReference(configSchema),
      }),
    )
    .views(self => ({
      get rendererTypeName() { return 'MyRenderer' },
      renderProps() { /* … */ },
    }))
}
```

- A DisplayType is the *only* place a `trackType ↔ viewType ↔ renderer` triple
  is bound. Pluggable elements are looked up by name strings, not class
  references, so spelling must match the registered names exactly.
- For LGV displays, extend `BaseLinearDisplay` (`plugins/linear-genome-view/src/BaseLinearDisplay/model.ts:67`)
  rather than `BaseDisplay` — it provides `blockState`, `renderDelay`,
  feature-density mixin, track-height mixin, and the autorun that drives
  block rendering.
- `ConfigurationReference(configSchema)` (`packages/core/src/configuration/configurationSchema.ts`)
  resolves a configuration node by identifier in the snapshot. This is how
  `display.configuration` points at a slot in `track.configuration.displays`
  without duplicating the snapshot.

`PluginManager.addTrackType` and `addViewType` post-process the registered
display list so that any display whose `trackType` (or `viewType`) names match
gets attached to the corresponding `TrackType.displayTypes` / `ViewType.displayTypes`
(PluginManager.ts:526-567). You can therefore register the DisplayType either
before or after its track/view; both orders work.

---

## Custom RendererType

```ts
// src/MyRenderer/index.ts
import { lazy } from 'react'
import MyRenderer from './MyRenderer.ts'
import configSchema from './configSchema.ts'
import type PluginManager from '@jbrowse/core/PluginManager'

export default function MyRendererF(pm: PluginManager) {
  pm.addRendererType(() => new MyRenderer({
    name: 'MyRenderer',
    ReactComponent: lazy(() => import('./MyRendering.tsx')),
    configSchema,
    pluginManager: pm,
  }))
}
```

```ts
// src/MyRenderer/MyRenderer.ts
import FeatureRendererType from '@jbrowse/core/pluggableElementTypes/renderers/FeatureRendererType'
import type { RenderArgsDeserialized } from '@jbrowse/core/pluggableElementTypes/renderers/ServerSideRendererType'

export default class MyRenderer extends FeatureRendererType {
  supportsSVG = true

  async render(args: RenderArgsDeserialized) {
    // 1. Use `args.features` (Map<id, SimpleFeature>) the parent fetched.
    // 2. Compute layout / draw into a canvas via
    //    @jbrowse/core/util/renderToAbstractCanvas → ImageBitmap.
    // 3. Return { imageData, height, width, layout } — `imageData` should be
    //    a Transferable.
    return { /* … */ }
  }
}
```

Base options:
- Extend `RendererType` for a pure main-thread renderer that returns
  `{reactElement}` (e.g. lightweight overlays).
- Extend `ServerSideRendererType` (`renderers/ServerSideRendererType.ts:69`)
  when you don't need features fetched for you.
- Extend `FeatureRendererType` (`renderers/FeatureRendererType.ts:51`) when you
  want JBrowse to call your adapter's `getFeatures` for you and pass the
  collected features in.
- Extend `BoxRendererType` (`renderers/BoxRendererType.ts`) when you also need
  a hit-test layout (most "feature track"-style renderers).

`@jbrowse/core/util/renderToAbstractCanvas` (`util/renderToAbstractCanvas.ts`)
abstracts over real `OffscreenCanvas`, a node-canvas polyfill, and a recording
canvas used for SVG export. Use it instead of `document.createElement('canvas')`
so SSR + worker rendering keep working.

---

## Custom WidgetType

```ts
// src/MyWidget/index.ts
import { lazy } from 'react'
import { WidgetType } from '@jbrowse/core/pluggableElementTypes'
import configSchema from './configSchema.ts'
import stateModel from './stateModel.ts'
import type PluginManager from '@jbrowse/core/PluginManager'

export default function MyWidgetF(pm: PluginManager) {
  pm.addWidgetType(() => new WidgetType({
    name: 'MyWidget',
    heading: 'My widget',
    configSchema,
    stateModel,
    ReactComponent: lazy(() => import('./MyWidgetComponent.tsx')),
  }))
}
```

Drawer widgets are tracked by `DrawerWidgetSessionMixin`
(`packages/product-core/src/Session/DrawerWidgets.ts:23`). Show via:
```ts
const widget = session.addWidget('MyWidget', 'my-widget-id', { props })
session.showWidget(widget)
```

`heading` is the drawer panel title. Pass `HeadingComponent` for a custom
heading instead. See `BaseFeatureWidget` (`packages/core/src/BaseFeatureWidget/`)
for the canonical reference widget.

---

## Custom RpcMethodType

```ts
import RpcMethodType from '@jbrowse/core/pluggableElementTypes/RpcMethodType'

export class MyComputeStat extends RpcMethodType {
  name = 'MyComputeStat'
  async execute(args: { adapterConfig: any; sessionId: string; range: [number,number] }) {
    const { dataAdapter } = await this.pluginManager
      .lib['@jbrowse/core/data_adapters/dataAdapterCache']
      .getAdapter(this.pluginManager, args.sessionId, args.adapterConfig)
    // do work …
    return result
  }
}

// install:
pm.addRpcMethod(() => new MyComputeStat(pm))

// invoke from anywhere on the main thread:
const rpcManager = getSession(self).rpcManager
const result = await rpcManager.call(getRpcSessionId(self), 'MyComputeStat', args)
```

If your method returns large binary data, return an `ImageBitmap`,
`ArrayBuffer`, or use the `transferables` helpers (`packages/core/src/util/transferables.ts`)
so postMessage transfers ownership instead of structured-cloning.

---

## ConfigurationSchema authoring patterns

```ts
ConfigurationSchema(
  'Foo',
  {
    name:    { type: 'string',   defaultValue: 'foo' },
    color:   { type: 'color',    defaultValue: '#888' },
    height:  { type: 'integer',  defaultValue: 24 },
    visible: { type: 'boolean',  defaultValue: true },
    options: { type: 'frozen',   defaultValue: {} },
    tags:    { type: 'stringArray', defaultValue: [] },

    // enum
    mode: {
      type: 'stringEnum',
      model: types.enumeration('Mode', ['a','b','c']),
      defaultValue: 'a',
    },

    // JEXL callback slot — argument names matter
    jexlColor: {
      type: 'color',
      defaultValue: `jexl:get(feature,'type')=='gene'?'red':'blue'`,
      contextVariable: ['feature'],
    },

    // nested schema
    index: ConfigurationSchema('Index', {
      location: { type: 'fileLocation', defaultValue: { uri: '/x', locationType: 'UriLocation' } },
    }),
  },
  {
    explicitlyTyped: true,        // required for AdapterType/DisplayType subtype unions
    explicitIdentifier: 'fooId',  // makes this addressable by id
    baseConfiguration: createBaseTrackConfig(pm),  // mix in shared track slots
    preProcessSnapshot: snap => /* back-compat reshape */ snap,
    actions: self => ({ /* extra MST actions on the config */ }),
    views:   self => ({ /* extra views */ }),
  },
)
```

Reads:

```ts
readConfObject(myConfig, 'color')                          // primitive
readConfObject(myConfig, ['index', 'location'])            // nested
readConfObject(myConfig, 'jexlColor', { feature })         // JEXL callback
getConf(stateNode, 'foo')                                  // shorthand
```

`getConf` walks `stateNode.configuration` then `parent.configuration` to find
the slot — useful inside MST views/actions.

---

## Common pitfalls

### MST identifier collisions

- Every MST type that participates in a `types.array(union(...))` must have an
  identifier slot. ConfigurationSchemas get one automatically when
  `explicitlyTyped: true` (gives a `type` discriminator) and one of
  `explicitIdentifier` / `implicitIdentifier` is set. Tracks use
  `explicitIdentifier: 'trackId'`; assemblies use `'name'`; displays use
  `'displayId'`.
- Same-name pluggable element collisions are logged but ignored — see
  `PluginManager.addElementType` (PluginManager.ts:325-328):
  `"${groupName} ${newElement.name} already registered, cannot register it again"`.
  Don't name your plugin's elements the same as a core one.
- `types.literal('MyView')` must match exactly the `name` you passed to
  `ViewType`. Otherwise `session.addView('MyView', {})` will reach for the
  ViewType but fail to construct the model.
- For polymorphic config arrays (e.g. `tracks`, `displays`), use
  `pluginManager.pluggableConfigSchemaType('track', 'configSchema')` so every
  track-config subtype is in the union.

### Async init

- `addPlugin` throws if called after `configure()` (PluginManager.ts:213). All
  plugin loading must finish before `configure()`. `PluginLoader.load(url)`
  returns a promise — chain it before constructing `PluginManager`. See
  `products/jbrowse-web/src/createPluginManager.ts:24`.
- Adapters MUST cache the `setup()` promise (`this.setupP ??= …`) so repeated
  calls during render don't double-fetch indexes. Pattern: every `BamAdapter`,
  `VcfTabixAdapter`, `BigWigAdapter` does this; copy them.
- `getAdapter(pluginManager, sessionId, snapshot)` caches the *promise*
  (`packages/core/src/data_adapters/dataAdapterCache.ts:71`). If your
  constructor throws after the promise resolves but during first use, the
  cache will hold a broken adapter — clear `setupP` and `configureResult` on
  failure (e.g. `BamAdapter.ts:86-89`).
- The worker `initializeWorker` flow waits for the configuration message
  before constructing `PluginManager`. RPC calls fired before that
  postMessage will block; the main-thread driver awaits worker `ready`.

### Worker symmetry (SSR concerns)

- Anything `install()` does runs on both main and worker. Don't import
  `react-dom` or browser-only modules inside install — wrap React components
  in `lazy()` so they aren't pulled into the worker chunk.
- `mobx-react` is put into `enableStaticRendering(true)` mode in workers
  (`products/jbrowse-web/src/rpcWorker.ts:7`) — `observer()` components there
  won't reactively re-render. Any worker-side React is one-shot SSR.
- `OffscreenCanvas` is used inside renderers for both worker and main paths.
  Do not assume `document` exists. Use
  `@jbrowse/core/util/offscreenCanvasPonyfill` and
  `renderToAbstractCanvas` for canvas work.
- `window` checks: `typeof window !== 'undefined'`. Many existing files use
  `isWebWorker.ts` (`util/isWebWorker.ts`) for branching.

### Snapshot gotchas

- A `ConfigurationSchema` that "matches default" emits `{}` post-process
  (`configurationSchema.ts:225`). Treat the absence of a key in a snapshot as
  "use default", not as "explicitly empty".
- Volatile properties are stripped. Don't put values you need on reload into
  `.volatile`. If you need a value to survive a reload, put it on the model
  and add a manual postProcessor if it shouldn't go to disk.
- `getSnapshot(rootModel)` on a partially-initialized tree (e.g. before
  `setRootModel` has run on the PluginManager) may resolve `ConfigurationReference`
  slots to `undefined` and produce a snapshot that won't restore.

### Menus and `configure()`

- The "Add" menu wiring lives in `plugin.configure(pluginManager)` and only
  runs once `rootModel` is set. `pluginManager.rootModel` is undefined inside
  the worker. Guard with `isAbstractMenuManager(pluginManager.rootModel)`
  (used in every plugin that adds menus).

### Avoiding circular deps between plugins

- A plugin can `import` from another plugin's package only if both are in the
  same product's `corePlugins.ts`. Prefer wiring through extension points
  (`pluginManager.addToExtensionPoint('Variant-someExt', cb)`) so plugin order
  doesn't matter.

### Extension point patterns

- Registration: `pluginManager.addToExtensionPoint(name, (extendee, props) => extendee)`
  (PluginManager.ts:601).
- Firing (sync): `pluginManager.evaluateExtensionPoint(name, value, props)`
  (line 613). Errors in callbacks are caught and `console.error`'d, then the
  accumulator passes through unchanged — callbacks should not assume their
  side-effects ran.
- Firing (async): `evaluateAsyncExtensionPoint` (line 632).
- `'Core-extendPluggableElement'` is fired for every registered element
  (PluginManager.ts:331); subscribe to extend any pluggable element after
  the source plugin registers it.

### Testing

- `MainThreadRpcDriver` is the default in unit tests so adapter+renderer code
  can be exercised without a worker harness. Set it explicitly with
  `pluginManager.rootModel.jbrowse.configuration.rpc.defaultDriver.set('MainThreadRpcDriver')`.
- `__mocks__/@jbrowse/core/util/useMeasure.ts` (`jest.config.js:3-5`) hands
  out a fixed width to MUI-style measurement hooks. Components that branch
  on `width` will see the mocked default.
