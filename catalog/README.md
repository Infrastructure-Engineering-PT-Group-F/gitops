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
`sqlinstance-gcp-cloudsql` Composition provisions a Cloud SQL `DatabaseInstance`
and `Database`. The `User` resource exists in the composition, but credential
wiring is handled separately.

| Parameter | Required | Default | Purpose |
|-----------|----------|---------|---------|
| `databaseName` | yes | - | Name of the application database to create |
| `region` | no | `europe-west1` | GCP region for the instance |
| `databaseVersion` | no | `POSTGRES_18` | Cloud SQL Postgres engine version. Constrained to `POSTGRES_16` / `POSTGRES_17` / `POSTGRES_18` |
| `tier` | no | `db-f1-micro` | Machine type (main cost driver) |
| `privateNetwork` | no | platform VPC self-link | VPC the instance gets its private IP on; must match the infra `vpc_self_link` output |

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
