# Git PR Exists Action

Composite action that checks if the current branch already has an open pull request in the same repository. When the action finds an open PR it outputs `exists: 'true'` (plus the number of matches) so workflows can skip redundant jobs such as running the full test suite twice. If no GitHub token is provided, the action exits early and defaults to `exists: 'false'` / `count: 0`.

## Inputs
| Name | Description | Required | Default |
| --- | --- | --- | --- |
| `github-token` | Token with `repo` scope (usually `${{ secrets.GITHUB_TOKEN }}`) that allows calling the pull requests API. When omitted the action short-circuits and reports `exists=false`. | No | `''` |

## Outputs
| Name | Description |
| --- | --- |
| `exists` | `'true'` when at least one open pull request targets the current branch, otherwise `'false'`. |
| `count` | Number of open pull requests discovered for the branch. |
| `ci-open-pull-requests` | Backwards-compatible alias for `exists` used by the reusable Node.js workflow. |

## Usage
```yaml
steps:
  - name: Check for existing pull requests
    id: git_pr_exists
    uses: ./.github/actions/git-state
    with:
      github-token: ${{ secrets.GITHUB_TOKEN }}

  - name: Run tests
    if: ${{ steps.git_pr_exists.outputs.exists == 'false' }}
    run: npm test
```

The action relies on the GitHub CLI (`gh`) that is preinstalled on the hosted runners. Custom runners must also provide the CLI (and make sure it can read `GITHUB_TOKEN`).
