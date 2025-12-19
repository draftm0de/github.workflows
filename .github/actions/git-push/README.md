# Git Push Action

Creates and pushes git tags to the remote repository.

## Inputs

| Name   | Description                                        | Required |
|--------|----------------------------------------------------|----------|
| `tags` | Space-separated list of tags to create and push    | Yes      |

## Usage

### Basic Example

```yaml
- name: Push git tags
  uses: draftm0de/github.workflows/.github/actions/git-push@main
  with:
    tags: v1.2.12 v1.2 v1
```

### With Git Tag Builder

```yaml
- name: Calculate git tags
  id: git-tags
  uses: draftm0de/github.workflows/.github/actions/git-tag-builder@main
  with:
    version: v1.2.12
    target-branch: main
    git-tag-levels: patch,minor,major

- name: Push git tags
  if: steps.git-tags.outputs.git-tags != ''
  uses: draftm0de/github.workflows/.github/actions/git-push@main
  with:
    tags: ${{ steps.git-tags.outputs.git-tags }}
```

## How It Works

1. Configures git with bot user credentials
2. Loops through space-separated tags
3. Creates each tag with `git tag -f` (force flag allows updating existing tags)
4. Pushes each tag to origin with `git push -f`
5. Writes summary to GitHub step output

## Permissions

Requires `contents: write` permission in the workflow:

```yaml
permissions:
  contents: write
```

## Notes

- Uses force flags (`-f`) to allow updating existing tags (for floating tags like `v1.2`)
- Skips operation if tags input is empty
- Automatically uses `github-actions[bot]` as committer
- Repository must be checked out with `actions/checkout@v4` before calling this action
- Requires `fetch-depth: 0` for proper git history when working with existing tags
