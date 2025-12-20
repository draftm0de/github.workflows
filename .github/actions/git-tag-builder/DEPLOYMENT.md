# Git Tag Builder - Deployment Guide

Technical reference for rebuilding the git-tag-builder action.

## Overview

Creates git tags based on version and target branch. Supports exact version tags and optional branch-level floating tags.

## Inputs & Outputs

**Inputs:**
- `version`: Version to tag (format: `[v]X.Y.Z[+postfix]`)
- `target-branch`: Branch name (e.g., `main`, `v1.2`)
- `enable-branch-tag`: Enable branch tagging (default: `true`)
- `git-tag-levels`: Comma-separated levels (default: `''` - empty/none)

**Outputs:**
- `git-tags`: Space-separated list of created tags
- `exact-tag`: Exact version tag (e.g., `v1.2.12`)
- `branch-tag`: Branch-level tag (e.g., `v1.2`), empty if not created

## Implementation Steps

### 1. Parse and Validate Version

Extract components from `version`:
- Prefix (optional `v` or `V`)
- Major, Minor, Patch numbers
- Postfix (optional `+...`)

Use regex: `^([vV]?)([0-9]+)\.([0-9]+)\.([0-9]+)(\+.+)?$`

Exit with error if format is invalid.

### 2. Check Version-Like Branch Coverage

Determine if the target branch is version-like and which tag levels it covers:
- Branch `v1.2` matching version `1.2.X` → covers major AND minor
- Branch `v1` matching version `1.X.X` → covers major only
- Branch `main` or non-matching → covers nothing

Set flags: `BRANCH_COVERS_MAJOR`, `BRANCH_COVERS_MINOR`

### 3. Create Exact Version Tag

Strip postfix from version and create exact tag:
- Tag name: `{prefix}{major}.{minor}.{patch}`
- Example: `v1.2.12+build` → tag `v1.2.12`

Check if tag already exists:
- If exists → Error and exit
- If not exists → Create tag with `git tag`

### 4. Create Multi-Level Tags

Parse `git-tag-levels` and for each level:

**patch:**
- Already created as exact tag, skip

**minor:**
- If `BRANCH_COVERS_MINOR == true` → skip (branch tag handles this)
- Else → add `{prefix}{major}.{minor}` to tags list

**major:**
- If `BRANCH_COVERS_MAJOR == true` → skip (branch tag handles this)
- Else → add `{prefix}{major}` to tags list

**latest:**
- Emit warning and skip (not allowed)

### 5. Determine Branch Tag (if enabled)

Check if `enable-branch-tag` is `true` and if branch name is version-like.

**Pattern matching:**
- Major.Minor format: `^([vV]?)([0-9]+)\.([0-9]+)$` (e.g., `v1.2`)
- Major-only format: `^([vV]?)([0-9]+)$` (e.g., `v1`)

**Version matching:**
- For major.minor branches: Branch major.minor must match version major.minor
- For major-only branches: Branch major must match version major
- If no match → Skip branch tag

### 6. Add Branch Tag to Output

If branch tag should be created, add it to tags list.

Note: Action does NOT create tags locally or remotely, only calculates them.

### 7. Write Outputs

Output to `$GITHUB_OUTPUT`:
- `exact_tag`: The exact version tag
- `branch_tag`: The branch tag (or empty)
- `git_tags`: Space-separated list of both

Write summary to `$GITHUB_STEP_SUMMARY` showing target branch, exact tag, branch tag, and all created tags.

## Key Behaviors

**Exact Tags:**
- Immutable - error if already exists
- Always created
- Postfix stripped

**Multi-Level Tags:**
- `minor` and `major` are floating - move to latest patch/minor
- Automatically filtered if covered by branch tags
- Can be disabled by omitting from `git-tag-levels`

**Branch Tags:**
- Floating - move to latest version
- Optional - only if branch is version-like
- Automatically filter out redundant multi-level tags
- Version must match branch

**Tag Filtering Logic:**
| Branch | Version | Levels | Tags Created | Filtered |
|--------|---------|--------|--------------|----------|
| `main` | `v1.2.12` | `patch,minor,major` | `v1.2.12`, `v1.2`, `v1` | None |
| `v1.2` | `v1.2.12` | `patch,minor,major` | `v1.2.12`, `v1.2`, `v1` | Minor covered by branch |
| `v1` | `v1.5.0` | `patch,minor,major` | `v1.5.0`, `v1.5`, `v1` | Major covered by branch |
| `v1.2` | `v2.0.0` | `patch,minor,major` | `v2.0.0`, `v2.0`, `v2` | Branch doesn't match version |

## Error Conditions

- Invalid version format → Exit 1
- Exact tag already exists → Exit 1
- Invalid tag level in `git-tag-levels` → Exit 1 (action fails)

## Dependencies

- Git CLI for tag checking (`git rev-parse`)
- Bash with regex support
- Repository checked out (read-only access sufficient)

## Example Flows

**Scenario: Multi-level tags with version branch**
```
Input: version=v1.2.12, target-branch=v1.2, levels=patch,minor,major
→ Calculate exact tag: v1.2.12
→ Branch is version-like: v1.2 (covers major + minor)
→ Skip minor tag (covered by branch)
→ Skip major tag (covered by branch)
→ Add branch tag: v1.2
→ Output: git-tags="v1.2.12 v1.2"
```

**Scenario: Multi-level tags without version branch**
```
Input: version=v1.2.12, target-branch=main, levels=patch,minor,major
→ Calculate exact tag: v1.2.12
→ Branch is not version-like
→ Add minor tag: v1.2
→ Add major tag: v1
→ Output: git-tags="v1.2.12 v1.2 v1"
```

**Scenario: Default (patch only)**
```
Input: version=v1.2.12, target-branch=main, levels=patch
→ Calculate exact tag: v1.2.12
→ Branch is not version-like
→ No multi-level tags requested
→ Output: git-tags="v1.2.12"
```
