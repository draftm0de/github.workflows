# Git Tag Builder - Deployment Guide

Technical reference for rebuilding the git-tag-builder action.

## Overview

Creates git tags based on version and target branch. Supports exact version tags and optional branch-level floating tags.

## Inputs & Outputs

**Inputs:**
- `version`: Version to tag (format: `[v]X.Y.Z[+postfix]`)
- `target-branch`: Branch name (e.g., `main`, `v1.2`)
- `enable-branch-tag`: Enable branch tagging (default: `true`)

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

### 2. Create Exact Version Tag

Strip postfix from version and create exact tag:
- Tag name: `{prefix}{major}.{minor}.{patch}`
- Example: `v1.2.12+build` → tag `v1.2.12`

Check if tag already exists:
- If exists → Error and exit
- If not exists → Create tag with `git tag`

### 3. Determine Branch Tag (if enabled)

Check if `enable-branch-tag` is `true` and if branch name is version-like.

**Pattern matching:**
- Major.Minor format: `^([vV]?)([0-9]+)\.([0-9]+)$` (e.g., `v1.2`)
- Major-only format: `^([vV]?)([0-9]+)$` (e.g., `v1`)

**Version matching:**
- For major.minor branches: Branch major.minor must match version major.minor
- For major-only branches: Branch major must match version major
- If no match → Skip branch tag

### 4. Create/Update Branch Tag

If branch tag should be created:

1. Check if tag already exists
2. If exists:
   - Delete local tag: `git tag -d {branch-tag}`
   - Delete remote tag: `git push origin :refs/tags/{branch-tag}`
3. Create new tag: `git tag {branch-tag}`

Branch tags are "floating" - they move to the latest version on that branch.

### 5. Write Outputs

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

**Branch Tags:**
- Floating - move to latest version
- Optional - only if branch is version-like
- Updated if already exists
- Version must match branch

**Version Matching Logic:**
| Branch | Version | Match? | Branch Tag Created? |
|--------|---------|--------|---------------------|
| `v1.2` | `v1.2.12` | ✅ Yes | Yes - `v1.2` |
| `v1.2` | `v2.0.0` | ❌ No | No |
| `v1` | `v1.5.0` | ✅ Yes | Yes - `v1` |
| `v1` | `v2.0.0` | ❌ No | No |
| `main` | `v1.2.12` | N/A | No (not version-like) |

## Error Conditions

- Invalid version format → Exit 1
- Exact tag already exists → Exit 1
- Branch tag creation never fails (skipped if criteria not met)

## Dependencies

- Git CLI for tag creation
- Bash with regex support
- Repository checked out with write access

## Example Flows

**Scenario: Update v1.2 branch**
```
Input: version=v1.2.12, target-branch=v1.2
→ Create exact tag: v1.2.12
→ Branch is version-like: v1.2
→ Version matches: 1.2 == 1.2
→ Update branch tag: v1.2
→ Output: git-tags="v1.2.12 v1.2"
```

**Scenario: Version doesn't match branch**
```
Input: version=v2.0.0, target-branch=v1.2
→ Create exact tag: v2.0.0
→ Branch is version-like: v1.2
→ Version doesn't match: 2.0 != 1.2
→ Skip branch tag
→ Output: git-tags="v2.0.0"
```

**Scenario: Non-version branch**
```
Input: version=v1.2.12, target-branch=main
→ Create exact tag: v1.2.12
→ Branch is not version-like
→ Skip branch tag
→ Output: git-tags="v1.2.12"
```
