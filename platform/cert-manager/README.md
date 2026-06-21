# cert-manager

cert-manager is the platform add-on that issues and renews TLS certificates for
cluster workloads. ArgoCD deploys the upstream Jetstack Helm chart from
`platform/cert-manager/application.yaml` with a pinned chart version.

This add-on configures the `letsencrypt-staging-dns01` ClusterIssuer for
Let's Encrypt DNS-01 validation through Google Cloud DNS. The solver is scoped
to `gcp.ajdininfrastructure.lol` and uses Google Cloud project
`dark-diagram-496907-k8`.

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

The configured issuer uses the Let's Encrypt staging endpoint until a production
issuer is intentionally added.

