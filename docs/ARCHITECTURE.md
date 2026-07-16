# ANT POS — Architecture Reference

> Update this file when routes, models, services, or project structure change.
> Last updated: 2026-07-10

## System overview

```
                         ┌──────────────────────────────────┐
  tenant1.domain ──────► │  React SPA (pos-frontend) —      │
  tenant2.domain ──────► │  POS admin + website             │
                         └───────────────┬──────────────────┘
                                         │  subdomain → tenant
                                         ▼
                         ┌──────────────────────────────────┐
                         │  Laravel 11 API (pos-backend)    │
                         │  stancl/tenancy v3               │
                         ├──────────────────────────────────┤
                         │ central DB: ant_pos_central      │
                         │ tenant DBs: ant_pos_{uuid}       │
                         └──────────────────────────────────┘
```

- Tenant identified by subdomain; each tenant has an isolated MySQL database.
- One SPA serves two UIs: POS admin (staff, Sanctum user tokens) and the public
  website (customers, Sanctum `auth:customer` guard).
- Timezone +05:45 (Nepal), VAT 13%, Nepali-date support.

## Backend (`pos-backend/`) — Laravel 11, PHP 8.3

### Key packages
`stancl/tenancy` (multi-tenancy), `laravel/sanctum` (auth), `spatie/laravel-permission`
(roles), `spatie/laravel-activitylog` (audit), `spatie/laravel-data`,
`phpoffice/phpspreadsheet` (Excel), `anuzpandey/laravel-nepali-date`,
dev: pint, larastan, phpunit 11, laravel/boost.

### Route files
| File | Prefix / guard | Purpose |
|---|---|---|
| `routes/central.php` | central domain, `auth:sanctum` | Platform login, profile, tenant creation |
| `routes/tenant.php` | wraps the below in `api` prefix per tenant | Tenant bootstrapping |
| `routes/api.php` | `/api`, `auth:sanctum` | The whole POS admin API (products, orders, inventory, cash, settings…) |
| `routes/admin/report.php` | `/api/report`, `auth:sanctum` | Reporting endpoints |
| `routes/website/guest.php` | `/api/website` public | Catalog: categories, attributes, product-groups, products, similar, feature flags |
| `routes/website/customerAuth.php` | `/api/website/customer` | Customer register/login/profile/logout/orders |
| `routes/website/cart.php` | `/api/cart`, `auth:customer` | Cart CRUD + checkout |
| `routes/website/order.php` | `/api/orders`, `auth:customer` | Customer order history |
| `routes/resource.php` | — | Defines the `Route::apiRoutes()` resource macro used by api.php |
| `routes/web.php` | web | Signed download of failed import rows |

### Controller layout
- `app/Http/Controllers/*` — POS admin controllers (one per module; generic CRUD
  driven by the resource macro + model hooks).
- `app/Http/Controllers/Website/*` — website controllers: ProductController,
  ProductGroupController (variants as groups), CartController, CustomerAuthController,
  OrderController, AttributeController, WebsiteSettingController.
- `app/Http/Controllers/Central/AuthController` — central login.
- `app/Http/Controllers/Api/SuperController` — base/super API controller.

### CRUD pattern (important)
Controllers use a shared resource pattern; models participate via hooks:
- `mergeRequest($id)` — inject computed fields before save (e.g. Product auto-code,
  auto-attach ProductVariant from SKU article number).
- `afterStore()` / `afterUpdate()` — post-save side effects (sync tags/attributes;
  ProductVariant bulk-creates child Products here).

### Domain model — the product/variant inversion (⚠️ read CLAUDE.md)
```
ProductVariant (parent "product group", code = SKU article number)
   └── hasMany Product (sellable unit: sku, price, stock)
          └── hasMany OrderItems / productables / stock transactions
```
`Product.mergeRequest()` splits `sku` on `-`; the prefix locates or creates the
parent ProductVariant. Both levels have tags/attributes/images pivots
(`product_tags`, `attribute_products` vs `product_variants_tags`,
`product_variants_attributes`).

### Models (app/Models)
Product, ProductVariant, Attribute, Tag, Category, Brand, Supplier, Store,
Customer, Order, OrderItems, Payment, Purchase, Productable, StockAdjustment,
InventoryStockTransaction, StockAudit(+Item/Result/Summary), SalesReturn,
SaleReturnItem, Cart, CartItem, CashSession, CashDenomination, Discount, Expense,
Image, Setting, Feature, Import, Exports, CustomerReturns (footfall), ActivityLog,
Role, User, Tenant, Location (self-referencing 3-level hierarchy via `parent_id`,
type = LocationTypeEnum), DeliveryFee (per-city fee, `location_id` FK).

### Services (app/Services)
- `Orders/CreateOrder` — order creation pipeline (stock deduction, payments).
- `Websites/CartService`, `Websites/WebsiteOrderService` — website cart/checkout.
- `StockTransactionService`, `ProductStockService`, `UpdateStockAdjustmentService`,
  `ProductableService` — inventory movements (always go through these).
- `CashDemoninationService` (sic), `FeatureService`,
  `ExcelDiscountProductReader`, `ExcelStockAuditReader`.

### Jobs (app/Jobs) — queued on the central `jobs` table
Queue: database driver; `DB_QUEUE_CONNECTION=mysql` pins the queue to the
central DB (tenant context round-trips via `tenant_id` in the payload —
stancl `QueueTenancyBootstrapper`). Worker: docker `queue` service.
- `ExportJob` — background Excel export. Controllers expose
  `static exportConfig()` (model/resource/dateColumn); `ExportExcel::exportToDisk()`
  writes to the tenant public disk; `exports` row tracks status. Endpoints:
  `POST /api/export/{uri}` (dispatch, immediate 201), `GET /api/exports`
  (+ `/{id}`), download via `GET /api/download/{path}`. Frontend Exports page:
  `pos-frontend/src/pages/exports/ExportList.jsx` (`/exports`).
- `ImportJob`, `ProcessStockAuditJob`.

### Enums (app/Enums)
OrderStatusEnum (Success/Pending/Draft), OrderTypeEnum (pos/website),
PaymentMethodEnum, DiscountTypeEnum (PERCENTAGE/FIXED/FIXED_PRICE),
StockUpdateTypeEnum (In/Out), StockUpdateReasonEnum, StockAuditEnum,
CartStatusEnum, CashSessionStatus, CashDenominationStage, BrandStatusEnum,
CategoryStatusEnum, ProductVariantStatusEnum, Sales/SaleReturnStatusEnum,
ImportStatusEnum, ExportStatusEnum (pending/processing/completed/failed),
ExportModelMap, FeatureKey (website_enabled/maintenance_mode),
RoleEnum, PermissionEnum, LocationTypeEnum (country/district/city, with
`parentType()` helper).

### Migrations
- Central: `database/migrations/` (tenants, domains, users, tokens, features).
- Tenant: `database/migrations/tenant/` — run with `php artisan tenants:migrate`.

### Newer tables NOT in `pos-backend/DATABASE_SCHEMA.md` (doc dated Oct 2025)
brands, categories (+categories_tags), sales_returns + sale_return_items,
carts + cart_items, images (polymorphic, thumbnail/order), cash_sessions,
cash_denominations, expenses, stock_audits (+items/results/summaries),
discounts + discountables, features (central), locations + delivery_fees
(Jul 2026); plus columns: orders.type,
orders.split_payments, orders.total_discount_amount, products.brand_id,
product_variants.status/image, customers login capability (customer_loginable).

## Frontend (`pos-frontend/`) — React 18 + Vite

### Stack
Vite 5, TypeScript + legacy JSX mix, MUI v6 (+ some Mantine v7, rsuite), Tailwind 3,
Redux Toolkit + redux-persist (auth, order form, brand slices), TanStack Query v5
(server state) + TanStack Table v8, react-router-dom v6, react-hook-form + zod,
chart.js, sonner (toasts), react-to-print.

### Structure (src/)
| Dir | Purpose |
|---|---|
| `routes.tsx` | All POS admin routes (guards: `AuthRoutes`/`GuestRoutes`) + mounts `websiteRoutes` |
| `pages/*` | One folder per module, typical files: `*List`, `Columns`, `Model` (form modal), `*Details` |
| `pages/website/` | Entire customer storefront: pages (Home, ProductDetail, Cart, Orders, Login, Register, TagProducts), `router/`, `layouts/WebsiteLayout`, `middlewares/WebsiteEnabledMiddleware`, own hooks/services |
| `api/` | Axios service modules (productService, orderService, websiteProductService, websiteOrderService, websiteAuthService, cartService…) |
| `redux/` | store, slices (authSlice, orderFormSlice, brandSlice), selectors |
| `components/` | Shared UI: Tables/CustomTable, Orders (billing, split billing, order summary), Products, Discount, Reports, Layout (sidebar/topnav), ui (shadcn-style) |
| `utilities/domain.ts` | Subdomain → tenant resolution (`VITE_DEFAULT_TENANT` fallback) |
| `guards/`, `hooks/`, `schemas/`, `types/`, `context/` | Route guards, shared hooks, zod schemas, TS types |

### Tenant/API wiring
The SPA derives the tenant from the browser subdomain (`getSubdomain()`), and API
calls target the tenant's backend. Local dev works with `tenant1.localhost`-style
hosts. Env: `VITE_DEFAULT_TENANT`.

### POS route map highlights
`/` dashboard · `/products` `/product-variants` (catalog) · `/orders` `/sales`
`/sales-return` `/invoice/:id` (sales) · `/website-orders` (website channel) ·
`/purchase` `/stock-adjustment` `/stock-transactions` `/stock-audit`
`/damage-products` (inventory) · `/cash-denominations` `/currency-config`
`/note-config` `/expenses` (cash) · `/discount` `/categories` `/brands` `/tags`
`/attributes` `/attribute-names` (catalog meta) · `/customers` `/suppliers`
`/stores` `/users` `/roles` (parties) · `/locations` `/delivery-fees` (shipping) · `/sales-report` `/products-report`
(reports) · `/pos-config` `/website-config` `/inventory-configs`
`/payment-methods` `/system-logs` `/footfall` (admin).

## Business rules quick reference

- **VAT (Nepal 13%, inclusive):** `pre_vat = sell_price / 1.13`;
  `vat_amount = final_total × 13 / 113`.
- **Order totals:** item total (qty×price or weight×price) → item % discount →
  order % discount → final flat discount → VAT extracted from result.
- **Stock:** every change creates an `InventoryStockTransaction` with
  previous/new stock and polymorphic source. Sales deduct, purchases add,
  adjustments/returns/audits correct.
- **Codes:** `PROD####`, `ORD####` — generated from the last row's code suffix.
- **Website orders:** created Pending via cart checkout; staff see them in
  `/website-orders` (`/api/orders/pending-website`) and confirm
  (`/api/order/{id}/confirm`).

## Testing & quality

- Backend: PHPUnit 11 (`php artisan test`), Pint formatting mandatory
  (`vendor/bin/pint --dirty`), Larastan available.
- Frontend: ESLint + Prettier; no test runner configured yet.

### Multi-tenant test setup (`tests/TestCase.php`)

Feature tests run on a **single sqlite `:memory:` database** that holds both
the central tables (tenants, domains, features, …) and all tenant tables —
no per-tenant database is created:

- The base `Tests\TestCase` uses `RefreshDatabase` and overrides
  `migrateFreshUsing()` to migrate `database/migrations` **and**
  `database/migrations/tenant` into the one DB (migrations with identical
  filenames — users/cache/jobs/personal_access_tokens — dedupe; the tenant
  version wins). Do **not** re-add `use RefreshDatabase;` to individual test
  classes: a trait on the child class silently overrides the base overrides.
- `setUp()` disables `tenancy.bootstrappers` (no DB switching), detaches the
  CreateDatabase/MigrateDatabase/DeleteDatabase job pipelines, empties
  `tenancy.central_domains`, creates a test `Tenant` + `Domain`, initializes
  tenancy, and sends the `X-Tenant` header on every request so
  `InitializeTenancyByHeader` resolves the tenant like in production.
- Validation errors are rendered by `ApiResponse::validationFailed` under an
  `error` key (not Laravel's `errors`), so tests use the base-class helper
  `assertApiValidationErrors($response, [...keys])` instead of
  `assertJsonValidationErrors`.
- `ReportController` date-grouping SQL is driver-aware (strftime on sqlite,
  DATE_FORMAT/WEEK/YEAR on MySQL) so the report tests can run on sqlite.
