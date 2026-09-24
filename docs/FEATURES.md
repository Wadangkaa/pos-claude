# ANT POS — Feature Inventory

> Update this file whenever a feature is added, changed, or removed.
> Last updated: 2026-09-24

The system has two sales channels sharing one backend and one product catalog:
1. **POS** — staff-facing admin/cashier app at `/pos/dashboard` (tenant subdomain,
   staff login at `/pos/login`)
2. **Website** — customer-facing e-commerce storefront selling the same products
   at `/` (same SPA, routes under `src/pages/website/`, customer login at
   `/login`, feature-flagged per tenant via `website_enabled`). Old
   `/website/*` links redirect to the corresponding customer routes.

Orders from both channels land in the same `orders` table, distinguished by
`orders.type` (`pos` | `website`).

---

## 1. Catalog

- **Products** — the sellable unit (SKU, barcode lookup, cost/sell price, pre-VAT
  auto-calc, stock, weight-based flag, description, images with thumbnail +
  ordering, supplier, brand). Auto code `PROD####`. Activity-logged.
  ⚠️ See CLAUDE.md: Product is the child/sellable; ProductVariant is the parent group.
- **Product Variants ("product groups")** — parent grouping of sibling products by
  article number (SKU prefix). Creating/editing a variant bulk-creates its child
  products; updating also syncs name/sell_price of existing child products
  (matched by SKU, only within the same variant). Has status
  (`ProductVariantStatusEnum`), images, tags, attributes.
- **Attributes** — key/value characteristics (Size, Color…) attachable to both
  products and variants; attribute-name management page.
- **Tags** — labels on products/variants/categories; used for website filtering and
  "most tag" reporting.
- **Categories** — with tag assignment (`categories_tags`); drives website navbar.
- **Brands** — with status; linked from products.
- **Locations** — 3-level hierarchy (country → district → city) in a single
  self-referencing `locations` table; parent type enforced by validation.
  Excel import (headers: `name`, `type`, `parent`); one file can hold all
  levels since the parent is resolved row-by-row at store time.
- **Delivery Fees** — per-city delivery charge (`delivery_fees`, one fee per
  city, FK to `locations`); managed at `/pos/delivery-fees`. Excel import
  (headers: `city`, `fee`). Website checkout requires choosing a deliverable
  city (`GET api/website/delivery-locations`, public,
  `DeliveryLocationController`) and adds the fee to the order
  (`orders.location_id`, `orders.delivery_fee`; fee included in `total_amount`
  and payment). Each row carries its full ancestry (`country`, `country_id`,
  `district`, `district_id`, `city`, `location_id`, `fee`) so the storefront
  checkout (`pos-frontend/src/pages/website/pages/Cart.tsx`) can offer a
  country → district → city cascade. Customers can also provide an optional
  delivery landmark/detailed location; it is saved on `orders.delivery_landmark`
  and shown with the complete delivery address in staff website-order details.
- **Product import/export** — Excel import (products, price updates, orders,
  purchases; failed-row download via signed URL). Excel export per module runs
  as a **queued background job** (`ExportJob`): `POST /api/export/{uri}` creates
  an `exports` row (pending/processing/completed/failed) and returns
  immediately; the file is written to tenant storage and downloaded from the
  **Exports page** (`/pos/exports`, `GET /api/exports`), which polls while a job is
  active. Export jobs run on a dedicated worker so large workbooks do not wait
  behind notifications or other background work. Optional from/to date range;
  no dates = full table.
  POS Sales exports contain two worksheets: **Orders** (one row per sale, including
  the overall order discount, return count, cumulative return total, and net
  sales after returns) and **Order Items** (one row per sold product).
  On the company admin host, product details additionally show every branch's
  physical, reserved, and available balance alongside company totals.

## 2. Sales (POS)

- **Order creation** — cart-style billing with barcode input, quantity/weight items,
  item-level % discount, order-level % discount, final flat discount, VAT 13%, and
  a selectable sale date and time
  (Nepal, inclusive: total × 13/113).
- **Sales list filters** — filter POS sales by payment status, sales status, or
  payment method (including split/multiple-method payments).
- **Payments** — single method, or **split payments** across methods; cash received /
  change; payment methods configurable in settings; payment status & sales status lists.
- **Draft orders** — save draft, update draft, confirm later (`/order-draft`,
  `/order/{id}/confirm`).
- **Invoices** — printable invoice view (react-to-print), Nepali date on orders.
- **Sales returns** — return items from an order (`sales_returns`,
  `sale_return_items`, `SaleReturnStatusEnum`), restocks via stock transactions.
- **Discount engine** — `Discount` with types PERCENTAGE / FIXED / FIXED_PRICE,
  time-windowed (`starts_at`/`ends_at`), active toggle, per-product override value,
  polymorphic `discountables`, bulk attach/detach and Excel import of discounted /
  fixed-price product lists (percentage/fixed import needs SKU column only;
  fixed-price import needs SKU + final price). Product exposes computed
  `discount_details`. Website product details show the discounted final price,
  original price, and discount label; product-group cards calculate and show their
  min/max range from child products' final prices and identify groups with a sale.
- **Customers** — CRM basics, linked to orders.
- **Footfall / customer returns** — records gender + reason of walk-outs (analytics,
  not product returns). Route `/pos/footfall`.

## 3. Website (e-commerce)

- **Storefront** — home, product-group listing, product detail, similar products,
  tag-filtered listing, category navbar, attribute filter sidebar. Navbar category
  labels open tag dropdowns without navigating; selecting a tag shows only
  matching product groups in the same sidebar-filterable, paginated catalog as
  the home page. A tag selected from the navbar checks its parent category in
  the sidebar. Unchecking that category or clearing filters removes the tag
  selection; a category can include multiple tags, while a single tag stays
  narrower. Child-product lookups for category and tag filters use an indexed
  `products.product_variant_id` relation.
  Public endpoints under `/api/website/*`.
- **Customer accounts** — register, login (Sanctum `auth:customer` guard on the
  Customer model), profile, logout.
- **Cart** — add/remove/update-quantity/clear, then **checkout** → creates a website
  order (`Cart`, `CartItem`, `CartStatusEnum`, `Services/Websites/CartService`).
  Checkout locks branch stock, automatically assigns the first branch that can
  fulfill every cart item, and records a pending reservation there. Confirmation
  consumes that reservation and deducts the same branch. On the company admin
  host, an administrator can move a pending order to another sufficiently stocked
  branch; the override is retained in the order fulfillment audit data.
- **Customer order history** — `/api/orders` for the logged-in customer.
- Cart items and order items show the product's own thumbnail/first image, falling
  back to its product group's images (`Product::displayThumbnail()`).
- **Website order management (POS side)** — staff list contains every website-channel
  order (Pending/Accepted/Cancelled status filter), website order details, and
  fulfilment flow. Staff can cancel a pending website order, which releases its
  reservation without changing physical stock or payments. Accepting updates the
  existing website order; it never creates or converts it into a POS sale, deducts
  the reserved stock, and starts a separate delivery status at Pending. Staff can
  later mark delivery Completed; each actual delivery-status change emails the
  customer when they have a valid email address, without duplicating emails for
  repeated selections. Delivery status is tracking-only and has no stock or
  payment effect. Storefront availability and quantity controls show physical
  stock less pending reservations. Checkout emails the customer and sends the admin
  alert to the POS-configured notification email. The POS Sales list contains
  POS-channel orders only. Website Orders has its own queued export, with Orders
  and Order Items worksheets limited to the website channel.
- **Website settings / feature flag** — website can be enabled/disabled per tenant
  (`FeatureKey::WEBSITE`); disabled state page; website details (name/branding)
  saved in settings and exposed publicly; `WebsiteConfig` admin page.

## 4. Inventory

- **Stock transactions** — immutable audit trail of every movement
  (`inventory_stock_transactions`: In/Out, previous/new stock, polymorphic source:
  Order, Purchase, StockAdjustment, SalesReturn…).
- **Purchases** — supplier purchase entry with line items (polymorphic
  `productables`), auto stock-in, Excel import.
- **Stock adjustments** — manual corrections with reasons (`StockUpdateReasonEnum`),
  auto stock transactions.
- **Damage products** — damaged-stock listing + adjust flow.
- **Stock audit** — Excel-upload physical count audit (`stock_audits`,
  `stock_audit_items`, `stock_audit_results`, `stock_audit_summaries`,
  `ExcelStockAuditReader`), compare counted vs. system stock. A completed audit
  can be exported through the standard Exports page as one workbook with
  **All**, **Matched**, **Mismatch**, **Missing in System**, and **Missing in
  Physical** worksheets.
- **Inventory configuration** page.
- **Branches and transfers** — one company tenant can operate multiple branches.
  A staff hostname fixes the active branch and its inventory balance
  (`branch_product_stocks`); stock transfers are sent then received between
  branches without changing company-wide stock. The company admin hostname can
  view consolidated or selected-branch reporting.
  New tenant provisioning creates `Main`, an admin hostname, and a super-admin
  tenant user; branches created from the company admin portal automatically get
  their own hostname mapping and creator assignment. Company admins can manage
  the shared Products/Product Variants catalog and create staff users with one
  or more branch assignments from the admin host.

## 5. Cash management

- **Cash sessions** — open/close a till session with denomination counts
  (`CashSession`, `CashSessionStatus`, `CashDenominationStage`), current-session
  lookup, session detail/update.
- **Cash denominations & currency notes** — NPR note/coin configuration
  (`CurrencyNote`, denomination settings pages).
- **Expenses** — expense tracking with Daily (default) and Overall types and an
  optional bill image, privately stored per tenant and viewable by signed-in
  staff. The dashboard shows today's total for Daily expenses only; expenses
  remain record-only and do not change cash, sales, or stock totals.

## 6. Procurement & partners

- **Suppliers** — vendor records (PAN number etc.), linked to products/variants/purchases.
- **Branches** — physical outlets with VAT/PAN info and tax %; orders, purchases,
  stock adjustments, sales returns, expenses, and inventory transactions belong to
  a branch. Staff are assigned to branches.

## 7. Reporting

- Sales dashboard, product dashboard, daily sales, weekly sales, most-sold-tag
  report (routes in `routes/admin/report.php`, prefix `/api/report`).
- Daily, weekly, monthly, and yearly sales reports have an All/POS/Website order
  type filter. It applies to sales, order counts, returns, payment summaries,
  charts, and exports; All preserves the combined view. The selected type stays
  active while switching sales-report tabs.
- The daily sales report shows each date's expense total with Daily and Overall
  breakdowns in both the screen table and exported workbook. Expenses are
  shown separately and do not reduce Net Sales; because expenses have no brand
  or order-type assignment, their figures remain across all brands and channels
  when sales are filtered.
- On an `admin.` company hostname, Sales and Product reports include an
  all-branches/selected-branch filter. The same branch filter is forwarded to
  queued report exports. A branch staff hostname ignores a supplied `branch_id`
  and always returns only its own data.
- Report export (`ReportExportController`, `useReportExport` hook) — queued like
  every other export (`ReportExportJob`), downloaded from the Exports page.
  Fixed bug (2026-08): export was dropping `limit`/`sort`/`sort_by`/
  `sort_direction`/`threshold`/`category_id`, so exports silently used each
  report's default row count instead of what was selected on screen; also
  `limit=all` used to produce an empty file (`LIMIT 0`). See
  `docs/ARCHITECTURE.md` Jobs section for detail.

## 8. Administration & platform

- **Multi-tenancy** — central API to create tenants (`POST /tenants`) and
  irreversibly delete them (`DELETE /tenants/{tenantId}` with the tenant domain as
  confirmation). Each tenant gets its own database + subdomain; deletion removes
  central tenant records and drops its tenant database through the tenancy pipeline.
  Central auth is required for both operations.
- **Company cutover audit** — `branches:cutover-audit` performs a read-only
  reconciliation of source single-branch tenants against mapped company branches
  before any Caliber data consolidation is approved.
- **Company cutover tooling** — dry-run/import/reconcile/activate commands move
  approved branch stock, customers, POS sales, payments, and sales returns into
  a company branch exactly once; the VPS procedure is documented in
  `docs/CALIBER_CUTOVER_RUNBOOK.md`.
- **Users, roles & permissions** — tenant permissions are defined for every
  staff-facing resource with view/create/update/delete actions. Staff login and
  `GET /api/profile` return effective roles and permissions; on branch hosts the
  left sidebar shows only entries covered by the relevant `*-view` permission.
  Company-admin navigation remains separate and is not filtered by this branch UI.
  Roles are managed from the branch sidebar; users can hold one or more roles.
  Built-in cashier and manager roles provide safe starting presets. Cashiers have
  POS product lookup permission without catalog-management access. Built-in
  `super-admin`, `admin`, `manager`, and `cashier` roles are read-only; create a
  custom role to tailor permissions.
- **Authentication** — staff login/profile/logout/change-password (Sanctum tokens).
- **Activity logs** — Spatie activity log on key models, system-logs UI.
- **Settings** — POS config (invoice message and notification email), website
  details, email config, payment methods, generic settings CRUD. Configuration
  edit dialogs preload the selected setting's stored values.
- **Imports/exports tracking** — `imports`/`exports` tables with statistics and
  status (`ImportStatusEnum`, `ExportStatusEnum`); exports listed per-user on
  the Exports page.
- **Feature flags** — `features` table (central), `website_enabled`, `maintenance_mode`.

---

## Deployment targets (frontend `npm run deploy:*`)

Tenant storefront/POS builds are scp'd to subdomains of `aitechnology.com.np`:
maharan (server), dev, kstore, kshoes, paharishoes, maharanventures, suryabinayak,
kritipur, f2c.

## Known gaps / notes

- `customer_returns` table is footfall analytics, not product returns (real returns
  are `sales_returns`).
- Some tenant databases may differ slightly in table count (older vs newer tenants).
- Frontend `routes.tsx` contains many duplicated route declarations (harmless but
  worth cleaning up someday).
