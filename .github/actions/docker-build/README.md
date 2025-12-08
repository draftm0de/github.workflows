# Build Docker Image Action

Composite action that builds a Docker image, optionally via Buildx for reproducible layers, and uploads the result as a workflow artifact for downstream jobs.

## Features
- Supports multi-stage targets, additional CLI flags, and build-arg files.
- Optional reproducible builds (`reproducible: 'true'`) leverage `docker/setup-buildx-action` with deterministic timestamps.
- Persists the built image with [`artifact-from-image`](../artifact-from-image) so publish-only jobs can load it later.

## Inputs
| Name | Description | Required | Default |
| --- | --- | --- | --- |
| `image` | Full image reference (falls back to `:latest` when no tag provided). | Yes | — |
| `target` | Multi-stage `--target` name. | No | — |
| `context` | Build context directory. | No | `.` |
| `options` | Extra CLI flags appended verbatim (for example `--no-cache`). | No | — |
| `build-args-file` | File relative to `context` containing `KEY=VALUE` pairs that become `--build-arg` entries. | No | — |
| `reproducible` | Set to `'true'` to enable Buildx reproducible builds. | No | `'false'` |

## Outputs
| Name | Description |
| --- | --- |
| `digest` | Docker image digest produced by the build. |
| `artifact` | Artifact handle (`name/path`) suitable for `artifact-to-image`. |

## Usage
```yaml
jobs:
  build-docker-image:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build image
        uses: draftm0de/github.workflows/.github/actions/docker-build@main
        with:
          image: ghcr.io/draftm0de/app:${{ github.sha }}
          context: ./apps/web
          build-args-file: build.args
          options: --no-cache
          reproducible: 'true'
```

Use the emitted `artifact` output with [`artifact-to-image`](../artifact-to-image) to reload the image during later jobs (for example pushing inside a release workflow).
