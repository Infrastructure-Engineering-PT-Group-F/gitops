# Operational runbook

Read-only checks and recovery steps for the Group F platform: how to inspect
GitOps, Crossplane, tenants, certificates, DNS and secrets and how to deal with
the failures that have actually occurred. Diagnostics are safe to run by anyone.
Steps that change cluster or cloud state are marked **operator action** and need
cluster-admin plus the real credentials.

Related docs:

- Tenant onboarding: [`docs/tenant-onboarding.md`](../tenant-onboarding.md)
- Secret handling:
  [`docs/security/secret-handling.md`](../security/secret-handling.md)
- Monitoring approach:
  [`docs/monitoring/basic-monitoring-approach.md`](../monitoring/basic-monitoring-approach.md)
- Tenant model: [`tenants/README.md`](../../tenants/README.md), catalog:
  [`catalog/README.md`](../../catalog/README.md)

## Access

`gcloud` must be on `PATH`. The cluster is zonal and its public endpoint changes
on every rebuild, so re-run `get-credentials` whenever `kubectl` cannot reach
the API or reports a certificate error. This is local-only and read-only to GCP.

```sh
gcloud container clusters get-credentials group-f-platform-gke \
  --zone europe-west1-b --project dark-diagram-496907-k8
kubectl config current-context
```

Key namespaces: `argocd`, `crossplane-system`, `external-secrets`,
`cert-manager`, `external-dns`, `envoy-gateway-system`, `platform-gateway`,
`gmp-system` and one `tenant-<name>` per tenant.

## Quick health readout

One block that shows the whole platform state. Run this first.

```sh
# GitOps: every Application should be Synced/Healthy.
kubectl get applications -n argocd \
  -o custom-columns='NAME:.metadata.name,SYNC:.status.sync.status,HEALTH:.status.health.status'

# Tenant composites and database.
kubectl get xtenant,sqlinstance -A

# Crossplane managed resources across all tenants.
kubectl get managed -A

# A specific tenant.
kubectl get pods,httproute,externalsecret,secret -n tenant-<name>

# Database ground truth in GCP.
gcloud sql instances list --project dark-diagram-496907-k8

# Public reachability.
curl -sS -m 8 -o /dev/null -w "HTTP %{http_code}\n" \
  https://<name>.gcp.ajdininfrastructure.lol/api/metar/LOWW
```

## GitOps (ArgoCD)

```sh
kubectl get applications -n argocd \
  -o custom-columns='NAME:.metadata.name,SYNC:.status.sync.status,HEALTH:.status.health.status'

# Why is an Application not Synced/Healthy?
kubectl get application <app> -n argocd \
  -o jsonpath='{range .status.conditions[*]}{.type}{": "}{.message}{"\n"}{end}'
kubectl get application <app> -n argocd -o jsonpath='{.status.operationState.message}{"\n"}'
```

ArgoCD does not auto-retry a sync that has already failed. After fixing the
cause, re-trigger it (**operator action**):

```sh
argocd app sync <app>
# or, without the argocd CLI:
kubectl -n argocd annotate application <app> argocd.argoproj.io/refresh=hard --overwrite
```

`Progressing` on `root` or a tenant Application is normal while a tenant is
still provisioning, the health checks surface `XTenant Ready=False` until the
workloads are up.

## Crossplane (claims, composites, managed resources)

```sh
# Tenant composites. Watch the READY column and the condition reason.
kubectl get xtenant,sqlinstance,release -n tenant-<name>
kubectl get xtenant <name> -n tenant-<name> \
  -o jsonpath='{range .status.conditions[*]}{.type}{"="}{.status}{" ("}{.reason}{")\n"}{end}'

# All managed resources (Cloud SQL instance, database, user, Objects).
kubectl get managed -A

# Providers must be INSTALLED and HEALTHY.
kubectl get providers.pkg.crossplane.io
kubectl get pods -n crossplane-system
```

Read the condition **reason**, not just `Ready=False`. `Creating` means it is
still provisioning (Cloud SQL takes about ten minutes). `Deleting` means it is
being torn down. Confirm against `gcloud sql instances list`, which is the
ground truth for the database.

## Tenant namespaces

```sh
kubectl get ns -l platform.fh-burgenland.at/tenant
kubectl get ns tenant-<name> --show-labels
kubectl get resourcequota,limitrange,networkpolicy -n tenant-<name>
```

## Workloads, services, routes

```sh
kubectl get pods,svc,httproute -n tenant-<name>
# Inspect a failing pod (image, secret, probe errors).
kubectl describe pod -l app.kubernetes.io/name=weather-app-backend -n tenant-<name>
kubectl logs -l app.kubernetes.io/name=weather-app-backend -n tenant-<name> --tail=50
# The shared Gateway and its listeners.
kubectl get gateway -n platform-gateway
```

## TLS (cert-manager)

The platform uses one wildcard certificate `wildcard-gcp` for
`*.gcp.ajdininfrastructure.lol`, issued by `letsencrypt-prod-dns01` via DNS-01.

```sh
kubectl get certificate,certificaterequest -A
kubectl get clusterissuer
# If a certificate is not Ready, walk the chain.
kubectl get order,challenge -A
kubectl describe certificate wildcard-gcp -n platform-gateway
```

## DNS (external-dns)

```sh
# external-dns manages records from HTTPRoute/Gateway hostnames.
kubectl logs -n external-dns deploy/external-dns --tail=50
# Get the current Gateway IP. It can change on a Terraform reapply, so read it
# live rather than assuming a fixed value. From the cluster:
kubectl get gateway shared-gateway -n platform-gateway \
  -o jsonpath='{.status.addresses[*].value}{"\n"}'
# or from the IaC source of truth: terraform output in infrastructure/platform.
# Then confirm the tenant hostname resolves to that IP:
dig +short <name>.gcp.ajdininfrastructure.lol
```

## Secrets (External Secrets Operator)

```sh
kubectl get clustersecretstore gcp-secret-manager
kubectl get externalsecret -A
kubectl get externalsecret,secret -n tenant-<name>
# Why is an ExternalSecret not SecretSynced?
kubectl get externalsecret <name> -n tenant-<name> \
  -o jsonpath='{range .status.conditions[*]}{.type}{"="}{.status}{" "}{.message}{"\n"}{end}'
```

Do not print Secret values during diagnostics. Check names, types readiness
only.

## Common failures and fixes

| Symptom                                                                 | Cause                                                                                                                                          | Check                                                                                  | Fix                                                                                                                                                                                          |
| ----------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `XTenant Ready=False` reason `Deleting`, hostname down, no pods         | Stuck `foregroundDeletion`, the GCP provider cannot init because a stale `ProviderConfigUsage` already exists, so it never confirms the delete | `kubectl get providerconfigusages.gcp.m.upbound.io -n tenant-<name>`                   | **operator action**: delete the stale `providerconfigusages`, or clear `finalizer.managedresource.crossplane.io` on the stuck `Database`/`User` MRs (the external Cloud SQL is already gone) |
| `User` MR `Error 400: Missing user password`                            | Ordering race, the Cloud SQL `User` reconciled before ESO wrote `weather-app-backend-db.password`                                              | `kubectl get secret weather-app-backend-db -n tenant-<name>`                           | Usually self-heals on retry once the Secret exists, else force a reconcile of the `User`                                                                                                     |
| ESO `SecretSyncedError: could not get secret data from provider`        | Secret Manager values empty, typically after a cluster rebuild                                                                                 | `gcloud secrets versions list avwx-api-key --project dark-diagram-496907-k8`           | **operator action**: re-seed the values (see recovery below), then force the ExternalSecrets to re-read                                                                                      |
| ArgoCD sync fails `no matches for kind "XTenant"`                       | CRD-then-CR race, the `XTenant` CRD is not `Established` before the claim is applied                                                           | `kubectl get crd xtenants.platform.fh-burgenland.at`                                   | **operator action**: re-sync the tenant Application once the CRD is `Established`                                                                                                            |
| backend pod `CreateContainerConfigError`                                | `weather-app-backend-db` Secret not built yet                                                                                                  | `kubectl get secret -n tenant-<name>`                                                  | Wait for `SQLInstance` Ready and ESO to build it, the pod recreates automatically                                                                                                            |
| frontend pod `ImagePullBackOff`                                         | Wrong image tag or private GHCR pull failing                                                                                                   | `kubectl describe pod -l app.kubernetes.io/name=weather-app-frontend -n tenant-<name>` | Use a valid `frontendImageTag` and confirm the `ghcr-pull` Secret. Tracked in `frontend#34`                                                                                                  |
| `SQLInstance Ready=False` over fifteen minutes, no instance in `gcloud` | Not creating, it is deleting or wedged                                                                                                         | `gcloud sql instances list`                                                            | Read the condition reason, treat `gcloud` as ground truth, then use the stuck-deletion row above                                                                                             |
| Hostname returns `404`                                                  | No `HTTPRoute` matches the path, or the route is not attached                                                                                  | `kubectl get httproute -n tenant-<name>`                                               | Confirm the route exists and its `parentRef` points at `shared-gateway` in `platform-gateway`                                                                                                |
| Hostname returns `503`                                                  | Route and TLS work but no healthy backend pod                                                                                                  | `kubectl get pods -n tenant-<name>`                                                    | Bring the target workload to Ready, see the pod rows above                                                                                                                                   |

## Recovery after a cluster rebuild

A cluster recreate wipes in-cluster state. ArgoCD and Secret Manager are
declarative, so most things re-bootstrap, but these manual pieces must be redone
(**operator actions**):

1. Refresh local credentials, the endpoint and CA change on rebuild.

   ```sh
   gcloud container clusters get-credentials group-f-platform-gke \
     --zone europe-west1-b --project dark-diagram-496907-k8
   ```

2. Re-seed Secret Manager. The containers `avwx-api-key` and `ghcr-pull` survive
   but lose their versions, so ESO fails until the values are added back.

   ```sh
   printf 'Token %s' '<AVWX_KEY>' | gcloud secrets versions add avwx-api-key \
     --project dark-diagram-496907-k8 --data-file=-
   printf '%s' '<FRESH_GHCR_PAT>' | gcloud secrets versions add ghcr-pull \
     --project dark-diagram-496907-k8 --data-file=-
   kubectl annotate externalsecret api-keys ghcr-pull ghcr-chart-pull \
     -n tenant-<name> force-sync="$(date +%s)" --overwrite
   ```

3. Restore runtime IAM by running `terraform apply` on the infrastructure
   `bootstrap/` stage, which manages the node and platform service-account
   grants.

4. Wait for ArgoCD to reconcile, then run the quick health readout above. If the
   `tenants` sync failed on the CRD-then-CR race, re-sync it once the `XTenant`
   CRD is `Established`.
