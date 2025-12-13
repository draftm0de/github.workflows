# Git PR Exists Action

Composite action that reports whether the current branch is already tied to an open pull request. It also emits a simple `is-pull-request` boolean so downstream workflows can detect when they are executing in the context of a PR vs. a push. The reusable Node.js workflow relies on this action to avoid running the full test suite twice for the same branch.

## Inputs
The action has no explicit inputs. It reads `GITHUB_REF` / `GITHUB_REPOSITORY` and uses `${{ github.token }}` automatically to authenticate GitHub CLI (`gh`) requests. Provide a PAT via the calling workflow only if `${{ secrets.GITHUB_TOKEN }}` is insufficient.

## Outputs
| Name                    | Description |
|-------------------------|-------------|
| `is-pull-request`       | `'true'` when the workflow already runs on a `pull_request` event (detected via `GITHUB_REF`). |
| `ci-open-pull-requests` | `'true'` when an open PR exists for the current branch during push events, otherwise `'false'`. |
| `branch`                | Branch name for push events, empty on PR refs. Useful for logging or downstream API calls. |

## Usage
```yaml
steps:
  - name: Detect PR state
    id: git_state
    uses: draftm0de/github.workflows/.github/actions/git-state@main

  - name: Run tests
    if: ${{ steps.git_state.outputs.is-pull-request == 'true' ||
            steps.git_state.outputs['ci-open-pull-requests'] == 'false' }}
    run: npm test
```

Hosted GitHub runners already ship with `gh` and have a scoped `GITHUB_TOKEN`, so no extra configuration is required. Self-hosted runners must install `gh`, `jq`, and expose `GITHUB_TOKEN` / `GH_TOKEN`. Each run appends checklist entries (`- [x] is-pull-request`, `- [ ] ci-open-pull-requests (no PR for the same branch detected)`, etc.) to the job summary so it is easy to confirm why later jobs ran or were skipped.
