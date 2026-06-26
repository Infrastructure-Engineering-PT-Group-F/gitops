# Multi-tenancy security concept

How tenants are separated on the Group F platform. The model is **soft
multi-tenancy**: tenants share one GKE cluster and its nodes and isolation is
enforced per namespace through Kubernetes and GCP controls. This is deliberate
for the assignment scope. It is strong enough to keep tenants apart in normal
operation, without the cost and complexity of hard isolation such as a cluster
or node pool per tenant.

Related docs:

- Tenant model and baseline: [`tenants/README.md`](../../tenants/README.md)
- Catalog and Composition: [`catalog/README.md`](../../catalog/README.md)
- Secret handling: [`secret-handling.md`](secret-handling.md)
- Tenant onboarding: [`docs/tenant-onboarding.md`](../tenant-onboarding.md)
- Operational runbook: [`docs/operations/runbook.md`](../operations/runbook.md)

## Isolation model at a glance

| Dimension          | Boundary                       | Mechanism                                                            |
| ------------------ | ------------------------------ | -------------------------------------------------------------------- |
| Workload placement | Namespace per tenant           | One `tenant-<name>` namespace, all tenant workloads live in it       |
| Network            | Namespace-scoped NetworkPolicy | Default-deny plus explicit allow rules, enforced by GKE Dataplane V2 |
| Compute            | Per-namespace caps             | `ResourceQuota` plus `LimitRange`                                    |
| Data               | Database per tenant            | One private-IP Cloud SQL instance, database and user per tenant      |
| Secrets            | Per-namespace delivery         | ESO writes each tenant's Secrets only into its own namespace         |
| Workload hardening | Pod Security Admission         | `baseline` enforced, `restricted` warned and audited                 |

What this is **not**: tenants share the control plane, nodes, the Crossplane
control plane, the shared Gateway and the platform add-ons. There is no
node-level or kernel-level isolation between tenants. That is the accepted
trade-off of soft multi-tenancy.

## One namespace per tenant

Every tenant gets exactly one namespace, `tenant-<name>` (lower case). The
prefix keeps tenant namespaces clearly separate from platform namespaces such as
`crossplane-system` or `external-secrets`. The namespace is the unit that
carries the NetworkPolicies, the quota and limit range, the runtime Secrets and
the tenant workloads.

### Naming and labels

| Label                                   | Example       | Purpose                                  |
| --------------------------------------- | ------------- | ---------------------------------------- |
| `platform.fh-burgenland.at/tenant`      | `staging`     | Tenant identifier, used by selectors     |
| `platform.fh-burgenland.at/environment` | `validation`  | `staging`, `validation`, or `production` |
| `app.kubernetes.io/part-of`             | `weather-app` | Groups the tenant workloads              |
| `pod-security.kubernetes.io/enforce`    | `baseline`    | Pod Security Admission level enforced    |

## Network policy model

NetworkPolicy is enforced because the cluster runs GKE Dataplane V2
(`ADVANCED_DATAPATH`). Each tenant namespace starts deny-all and opens only the
traffic the weather-app needs:

| Policy                               | Selected pods             | Allowed traffic                                                                       |
| ------------------------------------ | ------------------------- | ------------------------------------------------------------------------------------- |
| `default-deny`                       | All tenant pods           | Nothing, ingress and egress, unless another policy allows it                          |
| `allow-dns-egress`                   | All tenant pods           | DNS to `kube-system` and the Service CIDR on TCP/UDP 53                               |
| `allow-gateway-ingress-to-apps`      | Backend and frontend pods | Ingress from the Envoy Gateway data plane on TCP 8080                                 |
| `allow-backend-required-egress`      | Backend pods              | Cloud SQL on TCP 5432 (the Private Services Access range) and public HTTPS on TCP 443 |
| `allow-monitoring-scrape-to-backend` | Backend pods              | Managed Prometheus collectors in `gmp-system` scraping the internal management port   |

There is no tenant-to-tenant allow rule, so cross-tenant pod traffic stays
denied by `default-deny`. The backend egress rule excludes the RFC1918 ranges,
so the public-HTTPS allowance does not reopen private VPC, pod, Service, or
cross-tenant access. Selectors use the stable Helm chart labels
(`app.kubernetes.io/name=weather-app-backend` and `...-frontend`).

## ResourceQuota and LimitRange model

Every tenant namespace carries a `ResourceQuota` and a `LimitRange`. The quota
caps total namespace consumption so one tenant cannot starve the cluster and the
limit range gives every container default requests and limits so a pod without
explicit values is still admissible under the quota.

The default tier (the `small` class) caps the namespace at `1` CPU / `1Gi`
requests, `3` CPU / `2Gi` limits, `10` pods and a small number of PVCs and
Services. The values are anchored to the real backend and frontend footprints
plus rollout headroom and stay well inside the cluster baseline. The detailed
table and reasoning are in [`tenants/README.md`](../../tenants/README.md).

## Secret isolation

No secret value is stored in Git. Secrets are isolated per namespace:

- A shared `ClusterSecretStore gcp-secret-manager` provides read access to
  Google Secret Manager, but each tenant's `ExternalSecret`s materialize Secrets
  only into that tenant's namespace.
- Runtime Secrets (`api-keys`, `ghcr-pull`, `ghcr-chart-pull`) are delivered
  into each `tenant-<name>` namespace by ESO.
- Database credentials are per tenant. ESO generates a unique password per
  tenant and writes `weather-app-backend-db` into the tenant namespace only. The
  `eso-store-reader` ServiceAccount that reads the connection Secret is
  namespace-scoped.

High-level handling and the audit procedure are in
[`secret-handling.md`](secret-handling.md).

## Database isolation

Each tenant gets its own Cloud SQL for PostgreSQL instance, provisioned by the
`SQLInstance` Composition with a private IP only (no public IP). Each instance
has its own database (`appdb`) and application user (`appdb-user`) with a
per-tenant generated password. Tenants never share a database instance, so there
is no row-level or schema-level cross-tenant access to reason about.

## Staging tenant

`tenant-staging` is a permanent non-production tenant used to validate new
application versions and platform changes before they reach production tenants.
`tenant-validation` is a second non-production tenant used to prove tenant
integration and runtime-secret delivery end to end. Both use the same isolation
model as a production tenant.

## Workload hardening

Tenant namespaces enforce the Pod Security Admission `baseline` level and warn
and audit at `restricted`. `enforce` is `baseline` for now because the
application pods do not yet set a fully restricted security context. It can be
raised to `restricted` once the backend and frontend set restricted-compatible
contexts.

## Known limitations and what is intentionally simple

- **Soft, not hard, multi-tenancy.** Tenants share the cluster, nodes, control
  plane and platform add-ons. There is no per-tenant cluster, node pool, or
  kernel isolation. This is intentional for the assignment scope.
- **RBAC is a type and namespace boundary, not an object-name boundary.** The
  Crossplane provider ServiceAccounts are restricted to specific resource types
  and namespaces, but RBAC cannot restrict access to a specific object name.
- **Pod Security is at `baseline`, not `restricted`,** until the app workloads
  set restricted-compatible security contexts.
- **NetworkPolicies are currently static per tenant** (committed under
  `tenants/<name>/`) rather than rendered by the `XTenant` Composition. The
  staging and validation tenants carry the full model. Generalizing it into the
  Composition is future work.
- **Shared ingress.** All tenants share one Gateway, one public IP and one
  wildcard certificate. Isolation at the edge is by hostname and route, not by a
  dedicated load balancer per tenant.
- **No service mesh or tenant-to-tenant mTLS.** Cross-tenant separation relies
  on NetworkPolicy, not in-cluster encryption.

These simplifications keep the platform understandable and cheap to run while
still demonstrating real tenant separation across network, compute, data and
secrets.

## Validating the boundary

The isolation can be checked on a live tenant (see the
[runbook](../operations/runbook.md) and
[`tenants/README.md`](../../tenants/README.md)):

- Confirm the policies exist: `kubectl get networkpolicy -n tenant-<name>`.
- Positive test: tenant DNS resolution works.
- Negative test: an unlabeled pod, or a pod in another namespace, cannot reach
  the backend Service.
- Confirm the quota and limit range are applied and the workloads stay within
  them: `kubectl get resourcequota,limitrange -n tenant-<name>`.
