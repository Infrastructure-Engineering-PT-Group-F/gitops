# gitops

GitOps configuration for the FH Burgenland Group F multi-tenant SaaS platform.
The cluster is reconciled by ArgoCD: everything in this repo is applied to the
cluster by ArgoCD, not by hand.

## Layout

| Path | Purpose |
|------|---------|
| `platform/` | Cluster platform add-ons as ArgoCD `Application`s (Helm). |
| `catalog/` | Crossplane service catalog - XRDs + Compositions that define what a "tenant" is. |
| `tenants/` | Per-tenant Composite Resources (XRs) |

## Adding a tenant

Create a tenant resource under `tenants/<name>/` (copy `tenants/staging/` as a
template) and open a PR. Once merged, ArgoCD + Crossplane provision the tenant
automatically. See `tenants/README.md`.

## Root Application Ownership

Terraform in `Infrastructure-Engineering-PT-Group-F/infrastructure` creates and
owns only the root App-of-Apps directly. That root Application scans this
repository for child ArgoCD `Application` manifests.

This repository owns the child Applications and the reconciled content under
`platform/`, `catalog/`, and `tenants/`. Platform add-ons, catalog definitions,
and tenant resources should be changed here and reconciled by ArgoCD.

## Security and Secrets

See [Secret Handling](docs/security/secret-handling.md) for the secret
management architecture, repository rules, local-development exception and
audit procedure.

## Monitoring (Bonus)

See [Basic Monitoring Approach](docs/monitoring/basic-monitoring-approach.md)
for the documented Google Cloud Monitoring and Google Managed Service for
Prometheus decision, monitoring baseline, ownership model, security rules, and
implementation boundaries.

## Contributing

Conventional Commits, issue reference required, rebase-only (no merge commits).
See [CONTRIBUTING.md](CONTRIBUTING.md).
