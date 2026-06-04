# Examples

Each directory is a self-contained sample repo showing how to consume the
reusable workflows from this library. Copy the `.github/workflows/` file into
your own repo and adjust the inputs.

All examples pin `@v1`. Use `@main` only when testing unreleased changes.

| Example | Demonstrates | Reusable workflows used |
|---------|--------------|-------------------------|
| [docker-app](docker-app/) | Build a container image, scan it, and push to a registry | `docker-build.yml`, `trivy-scan.yml` |
| [helm-microservice](helm-microservice/) | Build, scan, and deploy a service to Kubernetes via Helm | `docker-build.yml`, `helm-deploy.yml`, `trivy-scan.yml` |
| [terraform-pipeline](terraform-pipeline/) | Plan-on-PR / apply-on-merge Terraform pipeline with OIDC | `terraform.yml` |
| [full-platform](full-platform/) | End-to-end: Terraform infra, image build/scan, Helm deploy, release automation | `terraform.yml`, `docker-build.yml`, `trivy-scan.yml`, `helm-deploy.yml`, `release-please.yml` |

## Using an example

1. Copy the workflow file(s) from the example's `.github/workflows/` into your repo.
2. Set the required secrets/variables (most use AWS OIDC - see the repo
   [OIDC setup](../docs/oidc-setup.md)).
3. Adjust inputs (image name, cluster, working directory) to match your project.

Input/output reference for each workflow lives in [docs/workflows.md](../docs/workflows.md);
composite actions are documented in [docs/actions.md](../docs/actions.md).
