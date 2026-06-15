# platform

Cluster **platform add-ons**, each as an ArgoCD `Application` that
deploys an upstream Helm chart with pinned versions. These provide the shared
capabilities every tenant relies on. They install no tenant resources themselves.

Add-ons (one subfolder each):

| Add-on | Purpose |
|--------|---------|
| `envoy-gateway` | Gateway API ingress (HTTP/HTTPS) for public endpoints |
| `cert-manager` | ACME TLS certificates (DNS-01) |
| `external-secrets` | Sync secrets from GCP Secret Manager (ESO) |
| `external-dns` | Manage DNS records from Gateway/HTTPRoute objects |
| `crossplane` | Control plane for the tenant service catalog |
