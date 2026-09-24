Local setup is now ready for this flow.

1. Create tenant with name Caliber through the central tenant API/UI.

This automatically creates:

- caliber.localhost — website
- admin.caliber.localhost — Caliber company admin portal
- main.caliber.localhost — default Main branch

The tenant email you provide becomes the initial Caliber admin account. Its initial password is password.

2. Open the company admin portal:

http://admin.caliber.localhost:3000/login

Log in using the tenant email and password.

3. Create store branches in:

Company → Branches → Add New

For example, create Pulchok. It will automatically:

- create branch code PULCHOK
- assign your Caliber admin user to that branch
- create pulchok.caliber.localhost

4. Open the branch POS:

http://pulchok.caliber.localhost:3000/login

Use the same initial admin credentials for local testing.

If those URLs do not resolve on your machine, add them to /etc/hosts:

127.0.0.1 admin.caliber.localhost main.caliber.localhost pulchok.caliber.localhost baneshwor.caliber.localhost
koteshwor.caliber.localhost

The tenant creation now provisions the admin domain, initial Main branch, admin role, and branch-domain mappings
automatically. Branches created later from the Caliber admin portal also create their own subdomain mapping automatically. I
also verified the backend tests and frontend production build successfully.
