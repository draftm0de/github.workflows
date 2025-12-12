# Git PR Exists Action

Composite action that checks if the current branch already has an open pull request in the same repository. When the action finds an open PR it surfaces `ci-open-pull-requests: 'true'` so workflows can skip redundant jobs such as running the full test suite twice. If no GitHub token is provided, the action exits early and defaults to `ci-open-pull-requests: 'false'` and reports the decision in the workflow summary.

## Inputs
| Name | Description | Required | Default |
| --- | --- | --- | --- |
| `github-token` | Token with `repo` scope (usually `${{ secrets.GITHUB_TOKEN }}`) that allows calling the pull requests API. When omitted the action short-circuits and reports `ci-open-pull-requests=false`. | No | `''` |

## Outputs
| Name | Description |
| --- | --- |
| `ci-open-pull-requests` | `'true'` when at least one open pull request targets the current branch, otherwise `'false'`. |

## Usage
```yaml
steps:
  - name: Check for existing pull requests
    id: git_pr_exists
    uses: draftm0de/github.workflows/.github/actions/git-state@main
    with:
      github-token: ${{ secrets.GITHUB_TOKEN }}

  - name: Run tests
    if: ${{ steps.git_pr_exists.outputs['ci-open-pull-requests'] == 'false' }}
    run: npm test
```

Each run appends a checklist line (for example `- [x] ci-open-pull-requests=false (No open pull requests)`) to the job summary so it is easy to confirm why later jobs were skipped. The action relies on the GitHub CLI (`gh`) that is preinstalled on the hosted runners. Custom runners must also provide the CLI (and make sure it can read `GITHUB_TOKEN`).
