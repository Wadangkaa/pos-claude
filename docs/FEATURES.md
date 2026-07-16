# ANT POS — Feature Inventory

> Update this file whenever a feature is added, changed, or removed.
> Last updated: 2026-07-10

The system has two sales channels sharing one backend and one product catalog:
1. **POS** — staff-facing admin/cashier app (tenant subdomain, staff login)
2. **Website** — customer-facing e-commerce storefront selling the same products
   (same SPA, routes under `src/pages/website/`, customer login, feature-flagged
   per tenant via `website_enabled`)

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
  products. Has status (`ProductVariantStatusEnum`), images, tags, attributes.
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
  city, FK to `locations`); managed at `/delivery-fees`. Excel import
  (headers: `city`, `fee`). Website checkout requires choosing a deliverable
  city (`GET api/website/delivery-locations`, public) and adds the fee to the
  order (`orders.location_id`, `orders.delivery_fee`; fee included in
  `total_amount` and payment).
- **Product import/export** — Excel import (products, price updates, orders,
  purchases; failed-row download via signed URL), Excel export per module.

## 2. Sales (POS)

- **Order creation** — cart-style billing with barcode input, quantity/weight items,
  item-level % discount, order-level % discount, final flat discount, VAT 13%
  (Nepal, inclusive: total × 13/113).
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
  fixed-price product lists. Product exposes computed `discount_details`.
- **Customers** — CRM basics, linked to orders.
- **Footfall / customer returns** — records gender + reason of walk-outs (analytics,
  not product returns). Route `/footfall`.

## 3. Website (e-commerce)

- **Storefront** — home, product-group listing, product detail, similar products,
  tag-filtered listing, category navbar, attribute filter sidebar.
  Public endpoints under `/api/website/*`.
- **Customer accounts** — register, login (Sanctum `auth:customer` guard on the
  Customer model), profile, logout.
- **Cart** — add/remove/update-quantity/clear, then **checkout** → creates a website
  order (`Cart`, `CartItem`, `CartStatusEnum`, `Services/Websites/CartService`).
- **Customer order history** — `/api/orders` for the logged-in customer.
- **Website order management (POS side)** — pending-website-orders list for staff,
  website order details, confirm flow (website orders arrive Pending, staff confirm).
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
  `ExcelStockAuditReader`), compare counted vs. system stock.
- **Inventory configuration** page.

## 5. Cash management

- **Cash sessions** — open/close a till session with denomination counts
  (`CashSession`, `CashSessionStatus`, `CashDenominationStage`), current-session
  lookup, session detail/update.
- **Cash denominations & currency notes** — NPR note/coin configuration
  (`CurrencyNote`, denomination settings pages).
- **Expenses** — expense tracking module.

## 6. Procurement & partners

- **Suppliers** — vendor records (PAN number etc.), linked to products/variants/purchases.
- **Stores** — physical outlets with VAT/PAN info and tax %; orders/purchases belong
  to a store.

## 7. Reporting

- Sales dashboard, product dashboard, daily sales, weekly sales, most-sold-tag
  report (routes in `routes/admin/report.php`, prefix `/api/report`).
- Report export (`ReportExportController`, `useReportExport` hook).

## 8. Administration & platform

- **Multi-tenancy** — central API to create tenants (`POST /tenants`), each tenant
  gets its own database + subdomain; central auth for platform admin.
- **Users, roles & permissions** — Spatie permissions; role management UI;
  default-permissions endpoint.
- **Authentication** — staff login/logout/change-password (Sanctum tokens).
- **Activity logs** — Spatie activity log on key models, system-logs UI.
- **Settings** — POS config, website details, email config, payment methods,
  generic settings CRUD.
- **Imports/exports tracking** — `imports`/`exports` tables with statistics and
  status (`ImportStatusEnum`).
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
