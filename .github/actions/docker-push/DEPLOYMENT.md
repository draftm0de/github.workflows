# Docker Push - Deployment Guide

Technical reference for rebuilding the docker-push action.

## Overview

Pushes a Docker image to a registry with multiple tags. Handles registry login, tag creation, and push operations for all tags in a single action invocation.

## Inputs & Outputs

**Inputs:**
- `image`: Local Docker image name to push
- `tags`: Comma-separated list of tags (e.g., `v1.2.12,v1.2,latest`)
- `registry`: Registry hostname (optional, e.g., `ghcr.io`)
- `username`: Registry username (optional, inferred if not provided)
- `password`: Registry password or token

**Outputs:**
- `tags-pushed`: Comma-separated list of fully-qualified tags that were pushed

## Implementation Steps

### 1. Validate Inputs

Check that required inputs are provided and image exists locally:

**Validation checks:**
- `image` is not empty
- `tags` is not empty
- Image exists locally using `docker inspect --type=image`

Exit with error if any validation fails.

Log notices for validated image and tags.

### 2. Determine Registry Username

Extract or use provided username for registry login.

**If `username` is provided:**
- Use provided value
- Log notice with provided username

**If `username` is empty:**
- Parse first tag from comma-separated list
- Extract username using regex: `^([^/]+)/.*$`
- Example: `ghcr.io/myorg/app:v1.2.12` → username: `myorg`
- If extraction fails → Error and exit
- Log notice with inferred username

**Pattern:** `^([^/]+)/.*$`
- Matches: `ghcr.io/myorg/app` → `ghcr.io`
- Matches: `myorg/app` → `myorg`
- No match: `app` (no slash, needs explicit username)

### 3. Login to Registry

Use `docker/login-action@v3` with:
- `registry`: From input (optional)
- `username`: From previous step
- `password`: From input

Action handles authentication and token storage.

### 4. Tag and Push Images

Iterate over comma-separated tags and push each one.

**For each tag:**
1. Trim whitespace from tag
2. Build fully-qualified tag:
   - If `registry` provided: `<registry>/<tag>`
   - If `registry` empty: `<tag>` (Docker Hub)
3. Tag local image: `docker tag <image> <full-tag>`
4. Push to registry: `docker push <full-tag>`
5. Append to pushed tags list

**Example iteration:**
```
Input tags: "v1.2.12,v1.2,latest"
Registry: "ghcr.io"
Image: "myapp:build"

Iteration 1:
  Tag: "v1.2.12"
  Full tag: "ghcr.io/v1.2.12"
  Command: docker tag myapp:build ghcr.io/v1.2.12
  Command: docker push ghcr.io/v1.2.12

Iteration 2:
  Tag: "v1.2"
  Full tag: "ghcr.io/v1.2"
  Command: docker tag myapp:build ghcr.io/v1.2
  Command: docker push ghcr.io/v1.2

Iteration 3:
  Tag: "latest"
  Full tag: "ghcr.io/latest"
  Command: docker tag myapp:build ghcr.io/latest
  Command: docker push ghcr.io/latest
```

### 5. Write Outputs

Output pushed tags list to `$GITHUB_OUTPUT`:
- Format: `tags_pushed=<tag1>,<tag2>,<tag3>`

Write summary to `$GITHUB_STEP_SUMMARY`:
- Source image
- Registry (or "Docker Hub" if empty)
- List of all pushed tags (bulleted)

## Key Behaviors

**Multiple Tags:**
- Pushes all tags in single action invocation
- Tags are comma-separated with optional whitespace
- Each tag is pushed independently

**Registry Handling:**
- Registry prefix is optional
- When provided, prepended to each tag
- When empty, assumes Docker Hub

**Username Inference:**
- Extracts from first tag if not provided
- Works for registry-prefixed tags: `registry.io/user/app`
- Works for Docker Hub format: `user/app`
- Fails for bare names: `app` (no slash)

**Error Handling:**
- Validation errors stop execution before push
- Any failed push stops entire action
- Uses `set -euo pipefail` for strict error handling

## Example Outputs

| Input | Output Tags | Notes |
|-------|-------------|-------|
| tags=`v1.2.12`, registry=`ghcr.io` | `ghcr.io/v1.2.12` | Single tag with registry |
| tags=`v1.2.12,latest`, registry=`ghcr.io` | `ghcr.io/v1.2.12,ghcr.io/latest` | Multiple tags with registry |
| tags=`myorg/app:v1.2.12`, registry=`` | `myorg/app:v1.2.12` | Docker Hub (no registry) |
| tags=`myorg/app:v1,myorg/app:latest` | `myorg/app:v1,myorg/app:latest` | Docker Hub with multiple tags |

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
