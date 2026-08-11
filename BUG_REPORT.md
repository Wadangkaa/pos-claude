# ANT POS — Bug Audit Report

**Date:** 2026-08-04
**Scope:** `pos-backend/` (Laravel 11) and `pos-frontend/` (React 18 + Vite)
**Method:** Manual code audit (read-only, no fixes applied). Findings verified by reading actual source, not speculative.

This report lists confirmed bugs, logic errors, and correctness issues, grouped by severity, backend first then frontend.

---

## Backend (`pos-backend`)

### CRITICAL

**B1. `SuperController::store()` overwrites every model's `status` field, corrupting string-backed enums.**
`app/Http/Controllers/Api/SuperController.php:77-79`
```php
if (array_key_exists('status', $model->getAttributes())) {
    $request->merge(['status' => OrderStatusEnum::Success->value]);
}
```
This runs for **any** model with a `status` attribute, not just `Order`. `Brand` and `Category` (both routed generically through `SuperController` via `Route::apiRoutes(...)`) cast `status` to string-backed enums (`BrandStatusEnum`, `CategoryStatusEnum`: `'active'|'inactive'`). The row is persisted with `status = 1` (an int), then resource serialization (`CategoryResource`/`BrandDetailResource`) triggers the enum cast on read, throwing `ValueError: 1 is not a valid backing value...`. This happens **after** `DB::commit()`, so it's not caught by `catch (\Exception $e)` — request 500s with a corrupt row already saved. For `Location`/`DeliveryFee` (plain string column) it silently discards whatever status the client submitted and forces `1`.
- **Trigger:** `POST /api/category` or `POST /api/brand` with any `status` value → corrupt row + 500.

**B2. Website checkout writes a `customers.id` into `created_by`/`updated_by` columns that are FK-constrained to `users`.**
`app/Services/Orders/CreateOrder.php:26-27, 50-51`
```php
'created_by' => $data->customerId,
'updated_by' => $data->customerId,
```
`orders.created_by`/`updated_by` and `order_items.created_by`/`updated_by` are `foreignId(...)->constrained('users')` (see the respective tenant migrations), but `$data->customerId` is a `customers.id` — an independent ID sequence.
- **Trigger:** any logged-in customer calls `POST /api/cart/checkout` → FK violation (SQLSTATE 23000) inside the transaction → checkout fails outright, or (if the customer ID happens to collide with a real user ID) the order gets silently mis-attributed to the wrong staff user.

**B3. Hardcoded `created_by = 1` on cascade-deletable rows is a mass-deletion trap.**
- `app/Models/Product.php:83-84` — `ProductVariant::create([..., 'created_by' => 1, ...])`, run whenever a new SKU article number auto-creates a product group.
- `app/Http/Controllers/Website/CustomerAuthController.php:69-70` — same for every self-registered website customer.
`product_variants.created_by` and `customers.created_by` are `foreignId(...)->constrained('users')->onDelete('cascade')`; `orders.customer_id` cascades from `customers`.
- **Trigger:** deleting tenant user #1 (off-boarding, DB cleanup) cascades to delete **every** self-registered customer and auto-created product-variant group, which cascades further into their orders/order_items — silent, wide-reaching data loss unrelated to the deleter's intent.

### HIGH

**B4. Stock deduction has a lost-update race — no row locking.**
`app/Services/ProductStockService.php:10-25` reads `product.stock`, computes new value in PHP, then `save()`s, with no `lockForUpdate()`.
- **Trigger:** two concurrent POS sales of the last unit both read `stock = 1`, both pass the check, both write `stock = 0` — overselling with no recorded shortfall.

**B5. Bulk stock-increase import bypasses the transaction ledger entirely.**
`app/Http/Controllers/ProductController.php:208-220` (`exportAddQuantity()`, called from `ImportController.php` when `type=increase`):
```php
$product->stock = (int) $product->stock + (int) $request->stock;
return $product->save();
```
No `InventoryStockTransaction` row is created, unlike every other stock-mutating path.
- **Trigger:** bulk "stock increase" import silently drifts `products.stock` away from the `inventory_stock_transactions` audit ledger — stock reports will never reconcile.

**B6. Sales-return uses the wrong field name, so `order.total_amount` is never adjusted.**
`app/Http/Controllers/OrderController.php:405-408`:
```php
'total_amount' => $order->total_amount - collect($validatedData['products'])->sum('sell_price'),
```
`SalesReturnRequest.php:26` validates the field as `products.*.price`, not `sell_price`, so `sum('sell_price')` is always `0`.
- **Trigger:** processing a sales return correctly restocks items but leaves `order.total_amount` unchanged — order totals permanently overstate revenue after any return.

**B7. Website cart discount calculation ignores quantity.**
`app/Services/Websites/WebsiteOrderService.php:75-86` (`calculateCartDiscount`) sums `original_price - final_price` per distinct cart line but never multiplies by `$cartItem->quantity`.
- **Trigger:** buying 3 units of a 20%-off product only discounts 1 unit's worth — customer is overcharged for the other 2.

**B8. `ProductVariant::afterStore()`/`afterUpdate()` throw an uncatchable `TypeError` when `tags`/`product_attributes` are omitted.**
`app/Models/ProductVariant.php:42,45` (store) and `:96,99` (update):
```php
foreach (request()->tags as $tag) { ... }
foreach (request()->product_attributes as $attr) { ... }
```
Neither field is validated as required in `StoreProductVariantRequest`. If omitted, `request()->tags` is `null`, and `foreach (null as ...)` throws `TypeError`, which extends `\Error`, not `\Exception` — so it slips past every surrounding `catch (\Exception $e)`.
- **Trigger:** `POST /api/product-variant` without a `tags` field → uncaught 500, transaction left in an inconsistent state.

**B9. Tenant resolution trusts an unvalidated client header, combined with fully open CORS.**
`app/Http/Middleware/InitializeTenancyByHeader.php:12-14` resolves the tenant purely from the `X-Tenant` header with no cross-check against `Host`/`Origin`. `config/cors.php` has `allowed_origins => ['*']`, `allowed_headers => ['*']`. This also governs the fully public `routes/website/guest.php` routes (no auth middleware).
- **Trigger:** any client from any origin can browse/act against an arbitrary tenant simply by setting `X-Tenant: <other-store>` — nothing binds the resolved tenant to the domain actually requested.

### MEDIUM

**B10. Sequential code generation (`products.code`, `orders.code`) races under concurrency.**
`Product.php:57` uses `lockForUpdate()` but `mergeRequest()` runs in `SuperController::store()` **before** `DB::beginTransaction()` starts, so the lock is released immediately and provides no protection. `Order::getCode()` has no locking at all. Both columns are unique.
- **Trigger:** two simultaneous product/order creations compute the same next code → second insert fails on unique-constraint violation.

**B11. Feature flags (`website_enabled`, `maintenance_mode`) are defined but never enforced.**
`FeatureService::enabled()` is never called anywhere; no `WebsiteEnabledMiddleware` exists (despite being referenced in the project's own `CLAUDE.md`); no website route applies a feature check.
- **Trigger:** disabling a tenant's website (`website_enabled = false`) has zero runtime effect — the API stays fully live.

### LOW

**B12. Dead import in `bootstrap/app.php:4`** — imports `App\Http\Middleware\InitializeTenant`, which doesn't exist (only `InitializeTenancyByHeader.php` does). Harmless but suggests incomplete middleware wiring worth a sanity check.

**Verified clean:** VAT formula (`sell_price/1.13`, `subtotal*13/113`) is consistent across `Product.php`, `Order.php`, `CreateOrder.php`. Multi-step writes in `Order::afterStore()`/`CreateOrder`/`CartService` are correctly wrapped in DB transactions.

---

## Frontend (`pos-frontend`)

### CRITICAL

**F1. `Productable.jsx:283` calls `handleWeightChange`, which is never defined.**
The Weight `TextField`'s `onChange` (rendered whenever `isWeightable()` is true) calls a function that doesn't exist in the component — only `handleQuantityChange`, `handleSellPriceChange`, `handleItemDiscountChange`, `handleRemoveOrderItem` are defined. The working version lives in the old, now-unused `useProductable.jsx:126` hook; the migration to the Redux-backed `orderFormSlice` never ported weight handling over (`Item` in `orderFormSlice.ts` has no `weight` field, no `updateItemWeight` action, and `recalculate()` never factors weight into `lineTotal`).
- **Trigger:** any tenant with `weightable` enabled opens the POS Add Sale form and edits an item's weight → `ReferenceError: handleWeightChange is not defined`, crashing the order form. Weight-based selling is entirely non-functional.

**F2. Cash-drawer close hardcodes `cash_sales: 0`.**
`src/pages/cash-denominations/AddCashDenominationModal.jsx:440` — `cash_sales: 0, // Default to 0, can be updated later if needed` is sent on every "Close Cash" submission regardless of actual sales during the shift.
- **Trigger:** closing a cash drawer at end-of-day always reports 0 cash sales, making `expected_total_npr`/`difference_npr` reconciliation meaningless — shows a fake "shortage" equal to the day's real cash sales even on a perfectly balanced drawer.

### HIGH

**F3. POS order-form Redux state is never persisted despite `redux-persist` being a listed dependency.**
`src/redux/store.ts` has no `persistReducer`/`persistStore`/`PersistGate` anywhere. Cart items, discounts, split payments, customer, and payment method live purely in memory.
- **Trigger:** browser refresh/crash mid-sale wipes the entire in-progress order with no warning — cashier must re-scan everything. Contradicts the documented state model in the project's own `CLAUDE.md`.

**F4. `restoreSession` action in `authSlice.ts:41` is dead code — never dispatched.**
Route guards read `localStorage.getItem("user")` directly instead, so login gating still functions today. But `state.auth.user`/`isAuthenticated`/`token` are always `null`/`false` after reload. Latent, not yet actively broken (no component currently reads `state.auth.*`), but any future feature relying on Redux auth state will silently see "logged out" after every refresh.

**F5. `queryClient.invalidateQueries(<string>)` misuse under TanStack Query v5 (5 locations) — targeted cache invalidation silently no-ops.**
v5 expects a filters object (`{ queryKey: [...] }`), not a bare string; the correct pattern is already used elsewhere in the same codebase (`useCreateOrder.ts:27`).
- `src/pages/orders/Model.jsx:462` — `invalidateQueries("customer-all")` (real key: `["customers", page, perPage, searchQuery]`)
- `src/pages/website-orders/Model.jsx:381`
- `src/pages/attributes/Model.jsx:49`
- `src/pages/purchase/Model.jsx:137`
- `src/pages/variant-product/Model.jsx:137`
- **Trigger:** creating a customer from the Add-Order modal doesn't invalidate the real customers query key — masked in that one spot by an optimistic local-array update, but any other open view keeps stale cached data. Same mismatch likely affects the other 4 call sites' target caches.

### MEDIUM

**F6. `src/utilities/domain.ts:11-13` subdomain parsing misclassifies apex domains, `www.`-prefixed hosts, and bare IPv4 addresses as tenants.**
```js
if (parts.length >= 3) {
  return parts[0]
}
```
Only checks label count, not actual structure.
- **Trigger:** visiting the bare apex `aitechnology.com.np` (3 labels) → sends `X-Tenant: "aitechnology"` (nonexistent tenant). `www.aitechnology.com.np` (4 labels) → `X-Tenant: "www"`. A raw IPv4 like `192.168.1.5` (4 dot-separated segments, common in staging) → `X-Tenant: "192"` instead of falling back to `VITE_DEFAULT_TENANT`. Backs every API call via `apiFetch.ts`/`apiFetchWeb.ts`/`useCustomForm.ts`.

**F7. `orderFormSlice.ts:339-344` `setCashReceived` computes change with unrounded floating-point subtraction.**
Every other derived value in the slice goes through `toFixedNum`; `change` doesn't.
- **Trigger:** `totalAmount = 133.33`, `cashReceived = 200` can produce a result like `66.67000000000002` displayed directly to the cashier for making change.

**F8. `orderFormSlice.ts:80-92` `recalculate()` rounds `discountedPrice` and `total` independently from different intermediate values.**
`lineTotal` is computed from the unrounded `discountedPrice`, then both are rounded separately for display.
- **Trigger:** non-evenly-dividing prices/discounts (e.g. `sell_price=99.99`, `discount=15%`) produce a displayed unit price × quantity that's off by ±0.01 from the displayed line total — visible discrepancy on the invoice screen.

**F9. `CashDenominationDetails.jsx:87-92` cash-difference color logic is inverted.**
```jsx
color={Number(session?.difference_npr) >= 0 ? "error" : "success"}
```
A balanced (`0`) or over drawer renders red ("error"); a cash shortage (the operationally bad case) renders green ("success") — backwards for a reconciliation screen.

**F10. `src/pages/website/pages/Cart.tsx:124,280` — free/100%-off items silently billed at full price.**
```js
const price = Number(item.product?.discount_details?.final_price) || item.product?.sell_price || 0
```
A legitimate `final_price = 0` is falsy, so `||` falls through to the undiscounted `sell_price`.
- **Trigger:** any item with a 100%-off promotion shows and charges full price in both cart subtotal and line total on the customer-facing website.

**F11. `src/hooks/useProducts.jsx:15` passes the removed v4 boolean `keepPreviousData: true` under TanStack Query v5.**
v5 requires `placeholderData: keepPreviousData` (imported helper); the boolean form is a silent no-op.
- **Trigger:** POS product search table flashes to loading/empty state on every page/search change instead of keeping previous results visible. Same root cause pattern as F5.

**F12. `orderFormSlice.ts` (`addItem`, `updateItemQuantity`) enforce no stock cap on the frontend.**
`addItem` only floors quantity at 1; `updateItemQuantity` only guards against `<= 0`; neither compares against `item.stock`. UI (`Productable.jsx`) sets `min: 0` but no `max` on the quantity field.
- **Trigger:** cashier can add/increment a cart line well past on-hand stock with no client-side warning — relies entirely on backend rejection at submit time, if any.

### LOW

**F13. `apiFetch.ts:75-76` / `apiFetchWeb.ts:66-67` — unreachable code after `throw`.**
```js
throw data
throw new Error(data?.message || "Something went wrong")
```
The second throw is dead. Harmless, but signals the error path was left half-refactored: callers only ever catch the raw `data` object, never a proper `Error` — inconsistent with `useCustomForm.ts`'s `api()`, which does throw an `Error`.

**Verified clean:** website Product vs. ProductVariant handling (`ProductDetail.tsx`, `ProductGroup.tsx`) correctly follows the documented inversion — `selectedVariant` (misleadingly named, actually a `Product`) is the source of `sell_price`/`stock`/`sku`. No confusion bug found there.

---

## Summary

| Severity | Backend | Frontend |
|---|---|---|
| Critical | 3 | 2 |
| High | 6 | 3 |
| Medium | 2 | 7 |
| Low | 1 | 1 |

Highest-priority items to look at first: **B1** (corrupts Category/Brand rows and 500s), **B2** (website checkout likely broken outright via FK violation), **B3** (cascade delete can wipe customers/orders), **F1** (weight-based POS sale crashes), **F2** (cash reconciliation numbers are fake).
