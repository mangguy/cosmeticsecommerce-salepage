# Phase 10 — Documentation Plan

**Project:** Cosmetics E-Commerce Platform (Online Perfume Store)
**Version:** 1.0
**Date:** 2026-07-02
**Status:** Approved

---

## 1. Purpose of This Document

This plan defines every documentation artifact the project produces: what it is for, who reads it, who owns it, where it lives, in what format, and when in the lifecycle it is written and updated. It also maps the existing phase documents (01–09) to standard artifact types, so nothing is written twice.

Guiding principle: **docs-as-code**. All documentation is plain Markdown (with Mermaid diagrams) stored in Git alongside the code it describes. Rationale:

- **Versioned with the code.** A doc change ships in the same PR as the code change it describes, so docs cannot silently drift from reality without a reviewer noticing.
- **Reviewable.** Docs go through the same PR review process as code — typos, inaccuracies, and stale sections get caught in review.
- **Diffable and searchable.** Plain text works with `git blame`, `grep`, and IDE search. Binary formats (Word, Visio) do not.
- **Zero-cost tooling.** Markdown renders on GitHub, GitHub Pages, and every IDE. Mermaid renders natively on GitHub — no diagram export/import cycle, no stale PNGs.
- **No vendor lock-in.** No licenses, no proprietary formats, no external wiki that dies when a subscription lapses.

Published location: the `docs/` folder in `cosmeticsecommerce-salepage` is served via **GitHub Pages**, making the full document set browsable without cloning the repo.

---

## 2. Documentation Inventory Overview

| # | Artifact | Repo | Path | Status |
|---|----------|------|------|--------|
| 1 | PRD | salepage | `docs/01-business-analysis.md` (§ product requirements) | Done (Phase 01) |
| 2 | BRD | salepage | `docs/01-business-analysis.md` (§ business requirements) | Done (Phase 01) |
| 3 | SRS | salepage | `docs/02-system-analysis.md` | Done (Phase 02) |
| 4 | User Stories | salepage | `docs/02-system-analysis.md` (§ user stories) | Done (Phase 02) |
| 5 | Functional Spec | salepage | `docs/02-system-analysis.md`, `docs/05-backend-design.md`, `docs/06-frontend-design.md` | Done (Phases 02/05/06) |
| 6 | Technical Spec | salepage | `docs/03-system-architecture.md`, `docs/05-backend-design.md` | Done (Phases 03/05) |
| 7 | API Documentation | server | Auto-generated Swagger UI at `/api/v1/docs` | Continuous |
| 8 | Database Documentation | salepage | `docs/04-database-design.md` + `server/prisma or /migrations` | Done (Phase 04) + continuous |
| 9 | Architecture Documentation | salepage | `docs/03-system-architecture.md` | Done (Phase 03) |
| 10 | Deployment Guide | salepage | `docs/09-devops-deployment.md` + per-repo `README.md` | Done (Phase 09) |
| 11 | Development Guide | each repo | `README.md` + `CONTRIBUTING.md` (setup section) | Per repo, at repo creation |
| 12 | Contribution Guide | each repo | `CONTRIBUTING.md` | Per repo, at repo creation |
| 13 | ADRs | salepage | `docs/adr/ADR-NNN-*.md` | Continuous, per decision |
| 14 | Use Case Diagram | salepage | `docs/02-system-analysis.md` (Mermaid) | Done (Phase 02) |
| 15 | Activity Diagrams | salepage | `docs/02-system-analysis.md` (Mermaid) | Done (Phase 02) |
| 16 | Class Diagram | salepage | `docs/05-backend-design.md` (Mermaid) | Done (Phase 05) |
| 17 | Sequence Diagrams | salepage | `docs/03-system-architecture.md`, `docs/05-backend-design.md` (Mermaid) | Done (Phases 03/05) |
| 18 | ER Diagram | salepage | `docs/04-database-design.md` (Mermaid `erDiagram`) | Done (Phase 04) |

---

## 3. Artifact Details

### 3.1 Product Requirements Document (PRD)

- **Purpose:** Defines *what* the product is and *why* — target users (guest, customer, admin), core value proposition (curated perfume storefront), feature set (browse/search, product detail, cart, checkout with mock payment, order tracking, reviews, wishlist; full admin back office), and success criteria.
- **Audience:** Everyone on the project; the first document a new contributor reads.
- **Owner:** Product owner / project lead.
- **Format/Tool:** Markdown in repo.
- **Location:** `cosmeticsecommerce-salepage/docs/01-business-analysis.md`.
- **Lifecycle:** Written in Phase 01 (planning). Updated only when scope changes; scope changes require an ADR referencing the PRD section changed.

### 3.2 Business Requirements Document (BRD)

- **Purpose:** Business-level goals, stakeholders, constraints (university project, zero-cost hosting tiers, mock payment instead of a real gateway), and non-goals.
- **Audience:** Stakeholders, graders/reviewers, project lead.
- **Owner:** Product owner.
- **Format/Tool:** Markdown.
- **Location:** `cosmeticsecommerce-salepage/docs/01-business-analysis.md` (business requirements sections).
- **Lifecycle:** Phase 01; frozen after approval, amended only via change request noted in the roadmap.

### 3.3 Software Requirements Specification (SRS)

- **Purpose:** Formal functional requirements (FR-xx) and non-functional requirements (NFR-xx: performance, security, availability, usability) with traceability to features and roles.
- **Audience:** Developers, testers (test cases trace back to FR/NFR IDs).
- **Owner:** System analyst / tech lead.
- **Format/Tool:** Markdown with requirement ID tables.
- **Location:** `cosmeticsecommerce-salepage/docs/02-system-analysis.md`.
- **Lifecycle:** Phase 02. Updated when a requirement changes; each API/feature PR should reference the FR it implements.

### 3.4 User Stories

- **Purpose:** Requirements framed from the user's perspective (`As a customer, I want to save products to a wishlist so that I can buy them later`) with acceptance criteria; drives sprint/workshop planning.
- **Audience:** Developers, testers.
- **Owner:** Product owner, refined by the team.
- **Format/Tool:** Markdown tables grouped by role (guest / customer / admin).
- **Location:** `cosmeticsecommerce-salepage/docs/02-system-analysis.md`; working copies may be mirrored as GitHub Issues per workshop.
- **Lifecycle:** Phase 02 for the initial backlog; refined at the start of each workshop (see `11-roadmap.md`).

### 3.5 Functional Specification

- **Purpose:** Precise behavior per feature: inputs, outputs, validation rules, edge cases, error states (e.g., checkout stock-check behavior, coupon validation rules, review moderation flow).
- **Audience:** Developers and testers.
- **Owner:** Tech lead.
- **Format/Tool:** Markdown.
- **Location:** Split by concern — flows in `docs/02-system-analysis.md`, API behavior in `docs/05-backend-design.md`, UI behavior in `docs/06-frontend-design.md`.
- **Lifecycle:** Phases 02/05/06; updated in the same PR as any behavior change.

### 3.6 Technical Specification

- **Purpose:** *How* the system is built: three-repo topology, NestJS module layout, Supabase (PostgreSQL/Auth/Storage) integration, JWT flow, Docker/Render/Railway deployment, Vercel + GitHub Pages hosting, Nuxt 3 + Pinia dashboard architecture.
- **Audience:** Developers, reviewers.
- **Owner:** Tech lead.
- **Format/Tool:** Markdown + Mermaid (C4-style context/container diagrams).
- **Location:** `cosmeticsecommerce-salepage/docs/03-system-architecture.md` (system level) and `docs/05-backend-design.md` (backend detail).
- **Lifecycle:** Phase 03/05; updated whenever an ADR changes the architecture.

### 3.7 API Documentation (Swagger / OpenAPI)

- **Purpose:** Complete, always-current reference for every `/api/v1` endpoint: request/response schemas, auth requirements, error codes, and a live "try it" console.
- **Audience:** Frontend developers (salepage + dashboard), testers, external reviewers.
- **Owner:** Backend developers — collectively, because it is generated from their code.
- **Format/Tool:** **Auto-generated OpenAPI 3 spec** via `@nestjs/swagger` decorators (`@ApiTags`, `@ApiOperation`, `@ApiResponse`, `@ApiBearerAuth`) on controllers and DTOs. Served as Swagger UI at `GET /api/v1/docs` and raw JSON at `/api/v1/docs-json`.
- **Why generated, not hand-written:**
  1. **Cannot drift.** Hand-written API docs are stale the day after the second endpoint changes. Generated docs are derived from the same DTO classes and validation decorators (`class-validator`) that the runtime actually enforces — the doc *is* the contract.
  2. **Single source of truth.** Adding a field to a DTO updates validation, TypeScript types, and the published schema in one edit.
  3. **Free tooling.** The OpenAPI JSON feeds client generation, Postman collections, and contract tests with zero extra authoring effort.
  4. **Review is automatic.** The decorators are visible in every endpoint PR, so API doc review happens as code review.
  The only hand-written API prose is a short "API conventions" section (versioning, pagination format, error envelope) in `docs/05-backend-design.md`.
- **Location:** `cosmeticsecommerce-server` (decorators in source; Swagger UI served by the running API).
- **Lifecycle:** Continuous — updated automatically with every endpoint change from Workshop 3 onward.

### 3.8 Database Documentation

- **Purpose:** Schema reference: every table, column, type, constraint, index, and relationship; naming conventions; migration policy; seed data description.
- **Audience:** Backend developers, DBAs (i.e., whoever runs migrations).
- **Owner:** Backend lead.
- **Format/Tool:** Markdown + Mermaid `erDiagram`; migrations as SQL/ORM files in the server repo are the executable source of truth.
- **Location:** `cosmeticsecommerce-salepage/docs/04-database-design.md` (design + ERD); `cosmeticsecommerce-server` migrations directory (authoritative schema).
- **Lifecycle:** Phase 04 for design; the ERD in the doc is updated in the same PR as any migration that changes relationships.

### 3.9 Architecture Documentation

- **Purpose:** System context, container/component views, request flows (auth, checkout, image upload to Supabase Storage), technology choices and their boundaries.
- **Audience:** All developers; graders assessing design quality.
- **Owner:** Tech lead / architect.
- **Format/Tool:** Markdown + Mermaid (flowcharts, sequence diagrams).
- **Location:** `cosmeticsecommerce-salepage/docs/03-system-architecture.md`, with decisions recorded in `docs/adr/`.
- **Lifecycle:** Phase 03; each significant change is an ADR, and the architecture doc is amended to match.

### 3.10 Deployment Guide

- **Purpose:** Step-by-step deployment of all three apps: Docker build + Render/Railway for the API, Vercel for the Astro storefront, dashboard hosting, Supabase project setup, environment variables, GitHub Pages for docs, rollback procedure.
- **Audience:** Whoever deploys (developers, ops, future maintainers).
- **Owner:** DevOps-responsible developer.
- **Format/Tool:** Markdown with copy-pasteable commands and env-var tables (`.env.example` files in each repo are the machine-readable companion).
- **Location:** `cosmeticsecommerce-salepage/docs/09-devops-deployment.md`; repo-specific quick-deploy steps in each repo's `README.md`.
- **Lifecycle:** Phase 09 draft; validated and finalized in Workshop 12 when the pipeline actually runs.

### 3.11 Development Guide

- **Purpose:** Get a developer from `git clone` to a running local stack: prerequisites (Node version, Docker, Supabase CLI), install, env setup, run, test, and common troubleshooting.
- **Audience:** New contributors; your future self.
- **Owner:** Each repo's maintainer.
- **Format/Tool:** Markdown.
- **Location:** `README.md` in each of the three repos (repo-specific), with cross-repo local orchestration notes in `docs/09-devops-deployment.md`.
- **Lifecycle:** Written at repo creation (Workshop 1); updated whenever setup steps change. Rule: if onboarding hits a snag not covered in the README, fixing the README is part of the fix.

### 3.12 Contribution Guide

- **Purpose:** How to contribute: branch naming (`feat/`, `fix/`, `docs/`), Conventional Commits, PR template and review rules, lint/format requirements, test expectations, definition of done for a PR.
- **Audience:** All contributors.
- **Owner:** Tech lead.
- **Format/Tool:** Markdown.
- **Location:** `CONTRIBUTING.md` in each repo (identical content, kept in sync; the salepage copy is canonical).
- **Lifecycle:** Workshop 1; rarely changes.

### 3.13 Architecture Decision Records (ADRs)

- **Purpose:** Capture *why* a significant decision was made, what alternatives were rejected, and what the consequences are — so decisions survive team turnover and are not endlessly relitigated.
- **Audience:** Current and future developers.
- **Owner:** Whoever proposes the decision writes the ADR; the team accepts it in PR review.
- **Format/Tool:** Markdown, one file per decision, numbered sequentially, never deleted (superseded ADRs are marked `Superseded by ADR-NNN`).
- **Location:** `cosmeticsecommerce-salepage/docs/adr/ADR-NNN-short-title.md`.
- **Lifecycle:** Continuous. Trigger for writing one: any decision that is expensive to reverse, affects more than one repo, or a teammate might reasonably ask "why did we do it this way?"

#### ADR Template

```markdown
# ADR-NNN: <Short decision title>

- **Status:** Proposed | Accepted | Superseded by ADR-NNN
- **Date:** YYYY-MM-DD
- **Deciders:** <names/roles>

## Context
What problem are we solving? What forces are in play
(constraints, requirements, deadlines, costs)?

## Decision
What we decided, stated plainly in one or two sentences,
followed by the essential detail.

## Alternatives Considered
1. **<Alternative A>** — why rejected.
2. **<Alternative B>** — why rejected.

## Consequences
- Positive: what this buys us.
- Negative: what it costs us / new risks.
- Follow-ups: work this decision creates.
```

#### Example — ADR-001: Three-Repository Split

```markdown
# ADR-001: Split the platform into three repositories

- **Status:** Accepted
- **Date:** 2026-07-02
- **Deciders:** Tech lead, team

## Context
The platform has three deployables with different frameworks, runtimes,
hosting targets, and release cadences: a NestJS REST API (Docker on
Render/Railway), an Astro + Tailwind customer storefront (Vercel, plus
GitHub Pages for docs), and a Nuxt 3 + Pinia admin SPA. The team is
small and needs simple CI, simple deploys, and clear ownership. We must
choose between a monorepo and multiple repositories.

## Decision
Use three repositories: `cosmeticsecommerce-server`,
`cosmeticsecommerce-salepage`, `cosmeticsecommerce-dashboard`. The API
contract (OpenAPI, generated from NestJS decorators) is the only
coupling point between them. Shared documentation lives in the salepage
repo under `docs/` and is published via GitHub Pages.

## Alternatives Considered
1. **Monorepo (pnpm workspaces / Turborepo)** — enables shared TypeScript
   types across apps, but adds workspace tooling the team must learn,
   complicates per-app deploy pipelines on free hosting tiers (Vercel,
   Render each expect a repo root), and mixes three unrelated framework
   toolchains (Nest, Astro, Nuxt) in one dependency tree.
2. **Two repos (backend + one combined frontend)** — Astro (static-first
   storefront) and Nuxt (SPA dashboard) share almost no code, have
   different build outputs and hosting, and would fight over tooling in
   one repo for no benefit.

## Consequences
- Positive: each repo has a single framework, a trivially simple CI
  pipeline, an independent deploy target, and independent versioning.
  Access can be scoped per repo. Onboarding to one app requires cloning
  one repo.
- Negative: no compile-time sharing of API types; the frontends must
  consume the API via the generated OpenAPI spec (mitigation: generate
  TypeScript clients from `/api/v1/docs-json`). Cross-repo changes
  (e.g., a breaking API change) require coordinated PRs.
- Follow-ups: document the API-contract workflow in CONTRIBUTING.md;
  add OpenAPI client generation in Workshop 8.
```

### 3.14 UML & Data Diagrams

All diagrams are **Mermaid embedded in Markdown** — same docs-as-code rationale: they render on GitHub, diff in PRs, and never rot as detached image files.

| Diagram | Purpose | Audience | Location | Written / Updated |
|---------|---------|----------|----------|-------------------|
| **Use Case Diagram** | Actors (guest, customer, admin) vs. system functions; scope at a glance | Stakeholders, testers | `docs/02-system-analysis.md` | Phase 02; on scope change |
| **Activity Diagrams** | Key flows: checkout with mock payment, order fulfillment, registration | Developers, testers | `docs/02-system-analysis.md` | Phase 02; on flow change |
| **Class Diagram** | Backend domain model: entities, services, module relationships | Backend developers | `docs/05-backend-design.md` | Phase 05; on model change |
| **Sequence Diagrams** | Cross-component interactions: JWT auth via Supabase, checkout, admin order update | All developers | `docs/03-system-architecture.md`, `docs/05-backend-design.md` | Phases 03/05; on interaction change |
| **ER Diagram** | Tables and relationships (products, brands, categories, orders, order_items, reviews, wishlists, coupons, banners, users, inventory) | Backend developers | `docs/04-database-design.md` | Phase 04; same PR as schema migrations |

- **Owner:** Author of the phase document that hosts the diagram.
- **Update rule:** A diagram is updated in the same PR as the change that invalidates it. A stale diagram is treated as a doc bug.

---

## 4. Coverage Map — Phase Docs 01–09 vs. Standard Artifacts

| Phase Document | Artifacts It Covers |
|----------------|---------------------|
| `01-business-analysis.md` | PRD, BRD |
| `02-system-analysis.md` | SRS, User Stories, Functional Spec (flows), Use Case Diagram, Activity Diagrams |
| `03-system-architecture.md` | Architecture Documentation, Technical Spec (system level), Sequence Diagrams (system flows) |
| `04-database-design.md` | Database Documentation, ER Diagram |
| `05-backend-design.md` | Technical Spec (backend), Functional Spec (API behavior), Class Diagram, Sequence Diagrams (API flows), API conventions prose |
| `06-frontend-design.md` | Functional Spec (UI behavior), frontend Technical Spec |
| `07-uiux-design.md` | Design system, wireframes, UX flows (supplements Functional Spec) |
| `08-security-design.md` | Security sections of the SRS/NFRs, threat model, auth/authz spec |
| `09-devops-deployment.md` | Deployment Guide, CI/CD spec, environments, monitoring plan |

**Gaps closed by this plan (net-new artifacts):**
1. `docs/adr/` directory with the template and ADR-001 (this phase).
2. `CONTRIBUTING.md` and setup-complete `README.md` in each repo (Workshop 1 deliverable — verify done).
3. Swagger decorators on all endpoints (Workshops 3–7, enforced in PR review).

---

## 5. Maintenance Rules

1. **Docs change with code.** Any PR that changes behavior, schema, API surface, or architecture updates the affected doc/diagram in the same PR. Reviewers block PRs that skip this.
2. **Every significant decision gets an ADR** before or with the implementing PR.
3. **Generated over hand-written** wherever possible (OpenAPI, `.env.example`). Hand-written docs are reserved for *why* and *how-to*; *what-exactly* should be derived from code.
4. **One canonical location per fact.** If two docs would state the same fact, one links to the other.
5. **End-of-workshop doc check.** Each workshop's acceptance criteria (see `11-roadmap.md`) include "affected docs updated" — documentation debt is not carried between workshops.
6. **Publishing.** `docs/` on the salepage `main` branch auto-publishes to GitHub Pages; merging to `main` is the release act for documentation.

---

*Previous: [09 — DevOps & Deployment](./09-devops-deployment.md) · Next: [11 — Development Roadmap](./11-roadmap.md)*
