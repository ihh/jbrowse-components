# 05 — UX and Conventions

**What this covers:** the UI stack (Material-UI v7, tss-react styling), the
shared component vocabulary in `packages/core/src/ui/`, how the session decides
between drawer widgets / dialogs / menus, the (honest) state of i18n and
accessibility, testing conventions (Jest projects, snapshots, image snapshots),
storybook usage, and lint/format rules likely to bite a new contributor.

---

## UI stack

- **Material-UI v7** — `@mui/material: ^7.3.8` (root `package.json:52`).
  Imports come from `@mui/material` and `@mui/icons-material` directly. No
  Tailwind, no Bootstrap, no Chakra.
- **Emotion** for MUI styling — `@emotion/react`, `@emotion/styled`,
  `@emotion/cache` are present in the root `package.json:46-49`.
- **tss-react (vendored)** for `makeStyles`-style hook styling —
  `packages/core/src/util/tss-react/index.ts:4` re-exports
  `makeStyles` from a local copy. The whole codebase uses:
  ```ts
  import { makeStyles } from '@jbrowse/core/util/tss-react/index'
  const useStyles = makeStyles()(theme => ({ root: { … } }))
  ```
  (e.g. `packages/core/src/ui/Dialog.tsx:21`,
  `packages/core/src/ui/EditableTypography.tsx:18`.) Do not introduce
  inline `sx={…}` props or `styled()` — the convention is `makeStyles()`
  hooks, returning `{ classes }`.
- **React 19** (`react: ^19.2.4`, root `package.json:122`). Use function
  components only. Class components are not used outside legacy error
  boundaries (`packages/core/src/ui/ErrorBoundary.tsx`).
- **mobx-react** for component reactivity — wrap any component that reads MST
  model fields in `observer(…)`. `enableStaticRendering(true)` is set in
  worker entries (`products/jbrowse-web/src/rpcWorker.ts:7`), so the same
  component is also rendered statically on the worker for SSR/SVG export.
- **`react-compiler` (`babel-plugin-react-compiler`)** is enabled
  (`eslint.config.mjs:103-108`). Don't memoize manually with
  `useMemo`/`useCallback` unless you have a profiler-backed reason; the
  compiler handles it.

### React rule of thumb in this repo

- Components live next to their state model in `components/` subdirs.
- Components do not call adapters/RPC directly; they observe their model and
  call MST actions. RPC happens inside the model's autoruns.
- Heavy components are `lazy(…)`-imported from the registration site
  (the `ViewType/WidgetType/DisplayType` constructor) so they only download
  when the user opens the corresponding view/widget/display.

---

## Theming and dark mode

### `defaultThemes`

`packages/core/src/ui/theme.ts:204-210`:

```ts
export const defaultThemes = {
  default:      getDefaultTheme(),
  lightStock:   getLightStockTheme(),
  lightMinimal: getMinimalTheme(),
  darkMinimal:  getDarkMinimalTheme(),
  darkStock:    getDarkStockTheme(),
}
```

`darkMinimal` / `darkStock` are `mode: 'dark'` MUI themes
(`theme.ts:164, 184`). The dark themes carry the same custom palette
extensions — `palette.tertiary`, `palette.quaternary`, `palette.highlight`,
`palette.bases.{A,C,G,T}`, `palette.frames`/`framesCDS` arrays, plus the
genomics-specific scalars `stopCodon`, `startCodon`, `insertion`, `softclip`,
`hardclip`, `skip`, `deletion`. Plugins that draw color-coded features
should read from these palette extensions rather than inventing colors.

### Theme switching

`ThemeManagerSessionMixin`
(`packages/product-core/src/Session/Themes.ts:17`) provides:

- `session.sessionThemeName` — volatile, persisted to `localStorage` under
  the key `'themeName'` (Themes.ts:21, 63-71).
- `session.themeName` — getter that falls back to `'default'` if the name
  isn't in `allThemes()`.
- `session.theme` — fully constructed MUI Theme via `createJBrowseTheme`.
- `session.allThemes()` — merges `defaultThemes` with `jbrowse.configuration.extraThemes`
  so a deployment can supply additional theme overrides via `config.json`.
- `session.setThemeName(name)` — the user-facing setter, called from the
  Preferences dialog.

Embedded products (`jbrowse-react-linear-genome-view`, etc.) accept
`themeName` in their `createViewState({configuration: {theme}})` argument.

---

## Common UI primitives — `packages/core/src/ui/`

Exported from `packages/core/src/ui/index.ts:1-20`:

| Export | File | Role |
| --- | --- | --- |
| `AssemblySelector` | `AssemblySelector.tsx` | dropdown of available assemblies |
| `BaseTooltip` | `BaseTooltip.tsx` | popper-based tooltip used by track tooltips |
| `CascadingMenu` | `CascadingMenu.tsx` | nested submenu component (recursive) |
| `CascadingMenuButton` | `CascadingMenuButton.tsx` | icon-button trigger for one |
| `Dialog` | `Dialog.tsx` | standard JBrowse dialog wrapper (title, close button, error boundary, dark-mode scoped) |
| `DraggableDialog` | `DraggableDialog.tsx` | dragable variant |
| `EditableTypography` | `EditableTypography.tsx` | click-to-edit text label |
| `ErrorBoundary` / `ErrorMessage` / `FatalErrorDialog` | `ErrorMessage*.tsx` | error rendering primitives |
| `ExternalLink` | `ExternalLink.tsx` | `<a target="_blank" rel="noopener">` w/ MUI styles |
| `FileSelector` | `FileSelector/FileSelector.tsx` | URI/local/internet-account file picker (used in Add-track flow) |
| `LoadingEllipses` | `LoadingEllipses.tsx` | animated "Loading…" |
| `LogoFull` / `Logomark` | `Logo.tsx` | JBrowse logos (SVG, `aria-label="JBrowse"`) |
| `Menu` (default) | `Menu.tsx` → `CascadingMenu.tsx` | the canonical menu component |
| `PrerenderedCanvas` | `PrerenderedCanvas.tsx` | draws an ImageBitmap onto a canvas; used by all server-rendered blocks |
| `ResizeHandle` | `ResizeHandle.tsx` | drag handles for drawer/track-height resizing |
| `SanitizedHTML` | `SanitizedHTML.tsx` | DOMPurify-backed safe HTML (use this, never `dangerouslySetInnerHTML`) |
| `VIEW_HEADER_HEIGHT` | `ui/index.ts:20` | constant = 28 px, shared by views |
| `Icons` | `Icons.tsx` | custom non-MUI icons (e.g. `Cable`, `DNA`) |

Snackbar/notifications are in `Snackbar.tsx` + `SnackbarModel.tsx` and wired
through the session's `notify` / `notifyError` API
(`util/types/index.ts:106`).

Other useful primitives outside `ui/`:

- `BaseFeatureWidget` (`packages/core/src/BaseFeatureWidget/`) — the default
  feature-details panel. Reused for almost every track type unless a plugin
  registers a more specific widget.

---

## Drawer widgets vs dialogs vs menus

Distinct mechanisms; choose by user intent.

### Drawer widgets

- A *persistent panel* that stays open until dismissed. Lives in
  `session.widgets` (a `types.map<id, widgetStateModel>`); the active set is
  `session.activeWidgets`.
- Open with:
  ```ts
  const w = session.addWidget('MyWidget', 'unique-id', { /* initial props */ })
  session.showWidget(w)
  ```
  Defined in `packages/product-core/src/Session/DrawerWidgets.ts:23`
  (`addWidget` line 116, `showWidget` line 133).
- Use for: feature details, hierarchical track selector, configuration editor,
  add-connection wizard. Anything the user might want to consult while still
  scrolling a view.
- Drawer position and width are persisted to session snapshot
  (`DrawerWidgets.ts:42-48`), so the same widget reopens at the same width on
  reload.

### Dialogs

- A *modal, transient* component. Lives in `session.queueOfDialogs`
  (volatile), one at a time.
- Open with:
  ```ts
  session.queueDialog((doneCallback) => [MyDialogComponent, { onClose: doneCallback, …props }])
  ```
  Defined in `packages/product-core/src/Session/DialogQueue.ts:13`. The
  session exposes `DialogComponent` + `DialogProps` getters
  (DialogQueue.ts:23-31) consumed by the app shell.
- The dialog itself should be built on top of `@jbrowse/core/ui/Dialog`
  (`Dialog.tsx:49`) which provides the title, close button, and theme/dark-
  mode scoping via `<ScopedCssBaseline><ThemeProvider>` so dialogs over
  rendered SVG/PNG content always have proper background.
- Use for: confirm/cancel flows, file pickers, factory reset, error stack
  traces.

### Menus

- *Transient, contextual*. Generally a `CascadingMenu` (`ui/CascadingMenu.tsx`),
  optionally triggered by a `CascadingMenuButton`.
- App-level menu items (the menubar) are mutable on the rootModel —
  `RootAppMenuMixin` (`packages/app-core/src/RootMenu/`) exposes
  `appendToSubMenu`, `appendMenu`, `insertInMenu`, etc.
- Track right-click menus: a track's state model exposes
  `trackMenuItems()` / `extraMenuItems()` returning `MenuItem[]` from
  `packages/core/src/ui/MenuTypes.ts`; the LGV reads these to assemble the
  context menu in `LinearGenomeView/menuItems.ts`.

### Decision matrix

| Need | Use |
| --- | --- |
| Persistent side panel of details while user keeps interacting with view | DrawerWidget |
| Modal "OK / cancel" or wizard | Dialog (via `queueDialog`) |
| Context menu on right-click or icon button | Menu (`CascadingMenu`) |
| Quick toast | `session.notify(message)` (Snackbar) |
| Recoverable error | `session.notifyError(message, error)` |

---

## Internationalization (i18n)

**Honest status: there is no i18n.** No `react-i18next`, no `i18next`, no
`formatMessage`/`<FormattedMessage>` anywhere in `packages/`, `plugins/`, or
`products/`. All strings are hard-coded English. There is no localization
infrastructure to extend; adding it would require a cross-cutting refactor of
every plugin's UI files.

If you must localize: wrap user-visible text via your own `t()` helper inside
your plugin, fed by the JBrowse config (e.g. add a plugin-level config slot
that holds a translation table). Don't expect upstream to accept it without
broader buy-in.

---

## Accessibility (a11y)

**Honest status: partial.** ARIA attributes appear sporadically. The codebase
relies heavily on MUI's built-in a11y (buttons, dialogs, popovers all have
sensible roles).

Specifically found:
- `aria-label` on logos (`packages/core/src/ui/Logo.tsx:84`) and on the
  file-selector toggle group (`ui/FileSelector/SourceTypeSelector.tsx:70-77`).
- `aria-label` set on color-picker sliders (`ui/react-colorful.ts:277, 305, 418`).
- `aria-autocomplete` / `aria-expanded` etc. inherited from MUI Autocomplete
  in the search box (visible in snapshot files).

Gaps to be honest about:
- The canvas-rendered tracks have no programmatic representation (no role,
  no live region, no alt-text equivalent). A screen reader sees a featureless
  `<canvas>`.
- Keyboard navigation works for menus/dialogs (MUI defaults) and the LGV has
  arrow-key handlers (`LinearGenomeView/keyboardHandler.ts`) but there is no
  focus-trap on drawer widgets and no consistent focus ring on internal track
  controls.
- Colour palettes (`palette.bases`, frame colors) are picked for visual
  distinction, not contrast — dark mode mostly inherits from MUI but the
  per-base colors are fixed bright values.

If you need to file or address a11y work, start from the MUI primitives and
the `ResizeHandle` / `CascadingMenu` / `EditableTypography` components — those
are home-grown and most likely to need attention.

---

## Testing

### Jest projects (`jest.config.js`)

Three projects run in parallel:

1. **`integration`** — only `<rootDir>/integration.test.js`, Node
   environment. Smoke test against built artifacts.
2. **`jbrowse-img`** — `<rootDir>/products/jbrowse-img/**/*.test.ts`, Node
   environment, *no* `jest-fetch-mock`. Uses native `fetch` since it tests
   actual file-fetching against fixtures.
3. **`default`** — everything else under `{packages,products,plugins}/**`
   matching `*.test.{ts,tsx,js,jsx}`, **jsdom** environment, with
   `config/jest/fetchMockAfterEnv.js` enabling `jest-fetch-mock` after the
   environment is set up.

All projects share the same Babel transform (`config/jest/babelTransform.cjs`),
CSS transform (`config/jest/cssTransform.cjs`), and these setupFiles:
- `config/jest/textEncoder.js` (polyfill `TextEncoder` in jsdom)
- `config/jest/console.js` (fail tests on unexpected console output)
- `config/jest/messagechannel.js`
- `config/jest/structuredClone.js`
- `config/jest/setHTML.js`

Module mocks (`jest.config.js:3-6`):
- `@jbrowse/core/util/useMeasure` → `packages/__mocks__/@jbrowse/core/util/useMeasure.ts`
  (returns a fixed width so MUI measurement hooks don't depend on real DOM
  layout).
- `@jbrowse/text-indexing-core` → its src directly.

### Conventions

- ~201 `.test.{ts,tsx}` files under `packages/` and `plugins/`. Tests sit
  next to source (`Foo.ts` + `Foo.test.ts`).
- **Snapshot tests** in `__snapshots__/` are widely used (each LGV view test
  emits 200+ KB of jsx snapshot). When intentional UI changes occur, update
  with `pnpm test -u`.
- **Image snapshots** (jest-image-snapshot, `@types/jest-image-snapshot`) are
  used for canvas-render diffing in `plugins/canvas/src/glyphs/__image_snapshots__/`
  and per-plugin folders. They require `canvas` (native binding) which is
  why CONTRIBUTING.md walks through `brew install pkg-config cairo pango …`.
- `@testing-library/react` + `@testing-library/jest-dom` are the React testing
  primitives.
- **End-to-end harnesses** live under `component_tests/` (puppeteer + Vite
  builds for embedded products) and `products/jbrowse-desktop/test/e2e.ts`
  (Electron). These are run manually, not from `pnpm test`.

### Storybook

Storybook 10 (`storybook: ^10.2.14`). Stories exist for the embeddable
products and core:

```
packages/core/stories/JBrowseCore.stories.tsx
products/jbrowse-react-app/stories/JBrowseReactApp.stories.tsx
products/jbrowse-react-linear-genome-view/stories/JBrowseLinearGenomeView.stories.tsx
products/jbrowse-react-circular-genome-view/stories/JBrowseCircularGenomeView.stories.tsx
```

Run with `cd products/jbrowse-react-linear-genome-view && pnpm storybook`
(CONTRIBUTING.md:84). There are no stories under `plugins/*` — the canonical
manual test path for plugin UI is the jbrowse-web dev server, not Storybook.

---

## Lint / format / typecheck

### Scripts (root `package.json:12-20`)

- `pnpm lint` — flat config, `eslint.config.mjs`, `--max-warnings 0`,
  `--report-unused-disable-directives`. Heavy; takes time
  (CONTRIBUTING.md:91).
- `pnpm format` — `prettier --write .`.
- `pnpm check-format` — `prettier --check .`.
- `pnpm typecheck` — `tsc --noEmit` from repo root using shared
  `tsconfig.json`. CI runs this; editors should pick it up automatically.
- `pnpm test` / `pnpm test-ci` — runs Jest. CI uses
  `NODE_OPTIONS='--max-old-space-size=7000'`.
- `pnpm autogen` — regenerates `website/docs/config` and
  `website/docs/models` from JSDoc-style comments (`#config`, `#stateModel`,
  `#slot`, etc.) embedded in source files. If you add a new pluggable
  element or config schema, run this and commit the result.

### Prettier (`.prettierrc.json`)

```json
{
  "semi": false,
  "singleQuote": true,
  "trailingComma": "all",
  "arrowParens": "avoid",
  "proseWrap": "always"
}
```

No semicolons. Single quotes. Trailing commas everywhere. Arrow with single
arg has no parens (`x => x`, not `(x) => x`).

### ESLint highlights (`eslint.config.mjs`)

The config composes recommended sets from `@eslint/js`,
`typescript-eslint/recommended` + `stylisticTypeChecked` +
`strictTypeChecked`, `import`, `react`, `react-hooks`, `unicorn`,
`react-refresh`, `react-compiler`, `tss-unused-classes`. Notable explicit
rules (lines 132-165):

- `no-restricted-globals: ['error', 'Buffer']` — never use Node's `Buffer`
  in this code (use `Uint8Array`).
- `no-console: ['error', { allow: ['error', 'warn'] }]` — only
  `console.warn` / `console.error` allowed.
- `tss-unused-classes/unused-classes: 'warn'` — `makeStyles` classes you
  define but never reference will warn.
- `curly: 'error'`, `semi: ['error', 'never']`, `prefer-template`,
  `one-var: ['error', 'never']`.
- `react-refresh/only-export-components: 'error'` — files exporting React
  components shouldn't also export unrelated values (breaks fast refresh).
- `react-compiler/react-compiler: 'error'` — enabled, so the React compiler
  will catch hook/component rule violations.

The flat config disables type-checking on several `unicorn/*` rules that are
noisy in this codebase, and ignores test fixtures, build outputs, webpack
configs, the website, the AWS lambda functions, and the workerPolyfill
files. See `eslint.config.mjs:14-83`.

### Typos check

`_typos.toml` (root) is the [`typos`](https://github.com/crate-ci/typos)
allowlist. Run `typos` if installed; otherwise it's a CI step.

### Doc generation

`pnpm configdocs` / `pnpm statedocs` rebuild documentation from inline
JSDoc-like markers in source:
- `#config <Name>` — declares a config schema for doc extraction.
- `#slot` / `#slot path.to.slot` — annotates a slot.
- `#preProcessSnapshot` — marks a preProcessor block.
- `#stateModel <Name>` — declares an MST model.
- `#property`, `#volatile`, `#getter`, `#method`, `#action` — annotate
  fields/actions of state models for the docs site.

These markers are pure comments; they don't affect runtime. But they are
linted into the website on release, so keep them in sync when you add or
rename config slots.

### Misc conventions

- Imports are sorted by `eslint-plugin-import` rules (relative imports after
  absolute, type-only imports last). Run prettier + lint after edits — the
  combined autofix sorts everything.
- `pnpm install` post-install runs `pnpm rebuild canvas` only on demand;
  most contributors will need it (CONTRIBUTING.md:32). Without the canvas
  native binding, image-snapshot tests fail.
- Symlinks are used in the repo (CONTRIBUTING.md:43). Windows users must
  `git config --global core.symlinks true` before cloning.
- Commit hook: none enforced by default in this branch, but CI runs
  `lint`, `typecheck`, and the full Jest suite — keep all three green.
