# System Architecture

```mermaid
flowchart TB
    subgraph Clients
        Customer["Customer Website<br/>(Astro — salepage)"]
        Admin["Admin Dashboard<br/>(Nuxt.js — dashboard)"]
    end

    API["Backend REST API<br/>(NestJS — server)"]

    subgraph Supabase
        Auth["Authentication"]
        DB[("PostgreSQL")]
        Storage["Storage"]
    end

    Customer -->|REST / HTTPS| API
    Admin -->|REST / HTTPS| API

    API --> Auth
    API --> DB
    API --> Storage

    Customer -.->|Auth session| Auth
    Admin -.->|Auth session| Auth
```

## Components

- **Customer Website (Astro)** — public storefront; browse, cart, checkout.
- **Admin Dashboard (Nuxt.js)** — staff management UI.
- **Backend API (NestJS)** — REST API; business logic; single integration point to Supabase.
- **Supabase**
  - **Authentication** — customer & admin sessions.
  - **PostgreSQL** — products, orders, customers.
  - **Storage** — product images/media.

## Data Flow

1. A client (customer or admin) calls the NestJS API over HTTPS.
2. The API authorizes via Supabase Auth, reads/writes PostgreSQL, and stores media in Storage.
3. Responses return to the client as JSON.
