# catalog

The Crossplane **service catalog**: the abstraction that defines what a
"tenant" is. Developers consume it from `tenants/` without touching the
underlying resources.

- `xrds/` — Composite Resource Definitions (the tenant API + its OpenAPIv3 schema)
- `compositions/` — Compositions that turn a tenant Composite Resource (XR) into real resources:
  namespace, NetworkPolicies, ResourceQuota/LimitRange, 
  and the backend/frontend Helm releases.

Validate offline with `crossplane render` / `crossplane beta validate` — no
cluster required to author these.
