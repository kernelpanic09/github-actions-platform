# Composite action reference

All actions are called with `uses: kernelpanic09/github-actions-platform/actions/<name>@v1`.

---

## setup-tooling

Install terraform, helm, kubectl, and trivy at pinned versions. Binaries are cached by `<tool>-<version>-<os>-<arch>` so repeat runs don't re-download.

Only tools with a non-empty version input are installed.

```yaml
- uses: kernelpanic09/github-actions-platform/actions/setup-tooling@v1
  with:
    terraform_version: "1.8.5"
    helm_version: "3.15.2"
    kubectl_version: "1.30.2"
    trivy_version: "0.69.3"
```

### Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `terraform_version` | No | `""` | Terraform version (e.g. `1.8.5`) |
| `helm_version` | No | `""` | Helm version (e.g. `3.15.2`) |
| `kubectl_version` | No | `""` | kubectl version (e.g. `1.30.2`) |
| `trivy_version` | No | `""` | Trivy version (e.g. `0.52.2`) |

### Outputs

| Output | Description |
|---|---|
| `terraform_path` | Absolute path to the terraform binary |
| `helm_path` | Absolute path to the helm binary |

---

## aws-oidc-assume

Assume an AWS IAM role via OIDC. Requires `id-token: write` permission on the calling job.

```yaml
permissions:
  id-token: write
  contents: read

steps:
  - uses: kernelpanic09/github-actions-platform/actions/aws-oidc-assume@v1
    with:
      role_arn: arn:aws:iam::123456789012:role/github-actions-deploy
      region: us-east-1
```

### Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `role_arn` | Yes | | IAM role ARN to assume |
| `region` | No | `us-east-1` | AWS region |
| `role_session_name` | No | `github-actions-<run_id>-<run_attempt>` | Session name |
| `role_duration_seconds` | No | `3600` | Session duration in seconds |

### Outputs

| Output | Description |
|---|---|
| `account_id` | AWS account ID of the assumed role |

See [oidc-setup.md](oidc-setup.md) for the IAM trust policy.

---

## pr-comment

Post a markdown comment on the current PR, or update it if a comment with the same marker already exists. Prevents duplicate comments on repeated pushes to the same branch.

```yaml
- uses: kernelpanic09/github-actions-platform/actions/pr-comment@v1
  with:
    marker: "terraform-plan-infra/prod"
    body: |
      ### Terraform plan

      3 to add, 0 to change, 0 to destroy.
```

### Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `marker` | Yes | | Unique slug to identify the comment |
| `body` | Yes | | Markdown comment body |
| `token` | No | `${{ github.token }}` | GitHub token |

### Outputs

| Output | Description |
|---|---|
| `comment_id` | ID of the created or updated comment |

The marker is embedded as `<!-- actions-platform-comment:<marker> -->` at the top of the comment. GitHub renders it as invisible.

Only works in pull request event contexts. Guard with `if: github.event_name == 'pull_request'` when needed.

---

## terraform-plan-summary

Parse a terraform plan JSON file and produce a markdown summary with resource counts and a collapsible diff block.

```yaml
- name: Generate plan summary
  id: summary
  uses: kernelpanic09/github-actions-platform/actions/terraform-plan-summary@v1
  with:
    plan_file_path: ./infra/tfplan.json
    plan_text_path: ./infra/plan.txt

- name: Post PR comment
  if: github.event_name == 'pull_request'
  uses: kernelpanic09/github-actions-platform/actions/pr-comment@v1
  with:
    marker: "terraform-plan-infra"
    body: ${{ steps.summary.outputs.summary }}
```

### Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `plan_file_path` | Yes | | Path to `tfplan.json` from `terraform show -json` |
| `plan_text_path` | No | `""` | Path to raw plan text for the detail block |
| `fmt_failed` | No | `false` | Prepend a fmt warning |

### Outputs

| Output | Description |
|---|---|
| `summary` | Markdown summary string |
| `has_changes` | `true` if the plan has any changes |
| `added` | Number of resources to be created |
| `changed` | Number of resources to be updated |
| `destroyed` | Number of resources to be destroyed |

If `plan_file_path` doesn't exist, `has_changes` is `false` and a "no changes" message is returned.

---

## kubectl-deploy

Apply Kubernetes manifests and wait for any Deployments to finish rolling out.

```yaml
- uses: kernelpanic09/github-actions-platform/actions/kubectl-deploy@v1
  with:
    manifests: "k8s/*.yml"
    namespace: my-app
    wait: "true"
    timeout: "5m"
```

### Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `manifests` | Yes | | Space-separated glob patterns for manifest files |
| `namespace` | No | `""` | Namespace override |
| `wait` | No | `true` | Wait for Deployment rollouts |
| `timeout` | No | `5m` | Rollout wait timeout |
| `dry_run` | No | `false` | Run with `--dry-run=server` |
| `prune` | No | `false` | Pass `--prune` |
| `prune_allowlist` | No | `""` | Comma-separated resource kinds for pruning |

### Outputs

| Output | Description |
|---|---|
| `applied_resources` | Newline-separated list of applied resources |

Assumes `kubectl` is already configured. Use a prior step to run `aws eks update-kubeconfig` or write a kubeconfig to `~/.kube/config`.
