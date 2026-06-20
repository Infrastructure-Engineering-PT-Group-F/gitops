# crossplane-providers

This add-on installs the Crossplane providers and ProviderConfigs that the tenant
service catalog uses to render tenant resources.

## Providers

| Provider | Package | Purpose |
|----------|---------|---------|
| `provider-kubernetes` | `xpkg.crossplane.io/crossplane-contrib/provider-kubernetes:v1.2.1` | Renders Kubernetes objects such as tenant namespaces, network policies, resource quotas, limit ranges, and selected secrets. |
| `provider-helm` | `xpkg.crossplane.io/crossplane-contrib/provider-helm:v1.2.0` | Renders tenant backend and frontend Helm releases. |
| `provider-gcp-sql` | `xpkg.upbound.io/upbound/provider-gcp-sql:v2.6.0` | Renders per-tenant Google Cloud SQL resources. |

Only `provider-gcp-sql` is declared explicitly for GCP. Its package dependency
resolution installs the matching `provider-family-gcp` dependency, so this repo
does not define a second owner for that family provider.

## ProviderConfigs

Future Compositions must reference these ProviderConfig names:

| ProviderConfig | API version | Authentication |
|----------------|-------------|----------------|
| `kubernetes-provider` | `kubernetes.crossplane.io/v1alpha1` | `InjectedIdentity` |
| `helm-provider` | `helm.crossplane.io/v1beta1` | `InjectedIdentity` |
| `gcp-provider` | `gcp.upbound.io/v1beta1` | `InjectedIdentity` with `projectID: project-03eb5b2f-b0b9-4171-b14` |

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
crossplane-system/provider-gcp -> crossplane-sa@project-03eb5b2f-b0b9-4171-b14.iam.gserviceaccount.com
```

## Security

No static Google service-account key, kubeconfig, token, or credential secret is
stored in this repository.

## RBAC Boundaries

The Kubernetes providers are intentionally scoped to the tenant resource types
they need.

`provider-kubernetes` can manage only namespaces, resource quotas, limit ranges,
secrets, and network policies.

`provider-helm` can manage only secrets, services, service accounts, deployments,
horizontal pod autoscalers, and ingresses rendered by the tenant Helm charts.

This supports soft multi-tenancy, but it is not a hard object-name boundary:
Kubernetes RBAC can restrict resource types and namespaces, but it cannot restrict
cluster-scoped namespace object names to only `tenant-*`.

## Validation

Cloud SQL smoke testing depends on infrastructure issue
`Infrastructure-Engineering-PT-Group-F/infrastructure#45`.

Manual future live-cluster validation:

1. Confirm the `crossplane-providers` ArgoCD Application is synced.
2. Confirm all provider packages report `Installed=True` and `Healthy=True`.
3. Confirm the three ProviderConfigs exist.
4. Confirm `crossplane-system/provider-gcp` has the Workload Identity annotation.
5. Run temporary smoke resources for provider-kubernetes, provider-helm, and
   provider-gcp-sql, then remove them.

Do not add executable smoke-test manifests under this GitOps deployment path.
