# tenants

Per-tenant **claims** against the `catalog/` abstraction. One subfolder
per tenant. 

- `staging/` — permanent staging tenant. Validate new app versions here before
  promoting to production tenants.
- `<tenant>/` — one folder per production tenant.

## Add a tenant

1. Copy `staging/` to `tenants/<name>/` and set the tenant name/parameters.
2. Open a PR (Conventional Commit, reference the issue).
3. On merge, ArgoCD + Crossplane provision the namespace, database, and app
   instance automatically.
