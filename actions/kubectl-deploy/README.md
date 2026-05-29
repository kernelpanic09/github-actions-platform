# kubectl-deploy

Apply Kubernetes manifests and wait for any Deployments to finish rolling out.

This action assumes `kubectl` is already configured with a valid kubeconfig (either from a prior step that ran `aws eks update-kubeconfig` or from a kubeconfig written to `~/.kube/config`).

## Usage

```yaml
- name: Configure kubeconfig
  run: |
    aws eks update-kubeconfig --region us-east-1 --name my-cluster

- name: Deploy
  uses: kernelpanic09/github-actions-platform/actions/kubectl-deploy@v1
  with:
    manifests: "k8s/*.yml"
    namespace: my-app
    wait: "true"
    timeout: "5m"
```

Multiple patterns:

```yaml
- uses: kernelpanic09/github-actions-platform/actions/kubectl-deploy@v1
  with:
    manifests: "k8s/base/*.yml k8s/overlays/prod/*.yml"
    namespace: my-app
```

Dry run (validate without applying):

```yaml
- uses: kernelpanic09/github-actions-platform/actions/kubectl-deploy@v1
  with:
    manifests: "k8s/*.yml"
    namespace: my-app
    dry_run: "true"
```

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `manifests` | Yes | | Space-separated glob patterns for manifest files |
| `namespace` | No | `""` | Namespace override (passed as `--namespace`) |
| `wait` | No | `true` | Wait for Deployment rollouts |
| `timeout` | No | `5m` | Rollout wait timeout |
| `dry_run` | No | `false` | Run with `--dry-run=server` |
| `prune` | No | `false` | Pass `--prune` to remove unlisted resources |
| `prune_allowlist` | No | `""` | Comma-separated resource kinds for pruning |

## Outputs

| Output | Description |
|---|---|
| `applied_resources` | Newline-separated list of resources that were applied |

## Notes

The rollout wait step extracts Deployment names from the manifest YAML files by looking for `kind: Deployment` followed by a `name:` field. It doesn't parse full YAML, so it works for typical single-document manifests. For complex multi-document files with deeply nested structures, you may want to handle the rollout wait separately.
