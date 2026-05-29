# Contributing

Pull requests are welcome. This is a personal portfolio project, so for anything bigger than a bug fix or a tweak to an existing workflow, please open an issue first to discuss the approach. Breaking changes to workflow interfaces affect every caller, so those need a bit more care.

## Adding a new reusable workflow

1. Create `.github/workflows/<name>.yml` with `on: workflow_call`.
2. Document every input and output in the workflow file itself (use `description:` fields) and in `docs/workflows.md`.
3. Add a usage example under `examples/`.
4. Add a row to the table in `README.md`.
5. If the workflow introduces a new AWS permission or new required secret, update `docs/oidc-setup.md` and `docs/conventions.md`.

## Adding a new composite action

1. Create `actions/<name>/action.yml` with `runs.using: composite`.
2. Create `actions/<name>/README.md` with a description and example.
3. Add a row to the table in `README.md` and a section in `docs/actions.md`.
4. Add a test step to `.github/workflows/ci-self-test.yml`.

## action.yml requirements

Every composite action must have:

- `name`, `description`, `author: kernelpanic09`
- `branding` with `icon` and `color` (for GitHub Marketplace display)
- Every input must have `description` and either `required: true` or a `default`
- Every output must have `description`
- Steps must include `shell: bash` on every `run` step

## Workflow requirements

Every reusable workflow (`on: workflow_call`) must:

- Declare all inputs and secrets explicitly (no `secrets: inherit` without reason)
- Include a `permissions:` block at the job level, not just at the top level
- Use `GITHUB_TOKEN` or OIDC for auth, never long-lived static credentials
- Pass `id-token: write` only to jobs that actually call OIDC

## Testing approach

**Composite actions** are tested in `ci-self-test.yml` by calling them directly in a job. Tests check that outputs are set and that the action exits 0 on valid input.

**Reusable workflows** are harder to test directly because `workflow_call` requires a caller. The approach here is:

- `docs-check.yml` validates that every `action.yml` file is valid YAML and has required fields
- `ci-self-test.yml` uses `workflow_dispatch` to trigger each reusable workflow with test inputs where possible
- For workflows that need real AWS credentials, there's a separate `integration-test` environment in this repo with a restricted test IAM role

## Versioning and breaking changes

This repo uses [release-please](https://github.com/googleapis/release-please) to manage changelogs and tags. Commit messages follow [Conventional Commits](https://www.conventionalcommits.org/).

- `fix:` commits bump the patch version
- `feat:` commits bump the minor version
- `feat!:` or any commit with `BREAKING CHANGE:` in the footer bumps the major version

When making a breaking change to a workflow or action interface (removing an input, changing an output name, changing default behavior), bump the major version and note it clearly in the changelog.

Callers that pin `@v1` are protected from major version changes. Callers on `@main` get everything immediately.

## Code style

- YAML: 2-space indent, no tabs
- Shell steps: `set -euo pipefail` where the logic is non-trivial
- Comments explain why, not what
- Don't repeat GitHub's own documentation in comments; link to it instead
