# Docker Build and Push Action

Builds a Docker image using docker/build-push-action with metadata and optional registry push.

## Features
- Supports multi-stage targets, platforms, and build arguments
- Automatic registry prefix handling when registry is provided
- Optional push to registry or local load for testing
- Automatic OCI labels for image metadata (version, title, description, source, revision)
- GitHub Actions cache support for faster builds
- Produces both the canonical tag and the resulting digest

## Inputs

| Name                | Description                                                                 | Required | Default   |
|---------------------|-----------------------------------------------------------------------------|----------|-----------|
| `image`             | Docker image name (e.g., 'ghcr.io/org/app' or 'org/app')                   | Yes      | -         |
| `registry`          | Docker registry URL (e.g., 'ghcr.io'). If provided and image lacks registry prefix, it will be added automatically. | No       | -         |
| `target`            | Optional multi-stage build target name                                      | No       | -         |
| `context`           | Build context directory                                                     | No       | `.`       |
| `platform`          | Target platforms for build (e.g., 'linux/amd64,linux/arm64')               | No       | -         |
| `options`           | Additional docker build flags appended verbatim                             | No       | -         |
| `build-args-file`   | Relative path (from context) to a KEY=VALUE file consumed as --build-arg entries | No       | -         |
| `push`              | Push built image to registry                                                | No       | `false`   |
| `image-version`     | Image version for org.opencontainers.image.version label                    | No       | -         |
| `image-title`       | Human-readable title for org.opencontainers.image.title label              | No       | -         |
| `image-description` | Human-readable description for org.opencontainers.image.description label  | No       | -         |

## Outputs

| Name           | Description                                                |
|----------------|------------------------------------------------------------|
| `digest`       | Docker image digest produced by the build                  |
| `image`        | Built docker image reference                               |
| `has-artifact` | Whether artifact is available (false when pushing)         |

## Usage

### Basic Build (No Push)

```yaml
- name: Build Docker image
  uses: draftm0de/github.workflows/.github/actions/docker-build-push@main
  with:
    image: myapp
    registry: ghcr.io
    context: .
    push: false
```

This builds `ghcr.io/myapp:<git-sha>` locally without pushing.

### Build and Push to GHCR

```yaml
permissions:
  contents: read
  packages: write

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Login to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push Docker image
        uses: draftm0de/github.workflows/.github/actions/docker-build-push@main
        with:
          image: myorg/myapp:v1.2.3
          registry: ghcr.io
          context: .
          push: true
          image-version: '1.2.3'
          image-title: 'My Application'
          image-description: 'A sample application'
```

### Multi-Platform Build

```yaml
- name: Build multi-platform image
  uses: draftm0de/github.workflows/.github/actions/docker-build-push@main
  with:
    image: myapp:latest
    registry: ghcr.io
    platform: linux/amd64,linux/arm64
    push: true
```

### With Build Args File

Create a build args file (e.g., `.build.args`):
```
NODE_VERSION=20
APP_ENV=production
```

Use in workflow:
```yaml
- name: Build with build args
  uses: draftm0de/github.workflows/.github/actions/docker-build-push@main
  with:
    image: myapp
    registry: ghcr.io
    build-args-file: .build.args
    push: false
```

### Multi-Stage Build

```yaml
- name: Build specific stage
  uses: draftm0de/github.workflows/.github/actions/docker-build-push@main
  with:
    image: myapp
    registry: ghcr.io
    target: production
    push: true
```

## How It Works

1. **Prepare Configuration**:
   - Parses image name and extracts tag (defaults to short git SHA if no tag provided)
   - If `registry` is provided and image name doesn't include registry prefix, adds it automatically
   - Validates build-args-file exists if provided

2. **Parse Build Args**:
   - Reads KEY=VALUE pairs from build-args-file
   - Converts to docker build-args format

3. **Set up Docker Buildx**:
   - Configures BuildKit for advanced features

4. **Generate Metadata**:
   - Creates OCI-compliant labels and tags using docker/metadata-action

5. **Build (and optionally Push)**:
   - Builds image with all specified options
   - If `push: true`, pushes to registry
   - If `push: false`, loads image locally for testing
   - Uses GitHub Actions cache for layer caching

## Registry Prefix Handling

The action automatically handles registry prefixes:

| Input Image      | Registry    | Result                    |
|------------------|-------------|---------------------------|
| `myapp`          | `ghcr.io`   | `ghcr.io/myapp:<tag>`     |
| `org/myapp`      | `ghcr.io`   | `ghcr.io/org/myapp:<tag>` |
| `ghcr.io/org/myapp` | `ghcr.io` | `ghcr.io/org/myapp:<tag>` |
| `myapp:v1.2.3`   | `ghcr.io`   | `ghcr.io/myapp:v1.2.3`    |
| `myapp`          | (empty)     | `myapp:<git-sha>`         |

## OCI Labels

The action automatically adds the following [OCI Image Spec](https://github.com/opencontainers/image-spec/blob/main/annotations.md) labels:

**Always added:**
- `org.opencontainers.image.created` - Build timestamp
- `org.opencontainers.image.revision` - Git commit SHA
- `org.opencontainers.image.source` - Repository URL

**Optional (when inputs provided):**
- `org.opencontainers.image.version` - Version string
- `org.opencontainers.image.title` - Human-readable title
- `org.opencontainers.image.description` - Description

## Registry Notes

**GitHub Container Registry (ghcr.io):**
- Requires `packages: write` permission
- Login before calling action with push: true
- Use `GITHUB_TOKEN` as password
- Images appear at `https://github.com/<user_or_org>?tab=packages`

**Docker Hub:**
- Leave `registry` input blank or omit it
- Login with Docker Hub credentials before calling action
- Use Docker Hub access token as password

## Example Output

Build summary shows:
```
### docker-build-push
- context: .
- image: ghcr.io/myorg/myapp:v1.2.3
- digest: sha256:abc123...
- platform: linux/amd64,linux/arm64
- pushed: true
```

## Notes

- When `push: false`, image is loaded locally and available for subsequent steps
- When `push: true`, image is pushed to registry and not loaded locally
- Build args file must contain KEY=VALUE pairs (one per line, comments with # supported)
- Platform builds require BuildKit (automatically enabled)
- GitHub Actions cache improves build performance across runs
- See [DEPLOYMENT.md](DEPLOYMENT.md) for implementation details
