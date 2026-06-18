# tenants

Per-tenant resources for the multi-tenant platform. One subfolder per tenant.

In the target model each tenant is a **Composite Resource (XR)** against the
`catalog/` abstraction, and the catalog Composition (Crossplane) provisions the
tenant namespace, network policies, resource quotas, and the backend and frontend
Helm releases. Until that Composition lands, the namespace baseline below is
applied directly by ArgoCD, so the staging tenant is reproducible either way.

- `staging/`: permanent staging tenant. Validate new app versions here before
  promoting to production tenants.
- `<tenant>/`: one folder per production tenant.

## Namespace baseline

Every tenant gets exactly one namespace. It is the foundation for the network
policies (#6) and the resource quotas and limit ranges (#7), and it implements the
multi-tenancy security concept (#4).

### Naming

`tenant-<name>`, lower case. The prefix keeps tenant namespaces separate from the
platform namespaces such as `crossplane-system` or `external-secrets`. Examples:
`tenant-staging`, `tenant-acme`.

### Labels

| Label | Example | Purpose |
|-------|---------|---------|
| `platform.fh-burgenland.at/tenant` | `staging` | Tenant identifier, used by network policies and selectors |
| `platform.fh-burgenland.at/environment` | `staging` | `staging` or `production` |
| `app.kubernetes.io/part-of` | `weather-app` | Groups the tenant workloads under the application |
| `pod-security.kubernetes.io/enforce` | `baseline` | Pod Security Admission level enforced in the namespace |
| `pod-security.kubernetes.io/warn` | `restricted` | Warns on workloads that are not yet restricted |
| `pod-security.kubernetes.io/audit` | `restricted` | Audits the same |

`enforce` is `baseline` for now because the app workloads do not yet set a full
restricted security context. It can be raised to `restricted` once the backend and
frontend pods comply (backend#11).

### Annotations

| Annotation | Purpose |
|------------|---------|
| `platform.fh-burgenland.at/owner` | Owning team, for operational clarity |
| `platform.fh-burgenland.at/description` | What the tenant is for |

### How namespaces are created

In the target model Crossplane creates the namespace through the catalog
Composition, so a single tenant resource provisions the namespace with this exact
shape. `staging/namespace.yaml` is the canonical reference for that shape and
doubles as the directly applied staging namespace until the Composition is in
place. For ArgoCD to apply manifests under `tenants/`, the Terraform-owned root
App-of-Apps must include this directory (infrastructure#41).

## Add a tenant

1. Copy `staging/` to `tenants/<name>/`, rename the namespace to `tenant-<name>`,
   and set the `tenant` and `environment` labels.
2. Open a PR (Conventional Commit, reference the issue).
3. On merge, ArgoCD applies the resources and, once the catalog Composition is in
   place, Crossplane provisions the namespace, database, and app instance.
