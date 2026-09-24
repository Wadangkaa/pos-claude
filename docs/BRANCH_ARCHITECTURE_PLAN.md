# Caliber Branch Architecture Plan

This document is the implementation checklist for converting a company from
store-per-tenant operation to one tenant with multiple branches. Changes remain
uncommitted for review.

## Decisions

- Caliber Shoes becomes one tenant.
- Pulchok, Baneshwor, and Koteshwor become branches with separate inventory.
- The branch is selected by the staff POS hostname, not by a POS form control.
- `admin.caliber.com.np` is the company administration portal.
- `caliber.com.np` remains the public website and automatically selects a
  fulfillment branch with stock.
- Product catalog and prices are shared within the company; products are
  identified during migration by SKU/article number.
- Customers are shared within the company and are merged during migration by
  normalized phone number.

## Implementation Checklist

### 1. Foundation — Done

**Objective:** resolve a full hostname to a tenant and an optional forced branch,
then authorize the authenticated user against that branch.

**Verified:** full hostnames resolve the tenant and forced branch, branch login
requires an explicit user assignment, and branch-host order requests are scoped
to the matching branch.

### 2. Branch Inventory — Done

**Objective:** introduce branch product balances, branch stock ledger entries,
and stock transfers.

**Verified:** per-branch balances and branch-tagged inventory ledger entries are
in place. A transfer keeps company stock unchanged while deducting source stock
on send and adding destination stock on receipt.

### 3. Branch Operations — In Progress

**Objective:** make operational documents branch-scoped without adding a branch
selector to branch-host POS screens.

**Progress:** orders, purchases, stock adjustments, sales returns, expenses, and
their inventory ledger movements receive the forced branch automatically; order
listings are branch-scoped. Cash-session ownership and remaining operational
queries still need branch scoping before this phase is marked done.

### 4. Caliber Admin Portal — Done

**Objective:** provide company dashboard, consolidated reports, customer search
and history, and website-order oversight at the admin host.

**Verified:** admin-host requests require an `admin` or `super-admin` role and
receive a consolidated company dashboard. The existing shared Customers and
Website Orders screens are exposed through the admin-only navigation. Sales and
product reports (and their exports) offer an all-branches or selected-branch
view; staff branch hosts always enforce their assigned branch even if a request
URL is manipulated.

### 5. Website Fulfillment — Done

**Objective:** assign website orders to an in-stock fulfillment branch with an
authorized override.

**Verified:** checkout locks branch balances, selects the first active branch
that can fulfill the whole cart, and reserves that branch's stock. Confirmation
consumes the reservation and deducts the same branch. An `admin` or
`super-admin` on the admin hostname can move a pending order only to a branch
that has enough available stock; every override is recorded in the order's
fulfillment audit data.

### 6. Migration and Cutover — Ready for VPS Execution

**Objective:** dry-run and import Pulchok, Baneshwor, and Koteshwor data into
the Caliber tenant, then configure domains and redirects.

**Progress:** `branches:cutover-audit` is a read-only Artisan command that
validates explicit source-tenant → target-branch mappings and reports SKU,
normalized-phone, order, sales-return, and target-branch baseline totals. It
does not copy or delete data. Use it before approving the one-time import:

```bash
php artisan branches:cutover-audit <caliber-tenant-uuid> \
  --source=<pulchok-tenant-uuid>:PULCHOK \
  --source=<baneshwor-tenant-uuid>:BANESHWOR \
  --source=<koteshwor-tenant-uuid>:KOTESHWOR
```

**Remaining:** review the audit with the business owner, approve the final SKU
and normalized-phone merge policy, then run a separately authorized import and
DNS cutover. Source tenants must remain read-only until target totals reconcile.

**Import readiness:** `branches:cutover-import <caliber-tenant-uuid>
--source=<source-tenant-uuid>:<TARGET-BRANCH-CODE>` is dry-run only. It rejects
a missing target branch, missing shared SKU, non-empty target branch stock, or a
source tenant that was already imported. It reports mergeable normalized-phone
customers and source order/return totals without writing data.

**After import:** run `branches:cutover-validate` with the same source mapping.
Only when it reports matching order and sales-return counts/totals may
`branches:cutover-activate` be run with the same source mapping and explicit
confirmation. Activation marks the existing branch-host mapping active in the
application; DNS and reverse-proxy routing remain a VPS deployment action.

**Current blocker (local environment, 2026-09-11):** the local central registry
contains only the `branch` and `test` tenants. It has no Caliber target or
Pulchok/Baneshwor/Koteshwor source tenants, so the audit and import must be run
against a production database backup or the VPS after the real tenant UUIDs and
branch codes are supplied.

**VPS procedure:** use `docs/CALIBER_CUTOVER_RUNBOOK.md` to audit, dry-run,
import, reconcile, and activate one branch at a time. The actual audit and
import remain a production-change operation and must not be run locally without
a production data backup.

## Domain Map

| Hostname | Context |
| --- | --- |
| `pulchok.caliber.com.np` | Caliber tenant, Pulchok branch |
| `baneshwor.caliber.com.np` | Caliber tenant, Baneshwor branch |
| `koteshwor.caliber.com.np` | Caliber tenant, Koteshwor branch |
| `admin.caliber.com.np` | Caliber tenant, company administration |
| `caliber.com.np` | Caliber tenant, public website |

## Tenant Creation

Every tenant created through the central tenant API now receives website and
`admin.` `tenant_branch_domains` records automatically. Tenant seeding creates
a `Main` branch and assigns the tenant-email user the `super-admin` role and
that branch. Its branch-host mapping is derived directly from the tenant domain,
so it is created even though the central website mapping is added immediately
after seeding. Creating a branch from the admin portal automatically creates its
`{branch-slug}.{company-domain}` branch-host mapping and assigns the creator.
For an existing tenant, `php artisan branches:sync-domains <tenant-id>` creates
any missing branch-host mappings without changing operational data.
