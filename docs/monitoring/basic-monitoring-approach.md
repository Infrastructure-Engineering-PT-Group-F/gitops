# Basic Monitoring Approach

## Purpose and Scope

This document records the basic monitoring decision for the FH Burgenland Group
F multi-tenant SaaS platform.

It fulfils GitOps issue #12 as a documentation-only bonus task. It defines the
selected tooling, minimum metric baseline, intended operational views, security
and cost controls, and the implementation boundaries for later work.

This document does not deploy monitoring resources, dashboards, alert policies,
Grafana, Prometheus Operator, or cloud configuration. Those activities are
explicitly deferred to GitOps issue #13 and to the tenant implementation work.

## Decision

The platform uses the following monitoring model:

```text
Google Cloud Monitoring
+ Google Managed Service for Prometheus (GMP) managed collection
+ GKE curated system metric packages
+ namespace-scoped PodMonitoring resources for application metrics
```

The platform will not deploy or operate a self-hosted Prometheus and Grafana
stack for the bonus MVP.

Google Cloud Monitoring is the central query, dashboard, and future alerting
surface. GMP provides managed scraping and storage for Prometheus-format
application metrics. GKE curated metric packages provide the Kubernetes and
container resource baseline.

Grafana remains optional and is not part of the selected MVP.

PodMonitoring is selected over ClusterPodMonitoring for the initial MVP
because each monitoring definition remains inside the relevant tenant
namespace. This aligns application scraping with the tenant namespace
boundary and avoids an unnecessary cluster-wide discovery rule.

## Decision Drivers

The selected model is based on the following priorities:

- Minimise operational overhead for a bonus task.
- Reuse the monitoring capabilities already enabled in the GKE cluster.
- Keep Kubernetes workload metrics and application metrics in one operational
  monitoring surface.
- Avoid operating Prometheus storage, retention, sharding, upgrades,
  Alertmanager, Grafana, backups, and additional cluster RBAC.
- Keep the approach compatible with future tenant compositions.
- Prevent monitoring work from delaying the required tenant end-to-end proof.

## Current Platform Evidence

The following evidence was collected through read-only commands on 2026-06-24.

| Area | Observed state | Decision impact |
|---|---|---|
| Managed Prometheus | `managedPrometheusConfig.enabled` is `true`. | GMP managed collection is available. |
| GKE metric packages | `SYSTEM_COMPONENTS`, `STORAGE`, `HPA`, `POD`, `DAEMONSET`, `DEPLOYMENT`, `STATEFULSET`, `CADVISOR`, `KUBELET`, `DCGM`, and `JOBSET` are enabled. | Core workload, resource, and Kubernetes-object signals are available. |
| GMP API resources | `PodMonitoring`, `ClusterPodMonitoring`, `Rules`, and related GMP resources are installed. | GMP-native scraping can be configured later without adding Prometheus Operator. |
| GMP components | GMP collector and GMP operator Pods are running. | The managed collection control plane is present. |
| Application targets | No `PodMonitoring` resources currently exist. | No application-specific scrape target has been configured yet. |
| Backend metrics capability | The backend includes Spring Boot Actuator and Micrometer Prometheus support. | The backend can become the first application scrape target after endpoint hardening. |
| Existing ServiceMonitor template | The backend Helm chart contains an optional, disabled Prometheus Operator `ServiceMonitor`. | This is not the selected mechanism for the GMP MVP. |

The enabled `DCGM` package is out of scope unless the platform intentionally
introduces GPU workloads.

## Target Architecture

```text
GKE cluster
|
+-- GKE curated system metrics
|   +-- Pod, Deployment, StatefulSet, DaemonSet, HPA, and storage signals
|   +-- cAdvisor and Kubelet container resource metrics
|
+-- Google Managed Service for Prometheus
|   +-- GMP operator and collector
|   +-- Future PodMonitoring resources in tenant namespaces
|       +-- Backend /actuator/prometheus endpoint
|
+-- Google Cloud Monitoring
    +-- Metrics Explorer and PromQL queries
    +-- Internal application-owner dashboard view
    +-- Platform-administrator dashboard view
    +-- Optional future alert policies
```

The management path is intentionally split by responsibility:

- GKE provides system and Kubernetes-object metrics.
- GMP collects Prometheus-format application metrics.
- Cloud Monitoring provides querying, visualization, and future alerting.
- GitOps owns future namespace-scoped `PodMonitoring` resources.
- The backend owns safe exposure of its application metric endpoint.

## Minimum Metric Baseline

| Monitoring goal | Initial signal | Primary source | Initial audience |
|---|---|---|---|
| Pod health | Ready versus desired workload replicas; Pending, Failed, and unavailable Pods | GKE POD, DEPLOYMENT, STATEFULSET, and DAEMONSET metrics | Application owner, platform administrator |
| Restarts | Container restart count and repeated non-ready workloads | GKE POD metrics | Application owner, platform administrator |
| CPU usage | Container and namespace CPU consumption | cAdvisor and Kubelet metrics | Application owner, platform administrator |
| Memory usage | Container and namespace memory consumption | cAdvisor and Kubelet metrics | Application owner, platform administrator |
| Namespace resource usage | Aggregated CPU and memory grouped by namespace | cAdvisor and Kubelet metrics | Application owner, platform administrator |
| Resource governance | Requested and limited resources after ResourceQuota and LimitRange implementation | Kubernetes object metrics and GitOps issue #7 | Platform administrator |
| Backend scrape health | Prometheus target availability through the `up` metric | GMP PodMonitoring after implementation | Application owner, platform administrator |
| Backend runtime metrics | JVM, process, HTTP, and Spring application metrics | Spring Boot Actuator and Micrometer after implementation | Application owner, platform administrator |
| HTTP and ingress health | Request volume, error rate, and latency | Deferred until backend and Gateway metric contracts are validated | Platform administrator |
| Database health | Cloud SQL availability and connection health | Deferred until GitOps issue #51 validates the staging database path | Platform administrator |

The first usable dashboard must cover Pod health, restarts, CPU, memory,
namespace resource usage, and backend scrape health.

## Views and Access Boundaries

### Application Owner View

The application-owner view is an internal operational dashboard filtered to one
tenant namespace, for example `tenant-staging`.

It should show:

- Backend and frontend readiness.
- Restart count and recent Pod failures.
- CPU and memory consumption for the selected namespace.
- Backend scrape availability after PodMonitoring is implemented.
- Later: application request errors and latency.

A namespace-filtered dashboard is a convenience and operational view. It is not
a technical authorization boundary.

External tenants must not receive unrestricted Google Cloud Monitoring
project-level access merely because their workloads run in a namespace. The
current shared-project model requires Cloud Monitoring access to remain with the
internal platform team.

### Platform Administrator View

The platform-administrator view covers the full cluster and platform
components.

It should show:

- Node and cluster resource pressure.
- Non-ready workloads across all namespaces.
- Restarting Pods and failed workload rollouts.
- Namespace-level CPU and memory distribution.
- GMP collector and scrape-target health.
- Later: ArgoCD, Crossplane, External Secrets Operator, Envoy Gateway,
  cert-manager, ExternalDNS, Cloud SQL, and tenant rollout status.

## Security and Data Handling

Monitoring must not weaken the platform security model.

- Do not expose `/actuator/prometheus` or detailed Actuator endpoints through a
  public HTTPRoute.
- Do not add a backend GMP scrape target until the backend management endpoint
  is limited to internal cluster access and the public route cannot reach it.
- Do not export credentials, tokens, API keys, user identifiers, client IP
  addresses, complete request URLs, request bodies, database connection strings,
  or other sensitive values as metric labels.
- Do not include secret values in dashboards, alert annotations, issue comments,
  screenshots, logs, or documentation.
- Keep application metric labels low-cardinality and stable.
- Use namespace labels for tenant grouping; do not use user-specific labels.
- Do not treat dashboard filters as access-control enforcement.
- Apply least privilege to Cloud Monitoring access and dashboard editing.

A separate backend hardening follow-up is required before public application
routing and GMP scraping are enabled together.

## Cost and Cardinality Controls

The monitoring MVP must remain cost-aware.

- Reuse the already enabled curated GKE metric packages.
- Begin with one backend scrape target in the staging tenant only.
- Use a 30-second scrape interval as the initial application metric baseline.
- Add metrics only when they answer a defined operational question.
- Avoid labels with many possible values, including user IDs, client IP
  addresses, request IDs, dynamic URLs, and random identifiers.
- Review ingestion volume, labels, and cardinality before adding further scrape
  targets or changing intervals.
- Do not add a self-hosted Prometheus federation layer.
- Do not introduce alert policies until a baseline has been observed and alert
  thresholds can be justified.

## Implementation Boundaries and Follow-up Work

### GitOps Issue #13 - Basic Health and Resource Dashboard

GitOps issue #13 is the implementation follow-up for this decision.

The implementation scope should be:

1. Add one GMP-native PodMonitoring resource for the backend in
   `tenant-staging`.
2. Select the backend Pod labels and named `http` port.
3. Scrape `/actuator/prometheus` only after the backend management endpoint is
   confirmed internal-only.
4. Confirm target availability through the PromQL query `up`.
5. Create two Cloud Monitoring dashboard views:
   - Internal application-owner view filtered to `tenant-staging`.
   - Platform-administrator view across platform namespaces.
6. Record dashboard configuration, screenshots, and safe validation evidence.
7. Keep alert policies optional until the tenant baseline is stable.

### GitOps Issue #39 - Tenant Composition

After the XTenant Composition is implemented, monitoring should become a tenant
lifecycle concern:

- Each tenant namespace receives its own PodMonitoring resource.
- Tenant name and namespace remain stable aggregation dimensions.
- The Composition must not expose management endpoints publicly.
- Tenant dashboard configuration remains internal unless a dedicated
  authorization model is designed.

### GitOps Issue #51 - Staging Database Credential Validation

Cloud SQL and database health metrics are deferred until the real private-IP
Cloud SQL and ESO credential delivery flow is validated and cleaned up.

### Backend Follow-up

Before GMP scraping is implemented, create or assign a backend hardening task
that verifies all of the following:

- The management endpoint is not reachable through public HTTPRoute routing.
- Actuator exposure is restricted to the minimum required endpoints.
- Health details are not unnecessarily public.
- The Prometheus endpoint remains reachable only from the intended in-cluster
  collector path.
- Application metric labels comply with the security and cardinality rules in
  this document.

## Ownership Model

| Area | Owner |
|---|---|
| GKE metric packages and GMP enablement | Infrastructure and cloud operator |
| GMP-native PodMonitoring resources | GitOps and tenant composition |
| Backend Actuator and Prometheus endpoint contract | Backend team |
| Cloud Monitoring dashboards and operational views | Platform team |
| Tenant operational review | Internal application owner |
| Alert policy design after baseline collection | Platform team |
| External tenant observability model | Separate future architecture decision |

## Read-only Validation Commands

The following commands inspect monitoring configuration and resource metadata
without reading Secret values:

```powershell
gcloud container clusters describe group-f-platform-gke `
  --zone europe-west1-b `
  --project dark-diagram-496907-k8 `
  --format="yaml(monitoringConfig)"

kubectl api-resources --api-group=monitoring.googleapis.com

kubectl get pods -A

kubectl get podmonitoring -A
```

After the first PodMonitoring resource is implemented, use Cloud Monitoring
Metrics Explorer with PromQL query `up` to confirm that the backend target is
being scraped. Do not publish sensitive metric labels or detailed endpoint
output as validation evidence.

## Non-Goals

The following are not part of GitOps issue #12:

- Deploying Prometheus Operator.
- Deploying Grafana.
- Creating executable PodMonitoring manifests.
- Creating dashboards or alert policies.
- Changing GKE monitoring configuration.
- Changing backend Actuator, HTTPRoute, or Helm chart configuration.
- Exposing monitoring views to external tenants.
- Implementing SLOs, error budgets, distributed tracing, or log analytics.

## References

- [GitOps issue #12 - Basic monitoring approach](https://github.com/Infrastructure-Engineering-PT-Group-F/gitops/issues/12)
- [GitOps issue #13 - Basic health and resource dashboard](https://github.com/Infrastructure-Engineering-PT-Group-F/gitops/issues/13)
- [GitOps issue #39 - Tenant XRD and Composition](https://github.com/Infrastructure-Engineering-PT-Group-F/gitops/issues/39)
- [GitOps issue #51 - Staging database credential validation](https://github.com/Infrastructure-Engineering-PT-Group-F/gitops/issues/51)
- [Google Cloud Managed Service for Prometheus managed collection](https://cloud.google.com/stackdriver/docs/managed-prometheus/setup-managed)
- [Google Kubernetes Engine metric collection](https://cloud.google.com/kubernetes-engine/docs/how-to/configure-metrics)
- [Query managed Prometheus metrics in Cloud Monitoring](https://cloud.google.com/stackdriver/docs/managed-prometheus/query-cm)
- [Managed Prometheus cost controls](https://cloud.google.com/stackdriver/docs/managed-prometheus/cost-controls)
