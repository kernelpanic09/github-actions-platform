# setup-tooling

Install terraform, helm, kubectl, and trivy at pinned versions. Only installs what you ask for. Binaries are cached by version so subsequent runs skip the download.

## Usage

```yaml
- name: Install platform tools
  uses: kernelpanic09/github-actions-platform/actions/setup-tooling@v1
  with:
    terraform_version: "1.8.5"
    helm_version: "3.15.2"
    kubectl_version: "1.30.2"
    trivy_version: "0.69.3"
```

You don't need to install all four. Any input left empty is skipped:

```yaml
- name: Install only terraform
  uses: kernelpanic09/github-actions-platform/actions/setup-tooling@v1
  with:
    terraform_version: "1.8.5"
```

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `terraform_version` | No | `""` | Terraform version. Leave empty to skip. |
| `helm_version` | No | `""` | Helm version. Leave empty to skip. |
| `kubectl_version` | No | `""` | kubectl version. Leave empty to skip. |
| `trivy_version` | No | `""` | Trivy version. Leave empty to skip. |

## Outputs

| Output | Description |
|---|---|
| `terraform_path` | Absolute path to the terraform binary (empty if not installed) |
| `helm_path` | Absolute path to the helm binary (empty if not installed) |

## Caching

Each binary is cached independently using `actions/cache` with a key of `<tool>-<version>-<os>-<arch>`. A version bump busts the cache for that tool only.

Binaries are installed to `~/.local/bin` which is added to `$GITHUB_PATH`.

## Notes

Versions must be exact (no `>= 1.8` style ranges). Pin versions in your calling workflow and update them deliberately. This makes CI reproducible.
