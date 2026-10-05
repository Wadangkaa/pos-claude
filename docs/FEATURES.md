# ANT POS — Feature Inventory

> Update this file whenever a feature is added, changed, or removed.
> Last updated: 2026-09-29

The system has two sales channels sharing one backend. Each branch owns its catalog:
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
  Products, parent groups, tags, categories, and attributes are owned by one
  branch. Branch hostnames show only their own catalog; SKUs are unique within
  a branch, so two branches may use the same SKU. Customers remain shared.
- **Product Variants ("product groups")** — parent grouping of sibling products by
  article number (SKU prefix). Creating/editing a variant bulk-creates its child
  products; updating also syncs name/sell_price of existing child products
  (matched by SKU, only within the same variant). Has status
  (`ProductVariantStatusEnum`), images, tags, attributes.
- **Attributes** — key/value characteristics (Size, Color…) attachable to both
  products and variants; attribute-name management page. Attribute names in
  `settings` are branch owned, along with all website, email, POS, payment,
  status, currency, and denomination settings.
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
  Checkout also requires a delivery phone number and accepts an optional
  alternative phone number. Both are saved on the order and shown in the
  Delivery Contact section of staff website-order details.
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
  Product details show the owning branch's physical, reserved, and available balance.

## 2. Sales (POS)

- **Order creation** — cart-style billing with barcode input, quantity/weight items,
  item-level % discount, order-level % discount, final flat discount, VAT 13%, and
  a selectable sale date and time
  (Nepal, inclusive: total × 13/113).
- **Barcode failure sound** — Sales plays the bundled error audio whenever a
  barcode lookup fails or returns no product, alongside the existing error message.
  Each failed scan restarts the sound; successful scans remain silent.
- **Sales list filters** — filter POS sales by payment status, sales status, or
  payment method (including split/multiple-method payments).
- **Payments** — single method, or **split payments** across methods; cash received /
  change; payment methods configurable in settings; payment status & sales status lists.
- **Draft orders** — save draft, update draft, confirm later (`/order-draft`,
  `/order/{id}/confirm`).
- **Invoices** — printable invoice view (react-to-print), Nepali date on orders.
  The receipt header uses the sale's branch name, branding, phone, location,
  and PAN details.
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
  Customers remain shared across the company's branches and website accounts.
- **Footfall / customer returns** — records gender + reason of walk-outs (analytics,
  not product returns). Route `/pos/footfall`; records belong to the active branch.

## 3. Website (e-commerce)

- **Storefront** — home, product-group listing, product detail, similar products,
  tag-filtered listing, category navbar, attribute filter sidebar. Navbar category
  labels open tag dropdowns without navigating; selecting a tag shows only
  matching product groups in the same sidebar-filterable, paginated catalog as
  the home page. A tag selected from the navbar checks its parent category in
  the sidebar. Unchecking that category or clearing filters removes the tag
  selection; a category can include multiple tags, while a single tag stays
  narrower. Child-product lookups for category and tag filters use an indexed
  `products.product_variant_id` relation. Below the `lg` breakpoint the filter
  sidebar is hidden and opens as a slide-over drawer from a "Filters" button
  (with an active-filter count) above the product grid.
  Public endpoints under `/api/website/*`.
- **Customer accounts** — register, login (Sanctum `auth:customer` guard on the
  Customer model), profile, logout.
- **Signup email verification** — new website accounts receive a store email
  with a verification link valid for 60 minutes and must verify before signing
  in. The verification page supports resending links, with a 60-second cooldown
  and rate limits. Existing accounts retain access without being marked verified.
- **Cart** — add/remove/update-quantity/clear, then **checkout** → creates a website
  order (`Cart`, `CartItem`, `CartStatusEnum`, `Services/Websites/CartService`).
  Each branch hostname has its own cart and storefront catalog. Checkout locks
  stock in that branch and records a pending reservation there. Confirmation
  consumes that reservation and deducts the same branch; cancellation releases
  the reservation without deducting stock. On the company admin host, an
  administrator can move a legacy shared-catalog order to another sufficiently
  stocked branch; branch-owned products stay with their owning branch.
- **Customer order history** — `/api/orders` for the logged-in customer.
- **Customer account menu** — logged-in navbar uses a round initials button
  (user icon fallback). Its menu shows the customer name, Orders, Change Password,
  and Logout. Password changes require the current password, a different new
  password of at least eight characters, and matching confirmation.
- **Checkout payment** — Cash on Delivery (COD) is the sole payment option and
  is always selected, with a reminder to pay cash on delivery.
- **Customer password recovery** — login links to Forgot Password; the customer
  receives a reset email for that store. Links expire after 60 minutes and can
  be used once; a successful reset revokes previous customer sessions. Unknown
  or non-login accounts receive the same confirmation, and requests are rate limited.
  Password fields in login, registration, change password, and reset password
  include show/hide eye buttons.
- **Product image gallery** — clicking the main product photo opens a full-screen
  gallery on desktop and mobile, with next/previous buttons, keyboard arrows,
  an image counter, and a close button (Escape also closes it).
- **Website order Notes** — staff can save or clear Notes from website-order
  details. Notes are internal, persist on the order, and are omitted from customer
  order responses. Saving notes preserves the order status, totals, and other data.
- Cart items and order items show the product's own thumbnail/first image, falling
  back to its product group's images (`Product::displayThumbnail()`).
- **Website order management (POS side)** — staff list contains every website-channel
  order (Pending/Accepted/Cancelled status filter), website order details, and
  fulfilment flow. Staff can cancel a pending website order, which releases its
  reservation without changing physical stock or payments. Accepting updates the
  existing website order and deducts the reserved stock; it never creates or
  converts it into a POS sale. Website checkout defaults delivery status to Pending.
  Delivery options come from Configuration → Sales Status, including newly
  added entries. After accepting an order, staff choose any option in the
  Delivery Status dropdown and click its Save button; no status sequence
  is assumed. If Pending is not configured, it remains available as the default.
  Payment Status uses Configuration → Payment Status, defaults to Pending, and
  has its own dropdown and Save button after acceptance. A compact status toolbar
  puts both dropdowns in one row on desktop and stacks them on mobile. Save buttons
  and the previously saved value appear only for changed selections; small
  configuration icons link to each status list. Payment status appears in the
  staff website-order list and customer order history. Changing this tracking
  status preserves notes, totals, stock, and payment records.
  The customer order history also shows delivery status. Each actual delivery change
  emails the customer when they have a valid email address, without duplicates for
  repeated selections. Delivery status is tracking-only and has no stock or
  payment effect. Storefront availability and quantity controls show physical
  stock less pending reservations. Checkout emails the customer and sends the admin
  alert to the POS-configured notification email. Both emails list each product's
  SKU and quantity, and show the saved order total in NPR, including delivery
  charges and discounts. The POS Sales list contains
  POS-channel orders only. Website Orders has its own queued export, with Orders
  and Order Items worksheets limited to the website channel. The Website Orders
  navigation tab is shown on branch portals, not the company admin portal.
- **Website settings / feature flag** — website can be enabled/disabled per tenant
  (`FeatureKey::WEBSITE`); disabled state page; website details (name/branding)
  saved in settings and exposed publicly; `WebsiteConfig` admin page.
- **Storefront footer** — Website Configuration has a separate footer form for
  a description, optional Facebook/Instagram/TikTok/X links, and an optional map
  iframe or HTTPS embed URL. The responsive storefront footer uses the configured
  store name/logo, description, social links, map, shopping links, and copyright.
  Blank social links/maps stay hidden; footer and branding save independently.
  Footer settings belong to the active branch and do not alter the POS footer.
  Unconfigured branches start with an empty footer and email configuration; Main
  keeps previously saved company settings. Company administrators select the
  branch to edit in Website/POS Configuration.

## 4. Inventory

- **Stock transactions** — immutable audit trail of every movement
  (`inventory_stock_transactions`: In/Out, previous/new stock, polymorphic source:
  Order, Purchase, StockAdjustment, SalesReturn…).
- **Purchases** — supplier purchase entry with line items (polymorphic
  `productables`), auto stock-in, Excel import.
- **Stock adjustments** — manual corrections with reasons (`StockUpdateReasonEnum`),
  auto stock transactions.
- **Damage products** — damaged-stock listing + adjust flow.
  Stock transaction and damage lists, details, and recovery use the active branch.
- **Stock audit** — Excel-upload physical count audit (`stock_audits`,
  `stock_audit_items`, `stock_audit_results`, `stock_audit_summaries`,
  `ExcelStockAuditReader`), compare counted vs. system stock. A completed audit
  can be exported through the standard Exports page as one workbook with
  **All**, **Matched**, **Mismatch**, **Missing in System**, and **Missing in
  Physical** worksheets.
  Audit uploads belong to the active branch; background comparisons use that
  branch's catalog and stock balances, including when branches share a SKU.
- **Inventory configuration** page.
- **Branches and transfers** — one company tenant can operate multiple branches.
  A staff hostname fixes the active branch and its inventory balance
  (`branch_product_stocks`); transfers between branch-owned products match the
  destination branch's separate product by SKU. Dashboard stock totals and stock
  transaction totals display whole units. The company admin hostname can
  view consolidated or selected-branch reporting.
  New tenant provisioning creates `Main`, an admin hostname, and a super-admin
  tenant user; branches created from the company admin portal automatically get
  their own hostname mapping and creator assignment. Company admins can manage
  staff users with one or more branch assignments from the admin host. Branch
  staff manage their own catalog; administrators see all customers and reports
  across all branches. Existing catalog rows migrate to the oldest branch;
  other branches start with empty catalogs and their live stock is consolidated
  into that branch. Outstanding reservations outside Main and sent transfers
  must be cleared before migration. The original branch balances are saved in
  `branch_catalog_stock_backups` for reconciliation. Historical orders in other
  branches still show their original product details, while product catalog
  endpoints stay branch scoped. Payments and order items can be viewed or changed
  only through orders in the active branch. Branch managers cannot delete a user
  assigned to multiple branches, and attribute names must belong to the active
  catalog branch.

## 5. Cash management

- **Cash sessions** — open/close a till session with denomination counts
  (`CashSession`, `CashSessionStatus`, `CashDenominationStage`), current-session
  lookup, session detail/update. Each branch has its own current session and
  history; denomination rows inherit their parent session's branch.
- **Cash denominations & currency notes** — NPR note/coin configuration
  (`CurrencyNote`, denomination settings pages).
- **Expenses** — expense tracking with Daily (default) and Overall types and an
  optional bill image, privately stored per tenant and viewable by signed-in
  staff. The dashboard shows today's total for Daily expenses only; expenses
  remain record-only and do not change cash, sales, or stock totals.
  New expenses inherit the active branch, keeping them visible in that branch's
  list for cashiers, managers, and admins. Company-portal expenses use the default
  branch.
  Editing and saving an unchanged expense accepts the API's numeric amount;
  amount validation also accepts typed numeric strings and zero, while rejecting
  blank, negative, and nonnumeric values.

## 6. Procurement & partners

- **Suppliers** — vendor records (PAN number etc.), linked to products/variants/purchases.
- **Branches** — physical outlets with VAT/PAN info and tax %; orders, purchases,
  stock adjustments, sales returns, expenses, and inventory transactions belong to
  a branch. Staff are assigned to branches.

## 7. Reporting

- **HR staff and attendance** — `/pos/staff` registers branch-owned staff members
  without creating POS login accounts. Staff records can be edited, made
  inactive, or archived while retaining their attendance history. At
  `/pos/attendance`, authorized users mark each active staff member present or
  absent for a selected date; unmarked staff are counted separately. HR is
  available only from individual branch hostnames, each showing its own roster
  and counts. The company admin portal has no HR access.
- **Branch staff incentives** — `/pos/incentives` lets a branch administrator or
  manager set effective-dated daily sales ranges and point awards. A day's
  branch net sales uses paid amounts minus sales returns, matching the Daily
  Sales report. A matching range's points are split among staff marked present
  that day, to four decimal places, with any rounding remainder distributed by
  staff ID. Offer plans show each sales range and its shared point award in
  separate table columns, with explicit open-ended limits and plan status.
  Daily awards and monthly totals use grouped numbers, suppress trailing zeros,
  and preserve fractional awards to four decimal places. The daily summary
  identifies the qualifying range. An unmatched
  range earns no points; points with no present staff remain unallocated.
  Historical offer plans are locked once their effective date has passed.
  Point balances are recalculated from sales and attendance; payroll conversion
  and money values are not set yet. Incentives are unavailable on the company
  admin portal.

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
  and always returns only its own data. Low-quantity reports include branch
  products with no stock-balance row as zero quantity.
- Report export (`ReportExportController`, `useReportExport` hook) — queued like
  every other export (`ReportExportJob`), downloaded from the Exports page.
  Export history and authenticated downloads are limited to the requesting
  user and active branch, and workers preserve the originating branch.
  File names carry a random part so exports started in the same second do not
  overwrite each other. The daily sales report's expense figures follow the same
  branch filter as its sales.
  Fixed bug (2026-08): export was dropping `limit`/`sort`/`sort_by`/
  `sort_direction`/`threshold`/`category_id`, so exports silently used each
  report's default row count instead of what was selected on screen; also
  `limit=all` used to produce an empty file (`LIMIT 0`). See
  `docs/ARCHITECTURE.md` Jobs section for detail.

## 8. Administration & platform

- **Daily dashboard email** — POS Configuration has Daily Report Time (Nepal
  time) and Daily Report Email. Setting both enables a daily Excel attachment;
  clearing both disables it. The report contains Title/Value (NPR) rows for each
  payment-mode total, Sales Return, and Daily Expenses across all brands, using
  the same calculations as the dashboard. Dashboard branch filters narrow the
  on-screen totals; each scheduled email contains only its configured branch's totals. Total Stock, Products, Suppliers, and
  Customers are excluded. Totals are captured at the scheduled minute, retained
  for queued delivery, and protected against repeated sends for the same date.
  Mail failures retry up to three times; reports use that branch's email settings.
  Multiple branches can each receive one report on the same date.
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
  The sidebar scrolls vertically without horizontal movement in expanded,
  collapsed, and mobile layouts.
  Roles are managed from the branch sidebar; users can hold one or more roles.
  Built-in cashier and manager roles provide safe starting presets. Cashiers have
  POS product lookup permission without catalog-management access, plus access
  to sales returns, footfall, cash denominations/sessions, expenses, purchases,
  and damage-product recovery. Built-in
  `super-admin`, `admin`, `manager`, and `cashier` roles are read-only; create a
  custom role to tailor permissions. Staff API actions enforce these permissions
  on the server, including reports, imports, exports, catalog edits, and user
  management. A cashier can search products for POS sales without catalog edit
  rights. Company summary and branch creation require the admin hostname.
  Signing in on the admin hostname is refused for anyone without the `admin` or
  `super-admin` role. Staff who can create or edit users (such as managers) can
  list the roles they are allowed to assign, without role-management access.
  Branch-host user management is limited to that branch; roles apply across all
  branches assigned to the user. Queued exports retain the requesting branch.
  Branch portals create users in the current branch automatically; only the
  company admin portal shows the branch assignment selector. Cashiers can load
  dashboard summaries and sales payment/status choices without configuration
  or detailed report access. Sales and customer tables hide actions the user
  cannot perform.
  Permission caches are isolated per tenant and reset when switching context,
  preventing another tenant's role IDs from incorrectly allowing or denying
  settings saves, website order updates, and other staff actions.
  Branch operational records and catalog entries are isolated even for an admin
  assigned to multiple branches. Sales (including drafts), purchases, stock
  adjustments, and discount product assignments reject another branch's product
  IDs. Customers remain shared company data. Suppliers, brands, discount definitions,
  locations, and delivery fees now belong to the active branch. Roles define
  company-wide staff permissions. All configuration is independently saved
  and read per branch. Branch creation remains exclusive to the company admin portal.
- **Authentication** — staff login/profile/logout/change-password (Sanctum tokens).
- **Activity logs** — Spatie activity log on key models (orders, products, stock
  adjustments), system-logs UI. A branch portal lists only activity on its own
  records; the company admin portal lists every branch.
- **Settings** — POS config (invoice message and notification email), website
  details, email config, payment methods, generic settings CRUD. Configuration
  edit dialogs preload the selected setting's stored values. Every setting belongs
  to a branch, including footer/branding, SMTP credentials, invoice text, order
  notification/report recipients, payment modes, and sales/payment statuses.
  Branch hosts reject foreign setting IDs; cash payment handling uses the branch's
  configured Cash method rather than a hardcoded ID. Queued emails restore their
  originating branch configuration.
- **Imports/exports tracking** — `imports`/`exports` tables with statistics and
  status (`ImportStatusEnum`, `ExportStatusEnum`); exports listed per-user on
  the Exports page.
- **Branch business reference isolation** — suppliers, brands, discount definitions,
  delivery locations and fees are independent per branch. Company administrators
  select a branch on create; branch users always save to their hostname branch.
  Related product, purchase, discount, and checkout selections reject foreign
  branch IDs. Existing shared records move to Main; references used by older
  documents in other branches receive independent copies so their data and
  applied discounts are preserved. Customers continue to be shared.
- **New branch defaults** — creating a branch automatically provisions its domain,
  creator access, Cash payment method, payment/sales statuses, currencies, NPR
  notes, and inventory weight setting. Defaults use a common template and receive
  separate branch-owned rows. Reseeding fills missing setting groups and preserves
  customized values. Cash denominations read and save the current branch's notes,
  including fractional denominations. Order and adjustment numbers use the
  tenant-wide sequence, preventing first-sale collisions with Main. Product-group
  saves accept omitted optional relations and update existing child attributes;
  stock audits accept ordinary plain-text CSV files with a `.csv` extension.
- **Deleting a branch** — company admins can delete a branch from the admin
  portal's Branches table. Only an empty branch can be deleted: its default
  settings, user assignments and hostname mapping are removed with it. A branch
  that has products, sales or any other records, or the company's only branch,
  is refused with an explanation. Branch portals show the Branches list read-only.
- **Feature flags** — `features` table (central), `website_enabled`, `maintenance_mode`.
  While `website_enabled` is off the public storefront API (`/api/website/*`,
  cart, customer auth and orders) answers 403; only `get-features` stays open so
  the storefront can show its disabled page.

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
