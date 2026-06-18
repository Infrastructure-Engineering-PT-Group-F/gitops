# external-secrets

External Secrets Operator (ESO) syncs secrets from Google Cloud Secret Manager
into the cluster as native Kubernetes Secrets, so no plaintext secret is ever
committed to Git. ArgoCD deploys the upstream Helm chart from
`platform/external-secrets/application.yaml` with a pinned chart version.

## Scope

- Provider: Google Cloud Secret Manager (`gcpsm`)
- Store: a cluster-wide `ClusterSecretStore` named `gcp-secret-manager`
- Project: `project-03eb5b2f-b0b9-4171-b14`

The `ClusterSecretStore` is rendered through the chart `extraObjects` value, so
it is reconciled by the same ArgoCD Application as the operator and is not an
orphan manifest.

## Identity

The chart creates the Kubernetes service account `external-secrets/external-secrets`
and uses GKE Workload Identity, annotated with:

    iam.gke.io/gcp-service-account: external-secrets-sa@project-03eb5b2f-b0b9-4171-b14.iam.gserviceaccount.com

The Google service account, its Workload Identity binding for
`external-secrets/external-secrets` and the `roles/secretmanager.secretAccessor`
grant come from a dedicated infrastructure IAM change. No static Google service
account JSON keys are stored in this repository.

## Usage

A workload consumes a secret by creating an `ExternalSecret` that references the
`gcp-secret-manager` `ClusterSecretStore` and a key in Secret Manager. ESO then
materialises a Kubernetes `Secret` with that value.
