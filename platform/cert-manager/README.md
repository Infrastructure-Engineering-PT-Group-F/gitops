# cert-manager

cert-manager is the platform add-on that issues and renews TLS certificates for
cluster workloads. ArgoCD deploys the upstream Jetstack Helm chart from
`platform/cert-manager/application.yaml` with a pinned chart version.

This add-on configures two ClusterIssuers for Let's Encrypt DNS-01 validation
through Google Cloud DNS:

- `letsencrypt-staging-dns01` - Let's Encrypt staging endpoint. Untrusted
  certificates, high rate limits, for testing.
- `letsencrypt-prod-dns01` - Let's Encrypt production endpoint. Browser-trusted
  certificates, strict rate limits, for the real public URL.

Both solvers are scoped to `gcp.ajdininfrastructure.lol` and use Google Cloud
project `dark-diagram-496907-k8`. Each issuer keeps its own ACME account key
(`letsencrypt-staging-dns01-account-key` and
`letsencrypt-prod-dns01-account-key`), since staging and production are separate
ACME accounts.

The ClusterIssuer is rendered through the cert-manager Helm chart
`extraObjects` value, so it is reconciled by the same ArgoCD Application as the
chart and is not an orphan manifest.

The Google Cloud DNS permissions are provided by infrastructure issue
`Infrastructure-Engineering-PT-Group-F/infrastructure#39`. That work exposes the
`cert_manager_dns01_sa_email` Terraform output, expected to be:

```text
cert-manager-dns01-sa@dark-diagram-496907-k8.iam.gserviceaccount.com
```

The cert-manager Kubernetes service account is mapped to that Google service
account with GKE Workload Identity:

```text
cert-manager/cert-manager -> cert-manager-dns01-sa@dark-diagram-496907-k8.iam.gserviceaccount.com
```

No static Google credentials are stored in this repository. The issuer does not
use a `serviceAccountSecretRef`, and no Google service account JSON key is
committed.

The ACME account email is configured as a non-secret operational contact address:
`2510781019@hochschule-burgenland.at`.

Workloads select an issuer by `issuerRef.name`: use `letsencrypt-staging-dns01`
while testing to avoid burning production rate limits, then switch to
`letsencrypt-prod-dns01` for a browser-trusted certificate.

