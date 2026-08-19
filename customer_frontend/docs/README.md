# Documentation Hub — Customer Support Frontend

This folder holds the complete, deep analysis of the `customer_frontend` project, produced by a six-specialist review team (solution architecture, full-stack development, QA, UX, DevOps, and business analysis). Each document is evidence-based: claims cite `file:line` references and were verified against the installed packages and runtime behavior — not assumed.

## Reading Guide

| If you want to understand… | Read |
|---------------------------|------|
| What this product is and where it's going | [06-product-analysis.md](06-product-analysis.md) |
| How the framework works (SSR, reactivity, routing) | [01-architecture.md](01-architecture.md) |
| Every file in `src/` and the configs, line by line | [02-codebase.md](02-codebase.md) |
| Why lint/test/format are all broken — with proof | [03-quality.md](03-quality.md) |
| What's wrong with the UI and how to design the real product | [04-ux-design.md](04-ux-design.md) |
| Docker defects, CI/CD pipeline, deployment targets | [05-devops.md](05-devops.md) |
| Where to start fixing (priority order) | [02-codebase.md](02-codebase.md) (P0s) + [03-quality.md](03-quality.md) |

## Document Index

| File | Title | Audience | Key Takeaway |
|------|-------|----------|--------------|
| [01-architecture.md](01-architecture.md) | Architecture & ADRs | Architects, devs | SolidStart SSR + fine-grained reactivity; 5 ADRs ready to adopt |
| [02-codebase.md](02-codebase.md) | Codebase Deep Review | Devs | Clean code, broken toolchain: 16 findings (F1–F16), 6 P0s |
| [03-quality.md](03-quality.md) | QA & Quality Audit | QA, devs, leads | Verdict **REJECTED**: no guardrail runs; move to Vitest + Cypress |
| [04-ux-design.md](04-ux-design.md) | UX & Design System | Designers, devs | WCAG 2.1 AA audit, design tokens, IA, user flows |
| [05-devops.md](05-devops.md) | DevOps & Deployment | DevOps, devs | Dockerfile/compose fixes, GitHub Actions YAML, presets |
| [06-product-analysis.md](06-product-analysis.md) | Product Analysis & Roadmap | PO, PM, stakeholders | Maturity 1.1/5; MVP roadmap with RICE priorities |

## Cross-Reference Map

```
06-product-analysis ── roadmap/features ──► 04-ux-design (IA, flows, components)
        │                                        │
        └── maturity scoring ◄── evidence from ──┴── 02-codebase (findings)
                                                        │
02-codebase ── P0 pipeline repair ──► 03-quality (toolchain verdict + Vitest plan)
        │                                    │
        └── Docker P0s ──► 05-devops (Dockerfile line 28, .dockerignore, CI YAML)
        │
        └── architecture seams ──► 01-architecture (ADRs, data layer, folder strategy)
```

The documents are **complementary**: e.g. `03-quality` proves the toolchain failures that `02-codebase` catalogs as findings; `04-ux-design` builds the UI strategy that `06-product-analysis` scopes as MVP epics; `05-devops` proposes the CI that `03-quality` requires as a gate.

## Verified Ground Truth (June 2026 audit)

- Stack: solid-js **1.8.22**, @solidjs/start **1.0.6**, @solidjs/router **0.14.3**, vinxi **0.4.2**, nitro **2.9.7**, Tailwind 3.4.3
- Branches: `main` + `develop` (work happens on `develop`); 21 gitmoji commits
- Zero tests, zero CI workflows, zero Cypress specs
- Only static asset: `public/favicon.ico` (no images to preserve yet — future screenshots go in `public/` or `docs/assets/`)

## Maintainers' Notes

- **Update on change**: when a source file, config, or script changes, update the affected doc(s) in the same PR. `02-codebase.md` and `03-quality.md` are the most sensitive to drift.
- **Diagrams**: Mermaid — regenerate/update when the architecture or flows change.
- **Evidence discipline**: keep the "verified vs assumed" distinction — mark anything you could not run as `[blocked]` or hypothesis, as the original audit does.
- **Adding docs**: continue the `NN-topic.md` numbering; always add a row to this index and to the cross-reference map.
- **Images**: if adding screenshots/diagrams, place them in `docs/assets/` and reference them with relative paths (e.g. `![alt](assets/screenshot.png)`) so they render on GitHub.
