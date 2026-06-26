# Staging tenant validation checklist

The staging tenant is the canary: validate a new application version or a
platform change here before promoting it to other tenants. This checklist
defines the technical checks to run and how to record the evidence.

All checks are read-only. Replace `<name>` with the tenant under test
(`staging`, or another tenant such as `validation` when it carries the full app
stack) and `<host>` with its hostname under `*.gcp.ajdininfrastructure.lol`.

Related docs:

- Operational runbook: [`runbook.md`](runbook.md)
- Tenant onboarding: [`docs/tenant-onboarding.md`](../tenant-onboarding.md)
- Multi-tenancy concept:
  [`docs/security/multi-tenancy.md`](../security/multi-tenancy.md)
- Tenant model: [`tenants/README.md`](../../tenants/README.md)

## Checklist

| #  | Check                                              | Command                                                                                                                 | Pass criteria                                                                        |
| -- | -------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------ |
| 1  | Tenant exists and is reconciled                    | `kubectl get ns tenant-<name>` and `kubectl get application tenant-<name> -n argocd`                                    | Namespace present, Application `Synced` and `Healthy`                                |
| 2  | Composite and database Ready                       | `kubectl get xtenant,sqlinstance -n tenant-<name>`                                                                      | `SQLInstance Ready=True` and `XTenant Ready=True` once workloads are up              |
| 3  | Backend reachable                                  | `curl -sS -m 8 -o /dev/null -w "%{http_code}\n" https://<host>/api/metar/LOWW`                                          | `200`                                                                                |
| 4  | Frontend reachable                                 | `curl -sS -m 8 -o /dev/null -w "%{http_code}\n" https://<host>/`                                                        | `200`                                                                                |
| 5  | Backend connects to the database                   | `kubectl get secret weather-app-backend-db -n tenant-<name>` and check backend logs                                     | Secret present, backend healthy, no DB connection errors                             |
| 6  | HTTPS certificate valid                            | `curl -sSI https://<host>/api/metar/LOWW` (no TLS error) and `kubectl get certificate wildcard-gcp -n platform-gateway` | TLS handshake succeeds, certificate `Ready=True`                                     |
| 7  | DNS record exists                                  | `dig +short <host>`                                                                                                     | Resolves to the current Gateway IP (see note)                                           |
| 8  | Network policies allow required and block unwanted | `kubectl get networkpolicy -n tenant-<name>` plus the negative test below                                               | Required traffic works, an unlabeled or cross-namespace pod cannot reach the backend |
| 9  | Quotas and limits do not break the deployment      | `kubectl get resourcequota,limitrange -n tenant-<name>`                                                                 | Pods `Running`, usage within the quota, no pod rejected for missing requests         |
| 10 | Application update tested                          | Bump `backendImageTag`/`frontendImageTag`, then re-run checks 1-9                                                       | Rollout completes and all checks pass on the new version                             |

> The Gateway IP is not fixed, it can change on a Terraform reapply. Get the
> current value from the cluster with
> `kubectl get gateway shared-gateway -n platform-gateway -o jsonpath='{.status.addresses[*].value}'`,
> or from `terraform output` in `infrastructure/platform`.

### Network policy negative test (check 8)

```sh
# Denied: an unlabeled tenant pod must not reach the backend.
kubectl -n tenant-<name> run np-deny-test --rm -it --restart=Never \
  --image=curlimages/curl:8.8.0 \
  -- curl -m 5 -fsS http://weather-app-backend:8080/actuator/health/readiness
# Expected: the request times out or is refused, not a 200.
```

## Recording a run

Copy this table into the issue or PR and fill it in. Record date, the version
under test, pass or fail per check and a short note.

| Check                | Result | Note |
| -------------------- | ------ | ---- |
| 1 Tenant reconciled  |        |      |
| 2 Composite/DB Ready |        |      |
| 3 Backend reachable  |        |      |
| 4 Frontend reachable |        |      |
| 5 Backend to DB      |        |      |
| 6 HTTPS valid        |        |      |
| 7 DNS                |        |      |
| 8 NetworkPolicies    |        |      |
| 9 Quotas/limits      |        |      |
| 10 Update tested     |        |      |
