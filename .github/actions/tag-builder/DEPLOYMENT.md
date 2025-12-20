# Tag Builder - Deployment Guide

Technical reference for rebuilding the tag-builder action.

## Overview

Builds semantic version tags by comparing a current version against the latest tag from a target branch. Supports automatic patch increments and prevents version drift.

## Inputs & Outputs

**Inputs:**
- `target-branch`: Branch to fetch latest tag from
- `current-version`: Version to build (format: `[v]X.Y.Z[+postfix]`)
- `patch`: Enable auto-increment patch mode (default: `false`)

**Outputs:**
- `next-version`: Full version with postfix preserved
- `next-version-short`: Version without postfix
- `is-latest-version`: Boolean indicating if built version is latest globally

## Implementation Steps

### 1. Parse Current Version

Extract components from `current-version`:
- Prefix (optional `v` or `V`)
- Major, Minor, Patch numbers
- Postfix (optional `+...`)

Use regex: `^([vV]?)([0-9]+)\.([0-9]+)\.([0-9]+)(\+.+)?$`

Exit with error if format is invalid.

### 2. Discover Latest Tag from Target Branch

Fetch tags with `git tag --merged origin/{target-branch}`.

Filter for semantic versions matching the pattern, sort by version number (not chronological), and select the highest.

Default to `0.0.0` if no tags found.

Preserve the prefix from current version when constructing the latest tag reference.

### 3. Validate Version Not Older

Compare current vs latest:
- Reject if latest.major > current.major
- Reject if latest.minor > current.minor (when major equal)
- Note: Patch comparison happens later in build step

### 4. Get Global Latest Tag

Fetch all tags from repository using `git tag --list` (not filtered by branch).

Strip prefixes and postfixes, filter for semantic versions, sort, and select highest.

Default to `0.0.0` if no tags exist.

### 5. Build Next Version

**Patch Mode (`patch: true`):**

Check conditions in order:
1. If `current.major > latest.major` → Use `current.major.minor.0` (new major, reset patch)
2. If `current.minor > latest.minor` → Use `current.major.minor.0` (new minor, reset patch)
3. If `current.patch > latest.patch` → Use `current.major.minor.patch` (current is ahead, use it)
4. Else → Use `current.major.minor.(latest.patch + 1)` (auto-increment)

**Non-Patch Mode (`patch: false`):**
- Use current version as-is
- Additional validation: Reject if latest.patch >= current.patch (when major.minor equal)

Add postfix back to create full version output.

### 6. Determine if Latest Version

Compare built version (without prefix) against global latest tag:
- Strip prefix from built version
- Compare major, minor, patch components
- Return `true` if built version > global latest, otherwise `false`

### 7. Write Outputs

Output to `$GITHUB_OUTPUT`:
- `next_version`: Full version with postfix
- `next_version_short`: Version without postfix
- `is_latest_version`: Boolean (`true`/`false`)

Write summary to `$GITHUB_STEP_SUMMARY` showing target branch, latest tag, global latest, current version, patch mode, built versions, and is-latest flag.

## Key Behaviors

**Prefix Handling:**
- Detected from current-version
- Applied to all comparisons and outputs
- Examples: `v1.2.3` → prefix `v`, `1.2.3` → no prefix

**Postfix Handling:**
- Stripped during version comparison
- Stripped in next-version-short output
- Preserved in next-version output
- Used for build metadata (e.g., `+build-123`)

**Version Sorting:**
- Use semantic version sort (`sort -V`), not lexical
- Compare only major.minor.patch components
- Ignore postfixes when determining "latest"

## Error Conditions

- Invalid current-version format → Exit 1
- Latest version ahead of current → Exit 1
- Non-patch mode: Duplicate or older patch → Exit 1

## Dependencies

- Git CLI (for `git tag --merged`)
- Bash with regex support
- Repository with fetched tags

## Examples

| Latest Tag | Current | Patch | Output next-version-short | Notes                         |
|------------|---------|-------|---------------------------|-------------------------------|
| v1.2.3     | v1.2.4  | false | v1.2.4                    | Simple increment              |
| v1.2.3     | v1.2.5  | true  | v1.2.5                    | Current > latest, use current |
| v1.2.3     | v1.2.0  | true  | v1.2.4                    | Current < latest, increment   |
| v1.2.3     | v1.3.0  | true  | v1.3.0                    | New minor, patch reset        |
| v1.2.3     | v2.0.0  | true  | v2.0.0                    | New major, patch reset        |
| v1.2.3     | v1.2.3  | false | ERROR                     | Duplicate not allowed         |
| v0.0.0     | v1.0.1  | true  | v1.0.1                    | Current > latest, use current |
| (none)     | v1.0.0  | false | v1.0.0                    | First tag                     |

## Use Case: Latest Version Detection

The `is-latest-version` output enables conditional Docker image tagging with `:latest` based on whether the built version is the newest globally.

**Example Scenarios:**
- Main branch (v3.2.1) → next v3.2.2 → `is-latest-version: true`
- Maintenance branch v1 (v1.3.4) → next v1.3.5 → `is-latest-version: false` (global is v3.2.1)
- Newer branch v4 (v4.0.0) → next v4.0.1 → `is-latest-version: true` (v4.0.1 > v3.2.1)

