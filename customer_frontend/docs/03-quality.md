# 03 — Quality Analysis Report

> **Auditor:** Carla (QA Engineer) · **Date:** 2026-08-19 · **Scope:** `customer_frontend/` (SolidStart 1.0.6 app)
> **Method:** Static review of all tracked config/source files (33 files) + **empirical execution** of every quality script on this machine (Windows, git 2.45.1, npm). Every claim below was either observed in a terminal or pinned to a `file:line` reference. Findings marked **[blocked]** could not be fully verified due to upstream failures — see Appendix A.

---

## 0. Executive summary

**Verdict: ❌ REJECTED — the quality toolchain is broken at 4 independent layers. Nothing the team believes is protecting the repo actually runs.**

Empirically confirmed, in order of blast radius:

| # | Finding | Severity | Evidence |
|---|---------|----------|----------|
| 1 | `npm test` crashes — `jest-cli/build/index.js` missing in `node_modules` | **P0** | terminal run, §1.3 |
| 2 | Pre-commit hook fails — `npx lint-staged` cannot resolve lint-staged at repo root | **P0** | terminal run, §1.5 |
| 3 | ESLint has no loadable config — `eslintrc.config.js` is not a valid flat-config filename | **P0** | terminal run, §1.1 |
| 4 | `code:ci` and `test:docker` reference a script (`test:coverage`) that does not exist | **P0** | `package.json:18-19` |
| 5 | Prettier config is never loaded — `prettierrc.js` is not a recognized filename | **P1** | terminal run, §1.2 |
| 6 | All lint/format globs `{ts,js}` match **zero** files — every component is `.tsx` | **P1** | terminal run, §1.1/1.2 |
| 7 | Jest runs `node` environment, no jsdom, no CSS transform — component tests are impossible today | **P1** | `jest.config.ts:4`, §1.3 |
| 8 | **Zero** unit tests, zero E2E specs, Cypress installed but never configured | **P1** | git history, §2 |
| 9 | Husky lives at repo root, app config lives in `customer_frontend/` — split-brain monorepo layout | **P1** | §1.5 |
| 10 | SolidStart 1.0.6 + vinxi 0.4.2: framework generation is EOL/being replaced (v2.0.1 current) | **P2** | §4.3 |

**Positives (verified):** Docker runs as `USER node` (`Dockerfile:5,21`), multi-stage build with `npm ci` (`Dockerfile:8-9`), `.env`/`.env*.local` ignored (`customer_frontend/.gitignore:10-11`), **no** `.env` or secrets tracked in git, clean gitmoji commit discipline, no build artifacts (`.output`/`.vinxi`) tracked.

---

## 1. Tooling audit (evidence-based)

### 1.1 ESLint 9 — config not found, deps missing, globs match nothing

**Config file naming.** ESLint 9 (flat config) only auto-detects `eslint.config.js|mjs|cjs`. The project ships `customer_frontend/eslintrc.config.js` — a **legacy-format file with the wrong prefix**. Empirical run:

```
$ npx eslint ./src --no-error-on-unmatched-pattern
ESLint couldn't find an eslint.config.(js|mjs|cjs) file.
From ESLint v9.0.0, the default configuration file is now eslint.config.js.
```

**Even after renaming, the config cannot load:**
- `eslintrc.config.js:1` uses `module.exports` (CJS) inside a `"type": "module"` package (`package.json:3`) → Node treats `.js` as ESM → `module.exports` is `undefined`.
- `eslintrc.config.js:9,23,29` reference `@typescript-eslint/parser` and `@typescript-eslint/eslint-plugin` — **neither is in `devDependencies`** (`package.json:35-45`: only `eslint`, `eslint-config-prettier`) and neither exists in `node_modules` (verified: `Test-Path node_modules/@typescript-eslint/parser` → `False`). A renamed config would fail with `Failed to load plugin '@typescript-eslint'`.
- `eslintrc.config.js:27` sets `parserOptions.project` for typed linting — requiring the parser that isn't installed.

**Globs exclude all real source.** `lint:fix` (`package.json:11`) runs `eslint ./src/**/*.{ts,js}`. All 10 source files are `.tsx` (verified via `git ls-files`). Prettier proves the glob semantics: `npx prettier --check "./src/**/*.{ts,js}"` → `All matched files use Prettier code style!` on **zero files**. ESLint never sees `src/app.tsx`, `src/components/*.tsx`, `src/routes/*.tsx`.

**Secondary Windows bug:** `lint:staged` = `eslint --fix;` (`package.json:12`). Under `cmd.exe` (npm's default Windows shell) `;` is **not** a separator — ESLint receives `--fix;` as a literal option. (Masked today by the missing config, so flagged as secondary; verified conceptually, see Appendix A.)

**Fixes required:** rename to `eslint.config.mjs` with flat-config `export default`; add `typescript-eslint` (meta package) to devDeps; change globs to `./src/**/*.{ts,tsx,js}` (or simply `./src`); remove trailing `;` in `lint:staged`; run from `customer_frontend/` cwd.

### 1.2 Prettier 3 — config silently ignored, `--check --write` redundancy

**Filename.** Prettier's search places are `.prettierrc`, `.prettierrc.json`, `.prettierrc.yml`, `.prettierrc.yaml`, `.prettierrc.json5`, `.prettierrc.js`, `.prettierrc.cjs`, `.prettierrc.mjs`, `prettier.config.js|cjs|mjs` — **`prettierrc.js` (no leading dot) is not among them**. Empirical:

```
$ npx prettier --find-config-path src/app.tsx
[error] Can not find configure file for "src/app.tsx".
```

Prettier therefore runs with **defaults** (`singleQuote: false`, etc.) — the team's formatting rules (`prettierrc.js:1-5`: `semi`, `singleQuote: true`, `trailingComma: "all"`, `tabWidth: 2`) are dead code. This also means `format:check`/`format:fix` enforce *default* style, not the project style, and `eslint-config-prettier` (`eslintrc.config.js:10`) disables stylistic rules in a linter whose formatter never loads its config — double inconsistency.

**Flag conflict.** `format:fix` = `prettier --check --write` (`package.json:13`). Empirically this does **not** error — `--write` wins and files get rewritten (verified on a temp file). Harmless but misleading; `--check` should be removed from the write script.

**Glob gap (same as ESLint):** `format:fix`/`format:check` (`package.json:13-14`) use `{ts,js}` → match zero `.tsx` files → the prettier check is a no-op that reports success. This is the *most dangerous* failure mode: **green CI output that validated nothing**.

**Fixes required:** rename to `.prettierrc.cjs` (CJS syntax + `type: module` package); fix globs to `{ts,tsx,js}`; drop `--check` from `format:fix`.

### 1.3 Jest 29 — runner is broken, environment wrong, CSS imports will crash

**Runner crash (P0).** `npm test` fails *before running anything*:

```
$ npm test
> jest --passWithNoTests --color
Error: Cannot find module 'C:\...\customer_frontend\node_modules\jest-cli\build\index.js'
```

`jest-cli@29.7.0` declares `"main": "./build/index.js"` (its `package.json`), but the `build/` directory does **not** exist locally (verified `Test-Path` → `False`, directory empty). The lockfile is consistent (`package-lock.json:8656` lists `jest-cli ^29.7.0`), so `node_modules` is locally corrupt (dates show a 2024-09-13 install). **A clean `npm ci` likely repairs it — unverified, see Appendix A.** Regardless: today, **every** test script (`test`, `test:staged`, `test:dev`, `test:docker`, `code:ci`, and the Docker build via `build:docker` → `npm run test`) fails.

**Wrong environment.** `jest.config.ts:4` → `testEnvironment: "node"`. SolidJS component tests require a DOM (jsdom). `jest-environment-jsdom` is **not installed** (verified). Only pure-logic unit tests (signals, utilities) could run under `node` — of which there are zero.

**CSS imports unhandled.** `src/app.tsx:5` does `import "./app.css"`. The jest config (`jest.config.ts:2-8`) has no `moduleNameMapper`/transform for CSS → any test importing `App`/routes will throw `Unexpected token ... app.css` under ts-jest. This is not hypothetical: **every** component in this app imports through `app.tsx` or `Nav.tsx` (router), so *every* component test hits this.

**Config-load risk (blocked).** `jest.config.ts` is CJS (`module.exports`, line 2) in a `"type": "module"` package, with tsconfig `"module": "ESNext"` (`tsconfig.json:6`) and `"moduleResolution": "bundler"` (`tsconfig.json:7`). Jest 29 loads `.ts` configs via ts-node honoring the tsconfig → `module: ESNext` transpilation would not set `module.exports`, and Jest would silently fall back to **defaults** (losing `collectCoverage`, `testEnvironment`, and the ts-jest preset). Could not be empirically confirmed because the runner is broken **[blocked]**. The safe fix: rename to `jest.config.cjs`.

**Missing scripts.** `code:ci` (`package.json:19`) and `test:docker` (`package.json:18`) both call `npm run test:coverage` — **no such script exists** in `package.json:4-22`. Both commands fail immediately. Additionally `db:sync` (`package.json:20`) calls `prisma generate` — `prisma` is not in `dependencies` or `devDependencies` (verified `npm ls` + `package.json:23-45`).

### 1.4 lint-staged — config unreachable from where the hook runs

`.lintstagedrc.json` exists at `customer_frontend/.lintstagedrc.json:1-7` with one task block `*.{ts,tsx}`. Two independent breaks:

1. **Binary cannot resolve.** The hook runs `npx lint-staged` from the **repo root**; root `node_modules` contains only `husky` (`npm ls --prefix . husky lint-staged` at root → `husky@9.1.6` only). Empirical: `npx --no-install lint-staged --version` at root → `npm error npx canceled due to missing packages and no YES option: ["lint-staged@17.3.0"]`. In a git hook there is no TTY to answer npx's install prompt → **hook exits non-zero → every commit is blocked** (or, if interactive, silently installs *latest* lint-staged 17.x — not the locked 15.2.10).
2. **Config is in the wrong directory tree.** lint-staged searches for config from the working directory **upward**. From repo root it finds root `package.json` (no `lint-staged` key — verified `package.json:1-8`) and never descends into `customer_frontend/.lintstagedrc.json` → would error `No lint-staged config found` even if the binary resolved.

Additionally the configured tasks themselves are broken: `npm run format:fix` (no-op glob, §1.2), `npm run lint:staged` (`--fix;` Windows bug, §1.1), `npm run test:staged` (jest crash, §1.3).

**Fix:** hoist lint-staged config to the repo root (or add a `lint-staged` key to root `package.json`) with commands prefixed `cd customer_frontend && ...`, and add `lint-staged` to root devDeps (or use `npx --yes lint-staged` with a pinned version).

### 1.5 Husky 9 — active, but points at a broken hook

Verified active: `git config core.hooksPath` → `.husky/_`; `.husky/_/pre-commit` shim sources `h` → runs `.husky/pre-commit`. The hook file is tracked (`git ls-files .husky` → `.husky/pre-commit`) and contains exactly `npx lint-staged` (`.husky/pre-commit:1`).

So the pre-commit pipeline **is** wired — and it is wired to the broken lint-staged invocation from §1.4. Result: every commit currently either **fails** (non-TTY npx cancel) or hangs waiting for an npx prompt. Note the repo split-brain: husky/root package.json live at repo root, the entire app (and its devDeps) live in `customer_frontend/`. `husky:setup` (`package.json:21`) papers over this with `cd ../ && husky init`.

---

## 2. Testing gaps & proposed test strategy

### 2.1 Current state (verified)

| Layer | Status | Evidence |
|-------|--------|----------|
| Unit tests | **0 files** | `git ls-files` → no `*.test.*`/`*.spec.*`; glob over repo → none |
| Component tests | **0 files** | same |
| E2E | **0 specs, 0 config** | no `cypress.config.*`, no `cypress/` dir; `cypress@13.14.2` is installed but dead weight (`package.json:37`) |
| Coverage | no `coverage/` dir, no thresholds | `jest.config.ts:6-8` collects but nothing runs |
| Git history | only *dep-add* commits for jest/ts-jest (`60a2922`, `1cd7e62`, `f7dd4d9`) — **tests were never written** | `git log` |

Testable surface today: `Counter` (signal logic + click), `Nav` (router-aware active-link logic), 3 routes, `App` (router root + Suspense). Small app — coverage ≥80% is achievable in a day of work.

### 2.2 Framework recommendation — vitest (official Solid path)

The **official SolidJS testing guide** (docs.solidjs.com/guides/testing, verified 2026-08-19) states: *"The recommended testing framework for Solid applications is vitest"*, with `@solidjs/testing-library` (current: 0.8.10), `@testing-library/user-event`, `@testing-library/jest-dom`, jsdom, and a **SolidStart-specific recipe** (`vitest.config.ts` with `vite-plugin-solid` + `resolve.conditions: ["development", "browser"]`).

Why switch vs. fixing Jest (both are real options; recommendation is evidence-based):

| Criterion | Vitest (recommended) | Fix Jest (fallback) |
|-----------|---------------------|---------------------|
| Official Solid guidance | ✅ first-class | ⚠️ community path |
| SolidStart config | ✅ documented recipe | ❌ no official recipe |
| JSX/TSX handling | ✅ via `vite-plugin-solid` (same pipeline as dev/build) | ⚠️ ts-jest + `jsxImportSource: "solid-js"` — works but diverges from Vite pipeline |
| CSS imports | ✅ Vite handles `app.css` natively | ❌ needs `moduleNameMapper` shims |
| Current state | greenfield (nothing to migrate) | 🩹 `node_modules` corrupt, config risky (`jest.config.ts` CJS/ESM), wrong env, no CSS transform — **fixing = rebuild anyway** |
| Speed | ⚡ Vite/esbuild (fine-grained HMR for `test:dev`) | 🐢 ts-jest |

**Decision: adopt vitest.** If the team insists on Jest (already invested in `@types/jest`, ts-jest), the required patch list is: `npm ci`, rename `jest.config.cjs`, `testEnvironment: "jsdom"` + `jest-environment-jsdom`, CSS `moduleNameMapper`, add `@solidjs/testing-library` (supports jest), fix globs. Both paths are documented; pick one this sprint.

### 2.3 Proposed test pyramid (targets ≥80%)

| Layer | Tooling | Scope (concrete) | Targets |
|-------|---------|------------------|---------|
| **Unit** | vitest (or jest) | `Counter` count state transitions; `Nav.active()` boundary logic (path match/mismatch, `/` vs `/about`); pure helpers | ≥80% lines on `src/lib`-type logic; 100% on new utils |
| **Component** | `@solidjs/testing-library` + user-event + jest-dom | `Counter` click → text updates (`getByRole("button")`); `Nav` renders both links with correct aria-current/active class; routes render headings; `App` renders children within Router+Suspense | ≥80% statements, ≥75% branches on `src/components`, `src/routes` |
| **E2E** | Cypress 13 (already installed — configure `cypress.config.ts`, `baseUrl` from `vinxi dev`/`start`, `e2e/` specs) | smoke: `/` loads, Counter increments, nav to `/about` works, unknown URL → 404 route; SSR check: response contains `Hello world!` | happy path + 404 (4-6 specs) |

Coverage thresholds (in the test config, not prose): `global: { statements: 80, branches: 75, functions: 80, lines: 80 }` — fail CI below.

---

## 3. Quality gates proposal

### 3.1 CI pipeline (what should run, in order, each gated)

| Stage | Command (from `customer_frontend/`) | Fails on |
|-------|-------------------------------------|----------|
| 1. Install | `npm ci` (root + app) | lockfile drift |
| 2. Typecheck | `npx tsc --noEmit` — **add script** `typecheck` (missing today; `tsconfig.json` is strict ✅) | type errors |
| 3. Format | `prettier --check "./src/**/*.{ts,tsx}"` | unformatted code |
| 4. Lint | `eslint ./src` (flat config) | lint errors |
| 5. Unit+component | vitest run --coverage (thresholds §2.3) | failures or <80% |
| 6. E2E | `cypress run` against built server | failing spec |
| 7. Build | `npm run build` (already in Docker) | build failure |

`code:ci` (`package.json:19`) should become **read-only**: `format:check && lint && typecheck && test:coverage` — never `--fix`/`--write` inside CI (mutating CI is a code smell and hides drift).

### 3.2 Pre-commit redesign (speed + reliability)

Current pre-commit is **all-or-nothing and broken** (§1.4-1.5). Proposed:

- Move husky/lint-staged config **to repo root** (single owner), tasks run `cd customer_frontend && ...`.
- Keep only **fast, staged-scoped** tasks per commit: `prettier --write` + `eslint --fix` on staged files (lint-staged file list). **Drop `test:staged` from the hot path** — with `--findRelatedTests` it's fast today (zero tests) but becomes slow as suites grow; move tests to CI.
- Use pinned `npx --yes lint-staged@15.2.10` or add lint-staged to root devDeps (no drift).
- Add a `commit-msg` hook validating gitmoji format (team convention is already consistent — make it enforced, cheap).

### 3.3 Definition of Done checklist (for every story)

- [ ] `tsc --noEmit` passes (strict)
- [ ] `eslint ./src` passes (flat config, parser installed)
- [ ] `prettier --check` passes on `{ts,tsx}`
- [ ] Tests written for changed behavior (red→green, not afterthought)
- [ ] Coverage thresholds ≥80% global, no regressions
- [ ] E2E smoke updated if routing/navigation changed
- [ ] CI green on the PR branch (all 7 stages, §3.1)
- [ ] No secrets in diff; `.env*` untouched
- [ ] `npm run build` succeeds locally

---

## 4. Security & ops risks

### 4.1 Secrets & VCS hygiene ✅ / ⚠️

| Check | Status | Evidence |
|-------|--------|----------|
| Secrets hardcoded in repo | ✅ none found | reviewed `package.json`, all configs, `src/`; `git ls-files` clean |
| `.env` tracked | ✅ not tracked | `git ls-files` → no `.env*` |
| `.env` ignored | ✅ | `customer_frontend/.gitignore:10-11` (`.env`, `.env*.local`) |
| `.env.production` / `.env.test` | ⚠️ **not ignored** | only `*.local` variants covered — add `.env.*` pattern |
| `coverage/` ignored | ✅ via root `.gitignore:23` | works (pattern applies repo-wide) but silently — document it or add to subdir `.gitignore` |
| Build artifacts (`dist`, `.output`, `.vinxi`) | ✅ not tracked | `git ls-files` verified |

### 4.2 Docker ✅ / ⚠️

- ✅ `USER node` in both stages (`Dockerfile:5,21`) — never root; `COPY --chown=node:node` consistent (`Dockerfile:8,11,24-28`).
- ✅ Multi-stage build; `npm ci` (reproducible); tests run in build (`build:docker` → `test && build`, `package.json:7`) — though "tests" today = jest crash (§1.3), so **the Docker build is currently broken by the same P0**.
- ⚠️ Production stage copies the **entire** build-stage tree (`Dockerfile:28`) after already copying `node_modules`, `.vinxi`, `.output`, `package*.json` — redundant layers, image bloat, and `src/` shipped into prod image. Slim to the 4 explicit copies.
- ⚠️ No `HEALTHCHECK` (`docker-compose.yaml:1-6` maps `3000:3000` with zero health config) — a container that crashes stays "running" for orchestrators.
- ⚠️ `docker-compose.yaml:5` maps `3000:3000` but nothing pins the server port to env — fine for dev, fragile for prod.

### 4.3 Dependency hygiene — the big one ⚠️

| Package | Installed | Current | Gap | Risk |
|---------|-----------|---------|-----|------|
| `@solidjs/start` | 1.0.6 | **2.0.1** | 1 major | v1 uses vinxi (see below); v2 is the "devinxified" rewrite (solidjs/solid-start#1743) — v1 is maintenance |
| `vinxi` | 0.4.2 | 0.5.11 | minor | **EOL in practice**: SolidStart maintainers announced replacing vinxi (discussion #1743, Jan 2025); v2 completed it. Direct `vinxi build` scripts (`package.json:6,9-10`) are on a deprecated tool |
| `solid-js` | 1.8.22 | 1.9.15 | minor | fine but drift — `^1.8.18` will not auto-bump past 1.8.x for new majors... actually `^1.8.18` allows 1.9.x ✅ — but *installed* is 1.8.22; plain `npm update` needed |
| `eslint` | 9.10.0 | 10.8.1 | 1 major | v9 flat-config journey already incomplete; v10 moves further — pin and finish v9 config first |
| `cypress` | 13.14.2 | 15.21.0 | 2 majors | installed but unconfigured; if adopted, upgrade during E2E setup (15.x is current stable) |
| `lint-staged` | 15.2.10 | 17.x | 2 majors | hook resolves **17.3.0** from registry when it does resolve (§1.5) — version drift between devDeps and executed hook |

**Hardcoded secrets:** none. **Lockfile:** present at both levels (`package-lock.json` root + app) ✅.

---

## 5. Priority action list

| Prio | Action | Effort | Unblocks |
|------|--------|--------|----------|
| P0-1 | `npm ci` in `customer_frontend/` (repair corrupt jest-cli) **or** decide vitest migration and install vitest stack (§2.2) | 0.5h | everything test-related |
| P0-2 | Rewrite ESLint config as `eslint.config.mjs` flat config + install `typescript-eslint`; fix globs `{ts,tsx,js}` | 1h | lint, CI |
| P0-3 | Fix pre-commit: hoist lint-staged config + binary to root, `cd customer_frontend` tasks, drop `test:staged` | 1h | commits |
| P0-4 | Rename `prettierrc.js` → `.prettierrc.cjs`; fix globs; drop `--check` from `format:fix`; add `typecheck` script; remove phantom `test:coverage` refs (point at real script) | 0.5h | format, CI |
| P1 | Write unit + component tests (Counter, Nav, routes) with `@solidjs/testing-library`; set coverage thresholds ≥80% | 1-2d | DoD, regressions |
| P1 | Configure Cypress (`cypress.config.ts` + e2e smoke) | 0.5-1d | E2E layer |
| P2 | Plan SolidStart 2.x migration (retires vinxi); update cypress/eslint majors; add `.env.*` ignore; slim Docker prod stage + HEALTHCHECK | backlog | maintenance risk |

---

## Appendix A — Limitations & blocked verifications (honest)

1. **context7 MCP unavailable** (monthly quota exceeded on all 3 resolves: solid-js testing, jest, cypress). Best-practice claims for Solid testing were verified instead against the **official docs.solidjs.com/guides/testing** page (fetched 2026-08-19) — the vitest recommendation is directly from that page, including the SolidStart `vitest.config.ts` recipe.
2. **[blocked] `jest.config.ts` load behavior** — cannot execute jest to confirm whether the CJS/ESM mismatch silently drops config; the runner crashes first (§1.3). The risk assessment is static-analysis-based; rename to `.cjs` regardless.
3. **[blocked] `eslint --fix;` option-error on Windows** — the missing-config error fires before option parsing, so the `--fix;` argument bug could not be observed directly; it is deterministic from cmd.exe semantics but unproven end-to-end.
4. **[blocked] Does `npm ci` repair jest-cli?** — lockfile contains `jest-cli ^29.7.0` (`package-lock.json:8656`) which makes a clean install plausible, but I did not run `npm ci` (would mutate `node_modules`, outside the no-source-modification scope). Treat "fixed by npm ci" as hypothesis, not fact.
5. **No CI config exists in the repo** (no `.github/workflows`, `.gitlab-ci.yml`, etc. — verified via `git ls-files`), so the §3.1 pipeline is a proposal against a vacuum, not a patch to an existing pipeline.
6. All empirical commands were run 2026-08-19 on Windows PowerShell/cmd against the working tree; outputs quoted verbatim (truncated to first lines where noted).

*— Carla, QA Engineer. This report modifies no source files; only this document was created.*
