# gitops

GitOps configuration for the FH Burgenland Group F multi-tenant SaaS platform.
The cluster is reconciled by ArgoCD: everything in this repo is applied to the
cluster by ArgoCD, not by hand.

## Layout

| Path | Purpose |
|------|---------|
| `bootstrap/` | Platform add-on App-of-Apps manifests installed by Terraform after ArgoCD is bootstrapped. |
| `platform/` | Cluster platform add-ons as ArgoCD `Application`s (Helm): ingress-nginx, cert-manager, External Secrets, ExternalDNS, Crossplane. |
| `catalog/` | Crossplane service catalog - XRDs + Compositions that define what a "tenant" is. |
| `tenants/` | Per-tenant Composite Resources (XRs) |

## Adding a tenant

Create a tenant resource under `tenants/<name>/` (copy `tenants/staging/` as a
template) and open a PR. Once merged, ArgoCD + Crossplane provision the tenant
automatically. See `tenants/README.md`.

## Root Application Ownership

The bootstrap root App-of-Apps is owned by Terraform in the `infrastructure`
repository and installed through the `argocd-apps` Helm chart. This repository
owns only the child Application manifests under `platform/`, `catalog/`, and
`tenants/`.

## Contributing

Conventional Commits, issue reference required, rebase-only (no merge commits).
See [CONTRIBUTING.md](CONTRIBUTING.md).
