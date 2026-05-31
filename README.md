# github-actions-platform

[![CI](https://github.com/kernelpanic09/github-actions-platform/actions/workflows/ci-self-test.yml/badge.svg)](https://github.com/kernelpanic09/github-actions-platform/actions/workflows/ci-self-test.yml)
[![License: MIT](https://img.shields.io/github/license/kernelpanic09/github-actions-platform)](LICENSE)
[![Release](https://img.shields.io/github/v/release/kernelpanic09/github-actions-platform?include_prereleases&sort=semver)](https://github.com/kernelpanic09/github-actions-platform/releases)
[![Last commit](https://img.shields.io/github/last-commit/kernelpanic09/github-actions-platform)](https://github.com/kernelpanic09/github-actions-platform/commits)

Every team writes the same CI/CD boilerplate. This is a small library of reusable GitHub Actions workflows and composite actions for the common platform tasks: Terraform plan/apply, Docker build and push, Helm deploy, container scanning, and release automation.

The goal is a single place to fix things once. When Trivy adds a new flag, when cosign changes its signing flow, when Terraform releases a new version with a different init flag, you fix it here and all callers get the update on their next run.

---

## Quick start

### Using the Terraform workflow

In your infra repo, create `.github/workflows/terraform.yml`:

```yaml
name: Terraform

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

permissions:
  id-token: write      # OIDC
  contents: read
  pull-requests: write # PR comments

jobs:
  terraform:
    uses: kernelpanic09/github-actions-platform/.github/workflows/terraform.yml@v1
    with:
      working_directory: ./infra/prod
      terraform_version: "1.8.5"
      aws_role_arn: arn:aws:iam::123456789012:role/github-actions-terraform
      aws_region: us-east-1
      environment: production
      apply: ${{ github.ref == 'refs/heads/main' }}
```

On a pull request this posts a plan summary as a PR comment and updates it on each push to the branch. On merge to main with `apply: true`, it runs `terraform apply` inside the `production` GitHub environment (so approval gates apply).

See [docs/oidc-setup.md](docs/oidc-setup.md) for the IAM role and trust policy setup.

---

## Reusable workflows

Call these with `uses: kernelpanic09/github-actions-platform/.github/workflows/<name>.yml@v1`.

| Workflow | Description |
|---|---|
| [terraform.yml](.github/workflows/terraform.yml) | fmt check, init, validate, plan with PR comment, apply on merge |
| [docker-build.yml](.github/workflows/docker-build.yml) | Multi-platform build, push to ECR or GHCR, cosign signing |
| [helm-deploy.yml](.github/workflows/helm-deploy.yml) | Helm lint, template, install/upgrade with rollout wait |
| [trivy-scan.yml](.github/workflows/trivy-scan.yml) | Container and IaC scanning, SARIF upload to Security tab |
| [release-please.yml](.github/workflows/release-please.yml) | Conventional commits to changelogs and GitHub releases |

Full input/output reference: [docs/workflows.md](docs/workflows.md)

## Composite actions

Call these with `uses: kernelpanic09/github-actions-platform/actions/<name>@v1`.

| Action | Description |
|---|---|
| [setup-tooling](actions/setup-tooling/) | Install terraform, helm, kubectl, trivy with version caching |
| [aws-oidc-assume](actions/aws-oidc-assume/) | Assume an AWS role via OIDC and export credentials |
| [pr-comment](actions/pr-comment/) | Post or update a PR comment by hidden marker (no spam) |
| [terraform-plan-summary](actions/terraform-plan-summary/) | Parse a tfplan JSON and produce a markdown summary |
| [kubectl-deploy](actions/kubectl-deploy/) | kubectl apply with rollout wait for any Deployments found |

Full input/output reference: [docs/actions.md](docs/actions.md)

---

## Required permissions

Calling workflows must declare the permissions they need. The most common combinations:

```yaml
# Terraform workflow (OIDC + PR comments)
permissions:
  id-token: write
  contents: read
  pull-requests: write

# Docker build to ECR (OIDC only)
permissions:
  id-token: write
  contents: read
  packages: write  # only needed for GHCR

# Trivy scan (SARIF upload)
permissions:
  security-events: write
  contents: read
```

Reusable workflows inherit the calling workflow's permissions. Don't grant more than you need.

## Versioning

Pin to a tag in production:

```yaml
uses: kernelpanic09/github-actions-platform/.github/workflows/terraform.yml@v1
```

Use `@main` when testing changes to this library. Tags follow semver. Breaking changes bump the major version.

## OIDC setup

See [docs/oidc-setup.md](docs/oidc-setup.md) for how to create the AWS OIDC identity provider and IAM role. The short version: GitHub's OIDC provider is `token.actions.githubusercontent.com` and the trust policy should scope to your org and repo.

## Input and secret naming conventions

See [docs/conventions.md](docs/conventions.md).

## Testing changes

The [ci-self-test workflow](.github/workflows/ci-self-test.yml) runs on every PR to this repo. It exercises each composite action in isolation and does a dry-run of the reusable workflows using workflow dispatch.

To test a new action locally before opening a PR, use [act](https://github.com/nektos/act):

```bash
act workflow_dispatch -W .github/workflows/ci-self-test.yml
```

## Related Projects

- [terraform-aws-modules](https://github.com/kernelpanic09/terraform-aws-modules) — pairs naturally with the `terraform.yml` reusable workflow here. The `iam-roles` module produces a GitHub Actions OIDC role, and this library handles the plan/apply pipeline that consumes it.
- [mcp-server-aws](https://github.com/kernelpanic09/mcp-server-aws) — uses `release-please.yml` from this library for its release automation, so changelogs and tags are handled consistently.

---

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).
