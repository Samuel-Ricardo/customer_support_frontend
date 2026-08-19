# 05 — DevOps & Deployment Deep Analysis

> **Scope:** `customer_frontend/` (SolidStart 1.x app) + repo root (husky)
> **Date:** 2026-08-19 · **Branch checked out:** `develop` (upstream: `main` + `develop`)
> **Remote:** https://github.com/Samuel-Ricardo/customer_support_frontend
> **Type:** analysis + proposal document — **no files were modified** as part of this write-up
> **SolidStart preset reference:** verified against official docs (docs.solidjs.com `defineConfig` reference, vinxi CLI docs) on 2026-08-19.

---

## 1. Current state (verified, not assumed)

| Area | Fact | Assessment |
|---|---|---|
| CI | **No `.github/workflows` exists** — zero CI, zero status checks on `main`/`develop` | Critical gap: nothing gates merges |
| `.dockerignore` | **Does not exist** (checked repo-wide) | Build context ships `node_modules/`, `.git/`, `.vinxi/`, `.output/` |
| Dockerfile | `customer_frontend/Dockerfile`, 2-stage (`build` → `production`) | Structurally valid; one red flag (line 28) |
| docker-compose | `customer_frontend/docker-compose.yaml`, 6 lines | Minimal, dev-oriented, no health/restart |
| `app.config.ts` | `defineConfig({})` — **no deployment preset** | Runs on the implicit default (node); not reproducible across platforms |
| Secrets | `.env`, `.env*.local` already in `.gitignore` ✅ | Good — no cleanup needed |
| Husky | Root `.husky/pre-commit` → `npx lint-staged` | Broken wiring (see §4.2) |
| Build artifacts | `.output/`, `.vinxi/`, `node_modules/` exist locally; all gitignored | Will leak into Docker context without `.dockerignore` |
| Tests | Jest configured (`ts-jest`, node env, v8 coverage), **zero test files**; `--passWithNoTests` masks it | Coverage numbers currently meaningless |
| E2E | Cypress in devDeps, **no `cypress.config.*`** | E2E job in CI needs setup first |

---

## 2. Dockerfile review (`customer_frontend/Dockerfile`)

```dockerfile
 1: FROM node:20-slim as build
 2:
 3: #RUN apt-get update -y && apt-get install -y openssl
 4:
 5: USER node
 6: WORKDIR /home/node/app
 7:
 8: COPY --chown=node:node package*.json ./
 9: RUN npm ci
10:
11: COPY --chown=node:node . .
12: RUN npm run build:docker
13:
14: #CMD ["npm", "run", "start:dev"]
15: #CMD [ "tails", "-f", "/dev/null" ]
16:
17: FROM node:20-slim as production
18:
19: #RUN apt-get update -y && apt-get install -y openssl
20:
21: USER node
22: WORKDIR /home/node/app
23:
24: COPY --chown=node:node --from=build /home/node/app/node_modules ./node_modules
25: COPY --chown=node:node --from=build /home/node/app/package*.json ./
26: COPY --chown=node:node --from=build /home/node/app/.vinxi ./.vinxi
27: COPY --chown=node:node --from=build /home/node/app/.output ./.output
28: COPY --chown=node:node --from=build /home/node/app/ ./   <-- RED FLAG
29:
30: CMD [ "npm", "run", "start:docker" ]
```

### 2.1 Line-by-line

| Line | Verdict | Critique |
|---|---|---|
| 1 | ⚠️ | `node:20-slim` is a **mutable tag** — builds are not reproducible. Pin a specific minor (e.g. `node:20.19.4-slim`) and, for hard assurance, a digest (`node@sha256:…`). Automate bumps with Dependabot/Renovate. |
| 3 | ❌ | Commented `apt-get openssl` — dead code. Openssl is only needed for Prisma at runtime; Prisma is **not installed** in this project. If ever needed: `apt-get update && apt-get install -y --no-install-recommends openssl && rm -rf /var/lib/apt/lists/*` (single RUN, with cleanup) — and it belongs in the **production** stage, not build. |
| 5–6 | ✅ | `USER node` before `COPY`/`RUN` — non-root, correct. `WORKDIR /home/node/app` is the canonical non-root layout. |
| 8–9 | ✅/⚠️ | Dependency layer caching via `COPY package*.json` + `npm ci` is correct (lockfile-driven, reproducible). Improvements: `RUN npm ci --no-audit --no-fund && npm cache clean --force` — `npm ci` populates `~/.npm` inside the layer (~100 MB+ of junk) unless cleaned. |
| 11 | ❌ | `COPY . .` with **no `.dockerignore`**: the entire context (local `node_modules/`, `.git/`, `.vinxi/`, `.output/`) is copied over the fresh `npm ci` install. Local `node_modules` can **shadow/overwrite** the clean install (platform drift, stale versions) and `.git` ships into the image. This file must exist (proposal in §3). |
| 12 | ⚠️ | `npm run build:docker` = `test && build`. Jest runs **inside the image build** with zero tests (`--passWithNoTests` masks the empty suite) while `collectCoverage: true` writes `coverage/` into the layer. Tests belong in CI, not in `docker build`. |
| 14–15 | ❌ | Commented dev CMDs — dead code, delete. |
| 17–22 | ✅ | Production stage is genuinely separate; `USER node` again — good. |
| 24 | ⚠️ | Copies **full `node_modules` including devDependencies** — jest, ts-jest, eslint, prettier, ts-node, and **Cypress** (hundreds of MB) ship in the production image. Fix: `npm prune --omit=dev` after the copy (see §2.3). Note: `vinxi` is in `dependencies` (verified) so `vinxi start` keeps working after pruning — by design, if unusual. |
| 25 | ✅ | `package*.json` — needed by `npm run start:docker`. Keep. |
| 26 | ❌ | `.vinxi` is the **build-time router cache** — not needed at runtime. `vinxi start` (node-server preset) serves from `.output` (verified against vinxi CLI docs). Remove. |
| 27 | ✅ | `.output` is the runtime payload for the node-server preset. Keep. |
| 28 | 🚩 **RED FLAG** | `COPY --from=build /home/node/app/ ./` copies the **entire build-stage working directory** over the destination image. Consequences: (a) it **duplicates and overwrites** lines 24–27, making them pointless; (b) everything `COPY . .` pulled into the build stage — including `.git`, `coverage/`, stray artifacts — lands in the production image, defeating multi-stage separation; (c) any future build-stage change silently changes the runtime image — no provenance. **Delete this line.** |
| 30 | ⚠️ | `npm run start:docker` → `npm run start` → `vinxi start`: two-hop indirection and an `sh` wrapper as PID 1 (no zombie reaping). Acceptable for now; see §2.2 for the tightened version. Missing: `ENV NODE_ENV=production` (add — dev-mode detection, npm lifecycle behavior), `EXPOSE 3000`, `HEALTHCHECK`. |

### 2.2 Recommended Dockerfile (replaces the above)

```dockerfile
# syntax=docker/dockerfile:1

FROM node:20.19.4-slim AS build

USER node
WORKDIR /home/node/app

COPY --chown=node:node package*.json ./
RUN npm ci --no-audit --no-fund && npm cache clean --force

COPY --chown=node:node . .
RUN npm run build

FROM node:20.19.4-slim AS production

ENV NODE_ENV=production
ENV PORT=3000

USER node
WORKDIR /home/node/app

COPY --chown=node:node package*.json ./
RUN npm ci --omit=dev --no-audit --no-fund && npm cache clean --force

COPY --chown=node:node --from=build /home/node/app/.output ./.output

EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=3s --start-period=15s --retries=3 \
  CMD ["node", "-e", "fetch('http://127.0.0.1:3000/health').then(r=>process.exit(r.ok?0:1)).catch(()=>process.exit(1))"]

CMD ["node", ".output/server/index.mjs"]
```

Notes on the changes:

- **`npm ci --omit=dev` in the production stage** (instead of copying build `node_modules`): only runtime deps ship; no Cypress/Jest/TS toolchain in the image.
- `node:20-slim` has **no `curl`/`wget`** — don't apt-install curl just for a healthcheck. Node's built-in `fetch` (Node ≥18) is the lean choice.
- `CMD` runs the node-server entrypoint directly — real PID 1, no `sh` wrapper. If the entry filename differs in your SolidStart/vinxi version, confirm with `ls .output/server/` after a build; `vinxi start` remains the safe fallback.
- Keep `npm run build` (not `build:docker`) in the build stage — tests move to CI (§6).
- Still pin the base digest for full reproducibility; Dependabot keeps tags fresh.

### 2.3 Image-size impact

Current production image ships: full `node_modules` (dev + prod), `.vinxi`, `.output`, `coverage/`, `.git`, source files. The proposed version ships: prod-only `node_modules` + `.output`. Removing Cypress alone typically saves hundreds of MB; expect a **roughly 50–70% smaller production image** — worth measuring before/after.

---

## 3. `.dockerignore` — does not exist, must be created

Verified: no `.dockerignore` anywhere in the repo. `.gitignore` does **not** help Docker — the build context is sent wholesale. With `node_modules/` + `.output/` + `.vinxi/` + `.git/` present locally, every build uploads tens/hundreds of MB and risks shadowing the clean `npm ci` (line 11 of the current Dockerfile). Proposed content:

```dockerignore
# Dependencies
node_modules

# Build artifacts
.output
.vinxi
.solid
dist
coverage

# Git & CI
.git
.github
.husky

# Environment & secrets
.env
.env*.local

# Docs & tooling
docs
*.md
docker-compose.yaml
Dockerfile
*.log
.eslintcache

# OS junk
.DS_Store
Thumbs.db
```

Place at `customer_frontend/.dockerignore`. Crash-test after adding: `docker build . --progress=plain` and confirm the context size in the first lines (`=> [internal] load build context`).

---

## 4. docker-compose review

```yaml
services:
  frontend:
    image: customer_frontend
    build: .
    ports:
      - 3000:3000
```

### 4.1 Critique

| Aspect | Verdict | Analysis |
|---|---|---|
| `image: customer_frontend` | ⚠️ | Local-only name. For registry/CD, use `ghcr.io/<owner>/<repo>:<branch-or-tag>` (lowercase required by GHCR). |
| `build: .` | ✅ | Context = `customer_frontend/` (compose lives there) — correct. |
| `ports: 3000:3000` | ✅ | Fine. `"3000:3000"` quoted is marginally safer YAML. |
| restart policy | ❌ | **Missing** — container dies once and stays down. Add `restart: unless-stopped`. |
| env | ❌ | No `.env` / `env_file` — any runtime config must be baked or absent. Add `env_file: .env` (gitignored) or explicit `environment:`. |
| healthcheck | ❌ | No healthcheck — orchestrators/uptime tooling cannot assess liveness. |
| volume | ⚠️ | No volume — correct for a built image (do **not** bind-mount `node_modules` in a prod-style compose); for local dev, a `src` bind-mount + anonymous `node_modules` volume is a separate dev-compose concern. |
| `init` | ❌ | No `init: true` — node behind an `sh` wrapper won't reap zombies. |

### 4.2 Recommended compose (drop-in for existing usage)

```yaml
services:
  frontend:
    image: ghcr.io/samuel-ricardo/customer_support_frontend:develop
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "3000:3000"
    restart: unless-stopped
    env_file:
      - .env
    init: true
    healthcheck:
      test: ["CMD", "node", "-e", "fetch('http://127.0.0.1:3000/health').then(r=>process.exit(r.ok?0:1)).catch(()=>process.exit(1))"]
      interval: 30s
      timeout: 3s
      retries: 3
      start_period: 15s
```

`.env` stays **gitignored** (already is) and holds runtime-only values.

---

## 5. Git workflow & repository hygiene

### 5.1 Current state + gaps

- `main` + `develop`, both tracked, `develop` checked out. No CI status checks on either.
- Commit style is already **gitmoji-derived**: `[ :construction_worker: ] | create: Dockerfile (docker::image)` — good, but informal.
- **Husky wiring is broken** (verified):
  - Root `.husky/pre-commit` runs `npx lint-staged` at the repo root, but `.lintstagedrc.json` lives in `customer_frontend/` — lint-staged finds **no config** at the hook's CWD.
  - Even if found: `"npm run lint:staged"` = `eslint --fix;` — with a trailing `;`, no file list is passed, and staged filenames never reach eslint (lint-staged cannot inject files through `npm run`).
  - Root `package.json` (husky bootstrap) vs. app scripts: **stray root manifest** — the app's scripts are all in `customer_frontend/`.
- **ESLint 9 config mismatch** (verified): file is `eslintrc.config.js`, in **legacy** `.eslintrc` format (`module.exports = { env, extends, parser, parserOptions }`), but ESLint 9 auto-discovers **flat** `eslint.config.js` only. Also, `@typescript-eslint/parser`/`plugin` are **not in devDependencies**. Net effect: `npm run lint:fix` is broken or inert today — CI would run against an unconfigured linter.
- **Broken scripts** (verified in `package.json`):
  - `code:ci` calls `npm run test:coverage` — **script does not exist** → `npm run code:ci` fails.
  - `db:sync` calls `prisma` — **not in dependencies** → fails.
  - `docs::swagger` calls `./swagger.js` — **file missing** → fails.

### 5.2 Proposed flow: GitHub Flow (simplified)

For a single-app repo, `main` + `develop` adds ceremony without benefit; **GitHub Flow** is the recommended simplification — `main` is always deployable, every change ships via a feature branch + PR.

```
main (protected, always deployable)
  └── feature/xxx ──PR──▶ main   (CI must pass; at least 1 approval)
```

Keep `develop` only if you genuinely want a shared integration branch; if kept: never deploy from it, require PRs into it too.

### 5.3 Branch protection (repo settings — enforce on `main`)

- ✅ Require pull request before merging; require ≥ 1 approval
- ✅ Require status checks: `quality`, `test`, `docker` (the jobs in §6)
- ✅ Require branches up to date (keeps `main` green by forcing rebase)
- ⚠️ Optionally: require linear history; **never** allow force-push to `main`
- ✅ Enable Dependabot (npm + Docker) — this repo currently has none

### 5.4 PR template (propose as `.github/PULL_REQUEST_TEMPLATE.md`)

```markdown
## Summary
_What and why, one paragraph._

## Changes
- [ ] Feature / [ ] Fix / [ ] Refactor / [ ] Infra / [ ] Docs

## Verification
- [ ] `npm run lint` passes
- [ ] `npm run typecheck` passes
- [ ] `npm run test` passes (coverage %: ___)
- [ ] E2E: ___
- [ ] Docker image builds and `/health` responds

## Blast radius
_Which environments/services are affected by this change? Rollback plan if any._

## Screenshots / logs
```

### 5.5 Commit conventions (formalize the existing gitmoji style)

Keep the current format; make it explicit:

```
[ :emoji: ] | <action>: <subject> (<scope>)

examples:
[ :construction_worker: ] | create: Dockerfile (docker::image)
[ :bug: ] | fix: missing null-check on ticket query (app::api)
[ :white_check_mark: ] | add: jest coverage thresholds (test::unit)
```

Rules: one logical change per commit; subject ≤ 72 chars; scopes from a fixed list (`app::*`, `test::*`, `docker::*`, `ci::*`, `docs::*`, `hook`); no unrelated formatting in the same commit. Optionally enforce with `commitlint` (`@commitlint/config-conventional` + a gitmoji-friendly rule set) in a `commit-msg` hook — defer until the pre-commit wiring is fixed, since hooks are currently dead.

---

## 6. CI/CD pipeline proposal (GitHub Actions)

### 6.1 Required script additions first (`customer_frontend/package.json`)

CI must **check**, not fix. The current scripts are fix-oriented (`lint:fix`, `format:fix`) and `test` hides an empty suite. Add:

```jsonc
"lint": "eslint ./src/**/*.{ts,js}",
"lint:check": "eslint ./src/**/*.{ts,js} --max-warnings 0",
"typecheck": "tsc --noEmit",
"test:ci": "jest --ci --coverage --passWithNoTests --color",
"cypress:run": "cypress run"   // requires cypress.config.ts to be created first
```

Also fix: `code:ci` → `"npm run format:check && npm run lint:check && npm run test:ci"` (it currently references the missing `test:coverage`). Migrate ESLint to flat config (`eslint.config.js`) + add `@typescript-eslint/parser`/`@typescript-eslint/eslint-plugin` to devDependencies, or lint will never actually run. Fix the husky/lint-staged wiring per §5.1. Add `coverageThreshold` to `jest.config.ts` once real tests exist (80% lines is a sane start).

### 6.2 Workflow — full YAML (proposed: `.github/workflows/ci.yml`)

> Documented only — **not created** in this task.

```yaml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

defaults:
  run:
    working-directory: customer_frontend

jobs:
  quality:
    name: Lint · Format · Types
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
          cache-dependency-path: customer_frontend/package-lock.json
      - run: npm ci
      - run: npm run lint:check
      - run: npm run format:check
      - run: npm run typecheck

  test:
    name: Unit tests (Jest)
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
          cache-dependency-path: customer_frontend/package-lock.json
      - run: npm ci
      - run: npm run test:ci
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: coverage
          path: customer_frontend/coverage

  e2e:
    name: E2E (Cypress)
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
          cache-dependency-path: customer_frontend/package-lock.json
      - run: npm ci
      - name: Cache Cypress binary
        uses: actions/cache@v4
        with:
          path: ~/.cache/Cypress
          key: cypress-${{ runner.os }}-${{ hashFiles('customer_frontend/package-lock.json') }}
      - run: npx cypress verify
      - run: npm run build
      - run: npm run cypress:run
      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: cypress-screenshots
          path: customer_frontend/cypress/screenshots

  docker:
    name: Build & push image (GHCR)
    if: github.event_name == 'push' && (github.ref == 'refs/heads/main' || github.ref == 'refs/heads/develop')
    needs: [quality, test]
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - run: docker build --target production --check  # early Dockerfile lint (buildx)
      - uses: docker/build-push-action@v6
        with:
          context: customer_frontend
          push: true
          tags: |
            ghcr.io/samuel-ricardo/customer_support_frontend:develop
            ghcr.io/samuel-ricardo/customer_support_frontend:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
      # main gets :latest on merge — add a second job (or tag rule) for main:
      # tags: ghcr.io/samuel-ricardo/customer_support_frontend:latest + :<sha>

  deploy-dev:
    name: Deploy to dev
    if: github.ref == 'refs/heads/develop'
    needs: [docker]
    runs-on: ubuntu-latest
    steps:
      - name: Pull & restart on dev host
        uses: appleboy/ssh-action@v1   # third-party — audit before adopting; or use plain ssh + run
        with:
          host: ${{ secrets.DEV_HOST }}
          username: ${{ secrets.DEV_USER }}
          key: ${{ secrets.DEV_SSH_KEY }}
          script: |
            docker login ghcr.io -u ${{ github.actor }} -p ${{ secrets.GITHUB_TOKEN }}
            docker pull ghcr.io/samuel-ricardo/customer_support_frontend:develop
            cd /opt/customer_frontend && docker compose up -d frontend
            docker image prune -f
```

**Decisions embedded in the YAML (state them in review):**

- `quality` + `test` + `e2e` gate every PR; **`docker` only on push to `main`/`develop`** (PR images are throwaway and slow the loop; build-on-PR can be added later as a non-push job).
- Image tags: `:develop` (rolling), `:<sha>` (pinned), `:latest` (main). Deployments consume **digest-pinned** tags ideally.
- GH Actions secrets used: `DEV_HOST`, `DEV_USER`, `DEV_SSH_KEY` (self-host container flow). If Vercel/Netlify is chosen instead (§7), swap `deploy-dev` for `vercel-action`/`netlify-cli` with `VERCEL_TOKEN`/`NETLIFY_AUTH_TOKEN` + project IDs.
- `appleboy/ssh-action` is a widely used third-party action — pin to a full commit SHA and audit, or replace with a plain `ssh` step reading the same secrets.

---

## 7. Deployment targets (SolidStart presets)

Verified against official SolidStart `defineConfig` docs (docs.solidjs.com/solid-start) and vinxi CLI docs:

- SolidStart uses **Nitro** under the hood; the `server.preset` option selects the target platform.
- Supported (non-exhaustive): `node` / `node-server` (default) · `deno_server` · `bun` · `netlify`, `netlify-edge` · `vercel`, `vercel-edge` · `aws_lambda` · `cloudflare`, `cloudflare_pages`, `cloudflare_module` · `deno_deploy`.
- CLI/CI alternative: `vinxi build --preset <name>` or `SERVER_PRESET=<name> vinxi build`.

### 7.1 Current app.config.ts — make the default explicit

```ts
import { defineConfig } from "@solidjs/start/config";

export default defineConfig({
  server: {
    preset: "node-server", // self-host / Docker — current runtime actually relies on this being the implicit default
  },
});
```

This is the preset the Docker image assumes (`vinxi start` serves `.output/`). Pin it explicitly so a future config reorg cannot silently switch the build target.

### 7.2 Platform branches (choose per environment)

| Target | `preset` | Notes |
|---|---|---|
| **Self-host / Docker (this repo's path)** | `node-server` | Build `.output/`; run with node or `vinxi start`; ship via GHCR + compose/k8s. |
| Vercel | `vercel` (or `vercel-edge`) | Auto-detected by the platform; requires `vercel.json`/project link; async local storage supported out of the box. |
| Netlify | `netlify` (or `netlify-edge`) | Same story; `.netlify` already gitignored. |
| Cloudflare | `cloudflare_pages` / `cloudflare_module` | **Extra config required** — SolidStart docs: `rollupConfig: { external: ["__STATIC_CONTENT_MANIFEST", "node:async_hooks"] }`. |

### 7.3 Env var & secrets strategy (already half-right — complete it)

- ✅ `.env` / `.env*.local` are gitignored — **never commit env files**; add `.env*.example` (documented keys, dummy values) so local onboarding works.
- Runtime secrets flow: local `docker compose --env-file .env` → orchestrator secrets (k8s `Secret`/cloud secrets manager) in prod. **Never** `ENV`/`ARG` secrets in the Dockerfile — build args are visible in image history.
- CI secrets live in **GitHub Actions secrets** (`DEV_HOST`, `DEV_USER`, `DEV_SSH_KEY`, `VERCEL_TOKEN`/`NETLIFY_AUTH_TOKEN` if adopted) and are referenced only as `${{ secrets.X }}` — never written into YAML or logs.
- Public/build-time values (e.g. `BUILD_SHA` for the health payload) can be injected as build args/labels safely.

---

## 8. Observability

### 8.1 `/health` endpoint (proposal — implement as a SolidStart route)

```tsx
// src/routes/health.tsx  (SolidStart route; JSON API route)
export function GET() {
  return new Response(
    JSON.stringify({
      status: "ok",
      version: process.env.npm_package_version ?? "unknown",
      build: process.env.BUILD_SHA ?? "unknown",
      uptime: process.uptime(),
      ts: new Date().toISOString(),
    }),
    {
      status: 200,
      headers: { "content-type": "application/json" },
    }
  );
}
```

Wired to: Docker `HEALTHCHECK` (§2.2), compose healthcheck (§4.2), and external uptime probes. Keep it dependency-free (no db/network calls) or it becomes the thing that fails first in an incident.

### 8.2 Logging

- Current: zero logging configuration — SolidStart/Nitro default output to stdout, which Docker captures. Minimum viable: **structured JSON logs** via `pino` (+ `pino-http` for request logs) attached in `entry-server.tsx`, consumed with `docker logs <container> --tail`.
- Never log secrets/PII; add a request-id header through middleware for correlating `docker logs` with traces later.
- Log rotation is the host's job (docker json-file `max-size`/`max-file` options or a log driver).

### 8.3 Uptime

- Container level: `restart: unless-stopped` + `HEALTHCHECK` (see §4.2).
- External level (pick one, keep it simple): **Uptime Kuma** (self-host, probe `/health` every 60 s) or UptimeRobot/Grafana Cloud Synthetics (managed). Alert on 2 consecutive failures.
- Optional: a scheduled GitHub Actions ping (`cron: '*/10 * * * *'`, `workflow_dispatch` too) hitting the public `/health` and failing the run on 5xx — free synthetic check with zero extra infra, though it cannot page anyone by itself.

---

## 9. Prioritized backlog

| # | Priority | Item | Where |
|---|---|---|---|
| 1 | **P0** | Create `.dockerignore` (content in §3) | `customer_frontend/.dockerignore` |
| 2 | **P0** | Remove `COPY --from=build /home/node/app/ ./` (line 28); prune dev deps in prod stage; drop `.vinxi` copy; add `NODE_ENV=production`, `EXPOSE`, `HEALTHCHECK` | Dockerfile |
| 3 | **P0** | Add CI workflow (this doc §6.2) + fix broken scripts (`test:coverage` missing, `code:ci`), add `lint`/`lint:check`/`typecheck`/`test:ci` | `.github/workflows/ci.yml` + `package.json` |
| 4 | **P0** | Reset ESLint to ESLint 9 flat config (`eslint.config.js`) or drop to legacy filename; add `@typescript-eslint/*` devDeps — lint is inert today | repo |
| 5 | **P1** | Fix husky/lint-staged wiring: move `.lintstagedrc.json` to repo root (or `cd customer_frontend` in hook + `--config`), remove trailing `;` in `lint:staged`, pass staged files correctly | repo root |
| 6 | **P1** | `preset: "node-server"` explicit in `app.config.ts`; add `.env.example` | app config |
| 7 | **P1** | Branch protection on `main` (status checks from §6) + enable Dependabot | GitHub settings |
| 8 | **P1** | Implement `/health` route; wire compose healthcheck + `restart: unless-stopped` + `env_file` | app + compose |
| 9 | **P2** | Pin base image minor + digest; Dependabot for Docker | Dockerfile |
| 10 | **P2** | Add `cypress.config.ts` + first E2E spec (CI job is blocked on this) | app |
| 11 | **P2** | Go GitHub Flow (drop `develop`) or formalize develop rules; PR template §5.4 | repo |
| 12 | **P2** | Jest `coverageThreshold` (80%) once tests exist; remove `--passWithNoTests` | `jest.config.ts` |
| 13 | **P2** | Structured logging (pino) + request-ids | app |
| 14 | **P3** | Fix stray scripts: `db:sync` (prisma not installed), `docs::swagger` (`swagger.js` missing) — wire or delete | `package.json` |
| 15 | **P3** | SBOM + artifact signing (cosign) on pushed images; digest-pinned deploys | CI |

---

## 10. Appendix — verification log

| Claim | Verified by |
|---|---|
| No `.github/workflows` | `git ls-files` (repo-wide) shows no `.github` path |
| No `.dockerignore` | repo-wide search: 0 matches |
| Dockerfile content (33 lines, incl. red-flag line 28, commented openssl + CMDs) | read `customer_frontend/Dockerfile` |
| compose content (6 lines, no restart/env/healthcheck) | read `customer_frontend/docker-compose.yaml` |
| `app.config.ts` = `defineConfig({})` | read |
| `.gitignore` covers `.env*`, `.output`, `.vinxi`, `node_modules` | read `customer_frontend/.gitignore` |
| Local `node_modules/`, `.output/`, `.vinxi/` present on disk | `Get-ChildItem` |
| `vinxi` in `dependencies` (start survives `--omit=dev`), Cypress in devDeps | read `package.json` |
| `test:coverage` script missing while `code:ci` references it | read `package.json` |
| Root husky bootstrap + root `.husky/pre-commit` = `npx lint-staged`; lint-staged config in subdir | read root `package.json`, `.husky/pre-commit`, `customer_frontend/.lintstagedrc.json` |
| ESLint config legacy format + missing `@typescript-eslint` devDeps | read `eslintrc.config.js`, `package.json` |
| No test files, no `cypress.config.*`, no coverage threshold | `git ls-files` + `Get-ChildItem -Recurse` + `jest.config.ts` |
| Presets: `server.preset` options, `SERVER_PRESET` env, `.output` runtime dir | docs.solidjs.com `defineConfig`, vinxi CLI docs (2026-08-19) |

**Next step:** implement P0 items 1–4 first — they unblock every other improvement and turn `docker build` + CI into trustworthy signals.