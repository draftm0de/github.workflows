# Tag Builder Action

Composite action that builds and validates semantic version tags based on a current version and the latest tag from a target branch. It prevents version drift by ensuring the current version is not older than what already exists in the target branch's git history. The action supports patch mode for automatic patch-level increments and preserves version postfixes (like `+1`, `+2`) from the input version.

## Inputs

| Name               | Description                                                                 | Required | Default   |
|--------------------|-----------------------------------------------------------------------------|----------|-----------|
| `target-branch`    | Target branch name to fetch the latest tag from (e.g., `main`, `develop`)  | Yes      | -         |
| `current-version`  | Current version with optional postfix (e.g., `v1.0.1+1`, `1.0.1`, `v1.2.3`)| Yes      | -         |
| `patch`            | Enable patch mode to auto-increment patch version                           | No       | `'false'` |

## Outputs

| Name             | Description |
|------------------|-------------|
| `version`        | Built version with postfix if present in input (e.g., `v1.0.1+1`). This is the full version string that preserves any postfix metadata from the input. |
| `version-short`  | Built version without postfix (e.g., `v1.0.1`). This is the clean semantic version suitable for git tagging. |

## Usage

```yaml
steps:
  - name: Build version tag
    id: tag_builder
    uses: draftm0de/github.workflows/.github/actions/tag-builder@main
    with:
      target-branch: main
      current-version: v1.2.3
      patch: 'false'

  - name: Create git tag
    run: |
      git tag ${{ steps.tag_builder.outputs.version-short }}
      git push origin ${{ steps.tag_builder.outputs.version-short }}
```

### Patch Mode Example

```yaml
steps:
  - name: Build patch version
    id: tag_builder
    uses: draftm0de/github.workflows/.github/actions/tag-builder@main
    with:
      target-branch: main
      current-version: v1.2.3+build-123
      patch: 'true'

  - name: Use built version
    run: |
      echo "Version: ${{ steps.tag_builder.outputs.version }}"
      echo "Version Short: ${{ steps.tag_builder.outputs.version-short }}"
```

## Behavior

### Version Parsing
The action accepts versions in these formats:
- `1.0.1` - plain semantic version
- `v1.0.1` - semantic version with `v` prefix
- `1.0.1+1` - semantic version with postfix
- `v1.0.1+build-123` - semantic version with `v` prefix and postfix

The prefix (`v` or empty) from `current-version` determines the prefix used in outputs. Any postfix (characters after `+`) is preserved in the `version` output but stripped from `version-short`.

### Latest Tag Discovery
The action fetches all tags from the `target-branch` and identifies the latest semantic version tag:
- Tags are filtered to match the prefix pattern from `current-version`
- Only valid semantic versions (`X.Y.Z` with optional prefix/postfix) are considered
- The latest tag is determined by semantic version ordering (not chronological)
- If no tags are found, defaults to `v0.0.0` or `0.0.0` based on input prefix

### Version Validation
Before building the output version, the action validates that the current version is not older than the latest tag:

1. **Major version check**: If `latest.major > current.major`, the action exits with error (prevents downgrade)
2. **Minor version check**: If `latest.major == current.major && latest.minor > current.minor`, the action exits with error
3. **Patch version check** (non-patch mode only): If `latest.major == current.major && latest.minor == current.minor && latest.patch >= current.patch`, the action exits with error

### Patch Mode Logic
When `patch: 'true'`:

1. **New major version** (`current.major > latest.major`):
   - Output: `current.major.current.minor.0`
   - Patch is reset to `0` for new major versions

2. **New minor version** (`current.major == latest.major && current.minor > latest.minor`):
   - Output: `current.major.current.minor.0`
   - Patch is reset to `0` for new minor versions

3. **Same major.minor** (`current.major == latest.major && current.minor == latest.minor`):
   - Output: `current.major.current.minor.(latest.patch + 1)`
   - Patch is incremented from the latest tag

### Non-Patch Mode Logic
When `patch: 'false'` (default):

- Output: `current.major.current.minor.current.patch`
- Uses the exact version from `current-version` input
- Additional validation ensures `latest.patch` is not `>=` `current.patch` when major and minor are equal
- Prevents creating duplicate or downgraded tags

### Postfix Handling
The postfix from `current-version` (everything after `+`) is:
- **Stripped** when calculating `version-short`
- **Stripped** when comparing against latest tag
- **Preserved** in the final `version` output
- Used to carry build metadata, commit hashes, or build numbers

### Example Scenarios

| Latest Tag | Current Version | Patch Mode | Output version-short | Output version    | Notes |
|------------|----------------|------------|---------------------|-------------------|-------|
| `v1.2.3`   | `v1.2.4`       | `false`    | `v1.2.4`            | `v1.2.4`          | Simple increment |
| `v1.2.3`   | `v1.2.4+10`    | `false`    | `v1.2.4`            | `v1.2.4+10`       | Postfix preserved |
| `v1.2.3`   | `v1.2.0`       | `true`     | `v1.2.4`            | `v1.2.4`          | Patch auto-incremented |
| `v1.2.3`   | `v1.3.0`       | `true`     | `v1.3.0`            | `v1.3.0`          | New minor, patch reset |
| `v1.2.3`   | `v2.0.0`       | `true`     | `v2.0.0`            | `v2.0.0`          | New major, patch reset |
| `v1.2.3`   | `v1.2.3`       | `false`    | ERROR               | ERROR             | Same version not allowed |
| `v1.2.5`   | `v1.2.4`       | `false`    | ERROR               | ERROR             | Downgrade prevented |
| (none)     | `v1.0.0`       | `false`    | `v1.0.0`            | `v1.0.0`          | First tag created |

## Dependencies

This action requires:
- `git` - for fetching tags and repository operations
- Standard bash shell with regex support
- Access to the target branch (usually via prior checkout action)

The action uses `git tag --merged origin/<target-branch>` to discover tags, so ensure the repository has been fetched with tags enabled (`git fetch --tags`). Each run appends a summary to the GitHub Actions job summary showing the target branch, latest tag, current version, patch mode setting, and the built version outputs.
