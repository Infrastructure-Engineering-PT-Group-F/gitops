# Tenant onboarding

How a new tenant is created on the Group F platform, what each field means and
what ArgoCD and Crossplane do in the background once the change is merged. This
guide is self-contained so it can be followed during the final demo without
opening other files. For deeper reference it links to the catalog and tenants
READMEs.

- Catalog (the tenant API and Compositions):
  [`catalog/README.md`](../catalog/README.md)
- Tenant directory model and baseline:
  [`tenants/README.md`](../tenants/README.md)
- Database credentials example:
  [`tenants/db-credentials-example.md`](../tenants/db-credentials-example.md)
- Secret handling review:
  [`docs/security/secret-handling.md`](security/secret-handling.md)

## Overview

A tenant is one directory under `tenants/`. Adding `tenants/<name>/` to the
`gitops` repository and merging it is the entire user-facing action. ArgoCD
detects the new directory, applies the tenant manifests and Crossplane
provisions the database and the application releases. No manual `kubectl` or
`gcloud` changes are needed on the cluster.

Each tenant directory holds four manifests:

| File                   | Owner                                         | Purpose                                                                               |
| ---------------------- | --------------------------------------------- | ------------------------------------------------------------------------------------- |
| `namespace.yaml`       | ArgoCD                                        | The `tenant-<name>` namespace with platform labels and Pod Security Admission labels. |
| `runtime-secrets.yaml` | ArgoCD (ESO)                                  | Delivers `api-keys`, `ghcr-pull`, and `ghcr-chart-pull` from Google Secret Manager.   |
| `resource-limits.yaml` | ArgoCD                                        | The `ResourceQuota` and `LimitRange` for the namespace.                               |
| `xtenant.yaml`         | ArgoCD applies it, Crossplane renders from it | The `XTenant` claim that provisions the database, backend, and frontend.              |

The namespace, runtime secrets and resource limits stay ArgoCD-owned in the
current milestone. The `XTenant` claim is the only part Crossplane expands into
further resources.

## The tenant claim (`XTenant`)

`XTenant` (`platform.fh-burgenland.at/v1alpha1`, namespaced) is the tenant
application API. One `XTenant` is committed in the tenant namespace and the
`xtenant-weather-app` Composition renders the rest. The schema lives in
[`catalog/xrds/xtenant.yaml`](../catalog/xrds/xtenant.yaml).

| Field              | Required | Default | Purpose                                                                                                 |
| ------------------ | -------- | ------- | ------------------------------------------------------------------------------------------------------- |
| `name`             | yes      | none    | Short tenant identifier without the `tenant-` prefix. Must match `^[a-z0-9]([-a-z0-9]*[a-z0-9])?$`      |
| `environment`      | yes      | none    | One of `staging`, `validation`, or `production`                                                         |
| `hostname`         | yes      | none    | Public hostname under the platform wildcard domain `*.gcp.ajdininfrastructure.lol`                      |
| `backendImageTag`  | yes      | none    | Backend container image tag passed to the backend chart                                                 |
| `frontendImageTag` | yes      | none    | Frontend container image tag passed to the frontend chart                                               |
| `size`             | no       | `small` | Size class. Only `small` is rendered today. `medium` and `large` are accepted for forward compatibility |

### Minimal example

The onboarding unit is the directory, but the only file a new tenant truly
authors is the claim. This is the worked validation example
([`tenants/validation/xtenant.yaml`](../tenants/validation/xtenant.yaml)):

```yaml
apiVersion: platform.fh-burgenland.at/v1alpha1
kind: XTenant
metadata:
   name: validation
   namespace: tenant-validation
   annotations:
      argocd.argoproj.io/sync-wave: "2"
spec:
   parameters:
      name: validation
      environment: validation
      hostname: validation.gcp.ajdininfrastructure.lol
      backendImageTag: "v1.3.1"
      frontendImageTag: "v1.0.0"
      size: small
```

## Onboarding workflow

1. Copy an existing tenant directory, for example `tenants/validation/`, to
   `tenants/<name>/`. It already contains the namespace baseline, runtime-secret
   delivery, resource limits, and an `XTenant`. Do not add an
   `application.yaml`, the `tenants` ApplicationSet generates the per-tenant
   Application from the directory name.
2. Rename the namespace to `tenant-<name>` in every file, and set the `XTenant`
   `name` and the namespace labels to the same short tenant name. Pick the
   `environment`.
3. Choose backend and frontend image tags and a `hostname` under
   `*.gcp.ajdininfrastructure.lol`.
4. No per-tenant secret seeding is needed. `api-keys`, `ghcr-pull`, and
   `ghcr-chart-pull` resolve from the shared Google Secret Manager values that
   are seeded once at platform bootstrap. See the note in
   [Secrets and database credentials](#secrets-and-database-credentials) about
   re-seeding after a cluster rebuild.
5. Open a pull request following the repository contribution rules.
6. On merge, the ApplicationSet generates the `tenant-<name>` Application,
   ArgoCD applies the directory in order, and Crossplane provisions the database
   and releases. Expect roughly ten minutes while Cloud SQL comes up. The
   Application stays `Progressing` until the database and releases are Ready.

## What happens in the background

### Responsibility split

| Owner           | Provisions                                                                                                                                                                                                                                                                        | Triggered by                                   |
| --------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------- |
| GitOps          | Detects `tenants/<name>/` through the `tenants` ApplicationSet git generator and creates the `tenant-<name>` Application. Applies, in order, the namespace and Pod Security labels, then the runtime secrets and the `ResourceQuota` and `LimitRange`, then the `XTenant` claim.  | A git commit to `tenants/`                     |
| Crossplane      | Expands the `XTenant` into a `SQLInstance`, the database credential helper resources, a backend Helm `Release`, and a frontend Helm `Release`. The `SQLInstance` Composition in turn renders the Cloud SQL `DatabaseInstance`, a `Database`, a `User`, and the connection Secret. | The `XTenant` claim appearing in the namespace |
| Shared platform | Envoy `shared-gateway` in `platform-gateway`, the cert-manager wildcard certificate for `*.gcp.ajdininfrastructure.lol`, external-dns hostname records, the Crossplane providers and their ProviderConfigs, and the `ClusterSecretStore gcp-secret-manager`.                      | Platform bootstrap, not per tenant             |

The `tenants` ApplicationSet uses `preserveResourcesOnDeletion: true`. Removing
a tenant directory deletes only the generated Application, not the namespace,
database, or secrets. Offboarding is therefore an explicit operation, see
[Offboarding](#offboarding).

### Render flow

1. ArgoCD applies the tenant namespace, the runtime-secret delivery and the
   resource limits from `tenants/<name>/`. The runtime secrets `api-keys`,
   `ghcr-pull` and `ghcr-chart-pull` must exist before the Helm releases can
   become Ready.
2. ArgoCD applies the namespaced `XTenant` claim (sync-wave `2`).
3. Crossplane renders the `SQLInstance`. The `SQLInstance` Composition
   provisions a private-IP Cloud SQL `DatabaseInstance`, a `Database` named
   `appdb` and a `User` named `appdb-user`, then writes the non-secret
   connection Secret `weather-app-backend-db-conn` with the instance host, port
   and database name.
4. Crossplane renders the namespace-local ESO helper resources. ESO generates a
   random password, reads the host from the connection Secret and builds the
   `weather-app-backend-db` Secret with `url`, `username` and `password`. The
   `User` reads the same password through `passwordSecretRef`, so the Cloud SQL
   user and the backend share one generated credential.
5. Crossplane renders the backend and frontend Helm releases with chart pull
   credentials from `ghcr-chart-pull`. The backend uses the `external-pg`
   profile, the `api-keys` Secret and the `weather-app-backend-db` Secret and
   attaches an `HTTPRoute` to `shared-gateway` in `platform-gateway` on the
   `https` listener. The frontend uses `ghcr-pull` for its private image and
   points at the tenant backend URL.

### Known routing boundary

The backend chart supports the Gateway API `HTTPRoute`, so the backend is
publicly reachable on the tenant hostname. The frontend chart currently supports
Ingress but not the Gateway API `HTTPRoute`, so the frontend workload and
Service deploy but are not publicly reachable yet. Closing this gap, by adding
`HTTPRoute` support to the frontend chart or another approved route owner, is
tracked in
[frontend #34](https://github.com/Infrastructure-Engineering-PT-Group-F/frontend/issues/34)
and the onboarding implementation issue
[gitops #97](https://github.com/Infrastructure-Engineering-PT-Group-F/gitops/issues/97).

## Secrets and database credentials

No secret value is ever committed to git. Two delivery paths feed a tenant.

- Runtime secrets. `api-keys`, `ghcr-pull` and `ghcr-chart-pull` are delivered
  by the ArgoCD-owned `runtime-secrets.yaml` through ESO, reading the shared
  values from Google Secret Manager through the
  `ClusterSecretStore
  gcp-secret-manager`. `ghcr-pull` carries the
  `kubernetes.io/dockerconfigjson` type expected by image pulls.
- Database credentials. The `XTenant` Composition renders an ESO `Password`
  generator, an `in-cluster` `SecretStore` and the `weather-app-backend-db`
  `ExternalSecret`. ESO combines the generated password with the host from the
  Crossplane-owned `weather-app-backend-db-conn` connection Secret. The full
  reference shape is in
  [`tenants/db-credentials-example.md`](../tenants/db-credentials-example.md).
  The wider review is in
  [`docs/security/secret-handling.md`](security/secret-handling.md).

The Secret Manager containers `avwx-api-key` and `ghcr-pull` survive a cluster
recreate, but their values do not. After a rebuild they are empty containers
with zero versions, so ESO reports
`SecretSyncedError: could not get secret data from provider` and no tenant can
reach Ready until the values are re-seeded. Re-seeding is an operator action
with the real values, not part of the GitOps flow.

## Verifying a tenant

All checks are read-only. Replace `<name>` with the tenant short name.

```sh
# GitOps: the per-tenant Application should be Synced, Healthy once provisioned.
kubectl get applications -n argocd \
  -o custom-columns='NAME:.metadata.name,SYNC:.status.sync.status,HEALTH:.status.health.status'

# Crossplane: the claim and the database composite should reach Ready=True.
kubectl get xtenant,sqlinstance -n tenant-<name>

# Workloads, route, and secrets.
kubectl get pods,httproute -n tenant-<name>
kubectl get externalsecret,secret -n tenant-<name>

# Ground truth for the database in GCP.
gcloud sql instances list --project dark-diagram-496907-k8

# Public reachability of the backend on the tenant hostname.
curl -sS -m 8 -o /dev/null -w "HTTP %{http_code}\n" \
  https://<name>.gcp.ajdininfrastructure.lol/actuator/health
```

A fully onboarded tenant shows the Application `Synced`/`Healthy`, the `XTenant`
and `SQLInstance` `Ready=True`, the backend pod `Running`, the `HTTPRoute`
attached and the health endpoint returning `HTTP 200`.

## Troubleshooting

| Symptom                                                                                                | Likely cause                                                                                                                                                                                                                                          | Check                                                                                                                                                     | Resolution                                                                                                                                                                                                                                                                                                                            |
| ------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `XTenant Ready=False` with reason `Deleting`, hostname returns 404, no pods in the namespace           | The `XTenant` is stuck in `foregroundDeletion`. The GCP provider cannot initialize because a stale `ProviderConfigUsage` from the previous reconcile still exists, so it never confirms the deletion and never drops the managed-resource finalizers. | `kubectl get xtenant <name> -n tenant-<name> -o jsonpath='{.status.conditions}'` and `kubectl get providerconfigusages.gcp.m.upbound.io -n tenant-<name>` | Operator action. Delete the stale `providerconfigusages` for the tenant so the provider can re-create them, or, since the external Cloud SQL is already gone, clear the `finalizer.managedresource.crossplane.io` on the stuck `Database` and `User` managed resources. The `XTenant` then finishes deleting and ArgoCD recreates it. |
| `User` managed resource reports `Error 400: Missing user password for PostgreSQL instance`             | Ordering race. The Cloud SQL `User` reconciled before ESO wrote the `password` key into `weather-app-backend-db`.                                                                                                                                     | `kubectl get secret weather-app-backend-db -n tenant-<name>` and confirm it has a `password` key                                                          | Usually self-heals on the next reconcile once the Secret exists. If it stays wedged, force a reconcile of the `User` resource.                                                                                                                                                                                                        |
| ESO reports `SecretSyncedError: could not get secret data from provider`                               | The Secret Manager values are empty, typically after a cluster rebuild.                                                                                                                                                                               | `gcloud secrets versions list avwx-api-key --project dark-diagram-496907-k8`                                                                              | Operator re-seeds the values, then forces the ExternalSecrets to re-read.                                                                                                                                                                                                                                                             |
| ArgoCD sync fails with `no matches for kind "XTenant"`                                                 | CRD-then-CR race. The `XTenant` CRD from `catalog/` was not `Established` before the claim was applied.                                                                                                                                               | `kubectl get crd xtenants.platform.fh-burgenland.at`                                                                                                      | Sync the tenant Application again once the CRD is `Established`.                                                                                                                                                                                                                                                                      |
| backend pod in `CreateContainerConfigError`                                                            | The `weather-app-backend-db` Secret is not built yet, so the pod cannot mount it.                                                                                                                                                                     | `kubectl get secret -n tenant-<name>`                                                                                                                     | Wait for the `SQLInstance` to reach Ready and ESO to build the Secret. The pod recreates automatically.                                                                                                                                                                                                                               |
| frontend pod in `ImagePullBackOff`                                                                     | The frontend image tag is wrong or the private-image pull is failing.                                                                                                                                                                                 | `kubectl describe pod -l app.kubernetes.io/name=weather-app-frontend -n tenant-<name>`                                                                    | Use a valid `frontendImageTag` and confirm the `ghcr-pull` Secret. Tracked in [frontend #34](https://github.com/Infrastructure-Engineering-PT-Group-F/frontend/issues/34).                                                                                                                                                            |
| `SQLInstance Ready=False` for more than fifteen minutes and no instance in `gcloud sql instances list` | The instance is not creating, it is deleting or wedged. A first build that is genuinely creating shows the instance in GCP within a minute or two.                                                                                                    | Read the condition reason, do not rely on `Ready=False` alone, and treat `gcloud sql instances list` as ground truth                                      | If the reason is `Deleting`, use the stuck-deletion row above. If it is `Creating` and the instance is present in GCP, wait.                                                                                                                                                                                                          |

## Demo script

A five-beat live walkthrough that covers creating a tenant and explaining the
background processes.

1. Show a tenant directory in git (`tenants/validation/`) and the four files.
   Explain that adding such a directory is the entire onboarding action.
2. Show the platform doing the work, the
   [responsibility split](#responsibility-split) table on screen, while running
   `kubectl get applications -n argocd`.
3. Walk the [render flow](#render-flow), then show
   `kubectl get xtenant,sqlinstance,release,pods,httproute -n tenant-validation`.
4. Show the database in GCP with `gcloud sql instances list` to prove the
   private Cloud SQL instance is real.
5. Curl `https://validation.gcp.ajdininfrastructure.lol/actuator/health` and
   show `HTTP 200`. Mention the known frontend routing gap as a documented
   limitation.

For a repeatable second tenant, copy the directory, change `name`, `namespace`,
and `hostname`, and merge. The platform onboards it with no further manual
steps.

## Offboarding

Because `preserveResourcesOnDeletion: true` keeps tenant resources when the
Application is removed, teardown must deprovision through Crossplane first,
while the resources are still managed, and only then remove the git directory.
The full procedure is in
[`tenants/README.md`](../tenants/README.md#remove-a-tenant). In short:

1. `kubectl -n tenant-<name> delete xtenant <name>` so Crossplane deprovisions
   Cloud SQL and the Helm releases.
2. Confirm the composed resources are gone and no Cloud SQL instance is still
   billing with `gcloud sql instances list`.
3. `kubectl delete namespace tenant-<name>` to clear the remaining resources.
4. Remove `tenants/<name>/` from git and open a pull request.
