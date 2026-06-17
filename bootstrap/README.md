# bootstrap

This directory now holds the platform add-on App-of-Apps configuration that the
`infrastructure` repository installs after ArgoCD is bootstrapped.

The root Application is no longer stored here. It is now owned by Terraform in
the `infrastructure` repository and installed via the `argocd-apps` Helm chart.

`bootstrap/` remains the entrypoint for the add-on hierarchy under `platform/`,
with each add-on represented by its own ArgoCD `Application` manifest.
