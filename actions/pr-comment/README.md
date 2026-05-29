# pr-comment

Post a markdown comment to the current pull request, or update the existing comment if one already exists. Uses a hidden HTML marker so pushing to a PR branch doesn't spam it with duplicate plan comments.

## Usage

```yaml
permissions:
  pull-requests: write

steps:
  - name: Post plan comment
    uses: kernelpanic09/github-actions-platform/actions/pr-comment@v1
    with:
      marker: "terraform-plan-infra/prod"
      body: |
        ### Terraform plan

        3 to add, 1 to change, 0 to destroy.
```

On the first push to the PR branch this creates a new comment. On subsequent pushes it finds and overwrites the same comment. The marker is embedded as `<!-- actions-platform-comment:<marker> -->` at the top of the comment body. GitHub renders that as invisible.

If you have multiple Terraform root modules posting plan comments to the same PR, give each one a distinct marker (e.g. `terraform-plan-infra/prod`, `terraform-plan-infra/staging`).

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `marker` | Yes | | Unique slug to identify the comment |
| `body` | Yes | | Markdown content |
| `token` | No | `${{ github.token }}` | GitHub token with `pull-requests: write` |

## Outputs

| Output | Description |
|---|---|
| `comment_id` | GitHub comment ID of the created or updated comment |

## Notes

This action only works inside a pull request event context. If called from a `push` or `workflow_dispatch` event where there's no PR number, the GitHub API call will fail. Guard it with `if: github.event_name == 'pull_request'` when the calling workflow runs on multiple trigger types.
