# Task 2 - Continuous Deployment (Docker build and push)

Builds the application image and publishes it to DockerHub on every push,
defined in [`.github/workflows/task-2-cd.yaml`](../../.github/workflows/task-2-cd.yaml).

## Flow

```
push -> checkout -> docker login (secrets) -> docker build -> docker push
```

The published image is `hollowy4t/action:<short-sha>`, for example
`hollowy4t/action:e9fcb04`.

## Why tag with the commit SHA

Every image maps back to the exact commit that produced it. That makes it
possible to say precisely what is running in an environment, and to roll back to
a known-good build. A floating `latest` tag cannot do either: it changes under
you, and two people pulling `latest` an hour apart can get different code.

```bash
registery="hollowy4t"
sha=$(git rev-parse --short HEAD)
primary_tag="${registery}/action:${sha}"
docker build -t $primary_tag -f Dockerfile .
docker push $primary_tag
```

## Secrets

Authentication uses `docker/login-action` with two repository secrets:

| Secret | Purpose |
|--------|---------|
| `DOCKER_USERNAME` | DockerHub account |
| `DOCKER_PASSWORD` | DockerHub access token |

They are never written into the workflow file, and GitHub masks them in run
logs - which is why the log line reads `registry="***"`.

The job also narrows its token with `permissions: contents: read`, rather than
inheriting the broader default.

## What the image contains

[`Dockerfile`](../../Dockerfile) is based on `python:3.8-slim`, copies the repo
into `/app`, and runs [`app.py`](../../app.py).

## Improvements

- Replace raw `docker build`/`docker push` with `docker/build-push-action` plus
  `docker/metadata-action` to get layer caching and automatic tag generation.
- Also publish a moving tag (`main`) alongside the immutable SHA tag.
- Move `registery` (typo intended to be `registry`) into a workflow-level `env`.
- Uncomment the dependency install in the Dockerfile, move to a newer base
  image, and run as a non-root user.
