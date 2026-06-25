# namespaces

Pre-creates the namespaces for add-ons whose charts ship **namespaced RBAC** but
**cannot create their own namespace** (no `extraObjects` hook).

## Why

On a fresh cluster, ArgoCD reconciles a chart's namespaced RBAC
(`Role`/`RoleBinding`) with `kubectl auth reconcile`, which requires the target
namespace to already exist. With only `CreateNamespace=true`, namespace creation
races that RBAC step on the first sync and can lose — the whole Application sync
then fails (`namespaces "<ns>" not found ... error running kubectl auth
reconcile`) and loops.

cert-manager and external-secrets solve this **inline**: their charts support
`extraObjects`, so they render their own `Namespace` at sync-wave `-20` within the
same app (reliable by in-app wave ordering, and the namespace stays owned by that
app). Verified with `helm template`: the `crossplane` and `gateway-helm`
(envoy-gateway) charts do **not** honor `extraObjects`, so they cannot self-create
and are handled here instead.

## Namespaces

| Namespace | Add-on | Confirmed namespaced RBAC |
|-----------|--------|---------------------------|
| `envoy-gateway-system` | envoy-gateway | yes (3 Role + 3 RoleBinding) |
| `crossplane-system` | crossplane, crossplane-providers | precautionary |
| `external-dns` | external-dns | precautionary |

Not here: `cert-manager`, `external-secrets` (self-create via `extraObjects`);
`platform-gateway` (gateway add-on ships no namespaced RBAC, so
`CreateNamespace=true` is race-free).

These namespaces were previously created only by `CreateNamespace=true` (never a
tracked manifest), so adopting them in this app is **prune-safe** — no app owned
them before.

## Ordering dependency (important)

This Application runs at sync-wave `-30`, but in app-of-apps the parent only
*orders the creation of the child Application CR* — it does not wait for this app
to actually create the Namespaces unless ArgoCD **Application health** is
restored. That customization
(`resource.customizations.health.argoproj.io_Application`) is configured in the
infrastructure ArgoCD bootstrap (`infrastructure/platform/argocd.tf`); without it,
the wave `-30` ordering is not a reliable guarantee.
