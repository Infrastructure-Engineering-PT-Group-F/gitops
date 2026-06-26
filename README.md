# gitops

GitOps configuration for the FH Burgenland Group F multi-tenant SaaS platform.
The cluster is reconciled by ArgoCD: everything in this repo is applied to the
cluster by ArgoCD, not by hand.

## Layout

| Path        | Purpose                                                                          |
| ----------- | -------------------------------------------------------------------------------- |
| `platform/` | Cluster platform add-ons as ArgoCD `Application`s (Helm).                        |
| `catalog/`  | Crossplane service catalog - XRDs + Compositions that define what a "tenant" is. |
| `tenants/`  | Per-tenant Composite Resources (XRs)                                             |

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

Child Applications run under the restricted `platform` ArgoCD `AppProject`
(created by Terraform alongside the root App), not the built-in `default`
project, so every `application.yaml` sets `spec.project: platform`. The project
allowlists the permitted source repos, so **adding a platform add-on that pulls
from a new chart repo also requires adding that repo to the project's
`sourceRepos` in `infrastructure/platform/argocd.tf`** — otherwise ArgoCD
rejects the Application.

## Security and Secrets

See [Secret Handling](docs/security/secret-handling.md) for the secret
management architecture, repository rules, local-development exception and audit
procedure.

## Monitoring (Bonus)

See [Basic Monitoring Approach](docs/monitoring/basic-monitoring-approach.md)
for the documented Google Cloud Monitoring and Google Managed Service for
Prometheus decision, monitoring baseline, ownership model, security rules, and
implementation boundaries.

## Operations

See [Operational Runbook](docs/operations/runbook.md) for read-only health
checks across GitOps, Crossplane, tenants, TLS, DNS and secrets, plus the common
failure cases and the post-rebuild recovery steps.

## Contributing

Conventional Commits, issue reference required, rebase-only (no merge commits).
See [CONTRIBUTING.md](CONTRIBUTING.md).
