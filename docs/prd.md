# Product Requirements Document (PRD)

## 1. Product

Cosmetics E-Commerce Platform — an online perfume store.

## 2. Goals

- Let customers discover and purchase perfumes online.
- Let admins manage catalog, orders, and customers.
- Ship a maintainable, team-friendly, standards-based foundation.

## 3. Scope

### In scope (roadmap)
- Product catalog, search, product detail
- Cart & checkout
- Customer accounts (Supabase Auth)
- Order management
- Admin dashboard for products/orders/customers
- Media storage (Supabase Storage)

### Out of scope (Workshop 1)
- No feature implementation yet. Workshop 1 delivers project foundation only:
  repositories, README/LICENSE/.gitignore, docs, architecture diagram, Pages.

## 4. Functional Requirements

| ID | Requirement |
|----|-------------|
| FR-1 | Customers can browse and search products |
| FR-2 | Customers can view product details |
| FR-3 | Customers can add to cart and checkout |
| FR-4 | Customers can register and log in |
| FR-5 | Admins can CRUD products and inventory |
| FR-6 | Admins can view and update orders |
| FR-7 | Admins can upload product media |
| FR-8 | System exposes a REST API consumed by both frontends |

## 5. Non-Functional Requirements

- Public repos, MIT licensed.
- Secrets kept out of Git (`.env`).
- Documentation published via GitHub Pages.
- Conventional Commits + `main`/`develop`/`feature/*` branching.

## 6. Milestones

- **Workshop 1** — Foundation: repos, docs, architecture, Pages. *(this)*
- **Workshop 2** — Scaffold Nuxt (dashboard), NestJS (server), Astro (salepage);
  connect Supabase.
