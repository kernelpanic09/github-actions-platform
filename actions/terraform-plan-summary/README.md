# terraform-plan-summary

Parse a terraform plan JSON file and produce a markdown summary with resource counts and a collapsible details block. Designed to pipe into the `pr-comment` action.

## Usage

```yaml
- name: Plan
  run: |
    terraform plan -out=tfplan.binary -no-color 2>&1 | tee plan.txt
  working-directory: ./infra

- name: Convert to JSON
  run: terraform show -json tfplan.binary > tfplan.json
  working-directory: ./infra

- name: Generate summary
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

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `plan_file_path` | Yes | | Path to the tfplan JSON file from `terraform show -json` |
| `plan_text_path` | No | `""` | Path to raw plan text output. Used for the collapsible detail block. |
| `fmt_failed` | No | `false` | Prepend a fmt warning to the summary |

## Outputs

| Output | Description |
|---|---|
| `summary` | Markdown summary string |
| `has_changes` | `true` if the plan has any changes |
| `added` | Number of resources to be created |
| `changed` | Number of resources to be updated |
| `destroyed` | Number of resources to be destroyed |

## Notes

If `plan_file_path` doesn't exist (because the plan exited 0 with no changes and you only write the JSON file when there are changes), the action treats this as a no-changes plan and sets `has_changes=false`.

The changed-resources table is capped at 50 entries and the full plan text at 200 lines to stay under GitHub's comment character limit.
