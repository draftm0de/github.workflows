# Git Tag Builder Action

Creates git tags based on version and target branch. Supports exact version tags and optional branch-level tags for version-like branches.

## Inputs

| Name | Description | Required | Default |
|------|-------------|----------|---------|
| `version` | Version to tag (e.g., `v1.2.12`, `1.2.12`) | Yes | - |
| `target-branch` | Target branch name (e.g., `main`, `v1.2`, `develop`) | Yes | - |
| `enable-branch-tag` | Enable branch-level tagging when branch is version-like | No | `'true'` |

## Outputs

| Name | Description |
|------|-------------|
| `git-tags` | List of git tags created (space-separated) |
| `exact-tag` | Exact version tag created (e.g., `v1.2.12`) |
| `branch-tag` | Branch-level tag created (e.g., `v1.2`), empty if not created |

## Usage

### Basic Usage

```yaml
- name: Create git tags
  id: tagger
  uses: draftm0de/github.workflows/.github/actions/git-tag-builder@main
  with:
    version: v1.2.12
    target-branch: v1.2

- name: Push tags
  run: |
    git push origin ${{ steps.tagger.outputs.git-tags }}
```

### With Tag Builder

```yaml
- name: Build next version
  id: tag_builder
  uses: draftm0de/github.workflows/.github/actions/tag-builder@main
  with:
    target-branch: v1.2
    current-version: v1.2.11
    patch: 'true'

- name: Create git tags
  id: tagger
  uses: draftm0de/github.workflows/.github/actions/git-tag-builder@main
  with:
    version: ${{ steps.tag_builder.outputs.next-version-short }}
    target-branch: v1.2

- name: Push tags
  run: |
    git push origin --tags
```

### Disable Branch Tagging

```yaml
- name: Create git tags
  uses: draftm0de/github.workflows/.github/actions/git-tag-builder@main
  with:
    version: v1.2.12
    target-branch: main
    enable-branch-tag: 'false'
```

## How It Works

**Exact Tag:**
- Always creates a tag with the exact version (postfix stripped)
- Example: Input `v1.2.12+build` → Creates tag `v1.2.12`

**Branch Tag (when `enable-branch-tag: true`):**
- Detects if branch name is version-like (`v1.2`, `1.2`, `v1`)
- Creates/updates branch-level tag if version matches
- Example: Branch `v1.2`, version `v1.2.12` → Creates tags `v1.2.12` and `v1.2`

**Branch Tag Matching:**
- Branch `v1.2` + version `v1.2.12` → Creates `v1.2` tag ✅
- Branch `v1.2` + version `v2.0.0` → No branch tag ❌
- Branch `main` + version `v1.2.12` → No branch tag (not version-like) ❌

**Updating Branch Tags:**
- Branch tags are updated (deleted and recreated) if they already exist
- Exact tags must not exist (will error if duplicate)

## Example Scenarios

| Branch | Version | Exact Tag | Branch Tag | Notes |
|--------|---------|-----------|------------|-------|
| `v1.2` | `v1.2.12` | `v1.2.12` | `v1.2` | Both tags created |
| `v1.2` | `v1.2.13` | `v1.2.13` | `v1.2` | Branch tag updated |
| `v1.2` | `v2.0.0` | `v2.0.0` | - | Version doesn't match branch |
| `main` | `v1.2.12` | `v1.2.12` | - | Branch not version-like |
| `v1` | `v1.5.0` | `v1.5.0` | `v1` | Major-only branch |

## Notes

- Postfixes (e.g., `+build-123`) are stripped before tagging
- Branch tags are "floating" - they move to the latest patch for that branch
- Exact tags are immutable - action will error if tag already exists
- Use with `git push origin --tags` or push specific tags from outputs
- See [DEPLOYMENT.md](DEPLOYMENT.md) for implementation details
