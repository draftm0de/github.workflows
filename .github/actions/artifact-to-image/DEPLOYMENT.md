# Artifact To Image - Deployment Guide

Technical reference for rebuilding the artifact-to-image action.

## Overview

Downloads a Docker image artifact and loads it into Docker for subsequent use.

## Inputs & Outputs

**Inputs:**
- `artifact`: Artifact reference in `name/path` format (e.g., `myrepo-myimage-1-0/image.tar`)

**Outputs:**
- `image`: Docker image reference extracted from `docker load` (e.g., `myrepo/myimage:1.0`)

## Implementation Steps

### 1. Parse Artifact Reference

Split the artifact input into name and path components:

```bash
artifact="${{ inputs.artifact }}"
if [[ "$artifact" == */* ]]; then
  artifact_name="${artifact%%/*}"
  artifact_file="${artifact#*/}"
else
  echo "::error::Artifact input must be in 'name/path' form"
  exit 1
fi
```

**Format validation:**
- Input MUST contain `/` separator
- Format: `{artifact-name}/{file-path}`

**Examples:**
- `myrepo-myimage-1-0/image.tar` → name: `myrepo-myimage-1-0`, file: `image.tar`
- `ghcr-io-org-app-v2-3-4/image.tar` → name: `ghcr-io-org-app-v2-3-4`, file: `image.tar`

**Invalid examples:**
- `image.tar` → Error (no name separator)
- `myartifact` → Error (no path component)

### 2. Download Artifact

Use `actions/download-artifact@v4` to download the artifact:

```yaml
- uses: actions/download-artifact@v4
  with:
    name: ${{ artifact_name }}
    path: .artifact
```

**Important:**
- Downloads to `.artifact/` directory to avoid conflicts
- Action fails if artifact doesn't exist
- Artifact must have been uploaded in a previous job

### 3. Load Docker Image

Use `docker load` to import the image from the tar file:

```bash
file=".artifact/${{ artifact_file }}"
image=$(docker load --input "$file" | awk '/Loaded image:/ { print $3 }')
if [ -z "$image" ]; then
  echo "::error::Failed to load image name from artifact output"
  exit 1
fi
```

**Output parsing:**
- `docker load` outputs: `Loaded image: myrepo/myimage:1.0`
- `awk` extracts the third field: `myrepo/myimage:1.0`
- Validation ensures image reference was extracted

### 4. Write Outputs

Output the loaded image reference:

```bash
echo "image=$image" >> "$GITHUB_OUTPUT"
```

**Note:** This is the original image reference, not the artifact name.

### 5. Write Summary

Add a summary to `$GITHUB_STEP_SUMMARY`:

```bash
{
  echo "### artifact-to-image"
  echo "- Artifact: \`$artifact\`"
  echo "- Loaded image: \`$image\`"
} >> "$GITHUB_STEP_SUMMARY"
```

## Key Behaviors

**Artifact Location:**
- Downloads to `.artifact/` subdirectory
- Prevents conflicts with repository files
- Directory is ephemeral (job-scoped)

**Image Reference Extraction:**
- Parses `docker load` output to get image name
- Preserves original image name and tag
- Fails if parsing fails

**Error Handling:**
- Invalid artifact format → Exit 1
- Artifact not found → Exit 1 (download-artifact fails)
- Load fails → Exit 1 (docker load fails)
- Image reference not extracted → Exit 1

## Error Conditions

- Malformed artifact input → Exit 1 (validation)
- Artifact doesn't exist → Exit 1 (download fails)
- Corrupted tar file → Exit 1 (docker load fails)
- Image reference parsing fails → Exit 1 (validation)

## Dependencies

- Docker CLI
- `actions/download-artifact@v4`
- Bash with parameter expansion
- `awk` for output parsing

## Example Flows

**Scenario: Load saved image**
```
Input: artifact=myapp-build/image.tar
→ Parse: name=myapp-build, file=image.tar
→ Download: actions/download-artifact (name=myapp-build) → .artifact/
→ Load: docker load --input .artifact/image.tar
→ Docker outputs: "Loaded image: myapp:build"
→ Extract: myapp:build
→ Output: image=myapp:build
```

**Scenario: Load registry image**
```
Input: artifact=ghcr-io-org-app-v1-2-3/image.tar
→ Parse: name=ghcr-io-org-app-v1-2-3, file=image.tar
→ Download: actions/download-artifact (name=ghcr-io-org-app-v1-2-3) → .artifact/
→ Load: docker load --input .artifact/image.tar
→ Docker outputs: "Loaded image: ghcr.io/org/app:v1.2.3"
→ Extract: ghcr.io/org/app:v1.2.3
→ Output: image=ghcr.io/org/app:v1.2.3
```

**Scenario: Invalid artifact format**
```
Input: artifact=image.tar
→ Parse: no '/' separator found
→ Error: "Artifact input must be in 'name/path' form"
→ Exit 1
```

## Integration Pattern

The action is designed to work with `artifact-from-image`:

```yaml
# Job 1: Build and save
build:
  outputs:
    artifact: ${{ steps.upload.outputs.artifact }}
  steps:
    - id: build
      run: docker build -t myapp:v1 .
    - id: upload
      uses: .../artifact-from-image
      with:
        image: myapp:v1
    # Output: artifact=myapp-v1/image.tar

# Job 2: Load and use
push:
  needs: build
  steps:
    - id: download
      uses: .../artifact-to-image
      with:
        artifact: ${{ needs.build.outputs.artifact }}
    # Output: image=myapp:v1

    - uses: .../docker-push
      with:
        image: ${{ steps.download.outputs.image }}
        tags: v1.0.0 latest
```

## Notes

- The action restores the original image name/tag (not the artifact name)
- Downloaded artifacts are cleaned up automatically after job completion
- The `.artifact/` directory is created automatically by `download-artifact`
- Image is immediately available in Docker after loading
