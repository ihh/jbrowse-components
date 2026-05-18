# 01 — Architecture and Layout

**What this covers:** the JBrowse 2 monorepo at a structural level: which workspace
holds what, how `products/` (user-facing apps) consume `packages/` (libraries)
and `plugins/` (most feature code), how a `Plugin` subclass registers itself
through `PluginManager`, and the catalog of pluggable element types that a
plugin may add. Read this first before touching any code in this repo.

---

## Repo top-level

Working tree root: `/home/user/jbrowse-components` (a single pnpm workspace).

```
package.json            workspaces: packages/* products/* plugins/*  (root:13)
pnpm-workspace.yaml     same set, declared for pnpm
tsconfig.json           shared TS root
eslint.config.mjs       flat eslint config (eslint v9)
babel.config.cjs        babel config (used for jest transform)
jest.config.js          three projects: integration, jbrowse-img, default
CONTRIBUTING.md         dev quickstart; explains monorepo
README.md               product blurb + links
website/                docusaurus site + auto-generated docs
docs/                   doc generator scripts (configdocs, statedocs)
test_data/, extra_test_data/  fixture data (BAM, VCF, GFF, FASTA, etc.)
component_tests/        out-of-jest e2e harnesses (puppeteer, vite)
integration.test.js     root-level integration smoke test
```

`@jbrowse/mobx-state-tree` (a JBrowse-maintained fork) is the MST flavor used
throughout. All workspace packages import from it directly, not from upstream
`mobx-state-tree`.

---

## Three-bucket model

```
packages/   — libraries shared by products and plugins (no UI app)
plugins/    — most feature code; each is a Plugin subclass that registers
              pluggable elements (adapters, views, tracks, renderers, …)
products/   — user-facing apps; each bundles a set of plugins and provides
              the root model + session model glue
```

Every workspace folder has its own `package.json`. Plugin packages publish
under `@jbrowse/plugin-<name>`; core packages under `@jbrowse/<name>`; products
that are libraries publish under their own scoped names (e.g.
`@jbrowse/react-linear-genome-view`). Plugins authored by third parties follow
the `jbrowse-plugin-<name>` npm name convention so JBrowse can discover them.

Build trick worth knowing: plugin `package.json` `main` points at `src/index.ts`
during development so products build plugins from source. `publishConfig`
overrides `main`/`module` to point at `dist/` at publish time
(CONTRIBUTING.md:171).

---

## `packages/` (libraries, level 2)

```
packages/
├── core/                       @jbrowse/core — the heart; everything imports it
│   └── src/
│       ├── Plugin.ts                  abstract base class for plugins
│       ├── PluginManager.ts           registry + element creation scheduler
│       ├── PluginLoader.ts            runtime plugin loading (UMD/ESM/CJS)
│       ├── CorePlugin.ts              auto-registered baseline plugin
│       ├── PhasedScheduler.ts         orders element creation phases
│       ├── pluggableElementTypes/     AdapterType, ViewType, TrackType, …
│       ├── data_adapters/             BaseAdapter, dataAdapterCache
│       ├── configuration/             ConfigurationSchema + slots
│       ├── rpc/                       RpcManager, drivers, coreRpcMethods
│       ├── ui/                        shared MUI components, theme
│       ├── util/                      Base1D, blocks, simpleFeature, io, …
│       ├── ReExports/                 jbrequire-able libs for runtime plugins
│       ├── assemblyManager/           assembly/region utilities
│       ├── TextSearch/                TextSearchManager
│       └── BaseFeatureWidget/         default feature-details widget
├── app-core/                   @jbrowse/app-core — root/menu mixins for full apps
│   └── src/{JBrowseModel,RootMenu,HistoryManagement,DockviewLayout,…}
├── product-core/               base session mixins and worker bootstrap
│   └── src/{RootModel,Session,rpcWorker.ts,ui}
├── web-core/                   BaseWebSession (shared by jbrowse-web + react-app)
├── embedded-core/              shared bits for embedded react products
├── sv-core/                    structural-variant shared logic
├── text-indexing/              jbrowse-cli text-indexing helpers
├── text-indexing-core/         text-indexing data structures
└── __mocks__/                  jest module mocks (e.g. useMeasure)
```

`@jbrowse/core` is intentionally "flat" — products import nested sub-paths
directly, e.g. `import PluginManager from '@jbrowse/core/PluginManager'` and
`import { ConfigurationSchema } from '@jbrowse/core/configuration'`
(CONTRIBUTING.md "Notes about monorepo setup"). `tsconfig.build.json` handles
generating types for those sub-paths.

---

## `products/` (level 2)

```
products/
├── jbrowse-web/                   full SPA, webpack; webworker RPC
│   └── src/{index.tsx, InitialLoad.tsx, createPluginManager.ts,
│             corePlugins.ts, rootModel/, sessionModel/,
│             rpcWorker.ts, makeWorkerInstance.ts, tests/}
├── jbrowse-desktop/               Electron wrap of jbrowse-web (local files, offline)
├── jbrowse-react-linear-genome-view/   embeddable LGV (createViewState)
├── jbrowse-react-circular-genome-view/ embeddable CGV
├── jbrowse-react-app/             full app as an embeddable React component
├── jbrowse-img/                   CLI/Node SSR — produces SVG/PNG with no DOM
├── jbrowse-cli/                   `jbrowse` CLI (admin-server, text-index, add-track, …)
└── jbrowse-aws-lambda-functions/  serverless companions
```

Each product owns its own `corePlugins.ts` array, its own root model factory,
and its own webpack/worker entrypoint. They share UI primitives via
`packages/{app,product,web,embedded}-core/` and re-render whichever views the
included plugins register.

`products/jbrowse-web` is the canonical full app to read first. Start at
`products/jbrowse-web/src/createPluginManager.ts:17` to see the complete
boot path (assemble PluginManager → build RootModel → setSession → configure →
expose to React).

---

## `plugins/` (level 2)

Each plugin is an npm package; the source root is `src/`. Plugin source-tree
shape is repetitive: an `index.ts` exporting the Plugin subclass; one folder
per pluggable thing the plugin registers (track/display/renderer/adapter/RPC
method/widget); each folder has an `index.ts` registering with `pluginManager`,
plus `configSchema.ts`, `model.ts`, and React components.

```
plugins/
├── linear-genome-view/        the LGV view; BaseLinearDisplay; LinearBasicDisplay
├── alignments/                BamAdapter, CramAdapter, PileupRenderer, SNPCoverage
├── variants/                  VcfTabix/Vcf/Plink adapters; VariantTrack; LDDisplay
├── wiggle/                    BigWig adapter, XY/Line/Density renderers, MultiWiggle
├── canvas/                    CanvasFeatureRenderer + glyph types (Box, CDS, …)
├── sequence/                  IndexedFasta/BgzipFasta/TwoBit adapters + display
├── gff3/                      Gff3 + Gff3Tabix adapters
├── gtf/                       Gtf adapter
├── bed/                       BedTabix, BigBed adapters
├── hic/                       Hic adapter + circular/linear renderers
├── arc/                       Arc adapter + arc rendering
├── breakpoint-split-view/     view type that pairs LGVs across a breakpoint
├── circular-view/             whole-genome chord/circos view
├── dotplot-view/              synteny dotplot view
├── linear-comparative-view/   stacked LGVs with synteny links
├── spreadsheet-view/          tabular view backing other views
├── sv-inspector/              superview = circular + spreadsheet
├── comparative-adapters/      PAF/MCSCAN/Chain/MashMap adapters
├── config/                    runtime configuration editing UI
├── data-management/           AssemblyManager + Add-Track workflows
├── menus/                     app menubar wiring
├── authentication/            InternetAccount types (OAuth, HTTPBasic, Dropbox, …)
├── grid-bookmark/             bookmark widget
├── jobs-management/           background job tracker
├── legacy-jbrowse/            JBrowse 1 config compatibility
├── lollipop/                  variant lollipop renderer
├── rdf/                       SPARQL adapter
├── trix/                      Trix text-search adapter
├── text-indexing/             text-indexing trigger plugin
└── gccontent/                 GC content quantitative track
```

Cross-references: `products/jbrowse-web/src/corePlugins.ts:1` enumerates the
exact set of plugins bundled into jbrowse-web (29 of them).

---

## The 10 directories worth knowing first

| Directory | Why |
| --- | --- |
| `packages/core/src/pluggableElementTypes/` | every kind of thing a plugin can register; types live here. |
| `packages/core/src/PluginManager.ts` | registry; `addAdapterType`, `addViewType`, `addToExtensionPoint`, phased element creation. |
| `packages/core/src/configuration/` | `ConfigurationSchema`, slot types, `readConfObject`. |
| `packages/core/src/rpc/` | `RpcManager` + Main/WebWorker drivers + `coreRpcMethods`. |
| `packages/core/src/data_adapters/BaseAdapter/` | abstract base classes for adapters; `BaseFeatureDataAdapter` is the most common parent. |
| `packages/core/src/util/` | grab-bag: `Base1DUtils`, `calculateStaticBlocks`, `simpleFeature`, `io`, `rxjs`, `tracks`, type predicates. |
| `packages/core/src/ui/` | shared MUI components: `Dialog`, `Menu`, `ErrorMessage`, `theme.ts`, `Icons`. |
| `packages/product-core/src/Session/` | session mixins (`DrawerWidgets`, `DialogQueue`, `MultipleViews`, `Themes`). |
| `products/jbrowse-web/src/` | the canonical product boot path; mirror this for new products. |
| `plugins/linear-genome-view/src/` | reference plugin: view type, display type, base linear display, block rendering. |

---

## Plugin model

### Class shape (`packages/core/src/Plugin.ts:7`)

```ts
abstract class Plugin {
  abstract name: string
  url?: string
  version?: string
  install(pluginManager: PluginManager): void {}    // register element types
  configure(pluginManager: PluginManager): void {}  // wire to rootModel/menus
  configurationSchema?: AnyConfigurationSchemaType            // → config.<PluginName>.<slot>
  configurationSchemaUnnamespaced?: AnyConfigurationSchemaType // → config.<slot>
  rootConfigurationSchema?: (pm: PluginManager) =>             // spread into root config
    Record<string, AnyConfigurationSchemaType>
}
```

### Lifecycle (`PluginManager.ts`)

1. `new PluginManager([…records])` — pushes `CorePlugin` first
   (PluginManager.ts:163), then calls `plugin.install(this)` for each
   (PluginManager.ts:232). Plugins schedule element creation via
   `pluginManager.addAdapterType(cb)` / `addViewType(cb)` / … which put work
   into a `PhasedScheduler` (PluginManager.ts:110).
2. `.createPluggableElements()` runs the schedule in this fixed order
   (PluginManager.ts:110-123):
   `glyph → renderer → adapter → text search adapter → display → track →
   connection → view → widget → rpc method → internet account →
   add track workflow`. Order matters because tracks pick up displays that
   target them, views pick up displays whose `viewType` matches, etc.
3. `.setRootModel(rootModel)` — gives plugins a handle to the live MST tree.
4. `.configure()` — calls `plugin.configure(this)` on each, typically to add
   menu items via `isAbstractMenuManager(pluginManager.rootModel)` (see
   `plugins/linear-genome-view/src/index.ts:66`).

After `configure()` is called, `addPlugin` throws (PluginManager.ts:213).

### Runtime plugins

`PluginLoader` (`packages/core/src/PluginLoader.ts:1`) loads plugins at
runtime via:

- `UMDLocPluginDefinition` / `UMDUrlPluginDefinition` / `LegacyUMDPluginDefinition`
  (PluginLoader.ts:7-23)
- `ESMLocPluginDefinition` / `ESMUrlPluginDefinition` (PluginLoader.ts:43-51)
- `CJSPluginDefinition` (PluginLoader.ts:67)

These are stored on the JBrowseConfig (the `plugins` slot in `config.json`)
and re-applied inside the web worker so adapter code runs there too
(`packages/product-core/src/rpcWorker.ts:33`).

### Composition example

`products/jbrowse-web/src/createPluginManager.ts:24-49` builds the PluginManager
by concatenating: corePlugins (29 from `corePlugins.ts`) + runtimePlugins
(`config.json` plugins) + sessionPlugins (per-session add-ons). The same array
is shipped to the worker for symmetric installation.

---

## Session composition

A session is an MST node nested as `rootModel.session`. It is built by composing
mixins. For jbrowse-web:

```
JBrowseWebSessionModel  (products/jbrowse-web/src/sessionModel/index.ts:11)
  = BaseWebSession                   (packages/web-core/src/BaseWebSession/…)
    composes →
      BaseSessionModel               (packages/product-core/src/Session/BaseSession.ts:17)
      ConnectionManagementSession    (Connections.ts)
      DrawerWidgetSessionMixin       (DrawerWidgets.ts:23)
      DialogQueueSessionMixin        (DialogQueue.ts:13)
      MultipleViewsSessionMixin      (MultipleViews.ts)
      ReferenceManagementSessionMixin(ReferenceManagement.ts)
      SessionTracksManagementMixin   (SessionTracks.ts)
      ThemeManagementMixin           (Themes.ts)
      TracksManagementSessionMixin   (Tracks.ts)
```

The root model itself composes `BaseRootModelFactory`,
`InternetAccountsRootModelMixin`, `HistoryManagementMixin`, `RootAppMenuMixin`
(`products/jbrowse-web/src/rootModel/rootModel.ts:22-33`). Products differ
mainly in *which* mixins they include — e.g. embedded LGV omits drawer widgets
and menus.

---

## Pluggable element types (one-line each)

Registered in `PluginManager` (line numbers in `packages/core/src/PluginManager.ts`):

| Type | Registry field | `add*` method | Base class file | Purpose |
| --- | --- | --- | --- | --- |
| ViewType | `viewTypes` :142 | `addViewType` :549 | `pluggableElementTypes/ViewType.ts` | a top-level view (LGV, CGV, Dotplot, Spreadsheet) — has stateModel + ReactComponent |
| TrackType | `trackTypes` :136 | `addTrackType` :525 | `TrackType.ts` | a track config-class; aggregates compatible DisplayTypes |
| DisplayType | `displayTypes` :138 | `addDisplayType` :545 | `DisplayType.ts` | how a track renders inside a specific view; ties trackType↔viewType |
| AdapterType | `adapterTypes` :129 | `addAdapterType` :517 | `AdapterType.ts` | wraps a file format into Feature/Region/Sequence APIs |
| RendererType | `rendererTypes` :127 | `addRendererType` :513 | `pluggableElementTypes/renderers/RendererType.tsx` | turns features + props into pixels or SVG (often server-side) |
| WidgetType | `widgetTypes` :144 | `addWidgetType` :569 | `WidgetType.ts` | drawer/dialog widget (feature details, hierarchical track selector) |
| ConnectionType | `connectionTypes` :140 | `addConnectionType` :573 | `ConnectionType.ts` | bulk track-source connector (JBrowse1 hub, UCSC trackHub) |
| RpcMethodType | `rpcMethods` :146 | `addRpcMethod` :577 | `RpcMethodType.ts` | a method runnable on the worker via `rpcManager.call` |
| InternetAccountType | `internetAccountTypes` :150 | `addInternetAccountType` :581 | `InternetAccountType.ts` | auth helper (OAuth2, HTTPBasic, Dropbox, GoogleDrive) |
| TextSearchAdapterType | `textSearchAdapterTypes` :131 | `addTextSearchAdapterType` :521 | `TextSearchAdapterType.ts` | search index backend (Trix-based by default) |
| AddTrackWorkflowType | `addTrackWidgets` :148 | `addAddTrackWorkflowType` :585 | `AddTrackWorkflowType.ts` | custom "Add track" wizard panels |
| GlyphType | `glyphTypes` :125 | `addGlyphType` :589 | `GlyphType.ts` | a sub-renderer used inside CanvasFeatureRenderer for gene drawing |

`PluggableElementType` is the union (`pluggableElementTypes/index.ts:18`). Every
class extends `PluggableElementBase` (`PluggableElementBase.ts:1`) which only
carries `name` and optional `displayName`.

### Extension points (open-ended hooks)

Not a separate type — string-keyed callbacks registered via
`pluginManager.addToExtensionPoint(name, cb)` (PluginManager.ts:601) and fired
with `evaluateExtensionPoint` / `evaluateAsyncExtensionPoint` (lines 613, 632).
`'Core-extendPluggableElement'` is itself fired around every element registration
(PluginManager.ts:331). Other examples are sprinkled through plugins (e.g.
`plugins/variants/src/VcfExtensionPoints/`).

---

## Why this architecture

- **Extensibility by composition, not subclassing.** Track behavior = TrackType
  config-class + one or more DisplayTypes + a RendererType + an AdapterType.
  A new file format usually needs only an AdapterType + a `GuessAdapter` rule;
  it re-uses existing displays/renderers.
- **Same plugins on main + worker.** Adapter code (file parsing) must run on
  the worker; rendering may run on the worker; UI runs on the main thread.
  Because each plugin's `install()` is run in *both* environments (worker via
  `packages/product-core/src/rpcWorker.ts:33`), the same registry is available
  on both sides.
- **Code-splitting friendly.** `ReactComponent` and `getAdapterClass` are
  lazy. `ViewType.ReactComponent`, `WidgetType.ReactComponent`, and
  `DisplayType.ReactComponent` are usually `React.lazy(() => import(…))`; the
  adapter's `import()` runs only when first needed (e.g. `BamAdapter/index.ts:11`).
- **Embedding.** `@jbrowse/react-linear-genome-view` and friends instantiate
  just the components and plugins needed — no menus, no drawer, no multi-view
  router — because the relevant mixins are not composed into the session.
- **Configurability.** Schemas are MST models, so JBrowse can serialize the
  entire app + session as JSON, persist to `localStorage`/`IndexedDB`/URL, and
  re-hydrate deterministically. JEXL callbacks (`@jexl-expression`) inside
  config slots are evaluated lazily and are stored as strings, so snapshots
  round-trip safely.
