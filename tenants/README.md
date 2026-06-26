# tenants

Per-tenant resources for the multi-tenant platform. One subfolder per tenant.

In the current model each tenant has an ArgoCD-owned namespace baseline and a
namespaced **Composite Resource (XR)** against the `catalog/` abstraction. The
first `XTenant` Composition renders the tenant database, database credential
helper resources, and backend/frontend Helm releases. Resource limits, and any
tenant NetworkPolicies that are in scope for that tenant, remain explicit
ArgoCD-owned manifests for now.

- `staging/`: permanent staging tenant. Validate new app versions here before
  promoting to production tenants.
- `validation/`: non-production validation tenant for runtime-secret delivery
  and tenant integration checks.
- `<tenant>/`: one folder per production tenant.

`application.yaml` is the ArgoCD child Application for this directory. The
Terraform-owned root App-of-Apps discovers it and this child Application
reconciles tenant manifests from `tenants/` recursively, including namespace
baselines and namespaced `XTenant` resources.

## XTenant application stack

`XTenant` is the tenant application resource. It is namespaced because this repo
uses Crossplane v2, where the onboarding unit is a tenant directory containing
the namespace baseline plus the namespaced XR.

The current fields are:

| Field | Purpose |
| ----- | ------- |
| `name` | Short tenant identifier, without the `tenant-` namespace prefix. |
| `environment` | `staging`, `validation`, or `production`. |
| `hostname` | Public tenant hostname under the platform wildcard domain. |
| `backendImageTag` | Backend container image tag. |
| `frontendImageTag` | Frontend container image tag. |
| `size` | Future size class. `small` is accepted and documented, but quotas still come from static manifests. |

The worked validation example is `validation/xtenant.yaml`. It renders:

- `SQLInstance/weather-app-db`
- ESO helper resources for `weather-app-backend-db`
- `Release/weather-app-backend`
- `Release/weather-app-frontend`

The backend release uses `external-pg`, `api-keys`, `weather-app-backend-db`,
and an `HTTPRoute` attached to `shared-gateway` in `platform-gateway`. The
frontend release uses the tenant `ghcr-pull` image-pull Secret and points at
the tenant backend URL, but the current frontend chart has no Gateway API
`HTTPRoute` support. Public frontend reachability therefore needs a frontend
chart follow-up or another approved route owner.

`api-keys`, `ghcr-pull`, and `ghcr-chart-pull` are prerequisites for the Helm
releases. They are delivered by the ArgoCD-owned runtime-secret manifests, not
by the `XTenant` Composition, so each tenant directory must include that secret
delivery before its `XTenant` can reconcile to Ready.

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

Namespaces are ArgoCD-owned in the first `XTenant` milestone. This avoids two
controllers owning the same namespace and keeps the baseline visible in each
tenant directory. `staging/namespace.yaml` remains the canonical reference for
that shape, and `validation/namespace.yaml` is the first tenant namespace used
with an `XTenant`.

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

## Network policies

Every tenant namespace should carry a deny-by-default `NetworkPolicy` baseline
plus explicit allow policies for the tenant traffic model. The current staging
reference is `staging/network-policies.yaml`; future production tenants and the
Crossplane tenant Composition should render the same model with tenant-specific
namespaces and selectors.

The staging policy model is:

| Policy | Selected Pods | Allowed traffic |
| ------ | ------------- | --------------- |
| `default-deny` | All tenant Pods | No ingress or egress unless another policy allows it |
| `allow-dns-egress` | All tenant Pods | DNS to kube-system and the GKE Service CIDR on TCP/UDP 53 |
| `allow-gateway-ingress-to-apps` | Backend and frontend Pods | Envoy Gateway data-plane ingress on TCP 8080 |
| `allow-backend-required-egress` | Backend Pods | Cloud SQL on TCP 5432 and public HTTPS on TCP 443 |

Selectors intentionally use stable Helm chart labels inside the tenant
namespace: `app.kubernetes.io/name=weather-app-backend` and
`app.kubernetes.io/name=weather-app-frontend`. They do not depend on the
frontend chart carrying a tenant label, because the current frontend chart does
not add one to Pods.

The frontend serves static assets. Browser clients call the backend through the
public Gateway route, so there is no frontend-Pod-to-backend-Pod allow rule.
Backend ingress is allowed from the Envoy Gateway data plane only.

Cloud SQL uses private IP only. Infrastructure reserves `10.30.0.0/16` for
Google Private Services Access, and the backend egress policy limits PostgreSQL
traffic to that range on TCP 5432. The backend also needs public HTTPS egress for
the external weather/METAR API integration. That rule excludes RFC1918 ranges so
it does not reopen private VPC, Pod, Service, or cross-tenant access.

The policies express the desired GitOps model, but Kubernetes only enforces
`NetworkPolicy` when the cluster dataplane supports it. The current
infrastructure code does not explicitly enable the GKE Network Policy add-on or
Dataplane V2, so cross-tenant blocking must be treated as unproven until the
negative validation test below succeeds on the live cluster. If enforcement is
not active, close the gap in the infrastructure repository before relying on
these policies as a security boundary.

### Validation

Run these checks after ArgoCD reconciles `tenants/staging/network-policies.yaml`:

```powershell
kubectl get networkpolicy -n tenant-staging
kubectl describe networkpolicy -n tenant-staging
kubectl get pods -n tenant-staging --show-labels
kubectl get pods -n envoy-gateway-system --show-labels
kubectl get svc -n kube-system
```

Then validate allowed and denied flows with temporary test Pods or existing app
Pods:

```powershell
# Allowed: tenant DNS.
kubectl -n tenant-staging run np-dns-test --rm -it --restart=Never `
  --image=busybox:1.36 -- nslookup kubernetes.default.svc.cluster.local

# Denied: frontend Pods should not call the backend Service directly.
kubectl -n tenant-staging run np-frontend-test --rm -it --restart=Never `
  --image=curlimages/curl:8.8.0 `
  --labels=app.kubernetes.io/name=weather-app-frontend `
  -- curl -m 5 -fsS http://weather-app-backend:8080/actuator/health/readiness

# Denied: an unlabeled tenant Pod should not reach the backend.
kubectl -n tenant-staging run np-deny-test --rm -it --restart=Never `
  --image=curlimages/curl:8.8.0 `
  -- curl -m 5 -fsS http://weather-app-backend:8080/actuator/health/readiness
```

For database validation, verify that the backend remains healthy after
`weather-app-backend-db` is populated and the SQLInstance has a private host.
Do not print Secret data during validation.

## Add a tenant

1. Create `tenants/<name>/` with a namespace baseline, runtime-secret delivery,
   resource limits, and an `XTenant` resource.
2. Set the namespace and `XTenant` tenant labels/fields to the same short
   tenant name.
3. Confirm runtime-secret delivery creates `api-keys`, `ghcr-pull`, and
   `ghcr-chart-pull` before the `XTenant` is expected to become Ready.
4. Choose image tags and a hostname under `*.gcp.ajdininfrastructure.lol`.
5. Open a PR following the repository contribution rules.
6. On merge, ArgoCD applies the tenant manifests and Crossplane provisions the
   database and app releases.
