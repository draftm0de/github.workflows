# Build Docker Image Action

Composite action that builds a Docker image (optionally using Buildx for reproducible layers) and emits the fully-qualified tag plus digest so downstream jobs can push, save, or publish the image however they like.

## Features
- Supports multi-stage targets, additional CLI flags, and build-arg files
- Multi-platform builds (e.g., `linux/amd64,linux/arm64`) for cross-architecture support
- Optional reproducible builds (`reproducible: 'true'`) leverage `docker/setup-buildx-action` with deterministic timestamps
- Automatic OCI labels for image metadata (version, title, description, source, revision)
- Produces both the canonical tag and the resulting digest so you can chain into push/publish steps or save the image with [`artifact-from-image`](../artifact-from-image)

## Inputs
| Name              | Description                                                                                | Required | Default   |
|-------------------|--------------------------------------------------------------------------------------------|----------|-----------|
| `image`           | Full image reference (defaults to short git SHA when no tag provided, e.g., `:abc1234`).  | Yes      | —         |
| `target`          | Multi-stage `--target` name.                                                               | No       | —         |
| `context`         | Build context directory.                                                                   | No       | `.`       |
| `reproducible`    | Set to `'true'` to enable Buildx reproducible builds.                                      | No       | `'false'` |
| `options`         | Extra CLI flags appended verbatim (for example `--no-cache`).                              | No       | —         |
| `platform`        | Target platform(s) for build (e.g., `linux/amd64`, `linux/arm64`, `linux/amd64,linux/arm64`). | No       | —         |
| `build-args-file` | File relative to `context` containing `KEY=VALUE` pairs that become `--build-arg` entries. | No       | —         |
| `image-version`         | Image version for `org.opencontainers.image.version` label.                                | No       | —         |
| `image-title`           | Human-readable title for `org.opencontainers.image.title` label.                          | No       | —         |
| `image-description`     | Description for `org.opencontainers.image.description` label.                              | No       | —         |

## Outputs
| Name     | Description                                                                               |
|----------|-------------------------------------------------------------------------------------------|
| `digest` | Docker image digest produced by the build.                                                |
| `image`  | Fully-qualified tag (with automatic short SHA default) that was passed to `docker build`. |

## Usage
```yaml
jobs:
  build-docker-image:
    runs-on: ubuntu-latest
    steps:
      - name: Build image
        uses: draftm0de/github.workflows/.github/actions/docker-build@main
        with:
          image: ghcr.io/draftm0de/app:${{ github.sha }}
          context: ./apps/web
          build-args-file: build.args
          options: --no-cache
          reproducible: 'true'
          platform: 'linux/amd64,linux/arm64'
          image-version: '1.2.3'
          image-title: 'My Application'
          image-description: 'A sample application'
```

Follow-up jobs can reuse the `image` + `digest` outputs directly (for example to push with `docker login && docker push ${{ needs.build.outputs.image }}`), or choose to serialize the image by running [`artifact-from-image`](../artifact-from-image) with the emitted `image` tag.

## Multi-Platform Builds

To build images for multiple architectures (e.g., AMD64 and ARM64):

```yaml
- name: Build multi-platform image
  uses: draftm0de/github.workflows/.github/actions/docker-build@main
  with:
    image: ghcr.io/draftm0de/app:latest
    platform: 'linux/amd64,linux/arm64'
    reproducible: 'true'
```

**Note:** Multi-platform builds require `reproducible: 'true'` as they use Docker Buildx.

Common platform values:
- `linux/amd64` - x86-64 (standard Linux/Windows servers)
- `linux/arm64` - ARM 64-bit (Apple Silicon, AWS Graviton, Raspberry Pi 4+)
- `linux/arm/v7` - ARM 32-bit (older Raspberry Pi)
- `linux/amd64,linux/arm64` - Multi-arch (both Intel and ARM)

## OCI Labels

The action automatically adds the following [OCI Image Spec](https://github.com/opencontainers/image-spec/blob/main/annotations.md) labels to the built image:

**Always added:**
- `org.opencontainers.image.created` - Build timestamp (UTC, ISO 8601)
- `org.opencontainers.image.revision` - Git commit SHA (`github.sha`)
- `org.opencontainers.image.source` - Repository URL (`github.server_url/github.repository`)

**Optional (when inputs provided):**
- `org.opencontainers.image.version` - Version string (from `image-version` input)
- `org.opencontainers.image.title` - Human-readable title (from `image-title` input)
- `org.opencontainers.image.description` - Description (from `image-description` input)

These labels help with image inspection, auditing, and tool integration (e.g., `docker inspect`, security scanners).
