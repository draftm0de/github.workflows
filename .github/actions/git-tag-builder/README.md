# Git Tag Builder Action

Creates git tags based on version and target branch. Supports exact version tags and optional branch-level tags for version-like branches.

## Inputs

| Name                | Description                                                                                                                                       | Required | Default   |
|---------------------|---------------------------------------------------------------------------------------------------------------------------------------------------|----------|-----------|
| `version`           | Version to tag (e.g., `v1.2.12`, `1.2.12`)                                                                                                        | Yes      | -         |
| `target-branch`     | Target branch name (e.g., `main`, `v1.2`, `develop`)                                                                                              | Yes      | -         |
| `enable-branch-tag` | Enable branch-level tagging when branch is version-like                                                                                           | No       | `'true'`  |
| `git-tag-levels`    | Comma-separated tag levels: `patch`, `minor`, `major` (e.g., `'patch,minor,major'`). Filters out levels already covered by version-like branches. | No       | `''` (none) |

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
    target-branch: main
    git-tag-levels: 'patch,minor,major'

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

### Only Patch Tag (Default)

```yaml
- name: Create git tags
  uses: draftm0de/github.workflows/.github/actions/git-tag-builder@main
  with:
    version: v1.2.12
    target-branch: main
    git-tag-levels: 'patch'
```

### Multi-Level Tags with Version Branch

```yaml
- name: Create git tags
  uses: draftm0de/github.workflows/.github/actions/git-tag-builder@main
  with:
    version: v1.2.12
    target-branch: v1.2
    git-tag-levels: 'patch,minor,major'
```
Result: Creates `v1.2.12` and `v1.2` (branch tag). Major tag `v1` is skipped because `v1.2` branch already covers minor level.

## How It Works

**Exact Tag (Patch):**
- Always creates a tag with the exact version (postfix stripped)
- Example: Input `v1.2.12+build` → Creates tag `v1.2.12`

**Multi-Level Tags (via `git-tag-levels`):**
- `patch`: Exact version tag (e.g., `v1.2.12`) - always included
- `minor`: Minor-level tag (e.g., `v1.2`)
- `major`: Major-level tag (e.g., `v1`)

**Branch Tag (when `enable-branch-tag: true`):**
- Detects if branch name is version-like (`v1.2`, `1.2`, `v1`)
- Creates branch-level tag if version matches
- Automatically filters out redundant multi-level tags
- Example: Branch `v1.2`, version `v1.2.12`, levels `patch,minor,major`
  - Creates: `v1.2.12`, `v1.2` (branch), `v1`
  - Note: Minor tag from levels is skipped (covered by branch tag)

**Branch Tag Matching:**
- Branch `v1.2` + version `v1.2.12` → Creates `v1.2` tag ✅
- Branch `v1.2` + version `v2.0.0` → No branch tag ❌
- Branch `main` + version `v1.2.12` → No branch tag (not version-like) ❌

**Updating Branch Tags:**
- Branch tags are updated (deleted and recreated) if they already exist
- Exact tags must not exist (will error if duplicate)

## Example Scenarios

### With Default Settings (`git-tag-levels: 'patch'`)

| Branch | Version | Tags Created | Notes |
|--------|---------|--------------|-------|
| `main` | `v1.2.12` | `v1.2.12` | Only patch tag |
| `v1.2` | `v1.2.12` | `v1.2.12`, `v1.2` | Patch + branch tag |
| `v1` | `v1.5.0` | `v1.5.0`, `v1` | Patch + branch tag |

### With Multi-Level Tags (`git-tag-levels: 'patch,minor,major'`)

| Branch | Version | Tags Created | Notes |
|--------|---------|--------------|-------|
| `main` | `v1.2.12` | `v1.2.12`, `v1.2`, `v1` | All three levels |
| `v1.2` | `v1.2.12` | `v1.2.12`, `v1.2`, `v1` | Branch tag `v1.2` covers minor |
| `v1` | `v1.5.0` | `v1.5.0`, `v1.5`, `v1` | Branch tag `v1` covers major |
| `v1.2` | `v2.0.0` | `v2.0.0`, `v2.0`, `v2` | Version doesn't match branch |

## Notes

- Postfixes (e.g., `+build-123`) are stripped before tagging
- Branch tags are "floating" - they move to the latest version for that branch
- Multi-level tags (minor/major) are also "floating" - they move to the latest patch/minor
- Patch tags (exact versions) are immutable - action will error if tag already exists
- Version-like branches automatically filter out redundant multi-level tags
- Action only calculates tags, does not create them (no git operations)
- Use with `git tag` + `git push` or let your workflow handle tag creation
- Invalid tag levels will cause action to fail with exit code 1
- See [DEPLOYMENT.md](DEPLOYMENT.md) for implementation details
