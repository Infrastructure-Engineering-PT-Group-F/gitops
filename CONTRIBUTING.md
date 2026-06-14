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
