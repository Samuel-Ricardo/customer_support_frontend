# customer_support_frontend

Repository for the **Customer Support Frontend** — a self-hostable customer support portal built with SolidStart (SolidJS + SSR + TailwindCSS).

## Repository Layout

```
customer_support_frontend/
├── customer_frontend/      # The application (SolidStart app + all docs)
│   ├── src/                # Source: app shell, components, file-based routes
│   ├── docs/               # Full project analysis (architecture, code, QA, UX, DevOps, product)
│   ├── Dockerfile          # Multi-stage Docker build
│   └── docker-compose.yaml # Frontend service on port 3000
├── .husky/                 # Git hooks (pre-commit → lint-staged)
└── .gitignore
```

## Getting Started

```bash
cd customer_frontend
npm install
npm run start:dev   # dev server with HMR
```

Prerequisites: Node >= 18.

## Documentation

Start at [customer_frontend/README.md](customer_frontend/README.md), then use the [docs hub](customer_frontend/docs/README.md) for the deep-dive analysis: architecture, codebase review, quality audit, UX design system, DevOps/deployment, and product roadmap.

## Git Workflow

- Branches: `main` (releases) + `develop` (integration).
- Commits: gitmoji convention, e.g. `[:construction_worker:] create: Dockerfile (docker::image)`.

## Author

Samuel-Ricardo (Avanade) — [github.com/Samuel-Ricardo](https://github.com/Samuel-Ricardo)