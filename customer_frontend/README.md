# Customer Support Frontend

Self-hostable customer support portal built with **SolidStart** — the SSR-first SolidJS framework. Currently at the foundation stage (SolidStart boilerplate with scaffolded tooling); the product roadmap toward a full support platform is documented in [`docs/06-product-analysis.md`](docs/06-product-analysis.md).

> Repo layout note: this application lives in the `customer_frontend/` subdirectory of the `customer_support_frontend` repository (the repo root hosts the shared git hooks).

## Tech Stack

| Layer | Technology | Version |
|-------|-----------|---------|
| Framework | SolidJS + SolidStart | 1.8.22 / 1.0.6 |
| Routing | @solidjs/router (file-based) | 0.14.3 |
| Bundler | Vinxi (Nitro runtime) | 0.4.2 / 2.9.7 |
| Styling | Tailwind CSS + PostCSS | 3.4.3 |
| Language | TypeScript (strict) | ESNext |
| Unit tests | Jest + ts-jest (planned: Vitest) | 29.7 |
| E2E tests | Cypress (installed, not configured) | 13.14 |
| Linting / Format | ESLint 9 / Prettier 3 (both misconfigured — see below) | 9.10 / 3.3 |
| Git hooks | Husky 9 + lint-staged (wiring broken) | 9.1 |
| Containers | Docker multi-stage (node:20-slim) + docker-compose | — |

## Quick Start

Prerequisites: **Node >= 18** and npm.

```bash
# install dependencies (from customer_frontend/)
npm install

# development server with HMR (Vinxi)
npm run start:dev

# production build
npm run build

# start production server (after build)
npm run start

# run tests (currently zero tests — suite passes with --passWithNoTests)
npm test
```

> ⚠️ The quality toolchain is currently **broken in multiple layers** (ESLint, Prettier, Jest install, Husky wiring, CI scripts). See [Tooling Status](#tooling-status) and [`docs/03-quality.md`](docs/03-quality.md) for the evidence and repair plan.

## Project Structure

```
customer_frontend/
├── src/
│   ├── entry-client.tsx        # Client hydration mount
│   ├── entry-server.tsx        # SSR handler + HTML shell
│   ├── app.tsx                 # Router root layout (Nav + Suspense + FileRoutes)
│   ├── app.css                 # Tailwind directives + dark-mode CSS vars
│   ├── global.d.ts             # SolidStart env types
│   ├── components/
│   │   ├── Nav.tsx             # Top navigation (active-link styling)
│   │   └── Counter.tsx         # createSignal demo component
│   └── routes/                 # File-based routing
│       ├── index.tsx           # Home ("Hello world" demo)
│       ├── about.tsx           # About page
│       └── [...404].tsx        # Catch-all not-found route
├── public/                     # Static assets (favicon.ico, future images)
├── docs/                       # Full analysis documentation (see index below)
├── app.config.ts               # SolidStart config (no deployment preset yet)
├── tailwind.config.cjs         # Tailwind theme
├── jest.config.ts              # Jest config (needs repair)
├── eslintrc.config.js          # ESLint config (NOT auto-detected in v9)
├── Dockerfile                  # Multi-stage Docker build
└── docker-compose.yaml         # Frontend service on port 3000
```

## Feature Status

| Area | Status | Roadmap |
|------|--------|---------|
| SSR + hydration | ✅ Working (boilerplate) | — |
| File-based routing | ✅ Working (3 routes) | → `/tickets`, `/kb`, `/status` |
| Responsive styling | ✅ Tailwind scaffolded | → design system + dark mode fix |
| Tests | ❌ Zero (toolchain broken) | Vitest + Cypress E2E |
| Ticket intake | ⬜ Not started | **MVP** (4–6 weeks) |
| Agent workspace | ⬜ Not started | **v1** (6–8 weeks) |
| Realtime + insights | ⬜ Not started | **v2** (8+ weeks) |

Full roadmap, epics and RICE prioritization: [`docs/06-product-analysis.md`](docs/06-product-analysis.md).

## Documentation

The `docs/` folder contains a complete, deep analysis of this project produced by a six-specialist review (architecture, code, quality, UX, DevOps, product):

| Doc | Topic | Key Takeaway |
|-----|-------|--------------|
| [01-architecture.md](docs/01-architecture.md) | SolidStart architecture, rendering model, 5 ADRs | Fine-grained reactivity + SSR; ADR-ready foundations |
| [02-codebase.md](docs/02-codebase.md) | File-by-file code review, findings F1–F16 | Code is clean; toolchain is 100% broken (6 P0s) |
| [03-quality.md](docs/03-quality.md) | QA audit: verdict **REJECTED** | No quality guardrail actually runs; adopt Vitest |
| [04-ux-design.md](docs/04-ux-design.md) | UX audit, WCAG 2.1 AA, design system | Dark mode broken (2.04:1); 4 P0 UX fixes |
| [05-devops.md](docs/05-devops.md) | Docker, CI/CD, deployment presets | Dockerfile line 28 defect; no .dockerignore; CI pipeline proposal |
| [06-product-analysis.md](docs/06-product-analysis.md) | Product intent, roadmap, RICE | Maturity ≈1.1/5: infra-complete, product-empty |

Start with [`docs/README.md`](docs/README.md) — the documentation hub with a reading guide.

## Architecture in Brief

SolidStart renders the app on the server (Nitro) and hydrates on the client: the router root (`src/app.tsx`) composes `Nav` + `Suspense` + `FileRoutes`, each route is lazy-loaded, and reactivity is signal-based (no virtual DOM — `createSignal` updates only the affected DOM nodes). A future data layer can plug into the existing `Suspense` seam via `createResource`. Diagrams and 5 proposed ADRs: [`docs/01-architecture.md`](docs/01-architecture.md).

## Tooling Status

| Guardrail | Intended | Actual |
|-----------|----------|--------|
| ESLint 9 | Lint on commit/CI | **Dead** — config filename not auto-detected; `@typescript-eslint/*` not installed |
| Prettier 3 | Format on commit | **Dead** — `prettierrc.js` (no dot) ignored; runs with defaults |
| Lint/format globs | `./src/**/*.{ts,js}` | **No-op** — matches zero files (all sources are `.tsx`) |
| Jest | Unit tests | **Broken** — corrupt `jest-cli` install; `node` env wrong for TSX/CSS; zero tests |
| Husky pre-commit | lint-staged | **Broken** — hook at repo root, config in subdir; `npx` fails |
| `code:ci` / `test:docker` | CI + Docker gate | **Broken** — reference missing `test:coverage`; watch-mode hang |
| Docker build | `npm run build:docker` | **Fails today** — runs the broken test pipeline |
| Git hygiene | gitmoji conventions | ✅ Solid — 21 clean commits, `.env` ignored, no secrets tracked |

Repair plan (P0, prioritized): [`docs/03-quality.md`](docs/03-quality.md) and [`docs/02-codebase.md`](docs/02-codebase.md).

## Docker

```bash
# build the image (runs tests + build — currently blocked by broken test pipeline)
docker compose build

# run the app on http://localhost:3000
docker compose up
```

Known defects (see [`docs/05-devops.md`](docs/05-devops.md)): no `.dockerignore` (build context ships `node_modules`, `.git`), Dockerfile line 28 copies the entire build stage into production, production image keeps dev dependencies. All fixes proposed in the doc.

## Git Workflow

- **Branches**: `main` (releases) + `develop` (integration); PRs target `develop`.
- **Commit style**: gitmoji convention, e.g. `[:construction_worker:] create: Dockerfile (docker::image)`.
- **Hooks**: Husky pre-commit → lint-staged (currently broken — repair per docs/03).

## Roadmap Highlights

- **MVP (4–6 weeks)**: ticket submission + list + detail + status (RICE 9.6 — top priority); knowledge base; anonymous tracking.
- **v1 (6–8 weeks)**: agent workspace, authentication, KB management, SLA views.
- **v2 (8+ weeks)**: realtime chat, analytics, integrations.

RICE table and success metrics: [`docs/06-product-analysis.md`](docs/06-product-analysis.md).

## License & Authorship

Developed by **Samuel-Ricardo** (Avanade). Remote repository: [github.com/Samuel-Ricardo/customer_support_frontend](https://github.com/Samuel-Ricardo/customer_support_frontend).
