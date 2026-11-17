---
name: terrakube-ui
description: Toolchain, commands, layout, API clients, state, dates, bundle-size rules, and testing pitfalls for the Terrakube UI (`terrakube/ui`), a React 19 + TypeScript + antd 6 single-page app built with Vite and tested with Jest.
---

# Terrakube UI Guide (`terrakube/ui`)

React 19 + TypeScript + Ant Design 6, built with Vite (rolldown), tested with
Jest + Testing Library. Read this before changing anything under `terrakube/ui`.

---

## 1. Toolchain

- Node 24 (the CI matrix in `.github/workflows/pull_request_ui.yml` and the
  `node:24-alpine` builder in `ui/Dockerfile`).
- Yarn 4, pinned by `packageManager` in `package.json` and run through Corepack:

  ```bash
  corepack enable
  yarn install --immutable
  ```

  Yarn 1 or npm leave a broken tree (for example, `ts-jest` or the ESLint
  resolver's native binding goes missing).

---

## 2. Commands

| Command | What it does |
|---|---|
| `yarn check` | lint, format check, typecheck and unit tests: run it before every commit |
| `yarn lint:check` / `yarn lint` | ESLint on all of `src` (check / auto-fix); Prettier runs inside ESLint |
| `yarn format:check` / `yarn format` | Prettier on all of `src` |
| `yarn typecheck` | `tsc` for app code and for tests (`tsconfig.app.json`, `tsconfig.test.json`) |
| `yarn test` | Jest; use `yarn test --watchAll=false` or `CI=true yarn test` in automation |
| `yarn build` | production build into `build/`; it does **not** typecheck |
| `yarn build:analyze` | build plus a treemap at `build/bundle-stats.html` |
| `yarn start` | dev server on :3000 (needs `.env`, see `ui/README.md`) |
| `yarn serve` | preview the production build on :3000 |

CI runs `lint:check`, `format:check`, `typecheck`, `test --ci` and `build` on
every UI pull request. Lint warnings do not fail CI yet, but don't add new ones.

Run a single test file with
`yarn test src/modules/workspaces/components/__tests__/WorkspaceCard.test.tsx --watchAll=false`.

---

## 3. Layout

- New code goes in `src/modules/<feature>/` (services, pages, feature
  components) and `src/components/<kind>/` (shared UI: `display`, `feedback`,
  `forms`, `layout`, `modals`, `navigation`, `settings`).
- `src/domain/` is the older layout. Fix things there, but don't add new
  features to it.
- Tests live next to the code in `__tests__/` folders. Use `@/` imports
  (`@/modules/...`), not long `../../` paths.
- Routes are declared in `src/domain/Home/App.tsx`; every page there is
  `lazy()`-loaded.

---

## 4. Talking to the API

| Client | Use it for |
|---|---|
| `apiGet` / `apiPost` / `apiPut` / `apiDelete` from `@/modules/api/apiWrapper` | default for new code: adds the token, never throws, returns `ApiResponse<T>` (`isError`, `data`, `error`), signs out on 401 |
| `axiosInstance` (default export of `@/config/axiosConfig`) | JSON:API calls whose failure should show the app-wide "API not responding" screen |
| `axiosAuxiliary` | optional panels (logs, context, polls): same auth, but 404/5xx/timeouts stay local to the component |
| `axiosGraphQL` | Elide GraphQL (`/graphql/api/v1`) |
| `axiosRegistry` | the registry proxy and other paths at the API origin without `/api/v1` |
| `axiosClient` | unauthenticated calls only: it has no interceptors |

Runtime configuration comes from `window._env_` (written to `env-config.js` at
container start by `env.sh`), never from `import.meta.env`. `jest.setup.ts`
defines it for tests.

---

## 5. State

React state and contexts (`src/context/`, `src/modules/*/…Context.tsx`). A few
values cross pages through `sessionStorage` under the keys in
`src/config/actionTypes.ts`. `src/store` and `src/reducers` are unused Redux
leftovers: don't wire them up or add another global store.

---

## 6. Dates

Format dates through the helpers in `src/modules/utils/dates.ts`
(`relativeTime`, `formatDateTime`, `formatDateTimeWithZone`, `formatDuration`)
instead of calling a date library in components. Don't add another date
library.

---

## 7. Keeping the bundle small

The first page should only load the shell (layout, sidebar, login). Measure
before and after a change that touches imports:

```bash
yarn build >/dev/null && cd build && tot=0
for f in $(grep -o 'assets/[^"]*\.\(js\|css\)' index.html | sort -u); do
  tot=$((tot + $(gzip -c "$f" | wc -c))); done; echo "entry closure: $tot bytes gzip"
```

`yarn build:analyze` writes a treemap to `build/bundle-stats.html`.

Rules:

- Don't add static imports of heavy libraries to the shell or to modules the
  shell imports: `monaco-editor` / `@monaco-editor/react`, `@scalar/*`,
  `react-markdown` and its plugins, `reactflow`, `acorn`, `sucrase`. Load them
  inside a lazy route or with `await import()`.
- Use the code editor only through `@/components/forms/CodeEditor`; elsewhere,
  import only types from `@monaco-editor/react`.
- Import antd components from the package root
  (`import { Button } from "antd"`).
- `src/ActionLoader.tsx` compiles user-defined actions at runtime and gives
  them the antd namespace. Don't narrow what it exposes: existing actions rely
  on it.

---

## 8. Pitfalls

- `console.log/info/debug/warn` are compiled out of production builds
  (`vite.config.mts`); use `console.error` for real errors.
- jsdom lacks a few browser APIs (`ResizeObserver`, `matchMedia`, pseudo-element
  `getComputedStyle`, `TextEncoder`); the shims are in `jest.setup.ts`. Mock
  modules by the same path the component imports
  (`jest.mock("@/config/axiosConfig", …)`).
- Some modules read `window._env_` at import time, so a test must not reset it
  before importing them.
- Unused imports and variables fail lint (`@typescript-eslint/no-unused-vars`).
- Don't silence a check to get green: no `eslint-disable`, `@ts-ignore` or
  `test.skip` without a reason in the PR.
- Public text (code, commits, PRs) must not name competing commercial products.
