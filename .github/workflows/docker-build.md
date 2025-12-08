# Docker Build Image Workflow

Reusable workflow that wraps the `docker-build` composite action so repositories can share an identical build definition (including optional Buildx reproducibility and artifact export).

## Features

- Supports advanced Docker build configurations such as `--target` and additional build options.
- Reads build arguments from a file for dynamic and reusable builds.
- Provides reproducible builds for consistency across environments.
- Outputs the built image artifact name and digest for downstream workflows.

## Inputs

| Name | Description | Required | Default |
| --- | --- | --- | --- |
| `image` | Destination image reference (`repo/name:tag`). | Yes | — |
| `target` | Build `--target` stage name. | No | — |
| `context` | Docker build context. | No | `.` |
| `options` | Additional CLI options (for example `--no-cache`). | No | — |
| `build-args-file` | Relative path within `context` containing `KEY=VALUE` lines. | No | — |
| `reproducible` | `'true'` to enable Buildx deterministic mode. | No | `'false'` |

## Outputs

| Name | Description |
| --- | --- |
| `artifact` | Artifact reference (name/path) produced by [`artifact-from-image`](../actions/artifact-from-image). |
| `digest` | Docker image digest as reported by `docker inspect`. |

## Usage Example

```yaml
name: Build Docker image

on:
  workflow_dispatch:

jobs:
  build:
    uses: draftm0de/github.workflows/.github/workflows/docker-build.yml@main
    secrets: inherit
    with:
      image: ghcr.io/draftm0de/app:${{ github.sha }}
      context: ./docker
      build-args-file: build-args.txt
      reproducible: 'true'
```

Chain the resulting `jobs.build.outputs.artifact` into the push workflow (or call the [`docker-push`](./docker-push.md) reusable workflow) so later jobs can upload the exact same image.
