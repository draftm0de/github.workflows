# Docker Push - Deployment Guide

Technical reference for rebuilding the docker-push action.

## Overview

Pushes a Docker image to a registry with multiple tags. Handles registry login, tag creation, and push operations for all tags in a single action invocation.

## Inputs & Outputs

**Inputs:**
- `image`: Local Docker image name to push (e.g., `myuser/myimage` or `myimage`)
- `tags`: Space-separated list of tags (e.g., `v1.2.12 v1.2 latest`)
- `registry`: Registry hostname (optional, e.g., `ghcr.io`)
- `username`: Registry username (optional, inferred from image name if not provided)
- `password`: Registry password or token (required)

**Outputs:**
- `tags-pushed`: Space-separated list of fully-qualified tags that were pushed

## Implementation Steps

### 1. Validate Inputs

Check that required inputs are provided and image exists locally:

**Validation checks:**
- `image` is not empty
- `tags` is not empty
- `password` is not empty
- Image exists locally using `docker inspect --type=image`

**Username inference and image name processing:**
- If `username` is empty:
  - Extract username from image name using regex: `^([^/]+)/(.+)$`
  - Example: `myuser/myimage` → username: `myuser`, image: `myimage`
  - Strip username from image name for later use
  - Log notice: "extracted username from image name: {username}"
  - If extraction fails → Error "Cannot infer username from image name" and exit
- If `username` is provided:
  - Use provided value
  - Log notice: "Using provided username: {username}"
  - Image name remains unchanged

**Tag stripping:**
- Check if `IMAGE` contains `:` character
- If yes: Strip everything after `:` (e.g., `myimage:abc1234` → `myimage`)
- Log notice: "Stripped tag from image name: {image}"
- This ensures clean tag construction regardless of docker-build's default SHA tag

Exit with error if any validation fails.

Log notices for validated image and tags.

Output `username` and `image` (stripped of username/tag) to `$GITHUB_OUTPUT`.

### 2. Login to Registry

Use `docker/login-action@v3` with:
- `registry`: From input (optional)
- `username`: From validation step output
- `password`: From input

Action handles authentication and token storage.

### 3. Tag and Push Images

Iterate over space-separated tags and push each one.

**Variables:**
- `SOURCE_IMAGE`: Original image input (e.g., `myuser/myimage:abc1234`)
- `IMAGE`: Processed image name from validation step (e.g., `myimage` - username and tag stripped)
- `USERNAME`: Username from validation step
- `REGISTRY`: Registry input

**For each tag:**
1. Build fully-qualified tag:
   - If `registry` provided: `<registry>/<username>/<image>:<tag>`
   - If `registry` empty: `<username>/<image>:<tag>` (Docker Hub)
2. Tag local image: `docker tag <SOURCE_IMAGE> <full-tag>`
3. Push to registry: `docker push <full-tag>`
4. Append to pushed tags list

**Example iteration:**
```
Input image: "myuser/myapp:abc1234" (from docker-build with SHA tag)
Extracted username: "myuser"
Stripped to: "myapp:abc1234"
Tag stripped to: "myapp"
Input tags: "v1.2.12 v1.2 latest"
Registry: "ghcr.io"

Iteration 1:
  Tag: "v1.2.12"
  Full tag: "ghcr.io/myuser/myapp:v1.2.12"
  Command: docker tag myuser/myapp:abc1234 ghcr.io/myuser/myapp:v1.2.12
  Command: docker push ghcr.io/myuser/myapp:v1.2.12

Iteration 2:
  Tag: "v1.2"
  Full tag: "ghcr.io/myuser/myapp:v1.2"
  Command: docker tag myuser/myapp:abc1234 ghcr.io/myuser/myapp:v1.2
  Command: docker push ghcr.io/myuser/myapp:v1.2

Iteration 3:
  Tag: "latest"
  Full tag: "ghcr.io/myuser/myapp:latest"
  Command: docker tag myuser/myapp:abc1234 ghcr.io/myuser/myapp:latest
  Command: docker push ghcr.io/myuser/myapp:latest
```

### 4. Write Outputs

Output pushed tags list to `$GITHUB_OUTPUT`:
- Format: `tags_pushed=<tag1> <tag2> <tag3>` (space-separated)

Write summary to `$GITHUB_STEP_SUMMARY`:
- Source image (original input)
- Registry (or "Docker Hub" if empty)
- List of all pushed tags (bulleted)

## Key Behaviors

**Multiple Tags:**
- Pushes all tags in single action invocation
- Tags are space-separated
- Each tag is pushed independently

**Registry Handling:**
- Registry prefix is optional
- When provided, builds: `<registry>/<username>/<image>:<tag>`
- When empty, builds: `<username>/<image>:<tag>` (Docker Hub)

**Username Inference:**
- Extracts from image name if not provided
- Works for format: `username/image`
- When extracted, image name is stripped to just the image part
- Fails for bare names: `image` (no slash, needs explicit username)

**Image Name Processing:**
- Original image input is preserved as SOURCE_IMAGE for docker tag command
- If username extracted from image name, the image variable is stripped
- Stripped image is used to build final tag paths

**Error Handling:**
- Validation errors stop execution before push
- Any failed push stops entire action
- Uses `set -euo pipefail` for strict error handling

## Example Outputs

| Input Image | Username | Tags | Registry | Output Tags | Notes |
|-------------|----------|------|----------|-------------|-------|
| `myuser/myapp` | (extracted) | `v1.2.12` | `ghcr.io` | `ghcr.io/myuser/myapp:v1.2.12` | Username extracted from image |
| `myuser/myapp` | (extracted) | `v1.2.12 latest` | `ghcr.io` | `ghcr.io/myuser/myapp:v1.2.12 ghcr.io/myuser/myapp:latest` | Multiple tags |
| `myapp` | `myuser` | `v1.2.12` | `` | `myuser/myapp:v1.2.12` | Docker Hub, explicit username |
| `myuser/myapp` | (extracted) | `v1` | `` | `myuser/myapp:v1` | Docker Hub, extracted username |

## Dependencies

- Docker CLI for tagging and pushing
- `docker/login-action@v3` for registry authentication
- Bash with regex support
- Image must exist locally before action runs

## Integration Example

```yaml
# Step 1: Build version
- uses: tag-builder
  outputs: next-version-short, is-latest-version

# Step 2: Build Docker tags
- uses: docker-tag-builder
  inputs: version, is-latest-version, tag-levels
  outputs: docker-tags

# Step 3: Build Docker image
- uses: docker-build
  outputs: image

# Step 4: Push with all tags
- uses: docker-push
  inputs: image, tags (from docker-tag-builder), registry, password
```

## Error Conditions

- Missing required input (`image` or `tags`) → Exit 1
- Image not found locally → Exit 1
- Cannot infer username and none provided → Exit 1
- Docker tag command fails → Exit 1
- Docker push command fails → Exit 1
- Registry login fails → Exit 1 (from docker/login-action)

## Security Notes

- Never log or expose `password` input
- Use secrets for password parameter
- For GHCR, use `GITHUB_TOKEN` with `packages: write` permission
- For Docker Hub, use access tokens instead of account passwords
