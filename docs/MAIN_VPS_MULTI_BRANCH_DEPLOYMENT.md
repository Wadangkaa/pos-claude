# Main VPS Multi-Branch Deployment Runbook

Use this runbook only after the branch-architecture work has been merged into
the root repository's `main` branch. It deploys the new company-admin and
retail-branch capability to the VPS currently serving `/var/www/pos`.

This is a production change: take a database backup and schedule a maintenance
window before running migrations. Do **not** use `migrate:fresh`, `db:wipe`, or
any other database-reset command.

## What this deployment changes

- The existing tenant can gain a company administration host, such as
  `admin.kstore.aitechnology.com.np`.
- A branch created in that admin portal gets an application mapping such as
  `lalitpur.kstore.aitechnology.com.np`, and its creator is assigned to it.
- Customers remain tenant-wide and are shared by every branch. Branches do not
  create customer copies.
- Products, inventory, staff operations, and other branch-owned data are scoped
  to the hostname used by the staff member.

## Short answer: what is automatic and what is not

| Item | Automatic after deployment? | Action required |
| --- | --- | --- |
| Git code on the VPS | No | Pull the merged `main` branch and update submodules. |
| Database schema and role permissions | No | Run central and tenant migrations; then sync role presets. |
| Existing-tenant admin/website/branch mappings | No | Perform the one-time bootstrap below. |
| Mapping for a branch created from the new admin portal | Yes | The app creates its database mapping and assigns the creator. |
| DNS for `branch.tenant.aitechnology.com.np` | No | Create a per-tenant wildcard DNS record. |
| Nginx request routing | Usually yes | Your `*.aitechnology.com.np` Nginx name matches nested subdomains too; verify it with `nginx -t`. |
| TLS for `branch.tenant.aitechnology.com.np` | No | Issue a per-tenant wildcard certificate and add its Nginx HTTPS server block. |

The current certificate for `*.aitechnology.com.np` is suitable for hosts such
as `kstore.aitechnology.com.np`, but not a two-label host such as
`lalitpur.kstore.aitechnology.com.np`. A per-tenant certificate for
`*.kstore.aitechnology.com.np` is required. Wildcard certificates require the
DNS-01 challenge. [Let's Encrypt challenge documentation](https://letsencrypt.org/docs/challenge-types/)

Nginx's leading wildcard name can match several name parts, so the existing
`server_name *.aitechnology.com.np` is capable of routing the nested host. The
TLS certificate is the limiting factor, not that server-name rule. [Nginx server-name documentation](https://nginx.org/en/docs/http/server_names.html)

## 1. Preflight on the VPS

Log in as the deployment user and confirm the code directory is clean before
replacing anything. If `git status` reports local edits, stop and preserve or
commit them first.

```bash
cd /var/www/pos
git status --short
git fetch origin
git log --oneline HEAD..origin/main
```

Back up the central database and every tenant database before migration. Use
your existing backup process; do not put database passwords in shell history or
this repository.

Check the backend's production domain settings as well. `DOMAIN` is used when
the application constructs tenant and branch hostnames:

```bash
cd /var/www/pos/pos-backend
grep -E '^(APP_ENV|APP_URL|DOMAIN)=' .env
```

For this VPS, `DOMAIN` must be `aitechnology.com.np`; `APP_URL` should remain
the public API URL. Correct these values before caching configuration in the
backend deployment step.

Also record the current release for rollback:

```bash
cd /var/www/pos
git rev-parse HEAD
git -C pos-backend rev-parse HEAD
git -C pos-frontend rev-parse HEAD
```

## 2. Update the merged `main` release

This workspace is a Git superproject with `pos-backend` and `pos-frontend` as
submodules. The root `main` commit selects the backend and frontend revisions,
so update the root repository first and then synchronize submodules.

```bash
cd /var/www/pos
git switch main
git pull --ff-only origin main
git submodule sync --recursive
git submodule update --init --recursive
```

If the VPS was cloned as three independent repositories instead, update each
repository to the revision selected by the merged release. Do not deploy the
feature branch directly after it has been merged; deploy `main`.

## 3. Deploy backend code and database changes

Optionally enable your normal maintenance page before schema changes. Bring it
back only after the verification section succeeds.

```bash
cd /var/www/pos/pos-backend
composer install --no-dev --prefer-dist --optimize-autoloader
php artisan optimize:clear
php artisan migrate --force
php artisan tenants:migrate --force
php artisan tenants:run db:seed --option=class=Database\\Seeders\\RoleSeeder
php artisan config:cache
php artisan route:cache
php artisan view:cache
```

The central migration creates the `tenant_branch_domains` registry. The tenant
migrations add branch-owned tables/data and sync the newer HR and cashier
permissions. The final `RoleSeeder` command is idempotent and makes the role
presets consistent across existing tenant databases.

Restart the process manager actually used on the VPS after deployment. For a
systemd/PHP-FPM setup this is commonly:

```bash
sudo systemctl reload php8.3-fpm
# If queue workers are managed by Supervisor, restart the relevant worker group.
sudo supervisorctl status
```

Do not run `supervisorctl restart all` unless it is already the approved worker
deployment procedure for this VPS.

## 4. One-time bootstrap for each existing tenant

New tenants created after this deployment get website and admin mappings during
provisioning. Existing tenants from the old single-branch version do not. Do
this once per existing company tenant before attempting to use its admin host.

First decide the company hostname. For example:

```text
Company website/POS: kstore.aitechnology.com.np
Company admin portal: admin.kstore.aitechnology.com.np
Existing Main branch: main.kstore.aitechnology.com.np
Future Lalitpur branch: lalitpur.kstore.aitechnology.com.np
```

Obtain the tenant UUID from the central `tenants` table, then use Tinker from
`/var/www/pos/pos-backend`. Replace the UUID and hostname below. This is an
idempotent write: `firstOrCreate` leaves a correct existing mapping unchanged.

```php
use App\Models\Tenant;
use App\Models\TenantBranchDomain;

$tenant = Tenant::findOrFail('<tenant-uuid>');
$companyDomain = 'kstore.aitechnology.com.np';

TenantBranchDomain::firstOrCreate(
    ['domain' => $companyDomain],
    ['tenant_id' => $tenant->id, 'branch_id' => null, 'portal' => 'website'],
);

TenantBranchDomain::firstOrCreate(
    ['domain' => 'admin.'.$companyDomain],
    ['tenant_id' => $tenant->id, 'branch_id' => null, 'portal' => 'admin'],
);
```

Then create mappings for every already-existing branch, including the legacy
`Main` branch:

```bash
cd /var/www/pos/pos-backend
php artisan branches:sync-domains <tenant-uuid>
```

Confirm that the user who will open the admin portal has the `admin` or
`super-admin` role and an assignment to at least one branch. The new app can
then use `https://admin.kstore.aitechnology.com.np/pos/login` to create and
assign further branches/users.

## 5. DNS, TLS, and Nginx for a company with branches

For each company tenant that will use branch hosts, create these DNS records at
your DNS provider:

```text
admin.kstore.aitechnology.com.np   A      <VPS public IP>
*.kstore.aitechnology.com.np       A      <VPS public IP>
```

The second record is important: `*.aitechnology.com.np` does not make DNS for
`lalitpur.kstore.aitechnology.com.np` automatic. Once the per-tenant wildcard
exists, future branch slugs created in the app need no individual DNS record.

Issue/renew a DNS-01 certificate that contains at least:

```text
kstore.aitechnology.com.np
*.kstore.aitechnology.com.np
```

Add a more-specific Nginx HTTPS server for that tenant so it selects the
per-tenant certificate by SNI. It serves the same built frontend as the
existing wildcard block:

```nginx
server {
    listen 80;
    server_name .kstore.aitechnology.com.np;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name .kstore.aitechnology.com.np;

    root /var/www/pos/pos-frontend/dist;
    index index.html;

    ssl_certificate /etc/letsencrypt/live/kstore.aitechnology.com.np/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/kstore.aitechnology.com.np/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location /assets/ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```

Enable the site using your existing Nginx convention, then validate and reload:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

No new API Nginx server is needed. The frontend sends the complete browser host
in `X-Tenant`, and `api.aitechnology.com.np` resolves the tenant/branch from
that header. Current backend CORS permits the new origin. Keep the API host at
`https://api.aitechnology.com.np`.

## 6. Build the production frontend

The frontend environment is compiled into the Vite bundle. Do not accidentally
build the VPS with the repository's local `http://api.localhost:8000` values.
Use the production API URLs when building:

```bash
cd /var/www/pos/pos-frontend
npm ci
VITE_BASE_URL=https://api.aitechnology.com.np/api \
VITE_BASE_URL_WITHOUT_API=https://api.aitechnology.com.np \
npm run build
```

The resulting `dist/` directory is already the Nginx root in your supplied
configuration. No separate frontend deployment command is necessary when the
build is performed on this VPS.

## 7. Verification and release

1. Confirm the existing company site still opens over HTTPS.
2. Sign in at the new admin host with an `admin` or `super-admin` user.
3. Confirm the admin portal shows company-level Customers and reports.
4. Create one test branch from the admin portal. Its application mapping and
   creator assignment should be created automatically.
5. Open the resulting branch hostname over HTTPS, log in as an assigned user,
   and verify it sees only that branch's operational data.
6. Confirm Customers remain visible as the tenant-wide shared list where the
   role has customer access.
7. Run the QA regression only if explicitly requested: copy
   `docs/QA_REGRESSION_TEMPLATE.md` to `docs/qa-runs/QA_RUN_YYYY-MM-DD.md`
   and record the results there.

If maintenance mode was enabled, bring the app back up only after these checks:

```bash
cd /var/www/pos/pos-backend
php artisan up
```

## Rollback boundary

Before running migrations, rollback is a normal code checkout back to the
recorded release and a frontend rebuild. After migrations or branch data writes,
rollback may require the verified database backup. Do not roll back schema or
restore production data casually; first determine which migrations and branch
records were applied.
