# ANT POS — Workspace Guide

Multi-tenant Point of Sale system + e-commerce website for retail stores in Nepal.
Two projects live in this workspace:

| Directory | What it is |
|---|---|
| `pos-backend/` | Laravel 11 API (PHP 8.3, MySQL, Sanctum, stancl/tenancy). Serves both the POS admin API and the public website API. Has its own `CLAUDE.md` with Laravel Boost coding rules — follow it when working there. |
| `pos-frontend-with-react/` | React 18 + Vite SPA (mixed TS/JSX). Contains BOTH the POS admin app and the customer-facing website (under `src/pages/website/`). |

## ⚠️ CRITICAL: Product / Variant model is INVERTED vs. typical e-commerce

**Do not assume the usual e-commerce pattern.** In most systems the "product" is the
parent and "variants" are the sellable units. **Here it is the opposite:**

- **`Product` is the sellable unit.** It has the SKU, sell_price, stock, barcode, and
  is what appears on order items, purchases, stock transactions, and the website
  product detail page. Historical reason: the `products` table existed first and was
  always the thing sold; variants were added later.
- **`ProductVariant` is the PARENT / GROUPING** ("product group" / "bulk"). It groups
  sibling products (e.g. same shoe in different sizes). `products.product_variant_id`
  is a `belongsTo` FK pointing UP to the variant. The code often calls it
  `$productGroup` (see `Product::mergeRequest()`), and the website exposes variants
  via `/api/website/product-groups`.
- SKU convention: `{articleNumber}-{...}` — the part before the first `-` is the
  article number, which equals the `ProductVariant.code`. Creating a product
  auto-creates/attaches its variant from this prefix.
- Creating/updating a ProductVariant bulk-creates its child Products (see
  `ProductVariant::afterStore()`).

So: "add a variant" = add a parent grouping; "sell an item" = sell a **Product**.
Never move price/stock/SKU logic up to ProductVariant.

## Architecture in one paragraph

Stancl multi-tenancy, **database-per-tenant** (`ant_pos_central` + `ant_pos_{uuid}`),
tenant resolved by **subdomain** (frontend `src/utilities/domain.ts` extracts it,
e.g. `kstore.aitechnology.com.np`). Auth is Sanctum: central users (`routes/central.php`),
tenant staff users (`routes/api.php`, `auth:sanctum`), and website customers
(`auth:customer` guard, `routes/website/*.php`). One order pipeline serves both
channels — `orders.type` is `OrderTypeEnum: 'pos' | 'website'`. Nepal specifics:
13% VAT (`pre_vat = sell_price / 1.13`, VAT = total × 13/113), Nepali dates
(anuzpandey/laravel-nepali-date), NPR cash denominations.

## Documentation map (keep these updated!)

| File | Contents |
|---|---|
| `CLAUDE.md` (this file) | Workspace overview + the product/variant rule |
| `docs/FEATURES.md` | Full feature inventory for POS + website |
| `docs/ARCHITECTURE.md` | Backend/frontend structure, routes, models, services, conventions |
| `pos-backend/DATABASE_SCHEMA.md` | DB schema (written Oct 2025 — core tables accurate, but see the "Newer tables" note in docs/ARCHITECTURE.md for entities added since) |
| `pos-backend/CLAUDE.md` | Laravel Boost coding guidelines for backend work |

**Update policy:** whenever a feature is added/changed, update `docs/FEATURES.md`;
whenever structure/routes/models change, update `docs/ARCHITECTURE.md`; whenever a
migration is added, update `pos-backend/DATABASE_SCHEMA.md`. These docs are the
session-to-session memory of this project.

## Common commands

Backend (run in `pos-backend/`):
```bash
php artisan serve                 # dev server
php artisan test                  # PHPUnit tests
vendor/bin/pint --dirty           # format (required before finalizing changes)
php artisan migrate               # central migrations
php artisan tenants:migrate       # tenant migrations (database/migrations/tenant/)
```

Frontend (run in `pos-frontend-with-react/`):
```bash
npm run dev                       # Vite dev server
npm run build                     # tsc -b && vite build
npm run lint                      # eslint
npm run format                    # prettier
npm run deploy:<tenant>           # build + scp dist/ to a tenant subdomain
                                  # (kstore, kshoes, pahari, ventures, surya, kritipur, f2c, server, dev)
```

## Conventions worth knowing

- All tenant tables carry `created_by`, `updated_by`, `custom_fields` (json),
  `extras` (json), soft deletes. Codes auto-generate (`PROD0001`, `ORD0001`).
- Backend controllers use a shared resource pattern (`Route::apiRoutes(...)` macro,
  see `routes/resource.php`) with model hooks like `mergeRequest()`, `afterStore()`,
  `afterUpdate()`.
- Every stock movement goes through `InventoryStockTransaction` (type In/Out,
  polymorphic `source_id`/`source_tag`) — never mutate `products.stock` directly;
  use the services in `app/Services/`.
- Frontend is a TS/JSX mix; UI is MUI + some Mantine + Tailwind; state is Redux
  Toolkit (+redux-persist) for auth/order form, TanStack Query for server data.
- Feature flags via `features` table + `FeatureKey` enum (`website_enabled`,
  `maintenance_mode`); website checks it through `WebsiteEnabledMiddleware`.
