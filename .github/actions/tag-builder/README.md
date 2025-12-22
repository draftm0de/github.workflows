# Tag Builder Action

Builds and validates semantic version tags based on a current version and the latest tag from a branch. Prevents version drift and supports automatic patch increments.

## Inputs

| Name | Description | Required | Default |
|------|-------------|----------|---------|
| `branch` | Target branch name (e.g., `main`, `develop`) | Yes | - |
| `current-version` | Current version with optional postfix (e.g., `v1.0.1+1`, `1.0.1`) | Yes | - |
| `patch` | Enable patch mode to auto-increment patch version | No | `'false'` |
| `ci-tag-source` | Version source type (`nodejs`, `flutter`, `branch`). Affects versioning behavior. | No | `''` |

## Outputs

| Name | Description |
|------|-------------|
| `next-version` | Built version with postfix if present (e.g., `v1.0.1+1`) |
| `next-version-short` | Built version without postfix (e.g., `v1.0.1`) |
| `is-latest-version` | Whether the built version is the latest across all branches (`true`/`false`) |

## Usage

### Basic Usage

```yaml
- name: Build version tag
  id: tag_builder
  uses: draftm0de/github.workflows/.github/actions/tag-builder@main
  with:
    branch: main
    current-version: v1.2.3
    patch: 'false'

- name: Create git tag
  run: |
    git tag ${{ steps.tag_builder.outputs.next-version-short }}
    git push origin ${{ steps.tag_builder.outputs.next-version-short }}
```

### Patch Mode

```yaml
- name: Build patch version
  id: tag_builder
  uses: draftm0de/github.workflows/.github/actions/tag-builder@main
  with:
    branch: main
    current-version: v1.2.3+build-123
    patch: 'true'

- name: Use built version
  run: |
    echo "Next Version: ${{ steps.tag_builder.outputs.next-version }}"
    echo "Next Version Short: ${{ steps.tag_builder.outputs.next-version-short }}"
    echo "Is Latest: ${{ steps.tag_builder.outputs.is-latest-version }}"
```

## How It Works

**Version Validation**
- Ensures the current version is not older than the latest tag on the branch
- Prevents version drift by checking major, minor, and patch components

**Patch Mode (`patch: 'true'`)**

For `ci-tag-source: nodejs/flutter`:
- Always increments patch from current file version

For `ci-tag-source: branch` (or empty):
- New major version: Resets patch to `0`
- New minor version: Resets patch to `0`
- No existing tags: Increments patch from current version
- Same major.minor: Increments patch from current version

**Non-Patch Mode (`patch: 'false'`)**
- Always uses the current version as-is
- No validation against git tags

**Postfix Handling**
- Postfixes (e.g., `+build-123`) are preserved in `next-version`
- Postfixes are stripped in `next-version-short`

**Latest Version Detection**
- Compares the built version against all tags in the repository (not just branch)
- Returns `true` if the built version is greater than any existing tag globally
- Useful for conditional Docker image tagging with `:latest`

## Example Scenarios

| ci-tag-source | Latest Tag | Current Version | Patch Mode | Result | Notes |
|---------------|------------|-----------------|------------|--------|-------|
| (any) | (any) | `v1.2.4` | `false` | `v1.2.4` | Non-patch always uses current |
| `nodejs` | (any) | `v1.0.1` | `true` | `v1.0.2` | File-based: increment from file |
| `flutter` | (any) | `v2.3.5` | `true` | `v2.3.6` | File-based: increment from file |
| `branch` | `v1.2.3` | `v1.2.0` | `true` | `v1.2.1` | Git-based: increment from current |
| `branch` | `v1.2.3` | `v1.3.0` | `true` | `v1.3.0` | New minor, patch reset to 0 |
| `branch` | `v1.2.3` | `v2.0.0` | `true` | `v2.0.0` | New major, patch reset to 0 |
| `branch` | `v0.0.0` | `v1.0.1` | `true` | `v1.0.2` | No tags, increment from current |

## Notes

- Supported version formats: `v1.0.1`, `1.0.1`, `v1.0.1+build`, `1.0.1+build`
- The `v` prefix from `current-version` is preserved in outputs
- Repository must be checked out with tags fetched
- See [DEPLOYMENT.md](DEPLOYMENT.md) for detailed implementation logic
