# tenants

Per-tenant resources for the multi-tenant platform. One subfolder per tenant.

In the target model each tenant is a **Composite Resource (XR)** against the
`catalog/` abstraction, and the catalog Composition (Crossplane) provisions the
tenant namespace, network policies, resource quotas, and the backend and
frontend Helm releases. Until that Composition lands, the namespace baseline
below is applied directly by ArgoCD, so the staging tenant is reproducible
either way.

- `staging/`: permanent staging tenant. Validate new app versions here before
  promoting to production tenants.
- `validation/`: non-production validation tenant for runtime-secret delivery
  and tenant integration checks.
- `<tenant>/`: one folder per production tenant.

`application.yaml` is the ArgoCD child Application for this directory. The
Terraform-owned root App-of-Apps discovers it and this child Application
reconciles tenant manifests from `tenants/` recursively, including the current
`staging/namespace.yaml` baseline.

## Namespace baseline

Every tenant gets exactly one namespace. It is the foundation for the tenant
network policies and the resource quotas and limit ranges and it implements the
multi-tenancy security concept.

### Naming

`tenant-<name>`, lower case. The prefix keeps tenant namespaces separate from
the platform namespaces such as `crossplane-system` or `external-secrets`.
Examples: `tenant-staging`, `tenant-acme`.

### Labels

| Label                                   | Example       | Purpose                                                   |
| --------------------------------------- | ------------- | --------------------------------------------------------- |
| `platform.fh-burgenland.at/tenant`      | `staging`     | Tenant identifier, used by network policies and selectors |
| `platform.fh-burgenland.at/environment` | `staging`     | `staging`, `validation`, or `production`                  |
| `app.kubernetes.io/part-of`             | `weather-app` | Groups the tenant workloads under the application         |
| `pod-security.kubernetes.io/enforce`    | `baseline`    | Pod Security Admission level enforced in the namespace    |
| `pod-security.kubernetes.io/warn`       | `restricted`  | Warns on workloads that are not yet restricted            |
| `pod-security.kubernetes.io/audit`      | `restricted`  | Audits the same                                           |

`enforce` is `baseline` for now because the app workloads do not yet set a full
restricted security context. It can be raised to `restricted` once the backend
and frontend pods set a restricted-compatible security context.

`validation` is a deliberate non-production environment for tenant integration
and runtime-secret delivery checks. Tenant runtime-secret delivery selectors
must include it when validation namespaces participate in that flow.

### Annotations

| Annotation                              | Purpose                              |
| --------------------------------------- | ------------------------------------ |
| `platform.fh-burgenland.at/owner`       | Owning team, for operational clarity |
| `platform.fh-burgenland.at/description` | What the tenant is for               |

### How namespaces are created

In the target model Crossplane creates the namespace through the catalog
Composition, so a single tenant resource provisions the namespace with this
exact shape. `staging/namespace.yaml` is the canonical reference for that shape
and doubles as the directly applied staging namespace until the Composition is
in place. ArgoCD applies manifests under `tenants/` through the `tenants` child
Application in this directory.

## Resource quotas and limit ranges

Every tenant namespace carries a `ResourceQuota` and a `LimitRange`
(`<tenant>/resource-limits.yaml`). They are applied at sync-wave `1`, after the
namespace. The quota caps total namespace consumption so a single tenant cannot
starve the cluster, and the limit range gives every container default requests
and limits. The two work together: once a `ResourceQuota` constrains
`requests`/`limits`, a pod without explicit values would be rejected, so the
`LimitRange` defaults keep workloads admissible.

### Default tier

| Resource                           | Value       | Reasoning                                                                       |
| ---------------------------------- | ----------- | ------------------------------------------------------------------------------- |
| `requests.cpu` / `requests.memory` | `1` / `1Gi` | Real backend and frontend need ~`275m`/`416Mi`; the rest is rollout-surge headroom |
| `limits.cpu` / `limits.memory`     | `3` / `2Gi` | Burst ceiling (not reserved); the `3` CPU leaves room for a rolling update that briefly doubles the backend |
| `pods`                             | `10`        | Backend, frontend, a rollout surge pod, and a migration pod                         |
| `persistentvolumeclaims`           | `2`         | The database is external Cloud SQL, so PVCs are spare capacity only                 |
| `services`                         | `5`         | Backend, frontend, and headroom                                                     |

LimitRange container defaults: `50m`/`64Mi` requests and `250m`/`256Mi` limits,
with a `1` CPU / `1Gi` per-container max and a `10m` / `16Mi` min. The min sits
below the frontend's `25m`/`32Mi` request so the light pod is still admitted, and
the max accommodates the backend's `1` CPU / `512Mi` limit.

The values are anchored to the deployed cluster baseline of three
`n2-standard-2` worker nodes (6 vCPU / 24 GiB total, autoscaling 1 to 5
nodes, per `infrastructure/platform`) and the real backend (`250m`/`384Mi`
requests, `1`/`512Mi` limits) and frontend (`25m`/`32Mi` requests,
`100m`/`64Mi` limits) chart resources. The earlier capacity estimate in
infrastructure #14 assumed a smaller two-node `e2-standard-2` baseline. The
default tier stays well inside either, because one tenant reserves at most `1`
CPU / `1Gi` and bursts to `3` CPU / `2Gi`. The database tier runs on Cloud
SQL, outside the namespace, so the quota only sizes the in-cluster workloads. In
the target model the catalog Composition renders these per tenant, selected by
the XTenant `size` field, so this default tier becomes the `small` class and
larger classes scale the same fields up.

## Add a tenant

1. Copy `staging/` to `tenants/<name>/`, rename the namespace to
   `tenant-<name>`, and set the `tenant` and `environment` labels.
2. Open a PR following the repository contribution rules.
3. On merge, ArgoCD applies the resources and, once the catalog Composition is
   in place, Crossplane provisions the namespace, database, and app instance.
