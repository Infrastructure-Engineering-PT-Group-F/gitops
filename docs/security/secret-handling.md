# Secret Handling

## Purpose and Scope

This document defines how sensitive values are handled in this GitOps
repository and in the related platform components that deliver those values to
workloads. It documents the approved model for repository content,
operator-managed delivery, workload consumption, local-only exceptions, and the
point-in-time audit evidence collected on 2026-06-23.

## Security Principles

- No plaintext secrets, passwords, API keys, tokens, private keys, service
  account JSON files, or kubeconfig files in Git.
- GitOps manifests refer to a Secret name and key, never a secret value.
- GKE Workload Identity is used instead of static Google service-account keys.
- Google Secret Manager is the external secret source.
- Kubernetes Secrets are delivery objects for workloads, not the source of
  truth.
- Secret values must never be printed in logs, issue comments, PR descriptions,
  screenshots, or documentation.

## Architecture and Responsibilities

The approved delivery flow is:

```text
Google Secret Manager
-> External Secrets Operator
-> Kubernetes Secret in the target namespace
-> Workload through secretKeyRef or imagePullSecrets
```

ArgoCD installs External Secrets Operator. The cluster SecretStore integration
uses the `ClusterSecretStore` named `gcp-secret-manager`, with Google Cloud
Secret Manager as the provider.

The ESO Kubernetes ServiceAccount is
`external-secrets/external-secrets`. ESO uses GKE Workload Identity with the
Google service account identity
`external-secrets-sa@dark-diagram-496907-k8.iam.gserviceaccount.com`.

The manual `roles/secretmanager.secretAccessor` grant is a controlled bootstrap
exception owned by the operator. It is not a reason to create static Google
service-account keys or store credential files in Git.

## Workload Consumption Rules

The current intended workload consumption patterns are:

- Backend API key: existing Secret `api-keys`, key `avwx-api-key`.
- Backend external PostgreSQL mode: existing Secret `weather-app-backend-db`
  with `url`, `username`, and `password` keys.
- Frontend private GHCR image: `imagePullSecrets` must be supplied per tenant,
  preferably through the approved external-secret process.
- A tenant Secret must be namespace-scoped.
- Do not create or share a global tenant application Secret across namespaces.

## Local Development Exception

The backend Helm chart supports local-only Secret rendering when
`apiKeys.useExistingSecret=false`.

This mode is prohibited for GitOps, staging, and production values committed to
Git. Local values containing `apiKeys.avwx` must stay outside Git and be ignored
by local ignore rules.

## Audit Evidence - 2026-06-23

The following audit evidence reflects a point-in-time review on 2026-06-23:

- ESO ArgoCD Application was Synced and Healthy.
- ClusterSecretStore `gcp-secret-manager` was Valid.
- The ESO Kubernetes ServiceAccount had the expected Workload Identity
  annotation.
- No ExternalSecret resources existed at audit time.
- Therefore, the end-to-end delivery path from Google Secret Manager through
  ESO to a tenant-scoped Kubernetes Secret had not yet been validated.
- The cluster Secret inventory was inspected by names and types only; no values
  were read.
- A point-in-time pattern scan of locally available gitops, infrastructure, and
  backend repositories found no candidate plaintext-secret, token, API-key,
  private-key, or service-account JSON patterns.
- frontend and first-cluster were not available as local Git working copies at
  audit time, so their source content was not included in that local scan.
- The scan was not a Git-history scan and does not replace continuous secret
  scanning.

## Validation Commands

The following commands are safe validation commands because they do not print
secret data:

```powershell
kubectl -n argocd get application external-secrets
kubectl get clustersecretstore
kubectl get externalsecret -A
kubectl get secret -A -o custom-columns=NAMESPACE:.metadata.namespace,NAME:.metadata.name,TYPE:.type
git ls-files
```

Do not use commands that print Secret manifests, encoded Secret data, decoded
Secret data, or Secret descriptions.

## Required Follow-up

The following work is tracked as GitHub issues:

- Add a GitOps CI secret-scanning control, such as Gitleaks. Tracked by
  [gitops #50](https://github.com/Infrastructure-Engineering-PT-Group-F/gitops/issues/50).
- Validate at least one tenant-scoped `ExternalSecret` after XTenant
  provisioning is implemented. Tracked by
  [gitops #39](https://github.com/Infrastructure-Engineering-PT-Group-F/gitops/issues/39).
- Deliver tenant database credentials through ESO. Tracked by
  [gitops #42](https://github.com/Infrastructure-Engineering-PT-Group-F/gitops/issues/42).
- Validate tenant image-pull Secret delivery for the private frontend image.
  Tracked by
  [frontend #34](https://github.com/Infrastructure-Engineering-PT-Group-F/frontend/issues/34).
- Validate backend consumption of the tenant-scoped external PostgreSQL Secret.
  Tracked by
  [backend #28](https://github.com/Infrastructure-Engineering-PT-Group-F/backend/issues/28).
- Do not treat database Secret automation as live until the related GitOps and
  backend validation work is merged and verified.

## Ownership and Escalation

- Application teams request and consume named secrets.
- Platform team operates ESO and SecretStore integration.
- Cloud operator manages the controlled Google IAM bootstrap exception.
- A suspected exposure requires immediate credential rotation, removal from Git,
  history remediation if necessary, and incident documentation.
