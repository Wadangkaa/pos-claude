# ANT POS — Architecture Reference

> Update this file when routes, models, services, or project structure change.
> Last updated: 2026-09-11

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
  website (customers, Sanctum `auth:customer` guard). The website owns `/` and
  customer paths such as `/product/:sku`, `/cart`, `/orders`, and `/login`;
  staff pages are all under `/pos`, including `/pos/login` and `/pos/dashboard`.
  Backend API paths are unchanged.
- Timezone +05:45 (Nepal), VAT 13%, Nepali-date support.

## Local Docker development

`docker compose up -d` starts the frontend, backend, database, and queue workers.
The frontend host port defaults to `3000`; set `POS_FRONTEND_PORT=3001` (or another
free port) in the workspace root `.env` to avoid conflicts with other projects.
Use `.env.example` as a starting point. Vite still listens on port `3000` inside
the container; access the frontend using the configured host port, including
tenant/branch localhost hostnames. The backend remains on port `8000`.

## Backend (`pos-backend/`) — Laravel 11, PHP 8.3

### Key packages
`stancl/tenancy` (multi-tenancy), `laravel/sanctum` (auth), `spatie/laravel-permission`
(roles), `spatie/laravel-activitylog` (audit), `spatie/laravel-data`,
`phpoffice/phpspreadsheet` (Excel), `anuzpandey/laravel-nepali-date`,
dev: pint, larastan, phpunit 11, laravel/boost.

### Route files
| File | Prefix / guard | Purpose |
|---|---|---|
| `routes/central.php` | central domain, `auth:sanctum` | Platform login, profile, tenant creation/deletion |
| `routes/tenant.php` | wraps the below in `api` prefix per tenant | Tenant bootstrapping; report routes enforce branch/admin access |
| `routes/api.php` | `/api`, `auth:sanctum` | The whole POS admin API (products, orders, inventory, cash, settings…) |
| `routes/admin/report.php` | `/api/report`, `auth:sanctum` + branch access | Reporting endpoints, including today's Daily expense total |
| `routes/website/guest.php` | `/api/website` public | Catalog: categories, attributes, product-groups (including `tag_id` filtering and raw/discount-adjusted child-product price ranges), products, similar, feature flags |
| `routes/website/customerAuth.php` | `/api/website/customer` | Customer signup/email verification, login/profile/logout/orders, password change/recovery |
| `routes/website/cart.php` | `/api/cart`, `auth:customer` | Cart CRUD + checkout |
| `routes/website/order.php` | `/api/orders`, `auth:customer` | Customer order history |
| `routes/resource.php` | — | Defines the `Route::apiRoutes()` resource macro used by api.php |
| `routes/web.php` | web | Signed download of failed import rows |

`routes/api.php` also exposes `PUT /api/order/{id}/fulfillment-branch`. It is
available only to `admin`/`super-admin` users on an admin hostname and moves a
pending website order's reservation to a sufficiently stocked branch.
It also exposes staff CRUD at `/api/staff` and daily attendance reads and marks
at `/api/staff-attendance` and `/api/staff-attendance/{staffId}/{date}`. Staff
records are separate from authenticated users. These routes require a branch
hostname and use its forced branch; the company admin hostname is forbidden.
Branch incentive plans are at `GET/POST /api/staff-incentive-plans` and
`PUT /api/staff-incentive-plans/{id}`; calculated daily and monthly points are
at `GET /api/staff-rewards/daily?date=YYYY-MM-DD` and
`GET /api/staff-rewards/monthly?month=YYYY-MM`. These routes also require a
branch hostname and the `hr-incentives-*` permissions.

Tenant authorization definitions live in `config/permissions.php`; names use
`{module}-{resource}-{action}`. `PermissionSeeder` is called for new tenants and
the `2026_09_15_000000_seed_model_permissions` tenant migration backfills existing
databases, granting all configured permissions to `super-admin` and `admin`.
The follow-up `2026_09_15_000001_seed_role_presets` migration synchronizes the
cashier/manager presets and their dashboard/POS lookup permissions.
The `2026_09_29_160939_expand_cashier_module_permissions` tenant migration
refreshes the cashier preset for sales returns, footfall, cash denominations and
sessions, expenses, purchases, and damage products. Damage listing/recovery use
`inventory-damage_products-view/update`; legacy stock-transaction viewers and
product editors retain their existing access. Purchase creators/editors may read
the supplier lookup without supplier-management permissions.
`TenancyServiceProvider` scopes Spatie's permission cache key to the tenant ID
after tenancy boots, resets loaded permissions on each context switch, and
restores the central key after tenancy ends. This isolates role IDs between
tenant databases, including requests and workers that visit multiple tenants.
`UserResource` returns effective roles and permissions on login and `GET
/api/profile`. The branch-only `LeftSidebarMenu` filters items by their `view`
permission; the company-admin menu deliberately remains independent.
`styles/left-sidebar-menu.css` uses border-box sizing for accordion summaries
and disables horizontal overflow on the inner scroll container while retaining
vertical navigation.
`RoleSeeder` creates `super-admin`, `admin`, `manager`, and `cashier` presets;
the cashier's `pos-product_lookup-view` permission is intentionally distinct
from `inventory-products-view`, which controls catalog navigation. User CRUD
accepts `role_names` and synchronizes those roles through Spatie.
`StoreUserRequest` defaults branch-host user writes to the hostname's forced
branch and rejects foreign assignments. The user form loads and shows branch
choices only on the company admin portal.
The Roles API and table prevent the built-in presets from being edited or deleted;
custom roles retain the editable permission matrix.
`EnforceTenantPermission` runs after Sanctum authentication and branch assignment
for staff API and report routes. It maps each action to its configured permission
and denies unmapped actions. POS product reads accept either the catalog view or
POS lookup permission; catalog writes require their own action permission.
`role/all` is also readable by staff who can create or edit users, limited to the
roles they may assign. `EnsureWebsiteEnabled` guards the storefront route files
(`routes/website/*`, except `get-features`) and answers 403 while the tenant's
`website_enabled` feature is off. Staff login on the admin hostname requires the
`admin` or `super-admin` role.
Dashboard view permission also allows its specific summary/count requests and
brand lookup. Sales/payment viewers can read settings lists filtered to exactly
`payment`, `payment-status`, or `sales-status`; unrestricted settings reads and
configuration writes retain their own permissions. These dependencies let
cashiers use the dashboard and sales form without granting management access.
Company summary and branch mutations require the admin hostname, and the
summary also requires an administrator role. Role editors cannot grant a
permission they lack; user editors cannot assign administrator roles or roles
with permissions they lack. Branch-host user management sees only users assigned
to that branch. Deleting a user assigned to multiple branches requires the
company admin portal. Roles remain tenant-wide across a user's assigned branches.
Queued catalog and report exports carry the hostname's branch context into the
worker, and a branch-host report export ignores a supplied foreign `branch_id`.

`TenantController::store()` creates both the website and `admin.` hostname
records in the central `tenant_branch_domains` table. The tenant seeder creates
the `Main` branch, gives the tenant-email user `super-admin`, and assigns that
user to Main. `Branch::afterStore()` derives and stores a branch hostname from
the website hostname and branch name.

`DELETE /api/central/tenants/{tenantId}` requires `confirmation` equal to the
tenant's domain. It deletes the central tenant record (cascading its domains,
features, and branch-host mappings) and dispatches Stancl's synchronous
`DeleteDatabase` pipeline to drop the isolated tenant database.

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
Product, ProductVariant, Attribute, Tag, Category, Brand, Supplier, Branch,
Customer, StaffMember, StaffAttendance, Order, OrderItems, Payment, Purchase, Productable, StockAdjustment,
InventoryStockTransaction, BranchProductStock, StockTransfer, StockAudit(+Item/Result/Summary), SalesReturn,
SaleReturnItem, Cart, CartItem, CashSession, CashDenomination, Discount, Expense,
Image, Setting, Feature, Import, Exports, CustomerReturns (footfall), ActivityLog,
Role, User, Tenant, Location (self-referencing 3-level hierarchy via `parent_id`,
type = LocationTypeEnum), DeliveryFee (per-city fee, `location_id` FK).

### Services (app/Services)
- `Orders/CreateOrder` — order creation pipeline (stock deduction, payments).
- `Websites/CartService`, `Websites/WebsiteOrderService` — website cart/checkout;
  `Websites/WebsiteOrderNotificationService` sends customer receipts and configured
  admin order notifications after checkout, plus queued customer emails when
  website delivery status changes. Checkout emails eager-load `orderItems.product`
  to list the sellable products' SKUs and quantities; both show the saved
  `orders.total_amount` in NPR.
  `WebsiteOrderDeliveryStatusNotification` captures the order code and configured
  label before queueing (the old enum is supported only for existing queued jobs);
  unchanged statuses and customers without a valid email do not dispatch it.
- `Websites/WebsiteFulfillmentService` — locks branch balances, selects and
  reserves a fulfillment branch for a pending checkout, consumes reservations on
  confirmation, releases them on cancellation, and safely reassigns them for
  authorized administrators. Legacy pending orders without reservations remain
  confirmable and cancellable.
- `BranchContext` — holds the tenant-domain-selected forced branch or portal type;
  `ScopesToBranch` applies that branch to Eloquent operational models.
  This includes stock transactions, damage listings, footfall (`CustomerReturns`),
  cash sessions, stock audits, and export history. `AssignsBranchOnCreate` assigns
  footfall, cash sessions, and audits to the forced branch (or oldest branch for
  company writes) and prevents subsequent reassignment. Cash denominations
  inherit branch ownership through their scoped session.
  `Exports` assigns the request's forced branch; company-wide exports may have
  a null branch. History/detail endpoints also enforce requesting-user ownership,
  and `DownloadController::download()` checks both for files under `exports/`.
  Stock audit workers restore their stored branch context and reset the previous
  context afterwards; comparisons read that branch's catalog and balances.
  Sales/draft, purchase, stock-adjustment, and discount product-assignment rules
  reject foreign-branch product IDs before saving.
- `CatalogBranch` and `BelongsToCatalogBranch` — assign and scope Products,
  ProductVariants, Tags, Categories, Attributes, Carts, Settings, Suppliers,
  Brands, Discounts, Locations, and DeliveryFees to a branch. All rows in `settings` (including website, email, POS, payment modes,
  statuses, currencies, denominations, weightable, and attribute names) are branch scoped. On the
  company admin host, catalog writes require `branch_id` when more than one
  branch exists. Product SKU uniqueness is `(branch_id, sku)`; the internal
  product code remains tenant-wide. The public website on a branch hostname
  uses that branch, and the company website defaults to the oldest branch.
  Customers have no branch scope, while their carts do.
- `BelongsToOrderBranch` scopes `Payment` and `OrderItems` reads, updates, and
  deletes through their parent order on a branch hostname. Their request rules
  also reject a foreign `order_id` on create and update; direct order-item
  requests validate the sellable `product_id` in the active branch.
  `OrderItems::product`
  and `products` deliberately bypass only the catalog branch read scope so
  historical order lines remain readable after their product moves to Main;
  direct catalog queries and writes remain branch scoped. Attribute requests
  require an attribute-name setting from the owning catalog branch. The
  low-quantity report uses an optional balance join and treats a missing
  branch balance as zero while filtering products by branch.
- `BranchStockService`, `StockTransactionService`, `ProductStockService`, `UpdateStockAdjustmentService`,
  `ProductableService` — inventory movements (always go through these).
- `CashDemoninationService` (sic), `FeatureService`,
  `ExcelDiscountProductReader`, `ExcelStockAuditReader`.

### Cutover tooling

`branches:cutover-audit {target-tenant-id} --source={source-tenant-id}:{target-branch-code}`
is a read-only Artisan command for the Caliber consolidation. It initializes
each tenant in turn, compares source catalog SKUs with the target branch's SKUs
and normalized customer phones with the tenant, and records source sales/return totals alongside the target
branch baseline. It intentionally has no write mode: a data import must be
reviewed and authorized after the audit report reconciles.

### Jobs (app/Jobs) — queued on the central `jobs` table
Queue: database driver; `DB_QUEUE_CONNECTION=mysql` pins the queue to the
central DB (tenant context round-trips via `tenant_id` in the payload —
stancl `QueueTenancyBootstrapper`). The docker `queue` worker handles default
jobs and the dedicated `exports` worker handles Excel/report exports. Its
30-minute worker timeout is paired with `DB_QUEUE_RETRY_AFTER=1860` seconds,
preventing a slow workbook from being executed twice.
`MailConfigServiceProvider` loads branch email settings on Stancl's
`TenancyBootstrapped` event, after the tenant database connection is active;
queued tenant jobs therefore must not query mail settings during
`TenancyInitialized`.
The provider restores central mail settings and clears resolved transports when
tenancy ends and before loading the next tenant, preventing SMTP credentials or
sender addresses from carrying over between queue jobs. `BranchContext::set()`
also emits `branch.context.changed`; the provider restores baseline mail settings
before loading the selected branch's `email_config`, with no fallback to another
branch's credentials. Queued customer verification/reset and order notifications
carry `UseBranchContext` middleware; order notifications select the order's branch,
and authentication notifications capture the originating website branch. The
middleware restores the worker's previous context even after failures.
The queue-level `JobFailed` listener re-enters the payload's tenant context and
marks failed `ExportJob`/`ReportExportJob`/`StockAuditExportJob` records as
failed, including errors raised before the job's `handle()` method can run.
- `ExportJob` — background Excel export. Controllers expose
  `static exportConfig()` (model/resource/dateColumn, with optional worksheet
  definitions); `ExportExcel::exportToDisk()`
  writes to the tenant public disk; `exports` row tracks status. Endpoints:
  `POST /api/export/{uri}` (dispatch, immediate 201), `GET /api/exports`
  (+ `/{id}`), download via `GET /api/download/{path}`. Frontend Exports page:
  `pos-frontend/src/pages/exports/ExportList.jsx` (`/pos/exports`).
  The Sales workbook config produces an `Orders` summary sheet (including the
  overall order discount) plus an `Order Items` detail sheet.
- `ReportExportJob` — same pattern for `POST /api/report/export`; report data is
  plain aggregated arrays (not Eloquent models), so `ReportExportController`
  exposes static `resolveReportData()`/`buildSpreadsheet()` helpers the job
  calls directly. Shares the `exports` table/page with model exports.
  ⚠️ Two bugs fixed here (2026-08): `ReportExportController::REPORT_PARAMS`
  must list every param a report method reads (`branch_id`, `limit`, `threshold`, `sort`,
  `sort_by`, `sort_direction`, `category_id`, on top of `brand_id`/dates) —
  anything missing gets silently dropped and the export falls back to the
  report's own default (top 20/10/5), quietly disagreeing with what the user
  picked on screen. And `limit` can be the string `"all"`; passing that to
  `->limit()` casts to `0` = `LIMIT 0` = empty file. Both report controllers
  now go through `ReportController::applyResultLimit()`, which only applies
  the cap when `is_numeric($limit) && (int) $limit > 0`.
- `StockAuditExportJob` — `POST /api/stock-audit/{id}/export` queues a completed
  audit's workbook. `StockAuditExport` writes the All, Matched, Mismatch,
  Missing in System, and Missing in Physical result sheets; the finished file
  is listed and downloaded through the shared Exports page.
- `ImportJob`, `ProcessStockAuditJob`.
- `SendDailyDashboardReport` — a small default-queue job sends the captured
  `daily_dashboard_reports.summary` as an in-memory XLSX attachment through
  `DailyDashboardReportMail`; retries up to three times and records sent/failure
  state. A locked report row prevents a repeated job from sending an already sent
  report, and the job verifies its original tenant ID. These system reports use
  their own table, rather than user-requested exports.

### Daily dashboard scheduler

`SavePosConfigRequest` validates nullable `daily_report_time` (HH:MM) and
`daily_report_email` as a pair in the existing `settings/pos-config` endpoints.
`settings.value_json` stores them per branch; saving omitted fields preserves prior values.
The UI at `/pos/pos-config` shows Nepal time and a separate report recipient.
`reports:send-daily-dashboard` is registered every minute in `routes/console.php`
with an overlap lock. It visits active tenants and their active branches, checks each branch's configured time in
`Asia/Kathmandu`, captures all-brand rows from `DashboardSummaryService`, and
creates one `daily_dashboard_reports` row per branch/date before queuing delivery.
The service also supplies the dashboard's payment-mode, Sales Return, and Daily
Expenses endpoints, preserving their brand and selected/forced branch filters.
Scheduled reports use only their configured branch's totals and SMTP settings. The
command and delivery worker restore the prior branch context after processing. Excel contains numeric NPR
values with text titles; the four stock/product/supplier/customer counts are excluded.

Docker Compose runs a `scheduler` service with `php artisan schedule:work`,
alongside the default queue worker. Outside Docker, keep a queue worker running
and install a cron entry that invokes `php artisan schedule:run` every minute
from the backend directory. Apply `php artisan tenants:migrate` before enabling
reports on existing tenants. Clearing both report fields disables future reports.

### Enums (app/Enums)
OrderStatusEnum (Success/Pending/Draft/Cancelled), OrderTypeEnum (pos/website),
WebsiteOrderDeliveryStatusEnum (pending/completed),
PaymentMethodEnum, ExpenseTypeEnum (daily/overall),
DiscountTypeEnum (PERCENTAGE/FIXED/FIXED_PRICE),
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

`2026_09_29_163706_scope_operational_records_to_branches` adds nullable branch
foreign keys to `customer_returns`, `cash_sessions`, `stock_audits`, and `exports`.
It requires the branch architecture migrations first and supports retry after a
partial MySQL schema change. Historical rows are not assigned by guesswork;
records with an unknown branch remain accessible from the company admin portal.

Expenses use `ExpenseTypeEnum` (`daily`/`overall`). The additive expense migration
defaults existing rows to Daily and adds a private `bill_image_path`; uploaded
bills are fetched only through authenticated `GET /api/expense/{id}/bill-image`.
The custom expense upload controller applies `Expense::mergeRequest()` when
creating a record, assigning the forced branch or the default branch on the
company portal. Request-supplied branch IDs cannot override this assignment;
expense lists, details, and bill images respect the same branch scope.
`GET /api/report/total-daily-expenses` sums Daily expenses whose expense date is
today, independent of sales, cash, or stock accounting.
`GET /api/report/sales/daily` also groups all expenses by expense date and type,
returning `expenses`, `daily_expenses`, and `overall_expenses` on each date row.
These values are separate from Net Sales and pass through the generic daily
report export. Expenses have no brand relation, so brand-filtered sales rows
still show expenses across all brands.
All four sales-period endpoints (`/api/report/sales/{daily,weekly,monthly,yearly}`)
accept optional `order_type=all|pos|website` (default `all`). The filter joins
payments and sales returns to `orders.type` so sales, counts, returns, and the
daily payment summary stay in the selected channel. `ReportExportController`
forwards the same filter to queued exports. Expense figures remain across all
order types because expenses are not linked to orders.

### Newer tables NOT in `pos-backend/DATABASE_SCHEMA.md` (doc dated Oct 2025)
brands, categories (+categories_tags), sales_returns + sale_return_items,
carts + cart_items, images (polymorphic, thumbnail/order), cash_sessions,
cash_denominations, expenses, daily_dashboard_reports, stock_audits (+items/results/summaries),
discounts + discountables, features (central), locations + delivery_fees
(Jul 2026); plus columns: orders.type,
orders.split_payments, orders.total_discount_amount, products.brand_id,
product_variants.status/image, customers login capability (customer_loginable).
Branch architecture additions (Sep 2026): central `tenant_branch_domains`; tenant
`branches`, `branch_user`, `branch_product_stocks`, `stock_transfers`, and
`stock_transfer_items`; `branch_id` on operational documents and inventory ledger;
and `branch_cutovers`, the idempotency and reconciliation record for an approved
single-branch-tenant import. The Sep 25 catalog migration adds `branch_id` to
products, product_variants, tags, categories, attributes, carts, and attribute
name rows in settings. It assigns
existing catalog data to the oldest branch, moves current stock there, and
leaves other branch catalogs empty. Historical order and stock ledger references
are retained. Original balances are captured in `branch_catalog_stock_backups`
before the live balances change. Transfers resolve the destination product by SKU.
The HR migration adds tenant `staff_members` and `staff_attendances`, each with
`branch_id`; attendance has a unique staff/date key and daily present/absent
status. It also seeds `hr-staff_members-*` and `hr-attendance-*` permissions.
Existing manager roles receive these HR permissions; cashier roles do not.
The permission middleware additionally requires a branch hostname for every HR
action, so company administrators cannot use HR from the admin portal.
The Sep 26 incentive migration adds `staff_incentive_plans` (one plan per
branch/effective date) and child `staff_incentive_tiers` (nonoverlapping,
lower-inclusive/upper-exclusive sales ranges and integer points). It seeds
`hr-incentives-view/create/update` and grants them to managers. Plan history
is effective-dated; plans from before today cannot be edited. The
`StaffIncentiveService` calculates daily points using the same payment and
sales-return dates as `ReportController::dailySalesReport`, scoped to the
hostname branch. It divides each day's integer award into 10,000-unit point
fractions among present staff and aggregates these live calculations by month.
There is no payroll money conversion or stored point ledger yet.
`IncentivesPage.jsx` presents plan tiers as separate sales-from, sales-below,
and points-to-share columns. Sales amounts use grouped NPR values and point
displays retain up to four decimal places without trailing zeros. Formatting
does not change the original tier values used when editing or saving. Offer
tables, point tables, and the plan dialog adapt to narrow screens.

## Frontend (`pos-frontend/`) — React 18 + Vite

### Stack
Vite 5, TypeScript + legacy JSX mix, MUI v6 (+ some Mantine v7, rsuite), Tailwind 3,
Redux Toolkit + redux-persist (auth, order form, brand slices), TanStack Query v5
(server state) + TanStack Table v8, react-router-dom v6, react-hook-form + zod,
chart.js, sonner (toasts), react-to-print.

### Structure (src/)
| Dir | Purpose |
|---|---|
| `routes.tsx` | All `/pos/*` staff routes (guards: `AuthRoutes`/`GuestRoutes`) + mounts root-level `websiteRoutes` |
| `pages/*` | One folder per module, typical files: `*List`, `Columns`, `Model` (form modal), `*Details` |
| `pages/website/` | Entire customer storefront: pages (Home, ProductDetail, Cart, Orders, Login, Register, TagProducts), `router/`, `layouts/WebsiteLayout`, `middlewares/WebsiteEnabledMiddleware`, own hooks/services. TagProducts reuses Home's catalog grid, sidebar filters, and pagination with a `tag_id` filter; navbar tag links carry their parent `category_id` so the matching sidebar category is checked. |
| `api/` | Axios service modules (productService, orderService, websiteProductService, websiteOrderService, websiteAuthService, cartService…) |
| `redux/` | store, slices (authSlice, orderFormSlice, brandSlice), selectors |
| `components/` | Shared UI: Tables/CustomTable, Orders (billing, split billing, order summary), Products, Discount, Reports, Layout (sidebar/topnav), ui (shadcn-style) |
| `utilities/domain.ts`, `utilities/routePaths.js` | Browser hostname → tenant resolution; POS path and legacy `/website/*` redirect helpers |
| `guards/`, `hooks/`, `schemas/`, `types/`, `context/` | Route guards, shared hooks, zod schemas, TS types |

### Tenant/API wiring
The SPA derives the tenant from the browser subdomain (`getSubdomain()`), and API
calls target the tenant's backend. Local dev works with `tenant1.localhost`-style
hosts. Env: `VITE_DEFAULT_TENANT`.

### Frontend route map highlights
The website Navbar uses a MUI account menu with customer initials and name,
Orders/Logout links, and a `ChangePasswordDialog`. It calls authenticated
`PUT /api/website/customer/change-password`; `ChangeCustomerPasswordRequest`
verifies the current password using the customer guard, requires a different
new password (minimum eight characters), and validates confirmation. The
controller hashes the new password on the authenticated customer. Checkout
shows a permanently selected Cash on Delivery radio option; the existing cash
order pipeline is unchanged.
Customer recovery uses public, rate-limited `POST
/api/website/customer/forgot-password` and `POST
/api/website/customer/reset-password` routes. The `customers` password broker
uses the active tenant database's `customer_password_reset_tokens` table, separate
from staff tokens; credentials require `has_login`. Reset tokens are hashed,
expire after 60 minutes, and are deleted on success along with customer API tokens.
`CustomerPasswordResetNotification` queues mail using the originating branch's email settings.
Its frontend URL is captured before queueing and built from trusted tenant data
and `config/customer_auth.php`, never the request Origin or Host headers.
`CUSTOMER_FRONTEND_URL` supports `{domain}` and `{tenant}` placeholders; it defaults
to `https://{domain}` in production and `http://{tenant}.localhost:3000` locally.
Set the local port in this variable when using a different frontend port.
Frontend `/forgot-password` and `/reset-password?token=...&email=...` routes handle
the flow; shared `PasswordInput`/`PasswordTextField` components add eye toggles.

New website registrations set `customers.email_verification_required = true`
and clear `email_verified_at`. Login rejects pending verification with a 403
response that sends the frontend to `/verify-email`. Existing accounts keep the
flag's default `false`; their email is not falsely marked verified.
`CustomerEmailVerificationService` queues `CustomerEmailVerificationNotification`
using the same trusted frontend URL configuration as password recovery. Links
contain a relative signed `GET /api/website/customer/verify-email/{id}/{hash}` URL
with a 60-minute expiry and signed tenant ID; verification checks the current
tenant and email hash before recording `email_verified_at`. Public `POST
/api/website/customer/verification-email` resends only for pending login-enabled
accounts, with a 60-second per-customer cooldown and generic confirmation.
Signup, resend, and verification have separate rate-limit buckets. The frontend
only follows verification paths matching this API route.

The expense modal uses `src/pages/expense/expenseSchema.js` to accept numeric
amounts from saved records and strings from the number input. It converts valid
amounts to strings for multipart submission, including unchanged edits and resets.

Customer storefront: `/` home, `/product/:sku`, `/tag/:id`, `/cart`,
`/orders`, `/orders/:id`, `/login`, `/register`. The former `/website/*`
paths redirect client-side to their root-level equivalents, preserving queries.
Staff: `/pos` redirects to `/pos/dashboard`; other pages include
`/pos/sales`, `/pos/orders`, `/pos/sales-return`, `/pos/website-orders`,
`/pos/products`, `/pos/stock-audit`, `/pos/exports`, `/pos/sales-report`,
`/pos/staff`, `/pos/attendance`, `/pos/incentives`, `/pos/website-config`, and `/pos/login`. Every
staff route is under `/pos`.

### Storefront footer configuration

`GET /api/settings/website-details` exposes public branding plus nullable
`footer_description`, `facebook_url`, `instagram_url`, `tiktok_url`, `x_url`, and
`map_embed_url`. Staff save these through the existing authenticated
`POST /api/settings/website-details` endpoint and `SaveWebsiteDetailsRequest`.
Website, email, and POS config reads/upserts use `(branch_id, key)`. Branch
hostnames always force their own branch; the company website uses Main (oldest
branch). Company administrators can select a branch in Website/POS Configuration
and pass `branch_id`; ordinary branch users cannot override their hostname.
Updates merge only supplied validated fields into the existing tenant
`settings.value_json` for `website_details`, preserving uploaded branding and
omitted footer values. No migration is needed. The request validates social URL
protocols and extracts an HTTPS `src` from pasted iframe HTML; only that URL is
stored. `WebsiteFooter.tsx` renders plain text, external links, and a titled,
sandboxed, lazy-loaded iframe inside `WebsiteLayout`, exclusively for storefront
routes. The admin footer form saves separately and invalidates the
`website-config` query cache.

### Website-order fulfilment

Staff manage website orders through `POST /api/order/{id}/confirm`,
`POST /api/order/{id}/cancel`, `PUT /api/order/{id}/delivery-status`, and
`PUT /api/order/{id}/payment-status`.
`PUT /api/order/{id}/notes` uses `UpdateWebsiteOrderNotesRequest` and accepts
staff-only, nullable text up to 5,000 characters. It updates
`orders.custom_fields.admin_notes` while preserving other custom fields;
`OrderResource` exposes `admin_notes` only to staff, and the admin details page
provides a Notes textarea and Save Notes button. No migration is needed.
The notes, delivery-status, and payment-status actions require
`sales-orders-update` through `EnforceTenantPermission`.
`ProductDetail.tsx` opens its existing gallery images in a full-screen MUI Dialog
from the main desktop/mobile photo. The dialog keeps its own selected image index,
supports cyclic buttons and arrow keys, and provides Escape/close controls,
focus management, and background scroll locking.
Cancellation is allowed only while a website order remains pending. Checkout
sets delivery to Pending, using a matching `settings.key = sales-status` entry
when available. Confirmation initializes missing historical delivery status but
preserves an existing selection. `orders.delivery_status_id` is a nullable FK to
that setting; `delivery_status` retains a label snapshot and supports legacy
`pending`/`completed` values. The `Order.deliveryStatus` relation is eager-loaded,
and both order resources expose its current label with the snapshot as fallback.
The tenant migration links matching historical labels and defaults unset website
delivery statuses to Pending without changing POS orders.
`UpdateWebsiteOrderDeliveryStatusRequest` accepts a Sales Status ID (or null to
return to Pending), rejects other setting keys, and requires staff authentication.
Only accepted website orders can be updated; row locking prevents duplicate
change notifications. Updates are tracking-only and do not create stock or payment
records. The admin dropdown loads all pages of configured Sales Status entries;
the customer order history shows the same label. Removing a setting nulls its FK
while preserving the saved label. No ordering of the configured statuses is assumed.
`UpdateWebsiteOrderPaymentStatusRequest` validates an active `payment-status`
setting ID (or null to return to Pending). The staff-only endpoint locks an accepted
website order and merges the ID and label into `orders.custom_fields`, preserving
other metadata and payment/stock records. Checkout selects configured Pending when
available, otherwise keeps the Pending label. `Order.paymentStatusId` exposes the
existing JSON ID as an accessor for the eager-loaded `paymentStatus` relation;
both resources expose the live configured label with its saved snapshot as fallback.
No migration is needed. `WebsiteOrderStatusPanel.jsx` loads all configuration pages
for both dropdowns and saves each status independently in a compact toolbar;
Save buttons appear only for changed selections.

Sales barcode lookups in `Productable.jsx` prepare a reusable audio element during
the scan and call `utilities/barcodeErrorSound.js` on failed or empty results.
The helper restarts playback for every error and preserves the visual error if
playback is unavailable. Vite bundles `src/assets/audio/error.wav` from its import
URL; the audio file was moved from the workspace root.
`POST /api/export/order` produces only POS-channel sales; Website Orders calls
`POST /api/export/website-orders-export`, backed by `WebsiteOrderExportController`,
to create a workbook limited to website-channel orders.

## Business rules quick reference

- **VAT (Nepal 13%, inclusive):** `pre_vat = sell_price / 1.13`;
  `vat_amount = final_total × 13 / 113`.
- **Order totals:** item total (qty×price or weight×price) → item % discount →
  order % discount → final flat discount → VAT extracted from result.
- **Stock:** every change creates an `InventoryStockTransaction` with
  previous/new stock and polymorphic source. Sales deduct, purchases add,
  adjustments/returns/audits correct.
- **Codes:** `PROD####`, `ORD####` — generated from the last row's code suffix.
- **Website orders:** created Pending via cart checkout; staff see all website-channel
  orders in `/pos/website-orders`, can filter by Pending/Confirmed status, and confirm
  them through `/api/order/{id}/confirm`. Confirmation updates the same row to
  Success and retains `type = website`; pending orders reserve stock by summing their
  quantities during an atomically locked checkout, and confirmation deducts that
  reserved inventory. Checkout requires a deliverable country → district → city
  selection and accepts an optional `delivery_landmark`; staff order details receive
  the country, district, city, and landmark as `delivery_address`. Website
  checkout validates contact fields through `WebsiteCheckoutRequest`: required
  `delivery_phone` and optional `delivery_alternative_phone`. Both are stored on
  `orders`, returned by `OrderResource`, and shown in staff website-order details
  separately from the customer profile. Spaces, parentheses, and hyphens are
  removed before validating 7–15 digits with an optional leading `+`.
  Website product and cart responses expose `available_stock`
  (physical stock less Pending website reservations) for the storefront quantity
  controls. The POS Sales view filters to `type = pos`.
- **Branch host isolation:** a `tenant_branch_domains` record maps the full browser
  hostname to the tenant, portal, and optional forced branch. A forced branch wins
  over a request `branch_id`; company administrators use the `admin.` host for
  consolidated and selected-branch dashboards/reports. The public website on
  each branch hostname lists only that branch's catalog.

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

### Branch configuration migration (Sep 29, 2026)

`2026_09_29_175730_scope_all_settings_and_daily_reports_to_branches` assigns
previously global settings (including saved footer and SMTP credentials) to Main.
It creates independent payment/status/currency/denomination/weight defaults in
other existing branches and relinks their historical order/payment setting IDs.
It does not copy footer, email credentials, or POS report/notification recipients
to other branches. New branches receive independent operational defaults through
`BranchSettingDefaults`; admin-created catalog/configuration records require a
branch selection. Cash selection and required cash entry use the configured
method's value rather than assuming setting ID 1.

`2026_09_29_181439_scope_business_reference_data_to_branches` adds nullable
branch FKs to suppliers, brands, discounts, locations, and delivery fees. It
assigns existing rows to Main and clones only dependencies referenced by existing
other-branch products, product groups, purchases, orders, and discount pivots,
relinking those documents to their independent records. Location ancestors are
cloned with their city; fee configuration is not copied. Supplier codes, brand
names, and location names use branch composite uniqueness. Catalog request
validation prevents foreign supplier/brand/parent/location IDs, and discount
assignment/import stays within the discount's branch. Reference-data forms use
`useConfigurationBranch` through `useAddModel` or their custom submit handlers
so company administrators choose ownership before creating a record.

`Branch::afterStore()` delegates operational defaults to `BranchSettingDefaults`
inside the branch-create transaction, then grants creator access and registers the
branch hostname. Tenant provisioning uses the same service for Main; legacy
`PaymentModeSeeder` and `CashSettingSeeder` delegate to it for each branch. The
service seeds only missing setting groups, so reruns preserve customized singleton
values and payment/status lists. Defaults are code templates rather than shared
database rows. `CashSessionController::currenciesNotes()` uses branch-scoped Setting
queries, and `SaveCurrencyNotesRequest` validates positive distinct values before
`CurrencyNoteController` stores a JSON array. `Order::getCode()` and stock-adjustment
numbering bypass only the branch scope to preserve tenant-wide document sequences.
Product-group hooks tolerate absent optional relationship arrays, preserve omitted
group relations on updates, and sync supplied child attribute IDs. Stock-audit file
validation allows the plain-text MIME detected for CSV while checking the `.csv` or
`.xlsx` extension. `NewBranchOnboardingTest` covers provisioning, safe reseeding,
note isolation, first purchase/sale/return/adjustment, group updates, and CSV uploads.
