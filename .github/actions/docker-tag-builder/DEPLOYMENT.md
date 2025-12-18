# Docker Tag Builder - Deployment Guide

Technical reference for rebuilding the docker-tag-builder action.

## Overview

Builds Docker image tags based on version, target branch, and global latest version detection. Outputs comma-separated tags for Docker operations.

## Inputs & Outputs

**Inputs:**
- `version`: Version to tag (format: `[v]X.Y.Z[+postfix]`)
- `is-latest-version`: Boolean indicating if this is globally latest
- `tag-levels`: Comma-separated levels: `patch`, `minor`, `major`, `latest` (default: `patch`)

**Outputs:**
- `docker-tags`: Comma-separated list of tags
- `exact-tag`: Exact version tag
- `latest-tag`: Latest tag (or empty)

## Implementation Steps

### 1. Parse and Validate Version

Extract components from `version`:
- Prefix (optional `v` or `V`)
- Major, Minor, Patch numbers
- Postfix (optional `+...`)

Use regex: `^([vV]?)([0-9]+)\.([0-9]+)\.([0-9]+)(\+.+)?$`

Exit with error if format is invalid.

### 2. Process Tag Levels

Parse `tag-levels` input (comma-separated) and build tags list:

**For each level:**
- `patch`: Add `{prefix}{major}.{minor}.{patch}`
- `minor`: Add `{prefix}{major}.{minor}`
- `major`: Add `{prefix}{major}`
- `latest`: Add `latest` (only if `is-latest-version: true`)

Strip postfix from all tags.

Fallback: If no valid levels provided, default to exact tag.

### 3. Build Output

Combine all tags into comma-separated string:
- Format: `tag1,tag2,tag3`
- Example: `v1.2.12,v1.2,latest`

Output individual components and combined list to `$GITHUB_OUTPUT`.

Write summary to `$GITHUB_STEP_SUMMARY`.

## Tag Levels Logic

| tag-levels | Version v1.2.12 | Output Tags |
|------------|-----------------|-------------|
| `patch` | v1.2.12 | `v1.2.12` |
| `patch,minor` | v1.2.12 | `v1.2.12,v1.2` |
| `patch,minor,major` | v1.2.12 | `v1.2.12,v1.2,v1` |
| `patch,minor,major,latest` (is-latest=true) | v1.2.12 | `v1.2.12,v1.2,v1,latest` |
| `minor,major` | v1.2.12 | `v1.2,v1` |
| `latest` (is-latest=true) | v1.2.12 | `latest` |

## Example Outputs

| Tag Levels | Version | Is Latest | Output Tags | Notes |
|------------|---------|-----------|-------------|-------|
| `patch` | v1.2.12 | false | `v1.2.12` | Default, exact version only |
| `patch,minor` | v1.2.12 | false | `v1.2.12,v1.2` | Maintenance branch pattern |
| `patch,minor,major` | v3.2.2 | false | `v3.2.2,v3.2,v3` | All version levels |
| `patch,minor,major,latest` | v3.2.2 | true | `v3.2.2,v3.2,v3,latest` | Main branch, full tagging |
| `patch,latest` | v3.2.2 | true | `v3.2.2,latest` | Simple latest tagging |
| `minor,major` | v1.2.12 | false | `v1.2,v1` | Skip patch level |
| `latest` | v3.2.2 | false | `v3.2.2` | Latest not added (not globally latest) |

## Key Behaviors

**Comma-Separated Output:**
- Easy to parse and iterate in Docker commands
- No spaces in output

**Postfix Handling:**
- Always stripped for Docker tags
- Docker tags should be clean semver

**Latest Tag Logic:**
- Only when globally newest version
- Requires `is-latest-version: true`

## Error Conditions

- Invalid version format → Exit 1
- No other errors (tags are optional and skipped gracefully)

## Dependencies

- Bash with regex support
- No external tools required

## Integration Example

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
    IFS=',' read -ra TAGS <<< "$docker-tags"
    for tag in "${TAGS[@]}"; do
      docker tag image:build image:$tag
      docker push image:$tag
    done
```
