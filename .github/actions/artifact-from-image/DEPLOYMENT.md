# Artifact From Image - Deployment Guide

Technical reference for rebuilding the artifact-from-image action.

## Overview

Saves a Docker image to a `.tar` file and uploads it as a GitHub Actions artifact for cross-job usage.

## Inputs & Outputs

**Inputs:**
- `image`: Docker image reference (format: `[registry/]name[:tag]`, tag defaults to `latest`)

**Outputs:**
- `artifact`: Artifact reference in `name/path` format (e.g., `myrepo-myimage-1-0/image.tar`)

## Implementation Steps

### 1. Parse Image Reference

Extract image name and tag from the input:

```bash
image="${{ inputs.image }}"
if [[ "$image" == *":"* ]]; then
  image_name="${image%%:*}"
  image_tag="${image#*:}"
else
  image_name="$image"
  image_tag="latest"
fi
```

**Examples:**
- `myrepo/myimage:1.0` → name: `myrepo/myimage`, tag: `1.0`
- `myrepo/myimage` → name: `myrepo/myimage`, tag: `latest`
- `ghcr.io/org/app:v2.3.4` → name: `ghcr.io/org/app`, tag: `v2.3.4`

### 2. Generate Artifact Name

Create a deterministic artifact name by replacing special characters:

```bash
artifact_name="$(echo "$image_name" | tr '/.' '-')-$image_tag"
artifact_path="image.tar"
```

**Character replacements:**
- `/` → `-` (directory separators)
- `.` → `-` (dots in registry names)

**Examples:**
- `myrepo/myimage:1.0` → `myrepo-myimage-1-0/image.tar`
- `ghcr.io/org/app:v2.3.4` → `ghcr-io-org-app-v2-3-4/image.tar`

### 3. Save Docker Image

Use `docker save` to export the image to a `.tar` file:

```bash
docker save -o "$artifact_path" "$image"
```

**Note:** The image must exist locally before saving. If not found, `docker save` will fail with exit code 1.

### 4. Upload Artifact

Use `actions/upload-artifact@v4` to upload the `.tar` file:

```yaml
- uses: actions/upload-artifact@v4
  with:
    name: ${{ artifact_name }}
    path: ${{ artifact_path }}
    if-no-files-found: error
```

**Important:** Set `if-no-files-found: error` to fail if the tar file wasn't created.

### 5. Write Outputs

Output the artifact reference in `name/path` format:

```bash
artifact="${artifact_name}/${artifact_path}"
echo "artifact=$artifact" >> "$GITHUB_OUTPUT"
```

### 6. Write Summary

Add a summary to `$GITHUB_STEP_SUMMARY`:

```bash
{
  echo "### artifact-from-image"
  echo "- Image: \`$image\`"
  echo "- Artifact: \`$artifact\`"
} >> "$GITHUB_STEP_SUMMARY"
```

## Output Format

The `artifact` output follows the pattern: `{sanitized-image-name}-{tag}/image.tar`

This format allows easy consumption by `artifact-to-image`:

```yaml
# Job 1: Save
- id: upload
  uses: .../artifact-from-image
  with:
    image: myrepo/myimage:1.0
# Output: myrepo-myimage-1-0/image.tar

# Job 2: Load
- uses: .../artifact-to-image
  with:
    artifact: ${{ needs.job1.outputs.artifact }}
```

## Key Behaviors

**Image Tag Handling:**
- If no tag provided → defaults to `:latest`
- Tag is included in artifact name for uniqueness

**Artifact Naming:**
- Deterministic (same image → same artifact name)
- URL-safe (no `/` or `.` characters)
- Includes tag to avoid collisions

**Error Handling:**
- Fails if image doesn't exist locally
- Fails if `docker save` fails
- Fails if upload fails

## Error Conditions

- Image not found locally → Exit 1 (docker save fails)
- Insufficient disk space → Exit 1 (docker save fails)
- Upload fails → Exit 1 (actions/upload-artifact fails)

## Dependencies

- Docker CLI
- `actions/upload-artifact@v4`
- Bash with parameter expansion

## Example Flows

**Scenario: Save built image**
```
Input: image=myapp:build
→ Parse: name=myapp, tag=build
→ Generate artifact name: myapp-build
→ Save: docker save -o image.tar myapp:build
→ Upload: actions/upload-artifact (name=myapp-build, path=image.tar)
→ Output: artifact=myapp-build/image.tar
```

**Scenario: Save registry image**
```
Input: image=ghcr.io/org/app:v1.2.3
→ Parse: name=ghcr.io/org/app, tag=v1.2.3
→ Generate artifact name: ghcr-io-org-app-v1-2-3
→ Save: docker save -o image.tar ghcr.io/org/app:v1.2.3
→ Upload: actions/upload-artifact (name=ghcr-io-org-app-v1-2-3, path=image.tar)
→ Output: artifact=ghcr-io-org-app-v1-2-3/image.tar
```

**Scenario: Save untagged image**
```
Input: image=myapp
→ Parse: name=myapp, tag=latest
→ Generate artifact name: myapp-latest
→ Save: docker save -o image.tar myapp:latest
→ Upload: actions/upload-artifact (name=myapp-latest, path=image.tar)
→ Output: artifact=myapp-latest/image.tar
```
