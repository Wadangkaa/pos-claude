# Caliber VPS Cutover Runbook

Use this runbook once for each existing single-branch tenant: Pulchok,
Baneshwor, and Koteshwor. Do not delete or write to a source tenant until the
business owner has accepted the reconciliation result.

## Preconditions

- Deploy the branch-architecture code and run central plus tenant migrations.
- Create the `Caliber` company tenant and its `PULCHOK`, `BANESHWOR`, and
  `KOTESHWOR` branches on the Caliber admin host.
- Confirm every source product SKU exists in Caliber with the intended shared
  price/catalog data.
- Put the source branch into read-only maintenance while its audit/import is in
  progress. Keep a database backup of the central database and every affected
  tenant database.

## 1. Audit

Record tenant IDs from the central `tenants` table, then run the read-only audit:

```bash
php artisan branches:cutover-audit <caliber-tenant-id> \
  --source=<pulchok-tenant-id>:PULCHOK \
  --source=<baneshwor-tenant-id>:BANESHWOR \
  --source=<koteshwor-tenant-id>:KOTESHWOR \
  --json > storage/app/cutover-audit.json
```

Review the SKU differences, normalized-phone duplicates, order totals, and
sales-return totals. Resolve missing SKUs before continuing.

## 2. Dry-run one branch

Start with Pulchok. The command makes no changes and rejects missing SKUs,
non-empty destination stock, or a previously imported source:

```bash
php artisan branches:cutover-import <caliber-tenant-id> \
  --source=<pulchok-tenant-id>:PULCHOK --json
```

## 3. Confirmed import

Only after the dry-run is accepted, import that source. The confirmation must
match the source mapping exactly:

```bash
php artisan branches:cutover-import <caliber-tenant-id> \
  --source=<pulchok-tenant-id>:PULCHOK \
  --apply --confirm=<pulchok-tenant-id>:PULCHOK
```

This imports current opening stock, phone-merged customers, historical POS
orders/items/payments, and sales returns into the Caliber Pulchok branch.

## 4. Reconcile

```bash
php artisan branches:cutover-validate <caliber-tenant-id> \
  --source=<pulchok-tenant-id>:PULCHOK
```

Do not continue unless the command reports matching order and sales-return
counts and totals. Check Caliber admin reports and sample invoices as a second
business verification.

## 5. Activate the branch hostname

```bash
php artisan branches:cutover-activate <caliber-tenant-id> \
  --source=<pulchok-tenant-id>:PULCHOK \
  --confirm=<pulchok-tenant-id>:PULCHOK
```

Then point `pulchok.caliber.com.np` to the frontend/reverse proxy serving this
deployment. Ensure the API receives the full browser host in `X-Tenant`.
Repeat steps 2–5 one branch at a time for Baneshwor and Koteshwor.

## Rollback

Before DNS is changed, rollback means do not run activation: keep staff on the
source tenant and investigate the reconciliation mismatch. After DNS changes,
route the branch hostname back to the source deployment and restore the Caliber
database backup if an approved correction requires removing imported data.
Never run the importer twice for one source tenant; `branch_cutovers` blocks it.
