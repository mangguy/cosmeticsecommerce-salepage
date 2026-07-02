# Phase 5 — Backend Design (NestJS)

**Project:** Cosmetics E-Commerce Platform (Online Perfume Store)
**Repository:** `cosmeticsecommerce-server`
**Stack:** NestJS + TypeScript, Supabase (PostgreSQL, Auth, Storage)
**API Base:** `/api/v1`
**Status:** Approved design — Phase 5

---

## 1. Objectives

1. Define the backend architecture for the single REST API that serves both the customer sale page (`cosmeticsecommerce-salepage`) and the admin dashboard (`cosmeticsecommerce-dashboard`).
2. Establish a module structure that maps 1:1 to business capabilities and to the database schema designed in Phase 4.
3. Specify the cross-cutting infrastructure: authentication, authorization (RBAC), validation, error handling, logging, and the response envelope contract.
4. Publish the complete REST API surface (endpoints, auth levels, conventions) so frontend teams can build against a stable contract before implementation is complete.
5. Keep the design realistic for a small team and a university timeline, while following production-grade conventions.

**Non-goals:** implementation code, deployment topology (Phase 7), frontend design (Phase 6).

---

## 2. Architecture Style

### 2.1 Decision: Modular Monolith, Layered

The backend is a **modular monolith**: a single deployable NestJS application, internally divided into feature modules with explicit boundaries. Within each module, code is organized in three layers:

```
HTTP request
   │
   ▼
Controller   — routing, auth guards, DTO validation, HTTP semantics only
   │
   ▼
Service      — business rules, orchestration, transactions across repositories
   │
   ▼
Repository   — data access; the ONLY layer that touches the Supabase client
   │
   ▼
Supabase (PostgreSQL / Auth / Storage)
```

Rules of the layering:

- Controllers never contain business logic and never call repositories directly.
- Services never read HTTP objects (request/response); they receive validated DTOs and return domain objects.
- Repositories never contain business rules; they translate between domain objects and database rows.
- A module's service may depend on another module's **service** (never its repository), keeping module boundaries meaningful.

### 2.2 Why Not Microservices

| Concern | Microservices | Modular monolith (chosen) |
|---|---|---|
| Team size | Designed for many teams owning services independently | One small team; no organizational boundary to mirror |
| Operational cost | Service discovery, distributed tracing, per-service CI/CD, network failure modes | One deploy, one log stream, one health check |
| Data consistency | Distributed transactions / sagas for order + payment + stock | Single PostgreSQL database; ordinary transactions |
| Latency | Inter-service network hops | In-process method calls |
| Migration path | — | Module boundaries already exist; a module can be extracted later if scale ever demands it |

For a store with one database and one team, microservices add distributed-systems failure modes without buying any independent scalability we need. The modular structure preserves the *option* to extract services later — the boundaries are the modules.

### 2.3 Why Not Full DDD (Aggregates, Domain Events, CQRS)

Tactical DDD (aggregate roots, value objects, domain events, separate read/write models) pays off when business rules are deep and volatile. This domain is a well-understood catalog-cart-order flow with mock payments. Full DDD here would mean:

- Mapping layers between persistence models, domain entities, and DTOs — three representations of `Product` where one suffices.
- Event buses and handlers for flows (order placed → stock decremented) that a single service method with a transaction expresses more clearly.
- CQRS read models duplicating tables that a paginated query already serves.

We keep the *strategic* part of DDD — modules aligned to bounded contexts (catalog, ordering, identity) — and skip the tactical ceremony. If order lifecycle rules grow complex, the `orders` module can adopt richer domain objects internally without changing its public contract.

---

## 3. Module Structure

### 3.1 Modules

| Module | Responsibility | Owns tables |
|---|---|---|
| `auth` | Register, login, refresh, logout; Supabase Auth integration; JWT strategy | — (delegates to Supabase Auth) |
| `users` | Profiles, addresses, admin user management | `profiles`, `addresses` |
| `products` | Product catalog, images, variants, search/filter/sort | `products`, `product_images`, `product_variants` |
| `brands` | Brand CRUD | `brands` |
| `categories` | Category tree CRUD | `categories` |
| `inventory` | Stock levels, stock movements, low-stock reporting | `stock_movements` (+ variant stock fields) |
| `cart` | Cart lifecycle for customers and guests | `carts`, `cart_items` |
| `orders` | Checkout, order lifecycle, status history | `orders`, `order_items`, `order_status_history` |
| `payments` | Mock payment processing, payment records | `payments` |
| `reviews` | Product reviews with purchase verification, moderation | `reviews` |
| `wishlist` | Customer wishlist | `wishlist_items` |
| `coupons` | Coupon CRUD and validation at checkout | `coupons` |
| `banners` | Marketing banners for the sale page | `banners` |
| `reports` | Admin analytics: sales, top products, order stats | — (reads across tables) |
| `common` | Shared infrastructure: guards, filters, interceptors, decorators, Supabase provider, audit logging | `audit_logs` |

### 3.2 Folder Tree

```
cosmeticsecommerce-server/
├── src/
│   ├── main.ts                          # bootstrap, global pipes/filters/interceptors
│   ├── app.module.ts                    # root module, imports all feature modules
│   │
│   ├── config/
│   │   ├── configuration.ts             # typed env config (port, Supabase URL/keys, CORS)
│   │   └── validation.schema.ts         # env validation at boot (fail fast)
│   │
│   ├── common/
│   │   ├── common.module.ts
│   │   ├── supabase/
│   │   │   ├── supabase.provider.ts     # single configured Supabase client (service role)
│   │   │   └── supabase.types.ts        # generated DB types
│   │   ├── guards/
│   │   │   ├── jwt-auth.guard.ts
│   │   │   └── roles.guard.ts
│   │   ├── decorators/
│   │   │   ├── roles.decorator.ts       # @Roles('admin')
│   │   │   ├── public.decorator.ts      # @Public() — skip JwtAuthGuard
│   │   │   └── current-user.decorator.ts# @CurrentUser() — typed user from request
│   │   ├── middleware/
│   │   │   ├── correlation-id.middleware.ts
│   │   │   └── request-logger.middleware.ts
│   │   ├── interceptors/
│   │   │   ├── response-envelope.interceptor.ts
│   │   │   └── timeout.interceptor.ts
│   │   ├── filters/
│   │   │   └── all-exceptions.filter.ts # unified error envelope
│   │   ├── dto/
│   │   │   └── pagination-query.dto.ts  # shared ?page=&limit=&sort=
│   │   └── errors/
│   │       ├── error-codes.ts           # canonical error code constants
│   │       └── app.exception.ts         # domain exception carrying code + status
│   │
│   ├── auth/
│   │   ├── auth.module.ts
│   │   ├── auth.controller.ts
│   │   ├── auth.service.ts
│   │   ├── strategies/jwt.strategy.ts   # Supabase JWT validation
│   │   └── dto/
│   │       ├── register.dto.ts
│   │       ├── login.dto.ts
│   │       └── refresh-token.dto.ts
│   │
│   ├── users/
│   │   ├── users.module.ts
│   │   ├── users.controller.ts          # /users/me, /users/me/addresses
│   │   ├── users.admin.controller.ts    # /admin/users
│   │   ├── users.service.ts
│   │   ├── repositories/
│   │   │   ├── profiles.repository.ts
│   │   │   └── addresses.repository.ts
│   │   ├── entities/
│   │   │   ├── profile.entity.ts
│   │   │   └── address.entity.ts
│   │   └── dto/
│   │       ├── update-profile.dto.ts
│   │       ├── create-address.dto.ts
│   │       └── update-address.dto.ts
│   │
│   ├── products/
│   │   ├── products.module.ts
│   │   ├── products.controller.ts       # public catalog reads
│   │   ├── products.admin.controller.ts # admin CRUD, image upload
│   │   ├── products.service.ts
│   │   ├── repositories/
│   │   │   ├── products.repository.ts
│   │   │   ├── product-images.repository.ts
│   │   │   └── product-variants.repository.ts
│   │   ├── entities/
│   │   │   ├── product.entity.ts
│   │   │   ├── product-image.entity.ts
│   │   │   └── product-variant.entity.ts
│   │   └── dto/
│   │       ├── product-query.dto.ts     # filters: brand, category, minPrice, maxPrice, q
│   │       ├── create-product.dto.ts
│   │       ├── update-product.dto.ts
│   │       └── create-variant.dto.ts
│   │
│   ├── brands/
│   │   ├── brands.module.ts
│   │   ├── brands.controller.ts
│   │   ├── brands.service.ts
│   │   ├── brands.repository.ts
│   │   ├── entities/brand.entity.ts
│   │   └── dto/{create-brand.dto.ts, update-brand.dto.ts}
│   │
│   ├── categories/
│   │   └── … (same shape as brands)
│   │
│   ├── inventory/
│   │   ├── inventory.module.ts
│   │   ├── inventory.admin.controller.ts
│   │   ├── inventory.service.ts         # stock adjust, reserve/release on orders
│   │   ├── stock-movements.repository.ts
│   │   ├── entities/stock-movement.entity.ts
│   │   └── dto/adjust-stock.dto.ts
│   │
│   ├── cart/
│   │   ├── cart.module.ts
│   │   ├── cart.controller.ts
│   │   ├── cart.service.ts
│   │   ├── repositories/{carts.repository.ts, cart-items.repository.ts}
│   │   ├── entities/{cart.entity.ts, cart-item.entity.ts}
│   │   └── dto/{add-cart-item.dto.ts, update-cart-item.dto.ts}
│   │
│   ├── orders/
│   │   ├── orders.module.ts
│   │   ├── orders.controller.ts         # customer: create, list mine, cancel
│   │   ├── orders.admin.controller.ts   # admin: list all, update status
│   │   ├── orders.service.ts            # checkout orchestration (cart → order → stock)
│   │   ├── repositories/
│   │   │   ├── orders.repository.ts
│   │   │   ├── order-items.repository.ts
│   │   │   └── order-status-history.repository.ts
│   │   ├── entities/{order.entity.ts, order-item.entity.ts}
│   │   └── dto/{create-order.dto.ts, update-order-status.dto.ts, order-query.dto.ts}
│   │
│   ├── payments/
│   │   ├── payments.module.ts
│   │   ├── payments.controller.ts
│   │   ├── payments.service.ts          # mock gateway: deterministic success/failure
│   │   ├── payments.repository.ts
│   │   ├── entities/payment.entity.ts
│   │   └── dto/create-payment.dto.ts
│   │
│   ├── reviews/
│   │   ├── reviews.module.ts
│   │   ├── reviews.controller.ts
│   │   ├── reviews.admin.controller.ts  # moderation
│   │   ├── reviews.service.ts
│   │   ├── reviews.repository.ts
│   │   ├── entities/review.entity.ts
│   │   └── dto/{create-review.dto.ts, review-query.dto.ts}
│   │
│   ├── wishlist/
│   │   ├── wishlist.module.ts
│   │   ├── wishlist.controller.ts
│   │   ├── wishlist.service.ts
│   │   ├── wishlist.repository.ts
│   │   └── entities/wishlist-item.entity.ts
│   │
│   ├── coupons/
│   │   ├── coupons.module.ts
│   │   ├── coupons.controller.ts        # POST /coupons/validate (customer)
│   │   ├── coupons.admin.controller.ts
│   │   ├── coupons.service.ts
│   │   ├── coupons.repository.ts
│   │   ├── entities/coupon.entity.ts
│   │   └── dto/{create-coupon.dto.ts, validate-coupon.dto.ts}
│   │
│   ├── banners/
│   │   └── … (same shape as brands; public read + admin CRUD)
│   │
│   └── reports/
│       ├── reports.module.ts
│       ├── reports.admin.controller.ts
│       ├── reports.service.ts
│       ├── reports.repository.ts        # aggregate queries
│       └── dto/report-query.dto.ts      # date range, granularity
│
├── test/                                # e2e tests per module
├── .env.example
├── nest-cli.json
├── tsconfig.json
└── package.json
```

Naming conventions: files are `kebab-case` with role suffixes (`.controller.ts`, `.service.ts`, `.repository.ts`, `.dto.ts`, `.entity.ts`); classes are `PascalCase` (`ProductsService`); one class per file. Admin-only routes live in separate `*.admin.controller.ts` files under `/api/v1/admin/...` so the RBAC surface is visible at a glance.

---

## 4. Layer Responsibilities

### 4.1 Controllers

- Declare routes, HTTP methods, and status codes (`@HttpCode(204)` for deletes).
- Apply guards (`JwtAuthGuard`, `RolesGuard`) and role decorators.
- Bind and validate DTOs (global `ValidationPipe`); pass validated input to services.
- Contain **zero** business logic and **zero** Supabase calls.
- Split public/customer controllers from admin controllers per module.

### 4.2 Services

- Hold all business rules: coupon applicability, stock sufficiency at checkout, order status transition rules (e.g., `pending → paid → shipped → delivered`, `cancelled` only from `pending/paid`), review eligibility (purchased + delivered).
- Orchestrate multi-repository operations. Checkout is the canonical case: validate cart → validate coupon → check stock → create order + items → record stock movements → clear cart, executed atomically (PostgreSQL function/RPC invoked through the repository layer, since multi-statement transactions are not composable across separate Supabase client calls).
- Throw typed `AppException`s with canonical error codes; never return raw HTTP responses.

### 4.3 DTOs (class-validator)

- Every request body and query string has a DTO class decorated with `class-validator` rules (`@IsEmail`, `@IsUUID`, `@Min(1)`, `@IsEnum`, `@MaxLength`, …) and `class-transformer` conversions (`@Type(() => Number)` for query params).
- Update DTOs are `PartialType(CreateDto)` — one source of truth for field rules.
- Shared `PaginationQueryDto` provides `page`, `limit` (capped at 100), `sort` with a whitelist of sortable fields per endpoint.
- DTOs define the **wire contract**; entities define the **domain shape**. They are allowed to diverge (e.g., entity has `costPrice`, public DTO responses never expose it).

### 4.4 Entities

- Plain TypeScript interfaces/classes mirroring the Phase 4 tables (camelCase properties mapped from snake_case columns in the repository layer).
- No ORM decorators — Supabase is accessed via its client, not TypeORM/Prisma, so entities stay persistence-agnostic type definitions plus the generated Supabase DB types in `common/supabase/supabase.types.ts`.

### 4.5 Repositories (Supabase client wrapper)

Each repository wraps the shared Supabase client (service-role key, held server-side only) and exposes intention-revealing methods: `findBySlug`, `findPageByFilters`, `decrementStock`, `insertWithItems`.

Why a repository abstraction instead of calling the Supabase client from services:

1. **Single translation point.** snake_case ↔ camelCase mapping, Supabase error objects → domain exceptions (`PGRST116` → `NOT_FOUND`, unique violation `23505` → `CONFLICT`) happen once, not in every service method.
2. **Testability.** Services are unit-tested against a mocked repository interface; no Supabase emulator needed for business-rule tests.
3. **Query reuse and review.** Filter/sort/pagination query building lives in one place per table; a wrong join or missing `.eq('is_active', true)` is a one-file fix.
4. **Portability insurance.** If Supabase were replaced by direct `pg`/Prisma, only repositories change. This is cheap insurance, not speculative architecture — the layer exists anyway for reasons 1–3.
5. **Boundary enforcement.** "Only the server talks to Supabase" (system rule) becomes "only repositories talk to Supabase" inside the server — greppable and lintable.

Deliberately **not** built: a generic `BaseRepository<T>` with abstract CRUD. Each repository is concrete; shared behavior is a small set of helper functions (pagination range calculation, error mapping), because generic repositories tend to leak query-builder types and invite bypassing the abstraction.

### 4.6 Guards

- **`JwtAuthGuard`** — global guard (registered as `APP_GUARD`). Validates the Supabase-issued JWT on every route except those marked `@Public()`. Attaches `{ userId, email, role }` to the request. Public catalog routes (`GET /products`, `GET /brands`, …) are explicitly opted out via `@Public()` — secure-by-default: forgetting a decorator locks a route down rather than exposing it.
- **`RolesGuard`** — reads `@Roles(...)` metadata; compares against the authenticated user's role from the `profiles` table (embedded in the JWT via Supabase custom claims, with a DB fallback check for admin routes). No metadata → any authenticated user passes.

### 4.7 Middleware

- **`CorrelationIdMiddleware`** — first in the chain. Reads `X-Request-Id` from the incoming request or generates a UUID; stores it in async-local context; echoes it on the response header and in the error envelope (`error.requestId`). Every log line for the request carries it.
- **`RequestLoggerMiddleware`** — logs method, path, status, duration, correlation id, and user id (if authenticated) on response finish. Structured JSON logs; bodies are never logged (PII), and `Authorization` headers are redacted.

### 4.8 Interceptors

- **`ResponseEnvelopeInterceptor`** — wraps every successful controller return value in the standard envelope (Section 6.2). Controllers return plain data; the envelope is applied centrally so no handler can drift from the contract. List handlers return `{ items, meta }` which the interceptor maps to `data` + `meta`.
- **`TimeoutInterceptor`** — aborts handlers exceeding 15 s with `504`-style failure (`REQUEST_TIMEOUT`), preventing a slow Supabase query from pinning connections.

### 4.9 Exception Filter

- **`AllExceptionsFilter`** — single global filter. Maps:
  - `AppException` (domain) → its status + code + message.
  - `ValidationPipe` errors → `422 VALIDATION_FAILED` with per-field details.
  - Nest `HttpException` → equivalent envelope.
  - Anything else → `500 INTERNAL_ERROR`, generic message to the client, full stack + correlation id to logs. Internal details never leak to responses.

---

## 5. Authentication & Authorization

### 5.1 Authentication (Supabase JWT)

- Supabase Auth is the identity provider: it stores credentials, hashes passwords, and issues **JWT access tokens** (short-lived, ~1 h) and **refresh tokens**.
- The NestJS server proxies auth flows: `POST /auth/register` calls Supabase Admin sign-up and creates the `profiles` row (role `customer`) in the same flow; `POST /auth/login` exchanges credentials for the token pair; `POST /auth/refresh` exchanges the refresh token. Frontends never call Supabase directly (system rule), so Supabase keys never ship to browsers.
- **Validation strategy:** `JwtStrategy` verifies incoming access tokens **locally** against the Supabase JWT secret (HS256) — signature, `exp`, `aud`, `iss` — with no network round-trip per request. The verified payload yields `sub` (user id) and the role claim.
- Token transport: `Authorization: Bearer <access_token>`. Refresh tokens are returned in the login/refresh response body; the dashboard/salepage decide storage (Phase 6), with the recommendation of memory + silent refresh.
- Logout (`POST /auth/logout`) revokes the refresh token via Supabase; access tokens simply expire (accepted trade-off of stateless JWTs — see Risks).

### 5.2 Authorization (RBAC)

- Roles: `guest` (no token), `customer`, `admin` — stored on `profiles.role` and mirrored into the JWT as a custom claim at token issuance.
- Declarative enforcement: `@Roles('admin')` on controllers/handlers + `RolesGuard`. Admin controllers apply it at class level.
- **Ownership checks live in services, not guards:** "a customer may read *their own* orders" is `ordersService.findOneForUser(orderId, userId)` returning `404` if the order belongs to someone else (404, not 403, to avoid confirming the resource exists).
- Guard order: `JwtAuthGuard` → `RolesGuard` (authentication before authorization), giving correct `401` vs `403` semantics.

---

## 6. Validation & Error Handling

### 6.1 Validation Strategy

- Global `ValidationPipe` with `whitelist: true` (strip unknown fields), `forbidNonWhitelisted: true` (reject unexpected fields — fail loud), `transform: true` (query strings become typed DTOs).
- Layered validation:
  1. **Shape/format** — DTO + class-validator → `422 VALIDATION_FAILED`.
  2. **Business rules** — services → `409`/`422` with a specific code (`INSUFFICIENT_STOCK`, `COUPON_EXPIRED`).
  3. **Database constraints** — Phase 4 constraints (FKs, unique, checks) as the last line of defense; repository maps violations to domain errors. Constraints in the DB, not re-implemented in app code, are the source of truth for integrity.
- Route params validated with `ParseUUIDPipe`; environment config validated at boot (fail fast on missing Supabase keys).

### 6.2 Response Envelope

Every response uses one envelope:

```json
// Success (single resource)
{ "success": true, "data": { "id": "…" }, "error": null, "meta": null }

// Success (list)
{
  "success": true,
  "data": [ { "id": "…" } ],
  "error": null,
  "meta": { "page": 1, "limit": 20, "totalItems": 137, "totalPages": 7 }
}

// Failure
{
  "success": false,
  "data": null,
  "error": {
    "code": "INSUFFICIENT_STOCK",
    "message": "Only 2 units of 'Noir Extrait 50ml' remain.",
    "details": [ { "field": "items[0].quantity", "issue": "requested 5, available 2" } ],
    "requestId": "9f8a1c2e-…"
  },
  "meta": null
}
```

### 6.3 Error Codes (canonical set)

| HTTP | Code | Used for |
|---|---|---|
| 400 | `BAD_REQUEST` | Malformed JSON, invalid UUID param |
| 401 | `UNAUTHENTICATED` | Missing/invalid/expired token |
| 401 | `TOKEN_EXPIRED` | Expired access token (frontends trigger refresh) |
| 403 | `FORBIDDEN` | Valid token, insufficient role |
| 404 | `NOT_FOUND` | Resource absent or not owned by caller |
| 409 | `CONFLICT` | Duplicate email/slug/SKU; state conflicts |
| 409 | `INSUFFICIENT_STOCK` | Checkout exceeds available stock |
| 409 | `INVALID_STATUS_TRANSITION` | Illegal order status change |
| 422 | `VALIDATION_FAILED` | DTO validation errors (field details included) |
| 422 | `COUPON_INVALID` / `COUPON_EXPIRED` | Coupon rejected at validation/checkout |
| 422 | `PAYMENT_DECLINED` | Mock gateway declined |
| 429 | `RATE_LIMITED` | Throttle exceeded (auth endpoints) |
| 500 | `INTERNAL_ERROR` | Unhandled — generic message, logged with requestId |

Codes are machine-readable constants (frontends branch on `error.code`, never on message text); messages are human-readable and safe to display.

---

## 7. REST API Design

### 7.1 Conventions

- **Versioning:** URI versioning — `/api/v1/...`. Chosen over header versioning because it is visible in logs, cacheable by path, trivially testable in a browser, and unambiguous for two frontend teams. Header versioning keeps URLs "pure" but hides the contract version from every log line and curl command; for one API with two known consumers, explicitness wins. `v2` would be introduced side-by-side only on a breaking change.
- **Status codes:** `200` read/update success · `201` created · `204` deleted (empty body, no envelope) · `400` malformed · `401` unauthenticated · `403` forbidden · `404` not found · `409` conflict · `422` validation/business rule · `429` throttled · `500` server error.
- **Pagination:** `?page=1&limit=20` (limit capped at 100, defaults `1`/`20`); responses carry the `meta` block (Section 6.2).
- **Filtering:** flat query params per resource, e.g. `GET /products?brand=chanel&category=eau-de-parfum&minPrice=50&maxPrice=200&q=noir` (slug-based filters; `q` = text search on name/description).
- **Sorting:** `?sort=price:asc` / `?sort=createdAt:desc` — single field, whitelisted per endpoint; invalid fields → `422`.
- **Auth column legend:** `public` = no token · `customer` = any authenticated user · `admin` = admin role. Guest carts use an `X-Cart-Token` (server-issued opaque token) merged into the user cart at login.

### 7.2 Auth — `/api/v1/auth`

| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/auth/register` | public | Create account + customer profile |
| POST | `/auth/login` | public | Exchange credentials for token pair |
| POST | `/auth/refresh` | public | Exchange refresh token for new pair |
| POST | `/auth/logout` | customer | Revoke refresh token |
| GET | `/auth/me` | customer | Current user identity + role |

**Register — `POST /api/v1/auth/register`**

```json
// Request
{
  "email": "mint@example.com",
  "password": "S3cure!pass",
  "fullName": "Mint Chaya",
  "phone": "+66812345678"
}

// Response 201
{
  "success": true,
  "data": {
    "user": { "id": "6f1e…", "email": "mint@example.com", "fullName": "Mint Chaya", "role": "customer" },
    "accessToken": "eyJhbGciOiJIUzI1NiIs…",
    "refreshToken": "v4.refresh.…",
    "expiresIn": 3600
  },
  "error": null,
  "meta": null
}
```

Duplicate email → `409 CONFLICT`. Weak password / bad email → `422 VALIDATION_FAILED`.

**Login — `POST /api/v1/auth/login`** — request `{ "email", "password" }`; response `200` with the same `data` shape as register. Bad credentials → `401 UNAUTHENTICATED` (single code for wrong email *or* password — no account enumeration).

### 7.3 Users — `/api/v1/users`, `/api/v1/admin/users`

| Method | Path | Auth | Description |
|---|---|---|---|
| GET | `/users/me` | customer | Get own profile |
| PATCH | `/users/me` | customer | Update own profile |
| GET | `/users/me/addresses` | customer | List own addresses |
| POST | `/users/me/addresses` | customer | Add address |
| PATCH | `/users/me/addresses/:id` | customer | Update address |
| DELETE | `/users/me/addresses/:id` | customer | Delete address |
| GET | `/admin/users` | admin | List users (paginated, `?q=&role=`) |
| GET | `/admin/users/:id` | admin | User detail with order summary |
| PATCH | `/admin/users/:id` | admin | Update role / active status |

### 7.4 Products — `/api/v1/products`, `/api/v1/admin/products`

| Method | Path | Auth | Description |
|---|---|---|---|
| GET | `/products` | public | List active products (paginate/filter/sort) |
| GET | `/products/:slug` | public | Product detail (images, variants, rating summary) |
| GET | `/products/:slug/reviews` | public | Approved reviews for a product |
| GET | `/admin/products` | admin | List all products incl. inactive/drafts |
| POST | `/admin/products` | admin | Create product |
| PATCH | `/admin/products/:id` | admin | Update product |
| DELETE | `/admin/products/:id` | admin | Soft-delete (deactivate) product |
| POST | `/admin/products/:id/images` | admin | Upload image (→ Supabase Storage) |
| DELETE | `/admin/products/:id/images/:imageId` | admin | Remove image |
| POST | `/admin/products/:id/variants` | admin | Add variant (size/SKU/price) |
| PATCH | `/admin/products/:id/variants/:variantId` | admin | Update variant |

**List products — `GET /api/v1/products?category=eau-de-parfum&brand=maison-noir&minPrice=50&maxPrice=200&sort=price:asc&page=1&limit=20`**

```json
// Response 200
{
  "success": true,
  "data": [
    {
      "id": "a1b2…",
      "slug": "noir-extrait-de-parfum",
      "name": "Noir Extrait de Parfum",
      "brand": { "id": "b9…", "name": "Maison Noir", "slug": "maison-noir" },
      "category": { "id": "c3…", "name": "Eau de Parfum", "slug": "eau-de-parfum" },
      "priceFrom": 89.00,
      "coverImageUrl": "https://…/storage/v1/object/public/products/noir-50.webp",
      "ratingAverage": 4.6,
      "ratingCount": 128,
      "variants": [
        { "id": "v1…", "sku": "MN-NOIR-50", "size": "50ml", "price": 89.00, "inStock": true },
        { "id": "v2…", "sku": "MN-NOIR-100", "size": "100ml", "price": 139.00, "inStock": false }
      ]
    }
  ],
  "error": null,
  "meta": { "page": 1, "limit": 20, "totalItems": 34, "totalPages": 2 }
}
```

### 7.5 Brands & Categories

| Method | Path | Auth | Description |
|---|---|---|---|
| GET | `/brands` | public | List brands |
| GET | `/brands/:slug` | public | Brand detail |
| POST | `/admin/brands` | admin | Create brand |
| PATCH | `/admin/brands/:id` | admin | Update brand |
| DELETE | `/admin/brands/:id` | admin | Delete brand (409 if products attached) |
| GET | `/categories` | public | Category tree |
| GET | `/categories/:slug` | public | Category detail |
| POST | `/admin/categories` | admin | Create category |
| PATCH | `/admin/categories/:id` | admin | Update category |
| DELETE | `/admin/categories/:id` | admin | Delete category (409 if in use) |

### 7.6 Inventory — `/api/v1/admin/inventory`

| Method | Path | Auth | Description |
|---|---|---|---|
| GET | `/admin/inventory` | admin | Stock levels per variant (`?lowStock=true`) |
| POST | `/admin/inventory/adjustments` | admin | Manual stock adjustment (+/- with reason) |
| GET | `/admin/inventory/movements` | admin | Stock movement ledger (`?variantId=&type=`) |

Stock is never set directly; every change is a `stock_movements` row (`purchase`, `sale`, `adjustment`, `return`) so current stock is auditable.

### 7.7 Cart — `/api/v1/cart`

| Method | Path | Auth | Description |
|---|---|---|---|
| GET | `/cart` | public* | Get current cart (guest via `X-Cart-Token`, customer via JWT) |
| POST | `/cart/items` | public* | Add variant to cart `{ variantId, quantity }` |
| PATCH | `/cart/items/:itemId` | public* | Change quantity |
| DELETE | `/cart/items/:itemId` | public* | Remove item |
| DELETE | `/cart` | public* | Clear cart |
| POST | `/cart/merge` | customer | Merge guest cart into user cart after login |

\* Guest access requires the `X-Cart-Token` issued on first cart write.

### 7.8 Orders — `/api/v1/orders`, `/api/v1/admin/orders`

| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/orders` | customer | Checkout: create order from cart |
| GET | `/orders` | customer | List own orders (paginated) |
| GET | `/orders/:id` | customer | Own order detail with items + status history |
| POST | `/orders/:id/cancel` | customer | Cancel (only `pending`/`paid` before shipping) |
| GET | `/admin/orders` | admin | All orders (`?status=&from=&to=&q=`) |
| GET | `/admin/orders/:id` | admin | Order detail |
| PATCH | `/admin/orders/:id/status` | admin | Advance status (validated transition) |

**Create order — `POST /api/v1/orders`**

```json
// Request
{
  "cartId": "ca11…",
  "addressId": "ad42…",
  "couponCode": "WELCOME10",
  "note": "Please gift-wrap."
}

// Response 201
{
  "success": true,
  "data": {
    "id": "or77…",
    "orderNumber": "ORD-2026-000481",
    "status": "pending",
    "items": [
      { "variantId": "v1…", "productName": "Noir Extrait de Parfum", "size": "50ml",
        "unitPrice": 89.00, "quantity": 2, "lineTotal": 178.00 }
    ],
    "subtotal": 178.00,
    "discount": 17.80,
    "shippingFee": 5.00,
    "total": 165.20,
    "coupon": { "code": "WELCOME10", "type": "percent", "value": 10 },
    "shippingAddress": { "recipient": "Mint Chaya", "line1": "…", "city": "Bangkok", "postalCode": "10110" },
    "createdAt": "2026-07-02T09:14:00Z"
  },
  "error": null,
  "meta": null
}
```

Failures: stock short → `409 INSUFFICIENT_STOCK`; coupon dead → `422 COUPON_EXPIRED`; empty cart → `422 VALIDATION_FAILED`. Prices are snapshotted into `order_items` at checkout — later price edits never mutate past orders.

### 7.9 Payments — `/api/v1/payments` (mock)

| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/payments` | customer | Pay for own pending order (mock gateway) |
| GET | `/payments/:id` | customer | Own payment detail |
| GET | `/admin/payments` | admin | All payments (`?status=&from=&to=`) |

**Mock payment — `POST /api/v1/payments`**

```json
// Request
{
  "orderId": "or77…",
  "method": "credit_card",
  "card": { "number": "4242424242424242", "expMonth": 12, "expYear": 2028, "cvc": "123" }
}

// Response 201
{
  "success": true,
  "data": {
    "id": "pa19…",
    "orderId": "or77…",
    "status": "succeeded",
    "amount": 165.20,
    "method": "credit_card",
    "transactionRef": "MOCK-TXN-8f31c2",
    "paidAt": "2026-07-02T09:15:07Z"
  },
  "error": null,
  "meta": null
}
```

Mock rules (deterministic, documented for graders/testers): card `4242…` → success, order becomes `paid`; card `4000000000000002` → `422 PAYMENT_DECLINED`, order stays `pending`; any other number → `422 VALIDATION_FAILED`. Paying a non-pending or already-paid order → `409 CONFLICT`. Card data is never persisted, even in mock mode.

### 7.10 Reviews — `/api/v1/reviews`

| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/reviews` | customer | Create review `{ productId, rating, title, body }` (must have a delivered order containing the product; one review per product per user) |
| PATCH | `/reviews/:id` | customer | Edit own review |
| DELETE | `/reviews/:id` | customer | Delete own review |
| GET | `/admin/reviews` | admin | All reviews (`?status=pending`) |
| PATCH | `/admin/reviews/:id/status` | admin | Approve / reject |

### 7.11 Wishlist — `/api/v1/wishlist`

| Method | Path | Auth | Description |
|---|---|---|---|
| GET | `/wishlist` | customer | List wishlist items |
| POST | `/wishlist/items` | customer | Add product `{ productId }` (409 if present) |
| DELETE | `/wishlist/items/:productId` | customer | Remove product |

### 7.12 Coupons — `/api/v1/coupons`, `/api/v1/admin/coupons`

| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/coupons/validate` | customer | Check code against current cart `{ code, cartId }` → discount preview |
| GET | `/admin/coupons` | admin | List coupons |
| POST | `/admin/coupons` | admin | Create coupon |
| PATCH | `/admin/coupons/:id` | admin | Update / deactivate |
| DELETE | `/admin/coupons/:id` | admin | Delete unused coupon |

### 7.13 Banners — `/api/v1/banners`

| Method | Path | Auth | Description |
|---|---|---|---|
| GET | `/banners` | public | Active banners ordered by position |
| GET | `/admin/banners` | admin | All banners |
| POST | `/admin/banners` | admin | Create banner (image via Storage) |
| PATCH | `/admin/banners/:id` | admin | Update / reorder / toggle |
| DELETE | `/admin/banners/:id` | admin | Delete banner |

### 7.14 Reports — `/api/v1/admin/reports`

| Method | Path | Auth | Description |
|---|---|---|---|
| GET | `/admin/reports/sales` | admin | Revenue over time (`?from=&to=&granularity=day\|month`) |
| GET | `/admin/reports/top-products` | admin | Best sellers by units/revenue |
| GET | `/admin/reports/orders-summary` | admin | Counts by status, average order value |
| GET | `/admin/reports/low-stock` | admin | Variants under threshold |

---

## 8. Design Decisions (Summary)

| # | Decision | Choice |
|---|---|---|
| D1 | Architecture | Modular monolith, controller → service → repository |
| D2 | Data access | Repository classes wrapping one shared Supabase client; no ORM |
| D3 | Auth | Supabase-issued JWTs, verified locally in `JwtStrategy`; server proxies all auth flows |
| D4 | RBAC | Global `JwtAuthGuard` + `@Public()` opt-out; `@Roles()` + `RolesGuard`; ownership in services |
| D5 | Contract | Uniform envelope `{ success, data, error, meta }` via interceptor + global exception filter |
| D6 | Validation | class-validator DTOs (whitelist + forbid unknown), business rules in services, DB constraints as backstop |
| D7 | Versioning | URI (`/api/v1`) over header versioning |
| D8 | Payments | Mock gateway with deterministic test cards |
| D9 | Checkout atomicity | Single PostgreSQL function (RPC) for order creation + stock decrement |
| D10 | Admin surface | Separate `/admin/*` controllers per module |

## 9. Reasoning

- **One deployable, many modules (D1):** matches team size and grading timeline; module boundaries provide the structure benefit of services without the operational tax. All consistency-critical flows (checkout, stock) stay in one transaction on one database.
- **Repository layer (D2):** centralizes case mapping, error translation, and query construction; makes services unit-testable; enforces the "only the server touches Supabase" rule at a greppable seam.
- **Local JWT verification (D3):** avoids a Supabase round-trip per request; the JWT secret is already shared with the server. Revocation gap is bounded by the 1-hour token lifetime.
- **Secure-by-default guarding (D4):** a forgotten decorator yields a 401, not a leak — the failure mode is annoying, not dangerous.
- **Envelope via interceptor (D5):** frontends write one response handler; the contract cannot drift per-endpoint because no endpoint constructs it manually.
- **Checkout as DB function (D9):** the Supabase JS client cannot span a transaction across multiple calls; a single RPC gives atomic order + stock + cart-clear with row locks preventing oversell under concurrency.

## 10. Alternatives & Trade-offs

| Alternative | Why rejected |
|---|---|
| Microservices | Distributed transactions for checkout, N deploy pipelines, no team boundary to justify it (Section 2.2) |
| Full tactical DDD / CQRS | Triple-modeling of simple entities; event machinery for flows a transaction expresses directly (Section 2.3) |
| Frontends calling Supabase directly (RLS) | Business rules (stock, coupons, order transitions) would live in RLS policies and triggers — hard to test/review; violates the single-API system rule; RLS remains enabled as defense in depth only |
| Prisma/TypeORM instead of Supabase client | Second connection path alongside Supabase Auth/Storage; migrations already owned by Supabase; ORM adds a layer the repository pattern already provides |
| Header/content-negotiation versioning | Invisible in logs and browser testing; overkill for two first-party consumers |
| GraphQL | Two known frontends with known screens; REST tables above are the cheaper contract; no third-party consumers demanding flexible queries |
| Cursor pagination | Offset (`page/limit`) is simpler for the dashboard's numbered pages; catalog sizes make offset cost negligible; revisit if feeds grow |
| Event-driven stock updates | Synchronous decrement inside the checkout transaction is simpler and cannot oversell |

## 11. Best Practices Applied

- Secure by default: global auth guard with explicit `@Public()` opt-out; admin routes namespaced and class-guarded.
- Fail fast: env schema validated at boot; unknown request fields rejected.
- Least privilege: service-role Supabase key server-side only; public responses exclude internal fields (cost price, internal notes).
- Observability: correlation id on every request, echoed in error envelopes; structured JSON logs; PII and credentials never logged.
- Immutable financial records: order item prices snapshotted; stock changes as append-only movements; admin mutations recorded in `audit_logs`.
- Rate limiting (`@nestjs/throttler`) on `/auth/*` → `429 RATE_LIMITED`; generic 401 on bad login (no enumeration).
- OpenAPI (`@nestjs/swagger`) generated from controllers + DTOs at `/api/v1/docs` — the contract in Section 7 stays executable documentation.
- Health endpoint `GET /api/v1/health` (public) for uptime checks.

## 12. Risks

| Risk | Impact | Mitigation |
|---|---|---|
| Stateless JWT revocation gap | Compromised/banned user keeps access ≤ 1 h | Short access-token TTL; refresh revoked on logout/ban; re-check role from DB on destructive admin actions |
| Module coupling creep (services importing services freely) | Monolith degrades into a tangle | Rule: cross-module via services only; dependency direction reviewed in PRs; `common` never imports feature modules |
| Checkout race conditions (oversell) | Negative stock, bad orders | Atomic RPC with row locks (D9); DB check constraint `stock >= 0` as backstop |
| Supabase vendor coupling | Migration cost if platform changes | Repository seam isolates client usage; auth flows proxied so frontends are Supabase-agnostic already |
| Envelope divergence during rapid development | Frontends break on inconsistent responses | Envelope applied only by interceptor/filter; e2e tests assert the envelope on every module |
| Mock payment mistaken for real design | Security gaps if later swapped for a real gateway | Payment module isolates the gateway call behind `payments.service`; card data already never persisted; real integration = replace one service method + add webhooks |
| Guest cart token theft | Guest cart tampering | Token is opaque, random, cart-scoped, carries no identity; worst case is a lost anonymous cart |

## 13. Recommendations

1. **Build order:** common (guards/filters/interceptors/Supabase provider) → auth → products/brands/categories → cart → orders + inventory → payments → reviews/wishlist/coupons/banners → reports. Frontends can start against products + auth early.
2. **Freeze the contract first:** generate the OpenAPI spec from Section 7 endpoints as stubs before implementing bodies, so salepage and dashboard develop in parallel.
3. **Test where the risk is:** unit-test services with mocked repositories (coupon math, status transitions, stock rules); e2e-test the checkout → payment → status flow and the envelope shape; skip exhaustive controller unit tests — they contain no logic.
4. **Write the checkout RPC early** and load-test it with concurrent orders on the same variant; it is the only concurrency-sensitive path in the system.
5. **Keep `reports` read-only and query-based** for now; introduce materialized views only if dashboard latency becomes a measured problem.
6. **Document mock payment test cards** in the dashboard README so graders can drive success and failure paths without reading code.

---

*End of Phase 5 — Backend Design. Next: Phase 6 — Frontend Design (salepage & dashboard).*
