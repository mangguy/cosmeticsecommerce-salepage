# cosmeticsecommerce-salepage

Customer-facing website for the **Cosmetics E-Commerce Platform** (Perfume Store).

## Project Overview

Public storefront where customers browse and buy perfumes. Consumes the
`cosmeticsecommerce-server` REST API. This repo also hosts the project
**documentation** (`docs/`) and publishes it via **GitHub Pages**.
Part of a 3-repository system: `cosmeticsecommerce-dashboard`,
`cosmeticsecommerce-server`, and **salepage** (this repo).

## Technology Stack

- **Astro**
- **TypeScript**
- **Tailwind CSS**
- **Supabase** — data via the backend API

## Folder Structure

```
cosmeticsecommerce-salepage/
├── README.md
├── LICENSE
├── .gitignore
├── .env.example
├── docs/                 # project documentation (GitHub Pages source)
│   ├── index.md
│   ├── analysis.md
│   ├── design.md
│   ├── prd.md
│   └── architecture.md   # Mermaid system diagram
└── (app source — added in Workshop 2)
```

## Installation

```bash
git clone https://github.com/<owner>/cosmeticsecommerce-salepage.git
cd cosmeticsecommerce-salepage
cp .env.example .env   # fill SUPABASE_URL / SUPABASE_ANON_KEY
npm install
```

## Development

```bash
npm run dev      # start dev server
npm run build    # production build
npm run preview  # preview build
```

> App scaffolding lands in Workshop 2. This repo currently holds project
> foundation and documentation.

## Documentation & GitHub Pages

Docs live in [`docs/`](./docs/). To publish via GitHub Pages:
**Settings → Pages → Source: `Deploy from a branch` → Branch `main` / folder `/docs`**.
Published URL: `https://<owner>.github.io/cosmeticsecommerce-salepage/`.

GitHub renders the Markdown (and Mermaid diagrams) automatically.

## Branch Strategy

- `main` — production-ready, protected
- `develop` — integration branch
- `feature/*` — one branch per feature, merged into `develop`

Commits follow [Conventional Commits](https://www.conventionalcommits.org/)
(`feat:`, `fix:`, `docs:`, `chore:` …).

## Future Features

- Product listing & detail pages
- Cart & checkout
- Customer auth (Supabase)
- Search & filtering
- Order tracking
