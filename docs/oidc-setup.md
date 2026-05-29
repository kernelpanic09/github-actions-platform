# AWS OIDC setup

GitHub Actions can authenticate to AWS without storing long-lived access keys. GitHub issues a short-lived OIDC token per workflow run, and AWS validates it against a trust policy you configure on an IAM role.

## 1. Add the GitHub OIDC provider to your AWS account

This only needs to happen once per AWS account.

```bash
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1 \
  --client-id-list sts.amazonaws.com
```

If you're using Terraform to manage IAM:

```hcl
resource "aws_iam_openid_connect_provider" "github" {
  url = "https://token.actions.githubusercontent.com"

  client_id_list = ["sts.amazonaws.com"]

  # GitHub's OIDC thumbprint. Verify this against GitHub's docs if pinning matters for you.
  thumbprint_list = ["6938fd4d98bab03faadb97b34396831e3780aea1"]
}
```

## 2. Create an IAM role with a trust policy scoped to your repo

The trust policy controls which GitHub repositories and branches can assume the role.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:your-org/your-repo:*"
        }
      }
    }
  ]
}
```

To scope to a specific branch only (e.g. main):

```json
"token.actions.githubusercontent.com:sub": "repo:your-org/your-repo:ref:refs/heads/main"
```

To scope to a specific GitHub environment (recommended for production roles):

```json
"token.actions.githubusercontent.com:sub": "repo:your-org/your-repo:environment:production"
```

## 3. Attach a permission policy to the role

Attach only the permissions the workflow needs. For Terraform managing S3 and EC2:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:ListBucket",
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:DeleteItem"
      ],
      "Resource": [
        "arn:aws:s3:::my-terraform-state/*",
        "arn:aws:s3:::my-terraform-state",
        "arn:aws:dynamodb:us-east-1:123456789012:table/terraform-state-lock"
      ]
    }
  ]
}
```

## 4. Add id-token: write to your calling workflow

```yaml
permissions:
  id-token: write
  contents: read
```

Without `id-token: write`, GitHub won't issue an OIDC token and the assume-role step will fail with a credentials error.

## 5. Use the aws-oidc-assume action

```yaml
- uses: kernelpanic09/github-actions-platform/actions/aws-oidc-assume@v1
  with:
    role_arn: arn:aws:iam::123456789012:role/github-actions-terraform
    region: us-east-1
```

## Separate roles per environment

Don't use one role for everything. Common pattern:

| Role | Trust condition | Permissions |
|---|---|---|
| `github-actions-plan` | Any branch | Read-only: S3 state read, describe resources |
| `github-actions-apply-staging` | `main` branch | Write: apply to staging |
| `github-actions-apply-prod` | `environment:production` | Write: apply to production |
| `github-actions-ecr-push` | Any branch in the app repo | ECR push to specific repo |
| `github-actions-eks-deploy` | `main` or `environment:*` | EKS access to specific clusters |

The `environment:production` trust condition pairs with GitHub environment protection rules (require reviewers) to enforce manual approval before production applies.

## Debugging

If OIDC auth fails, check:

1. The OIDC provider exists in the right AWS account
2. The `sub` claim in the trust policy matches the format GitHub sends (`repo:<org>/<repo>:ref:refs/heads/<branch>` or `repo:<org>/<repo>:environment:<name>`)
3. The calling job has `id-token: write` in its `permissions:` block, not just the top-level `permissions:`
4. The region in the action input matches the region where the role lives

GitHub's OIDC token issuer: `https://token.actions.githubusercontent.com`
AWS docs: https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_oidc.html
