# aws-oidc-assume

Assume an AWS IAM role using GitHub Actions OIDC. No long-lived credentials required.

The calling workflow must have `id-token: write` permission. See [docs/oidc-setup.md](../../docs/oidc-setup.md) for the IAM role and trust policy setup.

## Usage

```yaml
permissions:
  id-token: write
  contents: read

steps:
  - name: Assume AWS role
    id: aws
    uses: kernelpanic09/github-actions-platform/actions/aws-oidc-assume@v1
    with:
      role_arn: arn:aws:iam::123456789012:role/github-actions-deploy
      region: us-east-1

  - name: Use the credentials
    run: aws s3 ls
```

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `role_arn` | Yes | | IAM role ARN to assume |
| `region` | No | `us-east-1` | AWS region |
| `role_session_name` | No | `github-actions-<run_id>-<run_attempt>` | Session name |
| `role_duration_seconds` | No | `3600` | Session duration in seconds |

## Outputs

| Output | Description |
|---|---|
| `account_id` | AWS account ID of the assumed role |

## Why wrap configure-aws-credentials?

`aws-actions/configure-aws-credentials` does the heavy lifting. This wrapper adds:

1. A consistent default session name that includes `run_id` and `run_attempt` (useful for CloudTrail filtering)
2. The `account_id` output so callers don't have to call `aws sts get-caller-identity` themselves
3. A single place to update when the upstream action changes its interface
