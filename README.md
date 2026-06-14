# gitops

GitOps configuration for the FH Burgenland Group F multi-tenant SaaS platform.
The cluster is reconciled by **ArgoCD**: everything in
this repo is applied to the cluster by ArgoCD, not by hand.

## Layout

| Path | Purpose |
|------|---------|
| `bootstrap/` | Root **App-of-Apps** — the single ArgoCD `Application` (installed once by the infrastructure repo) that points ArgoCD at `platform/` and `catalog/`. |
| `platform/` | Cluster **platform add-ons** as ArgoCD `Application`s (Helm): ingress-nginx, cert-manager, External Secrets, ExternalDNS, Crossplane. |
| `catalog/` | Crossplane **service catalog** — XRDs + Compositions that define what a "tenant" is. |
| `tenants/` | Per-tenant **claims** |


## Adding a tenant

Create a tenant claim under `tenants/<name>/` (copy `tenants/staging/` as a
template) and open a PR. Once merged, ArgoCD + Crossplane provision the tenant
automatically. See `tenants/README.md`.

## Contributing

Conventional Commits, issue reference required, rebase-only (no merge commits).
See [CONTRIBUTING.md](CONTRIBUTING.md).
