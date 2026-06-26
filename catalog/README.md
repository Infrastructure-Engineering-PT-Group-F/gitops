# catalog

The Crossplane service catalog: the abstraction that defines what a "tenant" is.
Developers consume it from `tenants/` without touching the underlying resources.

- `xrds/` - Composite Resource Definitions (the tenant API + its OpenAPIv3 schema)
- `compositions/` - Compositions that turn a tenant Composite Resource (XR) into
  real resources: namespace, NetworkPolicies, ResourceQuota/LimitRange, and the
  backend/frontend Helm releases.

Validate offline with `crossplane render` / `crossplane beta validate` - no
cluster required to author these.

## XTenant - tenant application stack

`XTenant` (`platform.fh-burgenland.at/v1alpha1`, namespaced) is the tenant
application API. One `XTenant` is committed in the tenant namespace and the
`xtenant-weather-app` Composition renders the tenant database and application
Helm releases.

The tenant namespace itself remains ArgoCD-owned in this first milestone. The
namespace baseline, runtime-secret delivery, ResourceQuota and LimitRange stay
as explicit manifests under `tenants/<name>/`. The Composition starts after
those prerequisites and renders:

- a tenant `SQLInstance` named `weather-app-db`
- namespace-local ESO helper resources that build `weather-app-backend-db`
- a backend Helm `Release` of `weather-app-backend`
- a frontend Helm `Release` of `weather-app-frontend`
- Gateway API `HTTPRoute`s for `/api` to the backend and `/` to the frontend

| Parameter | Required | Default | Purpose |
|-----------|----------|---------|---------|
| `name` | yes | - | Short tenant identifier, for labels and operational ownership. |
| `environment` | yes | - | `staging`, `validation`, or `production`. |
| `hostname` | yes | - | Tenant hostname under `*.gcp.ajdininfrastructure.lol`. |
| `backendImageTag` | yes | - | Backend image tag passed to the backend chart. |
| `frontendImageTag` | yes | - | Frontend image tag passed to the frontend chart. |
| `size` | no | `small` | Future quota/limit size class. Accepted now, not rendered yet. |

```yaml
apiVersion: platform.fh-burgenland.at/v1alpha1
kind: XTenant
metadata:
  name: validation
  namespace: tenant-validation
spec:
  parameters:
    name: validation
    environment: validation
    hostname: validation.gcp.ajdininfrastructure.lol
    backendImageTag: "v1.5.0"
    frontendImageTag: "v1.1.1"
```

### Render flow

1. ArgoCD applies the tenant namespace, runtime-secret delivery and resource
   limits from `tenants/<name>/`. Runtime-secret delivery must create
   `api-keys`, `ghcr-pull`, and `ghcr-chart-pull` before the Helm releases can
   become Ready.
2. ArgoCD applies the namespaced `XTenant`.
3. Crossplane renders the `SQLInstance`, which provisions Cloud SQL and writes
   the non-secret connection Secret `weather-app-backend-db-conn`.
4. Crossplane renders namespace-local ESO helper resources. ESO combines the
   generated password with the connection Secret and creates
   `weather-app-backend-db`.
5. Crossplane renders the backend and frontend Helm releases with chart pull
   credentials from `ghcr-chart-pull`. The backend uses the `external-pg`
   profile, `api-keys`, and `weather-app-backend-db`; the frontend uses
   `ghcr-pull` for private image pulls.
6. Crossplane renders tenant `HTTPRoute`s on the shared HTTPS Gateway:
   `/api` goes to the backend Service and `/` goes to the frontend Service.

### Tenant routing

`XTenant` renders the public Gateway API routing directly. The backend chart's
own catch-all `HTTPRoute` is disabled so the tenant can use the same hostname
for both components: `/api` is routed to `weather-app-backend`, while `/` is
routed to `weather-app-frontend`. Both routes attach to `shared-gateway` in
`platform-gateway` on the `https` listener.

NetworkPolicies from #6 and the monitoring follow-up in #85 are intentionally
not rendered by `XTenant` yet. ResourceQuota and LimitRange from #7 also remain
ArgoCD-owned static tenant manifests in this milestone.

## SQLInstance - tenant Cloud SQL (Postgres)

`SQLInstance` (`platform.fh-burgenland.at/v1alpha1`, namespaced) is the self-service
database API. A tenant creates one `SQLInstance` in its namespace and the
`sqlinstance-gcp-cloudsql` Composition provisions a Cloud SQL `DatabaseInstance`,
a `Database`, and a `User`. The `User` password is supplied by an ESO-generated
Secret (see Credentials below). The composition also publishes the instance host
as a connection Secret for ESO to build the JDBC url.

| Parameter | Required | Default | Purpose |
|-----------|----------|---------|---------|
| `databaseName` | yes | - | Name of the application database to create |
| `region` | no | `europe-west1` | GCP region for the instance |
| `databaseVersion` | no | `POSTGRES_18` | Cloud SQL Postgres engine version. Constrained to `POSTGRES_16` / `POSTGRES_17` / `POSTGRES_18` |
| `tier` | no | `db-f1-micro` | Machine type (main cost driver) |
| `privateNetwork` | no | platform VPC self-link | VPC the instance gets its private IP on; must match the infra `vpc_self_link` output |
| `credentialsSecretName` | no | `weather-app-backend-db` | ESO-owned Secret holding the DB `url`/`username`/`password` |
| `connectionSecretName` | no | `weather-app-backend-db-conn` | Crossplane-owned Secret with `host`/`port`/`database` |

```yaml
apiVersion: platform.fh-burgenland.at/v1alpha1
kind: SQLInstance
metadata:
  name: tenant-a-db
  namespace: tenant-a
spec:
  parameters:
    databaseName: appdb
```

### Private IP (no public IP)

The instance is provisioned private-IP only (`settings.ipConfiguration` with
`ipv4Enabled: false` and `privateNetwork` = the platform VPC self-link). This depends
on the infrastructure Private Services Access connection - the service-networking
connection that consumes the infra `private_services_access_range_name` output - already
existing on that VPC. Keep `privateNetwork` in sync with the infra `vpc_self_link`
output so the gitops composition and the IaC peering do not drift.

### Credentials and connection

The backend Secret `credentialsSecretName` (default `weather-app-backend-db`) needs
`url`, `username`, and `password`. 

- **Connection Secret (Crossplane-owned).** The composition publishes the instance
  private IP (`status.atProvider.privateIpAddress`) plus `port`/`database` to
  `connectionSecretName` (default `weather-app-backend-db-conn`). Crossplane is the
  sole writer. Populated on the reconcile after the instance gets its IP.
- **Credentials Secret (ESO-owned).** An ESO `Password` generator + `ExternalSecret`
  own `credentialsSecretName`: ESO generates the `password`, reads `host` from the
  connection Secret via a Kubernetes `SecretStore`, and templates
  `url: jdbc:postgresql://<host>:5432/<database>`. 

