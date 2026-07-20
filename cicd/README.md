# CI/CD Module - GitHub Actions

Two pipelines built with GitHub Actions: a **CI** workflow that lints and validates infrastructure code on every push, and a **CD** workflow that builds a Docker image and pushes it to DockerHub, tagged with the commit SHA.

The interesting part of this module was not writing the YAML. It was debugging why it failed, four separate times, for four completely different reasons. Those are all documented in [Issues I hit and how I solved them](#issues-i-hit-and-how-i-solved-them).

---

## Repository layout

```
Action/
  .github/workflows/
    task-1-ci.yml        CI: Terraform fmt / init / validate
    task-2-cd.yaml       CD: Docker build + push to DockerHub
  cicd/
    task-1/              Terraform config the CI job validates
    task-2/              CD documentation
    notes/               YAML syntax learning notes
    screenshots/         pipeline run evidence
    README.md            this file
  matrix/                Python package + unit tests (hello.py, test_hello.py)
  app.py                 application entrypoint baked into the image
  Dockerfile             image definition
```

---

## Task 1 - Continuous Integration

**File:** [`.github/workflows/task-1-ci.yml`](../.github/workflows/task-1-ci.yml)

Runs Terraform quality gates against [`cicd/task-1`](./task-1) on every push to `main`.

```yaml
name: CI - Terraform Linting and Validation

on:
  push:
    branches: [ main ]

jobs:
  terraform:
    name: Terraform Linting and Validation
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: ./cicd/task-1
    steps:
      - name: Checkout code
        uses: actions/checkout@v2
      - name: Set up Terraform
        uses: hashicorp/setup-terraform@v1
        with:
          terraform_version: 1.5.7
      - name: Terraform Init
        run: terraform init -backend=false
      - name: Terraform Validate
        id: validate
        run: terraform validate
      - name: Terraform Format Check
        run: terraform fmt -check
        continue-on-error: false
```

What each gate does:

| Step | Purpose |
|------|---------|
| `terraform init -backend=false` | Initialise without touching remote state, so CI never needs cloud credentials |
| `terraform validate` | Catch syntax and type errors before they reach a plan |
| `terraform fmt -check` | Fail the build on unformatted code, keeping the diff clean |

`continue-on-error: false` is explicit on the format check: formatting drift **fails** the pipeline rather than being a warning.

---

## Task 2 - Continuous Deployment

**File:** [`.github/workflows/task-2-cd.yaml`](../.github/workflows/task-2-cd.yaml)

Builds the image and pushes it to DockerHub on every push, tagged with the short commit SHA so every image is traceable to the exact commit that produced it.

```yaml
name: Y4t CI/CD workflow automating image push to dockerhub
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
    steps:
      - name: Checkout code
        uses: actions/checkout@v2

      - name: Log in to DockerHub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Build and push Docker image
        run: |
          registery="hollowy4t"
          sha=$(git rev-parse --short HEAD)
          primary_tag="${registery}/action:${sha}"
          docker build -t $primary_tag -f Dockerfile .
          docker push $primary_tag
```

Design decisions worth calling out:

- **SHA tagging over `latest`.** `hollowy4t/action:e9fcb04` tells you exactly which commit is running. `latest` tells you nothing and cannot be rolled back to reliably.
- **Credentials come from repository secrets** (`DOCKER_USERNAME`, `DOCKER_PASSWORD`), never hardcoded.
- **Scoped permissions.** `contents: read` follows least privilege rather than leaving the default write token in place.

---

## Screenshots

Save each file in [`./screenshots/`](./screenshots) using the exact filename so it embeds automatically.

### Pipelines passing
- **`01-ci-terraform-passing.png`** - Task 1 CI run, all Terraform steps green
- **`02-cd-docker-push-passing.png`** - Task 2 CD run, build and push green
- **`03-dockerhub-image.png`** - the pushed image visible in DockerHub with its SHA tag

![CI passing](./screenshots/01-ci-terraform-passing.png)
![CD passing](./screenshots/02-cd-docker-push-passing.png)
![DockerHub image](./screenshots/03-dockerhub-image.png)

### Failures I debugged (evidence for the issues below)
- **`04-issue-invalid-tag.png`** - `invalid tag "/Action:e9fcb04"`
- **`05-issue-module-not-found.png`** - `ModuleNotFoundError: No module named 'matrix'`
- **`06-issue-unittest-requirement.png`** - `No matching distribution found for unittest`
- **`07-issue-python-version.png`** - matrix resolving `3.10` as `3.1`

![Invalid tag](./screenshots/04-issue-invalid-tag.png)
![Module not found](./screenshots/05-issue-module-not-found.png)
![unittest requirement](./screenshots/06-issue-unittest-requirement.png)
![Python version](./screenshots/07-issue-python-version.png)

---

## Issues I hit and how I solved them

### 1. `invalid tag "/Action:e9fcb04": invalid reference format`

**What happened:** the Docker build failed instantly. The tag came out as `/Action:e9fcb04` - with a leading slash and no namespace.

**Cause:** two things at once.
- The registry/username variable resolved to an **empty string**, so `"${registery}/Action:${sha}"` collapsed to `/Action:...`. A leading slash is not a valid image reference.
- Docker repository names **must be lowercase**. `Action` with a capital A is invalid regardless of the namespace.

**Fix:** set the namespace explicitly and lowercase the repository name:
```bash
registery="hollowy4t"
primary_tag="${registery}/action:${sha}"
```

**Lesson:** an unset shell variable in a workflow does not error, it silently becomes empty and corrupts whatever string it is interpolated into. The error surfaces far from the actual mistake.

### 2. `ModuleNotFoundError: No module named 'matrix'`

**What happened:** the test job failed importing the code under test. `matrix/test_hello.py` does `from matrix.hello import hello`.

**Cause:** the workflow ran `cd matrix` and then invoked the tests. That made `matrix/` the working directory, so the **repository root was not on `sys.path`** and the `matrix` package could not be resolved. There was also no `__init__.py`, so the directory was not importable as a package.

**Fix:** added [`matrix/__init__.py`](../matrix/__init__.py) to make it a real package, and ran discovery **from the repository root** instead of `cd`-ing into the folder:
```bash
python -m unittest discover -s matrix -t .
```
`-s` says where to find tests, `-t` sets the top-level directory that goes on `sys.path`.

**Lesson:** `cd` in a CI step changes import resolution. For Python, run from the repo root and point the runner at the subdirectory.

### 3. `Could not find a version that satisfies the requirement unittest`

**What happened:** `pip install -r requirements.txt` failed with `No matching distribution found for unittest`.

**Cause:** `unittest` was listed in `requirements.txt`. It is part of the **Python standard library**, not a PyPI package, so there is nothing to install and pip fails hard.

**Fix:** removed it from the requirements file. Nothing needs installing to use `unittest`.

**Lesson:** requirements files are for third-party dependencies only. If it ships with Python, listing it breaks the install step.

### 4. Python matrix resolved `3.10` as `3.1`

**What happened:** the build matrix requested Python `3.10` but the runner set up `3.1`, which does not exist as a modern runtime.

**Cause:** YAML parses an unquoted `3.10` as the **number** 3.10, and a trailing zero on a float is meaningless, so it becomes `3.1`.

**Fix:** quote the versions so YAML treats them as strings:
```yaml
python-version: [ "3.9", "3.10", "3.11" ]
```

**Lesson:** classic YAML type coercion. Always quote version numbers, and be careful with unquoted `yes`/`no`/`on`/`off` for the same reason.

---

## What I learnt

- **A pipeline is a feedback loop, not a script.** Every failure above was fixed in minutes once the log was read properly. The skill is reading the error, not memorising YAML.
- **Fail fast, fail loud.** `terraform fmt -check` with `continue-on-error: false` means formatting drift blocks the merge instead of rotting in the codebase.
- **Tag images with the commit SHA.** Immutable, traceable tags make rollback a real option. `latest` is a moving target that cannot be reasoned about.
- **Secrets belong in the secret store.** DockerHub credentials come from repository secrets and are masked in logs, which is why the run log shows `registry="***"`.
- **Least privilege applies to CI too.** The default `GITHUB_TOKEN` is broader than most jobs need, so `permissions` is set explicitly.
- **Empty variables are more dangerous than missing ones.** An unset variable produced a malformed image tag rather than a clear error.

## Improvements I would make next

These are known gaps rather than finished work:

1. **Wire the unit tests back into CI.** `matrix/` contains tests and a package, but no workflow currently runs them - the CI workflow was replaced by the Terraform job. This is the biggest gap against the brief.
2. **Trigger on pull requests.** Both workflows only run on `push`. Adding `pull_request` means checks run *before* code reaches `main`, which is the point of CI.
3. **Pin newer action versions.** `actions/checkout@v2` and `hashicorp/setup-terraform@v1` are outdated; `@v4` / `@v3` are current.
4. **Use `docker/build-push-action` with `docker/metadata-action`** for layer caching and automatic tagging, instead of raw `docker build`/`docker push`.
5. **Fix the `registery` typo** and promote it to a workflow-level `env` value.
6. **Harden the Dockerfile** - uncomment the dependency install, pin a newer base image than `python:3.8-slim`, and run as a non-root user.
