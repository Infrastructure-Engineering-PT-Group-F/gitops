# catalog

The Crossplane service catalog: the abstraction that defines what a "tenant" is.
Developers consume it from `tenants/` without touching the underlying resources.

- `xrds/` - Composite Resource Definitions (the tenant API + its OpenAPIv3 schema)
- `compositions/` - Compositions that turn a tenant Composite Resource (XR) into
  real resources: namespace, NetworkPolicies, ResourceQuota/LimitRange, and the
  backend/frontend Helm releases.

Validate offline with `crossplane render` / `crossplane beta validate` - no
cluster required to author these.

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

