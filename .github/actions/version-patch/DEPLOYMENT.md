# Version Patch - Deployment Guide

Technical reference for rebuilding the version-patch action.

## Overview

Updates version files (`package.json` or `pubspec.yaml`) with a new version and commits the change to the current branch. Only runs for `nodejs` and `flutter` source types.

## Inputs & Outputs

**Inputs:**
- `ci-tag-source`: Source type (`nodejs`, `flutter`, or `branch`)
- `new-version`: Version to write (format: `[v]X.Y.Z`)

**Outputs:**
- None (action performs git commit and push)

## Implementation Steps

### 1. Parse Inputs and Strip Version Prefix

Extract inputs:
- `SOURCE` = `ci-tag-source`
- `VERSION` = `new-version`

Strip `v` prefix if present:
```bash
CLEAN_VERSION="${VERSION#v}"
```

### 2. Update Version File Based on Source Type

**For `ci-tag-source: nodejs`:**

1. Check if `package.json` exists in repository root
   - If not found → Error and exit 1
2. Verify `jq` is installed
   - If not found → Error and exit 1
3. Update version field using `jq`:
   ```bash
   jq --arg version "$CLEAN_VERSION" '.version = $version' package.json > package.json.tmp
   mv package.json.tmp package.json
   ```
4. Set output: `file=package.json`

**For `ci-tag-source: flutter`:**

1. Check if `pubspec.yaml` exists in repository root
   - If not found → Error and exit 1
2. Update version field using `sed`:
   ```bash
   sed -i.bak "s/^version: .*/version: $CLEAN_VERSION/" pubspec.yaml
   rm -f pubspec.yaml.bak
   ```
3. Set output: `file=pubspec.yaml`

**For `ci-tag-source: branch`:**

1. Log notice: "Skipping version file update for ci-tag-source=branch"
2. Set output: `file=` (empty)
3. Exit 0 (success, skip remaining steps)

**For unknown source:**
- Error and exit 1

### 3. Commit and Push Version Update

**Skip if no file was updated:**
- Check if `file` output is empty
- If empty → Skip this step

**Configure git identity:**
```bash
git config user.name "github-actions[bot]"
git config user.email "github-actions[bot]@users.noreply.github.com"
```

**Stage the updated file:**
```bash
git add "$FILE"
```

**Check for changes:**
```bash
if git diff --cached --quiet; then
  # No changes to commit
  exit 0
fi
```

**Commit with version in message:**
```bash
git commit -m "chore: bump version to $VERSION"
```

Format: `chore: bump version to {clean_version}`

**Push to current branch:**
```bash
git push origin HEAD
```

### 4. Write Summary

Write to `$GITHUB_STEP_SUMMARY`:
- Source type
- New version
- File updated

## Key Behaviors

**Version Prefix Handling:**
- Input `v1.2.4` → Writes `1.2.4` to file
- Input `1.2.4` → Writes `1.2.4` to file

**File Update Methods:**
- **Node.js**: Uses `jq` for safe JSON manipulation
- **Flutter**: Uses `sed` for YAML field replacement

**Commit Behavior:**
- Only commits if file has changes
- Skips commit if version already matches
- Uses `github-actions[bot]` as author

**Push Behavior:**
- Pushes to current branch HEAD
- Typically runs on `main` branch
- Requires `contents: write` permission

## Error Conditions

- Unknown `ci-tag-source` → Exit 1
- Required file not found (`package.json` or `pubspec.yaml`) → Exit 1
- `jq` not installed (nodejs only) → Exit 1
- Git commit/push failure → Exit 1 (via `set -euo pipefail`)

## Dependencies

**System Tools:**
- `bash` with `set -euo pipefail`
- `git` CLI
- `jq` (for nodejs source type)
- `sed` (for flutter source type)

**Repository State:**
- Repository must be checked out
- Write access to current branch
- `contents: write` permission required

**GitHub-hosted runners:**
- `jq` is pre-installed ✅
- `sed` is pre-installed ✅
- `git` is pre-installed ✅

## Integration Flow

This action is designed to run **before** git tagging in workflows:

```
1. Checkout repository (fetch-depth: 0)
2. Read current version (version-reader)
3. Calculate next version (tag-builder)
4. **Update version file (version-patch)** ← This action
5. Create git tags (git-tag-builder)
6. Push tags (git-push)
```

**Why before tagging?**
- Ensures git tags point to commits with correct version in files
- Version file and git tag stay synchronized
- Downstream builds can trust the version in files matches git tags

## Example Scenarios

**Scenario 1: Node.js patch increment**
```
Input: ci-tag-source=nodejs, new-version=1.2.4
→ Update package.json: "version": "1.2.4"
→ Commit: "chore: bump version to 1.2.4"
→ Push to main
```

**Scenario 2: Flutter version update**
```
Input: ci-tag-source=flutter, new-version=v2.1.0
→ Update pubspec.yaml: version: 2.1.0
→ Commit: "chore: bump version to 2.1.0"
→ Push to main
```

**Scenario 3: Branch-based versioning (skip)**
```
Input: ci-tag-source=branch, new-version=1.2.4
→ Skip file update
→ No commit
→ No push
```

**Scenario 4: Version already up to date**
```
Input: ci-tag-source=nodejs, new-version=1.2.4
→ Update package.json: "version": "1.2.4"
→ Check git diff: no changes
→ Skip commit
→ Log notice and exit 0
```

## Permissions Required

```yaml
permissions:
  contents: write  # Required to commit and push
```

## Notes

- Action uses `git push origin HEAD` to push to current branch
- Commit message follows Conventional Commits format (`chore:`)
- Version prefix is always stripped before writing to files
- Action is idempotent - safe to run multiple times with same version
- For `ci-tag-source: branch`, action exits early without side effects
