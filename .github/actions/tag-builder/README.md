# Tag Builder Action

Builds and validates semantic version tags based on a current version and the latest tag from a target branch. Prevents version drift and supports automatic patch increments.

## Inputs

| Name | Description | Required | Default |
|------|-------------|----------|---------|
| `target-branch` | Target branch name (e.g., `main`, `develop`) | Yes | - |
| `current-version` | Current version with optional postfix (e.g., `v1.0.1+1`, `1.0.1`) | Yes | - |
| `patch` | Enable patch mode to auto-increment patch version | No | `'false'` |

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
    target-branch: main
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
    target-branch: main
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
- Ensures the current version is not older than the latest tag on the target branch
- Prevents version drift by checking major, minor, and patch components

**Patch Mode (`patch: 'true'`)**
- New major version: Resets patch to `0`
- New minor version: Resets patch to `0`
- Current patch > latest patch: Uses current patch
- Current patch ≤ latest patch: Increments patch from latest tag

**Non-Patch Mode (`patch: 'false'`)**
- Uses the current version as-is
- Validates that it's not a duplicate or older than the latest tag

**Postfix Handling**
- Postfixes (e.g., `+build-123`) are preserved in `next-version`
- Postfixes are stripped in `next-version-short`

**Latest Version Detection**
- Compares the built version against all tags in the repository (not just target branch)
- Returns `true` if the built version is greater than any existing tag globally
- Useful for conditional Docker image tagging with `:latest`

## Example Scenarios

| Latest Tag | Current Version | Patch Mode | Result | Notes |
|------------|-----------------|------------|--------|-------|
| `v1.2.3` | `v1.2.4` | `false` | `v1.2.4` | Uses current as-is |
| `v1.2.3` | `v1.2.5` | `true` | `v1.2.5` | Current patch > latest, uses current |
| `v1.2.3` | `v1.2.0` | `true` | `v1.2.4` | Current patch < latest, auto-increments |
| `v1.2.3` | `v1.3.0` | `true` | `v1.3.0` | New minor, patch reset to 0 |
| `v1.2.3` | `v2.0.0` | `true` | `v2.0.0` | New major, patch reset to 0 |
| `v1.2.3` | `v1.2.3` | `false` | ERROR | Duplicate version |
| `v0.0.0` | `v1.0.1` | `true` | `v1.0.1` | Current patch > latest, uses current |

## Notes

- Supported version formats: `v1.0.1`, `1.0.1`, `v1.0.1+build`, `1.0.1+build`
- The `v` prefix from `current-version` is preserved in outputs
- Repository must be checked out with tags fetched
- See [DEPLOYMENT.md](DEPLOYMENT.md) for detailed implementation logic
