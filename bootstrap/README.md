# bootstrap

The **root App-of-Apps**. This holds the single ArgoCD `Application` that the
`infrastructure` repo applies once, right after ArgoCD is installed.

That root Application points ArgoCD at the rest of this repo (`platform/`, then
`catalog/`), using **sync waves** so add-ons and CRDs come up before the
resources that depend on them. After this one apply, all further changes are made
by committing to this repo, no manual `kubectl`/`argocd` steps.

## `root.yaml`

`root.yaml` is that single Application. The `infrastructure` repo applies it once
after ArgoCD is up:

    kubectl apply -n argocd -f bootstrap/root.yaml

It tracks `platform/` and includes only `*/application.yaml`, so each add-on is
onboarded by adding `platform/<add-on>/application.yaml` - never by editing this
file. Sync is automated (`prune` + `selfHeal`) - the `resources-finalizer` makes deletion cascade to the child Applications.
