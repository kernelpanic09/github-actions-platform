# Conventions

Naming conventions used across this library for inputs, outputs, secrets, and markers.

## Input naming

- Use `snake_case` for all input names.
- Boolean inputs are `true`/`false` strings (GitHub Actions doesn't have a native bool type for composite actions; reusable workflows use the `boolean` type).
- Version inputs are named `<tool>_version` and take exact semver strings (e.g. `1.8.5`, not `~1.8` or `>=1.8`).
- Path inputs are named `<thing>_path` or `<thing>_directory`.
- ARN inputs are named `<service>_role_arn`.

## Output naming

- `has_<thing>`: boolean string, `true` or `false`.
- `<thing>_id`: opaque identifier.
- `<thing>_uri`: full URI including scheme.
- `<thing>_digest`: content-addressed hash (e.g. `sha256:abc123...`).
- `<thing>_outputs`: JSON-encoded map.

## Secret naming

Workflows in this library don't declare secrets (they use OIDC instead of static credentials). If you're adding a workflow that genuinely needs a secret, name it `<SERVICE>_<PURPOSE>` in screaming snake case, e.g. `SLACK_WEBHOOK_URL`.

Don't name secrets after the specific tool that consumes them. `SLACK_WEBHOOK_URL` says what it is. `NOTIFY_SECRET` doesn't.

## PR comment markers

The `pr-comment` action uses a marker string to identify its comment. Markers should be unique per logical operation per PR. Format:

```
<workflow-or-action>-<scope>
```

Examples:

```
terraform-plan-infra/prod
terraform-plan-infra/staging
helm-deploy-payments-service-payments
trivy-image-myorg/myapp
ci-self-test-pr-comment
```

Including the scope (working directory, namespace, release name) is important when the same workflow runs multiple times in the same PR for different targets.

## GitHub environment names

Use lowercase with hyphens. Standard names: `development`, `staging`, `production`. For multi-region setups: `production-us-east-1`, `production-eu-west-1`.

GitHub environment protection rules (require reviewers, deployment branches) are configured in the repo settings, not in this library. The library just forwards the environment name to the job's `environment:` field.

## Version pinning

All `uses:` references in this library's own CI pin to a specific tag or SHA. In examples, we use `@v1` (major version tag) to show callers how to pin without re-pinning every minor update.

Never use `@main` in production calling workflows. `@main` is appropriate when testing changes to this library from a feature branch.

## Shell scripting style

- `set -euo pipefail` at the top of non-trivial scripts.
- Prefer `[[ ]]` over `[ ]` in bash.
- Quote all variable expansions: `"${VAR}"`.
- Use `SCREAMING_SNAKE_CASE` for environment variables and script-level constants.
- Use `lowercase` for local variables.
- Avoid subshells where a variable assignment works (`VAR=$(cmd)` is fine; avoid `$(echo "$VAR")`).

## YAML style

- 2-space indent, no tabs.
- Strings: unquoted when safe, single-quoted for literals with special chars, double-quoted when the string contains single quotes or variable expansions.
- `name:` on every step. Short and descriptive.
- `id:` only on steps whose outputs are referenced.
