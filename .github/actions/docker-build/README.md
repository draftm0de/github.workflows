# Build Docker Image Action

Composite action that builds a Docker image (optionally using Buildx for reproducible layers) and emits the fully-qualified tag plus digest so downstream jobs can push, save, or publish the image however they like.

## Features
- Supports multi-stage targets, additional CLI flags, and build-arg files.
- Optional reproducible builds (`reproducible: 'true'`) leverage `docker/setup-buildx-action` with deterministic timestamps.
- Produces both the canonical tag and the resulting digest so you can chain into push/publish steps or save the image with [`artifact-from-image`](../artifact-from-image).

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
| `image` | Fully-qualified tag (with automatic `:latest` default) that was passed to `docker build`. |

## Usage
```yaml
jobs:
  build-docker-image:
    runs-on: ubuntu-latest
    steps:
      - name: Build image
        uses: ./.github/actions/docker-build
        with:
          image: ghcr.io/draftm0de/app:${{ github.sha }}
          context: ./apps/web
          build-args-file: build.args
          options: --no-cache
          reproducible: 'true'
```

Follow-up jobs can reuse the `image` + `digest` outputs directly (for example to push with `docker login && docker push ${{ needs.build.outputs.image }}`), or choose to serialize the image by running [`artifact-from-image`](../artifact-from-image) with the emitted `image` tag.
