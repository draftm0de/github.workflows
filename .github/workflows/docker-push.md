# Docker Push Image Workflow

Reusable workflow that validates inputs, optionally loads a saved artifact, and then delegates to the `docker-push` composite action to authenticate and push to a registry.

## Features

- Pushes Docker images directly or loads them from artifacts.
- Optionally tags images with a new target name before pushing.
- Uses a secure Docker token for authentication.

## Inputs

| Name | Description | Required | Default |
| --- | --- | --- | --- |
| `image` | Local Docker image to push. | No | — |
| `artifact` | Artifact handle created by the build workflow/action. | No | — |
| `target` | Optional retag applied before pushing. | No | — |
| `docker-username` | Explicit registry username. | No | — |
| `docker-registry` | Registry hostname (`ghcr.io`, etc.). | No | — |

## Secrets

| Name | Description | Required |
| --- | --- | --- |
| `DOCKER_TOKEN` | Registry password/token forwarded to `docker/login-action`. | Yes |

## How It Works

1. Workflow validates that either `image` or `artifact` was supplied.
2. Composite action loads the artifact when necessary, logs in, applies optional tags, and pushes the image.

## Usage Example

```yaml
name: Push Docker image

on:
  workflow_dispatch:

jobs:
  push:
    uses: draftm0de/github.workflows/.github/workflows/docker-push.yml@main
    secrets:
      DOCKER_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    with:
      artifact: ${{ needs.build.outputs.artifact }}
      target: ghcr.io/draftm0de/app:${{ github.ref_name }}
      docker-registry: ghcr.io
```

When targeting GHCR remember to grant `packages: write` permissions, as outlined in the [`docker-push` action docs](../actions/docker-push/README.md).
