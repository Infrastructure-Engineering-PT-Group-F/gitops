# Per-tenant database credentials — example (reference only)

This is a **reference**, not an applied manifest. It shows the per-tenant objects
that deliver a tenant's database credentials.

**Credentials (Secret, ESO-owned):** `username` + an auto-generated `password`
in `weather-app-backend-db`. No plaintext password in git — ESO generates it. The
backend reads `username`/`password` from this Secret.

```yaml
# The tenant's database.
apiVersion: platform.fh-burgenland.at/v1alpha1
kind: SQLInstance
metadata:
  name: tenant-a-db
  namespace: tenant-a
spec:
  parameters:
    databaseName: appdb
---
# Auto-generates the password.
apiVersion: generators.external-secrets.io/v1alpha1
kind: Password
metadata:
  name: db-password
  namespace: tenant-a
spec:
  length: 32
  digits: 5
  symbols: 0
  noUpper: false
  allowRepeat: true
---
# Credentials (username + generated password). 
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: weather-app-backend-db
  namespace: tenant-a
spec:
  refreshInterval: "0"
  target:
    name: weather-app-backend-db
    creationPolicy: Owner
    template:
      engineVersion: v2
      data:
        username: "appdb-user"
        password: "{{ .password }}"
  dataFrom:
    - sourceRef:
        generatorRef:
          apiVersion: generators.external-secrets.io/v1alpha1
          kind: Password
          name: db-password
```
