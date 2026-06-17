# platform

Cluster **platform add-ons**, each as an ArgoCD `Application` that
deploys an upstream Helm chart with pinned versions. These provide the shared
capabilities every tenant relies on. They install no tenant resources themselves.

Add-ons (one subfolder each):

| Add-on | Purpose |
|--------|---------|
| `envoy-gateway` | Gateway API ingress (HTTP/HTTPS) for public endpoints |
| `crossplane` | Control plane for the tenant service catalog |
