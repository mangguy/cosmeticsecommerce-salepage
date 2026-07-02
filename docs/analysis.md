# Analysis

## Project Introduction

Cosmetics E-Commerce Platform is an online store for selling **perfumes**. It
consists of a public customer website, an admin dashboard, and a backend API,
backed by Supabase for database, authentication, and file storage.

## Business Objective

- Sell perfume products online to end customers.
- Give staff a dashboard to manage catalog, orders, and customers.
- Provide a single, secure backend API shared by both frontends.
- Keep the system standard and maintainable so a team can collaborate via Git.

## Functional Overview

**Customer (salepage)**
- Browse and search products
- View product detail
- Cart & checkout
- Register / login (Supabase Auth)
- Track orders

**Admin (dashboard)**
- Manage products & inventory
- Manage orders
- Manage customers
- Upload product media (Supabase Storage)
- Role-based access

**Backend (server)**
- REST endpoints for products, orders, auth, media
- Business logic & validation
- Integration with Supabase (PostgreSQL, Auth, Storage)

## Actors

- **Customer** — browses and buys.
- **Admin** — manages the store via the dashboard.
- **System** — backend API + Supabase.
