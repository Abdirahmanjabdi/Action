# Task 1 - Continuous Integration (Terraform lint and validate)

Automated quality gates that run on every push to `main`, defined in
[`.github/workflows/task-1-ci.yml`](../../.github/workflows/task-1-ci.yml).

## What runs

| Step | Command | Fails the build when |
|------|---------|----------------------|
| Init | `terraform init -backend=false` | provider or module resolution breaks |
| Validate | `terraform validate` | the configuration has syntax or type errors |
| Format check | `terraform fmt -check` | any file is not canonically formatted |

`-backend=false` keeps CI credential-free: it initialises the working directory
without configuring or contacting remote state, so no cloud secrets are needed
just to validate syntax.

## About the Terraform in this folder

[`main.tf`](./main.tf) is a deliberately minimal, resource-free configuration.
It exists so the CI job has something real to initialise, validate and format
check without creating infrastructure or requiring credentials.

```hcl
terraform {
  required_version = ">= 1.0"
}

locals {
  portfolio_placeholder = "This is a safe placeholder config for CI validation"
}
```

Replace it with real Terraform when there is infrastructure to manage; the
pipeline needs no changes to start validating it.

## Known gap

The brief for Task 1 also lists **unit tests**. The Python tests in
[`matrix/`](../../matrix) are written and importable, but no workflow currently
executes them because this CI workflow was replaced by the Terraform job. See
the improvements list in the [module README](../README.md#improvements-i-would-make-next).
