# 02 — Codebase Analysis: `customer_frontend`

| | |
|---|---|
| **Date** | 2026-08-19 |
| **Scope** | `customer_frontend/` — SolidStart app (Solid 1.8.18, `@solidjs/start` 1.0.6, `@solidjs/router` 0.14.3 installed, Vinxi 0.4.2 installed, Tailwind 3.4.3) |
| **Method** | Static read of all source files and configs + read-only runtime probes (`eslint`, `jest`, `prettier` CLI) + inspection of installed package sources (`node_modules/@solidjs/router/dist`) |
| **Source files modified** | none |
| **Environment** | Windows, Node v25.4.0, npm 11.7.0; repo `develop` @ `73c0a6f` (21 commits) |

> **Verification note**: Context7 API quota was exhausted during this session, so router API claims were verified directly against the installed
> `@solidjs/router@0.14.3` distribution source (`node_modules/@solidjs/router/dist/...`). This is stronger evidence than docs: it is the exact code the app executes.

---

## 1. Executive summary

The app is a functional hello-world SolidStart template: correct SSR shell, correct FileRoutes wiring, reactive components. There is **zero product code and zero tests**. The toolchain (lint, format, test, CI, Docker, git hooks) is **non-functional end-to-end**, and several scripts reference artifacts that do not exist.

Top findings (details in §4):

| # | Finding | Severity |
|---|---------|----------|
| F1 | ESLint 9.10.0 cannot load any config — `eslintrc.config.js` is not a recognized config filename; `@typescript-eslint/*` packages are not installed | **P0** |
| F2 | Prettier silently runs with defaults — `prettierrc.js` (no leading dot) is not detected | **P0** |
| F3 | Lint/format script globs `./src/**/*.{ts,js}` match **only** `global.d.ts` — every `.tsx` source is exempt | **P0** |
| F4 | Jest is broken at install level — `node_modules/jest-cli` has no `build/` dir; `npx jest` crashes before loading config | **P0** |
| F5 | References to missing artifacts: `docs::swagger` → `./swagger.js` (absent), `db:sync` → `prisma` (not installed), `test:coverage` → script never defined | **P0** |
| F6 | Husky pre-commit chain is broken: hook runs `npx lint-staged` at repo root; config lives in `customer_frontend/.lintstagedrc.json`; commands would execute against the root package, which has none of those scripts | **P0** |
| F7 | `Nav.tsx` uses plain `<a>` instead of router `<A>` — **not** a full-page-reload bug under current defaults (verified), but a real a11y defect (no `aria-current`), pre-hydration full loads, and a latent reload bug | **P1** |
| F8 | No `<title>` anywhere (SSR shell + routes); nav active link has no `aria-current`; external link missing `rel="noopener"` | **P1** |
| F9 | Dockerfile: no `.dockerignore`, copies entire build stage, ships `node_modules` with devDeps, `build:docker` depends on broken Jest | **P1** |
| F10 | Package name still `example-with-tailwindcss`; config cruft (decorators flags, dead template boilerplate, `max-6-xs` invalid Tailwind class) | **P2** |

---

## 2. Repository layout and git history

The monorepo roots are nested: **two independent npm projects**.

```
customer_support_frontend/          <- git root (21 commits, branch develop)
├── package.json                    <- root: ONLY husky devDep + "prepare": "husky"
├── package-lock.json
├── .husky/pre-commit               <- "npx lint-staged" (runs at repo ROOT)
├── README.md                       <- single line: "# customer_support_frontend"
├── LICENSE
└── customer_frontend/              <- the actual app (all code below)
    ├── package.json                <- app deps/scripts
    ├── src/ ...
    ├── Dockerfile, docker-compose.yaml
    └── *.config.* / *.js configs
```

- Branches: `main`, `develop` (HEAD), `origin/main`, `origin/develop`. No feature branches in history.
- 21 commits, gitmoji-style messages, e.g. `[ :seedling: ] init: project :D`, `[ :construction_worker: ] | create: Dockerfile (docker::image)`. The custom `[ :emoji: ] | <verb>: <subject> (<scope>)` format is internally consistent but diverges from standard gitmoji (`:emoji: subject`).
- **Docker history**: commit `49c201c` **A**dds `customer_frontend/Dockerfile` (32 lines); commit `73c0a6f` with the identical message "create: Dockerfile" **M**odifies it (2 insertions, 1 deletion: the `COPY .../build ./build` line replaced by `.vinxi` + `.output` copies). So: one file, two "create" commits — history noise, not two files. The build output copy in the original commit (`/home/node/app/build ./build`) was wrong for Vinxi (Vinxi emits `.output`, not `build`); the follow-up fixed it.
- Root `package-lock.json` is checked in (husky dep tree); `customer_frontend/package-lock.json` is checked in too. Nested package roots confuse tooling (see F6) — a workspace layout or moving husky into the app would fix it.

---

## 3. File-by-file analysis

### 3.1 `src/entry-client.tsx` (4 lines)

```tsx
// @refresh reload
import { mount, StartClient } from "@solidjs/start/client";

mount(() => <StartClient />, document.getElementById("app")!);
```

- **Purpose**: hydrates the SSR shell on the client. Correct per SolidStart v1 contract. `@refresh reload` directive is the template's HMR marker.
- **Quality**: fine. The non-null assertion (`!`) on `getElementById` is the framework-invariant guarantee (`entry-server.tsx` always renders `<div id="app">`) — acceptable; a `document.getElementById("app") ?? throw` would be marginally more defensive but adds no value here.

### 3.2 `src/entry-server.tsx` (19 lines)

```tsx
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
```

- **Purpose**: SSR shell; correct use of `createHandler`/`StartServer`.
- **Issues**:
  - **No `<title>`** (`entry-server.tsx:7-12`). Nothing in the routes adds one either → browser tab shows the document URL. Empty `<head>` beyond viewport/favicon/assets. Use SolidStart head components (`<Title>` from `@solidjs/meta` or Solid's head primitives) per route.
  - `lang="en"` (`entry-server.tsx:5`) hard-coded — fine today, will need i18n plumbing later.
  - `charset`/`viewport` correct; favicon wired to `public/favicon.ico` (exists).

### 3.3 `src/app.tsx` (19 lines)

```tsx
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
```

- **Purpose**: app root — Router + file-based routes + global Suspense. This is the canonical SolidStart v1 composition (`Router` from `@solidjs/router`, `FileRoutes` from `@solidjs/start/router`).
- **Quality**: correct. Notes:
  - `<Suspense>` has no `fallback` → suspended route payloads render nothing until resolved (acceptable for tiny app; consider a skeleton).
  - No error boundary (`ERROR_BOUNDARY` is available in SolidStart for route-level error handling) — not needed at demo scale.
  - `Router` receives no `explicitLinks` prop → default `false` (consequence for `Nav` analyzed in F7).

### 3.4 `src/global.d.ts` (1 line)

```ts
/// <reference types="@solidjs/start/env" />
```

Correct — pulls env typings (import.meta.env, server/client contexts) into the TS program.

### 3.5 `src/app.css` (20 lines)

```css
@tailwind base;   @tailwind components;   @tailwind utilities;
:root { --background-rgb: 214,219,220; --foreground-rgb: 0,0,0; }
@media (prefers-color-scheme: dark) { :root { --background-rgb: 0,0,0; --foreground-rgb: 255,255,255; } }
body { background: rgb(var(--background-rgb)); color: rgb(var(--foreground-rgb)); }
```

- Tailwind v3 directives are the correct syntax for v3.4 (the `@apply`-era directives changed in v4 — no action here).
- Dark mode handled via `prefers-color-scheme` CSS vars — works, but it bypasses Tailwind's `dark:` variant system entirely (class-based by default). If `dark:` classes are ever wanted, this must be revisited (`tailwind.config.cjs` has no `darkMode` key).
- Minor: light-on-dark FOUC risk is nil (vars are static CSS); fine.

### 3.6 `src/components/Nav.tsx` (20 lines) — **F7**

```tsx
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

**Purpose**: global navigation with active-route underline. `useLocation()` is reactive, so the class strings update on navigation — this part is correct Solid.

**Issues**:

| Ref | Issue |
|-----|-------|
| `Nav.tsx:10`, `Nav.tsx:14` | Plain `<a>` instead of router `<A>`. **Verified impact below.** |
| `Nav.tsx:5` | Loose equality `==` — use `===`. |
| `Nav.tsx:4-8` | Exact-match active logic: `/` only matches `/` exactly, `/about` only `/about`. Consistent with current routes, but future subroutes (`/about/team`) will silently lose the active state. `<A>`'s default descendant matching has the opposite problem for `/` (root matches everything; needs the `end` prop — see README of installed router, line 744). |
| nav element | `<nav>` **is** a landmark (good) but has no `aria-label`, and the active link carries no `aria-current="page"` — screen readers cannot distinguish the current section. |
| — | Duplicates functionality `<A>` provides for free: base-path resolution, active/inactive classes, preload, `aria-current`. |

**`<a>` vs `<A>` — verified behavior, corrects the "full page reload" assumption**:

The router's delegated click handler is `handleAnchor` (`node_modules/@solidjs/router/dist/index.js:1204-1217`):

```js
function handleAnchor(evt) {
  if (evt.defaultPrevented || evt.button !== 0 || evt.metaKey || evt.altKey || evt.ctrlKey || evt.shiftKey) return;
  const a = evt.composedPath().find(el => ...nodeName === "A");
  if (!a || explicitLinks && !a.hasAttribute("link")) return;   // <-- line 1207
  ...
  evt.preventDefault();                                          // <-- line 1224
  navigateFromRoute(to, { resolve: false, replace: ... });
}
```

`explicitLinks` defaults to **`false`** (installed README, line 738: *"Disables all anchors being intercepted and instead requires `<A>`. Default: `false`"*), and `app.tsx` doesn't set it. Therefore, **with the current configuration, a left-click on the plain `<a href="/about">` is intercepted and navigated client-side — no full page reload happens** in a hydrated browser.

The real, evidence-based defects:

1. **Pre-hydration / no-JS full document loads.** Before `StartClient` hydrates (or without JS), nothing intercepts the click — the browser performs a full server round-trip. With `<A>` + `link` attribute the behavior is identical pre-hydration (the `link` attribute only matters to the JS handler), but the difference is SSR/SEO-visible: the served HTML exposes plain anchors for nav links either way. The consistent pattern matters for the shell, not just the links.
2. **No `aria-current="page"`.** `<A>` renders it automatically (`components.jsx:30`):
   ```jsx
   }} link aria-current={isActive()[1] ? "page" : undefined}/>
   ```
   The hand-rolled `active()` in Nav returns CSS classes only.
3. **Latent reload bug.** The moment anyone enables `explicitLinks: true` (the documented, stricter mode; the router README also recommends `<A>`), every plain `<a>` in Nav triggers line 1207's early return → browser default → **full page reload**. The app also mixes conventions: `index.tsx:13`, `about.tsx:12`, `[...404].tsx:9-14` already use `<A>`; Nav is the only holdout.
4. **Hard-coded hrefs.** `<A>` resolves hrefs through `useResolvedPath`/`useHref` (router base path). If the app is ever served under a base path, Nav's `/` and `/about` break; the rest of the app keeps working.
5. **No preloading.** `<A href>` participates in the router's link preload pass; plain anchors don't.

Recommendation: migrate Nav to `<A>` with explicit active/inactive classes:

```tsx
<A href="/" end class="border-b-2 mx-1.5 sm:mx-6"
   activeClass="border-sky-600"
   inactiveClass="border-transparent hover:border-sky-600">Home</A>
```

(`end` is required for `/` — otherwise the root link matches every route, per the router's descendant matching.) This removes `useLocation`/`active()` entirely and restores semantics parity with the rest of the routes.

### 3.7 `src/components/Counter.tsx` (11 lines)

```tsx
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

- **Purpose**: demo of fine-grained reactivity.
- **Quality**: idiomatic Solid — `createSignal`, reads inside JSX are reactive, `class` prop (Solid convention, not `className`). Notes:
  - `onClick={() => setCount(count() + 1)}` — correct here (single click events), but `setCount(c => c + 1)` (functional update) is the more robust form and immune to stale-closure patterns if the handler is ever shared/debounced.
  - Trailing whitespace after `py-[1rem]"` (`Counter.tsx:5`) — Prettier would fix it if it were wired up (it isn't; F3).
  - Accessible name derives from text content (`Clicks: N`) — acceptable; explicit `aria-live="polite"` would announce increments to screen readers; minor.
  - Arbitrary-value utilities (`w-[200px]`, `px-[2rem]`) are fine in Tailwind but are exactly the arbitrary values that belong in the theme (`theme.extend.spacing`) if they recur.

### 3.8 `src/routes/index.tsx` (20 lines)

```tsx
<h1 class="max-6-xs text-6xl text-sky-700 font-thin uppercase my-16">Hello world!</h1>
...
<a href="https://solidjs.com" target="_blank" class="text-sky-600 hover:underline">solidjs.com</a>
```

- **Purpose**: home route.
- `class="max-6-xs"` (`index.tsx:4`) is **not a valid Tailwind class** (typo from the upstream template — should be `max-w-6xl` or removed); Tailwind silently generates nothing. Dead class.
- External link `target="_blank"` without `rel="noopener noreferrer"` (`index.tsx:6-9`) — tabnabbing risk pattern; the reactive `new URL` origin check in `handleAnchor` doesn't apply here because `target` bails at line 1211, so the browser opens a fresh tab with `window.opener` access.
- `✔` uses `<A href="/about">` (`index.tsx:13`) — client-side nav, correct.
- Template boilerplate ("Visit solidjs.com ...") duplicated verbatim across `index.tsx`, `about.tsx`, `[...404].tsx` — candidate for a shared `Layout`/`Footer` component (P2 DRY).

### 3.9 `src/routes/about.tsx` (20 lines)

- Same boilerplate as index; `<A href="/">` correct. No issues beyond duplication.

### 3.10 `src/routes/[...404].tsx` (17 lines)

- **Correct FileRoutes catch-all naming** — matches every unmatched path. Renders "Not Found" with working `<A>` links. (Caveat: no `createAsync`/route data involved; the 404 route is static, which is fine.)

### 3.11 `package.json` (46 lines)

| Key | Value | Assessment |
|-----|-------|------------|
| `name` (`:2`) | `example-with-tailwindcss` | F10 — template name; wrong for this product. |
| `docs::swagger` (`:5`) | `node ./swagger.js` | F5 — file does not exist anywhere in the repo. |
| `build` (`:6`) | `vinxi build` | OK (produces `.output`). |
| `build:docker` (`:7`) | `npm run test && npm run build` | F4/F5 — `test` crashes (broken jest-cli), so Docker builds fail today. |
| `lint:fix` (`:11`) | `eslint ./src/**/*.{ts,js} --fix` | F1+F3 — no valid config; glob misses `.tsx`. |
| `lint:staged` (`:12`) | `eslint --fix;` | F3 — trailing `;` is a Bash-ism (breaks on cmd.exe), no file args → lints whole tree, and `--fix` with no config errors. |
| `format:fix` (`:13`) | `prettier --check --write ...` | F3 — glob misses `.tsx` (only `global.d.ts` matches); `--check`+`--write` combo verified to work but neutralizes `--check`'s failure contract (exits 0 after fixing). |
| `format:check` (`:14`) | same glob, `--check` | F3 — again only `global.d.ts`. |
| `test` (`:15`) | `jest --passWithNoTests` | F4 — crashes at module load. |
| `test:staged` (`:16`) | `jest --passWithNoTests --findRelatedTests` | OK design; unreachable (jest broken) and never passed file args by lint-staged anyway. |
| `test:docker` (`:18`) | `npm run test:coverage && npm run test:dev` | F5 — `test:coverage` undefined; `test:dev` is `--watchAll` → **hangs** in a container. |
| `code:ci` (`:19`) | `format:fix && lint:fix && test:coverage` | F3+F5 — dead formats/lint + missing script. CI gate is red by construction. |
| `db:sync` (`:20`) | `prisma generate && prisma db push` | F5 — prisma not in dependencies (verified: no `node_modules/prisma`, no lockfile entry); no `prisma/schema.prisma` exists. |
| `husky:setup` (`:21`) | `cd ../ && husky init` | works only as a manual bootstrap from the app dir; root package already has husky+prepare. Confusing. |
| deps (`:23-31`) | router/start/solid/vinxi/tailwind/postcss/autoprefixer | `autoprefixer`/`postcss` are build-time tools — they belong in `devDependencies` (minor). |
| devDeps (`:35-45`) | jest/ts-jest/eslint/prettier/lint-staged/cypress etc. | Missing: `@typescript-eslint/*`, `jest-environment-jsdom`, `@solidjs/testing-library`, `@testing-library/dom`, `@types/node`. `cypress` installed (`13.14.2`) with **no `cypress.config.*` and no specs** — dead dependency. |

### 3.12 `tsconfig.json` (21 lines)

- `strict: true`, `moduleResolution: "bundler"`, `jsxImportSource: "solid-js"`, `noEmit`, `isolatedModules`, `paths: {"~/*": ["./src/*"]}` — all correct for SolidStart.
- `emitDecoratorMetadata`/`experimentalDecorators` (`:3-4`) — inherited from the official SolidStart template; nothing in the codebase uses decorators. Harmless flag noise (P2).
- `types: ["vinxi/types/client"]` — template-standard; server-side context is typed via `global.d.ts` reference. OK.
- No `skipLibCheck` — template-standard; can speed typecheck later (P2).
- `allowJs: true` with zero JS files — harmless.

### 3.13 `app.config.ts` (3 lines)

```ts
export default defineConfig({});
```

Empty SolidStart config. Correct entry; nothing to configure yet. When SSR middleware/routes/BFF calls arrive, this is where they land.

### 3.14 `tailwind.config.cjs` / `postcss.config.cjs`

- `content: ["./src/**/*.{html,js,jsx,ts,tsx}"]` — correct (and does cover `.tsx`, unlike the npm scripts).
- PostCSS chain `tailwindcss` + `autoprefixer` — correct for Tailwind v3. Missing trailing semicolon in `postcss.config.cjs:6` — cosmetic.

### 3.15 `jest.config.ts` (9 lines) — **F4/F7-config**

```ts
// jest.config.js
module.exports = {
  preset: "ts-jest",
  testEnvironment: "node",
  moduleFileExtensions: [...],
  collectCoverage: true,
  coverageDirectory: "coverage",
  coverageProvider: "v8",
};
```

- Stale filename in the header comment (`jest.config.js`).
- `module.exports` in a `.ts` file while `package.json` has `"type": "module"` — Jest loads it via ts-node (installed), which works in practice for Jest 29, but CJS-style config inside an ESM package is fragile; `jest.config.mjs` with `export default` is the durable form.
- `testEnvironment: "node"` — **wrong for component tests** (no DOM). Component tests need `jest-environment-jsdom` (not installed) and `@solidjs/testing-library` (not installed). Node env is only appropriate for pure-logic tests.
- `collectCoverage: true` with **no `coverageThreshold`** — coverage is collected but never enforced, so the "80% minimum" gate (Bradesco standard) is absent.
- No `moduleNameMapper` for `~/*` → tests importing `~/components/...` will fail resolution.
- No `setupFilesAfterEnv` (no jest-dom-extended assertions, no cleanup hooks).
- **Runtime status: dead.** `npx jest --listTests` fails before config parsing (see §8).

### 3.16 `eslintrc.config.js` (34 lines) — **F1**

```js
module.exports = {
  env: { browser: true, es2021: true, jest: true },
  extends: ["eslint:recommended", "plugin:@typescript-eslint/recommended", "prettier"],
  parser: "@typescript-eslint/parser",
  parserOptions: { ecmaVersion: "latest", sourceType: "module", project: ["./tsconfig.json"] },
  plugins: ["@typescript-eslint"],
  rules: { "@typescript-eslint/no-unused-vars": "warn", "@typescript-eslint/no-explicit-any": "off" },
};
```

Problems, deadliest first:

1. **Filename is not loadable by any ESLint version.** ESLint 9 flat config only auto-loads `eslint.config.js|mjs|cjs`; legacy ESLint 8 only auto-loads `.eslintrc.*`. `eslintrc.config.js` matches neither → runtime error reproduced in §8. ESLint 9.10.0 is installed; eslintrc support was removed in v9.
2. **Legacy syntax in the flat-config era.** Even renamed to `eslint.config.js`, `env`/`extends`/`overrides`/`plugins`-as-strings are flat-config-invalid; `env` must become `languageOptions.globals` (e.g. `globals.browser`, `globals.jest`), `extends` becomes config spreading.
3. **Missing dependencies.** `@typescript-eslint/parser` and `@typescript-eslint/eslint-plugin` are referenced (`:9,:23,:29`) but **not installed** — verified: no `node_modules/@typescript-eslint`, zero lockfile entries. (The modern meta-package is `typescript-eslint`, which bundles both.)
4. `parserOptions.project` enables typed linting — slow, needs `tsconfigRootDir`; overkill for this codebase.
5. `overrides` targets `.eslintrc.{js,cjs}` (`:17`) — self-referential legacy noise.
6. `no-explicit-any: off` — acceptable pragmatic choice for a small app, but the codebase has zero `any` anyway; consider keeping it on.

**Observed behavior** (§8): `npx eslint src/app.tsx` → *"ESLint couldn't find an eslint.config.(js|mjs|cjs) file."* The linter does nothing.

### 3.17 `prettierrc.js` (6 lines) — **F2**

```js
module.exports = { semi: true, singleQuote: true, trailingComma: "all", tabWidth: 2 };
```

- **Filename wrong**: Prettier looks for `.prettierrc*` (dot-prefixed) or `prettier.config.js`. `prettierrc.js` is ignored → **Prettier runs with defaults**. Verified: `npx prettier --find-config-path src/app.tsx` → *"Can not find configure file"*.
- Practical drift: default Prettier `trailingComma: "es5"` (config wants `"all"`), `singleQuote: false` (config wants true). The codebase currently uses single quotes throughout — one day someone runs Prettier and gets a 100-file diff, or never runs it (today's reality).
- Also doesn't cover `.tsx`, `.css`, `.json` — see F3.

### 3.18 `.lintstagedrc.json` (7 lines)

```json
{
  "*.{ts,tsx}": ["npm run format:fix", "npm run lint:staged", "npm run test:staged"]
}
```

- Invokes **repo-wide** scripts instead of scoping to staged files — each commit would reformat/lint the *entire* tree (slow, and semantically wrong for lint-staged).
- The scripts it calls are broken (F1/F3) and run against the **root package** when invoked from the git root (F6).
- Correct shape: inline commands with `-- ` file passing, e.g. `["eslint --fix", "prettier --write", "jest --passWithNoTests --findRelatedTests"]` — see §7.

### 3.19 `Dockerfile` (33 lines) / `docker-compose.yaml` (6 lines)

```dockerfile
FROM node:20-slim as build
USER node
WORKDIR /home/node/app
COPY --chown=node:node package*.json ./
RUN npm ci
COPY --chown=node:node . .
RUN npm run build:docker
...
FROM node:20-slim as production
...
COPY --from=build /home/node/app/node_modules ./node_modules
COPY --from=build /home/node/app/.vinxi ./.vinxi
COPY --from=build /home/node/app/.output ./.output
COPY --from=build /home/node/app/ ./          # <- duplicates everything again
CMD [ "npm", "run", "start:docker" ]
```

Issues:

- **No `.dockerignore`** (verified absent) → build context ships `node_modules`, `.git` (git root is one level up — context is `customer_frontend`, so `.git` is out of context; but `node_modules` ~hundreds of MB, `coverage`, `dist` are in). This slows every build and can leak local `.env*` files (`.gitignore` ignores them, but `.dockerignore` is the correct gate).
- **Final `COPY --from=build ./ ./` after already copying `node_modules`, `.vinxi`, `.output`** — wasteful duplicate layers; should copy `.output` + `package.json` only (or the whole dir *instead of* the specific copies).
- **Production image ships devDependencies** — `npm ci` installs everything (no `--omit=dev`); the `production` stage carries jest/cypress/eslint in the image.
- `npm run build:docker` = `test && build` — Jest is broken (F4) → **`docker compose build` fails today**.
- Comment cruft: commented `apt-get openssl` lines (`:3,:19`), commented `CMD` alternatives (`:14-15`) — leftovers from debugging.
- `docker-compose.yaml`: minimal and correct (build + port 3000). No healthcheck/restart policy — fine for dev.

---

## 4. Findings register

### P0 — blocking (dev/CI/tooling is non-functional)

| # | Finding | Evidence | Impact |
|---|---------|----------|--------|
| **F1** | ESLint has no loadable config; `@typescript-eslint/*` not installed | `eslintrc.config.js` name matches no loader; ESLint 9.10.0 runtime error (see §8); `node_modules/@typescript-eslint` absent | Lint gate dead; `npm run lint:fix` = `lint:staged` = `code:ci` fail; no type-aware linting |
| **F2** | Prettier config ignored | `prettierrc.js` (no dot) is not a recognized config filename; `--find-config-path` → error (see §8) | Formatting silently uses defaults — repo style (singleQuote) and config (`trailingComma: all`) diverge; future format runs yield huge diffs |
| **F3** | Lint/format globs miss `.tsx` | scripts `:11-14` use `./src/**/*.{ts,js}` — only `global.d.ts` matches | All 8 `.tsx` source files are exempt from lint AND format |
| **F4** | Jest install is broken | `node_modules/jest-cli` has no `build/` (only `bin`, `node_modules`, `package.json`...); `npx jest --listTests` → `Cannot find module 'jest-cli/build/index.js'` | No test runner at all; `build:docker`/`code:ci`/pre-commit chain all fail |
| **F5** | Scripts reference nonexistent artifacts | `docs::swagger` → `./swagger.js` (glob: 0 matches repo-wide); `db:sync` → prisma (not installed, no lock entry, no schema); `test:docker`/`code:ci` → `test:coverage` undefined; `test:dev` = `--watchAll` hangs in containers | 4 scripts are guaranteed failures; CI gate (`code:ci`) cannot pass |
| **F6** | Husky pre-commit broken by repo layout | `.husky/pre-commit` = `npx lint-staged` runs with cwd = **git root**; lint-staged config only exists in `customer_frontend/.lintstagedrc.json`; even if found, its `npm run ...` commands don't exist in root `package.json` | Every commit bypasses quality gates silently (or blocks on config-not-found) |

### P1 — correctness/quality/a11y

| # | Finding | Evidence | Impact |
|---|---------|----------|--------|
| **F7** | `Nav.tsx` uses `<a>` instead of `<A>` | Router `dist/index.js:1207,1224`; `components.jsx:30`; README:738 | No `aria-current="page"` (a11y); pre-hydration/no-JS clicks do full page loads; latent **full-reload bug** if `explicitLinks: true` is ever set; inconsistent with `<A>` usage in all routes; hard-coded hrefs break under a router base path |
| **F8** | Head/a11y gaps | `entry-server.tsx:7-12` (no `<title>`); Nav (no `aria-label`, no `aria-current`); `index.tsx:6-9` `target="_blank"` without `rel="noopener noreferrer"` | Tagged pages show URL as title; screen readers can't identify current page; tabnabbing risk pattern |
| **F9** | Dockerfile hygiene | no `.dockerignore`; duplicate `COPY --from=build ./ ./`; devDeps shipped; builds depend on broken Jest | Slow builds, large images, broken `docker compose build`; `.env*` risk without dockerignore |

### P2 — polish/maintainability

| # | Finding |
|---|---------|
| **F10** | `package.json` name still `example-with-tailwindcss` |
| **F11** | Invalid Tailwind class `max-6-xs` (`index.tsx:4`); trailing whitespace (`Counter.tsx:5`); `postcss.config.cjs:6` missing semicolon |
| **F12** | Template boilerplate duplicated across the 3 routes (DRY: extract shared layout/footer) |
| **F13** | Duplicate "create: Dockerfile" commit messages (`49c201c` A, `73c0a6f` M — history noise); root README empty |
| **F14** | tsconfig decorator flags unused (template-inherited); no `skipLibCheck`; `allowJs` unused |
| **F15** | `coverage/` not in `.gitignore` (`.gitignore:1-28`) — Jest's first successful run will dirty the tree; also missing `.eslintcache`, `.prettiercache` |
| **F16** | Version drift: local Node v25.4.0 vs `engines`: `>=18` vs Docker `node:20-slim`; no `.nvmrc`/`volta` pin; `@solidjs/router` resolved 0.14.3 (declared ^0.14.1), vinxi 0.4.2 (declared ^0.4.1) — lockfile drift is fine, but pin exact versions for CI reproducibility |

---

## 5. Code quality assessment

**TypeScript (strict)** — strong: every source file is typed, no `any`, no implicit-any leaks (only one `!` non-null assertion in `entry-client.tsx`, justified). `strict: true` + `isolatedModules` are active. No type errors are expected from `tsc --noEmit` today.

**Solid idioms** — correct: signals created at component scope (`Counter.tsx:3`), reactive reads inside JSX, `class` prop (Solid convention — not `className`), `useLocation()` consumed reactively for the nav underline. The `active()` closure in Nav re-reads `location.pathname` per render — reactive, though `<A>` makes it unnecessary.

**Naming & structure** — consistent PascalCase components, camelCase functions, kebab-case files; `components/` + `routes/` split matches SolidStart conventions; `[...404].tsx` catch-all naming is exactly right. Component size is tiny and focused. No lib/util layer — everything is demo-grade; the moment real features arrive, `src/lib`/`src/api`/`src/store` will be needed.

**Accessibility** — the gaps are F7/F8: no `aria-current` on the active nav link, no `<title>`, `target="_blank"` without `noopener`, `<nav>` present but unlabeled[^1]. Buttons and links have accessible names from text content (good). No keyboard-trapping patterns; no forms exist yet.

[^1]: `<nav>` is itself a landmark — technically present; the missing piece is a distinguishing `aria-label` (`aria-label="Main"`) when multiple navs appear later.

**Maintainability** — the config layer is the weakest point: two silently-ignored config files (F1, F2), globs excluding the only source extension that matters (F3), a broken runner (F4), and a hook chain that operates on the wrong package root (F6). The code itself is clean; the pipeline around it is not.

---

## 6. Testing landscape

**Current state (verified):**

- Zero test files: repo-wide glob `**/*.{test,spec,cy}.{ts,tsx,js,jsx}` → 0 matches. `test` script runs `--passWithNoTests`, so "green" has always been vacuous.
- Jest cannot execute (F4). Cypress 13.14.2 is installed with **no** `cypress.config.*`, no `cypress/` dir, no specs — dead weight in `devDependencies` (`package.json:37`).
- No `jest-environment-jsdom`, no `@solidjs/testing-library`, no `@testing-library/dom` → component rendering is untestable even after jest is repaired.

**What must be tested** (in priority order):

1. **Counter logic** — signal start value, increment behavior, no negative counts (trivial but establishes the runner).
2. **Nav active state** — renders `border-sky-600` on the current route's link only; correct under `/` and `/about` (and the future subroute case).
3. **Route rendering** — Home/About/404 render expected landmarks/headings; 404 catch-all responds to arbitrary paths.
4. **SSR shell** — `entry-server.tsx` output: `lang="en"`, `<div id="app">`, favicon link (E2E/Cypress or `renderToString` snapshot).
5. **Router semantics** — client-side nav does not reload the document (would catch an `explicitLinks` regression instantly).

**Component/unit (Jest + `@solidjs/testing-library`)** — sketches only; full files are deliverables of the testing story:

```ts
// jest.config.mjs (replaces jest.config.ts)
export default {
  preset: "ts-jest",
  testEnvironment: "jsdom",                       // was "node" — required for component tests
  moduleNameMapper: { "^~/(.*)$": "<rootDir>/src/$1" },   // required: sources import via ~/
  transform: {
    "^.+\\.tsx?$": [
      "ts-jest",
      { tsconfig: { jsx: "react-jsx", jsxImportSource: "solid-js" } }, // compile Solid JSX in tests
    ],
  },
  collectCoverage: true,
  coverageProvider: "v8",
  coverageThreshold: { global: { lines: 80, statements: 80, functions: 80, branches: 80 } },
  setupFilesAfterEnv: ["<rootDir>/jest.setup.ts"],
};
```

```tsx
// src/components/Counter.test.tsx (sketch)
import { render, screen } from "@solidjs/testing-library";
import userEvent from "@testing-library/user-event";   // or fireEvent
import Counter from "./Counter";

describe("Counter", () => {
  it("renders initial count 0 and increments on click", async () => {
    render(() => <Counter />);
    const btn = screen.getByRole("button", { name: /Clicks: 0/ });
    await userEvent.click(btn);
    expect(await screen.findByRole("button", { name: /Clicks: 1/ })).toBeTruthy();
  });
});
```

```tsx
// src/components/Nav.test.tsx (sketch) — Nav needs router context (useLocation)
import { render, screen } from "@solidjs/testing-library";
import { MemoryRouter } from "@solidjs/router";   // exported (dist/index.js:1668)
import Nav from "./Nav";

describe("Nav", () => {
  it("marks the current route as active", async () => {
    render(() => (
      <MemoryRouter initialEntries={["/about"]}>
        <Nav />
      </MemoryRouter>
    ));
    // With <A>: expect(screen.getByRole("link", { name: "About" }))
    //   .toHaveAttribute("aria-current", "page");   // after F7 migration
    // With current <a>: assert the <li> classList contains border-sky-600
    //   and the Home <li> does not.
  });
});
```

```tsx
// src/routes/index.test.tsx (sketch) — route body inside router context
import { render, screen } from "@solidjs/testing-library";
import { MemoryRouter } from "@solidjs/router";
import Home from "./index";

it("renders home content", () => {
  render(() => (
    <MemoryRouter initialEntries={["/"]}>
      <Home />
    </MemoryRouter>
  ));
  expect(screen.getByRole("heading", { name: /Hello world/i })).toBeTruthy();
  expect(screen.getByRole("link", { name: "About Page" })).toHaveAttribute("href", "/about");
});
```

> Route-level integration with `FileRoutes` is not Jest-friendly (relies on Vinxi's virtual route manifest) — cover full app wiring with Cypress instead.

**End-to-end (Cypress)** — straightforward, and the only place where SSR vs hydration behavior (F7's real-world impact) is observable:

```ts
// cypress/e2e/app.cy.ts (sketch)
describe("customer support app", () => {
  it("navigates between pages without full reload", () => {
    cy.visit("/");
    cy.contains("h1", "Hello world!");
    cy.get("nav").contains("About").click();
    cy.url().should("include", "/about");
    cy.contains("h1", "About Page");
    cy.contains("nav a", "About").should("have.class", "border-sky-600"); // active state
  });
  it("shows the 404 page for unknown routes", () => {
    cy.visit("/does-not-exist");
    cy.contains("h1", "Not Found");
    cy.contains("a", "Home").click();
    cy.url().should("eq", Cypress.config().baseUrl + "/");
  });
  it("increments the counter", () => {
    cy.visit("/");
    cy.get("button").contains("Clicks: 0").click().click();
    cy.get("button").contains("Clicks: 2").should("exist");
  });
});
```

Plus minimal `cypress.config.ts` (`e2e: { baseUrl: "http://localhost:3000" }`) and `test:e2e` script (`cypress run` after `vinxi dev`/`build`+`start`).

**Coverage gate**: enforce `coverageThreshold` at 80% (per Bradesco/Avanade standard) once the runner works — the current `collectCoverage: true` collects but never enforces.

---

## 7. Remediation plan

### P0 — do first (unblocks everything)

1. **Fix ESLint** — replace `eslintrc.config.js` with a real flat config and install the missing deps:
   ```bash
   git rm eslintrc.config.js
   npm i -D typescript-eslint          # v8: bundles parser + plugin, ESLint 9 compatible
   ```
   ```js
   // eslint.config.js (new)
   import tseslint from "typescript-eslint";
   export default tseslint.config(
     { ignores: ["**/node_modules/**", ".output/**", "dist/**", "coverage/**"] },
     ...tseslint.configs.recommended,
     { rules: { "@typescript-eslint/no-unused-vars": "warn" } },
   );
   ```
   Verify: `npx eslint src/app.tsx` exits 0.

2. **Fix Prettier config discovery**:
   ```bash
   git mv prettierrc.js .prettierrc.js
   npx prettier --find-config-path src/app.tsx   # must print .prettierrc.js
   ```
3. **Fix lint/format globs** (include `.tsx`; drop the meaningless `--check` from `format:fix`):
   ```jsonc
   "lint": "eslint .",
   "lint:fix": "eslint . --fix",
   "format:check": "prettier --check \"src/**/*.{ts,tsx,js,jsx,css,json,md}\"",
   "format:fix": "prettier --write \"src/**/*.{ts,tsx,js,jsx,css,json,md}\"",
   "test:coverage": "jest --coverage"
   ```
4. **Repair Jest** (broken `jest-cli@29.7.0` tarball — `node_modules/jest-cli/build` is missing):
   ```bash
   # Option A (deterministic): bump to Jest 30 line (ts-jest 29.3+ supports it — check peers at install)
   npm i -D jest@^30 ts-jest@^29.3 @types/jest@^30
   # Option B (keep 29): rm -rf node_modules/jest-cli && npm i (re-fetches the tarball; may not fix if the
   # registry mirror still serves the broken publish — verify with: npx jest --version)
   ```
5. **Remove or implement dead scripts**:
   - `docs::swagger`: delete until a real swagger pipeline exists (`git rm` nothing — the file doesn't exist; just drop the script).
   - `db:sync`: delete until Prisma + schema are introduced.
   - `test:docker`: replace with `npm run test:coverage` (drop the `--watchAll` variant).
6. **Fix the husky chain** — point the hook at the app package and scope commands to staged files:
   ```bash
   # .husky/pre-commit (repo root)
   cd customer_frontend && npx lint-staged
   ```
   ```jsonc
   // customer_frontend/.lintstagedrc.json
   { "*.{ts,tsx}": ["eslint --fix", "prettier --write", "jest --passWithNoTests --findRelatedTests"] }
   ```
   (Commands run from `customer_frontend` after the `cd`; inline commands stay per-file scoped — do not wrap them in whole-repo npm scripts.)

### P1 — correctness & quality

7. **Migrate Nav to `<A>`** (fixes F7 + a11y, removes `useLocation`/`active()` hand-rolling):
   ```tsx
   <nav class="bg-sky-800" aria-label="Main">
     <ul class="container flex items-center p-3 text-gray-200">
       <li class="mx-1.5 sm:mx-6"><A href="/" end class="border-b-2"
         activeClass="border-sky-600"
         inactiveClass="border-transparent hover:border-sky-600">Home</A></li>
       <li class="mx-1.5 sm:mx-6"><A href="/about" class="border-b-2"
         activeClass="border-sky-600"
         inactiveClass="border-transparent hover:border-sky-600">About</A></li>
     </ul>
   </nav>
   ```
   Note the `end` prop on `/` — without it the Home link matches every route (descendant matching).
8. **Head/a11y pass**: add `<Title>`/`<Meta>` per route (e.g. `<Title>Home — Customer Support</Title>` from `@solidjs/meta` or Solid head primitives); `rel="noopener noreferrer"` on the external link (`index.tsx:6`, `about.tsx:6`, `[...404].tsx:6`).
9. **Dockerfile hygiene**:
   ```bash
   # .dockerignore (new, in customer_frontend/)
   node_modules
   .output .vinxi dist coverage
   .git .env* README.md docs
   npm-debug.log* yarn-debug.log* .eslintcache
   ```
   Production stage: `COPY --from=build /home/node/app/.output ./.output` + `package*.json` + `npm ci --omit=dev` (node_modules for runtime only); drop the blanket `COPY ./ ./`. Remove commented cruft. Verify `docker compose build` after jest is fixed — or decouple: `build:docker` → `npm run build` only (the test gate belongs in CI, not in the image build).
10. **Rename package** (`package.json:2` → `"name": "customer-support-frontend"`), `npm install` to sync lockfile.

### P2 — polish

11. Add `coverage/`, `.eslintcache`, `.prettiercache` to `.gitignore`; add `coverageThreshold` (80%).
12. Delete the `max-6-xs` class (`index.tsx:4`); run `prettier --write` once after F3 is fixed (expect single-quote normalization if defaults were applied meanwhile).
13. Extract the repeated "Visit solidjs.com" block + page shell from `index.tsx`/`about.tsx`/`[...404].tsx` into a shared component.
14. Pin Node: `.nvmrc` with `20` + `"engines": { "node": ">=20 <21" }` (align local v25.4.0, Docker 20, and the app's requirements); consider exact dependency pins (`save-exact`) for CI reproducibility.
15. tsconfig cleanup: drop `emitDecoratorMetadata`/`experimentalDecorators`/`allowJs`; add `skipLibCheck` (verify build first).
16. Git hygiene: squash/fix the duplicate Dockerfile-create commits on the next history rewrite; adopt standard gitmoji (`:construction_worker:`) and feature branches.

Order-of-operations note: **P0 items 1–6 must land together** — they are one broken pipeline (lint → format → test → hooks → CI). After that, P1 items 7/8 produce the first meaningful user-visible and a11y improvements, and P1 item 9 restores deliverability. Tests (testing story, §6) should be introduced in the same PR as P1 item 7 so the Nav migration is covered by the new suite.

---

## 8. Verification evidence (read-only probes, 2026-08-19)

| Command | Output (abridged) | Conclusion |
|---------|-------------------|------------|
| `npx eslint src/app.tsx` | `Oops! Something went wrong!` / `ESLint couldn't find an eslint.config.(js|mjs|cjs) file.` | F1 — no loadable config |
| `Test-Path node_modules/@typescript-eslint/parser` | `False` (scope dir absent) | F1 — plugin deps missing |
| `npx prettier --find-config-path src/app.tsx` | `[error] Can not find configure file for "src/app.tsx"` | F2 — config ignored |
| `npx prettier --check --write <tmp file>` | `Checking formatting... [warn] Code style issues fixed` (exit 0) | `--check`+`--write` combo writes and exits 0 — `--check` contract neutralized |
| `npx jest --listTests` | `Error: Cannot find module 'jest-cli/build/index.js'` | F4 — broken install |
| `Get-ChildItem node_modules/jest-cli` | `bin, node_modules, LICENSE, package.json, README.md` — **no `build/`** | F4 — missing compiled artifacts |
| `glob **/swagger.js` | 0 matches | F5 |
| `node_modules/prisma` / lockfile `"prisma"` | absent / 0 matches | F5 |
| `glob **/*.{test,spec,cy}.{ts,tsx,js,jsx}` | 0 matches | no tests anywhere |
| `node_modules/@solidjs/router/README.md:738` | `explicitLinks ... Default: false` | F7 — anchors intercepted by default |
| `node_modules/@solidjs/router/dist/index.js:1204-1227` | `handleAnchor` → `if (!a || explicitLinks && !a.hasAttribute("link")) return;` → `evt.preventDefault()` | F7 — plain `<a>` client-navs under default config; reloads only pre-hydration/with `explicitLinks` |
| `node_modules/@solidjs/router/dist/components.jsx:25-30` | `<A>` renders `link` attr + `aria-current={...}` | F7 — what Nav forfeits |
| `git log --all --oneline --name-status -- "*Dockerfile*"` | `73c0a6f` M, `49c201c` A (same message) | F13 — duplicated "create" messages |

All findings above were derived from the repository itself (`HEAD` @ `73c0a6f`, branch `develop`); no source files were modified.