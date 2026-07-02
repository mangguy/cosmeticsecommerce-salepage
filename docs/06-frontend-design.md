# Phase 6 — Frontend Design

**Project:** Cosmetics E-Commerce Platform (Online Perfume Store)
**Repositories covered:** `cosmeticsecommerce-salepage` (Astro customer website), `cosmeticsecommerce-dashboard` (Nuxt 3 admin SPA)
**Backend contract:** Both frontends consume only the NestJS REST API (`cosmeticsecommerce-server`, base path `/api/v1`). Authentication is via Supabase Auth JWTs. Roles: `guest`, `customer`, `admin`. Payment is mocked.

---

## Section A — Customer Website (Astro, `cosmeticsecommerce-salepage`)

### A.1 Technology Rationale

Astro is chosen for the storefront because the site is content- and SEO-dominated: most pages are read-heavy (product catalog, brand pages, blog) with small pockets of interactivity (cart, search, wishlist). Astro ships **zero JavaScript by default** and hydrates only the interactive "islands", giving the best Core Web Vitals profile for a commerce funnel where load speed directly correlates with conversion. TypeScript and Tailwind are used across both frontends for consistency.

### A.2 Page-by-Page Design

#### 1. Landing / Home (`/`)
- **Purpose:** Brand impression, hero campaigns, featured collections, new arrivals, bestsellers, entry point into the catalog.
- **UX notes:** Above-the-fold hero with a single clear CTA ("Shop Fragrances"). Curated sections (New In, Bestsellers, By Brand) in horizontally scrollable cards on mobile. Banner content is admin-managed (Banners module in the dashboard).
- **SEO:** `Organization` + `WebSite` JSON-LD (with `SearchAction` for sitelinks search box), meta title/description templated, canonical to `/`, Open Graph image for social shares.
- **Rendering mode:** **Hybrid — static shell, SSR-fetched banner/featured data at build time with scheduled rebuild (or short-TTL SSR fragment).** The page rarely changes per user; static gives instant TTFB. Islands: header cart badge, newsletter form.
- **Performance:** Hero image as responsive `<Image>` (AVIF/WebP, `fetchpriority="high"`, explicit dimensions to avoid CLS). Prefetch `/products` on hover. CDN cache with `stale-while-revalidate`.

#### 2. Product Listing (`/products`)
- **Purpose:** Browse the full catalog with pagination and sorting.
- **UX notes:** Grid of product cards (image, brand, name, price, wishlist toggle). Sort control (price, newest, popularity). Persistent filter sidebar (desktop) / filter drawer (mobile). Skeleton cards while paginating.
- **SEO:** `ItemList` JSON-LD, paginated canonical (`?page=2` canonicals to itself; `rel=prev/next` hints), meta description generated from active category context. Filtered/sorted URLs beyond page param get `noindex,follow` to avoid crawl-budget waste.
- **Rendering mode:** **SSR.** Inventory, prices, and sort order change frequently; SSR guarantees fresh, crawlable HTML. Filter/sort interactions are handled via URL query params so every state is shareable and indexable-controlled.
- **Performance:** Server response cached at the CDN for 60s. Product images lazy-loaded below the fold with `loading="lazy"` and fixed aspect ratio. Prefetch product detail pages on link hover (`data-astro-prefetch`).

#### 3. Product Detail (`/products/[slug]`)
- **Purpose:** Conversion page — full product info, variant (size/volume) selection, add to cart, reviews.
- **UX notes:** Image gallery with zoom, scent-note pyramid (top/heart/base — perfume-specific), volume variant selector updates price and stock in place, sticky add-to-cart bar on mobile, reviews with star breakdown, "You may also like" cross-sell row.
- **SEO:** The most SEO-critical template. `Product` JSON-LD with `offers` (price, availability), `AggregateRating`, `Review`; `BreadcrumbList` JSON-LD; canonical to the slug URL (variants are query params, not separate URLs); OG/Twitter product cards.
- **Rendering mode:** **SSR.** Price and stock must be accurate for both users and Google Merchant-style rich results. The static parts (description, notes) come from the same API payload, so per-request SSR with a 60s CDN cache is simpler and safer than partial static.
- **Performance:** First gallery image `fetchpriority="high"`; remaining gallery images lazy. Reviews fetched in an island after first paint (below the fold). Cross-sell row lazy-rendered on scroll.

#### 4. Search (`/search?q=`)
- **Purpose:** Full-text product search results.
- **UX notes:** Autocomplete island in the header (debounced 250ms, keyboard-navigable, shows top 5 products + "see all results"). Results page reuses the listing grid; shows the query, result count, and "no results" state with popular-category suggestions.
- **SEO:** `noindex,follow` — search result pages must not compete with category pages in the index. No structured data needed.
- **Rendering mode:** **SSR** for the results page (fresh results, works without JS); the **autocomplete is a client island**.
- **Performance:** Autocomplete requests debounced and aborted (`AbortController`) on new keystrokes; results endpoint cached briefly server-side.

#### 5. Filtering (facet UX, lives on listing/category/brand pages)
- **Purpose:** Narrow results by brand, category, scent family, price range, gender, availability.
- **UX notes:** All filter state encoded in the URL (`?brand=dior&scent=woody&price=50-150`) so results are shareable and back-button-safe. Applied filters shown as removable chips. Mobile: bottom-sheet filter drawer with an "Apply (n results)" button.
- **SEO:** Single-facet brand/category combinations may be indexable (curated); multi-facet combinations get `noindex,follow` + canonical to the unfiltered parent to prevent duplicate-content explosion.
- **Rendering mode:** **SSR** (same page templates). Filter form works via plain GET submission without JS; a small island progressively enhances it with instant counts.
- **Performance:** Facet counts returned by the listing API in the same response — no extra round trip.

#### 6. Brand Pages (`/brands`, `/brands/[slug]`)
- **Purpose:** Brand storytelling (hero, description) + brand-scoped product listing. Strong SEO landing pages ("Dior perfumes").
- **UX notes:** Brand hero banner and short editorial intro above the product grid; A–Z brand index page.
- **SEO:** `Brand`/`Organization` JSON-LD, `BreadcrumbList`, canonical per brand, descriptive meta from brand copy.
- **Rendering mode:** **Hybrid.** Brand copy/hero is near-static; the product grid is the SSR listing embedded in the same route. Practically: SSR with aggressive CDN caching (5 min), since brand catalogs change with inventory.
- **Performance:** Brand logos as small optimized SVG/WebP; grid follows listing-page rules.

#### 7. Category Pages (`/categories/[slug]`, e.g. Eau de Parfum, Unisex, Gift Sets)
- **Purpose:** Primary catalog navigation and SEO landing pages.
- **UX notes:** Category header with description, subcategory pills, then the filterable grid.
- **SEO:** `CollectionPage` + `ItemList` + `BreadcrumbList` JSON-LD; unique editorial category descriptions (avoid thin content); canonical per category.
- **Rendering mode:** **SSR** with CDN cache — same reasoning as brand pages.
- **Performance:** Identical to Product Listing.

#### 8. Blog (`/blog`, `/blog/[slug]`)
- **Purpose:** Content marketing (fragrance guides, "how to choose a perfume"), organic traffic capture, internal linking to products.
- **UX notes:** Clean readable article layout (65–75ch measure), inline product cards linking into the catalog, related posts.
- **SEO:** `Article`/`BlogPosting` JSON-LD with author and dates, canonical, OG article tags. Highest-value evergreen SEO surface on the site.
- **Rendering mode:** **Static (SSG)**, rebuilt on publish (webhook-triggered build or scheduled). Content changes only on editorial action — no reason to pay SSR cost. Zero islands except the shared header.
- **Performance:** Fully static HTML from CDN; inline images optimized at build time.

#### 9. About (`/about`) and 10. Contact (`/contact`)
- **Purpose:** Trust pages — story, values; contact form, store info.
- **UX notes:** Contact form with clear validation and success state; map/address block; link to FAQ.
- **SEO:** `AboutPage` / `ContactPage` JSON-LD, `LocalBusiness` if a physical store exists; canonical.
- **Rendering mode:** **Static.** Pure content. The contact form is a tiny island (or a plain HTML POST to the API — preferred, zero JS).
- **Performance:** Fully static; trivially fast.

#### 11. Wishlist (`/wishlist`)
- **Purpose:** Saved products for later; a retention and re-engagement feature.
- **UX notes:** Wishlist toggle (heart) appears on every product card/detail page as an island with optimistic UI. Guests get a localStorage wishlist with a prompt to sign in to persist; on login it merges to the server. Empty state suggests bestsellers.
- **SEO:** `noindex` — personal page.
- **Rendering mode:** **Hybrid — static shell + client island** that reads localStorage (guest) or calls `/api/v1/wishlist` (customer). Personalized content should never be SSR-cached, so rendering it client-side in a static shell is both faster and safer.
- **Performance:** Product data for wishlist items fetched in one batched API call.

#### 12. Shopping Cart (`/cart`)
- **Purpose:** Review items, adjust quantities, see totals, proceed to checkout.
- **UX notes:** Mini-cart drawer (island in the header) for quick add feedback; full cart page for editing. Quantity steppers with stock-limit validation, line-item remove with undo toast, order summary with shipping estimate, prominent checkout CTA. Cart badge count always visible.
- **SEO:** `noindex`. No structured data.
- **Rendering mode:** **Hybrid — static shell + cart island.** Cart contents are per-user client state; SSR would require session plumbing for no SEO benefit.
- **Performance:** Cart state reads are local-first (localStorage), so the page renders instantly; server sync happens in the background (see A.5).

#### 13. Checkout (`/checkout`)
- **Purpose:** Convert the cart into an order: address, shipping method, mock payment, confirmation.
- **UX notes:** Single-page checkout with clearly grouped steps (Contact → Shipping → Payment → Review) and a persistent order summary. Guest checkout allowed (email only) with post-purchase account prompt. Inline validation, disabled submit while processing, idempotent order submission (client-generated idempotency key) to prevent double orders. Mock payment step clearly labeled as a simulation.
- **SEO:** `noindex`. Exclude from sitemap.
- **Rendering mode:** **Static shell + one large checkout island** (the whole form). The flow is fully interactive and personal; there is nothing to server-render. Keeping it an island keeps the rest of the site JS-free.
- **Performance:** Checkout island code-split and loaded only on this route; address/shipping options fetched once and cached in memory.

#### 14. User Profile (`/account`, `/account/orders`, `/account/addresses`)
- **Purpose:** Account details, saved addresses, order history, wishlist link.
- **UX notes:** Simple tabbed/sidebar layout; order history rows link to order tracking; sign-out. Unauthenticated visitors are redirected to login (client-side guard, see A.6).
- **SEO:** `noindex`, excluded from sitemap.
- **Rendering mode:** **Static shell + authenticated client islands.** All data is per-user and JWT-gated; client fetching avoids SSR session handling entirely on the Astro side.
- **Performance:** Parallel fetches (profile + recent orders); skeleton rows during load.

#### 15. Order Tracking (`/orders/[id]`, plus guest lookup `/track`)
- **Purpose:** Show order status timeline (Placed → Paid → Packed → Shipped → Delivered).
- **UX notes:** Vertical status timeline with timestamps; item summary; guest lookup form (order number + email). Clear support contact for problems.
- **SEO:** `noindex`.
- **Rendering mode:** **Static shell + client island** — status must be live and is per-user; same reasoning as profile.
- **Performance:** Single order endpoint call; optional polling every 60s while the tab is visible.

### A.3 Rendering-Mode Summary

| Page | Mode | Why |
|---|---|---|
| Home | Hybrid (static + islands, rebuild on banner change) | Rarely changes; fastest TTFB |
| Product Listing | SSR (CDN cache 60s) | Fresh prices/stock, crawlable |
| Product Detail | SSR (CDN cache 60s) | Rich results need accurate price/stock |
| Search results | SSR, `noindex` | Fresh, works without JS |
| Brand / Category | SSR (CDN cache 5 min) | SEO landing pages + live catalog |
| Blog | Static (SSG) | Editorial content, rebuild on publish |
| About / Contact | Static | Pure content |
| Wishlist / Cart / Checkout / Profile / Tracking | Static shell + client islands, `noindex` | Personal, uncacheable, no SEO value |

### A.4 Folder Structure & Islands Architecture

```
cosmeticsecommerce-salepage/
├── src/
│   ├── pages/
│   │   ├── index.astro
│   │   ├── products/index.astro         # listing (SSR)
│   │   ├── products/[slug].astro        # detail (SSR)
│   │   ├── search.astro
│   │   ├── brands/{index,[slug]}.astro
│   │   ├── categories/[slug].astro
│   │   ├── blog/{index,[slug]}.astro
│   │   ├── about.astro  contact.astro
│   │   ├── wishlist.astro  cart.astro  checkout.astro
│   │   ├── account/{index,orders,addresses}.astro
│   │   └── orders/[id].astro  track.astro
│   ├── layouts/
│   │   ├── BaseLayout.astro             # <head>, meta/JSON-LD slots, header/footer
│   │   ├── CatalogLayout.astro          # filter sidebar + grid
│   │   └── AccountLayout.astro
│   ├── components/
│   │   ├── static/                      # zero-JS: ProductCard, Breadcrumbs, Footer, RatingStars...
│   │   └── islands/                     # hydrated: see below
│   └── lib/
│       ├── api/                         # typed fetch client, endpoint modules (products, cart, auth…)
│       ├── seo/                         # JSON-LD builders, meta helpers
│       └── stores/                      # nanostores: cart, wishlist, session
└── docs/
```

**Interactive islands (everything else ships no JS):**

| Island | Hydration | Reason |
|---|---|---|
| `CartDrawer` + header cart badge | `client:load` | Must reflect cart count immediately on every page |
| `SearchAutocomplete` | `client:idle` | Needed soon but not render-blocking |
| `WishlistToggle` (heart button) | `client:visible` | Many instances per page; hydrate only when scrolled into view |
| `VariantSelector` / `AddToCart` (PDP) | `client:load` | Core conversion interaction |
| `ReviewsSection` | `client:visible` | Below the fold |
| `CheckoutForm`, `AccountPanel`, `OrderTracker`, `WishlistPage`, `CartPage` | `client:load` | Fully client-driven personal pages |

Islands share state through **nanostores** (Astro's recommended pattern) — e.g. `AddToCart` writes to the cart store; the header badge island reacts — without a global framework runtime.

### A.5 Client-Side Cart State: localStorage + API Sync

- **Guest:** cart lives entirely in `localStorage` (versioned schema `{ v, items[], updatedAt }`).
- **Customer (logged in):** cart is persisted via `/api/v1/cart`; localStorage acts as a write-through cache. On login, the local cart **merges** into the server cart (sum quantities, cap at stock), then server becomes the source of truth.
- **Trade-off, explicitly:**
  - *Pros:* instant cart UX with zero latency, guests need no session on the server, cart survives page reloads, offline-tolerant.
  - *Cons:* cross-device carts only work when logged in; stale prices/stock in localStorage must be revalidated (the cart page re-fetches current price/stock per line and flags changes: "price updated", "only 2 left"); merge conflicts on login need a deterministic rule (we choose additive merge). Server-side carts for guests would fix cross-device at the cost of anonymous session management — not worth it at this scale. <!-- ponytail: additive merge; revisit if support tickets show confusion -->

### A.6 Auth Handling on an Astro Site

- Login/register pages call Supabase Auth via the NestJS API (`/api/v1/auth/*`); the API returns the Supabase **access JWT (short-lived, kept in memory + sessionStorage)** and sets the **refresh token as an httpOnly, Secure, SameSite=Lax cookie**.
- Because personal pages are client-rendered islands, **no SSR session is needed**: each island's API client attaches `Authorization: Bearer <jwt>` and transparently refreshes via `/api/v1/auth/refresh` (cookie) on 401.
- A tiny `session` nanostore exposes `{ user, role, status }` to all islands (header shows "Sign in" vs avatar).
- Route protection is **client-side redirect** on account pages (acceptable: the shell contains no private data; the API is the real security boundary — every protected endpoint validates the JWT server-side). SEO pages never depend on auth.

---

## Section B — Admin Dashboard (Nuxt 3, `cosmeticsecommerce-dashboard`)

### B.1 SPA Mode (`ssr: false`) — Why

The dashboard is a private, auth-gated tool: **no SEO, no anonymous traffic, no first-paint pressure from search engines**. SPA mode eliminates the entire SSR class of problems (server hydration mismatches, per-request auth on the Nuxt server, double data-fetching) and allows static hosting on any CDN — the only backend is the NestJS API. Admins load the app once per session; a slightly larger initial bundle is irrelevant, and Nuxt's route-level code splitting keeps it reasonable anyway.

### B.2 Navigation Structure (Sidebar Map)

```
┌─ Sidebar ──────────────────────────┐
│ ● Dashboard            /            │
│ ▾ Catalog                           │
│    Products            /products    │
│    Categories          /categories  │
│    Brands              /brands      │
│    Inventory           /inventory   │
│ ▾ Sales                             │
│    Orders              /orders      │
│    Customers           /customers   │
│ ▾ Marketing                         │
│    Banners             /banners     │
│    Coupons             /coupons     │
│ ● Reports & Analytics  /reports     │
│ ● Settings             /settings    │
└─────────────────────────────────────┘
Topbar: global search, notifications, admin avatar menu (profile, sign out)
```

Sidebar collapses to icons on tablet, becomes an off-canvas drawer on mobile. Active section highlighted; groups remember open/closed state (persisted in the `ui` store).

### B.3 Pages

A single **shared page pattern** applies to all CRUD modules: *data table (server-side pagination, sort, column filters) + toolbar (search, filter chips, "New" button) + drawer or modal form for create/edit + row actions (edit, duplicate, archive/delete with confirm)*. Only deviations are noted below.

| Page | Purpose | Key UI patterns |
|---|---|---|
| **Dashboard Overview** (`/`) | At-a-glance business health | KPI stat cards (revenue, orders, AOV, low-stock count), revenue line chart (7/30/90d), recent orders table, low-stock alert list linking to Inventory |
| **Products** (`/products`) | Full product CRUD | Standard pattern + **full-page form** (`/products/new`, `/products/:id`) — product forms are too large for a drawer: tabs for General, Variants (volumes/prices), Images (drag-drop upload + reorder), Scent Notes, SEO. Bulk actions (publish/unpublish). Status badges (draft/active/archived) |
| **Categories** (`/categories`) | Manage category tree | Tree view with drag-to-reorder + drawer form (name, slug, description, image, parent) |
| **Brands** (`/brands`) | Brand CRUD | Standard pattern; drawer form with logo upload and editorial copy |
| **Inventory** (`/inventory`) | Stock per variant | Table with inline-editable stock quantity, low-stock threshold column, filter "below threshold", stock adjustment modal with reason (audit trail) |
| **Orders** (`/orders`) | Fulfillment workflow | Table filtered by status tabs (New / Paid / Packed / Shipped / Delivered / Cancelled); order detail drawer: items, customer, address, payment (mock) info, **status transition buttons** with confirmation, internal notes timeline |
| **Customers** (`/customers`) | Customer lookup & support | Read-mostly table; detail drawer with profile, order history, lifetime value; deactivate action |
| **Banners** (`/banners`) | Home/campaign banners | Card grid with preview thumbnails, drag-to-reorder, drawer form (image, link, schedule start/end, active toggle) |
| **Coupons** (`/coupons`) | Discount codes | Standard pattern; drawer form (code, type %/fixed, min order, usage limit, validity window); usage-count column |
| **Reports & Analytics** (`/reports`) | Deeper analysis | Date-range picker, sales-by-category/brand charts, top products table, CSV export button |
| **Settings** (`/settings`) | Store config & admins | Tabbed form sections (store info, shipping rates, admin users); explicit Save per section with dirty-state warning |

### B.4 Folder Structure & Route Organization

```
cosmeticsecommerce-dashboard/
├── nuxt.config.ts                 # ssr: false
├── pages/                         # file-based routes = the sidebar map
│   ├── index.vue                  # overview
│   ├── login.vue                  # public (layout: blank)
│   ├── products/{index,new,[id]}.vue
│   ├── categories/index.vue  brands/index.vue  inventory/index.vue
│   ├── orders/index.vue  customers/index.vue
│   ├── banners/index.vue  coupons/index.vue
│   ├── reports/index.vue  settings/index.vue
├── layouts/
│   ├── default.vue                # sidebar + topbar shell
│   └── blank.vue                  # login
├── components/
│   ├── base/                      # BaseButton, BaseInput, BaseSelect, BaseTable,
│   │                              # BaseModal, BaseDrawer, BaseBadge, BaseToast…
│   └── domain/                    # ProductForm, OrderStatusStepper, StockAdjustModal,
│                                  # KpiCard, RevenueChart, CouponForm…
├── composables/
│   ├── useApi.ts                  # typed fetch wrapper (see B.6)
│   ├── useDataTable.ts            # pagination/sort/filter state ↔ query params
│   └── useToast.ts  useConfirm.ts
├── stores/                        # Pinia: auth.ts, products.ts, orders.ts, ui.ts
├── middleware/
│   ├── auth.global.ts             # session check on every route
│   └── admin.ts                   # role guard
└── types/                         # API DTO types shared shapes
```

- **Route organization:** file-based routes mirror the sidebar exactly — one folder per module; only Products gets sub-routes for full-page forms. Everything else edits in drawers/modals so list context (filters, page) is never lost.
- **Component organization:** `base/` components are app-agnostic primitives with a consistent prop API (variant/size/state) — the internal design system. `domain/` components compose base components with business logic and are the only components that touch stores/API types. Rule: base components never import from `stores/` or `lib/api`.

### B.5 State Management — Pinia

**Why Pinia:** it is the official Vue/Nuxt store — first-class Nuxt module, full TypeScript inference, devtools time-travel, and modular stores without boilerplate (no mutations layer). Alternatives (raw `useState`, provide/inject) don't scale to cross-page concerns like auth and toasts.

| Store | Holds | Notes |
|---|---|---|
| `auth` | user, role, access token (memory), session status | login/logout/refresh actions; token never in localStorage (XSS) — refresh cookie restores sessions |
| `products` | list cache for current query, active filters, selection for bulk actions | list data is otherwise fetch-per-page — the store caches only the *current* view, not the world |
| `orders` | status-tab counts, active order detail, optimistic status transitions | rollback on API failure + toast |
| `ui` | sidebar collapsed state, toasts queue, confirm-dialog state, active drawer | pure client concerns |

Server data that is only displayed once (reports, customers) stays in page-local `useAsyncData` — no store needed. <!-- ponytail: 4 stores only; add per-module stores when cross-page sharing actually appears -->

### B.6 API Layer

- `useApi()` composable wraps `$fetch` with: base URL `/api/v1`, `Authorization` header from the `auth` store, typed request/response generics per endpoint module (`api/products.ts`, `api/orders.ts`, …).
- **Token refresh interceptor:** on `401`, the client (a) queues concurrent failed requests, (b) calls `/auth/refresh` once (httpOnly cookie), (c) replays the queue with the new token, (d) on refresh failure clears the `auth` store and redirects to `/login?next=…`.
- Uniform error envelope from NestJS (`{ statusCode, message, error }`) mapped to toasts; validation errors (422) mapped to per-field form errors.

### B.7 Route Middleware — Auth/Role Guard

- `auth.global.ts`: runs on every navigation; if no session and route ≠ `/login`, attempt silent refresh, else redirect to `/login?next=<path>`.
- `admin.ts`: applied via `definePageMeta({ middleware: 'admin' })` on all dashboard pages; verifies `role === 'admin'` from the JWT claims — a logged-in `customer` is rejected with a "no access" screen.
- Middleware is UX-level gating only; **the API re-validates role on every request** (defense in depth).

---

## C. Alternatives & Trade-offs

| Decision | Alternative | Why we chose as we did |
|---|---|---|
| **Astro for storefront** | Next.js / Nuxt | The storefront is ~80% content, ~20% interaction. Astro's islands ship dramatically less JS than a full React/Vue hydration model, directly improving LCP/INP and SEO. Next/Nuxt would be justified if the storefront were heavily personalized or app-like (it isn't). Cost: two frontend frameworks in the org (Astro + Vue) — mitigated because Astro islands could be written in Vue if desired. |
| **SPA (`ssr:false`) for admin** | Nuxt universal SSR | SSR for an auth-gated tool buys nothing (no SEO, no anonymous users) and costs a Node server, hydration complexity, and per-request auth plumbing. SPA deploys as static files. Trade-off: blank-screen-until-JS on first load — acceptable for daily-use internal users, mitigated with an app-shell splash. |
| **SSR (not SSG) for catalog pages** | Full SSG with rebuilds | SSG would need a rebuild on every price/stock change — operationally fragile. SSR + 60s CDN cache gives near-static performance with guaranteed freshness. |
| **Cart in localStorage + sync** | Server-side guest carts | Avoids anonymous session management; instant UX. Cost: no cross-device guest carts (fine — login solves it). |
| **JWT in memory + refresh cookie** | JWT in localStorage | localStorage tokens are trivially exfiltrated by XSS; memory + httpOnly refresh cookie is the standard safer pattern at negligible complexity cost. |
| **Pinia** | Vuex / composables only | Vuex is legacy for Vue 3; bare composables lack devtools and cross-page structure. |

## D. Risks

1. **Two frameworks (Astro + Nuxt)** raises the team's surface area. *Mitigation:* shared Tailwind config/design tokens and a shared API type package keep the overlap thin.
2. **Client-only account pages** show a brief loading state and rely on client redirects; a misconfigured island could leak a personal-page shell (not data). *Mitigation:* API is the security boundary; shells contain no PII.
3. **localStorage cart staleness** (price/stock drift). *Mitigation:* revalidation on cart/checkout render; checkout re-validates server-side at order creation.
4. **Facet URL explosion** harming crawl budget. *Mitigation:* strict `noindex` rules on multi-facet URLs, curated indexable facets only, sitemap limited to canonical pages.
5. **Bundle growth in the dashboard SPA.** *Mitigation:* route-level code splitting (default in Nuxt), lazy-load charts.
6. **Token refresh race conditions** (parallel 401s). *Mitigation:* single-flight refresh queue in the interceptor (specified in B.6).

## E. Recommendations

1. Generate API types from the NestJS OpenAPI spec into a shared package consumed by both frontends — eliminates drift.
2. Enforce Core Web Vitals budgets in CI (Lighthouse CI): LCP < 2.5s, CLS < 0.1 on Home/PDP.
3. Add `sitemap.xml` + `robots.txt` generation to the Astro build from day one.
4. Ship skeleton states and empty states with the first version of every list view — retrofitting them is always deprioritized.
5. Keep the admin table/filter/drawer pattern as literal shared components (`useDataTable`, `BaseDrawer`) before building the second CRUD module, so all eleven modules stay consistent.
6. Document the cart merge rule and order idempotency key in the API contract (Phase 4/5 docs) so backend and frontend agree.
