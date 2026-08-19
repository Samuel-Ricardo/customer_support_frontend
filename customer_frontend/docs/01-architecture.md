# Customer Support Frontend — Architecture Analysis

**Status:** Baseline analysis of the current codebase (SolidStart boilerplate)
**Date:** 2026-08-19
**Scope:** `customer_frontend/` — the frontend application inside the `customer_support_frontend` repository
**Branches:** `develop` (checked out), `main`
**Remote:** `git@github.com:Samuel-Ricardo/customer_support_frontend.git`

---

## 1. Overview

The application is a **SolidStart v1 meta-framework** project, created from the official
`example-with-tailwindcss` template and still unmodified apart from tooling (Docker,
lint-staged, Jest, Cypress dev dependencies). It is a server-rendered SolidJS SPA with
file-based routing, granular reactive rendering (no virtual DOM), and a
**vinxi → Nitro** build pipeline that produces a standalone Node.js server.

### 1.1 Verified stack

| Layer | Package | Declared | Installed | Notes |
|---|---|---|---|---|
| Framework | `solid-js` | `^1.8.18` | `1.8.22` | Signals, fine-grained reactivity |
| Meta-framework | `@solidjs/start` | `^1.0.6` | `1.0.6` | SSR, file routes, server functions |
| Router | `@solidjs/router` | `^0.14.1` | `0.14.3` | `Router` + `FileRoutes` integration |
| Bundler | `vinxi` | `^0.4.1` | `0.4.2` | SolidStart v1 build engine |
| Server engine | `nitro` | — (transitive) | `2.9.7` | Node-server preset, `.output/` artifact |
| Styling | `tailwindcss` | `^3.4.3` | installed | JIT, via PostCSS |
| Language | TypeScript | — | strict | `jsx: preserve`, `~/*` → `./src/*` |
| Runtime | Node | `>=18` (engines) | — | Dockerfile uses `node:20-slim` |

### 1.2 Repository layout

```
customer_support_frontend/            git repo root (main + develop)
├── package.json                      husky only (prepare hook)
├── .husky/pre-commit                 pre-commit hook (lint-staged)
└── customer_frontend/                ← the application analyzed here
    ├── app.config.ts                 vinxi/SolidStart configuration
    ├── Dockerfile                    multi-stage node:20-slim build
    ├── docker-compose.yaml           port 3000:3000
    ├── public/
    │   └── favicon.ico               only static asset
    ├── src/
    │   ├── entry-client.tsx          client bootstrap (hydration)
    │   ├── entry-server.tsx          SSR handler + HTML document shell
    │   ├── app.tsx                   root layout: Router + Nav + Suspense + FileRoutes
    │   ├── app.css                   Tailwind directives + theme variables
    │   ├── global.d.ts               references @solidjs/start/env types
    │   ├── components/
    │   │   ├── Counter.tsx           reactive counter (createSignal)
    │   │   └── Nav.tsx               navigation bar
    │   └── routes/
    │       ├── index.tsx             GET /          (Hello world)
    │       ├── about.tsx             GET /about
    │       └── [...404].tsx          catch-all       (Not Found)
    ├── .vinxi/                       dev-generated types (nitro routes/imports)
    └── .output/                      production build (nitro node-server)
        ├── nitro.json                preset: node-server, nitro 2.9.7
        └── server/index.mjs          standalone Node.js server entry
```

### 1.3 Runtime architecture: SSR handler and hydration

Two entry points define the isomorphic runtime split:

- **`src/entry-server.tsx`** exports a handler built by `createHandler` from
  `@solidjs/start/server`. Every HTTP request flows through it; `StartServer` renders
  the full HTML document and streams/serializes the initial reactive state.
- **`src/entry-client.tsx`** mounts `StartClient` into `<div id="app">`; it reads the
  serialized payload (via **seroval** — confirmed in the bundle imports) and hydrates
  the reactive graph over the server-rendered DOM.

The production artifact `.output/server/index.mjs` confirms the runtime chain —
it imports `seroval`, `seroval-plugins/web`, and `solid-js/web/storage`, i.e. the
serialization + hydration pipeline required for SSR.

### 1.4 Build pipeline: vinxi → Nitro

`app.config.ts` is the single build entry:

```ts
import { defineConfig } from "@solidjs/start/config";

export default defineConfig({});
```

`vinxi dev` runs a Vite dev server (HMR via the `// @refresh reload` pragma in both
entries) plus a dev Nitro server. `vinxi build` produces:

- a **client bundle** (route chunks, code-split per route),
- a **server bundle** in `.output/` — Nitro `node-server` preset
  (`nitro.json`: `"preset": "node-server"`, preview command `node ./server/index.mjs`).

`.vinxi/types/` contains generated Nitro type shims (`nitro-routes.d.ts`,
`nitro-imports.d.ts`, `nitro-config.d.ts`) referenced by `tsconfig.json`'s
`"types": ["vinxi/types/client"]` for type-safe server imports.

### 1.5 Request flow

```mermaid
flowchart LR
    B[Browser] -->|"GET /"| N[Nitro node-server<br/>.output/server/index.mjs]
    N -->|"incoming request"| H[createHandler<br/>@solidjs/start/server]
    H -->|"render document"| SH[StartServer HTML shell<br/>entry-server.tsx]
    SH -->|"match route"| FR[FileRoutes<br/>routes/*.tsx]
    FR -->|"execute route module"| R[Route component<br/>index.tsx / about.tsx / [...404].tsx]
    R -->|"serialize state"| P[HTML + seroval payload<br/>streamed response]
    P -->|"200 text/html"| B
    B -->|"load JS chunks + payload"| C[StartClient<br/>entry-client.tsx]
    C -->|"hydrate #app"| D[Reactive DOM<br/>signals re-attached to nodes]
```

### 1.6 Rendering model

SolidJS compiles JSX (via `babel-preset-solid`) into fine-grained reactive bindings —
there is **no virtual DOM** and no diffing. Reading a signal inside JSX subscribes a
per-binding effect; a write updates only the affected DOM nodes.

```mermaid
flowchart TD
    Sig[createSignal 0<br/>count / setCount] -->|"read: count()<br/>subscribe effect"| Eff[Reactive effect<br/>compiled from JSX]
    Eff -->|"update textContent"| DOM[DOM text node<br/>"Clicks: 0"]
    Click[user click] -->|"onClick"| Write[setCount n]
    Write -->|"write + notify subscribers"| Sig
    Write2[any write] -->|"only subscribed bindings re-run"| Eff
    Sig -.->|"no VDOM, no diff"| DOM
```

---

## 2. Layer analysis

### 2.1 Entry layer

**`src/entry-client.tsx`** — client bootstrap:

```tsx
// @refresh reload
import { mount, StartClient } from "@solidjs/start/client";

mount(() => <StartClient />, document.getElementById("app")!);
```

`mount` (from `solid-js/web`) performs client-side hydration over the existing
`#app` node rendered by the server. `StartClient` re-hydrates the router and
re-attaches the serialized reactive state. The `@refresh reload` pragma enables
vinxi hot-module reload for this file.

**`src/entry-server.tsx`** — SSR handler and document shell:

```tsx
// @refresh reload
import { createHandler, StartServer } from "@solidjs/start/server";

export default createHandler(() => (
  <StartServer
    document={({ assets, children, scripts }) => (
      <html lang="en">
        <head>
          <meta charset="utf-8" />
          <meta name="viewport" content="width=device-width, initial-scale=1" />
          <link rel="icon" href="/favicon.ico" />
          {assets}
        </head>
        <body>
          <div id="app">{children}</div>
          {scripts}
        </body>
      </html>
    )}
  />
));
```

Architectural role: the server handler. `createHandler` wraps the SolidStart SSR
pipeline; `StartServer` renders the app and injects `assets` (CSS links, route
preloads) and `scripts` (entry JS). The `document` prop owns the **HTML shell
contract** — currently minimal: charset, viewport, and the favicon served from
`public/favicon.ico`. There is **no per-route metadata** (`<title>`, description,
Open Graph) — no `@solidjs/meta` is installed.

**`src/global.d.ts`** — type shim:

```ts
/// <reference types="@solidjs/start/env" />
```

Pulls in SolidStart environment typings (server/client ambient types).

### 2.2 App shell

**`src/app.tsx`** — root layout:

```tsx
import { Router } from "@solidjs/router";
import { FileRoutes } from "@solidjs/start/router";
import { Suspense } from "solid-js";
import Nav from "~/components/Nav";
import "./app.css";

export default function App() {
  return (
    <Router
      root={props => (
        <>
          <Nav />
          <Suspense>{props.children}</Suspense>
        </>
      )}
    >
      <FileRoutes />
    </Router>
  );
}
```

The shell composes four responsibilities:

1. **`Router`** — client-side routing (history state, location signal).
2. **`root` layout** — the persistent chrome around route content; `Nav` renders
   once and survives navigation.
3. **`Suspense`** — async boundary for route code-splitting and future
   `createResource` data loading.
4. **`FileRoutes`** — registers every module under `routes/` as a route; the
   router tree is derived from the file system (this is the SolidStart v1
   routing contract — `@solidjs/start/router`).

### 2.3 Route layer

Three routes, each a default-exported component:

| File | Path | Component | Notes |
|---|---|---|---|
| `routes/index.tsx` | `/` | `Home` | Hello-world marketing page, embeds `Counter` |
| `routes/about.tsx` | `/about` | `About` | Duplicate of Home, embeds `Counter` |
| `routes/[...404].tsx` | `/*` | `NotFound` | Catch-all (file-system 404) |

`routes/index.tsx` in full:

```tsx
import { A } from "@solidjs/router";
import Counter from "~/components/Counter";

export default function Home() {
  return (
    <main class="text-center mx-auto text-gray-700 p-4">
      <h1 class="max-6-xs text-6xl text-sky-700 font-thin uppercase my-16">Hello world!</h1>
      <Counter />
      <p class="mt-8">
        Visit{" "}
        <a href="https://solidjs.com" target="_blank" class="text-sky-600 hover:underline">
          solidjs.com
        </a>{" "}
        to learn how to build Solid apps.
      </p>
      <p class="my-4">
        <span>Home</span>
        {" - "}
        <A href="/about" class="text-sky-600 hover:underline">
          About Page
        </A>{" "}
      </p>
    </main>
  );
}
```

Route modules are the unit of code-splitting (each becomes a lazy chunk) and are
SSR-rendered on first request. The `A` component (anchor) performs client-side
navigation; plain `<a>` links (as in `Nav.tsx`) cause full document round-trips.
The `[...404].tsx` catch-all demonstrates the SolidStart "splat" route convention —
any unmatched path renders `NotFound` and returns HTTP 200 unless server logic
sets a status.

### 2.4 Component layer

**`src/components/Counter.tsx`** — the only stateful component (see §3.1):

```tsx
import { createSignal } from "solid-js";

export default function Counter() {
  const [count, setCount] = createSignal(0);
  return (
    <button
      class="w-[200px] rounded-full bg-gray-100 border-2 border-gray-300 focus:border-gray-400 active:border-gray-400 px-[2rem] py-[1rem]"
      onClick={() => setCount(count() + 1)}
    >
      Clicks: {count()}
    </button>
  );
}
```

**`src/components/Nav.tsx`** — chrome component using router context:

```tsx
import { useLocation } from "@solidjs/router";

export default function Nav() {
  const location = useLocation();
  const active = (path: string) =>
    path == location.pathname ? "border-sky-600" : "border-transparent hover:border-sky-600";
  return (
    <nav class="bg-sky-800">
      <ul class="container flex items-center p-3 text-gray-200">
        <li class={`border-b-2 ${active("/")} mx-1.5 sm:mx-6`}>
          <a href="/">Home</a>
        </li>
        <li class={`border-b-2 ${active("/about")} mx-1.5 sm:mx-6`}>
          <a href="/about">About</a>
        </li>
      </ul>
    </nav>
  );
}
```

`useLocation()` returns a reactive `location` object; the `active()` helper reads
`location.pathname` inside the class binding, so the border highlight updates
reactively on navigation. Note the inconsistency: `Nav` uses raw `<a href>`
(full page reloads) while `routes/index.tsx` uses `<A href>` (client-side
navigation) — the router-aware anchor should be used everywhere.

### 2.5 Styling layer

- **`src/app.css`** — Tailwind directives (`@tailwind base/components/utilities`)
  plus CSS custom properties for light/dark theme via `prefers-color-scheme`.
- **`tailwind.config.cjs`** — content scan `./src/**/*.{html,js,jsx,ts,tsx}`,
  default theme, no plugins.
- **`postcss.config.cjs`** — `tailwindcss` + `autoprefixer` pipeline (build-time,
  processed by vinxi/Vite).

Utility-first classes are authored inline in JSX (including arbitrary values such
as `w-[200px]`, `max-6-xs` in templates — note `max-6-xs` is not a valid Tailwind
class and is ignored by JIT, a template artifact). There is no design-token or
component-class layer yet.

### 2.6 Configuration layer

| File | Role |
|---|---|
| `app.config.ts` | Vinxi/SolidStart config (empty — defaults) |
| `tsconfig.json` | Strict TS; `jsx: preserve` + `jsxImportSource: solid-js`; `~/*` → `./src/*`; types `vinxi/types/client` |
| `package.json` | Scripts; **name still `example-with-tailwindcss`** |
| `Dockerfile` | Multi-stage: build (`npm run build:docker` = test + build) → production copies `node_modules`, `.vinxi`, `.output`, full app; runs `vinxi start` |
| `docker-compose.yaml` | Exposes port `3000:3000` |

### 2.7 Deployment artifact (nitro)

`.output/nitro.json` records the production build:

```json
{
  "preset": "node-server",
  "framework": { "name": "nitro", "version": "2.9.7" },
  "commands": { "preview": "node ./server/index.mjs" }
}
```

The output is a self-contained Node server (`server/index.mjs` + chunks + vendored
`node_modules`), deployed via Docker. `vinxi start` boots it. This means the app can
be hosted anywhere Node 18+ runs — including Azure App Service/Container Apps
without a static-site-only constraint (SSR requires a Node runtime, not a CDN).

---

## 3. Data flow

### 3.1 Reactivity in `Counter.tsx`

```
createSignal(0)
    │  returns [getter, setter]
    ▼
const [count, setCount] = createSignal(0);
    │
    ├── read path:  JSX "Clicks: {count()}"
    │   → compiler creates a reactive binding (effect) for the text node
    │   → the effect subscribes to `count`
    ▼
onClick={() => setCount(count() + 1)}
    │
    ▼
setCount(n) writes the value and notifies subscribers
    → ONLY the text node binding re-executes ("Clicks: n")
    → no component re-render, no VDOM diff
```

Mechanics:

- `createSignal` returns a `[getter, setter]` tuple. The getter reads the value and
  records the current computation as a subscriber (fine-grained dependency graph).
- JSX interpolation `{count()}` is compiled into a `createRenderEffect`-style
  binding scoped to the exact text node.
- `setCount` triggers synchronous notification; the effect updates
  `nodeValue` in place. This is why Solid performs well for high-frequency
  updates (ticket lists, live status) — updates are O(affected bindings),
  not O(component tree).

### 3.2 Routing + Suspense integration

`FileRoutes` builds the route table from `routes/`. Each route module is a lazy
boundary: navigating to `/about` loads `about.tsx`'s chunk, and `Suspense` in the
root renders the fallback (nothing — no fallback prop, so a blank frame) while the
chunk resolves. Because routes render under a single `Suspense`, any future
`createResource` inside a route will also suspend here — this is the intended seam
for async data (see ADR-003).

`Nav` re-renders reactively: `useLocation()` exposes a location signal, and the
class binding recomputes only the class attribute of the `<li>` elements when the
path changes — no page reload for the highlight, but the `<a href>` navigation
itself does a full document load, which destroys the SPA continuity the router
provides.

---

## 4. Architecture assessment

### 4.1 Strengths

| Aspect | Assessment |
|---|---|
| SSR by default | Full server rendering out of the box — HTML arrives without JS, good baseline for SEO and perceived performance |
| Fine-grained reactivity | No VDOM diffing; ideal for chat/ticket-list UIs with high update frequency |
| File-based routing | Routes self-register; no central route table to drift; catch-all 404 included |
| Isomorphic runtime | Same components on server and client; hydration handled by SolidStart (seroval) |
| Build pipeline | vinxi → Nitro yields a deployable Node server; `.output/` is self-contained |
| TypeScript strict | `strict: true`, path alias `~/*`, generated Nitro types |
| Tooling skeleton | Husky pre-commit + lint-staged, Jest + ts-jest, Cypress (declared), Docker multi-stage, ESLint 9 + Prettier |

### 4.2 Gaps for a customer-support product

| # | Gap | Evidence | Impact |
|---|---|---|---|
| G1 | **Package identity** | `package.json` name is `example-with-tailwindcss` | Wrong artifact name propagates to Docker image labels, docs, future registry publication |
| G2 | **No data layer** | Zero `fetch`, `createResource`, API routes, or server functions in `src/` | Cannot load tickets/customers; SSR renders only static content |
| G3 | **No state management** | Only one local `createSignal`; no stores/context | Multi-component shared state (session, filters) impossible |
| G4 | **Broken nav semantics** | `Nav.tsx` uses `<a href>`; routes use `<A>` | Full page reloads per navigation; loss of client-side routing, extra SSR round-trips |
| G5 | **No SEO/meta layer** | Static HTML shell only; no `@solidjs/meta`, no per-route title/OG | Poor search/embedding previews for support articles |
| G6 | **No error handling** | No `ErrorBoundary`, no 404 status code logic, no route error fallback | Blank screens on data/route failures; wrong HTTP semantics |
| G7 | **Broken CI scripts** | `code:ci`/`test:docker` call undefined `test:coverage`; `docs::swagger` references missing `./swagger.js`; `db:sync` calls uninstalled `prisma` | CI gate and auxiliary commands fail or no-op |
| G8 | **Lint config not loadable** | File named `eslintrc.config.js` (ESLint 9 auto-detects `eslint.config.js`, flat config) | `npm run lint:fix` cannot find config; lint gate ineffective |
| G9 | **Jest cannot compile TSX** | `jsx: "preserve"` + ts-jest without Babel transform | Any `.tsx` unit test fails to compile; current green suite is `--passWithNoTests` |
| G10 | **Cypress unused** | Installed ^13.14.2, no `cypress/` dir, no config, no specs | Dead dependency; no E2E evidence for the SSR+hydration flow |
| G11 | **No environment handling** | No `.env` usage, no env validation, no config module | Deployment params (API base URL, auth) have nowhere to live |
| G12 | **No caching strategy** | No `cache`/`query`, no `Cache-Control` headers | Repeat visits re-render everything server-side |
| G13 | **Docker image bloat** | Copies `node_modules`, `.vinxi`, `.output`, and full source into production | Large images, slow pulls, source leakage into image |
| G14 | **Security posture** | No CSP, no security headers (`helmet`-style), no rate limiting, no auth boundary | Baseline exposure once real endpoints exist |

---

## 5. ADR-style recommendations

All ADRs are **Proposed** — they define the direction for the next implementation
increment, not changes to the current baseline.

### ADR-001 — State management with Solid primitives (no external store)

- **Status:** Proposed
- **Context:** The app needs shared client state (current user, active ticket,
  filters). Redux/Zustand-style external stores duplicate Solid's reactivity.
- **Decision:** Use Solid primitives only: `createSignal`/`createStore` for local
  state, `createContext` for dependency injection of shared stores, `createMemo`
  for derived values. Revisit only if cross-window sync or time-travel debugging
  becomes a hard requirement.
- **Consequences:**
  - (+) Zero dependencies; fine-grained updates for free; idiomatic Solid.
  - (−) No devtools timeline out of the box; team must learn Solid mental model
    (effects over selectors).

### ADR-002 — Typed API client layer (fetch wrapper first, Solid Query later)

- **Status:** Proposed
- **Context:** No API calls exist. A customer-support UI needs a consistent,
  typed, cancellable HTTP layer with error normalization.
- **Decision:** Build `src/lib/api/` as a thin typed `fetch` client (base URL from
  validated env, JSON errors normalized to a `ApiError` type, `AbortSignal`
  support). Adopt `@tanstack/solid-query` only when server-state caching
  semantics (refetch, invalidation, optimistic updates) are actually needed.
- **Consequences:**
  - (+) Full control, minimal bundle, easy to test with mocked `fetch`.
  - (−) Manual cache/invalidation logic if adopted later; migration cost to
    Solid Query exists but is small at this stage.

### ADR-003 — Data fetching with `createResource` + `Suspense`

- **Status:** Proposed
- **Context:** Route modules will fetch tickets/messages; SSR must render with
  data and hydration must not duplicate requests.
- **Decision:** Use `createResource` per route (keyed, `SSRLoadEvent`-aware via
  SolidStart's serialization) inside `Suspense` boundaries; add explicit
  `fallback` skeletons. For cross-route dedup, use SolidStart's `query`/`cache`
  primitives. Never fetch in effects for initial load.
- **Consequences:**
  - (+) SSR renders complete HTML; payload hydration reuses server data (no
    flash/fetch storm); declarative loading states.
  - (−) Requires discipline: resources must be keyed correctly; streaming
    semantics need testing on the target host.

### ADR-004 — Feature-based folder structure

- **Status:** Proposed
- **Context:** Current flat `routes/ + components/` works for 3 files; a support
  product (tickets, customers, chat, admin) needs bounded ownership.
- **Decision:**

  ```
  src/
  ├── routes/                  # thin route shells only
  ├── features/
  │   ├── tickets/             # components/, api/, queries/, stores/
  │   ├── customers/
  │   └── auth/
  ├── shared/                  # ui/ (primitives), lib/ (api client), types/
  └── app/                     # app.tsx, entries, layout chrome
  ```

  Routes import from `features/`; features never import routes; `shared/` has no
  feature imports (dependency rule enforced in review, later via ESLint
  boundaries plugin).
- **Consequences:**
  - (+) Scalability, clear ownership, easier code-splitting and test colocation.
  - (−) Initial refactor cost; over-structuring risk if features stay tiny.

### ADR-005 — Package identity, repo structure, and monorepo strategy

- **Status:** Proposed
- **Context:** The package is still `example-with-tailwindcss`; the repo root
  holds only husky; the backend is expected to join the repository.
- **Decision:**
  1. Rename package to `@customer-support/frontend` (or `customer-frontend`).
  2. Keep the current repo layout for now, but fix the broken gates: define
     `test:coverage`, replace `eslintrc.config.js` with a real flat
     `eslint.config.js`, remove or implement `swagger.js`/`prisma` scripts,
     configure Cypress with one smoke spec.
  3. When a backend package lands, migrate to pnpm workspaces with
     `apps/frontend` + `packages/*` (shared API types) — do not build a custom
     monorepo mechanism before the second package exists (YAGNI).
- **Consequences:**
  - (+) Correct identity; CI actually gates; migration path ready when backend
    arrives.
  - (−) Script fixes are plumbing work with no visible feature output; pnpm
    migration touches lockfile and CI.

---

## 6. Quick wins (non-ADR, high leverage)

1. `Nav.tsx`: replace `<a href>` with `<A href>` from `@solidjs/router` (client-side nav).
2. Add `@solidjs/meta` and per-route `<Title>`/`<Meta>` (SEO, G5).
3. Wrap the router root in `<ErrorBoundary>` with a friendly fallback (G6).
4. Fix G7/G8/G9: define `test:coverage`, rename lint config to flat `eslint.config.js`, add a babel/jest transform for TSX, delete unused dev deps or add the missing specs.
5. Add an `ErrorBoundary`-safe 404 status: set `setResponseStatus(404)` from `@solidjs/start/server` in a server-only guard.
6. Docker: prune to `.output` + `node_modules` only (G13) and add `HEALTHCHECK` against `/`.

---

## 7. Appendix — evidence

- Versions resolved from `node_modules/*/package.json`: `solid-js 1.8.22`,
  `@solidjs/start 1.0.6`, `@solidjs/router 0.14.3`, `vinxi 0.4.2`.
- `.output/nitro.json`: nitro `2.9.7`, preset `node-server`, build date
  `2024-09-13`.
- `.output/server/index.mjs` imports: `seroval`, `seroval-plugins/web`,
  `solid-js/web/storage` (hydration serialization pipeline).
- `public/` contains only `favicon.ico`; referenced by
  `entry-server.tsx` as `/favicon.ico`.
- Git: current branch `develop`; remote `origin git@github.com:Samuel-Ricardo/customer_support_frontend.git`; branches `main`, `develop`.
- No `docs/` directory existed before this document; no source files were modified.
