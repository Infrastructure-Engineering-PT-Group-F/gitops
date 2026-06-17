# external-dns

`external-dns` publishes DNS records for Gateway API `HTTPRoute` hostnames in the
delegated platform domain.

## Scope

- Domain: `gcp.ajdininfrastructure.lol`
- Provider: Google Cloud DNS
- Source: Gateway API `HTTPRoute`
- Policy: `upsert-only`
- Registry: TXT ownership records with owner ID `gitops-platform-gcp`

## Identity

The Helm chart creates the Kubernetes service account `external-dns/external-dns`
and uses GKE Workload Identity.

The Kubernetes service account annotation in `application.yaml` must match the
Terraform output `external_dns_sa_email` from
`Infrastructure-Engineering-PT-Group-F/infrastructure#39`.

The Google Cloud project is not configured explicitly in this manifest. The
provider is expected to use the GKE metadata and Workload Identity context at
runtime.

No Google service account JSON keys or static credentials are stored in this
repository.
