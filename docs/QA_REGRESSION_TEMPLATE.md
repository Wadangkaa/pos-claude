# QA Regression Template — Tenant, Branches, Roles & Permissions

## Purpose

This is the **blueprint** for a full-application regression pass. It walks one
new tenant from provisioning (which creates no branch) through the company
admin creating the first branch, further branch creation, per-role checks on a
branch, branch data isolation, the company admin's consolidated view, and the
per-branch website lock.

This file is never filled in. Each run produces its own dated result file.

## How to run

1. Run this only after an explicit request to execute the QA regression.
2. Copy this file to `docs/qa-runs/QA_RUN_YYYY-MM-DD.md` (add `_2`, `_3` for
   further runs on the same date). Delete the "Purpose" and "How to run"
   sections from the copy.
3. Fill in **Run details**, then work through the sections in order — later
   sections depend on data created in earlier ones.
4. Replace every `⬜` in the copy with a result:

   | Mark | Meaning                                                 |
   | ---- | ------------------------------------------------------- |
   | ✅   | Passed — observed exactly as expected                   |
   | ❌   | Failed — add a row to **Defects** with the check ID     |
   | ⏭️   | Not applicable or skipped — say why in the Notes column |

5. A check passes only if it was actually performed in this run. Do not carry
   results over from an earlier run file.
6. Fill in **Summary** last. Never record passwords or tokens in a run file.

### Pass criteria for any page check

A sidebar entry or route passes only when it was opened and:

1. the browser stays authenticated (no redirect to login);
2. the URL matches the expected route; and
3. the page renders without the application error screen or a failed
   (4xx/5xx) data request.

An entry that is hidden from a role because of its permissions is a **passed**
permission check, not a failed navigation check.

## Run details

| Field                    | Value                               |
| ------------------------ | ----------------------------------- |
| Date                     |                                     |
| Run by                   |                                     |
| Environment              | local / staging / VPS               |
| Backend branch @ commit  |                                     |
| Frontend branch @ commit |                                     |
| Test tenant name / slug  |                                     |
| Website host             | `{tenant}.{domain}`                 |
| Company admin host       | `admin-{tenant}.{domain}`           |
| Branch A host (Main)     | `main-{tenant}.{domain}` (from B0)  |
| Branch B host            | `{branch-b-slug}-{tenant}.{domain}` |

Locally `{domain}` is `localhost`, the frontend is on port 3000 and the central
API is `api.localhost:8000`. A new tenant's first user is the tenant email with
the seeder's default password.

A new tenant has **no branch**. "Main" in this template is the first branch,
which the Owner creates in B0 and names `Main`; it is Branch A everywhere below.

### Test accounts

Create these during section B. Record names and emails only.

| Account           | Role                                              | Branch assignment            | Email        |
| ----------------- | ------------------------------------------------- | ---------------------------- | ------------ |
| Owner             | `super-admin`                                     | Main + Branch B (as creator) | tenant email |
| Branch admin      | `admin`                                           | Branch B                     |              |
| Manager           | `manager`                                         | Branch B                     |              |
| Cashier           | `cashier`                                         | Branch B                     |              |
| Main-only cashier | `cashier`                                         | Main only                    |              |
| Multi-branch user | `cashier`                                         | Main + Branch B              |              |
| Multi-role user   | `cashier` + custom `stock-keeper` (created in F9) | Branch B                     |              |
| Unassigned user   | `cashier`                                         | none                         |              |

---

## A. Tenant provisioning

Central API, authenticated as a central user.

| Status | ID  | Check                                                                           | Expected                                                                                                      | Notes |
| ------ | --- | ------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------- | ----- |
| ⬜     | A1  | `POST /auth/login` on the central API                                           | Token returned                                                                                                |       |
| ⬜     | A2  | `POST /tenants` with name, email, phone                                         | "Tenant created successfully"; slug derived from the name                                                     |       |
| ⬜     | A3  | `POST /tenants` again with the same email                                       | Rejected with a validation error                                                                              |       |
| ⬜     | A4  | Tenant database                                                                 | `ant_pos_{uuid}` exists and is migrated                                                                       |       |
| ⬜     | A5  | Domain mappings                                                                 | Website host and `admin-` host exist; there is no branch host yet                                             |       |
| ⬜     | A6  | No seeded branch                                                                | The tenant has no branch and no settings rows                                                                 |       |
| ⬜     | A7  | Seeded owner                                                                    | User with the tenant email, role `super-admin`, assigned to no branch                                         |       |
| ⬜     | A8  | Seeded roles                                                                    | `super-admin`, `admin`, `manager`, `cashier` exist                                                            |       |
| ⬜     | A9  | Seeded customer                                                                 | The walk-in `Cash` customer exists                                                                            |       |
| ⬜     | A10 | Feature flags                                                                   | Central `features` has `website_enabled` and `maintenance_mode` for the tenant, both off, with no `branch_id` |       |
| ⬜     | A11 | `php artisan features:set {tenant} website_enabled on` before any branch exists | Refused: "Branch not found for this tenant."; the flags are unchanged                                         |       |

## B. Company admin portal

On the company admin host, signed in as the Owner.

### B0. Before and at the first branch

Run this first. B0.1–B0.7 are done while the tenant still has no branch.

| Status | ID    | Check                                                                                          | Expected                                                                                                                                          | Notes |
| ------ | ----- | ---------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- | ----- |
| ⬜     | B0.1  | Owner logs in at the website host                                                              | Rejected with "Create a branch in the company admin portal first"; stays on the login page, no session                                            |       |
| ⬜     | B0.2  | Storefront at the website host                                                                 | Shows the website-disabled page; `get-features` reports both flags off; other `/api/website/*` calls return 403                                   |       |
| ⬜     | B0.3  | Owner logs in at the admin host                                                                | Succeeds; lands on `/pos/dashboard`                                                                                                               |       |
| ⬜     | B0.4  | Company dashboard with no branch                                                               | Shows "This company has no branch yet" with an Add branch button; all figures are zero except Customers (1, the seeded `Cash`); no failed request |       |
| ⬜     | B0.5  | Click Add branch                                                                               | Goes to `/pos/branches`; the list is empty                                                                                                        |       |
| ⬜     | B0.6  | Open Customers, Reports › Sales, Reports › Product and Users                                   | Each renders without an error; Customers lists `Cash`, Users lists the Owner                                                                      |       |
| ⬜     | B0.7  | Use the Owner's admin-host token against the website host API (for example `GET /api/product`) | 403 with the same "Create a branch…" message; nothing is saved without a branch                                                                   |       |
| ⬜     | B0.8  | Create the first branch, named `Main`                                                          | Saved and listed                                                                                                                                  |       |
| ⬜     | B0.9  | Main hostname                                                                                  | `main-{tenant}.{domain}` mapping created automatically; its login page loads without a certificate warning                                                                      |       |
| ⬜     | B0.10 | Creator assignment                                                                             | Owner is assigned to Main                                                                                                                         |       |
| ⬜     | B0.11 | Main defaults                                                                                  | Cash payment method, payment/sales statuses, currency, NPR notes, inventory setting present                                                       |       |
| ⬜     | B0.12 | Main feature flags                                                                             | No new `features` rows are added for Main: the tenant's original rows (no `branch_id`) apply to it                                                |       |
| ⬜     | B0.13 | Company dashboard again                                                                        | The "no branch yet" notice is gone; Main is in the branch filter                                                                                  |       |
| ⬜     | B0.14 | Open the website host's login page and its storefront root | Both forward to the same page on the Main host (`main-{tenant}.{domain}`); the Owner signs in there |       |
| ⬜     | B0.15 | Delete Main while it is the only branch                                                        | Rejected: "The only branch of a company cannot be deleted."                                                                                       |       |

### B1. Access and navigation

| Status | ID   | Check                           | Expected                                                                   | Notes |
| ------ | ---- | ------------------------------- | -------------------------------------------------------------------------- | ----- |
| ⬜     | B1.1 | Owner logs in at the admin host | Lands on `/pos/dashboard`                                                  |       |
| ⬜     | B1.2 | Dashboard                       | `/pos/dashboard` shows the company (all-branch) summary                    |       |
| ⬜     | B1.3 | Branches                        | `/pos/branches` lists Main                                                 |       |
| ⬜     | B1.4 | Customers                       | `/pos/customers` opens                                                     |       |
| ⬜     | B1.5 | Reports › Sales                 | `/pos/sales-report` opens with the branch filter                           |       |
| ⬜     | B1.6 | Reports › Product               | `/pos/products-report` opens with the branch filter                        |       |
| ⬜     | B1.7 | Users                           | `/pos/users` opens                                                         |       |
| ⬜     | B1.8 | Branch-only entries             | No POS, inventory, HR, Roles or Configuration entries in the admin sidebar |       |

### B2. Branch management

| Status | ID   | Check                                       | Expected                                                                                                                                 | Notes |
| ------ | ---- | ------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- | ----- |
| ⬜     | B2.1 | Create Branch B                             | Saved and listed                                                                                                                         |       |
| ⬜     | B2.2 | Branch B hostname                           | `{branch-b-slug}-{tenant}.{domain}` mapping created automatically                                                                        |       |
| ⬜     | B2.3 | Creator assignment                          | Owner is assigned to Branch B                                                                                                            |       |
| ⬜     | B2.4 | Branch B defaults                           | Own Cash method, statuses, currency, NPR notes, inventory setting                                                                        |       |
| ⬜     | B2.5 | Edit Branch B (phone, VAT/PAN, tax %)       | Changes saved and shown                                                                                                                  |       |
| ⬜     | B2.6 | Create a throwaway Branch C, then delete it | Removed from the list; its host no longer logs in                                                                                        |       |
| ⬜     | B2.7 | Branch B host opens                         | Login page loads on the Branch B hostname                                                                                                |       |
| ⬜     | B2.8 | Feature flags of later branches             | Branch B has its own `website_enabled` and `maintenance_mode` rows in central `features`, both off; the deleted Branch C's rows are gone |       |

### B3. Users and branch assignment

| Status | ID   | Check                                                            | Expected                                             | Notes |
| ------ | ---- | ---------------------------------------------------------------- | ---------------------------------------------------- | ----- |
| ⬜     | B3.1 | User form on the admin host                                      | Shows the branch assignment selector                 |       |
| ⬜     | B3.2 | Create Branch admin, Manager and Cashier for Branch B            | All three saved with the right role and branch       |       |
| ⬜     | B3.3 | Create Main-only cashier                                         | Saved, assigned to Main only                         |       |
| ⬜     | B3.4 | Create the Multi-branch user, assigned to both Main and Branch B | Saved; both branches shown on the user               |       |
| ⬜     | B3.5 | Manager logs in at the admin host                                | Rejected — admin host needs `super-admin`            |       |
| ⬜     | B3.6 | Cashier logs in at the admin host                                | Rejected                                             |       |
| ⬜     | B3.7 | Main-only cashier logs in at the Branch B host                   | Rejected — no assignment to that branch              |       |
| ⬜     | B3.8 | Create the Unassigned user with no branch selected               | Saved; shown with "No branch access"                 |       |
| ⬜     | B3.9 | Branch admin (`admin`, Branch B only) logs in at the admin host  | Rejected with "Company administrator access is required"; stays on the login page, no session |       |
| ⬜     | B3.10 | Use the Branch admin's Branch B token against the admin host API | 403 "Company administrator access is required"       |       |

### B4. Branch login access

Only users assigned to a branch can sign in on that branch's host. A user
assigned to several branches can sign in on each of them.

| Status | ID    | Check                                                             | Expected                                                                                 | Notes |
| ------ | ----- | ----------------------------------------------------------------- | ---------------------------------------------------------------------------------------- | ----- |
| ⬜     | B4.1  | Cashier (Branch B only) logs in at the Branch B host              | Succeeds; lands on `/pos/dashboard`                                                      |       |
| ⬜     | B4.2  | Same Cashier logs in at the Main host                             | Rejected with "You are not assigned to this branch"; stays on the login page, no session |       |
| ⬜     | B4.3  | Main-only cashier logs in at the Main host                        | Succeeds                                                                                 |       |
| ⬜     | B4.4  | Multi-branch user logs in at the Main host                        | Succeeds                                                                                 |       |
| ⬜     | B4.5  | Multi-branch user logs in at the Branch B host                    | Succeeds                                                                                 |       |
| ⬜     | B4.6  | Multi-branch user is signed in on both hosts at the same time     | Both sessions keep working independently                                                 |       |
| ⬜     | B4.7  | Unassigned user logs in at the Main host and at the Branch B host | Rejected on both                                                                         |       |
| ⬜     | B4.8  | Use the Cashier's Branch B token against the Main host API        | 403 — a token does not carry access to another branch                                    |       |
| ⬜     | B4.9  | Company admin removes Main from the Multi-branch user             | Login at Main is now rejected; Branch B still works                                      |       |
| ⬜     | B4.10 | That user's existing Main session makes another request           | Refused (403); no Main data returned                                                     |       |
| ⬜     | B4.11 | Company admin adds Main back                                      | Login at Main works again                                                                |       |
| ⬜     | B4.12 | Users list on each branch host                                    | Multi-branch user is listed on both; single-branch users only on their own               |       |

## C. Branch portal as Branch admin

On the Branch B host, signed in as the Branch admin. Every entry must be opened
from the sidebar.

### C1. Sidebar (41 entries)

| Status | Sidebar entry                   | Expected route            |
| ------ | ------------------------------- | ------------------------- |
| ⬜     | Dashboard                       | `/pos/dashboard`          |
| ⬜     | Sales                           | `/pos/sales`              |
| ⬜     | Website Orders                  | `/pos/website-orders`     |
| ⬜     | Sales Return                    | `/pos/sales-return`       |
| ⬜     | Discounts                       | `/pos/discount`           |
| ⬜     | Footfall                        | `/pos/footfall`           |
| ⬜     | Cash Denomination               | `/pos/cash-denominations` |
| ⬜     | Expenses                        | `/pos/expenses`           |
| ⬜     | Products                        | `/pos/products`           |
| ⬜     | Product Variants                | `/pos/product-variants`   |
| ⬜     | Categories                      | `/pos/categories`         |
| ⬜     | Attributes                      | `/pos/attributes`         |
| ⬜     | Tags                            | `/pos/tags`               |
| ⬜     | Brands                          | `/pos/brands`             |
| ⬜     | Purchase                        | `/pos/purchase`           |
| ⬜     | Stock Adjustment                | `/pos/stock-adjustment`   |
| ⬜     | Stock Transaction               | `/pos/stock-transactions` |
| ⬜     | Stock Audit                     | `/pos/stock-audit`        |
| ⬜     | Damage Products                 | `/pos/damage-products`    |
| ⬜     | Customers                       | `/pos/customers`          |
| ⬜     | Suppliers                       | `/pos/suppliers`          |
| ⬜     | Branches                        | `/pos/branches`           |
| ⬜     | Locations                       | `/pos/locations`          |
| ⬜     | Delivery Fees                   | `/pos/delivery-fees`      |
| ⬜     | Reports › Sales                 | `/pos/sales-report`       |
| ⬜     | Reports › Product               | `/pos/products-report`    |
| ⬜     | Exports                         | `/pos/exports`            |
| ⬜     | Staff                           | `/pos/staff`              |
| ⬜     | Attendance                      | `/pos/attendance`         |
| ⬜     | Incentives                      | `/pos/incentives`         |
| ⬜     | Users                           | `/pos/users`              |
| ⬜     | Roles & Permissions             | `/pos/roles`              |
| ⬜     | Configuration › Payment Methods | `/pos/payment-methods`    |
| ⬜     | Configuration › Sales Status    | `/pos/sales-status`       |
| ⬜     | Configuration › Payment Status  | `/pos/payment-status`     |
| ⬜     | Configuration › Attributes      | `/pos/attribute-names`    |
| ⬜     | Configuration › Inventory       | `/pos/inventory-configs`  |
| ⬜     | Configuration › Currency        | `/pos/currency-config`    |
| ⬜     | Configuration › Currency Note   | `/pos/note-config`        |
| ⬜     | Configuration › POS             | `/pos/pos-config`         |
| ⬜     | Configuration › Website         | `/pos/website-config`     |

### C2. Core workflow on Branch B

This creates the data that sections F and G rely on. Note the amounts.

| Status | ID    | Check                                            | Expected                                                                   | Notes |
| ------ | ----- | ------------------------------------------------ | -------------------------------------------------------------------------- | ----- |
| ⬜     | C2.1  | Create a category, brand and supplier            | Saved in Branch B                                                          |       |
| ⬜     | C2.2  | Create a product (SKU `{article}-{...}`)         | Product saved; its product group is auto-created from the article number   |       |
| ⬜     | C2.3  | Record a purchase for that product               | Stock increases; an "In" stock transaction is logged                       |       |
| ⬜     | C2.4  | Create a customer (note the phone)               | Saved                                                                      |       |
| ⬜     | C2.5  | Make a POS sale to that customer with payment    | Order saved, stock decreases, 13% VAT shown, receipt has Branch B's header |       |
| ⬜     | C2.6  | Sales return for part of that sale               | Return saved; stock comes back                                             |       |
| ⬜     | C2.7  | Stock adjustment                                 | Balance changes; stock transaction logged                                  |       |
| ⬜     | C2.8  | Record an expense                                | Listed under Branch B                                                      |       |
| ⬜     | C2.9  | Open and close a cash session with denominations | Saved against Branch B                                                     |       |
| ⬜     | C2.10 | Add a staff member and mark attendance           | Present/absent counts update                                               |       |
| ⬜     | C2.11 | Create an incentive plan                         | Plan listed with its sales ranges                                          |       |
| ⬜     | C2.12 | Queue a sales export                             | Appears on `/pos/exports` and downloads                                    |       |
| ⬜     | C2.13 | Change POS configuration (invoice message)       | Saved for Branch B                                                         |       |

### C3. Branch-host restrictions for an admin

| Status | ID   | Check                          | Expected                                       | Notes |
| ------ | ---- | ------------------------------ | ---------------------------------------------- | ----- |
| ⬜     | C3.1 | Branches page on a branch host | No option to create a branch (admin host only) |       |
| ⬜     | C3.2 | User form on a branch host     | No branch selector; new user lands in Branch B |       |
| ⬜     | C3.3 | Users list                     | Shows only Branch B users                      |       |

## D. Branch portal as Manager

On the Branch B host, signed in as the Manager. The manager has every
permission except role management.

| Status | ID  | Check                                              | Expected                                          | Notes |
| ------ | --- | -------------------------------------------------- | ------------------------------------------------- | ----- |
| ⬜     | D1  | Sidebar                                            | The same 40 entries as C1, each opening its route |       |
| ⬜     | D2  | Roles & Permissions                                | Not in the sidebar                                |       |
| ⬜     | D3  | Open `/pos/roles` by typing the URL                | No role data shown; the roles API returns 403     |       |
| ⬜     | D4  | Create and edit a product                          | Allowed                                           |       |
| ⬜     | D5  | Make a sale and a sales return                     | Allowed                                           |       |
| ⬜     | D6  | Create and edit a Branch B user                    | Allowed                                           |       |
| ⬜     | D7  | Delete the user assigned to both Main and Branch B | Rejected                                          |       |
| ⬜     | D8  | Sales and Product reports                          | Open and show Branch B data only                  |       |
| ⬜     | D9  | Incentives and attendance                          | Can view and update                               |       |
| ⬜     | D10 | Configuration pages                                | Can view and save                                 |       |

## E. Branch portal as Cashier

On the Branch B host, signed in as the Cashier.

### E1. Sidebar (10 entries)

| Status | Sidebar entry             | Expected route            |
| ------ | ------------------------- | ------------------------- |
| ⬜     | Dashboard                 | `/pos/dashboard`          |
| ⬜     | Sales                     | `/pos/sales`              |
| ⬜     | Website Orders            | `/pos/website-orders`     |
| ⬜     | Sales Return              | `/pos/sales-return`       |
| ⬜     | Footfall                  | `/pos/footfall`           |
| ⬜     | Cash Denomination         | `/pos/cash-denominations` |
| ⬜     | Expenses                  | `/pos/expenses`           |
| ⬜     | Purchase                  | `/pos/purchase`           |
| ⬜     | Damage Products           | `/pos/damage-products`    |
| ⬜     | Customers                 | `/pos/customers`          |
| ⬜     | All other entries from C1 | Not rendered              |

### E2. Permissions in practice

| Status | ID    | Check                                                        | Expected                                                                                                                         | Notes |
| ------ | ----- | ------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------- | ----- |
| ⬜     | E2.1  | Search a product while making a sale                         | Product found (lookup permission)                                                                                                |       |
| ⬜     | E2.2  | Complete a sale with payment                                 | Allowed                                                                                                                          |       |
| ⬜     | E2.3  | Dashboard                                                    | Summary loads without errors                                                                                                     |       |
| ⬜     | E2.4  | Sales table                                                  | No delete action                                                                                                                 |       |
| ⬜     | E2.5  | Customers table                                              | Can add; no edit or delete action                                                                                                |       |
| ⬜     | E2.6  | Sales return, expense, cash denomination, footfall, purchase | Can create each                                                                                                                  |       |
| ⬜     | E2.7  | Type `/pos/products`                                         | Product list may load (lookup permission) but shows no add, import, edit or delete actions; create/update/delete API returns 403 |       |
| ⬜     | E2.8  | Type `/pos/users` and `/pos/roles`                           | No data; API returns 403                                                                                                         |       |
| ⬜     | E2.9  | Type `/pos/sales-report`                                     | No report data; API returns 403                                                                                                  |       |
| ⬜     | E2.10 | Type `/pos/pos-config`                                       | Cannot view or save settings                                                                                                     |       |
| ⬜     | E2.11 | Type `/pos/staff` and `/pos/incentives`                      | No data; API returns 403                                                                                                         |       |

## F. Roles and permissions management

On the Branch B host, signed in as the Branch admin.

### Managing roles

| Status | ID  | Check                                                      | Expected                                                   | Notes |
| ------ | --- | ---------------------------------------------------------- | ---------------------------------------------------------- | ----- |
| ⬜     | F1  | Open each built-in role                                    | `super-admin`, `admin`, `manager`, `cashier` are read-only |       |
| ⬜     | F2  | Create a custom role with only Dashboard and Products view | Saved                                                      |       |
| ⬜     | F3  | Assign the custom role to a new user and log in as them    | Sidebar shows only Dashboard and Products                  |       |
| ⬜     | F4  | That user tries to create a product                        | No create action; API returns 403                          |       |
| ⬜     | F5  | Add Products create to the custom role; user logs in again | Create now works                                           |       |
| ⬜     | F6  | Give a user two roles (custom + cashier)                   | Sidebar is the union of both                               |       |
| ⬜     | F7  | Delete the custom role                                     | Removed; built-in roles cannot be deleted                  |       |
| ⬜     | F8  | Switch between two tenants in the same browser             | Permissions follow the tenant; no stale access             |       |

### Users with more than one role

Still as the Branch admin for setup, then signed in as the Multi-role user on
the Branch B host. Uses the product and stock created in C2.

| Status | ID  | Check                                                                                                         | Expected                                                                                                                   | Notes |
| ------ | --- | ------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- | ----- |
| ⬜     | F9  | Create custom role `stock-keeper`: Products view/create/update, Stock Adjustments view/create, Suppliers view | Saved                                                                                                                      |       |
| ⬜     | F10 | Create the Multi-role user and select both `cashier` and `stock-keeper` in the user form                      | Saved with both roles                                                                                                      |       |
| ⬜     | F11 | Users table and edit form for that user                                                                       | Roles column shows both roles; Edit opens with both preselected                                                            |       |
| ⬜     | F12 | Multi-role user logs in                                                                                       | Sidebar is the cashier's 10 entries plus Products, Stock Adjustment and Suppliers (13)                                     |       |
| ⬜     | F13 | Cashier feature: complete a sale with payment                                                                 | Allowed                                                                                                                    |       |
| ⬜     | F14 | Cashier feature: sales return and expense                                                                     | Allowed                                                                                                                    |       |
| ⬜     | F15 | Stock-keeper feature: create and edit a product                                                               | Allowed                                                                                                                    |       |
| ⬜     | F16 | Stock-keeper feature: stock adjustment                                                                        | Allowed; stock changes                                                                                                     |       |
| ⬜     | F17 | Something neither role grants: delete a product, open Users, Roles, Reports, Configuration                    | Not in the sidebar; API returns 403                                                                                        |       |
| ⬜     | F18 | Remove `stock-keeper` from the user (keep `cashier`)                                                          | Products, Stock Adjustment and Suppliers disappear; product create returns 403; sales still work                           |       |
| ⬜     | F19 | Swap: give the user only `stock-keeper`                                                                       | Can manage products; cannot make a sale (403); cashier-only entries gone                                                   |       |
| ⬜     | F20 | Give the user two built-in roles, `cashier` + `manager`                                                       | Manager's access applies: 40 sidebar entries, still no Roles & Permissions                                                 |       |
| ⬜     | F21 | Give the Multi-branch user two roles and log in on both branch hosts                                          | Same roles and sidebar on Main and on Branch B                                                                             |       |
| ⬜     | F22 | Manager opens the user form                                                                                   | Roles list offers only roles the manager may assign (no `admin`, `super-admin`); assigning one through the API returns 403 |       |
| ⬜     | F23 | Try to save a user with no role selected                                                                      | Rejected: "Select at least one role"                                                                                       |       |
| ⬜     | F24 | Branch admin opens the user form                                                                              | Roles list has no `super-admin`; assigning it, or editing or deleting the Owner, through the API returns 403               |       |

## G. Branch isolation and shared customers

Compare Main (Branch A) with Branch B. Use the Owner, who is assigned to both,
and the Multi-branch user for G14–G15.

| Status | ID  | Check                                                         | Expected                                                                                             | Notes |
| ------ | --- | ------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------- | ----- |
| ⬜     | G1  | Products on the Main host                                     | Branch B's product is not listed                                                                     |       |
| ⬜     | G2  | Create a product on Main with the same SKU as Branch B's      | Allowed — SKUs are unique per branch                                                                 |       |
| ⬜     | G3  | Suppliers, brands, categories, discounts on Main              | Branch B's records are not listed                                                                    |       |
| ⬜     | G4  | Sales, purchases, sales returns, expenses on Main             | Branch B's records are not listed                                                                    |       |
| ⬜     | G5  | Stock transactions and cash sessions on Main                  | Branch B's records are not listed                                                                    |       |
| ⬜     | G6  | Staff, attendance, incentives on Main                         | Branch B's records are not listed                                                                    |       |
| ⬜     | G7  | Configuration on Main                                         | Branch B's invoice message change (C2.13) is not applied                                             |       |
| ⬜     | G8  | Exports page on Main                                          | Branch B's export is not listed                                                                      |       |
| ⬜     | G9  | **Customers on Main**                                         | Branch B's customer (C2.4) **is** listed                                                             |       |
| ⬜     | G10 | Sell to that shared customer on Main                          | Allowed; history shows on the customer                                                               |       |
| ⬜     | G11 | On Main, call a list API with `branch_id` of Branch B         | Ignored — still returns Main's data                                                                  |       |
| ⬜     | G12 | On Main, submit a sale using Branch B's product ID            | Rejected                                                                                             |       |
| ⬜     | G13 | Make one sale on Main (note the amount)                       | Saved against Main                                                                                   |       |
| ⬜     | G14 | Multi-branch user on the Main host, then on the Branch B host | Each session lists only that branch's products, sales and stock; the receipt header is that branch's |       |
| ⬜     | G15 | Multi-branch user makes a sale on each host                   | Each sale is saved against the host's branch, never the other                                        |       |

## H. Company admin — consolidated view

Back on the company admin host as the Owner, after sections C–G.

| Status | ID  | Check                                                | Expected                                                | Notes |
| ------ | --- | ---------------------------------------------------- | ------------------------------------------------------- | ----- |
| ⬜     | H1  | Dashboard, all branches                              | Totals equal Main + Branch B sales from this run        |       |
| ⬜     | H2  | Dashboard filtered to Branch B                       | Only Branch B's totals                                  |       |
| ⬜     | H3  | Sales report, All branches                           | Includes sales from both branches                       |       |
| ⬜     | H4  | Sales report, Branch B selected                      | Matches what the Branch B host shows (D8)               |       |
| ⬜     | H5  | Sales report, Main selected                          | Matches the Main sale (G13)                             |       |
| ⬜     | H6  | Sales report order-type filter (All / POS / Website) | Totals change accordingly                               |       |
| ⬜     | H7  | Product report, all vs. selected branch              | Stock and sales figures follow the filter               |       |
| ⬜     | H8  | Export a report with a branch selected               | The file contains only that branch                      |       |
| ⬜     | H9  | Customers                                            | Customers created on both branches are listed once each |       |
| ⬜     | H10 | Customer history                                     | Shows purchases from both branches                      |       |
| ⬜     | H11 | Users                                                | Users from all branches listed with their assignments   |       |
| ⬜     | H12 | New users and roles created on Branch B              | Visible here without a manual refresh of data           |       |

## I. Website

### Storefront

On the website host, after `website_enabled` has been switched on for Main
(I8). Mark I1–I5 ⏭️ only if the website cannot be switched on in this environment.

| Status | ID  | Check                                        | Expected                                                                 | Notes |
| ------ | --- | -------------------------------------------- | ------------------------------------------------------------------------ | ----- |
| ⬜     | I1  | Home page                                    | Products listed; filters work on desktop and mobile                      |       |
| ⬜     | I2  | Product detail and add to cart               | Cart updates                                                             |       |
| ⬜     | I3  | Customer registers and checks out            | Order created as type `website`; stock reserved in the fulfilling branch |       |
| ⬜     | I4  | Staff accepts the order under Website Orders | Reservation consumed; that branch's stock decreases                      |       |
| ⬜     | I5  | Website customer in the POS customer list    | Same shared customer record                                              |       |

### Website lock per branch

Feature flags live in the central `features` table with a `branch_id`, so the
website can be opened or locked for one branch at a time. They are switched on
the server with `php artisan features:set {tenant} {key} {on|off} [--branch=CODE]`.
The website host has no storefront of its own: it forwards to the oldest
branch (Main), so on it you see Main's storefront or Main's disabled page. Run I6–I8 before the
storefront checks above and the rest after them. Always run this subsection.

| Status | ID  | Check                                                                      | Expected                                                                                                                                                             | Notes |
| ------ | --- | -------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----- |
| ⬜     | I6  | Storefront on the website, Main and Branch B hosts while every flag is off | Each shows the website-disabled page                                                                                                                                 |       |
| ⬜     | I7  | Storefront API on each of those hosts while off                            | Catalog and customer register/login return 403 (cart and customer orders: 401 with nobody signed in); `GET /api/website/get-features` returns 200 with the flags off |       |
| ⬜     | I8  | `features:set {tenant} website_enabled on` with no `--branch`              | Reports the flag on for Main (the first branch); the storefront opens on the website host and the Main host                                                          |       |
| ⬜     | I9  | Branch B host while only Main is on                                        | Still disabled: page shows the disabled message and the storefront API returns 403                                                                                   |       |
| ⬜     | I10 | `features:set {tenant} website_enabled on --branch={Branch B code}`        | Branch B's storefront opens and lists Branch B's products only                                                                                                       |       |
| ⬜     | I11 | `features:set {tenant} website_enabled off --branch=MAIN`                  | The website host and Main host are locked again; Branch B's storefront stays open                                                                                    |       |
| ⬜     | I12 | `get-features` on the Main host and on the Branch B host                   | Each reports its own branch's flags (Main off, Branch B on)                                                                                                          |       |
| ⬜     | I13 | A signed-in website customer on Branch B, then lock Branch B               | Their cart and order requests return 403; the page falls back to the disabled message                                                                                |       |
| ⬜     | I14 | Staff POS on a branch host whose website is locked                         | Login, sales and the rest of the POS keep working; only the storefront is locked                                                                                     |       |
| ⬜     | I15 | Company admin host while Main's website is locked                          | The root address goes to the admin app (or its login); login and the admin API work                                                                                  |       |
| ⬜     | I16 | `features:set {tenant} maintenance_mode on --branch={Branch B code}`       | `get-features` reports it on for Branch B and off for Main; switch it off again afterwards                                                                           |       |
| ⬜     | I17 | Create a new branch while Main's website is on                             | The new branch gets its own rows, both off; its storefront is disabled until switched on                                                                             |       |
| ⬜     | I18 | `features:set` with an unknown branch code, and with an unknown key        | "Branch not found for this tenant." and the usage message; no row is changed                                                                                         |       |

## J. Cleanup

Local and staging only. Never delete a real tenant.

| Status | ID  | Check                                                         | Expected                                       | Notes |
| ------ | --- | ------------------------------------------------------------- | ---------------------------------------------- | ----- |
| ⬜     | J1  | `DELETE /tenants/{id}` with a wrong confirmation              | Rejected                                       |       |
| ⬜     | J2  | `DELETE /tenants/{id}` with the tenant domain as confirmation | Tenant, its database and its hostnames removed |       |

---

## Summary

| Section                  | Passed | Failed | Skipped |
| ------------------------ | ------ | ------ | ------- |
| A. Tenant provisioning   |        |        |         |
| B. Company admin portal  |        |        |         |
| C. Branch admin          |        |        |         |
| D. Manager               |        |        |         |
| E. Cashier               |        |        |         |
| F. Roles and permissions |        |        |         |
| G. Branch isolation      |        |        |         |
| H. Consolidated view     |        |        |         |
| I. Website               |        |        |         |
| J. Cleanup               |        |        |         |
| **Total**                |        |        |         |

**Overall result:** ✅ / ❌

## Defects

| Check ID | What happened | Expected | Severity |
| -------- | ------------- | -------- | -------- |
|          |               |          |          |
