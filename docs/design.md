# Design

## Repository Structure

Three independent GitHub repositories:

| Repository | Role | Stack |
|---|---|---|
| `cosmeticsecommerce-salepage` | Customer website + docs/Pages | Astro + TS + Tailwind |
| `cosmeticsecommerce-dashboard` | Admin dashboard | Nuxt.js + Vue 3 + TS + Tailwind |
| `cosmeticsecommerce-server` | Backend REST API | NestJS + Supabase |

Each repo contains: `README.md`, `LICENSE` (MIT), `.gitignore`, `.env.example`.

## Development Workflow

**Branches**
- `main` — production-ready, protected.
- `develop` — integration branch.
- `feature/*` — one branch per feature, merged into `develop` via PR.

**Flow**
1. Branch `feature/<name>` off `develop`.
2. Commit using [Conventional Commits](https://www.conventionalcommits.org/).
3. Open PR into `develop`; review; merge.
4. Release: merge `develop` → `main`.

**Tools**
- Git + GitHub for version control and collaboration.
- SourceTree as the Git GUI (commit history, branching).
- GitHub Pages to publish documentation from `salepage/docs`.

## Environments & Secrets

- Supabase credentials via `.env` (never committed; see `.env.example`).
- `SUPABASE_URL`, `SUPABASE_ANON_KEY` at minimum.
