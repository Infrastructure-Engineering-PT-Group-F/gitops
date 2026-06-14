# bootstrap

The **root App-of-Apps**. This holds the single ArgoCD `Application` that the
`infrastructure` repo applies once, right after ArgoCD is installed.

That root Application points ArgoCD at the rest of this repo (`platform/`, then
`catalog/`), using **sync waves** so add-ons and CRDs come up before the
resources that depend on them. After this one apply, all further changes are made
by committing to this repo, no manual `kubectl`/`argocd` steps.
