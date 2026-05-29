# Reusable workflow reference

All workflows are called with `uses: kernelpanic09/github-actions-platform/.github/workflows/<name>.yml@v1`.

---

## terraform.yml

Runs `terraform fmt`, `init`, `validate`, `plan`, and optionally `apply`. Posts a plan summary as a PR comment on pull requests.

### Inputs

| Input | Type | Required | Default | Description |
|---|---|---|---|---|
| `working_directory` | string | Yes | | Path to the Terraform root module, relative to repo root |
| `terraform_version` | string | No | `1.8.5` | Terraform version to install |
| `aws_role_arn` | string | Yes | | IAM role ARN to assume via OIDC |
| `aws_region` | string | No | `us-east-1` | AWS region |
| `environment` | string | No | `""` | GitHub environment name (enables approval gates on apply) |
| `apply` | boolean | No | `false` | Whether to run terraform apply |

### Outputs

| Output | Description |
|---|---|
| `plan_changes` | `true` if the plan has any changes |
| `apply_outputs` | JSON-encoded terraform output map (only set when apply runs) |

### Required permissions (calling workflow)

```yaml
permissions:
  id-token: write
  contents: read
  pull-requests: write
```

### Notes

- The plan binary and JSON are not persisted between jobs. The apply job re-runs `init` and uses `-auto-approve`.
- The plan comment is posted by the `pr-comment` action using the `working_directory` as the marker, so multiple root modules in the same repo each get their own comment.
- Passing `apply: ${{ github.event_name == 'push' }}` is the standard pattern for plan-on-PR, apply-on-merge.

---

## docker-build.yml

Builds a Docker image with Buildx, pushes to ECR or GHCR, and optionally signs with cosign keyless signing.

### Inputs

| Input | Type | Required | Default | Description |
|---|---|---|---|---|
| `image_name` | string | Yes | | Image name without registry prefix (e.g. `myorg/myapp`) |
| `dockerfile` | string | No | `Dockerfile` | Path to Dockerfile, relative to context |
| `context` | string | No | `.` | Docker build context |
| `platforms` | string | No | `linux/amd64` | Comma-separated target platforms |
| `registry` | string | No | `ghcr` | Target registry: `ecr` or `ghcr` |
| `aws_role_arn` | string | No | `""` | IAM role ARN (required when `registry=ecr`) |
| `aws_region` | string | No | `us-east-1` | AWS region for ECR |
| `sign` | boolean | No | `true` | Sign with cosign keyless signing |
| `build_args` | string | No | `""` | Newline-separated `KEY=VALUE` build args |

### Outputs

| Output | Description |
|---|---|
| `image_digest` | SHA256 digest of the pushed image |
| `image_uri` | Full image URI including digest |

### Required permissions

```yaml
permissions:
  id-token: write
  contents: read
  packages: write  # only for GHCR
```

### Tagging strategy

| Trigger | Tags applied |
|---|---|
| Any push or PR | `sha-<short-sha>`, `<branch-name>` |
| Push to default branch | + `latest` |
| Push to version tag `v*.*.*` | + `<tag>` |

### Notes

- Layer cache uses `type=gha` (GitHub Actions cache). Cache is per-runner and scoped to the branch.
- Cosign signs the specific digest, not a tag, so the signature stays valid even after re-tagging.

---

## helm-deploy.yml

Runs `helm lint`, `helm template`, and `helm upgrade --install`. Optionally waits for rollout and rolls back automatically on failure.

### Inputs

| Input | Type | Required | Default | Description |
|---|---|---|---|---|
| `chart_path` | string | Yes | | Path to the Helm chart directory |
| `release_name` | string | Yes | | Helm release name |
| `namespace` | string | Yes | | Kubernetes namespace |
| `cluster_name` | string | No | `""` | EKS cluster name (used with `aws_role_arn`) |
| `aws_role_arn` | string | No | `""` | IAM role ARN for EKS auth |
| `aws_region` | string | No | `us-east-1` | AWS region for EKS |
| `kubeconfig_secret` | string | No | `""` | Repository secret name containing a kubeconfig |
| `helm_version` | string | No | `3.15.2` | Helm version |
| `values_files` | string | No | `""` | Newline-separated values file paths |
| `set_values` | string | No | `""` | Newline-separated `--set` overrides |
| `image_tag` | string | No | `""` | Sets `image.tag` via `--set` |
| `wait` | boolean | No | `true` | Wait for resources to be ready |
| `timeout` | string | No | `5m` | Helm wait timeout |
| `atomic` | boolean | No | `true` | Roll back automatically on failure |
| `dry_run` | boolean | No | `false` | Run with `--dry-run` |

### Outputs

| Output | Description |
|---|---|
| `release_revision` | Helm release revision after the upgrade |

### Required permissions

```yaml
permissions:
  id-token: write
  contents: read
  pull-requests: write
```

### Notes

- Either `aws_role_arn + cluster_name` (for EKS) or `kubeconfig_secret` must be provided for cluster access.
- History is capped at 10 revisions (`--history-max 10`).
- The namespace is created automatically if it doesn't exist.

---

## trivy-scan.yml

Runs Trivy to scan a container image, filesystem, or IaC config. Uploads SARIF results to the GitHub Security tab and optionally posts a findings summary to the PR.

### Inputs

| Input | Type | Required | Default | Description |
|---|---|---|---|---|
| `scan_type` | string | No | `fs` | Scan type: `image`, `fs`, or `config` |
| `target` | string | No | `.` | What to scan (image URI, or path for fs/config) |
| `trivy_version` | string | No | `0.52.2` | Trivy version |
| `severity` | string | No | `HIGH,CRITICAL` | Comma-separated severity levels |
| `exit_code` | number | No | `1` | Exit code when vulnerabilities found (0 to report-only) |
| `post_pr_comment` | boolean | No | `true` | Post findings summary to PR |
| `ignore_unfixed` | boolean | No | `false` | Skip vulnerabilities with no available fix |

### Required permissions

```yaml
permissions:
  security-events: write
  contents: read
  pull-requests: write  # only if post_pr_comment=true
```

### Notes

- SARIF is always uploaded regardless of `exit_code`, so findings appear in the Security tab even when you pass `exit_code: 0`.
- The job runs Trivy twice: once for SARIF (always exit 0), once to enforce the `exit_code` setting. The table output for the PR comment is a third invocation.

---

## release-please.yml

Wraps `googleapis/release-please-action` to automate changelog generation and GitHub releases from conventional commits.

### Inputs

| Input | Type | Required | Default | Description |
|---|---|---|---|---|
| `release_type` | string | No | `simple` | Release type: `node`, `python`, `terraform`, `go`, `simple` |
| `package_name` | string | No | `""` | Package name in changelog and release notes |
| `config_file` | string | No | `""` | Path to `release-please-config.json` |
| `manifest_file` | string | No | `""` | Path to `.release-please-manifest.json` |
| `target_branch` | string | No | `main` | Branch to open release PRs against |

### Outputs

| Output | Description |
|---|---|
| `release_created` | `true` if a new release was created |
| `tag_name` | Git tag of the new release (e.g. `v1.2.3`) |
| `version` | Version without `v` prefix |
| `upload_url` | GitHub Release upload URL for attaching assets |

### Required permissions

```yaml
permissions:
  contents: write
  pull-requests: write
```

### Notes

- This workflow should only run on pushes to the default branch.
- For monorepos with multiple releasable components, use `config_file` and `manifest_file` instead of `release_type`.
- Use `needs: release` with `if: needs.release.outputs.release_created == 'true'` to gate asset upload or deployment jobs on a new release.
