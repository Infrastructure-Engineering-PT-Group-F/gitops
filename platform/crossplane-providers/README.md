# crossplane-providers

This add-on installs the Crossplane providers and ProviderConfigs that the tenant
service catalog uses to render tenant resources.

## Providers

| Provider | Package | Purpose |
|----------|---------|---------|
| `provider-kubernetes` | `xpkg.crossplane.io/crossplane-contrib/provider-kubernetes:v1.2.1` | Renders Kubernetes objects such as tenant namespaces, network policies, resource quotas, limit ranges, selected secrets, and future tenant DB credential helper resources. |
| `provider-helm` | `xpkg.crossplane.io/crossplane-contrib/provider-helm:v1.2.0` | Renders tenant backend and frontend Helm releases. |
| `provider-gcp-sql` | `xpkg.upbound.io/upbound/provider-gcp-sql:v2.6.0` | Renders per-tenant Google Cloud SQL resources. |

Only `provider-gcp-sql` is declared explicitly for GCP. Its package dependency
resolution installs the matching `provider-family-gcp` dependency, so this repo
does not define a second owner for that family provider.

## Composition Function

The Crossplane Function resource is named `function-patch-and-transform` and is
pinned to
`xpkg.crossplane.io/crossplane-contrib/function-patch-and-transform:v0.8.2`.
Crossplane pipeline Compositions use this function for patch-and-transform
rendering.

## ProviderConfigs

Future namespaced XTenant Compositions must reference only the v2 `.m`
ProviderConfigs and ClusterProviderConfigs:

| Config name | Kind / scope | API version | Authentication |
|-------------|--------------|-------------|----------------|
| `kubernetes-provider` | namespaced `ProviderConfig` in `default` | `kubernetes.m.crossplane.io/v1alpha1` | `InjectedIdentity` |
| `helm-provider` | namespaced `ProviderConfig` in `default` | `helm.m.crossplane.io/v1beta1` | `InjectedIdentity` |
| `helm-provider` | `ClusterProviderConfig` | `helm.m.crossplane.io/v1beta1` | `InjectedIdentity` |
| `default` | `ClusterProviderConfig` | `gcp.m.upbound.io/v1beta1` | `InjectedIdentity` with `projectID: dark-diagram-496907-k8` |

Future namespaced tenant Helm `Release` resources must use this
`providerConfigRef` contract:

```yaml
providerConfigRef:
  kind: ClusterProviderConfig
  name: helm-provider
```

The legacy `kubernetes.crossplane.io/v1alpha1` and
`helm.crossplane.io/v1beta1` ProviderConfigs remain managed temporarily for
compatibility, but new Crossplane v2 tenant Compositions must not reference
them.

The existing `default/helm-provider` namespaced ProviderConfig remains unchanged
for compatibility and must not be referenced by tenant Releases.

The GCP `ClusterProviderConfig/default` explicitly declares an empty
`impersonateServiceAccount.name` to match the observed provider-defaulted live
state and prevent perpetual ArgoCD OutOfSync drift while
`credentials.source` remains `InjectedIdentity`.

## Identity

Each provider runs with a fixed Kubernetes ServiceAccount in
`crossplane-system`:

| ServiceAccount | Used by |
|----------------|---------|
| `provider-kubernetes` | `provider-kubernetes` |
| `provider-helm` | `provider-helm` |
| `provider-gcp` | `provider-gcp-sql` |

`provider-kubernetes` and `provider-helm` use `InjectedIdentity` to act on the
local cluster with their Kubernetes ServiceAccounts.

`provider-gcp-sql` also uses `InjectedIdentity`, but its pod identity is mapped
to the Google service account through GKE Workload Identity:

```text
crossplane-system/provider-gcp -> crossplane-sa@dark-diagram-496907-k8.iam.gserviceaccount.com
```

## Security

No static Google service-account key, kubeconfig, token, or credential secret is
stored in this repository.

## RBAC Boundaries

The Kubernetes providers are intentionally scoped to the tenant resource types
they need.

`provider-kubernetes` can manage only namespaces, resource quotas, limit ranges,
secrets, service accounts, namespace-local RBAC, ESO SecretStores,
ExternalSecrets, password generators, network policies, and Gateway API
HTTPRoutes for tenant routing. The ESO and RBAC permissions exist solely for
namespaced tenant DB credential helper resources.

`provider-helm` can manage only secrets, services, service accounts, deployments,
observe ReplicaSets for Helm release readiness, horizontal pod autoscalers,
ingresses, Gateway API HTTPRoutes, and Prometheus Operator ServiceMonitors
rendered by the tenant Helm charts.

This supports soft multi-tenancy, but it is not a hard object-name boundary:
Kubernetes RBAC can restrict resource types and namespaces, but it cannot restrict
cluster-scoped namespace object names to only `tenant-*`.

## Validation

Cloud SQL smoke testing depends on the required platform infrastructure
prerequisites being in place.

Manual future live-cluster validation:

1. Confirm the `crossplane-providers` ArgoCD Application is synced.
2. Confirm all provider packages report `Installed=True` and `Healthy=True`.
3. Confirm the legacy ProviderConfigs, the v2 namespaced ProviderConfigs, and
   the GCP ClusterProviderConfig exist.
4. Confirm `crossplane-system/provider-gcp` has the Workload Identity annotation.
5. Run temporary smoke resources for provider-kubernetes, provider-helm, and
   provider-gcp-sql, then remove them.

Do not add executable smoke-test manifests under this GitOps deployment path.
