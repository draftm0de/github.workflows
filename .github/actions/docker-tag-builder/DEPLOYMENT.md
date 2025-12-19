# Docker Tag Builder - Deployment Guide

Technical reference for rebuilding the docker-tag-builder action.

## Overview

Builds Docker image tags based on optional version and global latest version detection. Supports semantic version levels (major, minor, patch) and custom tag levels. Outputs space-separated tags for Docker operations.

## Inputs & Outputs

**Inputs:**
- `version`: Optional version to tag (format: `[v]X.Y.Z[+postfix]`). Required for `major`, `minor`, `patch` levels.
- `is-latest-version`: Boolean indicating if this is globally latest
- `tag-levels`: Comma-separated levels: `patch`, `minor`, `major`, `latest`, or custom tags (default: `patch`)

**Outputs:**
- `docker-tags`: Space-separated list of tags
- `exact-tag`: Exact version tag (empty if no version provided)
- `latest-tag`: Latest tag (or empty)

## Implementation Steps

### 1. Parse and Validate Version (if provided)

**If `version` is provided:**
- Extract components from `version`:
  - Prefix (optional `v` or `V`)
  - Major, Minor, Patch numbers
  - Postfix (optional `+...`)
- Use regex: `^([vV]?)([0-9]+)\.([0-9]+)\.([0-9]+)(\+.+)?$`
- Exit with error if format is invalid
- Set `EXACT_TAG` to `{prefix}{major}.{minor}.{patch}` (postfix stripped)

**If `version` is empty:**
- Set all version variables to empty
- Continue (will fail if semantic levels requested)

### 2. Process Tag Levels

Parse `tag-levels` input (comma-separated) and build tags list:

**For each level:**
- `patch`:
  - Check if `version` provided, exit 1 if not
  - Add `{prefix}{major}.{minor}.{patch}`
- `minor`:
  - Check if `version` provided, exit 1 if not
  - Add `{prefix}{major}.{minor}`
- `major`:
  - Check if `version` provided, exit 1 if not
  - Add `{prefix}{major}`
- `latest`:
  - Add `latest` (only if `is-latest-version: true`)
  - If not latest, log notice and skip
- **Any other string** (custom tag):
  - Add as-is (e.g., `sha`, `edge`, `beta`, `pr-123`)
  - Log notice: "Adding custom tag: {level}"

**Error handling:**
- If semantic level (`major`, `minor`, `patch`) requested without version: Exit 1
- If no tags generated: Exit 1

### 3. Build Output

Combine all tags into space-separated string:
- Format: `tag1 tag2 tag3`
- Example: `v1.2.12 v1.2 latest`

Determine if `latest` is in output:
- Set `LATEST_TAG` to `latest` if present, empty otherwise

Output to `$GITHUB_OUTPUT`:
- `exact_tag`: Exact version (or empty)
- `latest_tag`: `latest` or empty
- `docker_tags`: Space-separated tag list

Write summary to `$GITHUB_STEP_SUMMARY` including:
- Version (or "none")
- Is latest version
- Tag levels
- Calculated tags (bulleted)

## Tag Levels Logic

| tag-levels | Version | Is Latest | Output Tags |
|------------|---------|-----------|-------------|
| `patch` | v1.2.12 | false | `v1.2.12` |
| `patch,minor` | v1.2.12 | false | `v1.2.12 v1.2` |
| `patch,minor,major` | v1.2.12 | false | `v1.2.12 v1.2 v1` |
| `patch,minor,major,latest` | v1.2.12 | true | `v1.2.12 v1.2 v1 latest` |
| `minor,major` | v1.2.12 | false | `v1.2 v1` |
| `latest` | v1.2.12 | true | `latest` |
| `sha,edge` | (none) | false | `sha edge` |
| `patch,sha` | v1.2.12 | false | `v1.2.12 sha` |
| `beta,pr-123` | (none) | false | `beta pr-123` |

## Example Outputs

| Tag Levels | Version | Is Latest | Output Tags | Notes |
|------------|---------|-----------|-------------|-------|
| `patch` | v1.2.12 | false | `v1.2.12` | Default, exact version only |
| `patch,minor` | v1.2.12 | false | `v1.2.12 v1.2` | Maintenance branch pattern |
| `patch,minor,major` | v3.2.2 | false | `v3.2.2 v3.2 v3` | All version levels |
| `patch,minor,major,latest` | v3.2.2 | true | `v3.2.2 v3.2 v3 latest` | Main branch, full tagging |
| `patch,latest` | v3.2.2 | true | `v3.2.2 latest` | Simple latest tagging |
| `minor,major` | v1.2.12 | false | `v1.2 v1` | Skip patch level |
| `latest` | v3.2.2 | false | (empty) | Latest not added (not globally latest) |
| `sha,edge` | (none) | false | `sha edge` | Custom tags, no version |
| `patch,sha` | v1.2.12 | false | `v1.2.12 sha` | Mixed semantic + custom |
| `beta` | (none) | false | `beta` | Single custom tag |

## Key Behaviors

**Space-Separated Output:**
- Easy to iterate in bash loops
- Compatible with Docker tag commands

**Postfix Handling:**
- Always stripped for Docker tags
- Docker tags should be clean semver

**Latest Tag Logic:**
- Only added when globally newest version
- Requires `is-latest-version: true`

**Custom Tags:**
- Any tag level not in `patch`, `minor`, `major`, `latest` is treated as custom
- Custom tags are passed through as-is
- No version validation for custom tags

**Version Requirement:**
- Version optional unless using `major`, `minor`, or `patch` levels
- Action fails if semantic level requested without version

## Error Conditions

- Invalid version format (when provided) → Exit 1
- Semantic level (`major`, `minor`, `patch`) without version → Exit 1
- No tags generated → Exit 1

## Dependencies

- Bash with regex support
- No external tools required

## Integration Examples

### Semantic Version Tags

```yaml
# Step 1: Build version
- uses: tag-builder
  outputs: next-version-short, is-latest-version

# Step 2: Build Docker tags
- uses: docker-tag-builder
  inputs: version, is-latest-version, tag-levels
  outputs: docker-tags

# Step 3: Use tags for Docker
- run: |
    for tag in $docker-tags; do
      docker tag image:build image:$tag
      docker push image:$tag
    done
```

### Custom Tags Only

```yaml
# Build Docker tags without version
- uses: docker-tag-builder
  with:
    is-latest-version: 'false'
    tag-levels: 'sha,edge'
  outputs: docker-tags

# Use custom tags
- run: |
    for tag in $docker-tags; do
      docker tag image:build image:$tag
      docker push image:$tag
    done
```
