# Contributing

## Commit messages

Commits follow the [Conventional Commits](https://www.conventionalcommits.org)
specification **and** must reference a GitHub issue:

    <type>(<optional scope>): #<issue> <description>

- **type**: `feat`, `fix`, `docs`, `style`, `refactor`, `perf`, `test`, `build`, `ci`, `chore`, `revert`
- Reference the issue in the subject `#42` or in a footer (`Refs #42`, `Closes #42`).
- No merge commits — the repository is rebase-only with linear history.

These rules are enforced in CI by commitlint
(`.commitlintrc.yml` + `.github/workflows/commitlint.yml`).

## Enable the local commit-msg hook (optional but recommended)

To get the same checks **before** you push, point Git at the tracked hooks
directory once per clone:

    git config core.hooksPath .githooks

`.githooks/commit-msg` is a dependency-free POSIX script and runs under Linux,
macOS, and Git for Windows' bundled bash. CI remains the authoritative gate.

## YAML & manifest conventions

These keep the `manifest-lint` CI (yamllint + kubeconform) green. Run both
locally before pushing.

### YAML style (enforced by `.yamllint`)

- Start every file with `---`.
- Indent with 2 spaces - never tabs.
- No trailing whitespace - end the file with a single newline.
- Keep lines ≤ 120 chars (over that is a warning - wrap where practical - long
  URLs may exceed).
- No duplicate keys in a mapping.
- Use `true`/`false` for booleans (not `yes`/`no`/`on`/`off`).
- Quote values that must stay strings but look like numbers/bools/versions
  (e.g. `"true"`, `"v1.2"`, `"0755"`).
- Files are stored LF (enforced by `.gitattributes`) - set your editor to LF.

### Kubernetes manifests (enforced by `kubeconform -strict`)

- Put real Kubernetes manifests only under `bootstrap/`, `platform/`,
  `catalog/`, `tenants/` - those are the dirs CI schema-checks. CI/config YAML
  lives elsewhere (it is still yamllint-checked).
- Set a correct `apiVersion` and `kind`, and include all required fields.
- `-strict` rejects unknown/extra fields: no typos in field names, no stray keys.
- Use correct types - integers for `replicas`/ports, not strings.
- Separate multiple documents in one file with `---`.
- CRDs are validated only if their schema is in the Datree catalog - otherwise
  kubeconform **skips** them (not validated). When you add custom CRDs/XRDs,
  add a `-schema-location` for their schemas so they are actually checked.

### Run locally

    yamllint -c .yamllint .
    kubeconform -strict -ignore-missing-schemas \
      -schema-location default \
      -schema-location 'https://raw.githubusercontent.com/datreeio/CRDs-catalog/main/{{.Group}}/{{.ResourceKind}}_{{.ResourceAPIVersion}}.json' \
      $(find bootstrap platform catalog tenants -type f \( -name '*.yaml' -o -name '*.yml' \))
