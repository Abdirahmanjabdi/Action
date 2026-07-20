# CI/CD with GitHub Actions

CI/CD module of my DevOps learning path: two GitHub Actions pipelines, plus the
debugging notes from getting them green.

- **CI** - Terraform `init` / `validate` / `fmt -check` on every push to `main`
- **CD** - Docker image build and push to DockerHub, tagged with the commit SHA

## Full documentation

**[cicd/README.md](./cicd/README.md)** - what was built, the pipeline YAML, the
four failures I debugged and how I fixed each one, and what I would improve next.

| | |
|---|---|
| [Task 1 - CI](./cicd/task-1) | Terraform linting and validation |
| [Task 2 - CD](./cicd/task-2) | Docker build and push to DockerHub |
| [Workflows](./.github/workflows) | the pipeline definitions |
| [Screenshots](./cicd/screenshots) | pipeline run evidence |

## Repository layout

```
.github/workflows/    task-1-ci.yml, task-2-cd.yaml
cicd/                 documentation, Terraform config, notes, screenshots
matrix/               Python package and unit tests
app.py, Dockerfile    the application and its image definition
```

## Highlights

- Images are tagged `hollowy4t/action:<short-sha>` so every build is traceable
  to the commit that produced it, and rollback is a real option.
- CI runs Terraform validation without cloud credentials via
  `terraform init -backend=false`.
- Registry credentials come from repository secrets and are masked in logs.
- Four real pipeline failures are documented with root cause and fix, including
  an invalid Docker tag, a Python import-path break, a stdlib package in
  `requirements.txt`, and YAML coercing `3.10` into `3.1`.
