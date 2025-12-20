# Version Patch Action

Updates `package.json` or `pubspec.yaml` with a new version and commits the change to the current branch.

## Inputs

| Name            | Description                                    | Required |
|-----------------|------------------------------------------------|----------|
| `ci-tag-source` | Version source type: `nodejs`, `flutter`, or `branch` | Yes      |
| `new-version`   | New version to write (e.g., `1.2.4`, `v1.2.4`) | Yes      |

## Usage

### Node.js Project

```yaml
- name: Update package.json version
  uses: draftm0de/github.workflows/.github/actions/version-patch@main
  with:
    ci-tag-source: 'nodejs'
    new-version: '1.2.4'
```

This updates the `version` field in `package.json` to `1.2.4`, commits with message `chore: bump version to 1.2.4`, and pushes to the current branch.

### Flutter Project

```yaml
- name: Update pubspec.yaml version
  uses: draftm0de/github.workflows/.github/actions/version-patch@main
  with:
    ci-tag-source: 'flutter'
    new-version: '1.2.4'
```

This updates the `version` field in `pubspec.yaml` to `1.2.4`, commits with message `chore: bump version to 1.2.4`, and pushes to the current branch.

### Branch-based Versioning (No File Update)

```yaml
- name: Skip version file update
  uses: draftm0de/github.workflows/.github/actions/version-patch@main
  with:
    ci-tag-source: 'branch'
    new-version: '1.2.4'
```

When `ci-tag-source: 'branch'`, the action exits early without updating any files. This is useful when version is tracked via git tags only.

## How It Works

1. **Validate inputs**: Ensures `ci-tag-source` is valid and required files exist
2. **Update version file**:
   - `nodejs`: Updates `package.json` using `jq`
   - `flutter`: Updates `pubspec.yaml` using `sed`
   - `branch`: Skips file update
3. **Commit and push**:
   - Configures git with `github-actions[bot]` identity
   - Stages the updated file
   - Commits with message: `chore: bump version to {version}`
   - Pushes to current branch (typically `main`)

## Requirements

### Node.js Projects
- `jq` must be installed on the runner (pre-installed on GitHub-hosted runners)
- `package.json` must exist in repository root

### Flutter Projects
- `sed` must be available (pre-installed on all runners)
- `pubspec.yaml` must exist in repository root

## Integration with Workflows

This action is designed to run **before** git tagging in CI workflows:

```yaml
jobs:
  git_tag_push:
    runs-on: ubuntu-latest
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Read current version
        id: version
        uses: draftm0de/github.workflows/.github/actions/version-reader@main
        with:
          type: nodejs

      - name: Calculate next version
        id: next
        uses: draftm0de/github.workflows/.github/actions/tag-builder@main
        with:
          target-branch: main
          current-version: ${{ steps.version.outputs.version }}
          patch: 'true'

      - name: Update package.json with new version
        uses: draftm0de/github.workflows/.github/actions/version-patch@main
        with:
          ci-tag-source: 'nodejs'
          new-version: ${{ steps.next.outputs.next-version }}

      - name: Create git tags
        id: tagger
        uses: draftm0de/github.workflows/.github/actions/git-tag-builder@main
        with:
          version: ${{ steps.next.outputs.next-version }}
          target-branch: main
          git-tag-levels: 'patch,minor,major'

      - name: Push tags
        uses: draftm0de/github.workflows/.github/actions/git-push@main
        with:
          tags: ${{ steps.tagger.outputs.git-tags }}
```

## Commit Message Format

The commit message follows Conventional Commits format:

```
chore: bump version to 1.2.4
```

This format is compatible with automated changelog tools and clearly indicates a version bump.

## GitHub Configuration

This action requires specific permissions and settings to commit and push:

### Required Workflow Permissions

```yaml
permissions:
  contents: write  # Required to commit and push version updates
```

### Required Repository Settings

The repository must have "Allow GitHub Actions to create and approve pull requests" enabled:

1. Navigate to repository Settings → Actions → General
2. Scroll to "Workflow permissions"
3. Enable: ☑ Allow GitHub Actions to create and approve pull requests

See [GitHub Configuration](../../../README.md#github) in the main README for detailed setup instructions.

## Notes

- Version prefix (`v`) is automatically stripped before writing to files
- Action will skip commit if no changes are detected (version already up to date)
- Git push uses the current branch HEAD (typically `main`)
- For `ci-tag-source: 'branch'`, no files are modified and no commits are made
- See [DEPLOYMENT.md](DEPLOYMENT.md) for implementation details
