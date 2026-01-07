# Docker Build and Push - Deployment Guide

Technical reference for rebuilding the docker-build-push action.

## Overview

Builds a Docker image using docker/build-push-action with automatic registry prefix handling, OCI metadata labels, and optional push to registry. Combines build and push operations in a single action with intelligent defaults.

## Inputs & Outputs

**Inputs:**
- `image`: Docker image name (e.g., `ghcr.io/org/app` or `org/app`)
- `registry`: Registry hostname (optional, e.g., `ghcr.io`)
- `target`: Multi-stage build target name (optional)
- `context`: Build context directory (default: `.`)
- `platform`: Target platforms for build (optional, e.g., `linux/amd64,linux/arm64`)
- `options`: Additional docker build flags (optional)
- `build-args-file`: Relative path to KEY=VALUE file for build args (optional)
- `push`: Push built image to registry (default: `false`)
- `image-version`: Image version for OCI label (optional)
- `image-title`: Human-readable title for OCI label (optional)
- `image-description`: Description for OCI label (optional)

**Outputs:**
- `digest`: Docker image digest produced by the build
- `image`: Built docker image reference (with registry prefix if applicable)
- `has-artifact`: Whether artifact is available (`false` when `push: true`, `true` otherwise)

## Implementation Steps

### 1. Prepare Build Configuration

Parse image name, handle registry prefix, and validate inputs.

**Image name parsing:**
- Check if `image` contains `:` character
- If yes: Split into `image_name` (before `:`) and `image_tag` (after `:`)
- If no: Use entire input as `image_name`, set `image_tag` to short git SHA (first 7 chars of `github.sha`)

**Registry prefix handling:**
- Get `registry` input value
- If `registry` is not empty:
  - Check if `image_name` already starts with registry prefix
  - If NOT already prefixed: Prepend `registry/` to `image_name`
  - Logic: Check if image lacks `/` or `.` characters (simple image name), or doesn't start with registry value
  - Example: `myapp` + registry `ghcr.io` → `ghcr.io/myapp`
  - Example: `ghcr.io/myapp` + registry `ghcr.io` → `ghcr.io/myapp` (no double prefix)

**Build image reference:**
- Combine: `image_ref = image_name:image_tag`
- Output to `$GITHUB_OUTPUT`:
  - `image-ref`: Full image reference
  - `image-name`: Parsed image name (with registry prefix if added)
  - `image-tag`: Tag value

**Artifact availability:**
- Set `has-artifact` to `true` if `push != 'true'`, otherwise `false`
- When not pushing, image is loaded locally and available as artifact
- Output to `$GITHUB_OUTPUT`

**Build args file validation:**
- If `build-args-file` input provided:
  - Construct path: `context/build-args-file`
  - Check file exists with `[ -f "$build_args_file" ]`
  - If exists: Output path to `$GITHUB_OUTPUT` as `build-args-file`
  - If not exists: Error "Build-args file not found: {path}" and exit 1

### 2. Parse Build Args from File

Read KEY=VALUE pairs from build-args-file if provided.

**Conditional execution:**
- Only run if `steps.prepare.outputs.build-args-file` is not empty

**Processing:**
1. Read file line by line using `IFS='=' read -r key value`
2. Skip empty lines and comments (lines starting with `#`)
3. For each valid line: Append `${key}=${value}\n` to `build_args` variable
4. Output multi-line string to `$GITHUB_OUTPUT` using heredoc format:
   ```bash
   {
     echo 'build-args<<EOF'
     echo "$build_args"
     echo 'EOF'
   } >> "$GITHUB_OUTPUT"
   ```

**Example:**
```
Input file (.build.args):
NODE_VERSION=20
APP_ENV=production
# This is a comment

Output (build-args):
NODE_VERSION=20
APP_ENV=production
```

### 3. Set up Docker Buildx

Use `docker/setup-buildx-action@v3` to enable BuildKit features.

**Benefits:**
- Multi-platform builds
- Advanced caching
- Build secrets support
- Reproducible builds

No configuration needed - action uses defaults.

### 4. Generate Docker Metadata

Use `docker/metadata-action@v5` to create OCI-compliant labels and tags.

**Configuration:**
- `images`: Use `image-name` from prepare step (without tag)
- `tags`: Single raw tag using `image-tag` from prepare step
  ```yaml
  type=raw,value=${{ steps.prepare.outputs.image-tag }}
  ```

**Labels (always added):**
- `org.opencontainers.image.created`: Use `github.event.head_commit.timestamp` or `github.event.repository.updated_at`
- `org.opencontainers.image.revision`: Use `github.sha`
- `org.opencontainers.image.source`: Construct from `github.server_url/github.repository`

**Labels (conditional - only if input provided and not empty):**
- `org.opencontainers.image.version`: From `image-version` input
- `org.opencontainers.image.title`: From `image-title` input
- `org.opencontainers.image.description`: From `image-description` input

**Label format:**
```yaml
${{ inputs.image-version != '' && format('org.opencontainers.image.version={0}', inputs.image-version) || '' }}
```

**Outputs:**
- `tags`: Formatted tags for docker/build-push-action
- `labels`: Formatted labels for docker/build-push-action

### 5. Build and Push Docker Image

Use `docker/build-push-action@v5` to build (and optionally push) the image.

**Configuration:**
- `context`: From input (default: `.`)
- `target`: From input (multi-stage target name)
- `platforms`: From `platform` input (comma-separated list)
- `push`: From input (boolean, default: `false`)
- `load`: Inverse of `push` - if not pushing, load locally (`${{ inputs.push != 'true' }}`)
- `tags`: From metadata step output
- `labels`: From metadata step output
- `build-args`: From build-args step output (if file was provided)
- `cache-from`: `type=gha` (GitHub Actions cache - read)
- `cache-to`: `type=gha,mode=max` (GitHub Actions cache - write with maximum layer coverage)

**Load vs Push:**
- When `push: false`: Image built and loaded into local Docker daemon (available for docker commands)
- When `push: true`: Image built and pushed to registry (NOT loaded locally)
- Cannot do both simultaneously with BuildKit

**Outputs:**
- `digest`: Image digest (SHA256 hash)

### 6. Write Summary

Write build information to `$GITHUB_STEP_SUMMARY` for workflow UI.

**Format:**
```markdown
### docker-build-push
- context: {context}
- image: {image-ref}
- digest: {digest}
- platform: {platform or 'default'}
- pushed: {push}
```

**Example output:**
```markdown
### docker-build-push
- context: .
- image: ghcr.io/myorg/myapp:v1.2.3
- digest: sha256:abc123def456...
- platform: linux/amd64,linux/arm64
- pushed: true
```

## Key Behaviors

### Registry Prefix Logic

The action intelligently adds registry prefix when needed:

**Decision flow:**
1. If `registry` input is empty → No prefix added
2. If `registry` provided:
   - Check if `image_name` already contains registry (starts with registry value)
   - If yes → No change
   - If no → Prepend `registry/` to image name

**Detection logic:**
```bash
if [ -n "$registry" ]; then
  if [[ "$image_name" != *"/"* ]] || [[ "$image_name" != *"."* ]]; then
    if [[ "$image_name" != "$registry"* ]]; then
      image_name="$registry/$image_name"
    fi
  fi
fi
```

This prevents double-prefixing while ensuring registry is added when needed.

### Tag Defaulting

If no tag provided in `image` input:
- Uses short git SHA (first 7 characters)
- Ensures every build has unique, traceable tag
- Example: `myapp` → `myapp:abc1234` where `abc1234` is short SHA

### Multi-Platform Builds

When `platform` specified with multiple architectures:
- BuildKit builds for all platforms simultaneously
- Requires Buildx (automatically set up in step 3)
- Must use `push: true` (cannot load multi-platform images locally)
- Produces multi-arch manifest
- Example: `linux/amd64,linux/arm64` creates single image tag with platform-specific layers

### Build Args Processing

Build args file format:
```
# Comments start with #
KEY1=value1
KEY2=value2

# Empty lines ignored
KEY3=value3
```

Processed into format for docker/build-push-action:
```
KEY1=value1
KEY2=value2
KEY3=value3
```

### Caching Strategy

Uses GitHub Actions cache with `mode=max`:
- Caches all layers (not just final layers)
- Significantly speeds up incremental builds
- Automatic cache management (GitHub handles expiration)
- No configuration needed

### Push vs Load

**When `push: false` (default):**
- Image built and loaded into Docker daemon
- Available for: `docker run`, `docker save`, further processing
- Output `has-artifact: true`
- Good for: Testing, artifact creation, local validation

**When `push: true`:**
- Image built and pushed to registry
- NOT available locally
- Requires prior registry login
- Output `has-artifact: false`
- Good for: Deployment, distribution, publishing

## Error Conditions

- Missing required input `image` → Exit 1 in prepare step
- Build-args-file specified but doesn't exist → Exit 1 with error message
- Docker build fails → Action fails (from build-push-action)
- Push fails (network, auth, etc.) → Action fails (from build-push-action)
- Invalid platform specification → Action fails (from build-push-action)

## Dependencies

- `docker/setup-buildx-action@v3`: Enables BuildKit features
- `docker/metadata-action@v5`: Generates OCI-compliant labels and tags
- `docker/build-push-action@v5`: Builds and optionally pushes image
- Bash with parameter expansion and conditionals
- Docker BuildKit (automatically enabled by Buildx)

## Integration Examples

### With node-js-ci Workflow

**Build step (no push):**
```yaml
- name: Build docker image
  uses: draftm0de/github.workflows/.github/actions/docker-build-push@main
  id: build
  with:
    image: ${{ inputs.docker-image-name }}
    registry: ${{ inputs.docker-registry }}
    context: ${{ inputs.docker-build-context }}
    platform: ${{ inputs.docker-build-platform }}
    push: false
    image-version: ${{ needs.auto_tagging.outputs.next-version-short }}
```

**Push step (with registry login):**
```yaml
- name: Login to Docker registry
  uses: docker/login-action@v3
  with:
    registry: ${{ inputs.docker-registry }}
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}

- name: Build and push docker image
  uses: draftm0de/github.workflows/.github/actions/docker-build-push@main
  with:
    image: ${{ needs.docker_build.outputs.image }}
    registry: ${{ inputs.docker-registry }}
    context: ${{ inputs.docker-build-context }}
    platform: ${{ inputs.docker-build-platform }}
    push: true
    image-version: ${{ needs.auto_tagging.outputs.next-version-short }}
```

### Standalone Usage

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: draftm0de/github.workflows/.github/actions/docker-build-push@main
        with:
          image: myapp:v1.0.0
          registry: ghcr.io
          push: false
```

## Security Notes

- Never log registry credentials
- Use secrets for registry passwords
- For GHCR, use `GITHUB_TOKEN` with `packages: write` permission
- Build args may contain sensitive values - avoid logging
- OCI labels are public metadata - don't include secrets

## Performance Considerations

- GitHub Actions cache reduces build time for unchanged layers
- Multi-platform builds take longer (building for multiple architectures)
- Large contexts slow down builds - use `.dockerignore`
- BuildKit parallel builds improve performance
- Cache hits can reduce build time by 50-90%
