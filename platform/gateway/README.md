# gateway

The shared platform Gateway is the single public entrypoint for the cluster.
ArgoCD reconciles the raw Gateway API manifests under
`platform/gateway/manifests` through the `gateway` Application (a directory-type
source, like `crossplane-providers`). It depends on the `envoy-gateway`
Application, which installs the Envoy Gateway controller and the Gateway API
CRDs.

## Contents

| Manifest | Purpose |
|----------|---------|
| `00-gatewayclass.yaml` | Cluster-scoped `GatewayClass` `envoy-gateway`, bound to the Envoy Gateway controller (`gateway.envoyproxy.io/gatewayclass-controller`). The Helm chart ships the controller but no GatewayClass. |
| `10-gateway.yaml` | The `shared-gateway` Gateway in namespace `platform-gateway`, with an HTTP `:80` and an HTTPS `:443` listener. |
| `20-http-to-https-redirect.yaml` | Platform-owned `HTTPRoute` that 301-redirects `:80` traffic to HTTPS. |
| `30-wildcard-certificate.yaml` | cert-manager `Certificate` for `*.gcp.ajdininfrastructure.lol`, issued by `letsencrypt-prod-dns01`, producing the `wildcard-gcp-tls` Secret. |

## Listeners

Both listeners use the wildcard hostname `*.gcp.ajdininfrastructure.lol`. A
single wildcard keeps tenant onboarding TLS-free: a new tenant just picks a
subdomain and attaches an `HTTPRoute`, no per-tenant certificate needed.

- **`http` (:80)** — exists only to redirect to HTTPS. `allowedRoutes` is scoped
  to `from: Same`, so only the platform-owned redirect `HTTPRoute` in this
  namespace can attach; tenant routes never land on plaintext.
- **`https` (:443)** — `tls.mode: Terminate`, terminating with the wildcard
  certificate in the `wildcard-gcp-tls` Secret produced by
  `30-wildcard-certificate.yaml` (cert-manager, issued by `letsencrypt-prod-dns01`,
  in this same namespace as required by Gateway API). Tenant
  `HTTPRoute`s attach via `allowedRoutes.namespaces.from: Selector` matching any
  namespace that carries the `platform.fh-burgenland.at/tenant` label — the
  one-namespace-per-tenant soft-multi-tenancy model.

## HTTP→HTTPS redirect

Gateway API has no listener-level redirect. The redirect is realised by
`20-http-to-https-redirect.yaml`: an `HTTPRoute` with a `RequestRedirect` filter
(`scheme: https`, `statusCode: 301`) and no `backendRefs`, attached to the
Gateway's `http` listener via `sectionName: http`.

## Namespace

The Gateway and the `wildcard-gcp-tls` Secret share the `platform-gateway`
namespace. Gateway API resolves `certificateRefs` in the Gateway's own namespace
by default. A cross-namespace reference would require a `ReferenceGrant`, so
co-locating the cert avoids that. The namespace is created by the Application
(`CreateNamespace=true`).

## Verification

The Gateway reports `Programmed=True` and is assigned a Google Cloud
LoadBalancer address once the Envoy Gateway controller provisions the backing
proxy. ExternalDNS then publishes DNS records for tenant `HTTPRoute` hostnames
that point at the Gateway address.
