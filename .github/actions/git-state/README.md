# Git State Action

Composite action that detects the Git state of the current workflow run, including whether it's executing in a pull request context and whether the current branch has any open pull requests. This action helps workflows make intelligent decisions about when to run expensive operations like full test suites, avoiding duplicate work when both push and pull_request events trigger for the same branch.

## Inputs

| Name | Description | Required | Default |
|------|-------------|----------|----------|
| `ci-release-branch-patterns` | Comma-separated branch patterns to identify release branches (e.g., `'main,release/*'`) | No | `'main'` |

The action also reads environment variables provided by GitHub Actions:
- `GITHUB_REF` - The branch or tag ref that triggered the workflow
- `GITHUB_HEAD_REF` - The head ref or source branch of the pull request (only set for PR events)
- `GITHUB_BASE_REF` - The base ref or target branch of the pull request (only set for PR events)
- `GITHUB_REF_NAME` - The short ref name (branch name for push events)
- `GITHUB_REPOSITORY` - The owner and repository name (used to query GitHub API)
- `github.token` - Automatically provided GitHub token for API authentication

For most use cases, the default `GITHUB_TOKEN` is sufficient. If you need elevated permissions (e.g., accessing private repositories or different organizations), provide a PAT (Personal Access Token) by setting `GH_TOKEN` in the workflow environment.

## Outputs

| Name                      | Description |
|---------------------------|-------------|
| `is-release-branch`       | `'true'` when the target branch (PR) or current branch (push) matches `ci-release-branch-patterns`, `'false'` otherwise. |
| `has-open-pull-requests`  | `'true'` when an open PR exists for the current branch during push events, `'false'` when no PR exists. Always `'true'` for PR events. |
| `source-branch-name`      | The source branch name. For PR events, this is `GITHUB_HEAD_REF`. For push events, this is `GITHUB_REF_NAME`. |
| `target-branch-name`      | The target/base branch name. For PR events, this is `GITHUB_BASE_REF`. For push events, this is empty. |
| `ci-branch-name`          | The branch name for CI operations. For PR events, this is the target branch (`GITHUB_BASE_REF`). For push events, this is the source branch (`GITHUB_REF_NAME`). |

## Usage

### Basic Usage

```yaml
steps:
  - name: Detect Git state
    id: git_state
    uses: draftm0de/github.workflows/.github/actions/git-state@main
    with:
      ci-release-branch-patterns: 'main,release/*'

  - name: Run tests only when necessary
    if: ${{ github.event_name == 'pull_request' ||
            (github.event_name == 'push' && steps.git_state.outputs['has-open-pull-requests'] == 'false') }}
    run: npm test
```

### Detecting Release Branches

```yaml
steps:
  - name: Detect Git state
    id: git_state
    uses: draftm0de/github.workflows/.github/actions/git-state@main
    with:
      ci-release-branch-patterns: 'main,release/*,hotfix/*'

  - name: Display branch information
    run: |
      echo "Event: ${{ github.event_name }}"
      echo "Source branch: ${{ steps.git_state.outputs.source-branch-name }}"
      echo "Target branch: ${{ steps.git_state.outputs.target-branch-name }}"
      echo "Is release branch: ${{ steps.git_state.outputs.is-release-branch }}"

  - name: Deploy to production
    if: ${{ github.event_name == 'push' &&
            steps.git_state.outputs.is-release-branch == 'true' }}
    run: ./deploy-production.sh
```

### Avoiding Duplicate Test Runs

```yaml
name: CI

on:
  push:
    branches: [ main, develop, feature/* ]
  pull_request:
    branches: [ main, develop ]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Check Git state
        id: git_state
        uses: draftm0de/github.workflows/.github/actions/git-state@main
        with:
          ci-release-branch-patterns: 'main,develop'

      # Only run tests if:
      # 1. This is a PR event, OR
      # 2. This is a push event AND there's no open PR for this branch
      - name: Run test suite
        if: ${{ github.event_name == 'pull_request' ||
                (github.event_name == 'push' && steps.git_state.outputs['has-open-pull-requests'] == 'false') }}
        run: npm test
```

## Behavior

### Pull Request Events

When the workflow is triggered by a `pull_request` event:

1. `GITHUB_HEAD_REF` is populated by GitHub Actions (contains the source branch name)
2. `GITHUB_BASE_REF` is populated by GitHub Actions (contains the target branch name)
3. The action checks if `GITHUB_BASE_REF` matches any pattern in `ci-release-branch-patterns`
4. The action sets:
   - `is-release-branch='true'` (if target matches patterns) or `'false'`
   - `has-open-pull-requests='true'` (always true for PR events)
   - `source-branch-name=$GITHUB_HEAD_REF`
   - `target-branch-name=$GITHUB_BASE_REF`
   - `ci-branch-name=$GITHUB_BASE_REF`
5. The action exits immediately (no API calls needed)

### Push Events

When the workflow is triggered by a `push` event:

1. `GITHUB_HEAD_REF` is empty (not a PR context)
2. The action extracts the branch name from `GITHUB_REF_NAME`
3. The action checks if current branch matches any pattern in `ci-release-branch-patterns`
4. The action sets:
   - `is-release-branch='true'` (if branch matches patterns) or `'false'`
   - `source-branch-name=$GITHUB_REF_NAME`
   - `target-branch-name=''` (empty)
   - `ci-branch-name=$GITHUB_REF_NAME`
5. The action queries the GitHub API to count open PRs for the current branch:
   - Uses GraphQL API via `gh api graphql`
   - Queries `repository.pullRequests(states:OPEN, headRefName:$branch).totalCount`
   - If API call fails or returns invalid data: emits a warning and defaults to count `0`
   - If count is `0`: sets `has-open-pull-requests='false'`
   - If count is `> 0`: sets `has-open-pull-requests='true'`

### Pattern Matching

The action supports glob-style patterns for branch matching:

- **Exact match**: `'main'` matches only `main`
- **Wildcard**: `'release/*'` matches `release/1.0`, `release/2.0`, etc.
- **Multiple patterns**: `'main,release/*,hotfix/*'` matches any of the patterns

Patterns are evaluated using bash glob matching (`[[ branch == pattern ]]`).

### GitHub API Query

The action uses the GitHub CLI (`gh`) to query the GraphQL API:

```graphql
query($owner:String!, $name:String!, $head:String!) {
  repository(owner:$owner, name:$name) {
    pullRequests(states:OPEN, headRefName:$head) {
      totalCount
    }
  }
}
```

Where:
- `$owner` - Repository owner (extracted from `GITHUB_REPOSITORY`)
- `$name` - Repository name (extracted from `GITHUB_REPOSITORY`)
- `$head` - Source branch name (from `GITHUB_REF_NAME`)

This query counts open pull requests where the source branch matches the current branch.

### Error Handling

The action includes robust error handling for API failures:

1. **API Call Failures**: If the `gh api` command fails (network issues, authentication problems, rate limits), the action:
   - Captures the exit status
   - Emits a GitHub Actions warning message
   - Defaults to `COUNT=0` (assumes no open PRs exist)
   - Continues execution without failing the workflow

2. **Invalid Responses**: If the API returns empty or non-numeric data, the action:
   - Validates the response using regex pattern matching
   - Treats invalid responses as failures
   - Defaults to `COUNT=0` for safe fallback behavior

This ensures the action never fails the entire workflow due to temporary API issues, while still providing visibility through warning messages in the Actions log.

### Job Summary Output

Each run appends a checklist to the GitHub Actions job summary (`GITHUB_STEP_SUMMARY`):

**For PR events (targeting release branch):**
```
- [x] is-release-branch: true
- [x] has-open-pull-requests (current event, is pull_request)
- [x] source-branch-name: feature/new-feature
- [x] target-branch-name: main
```

**For push events to feature branch (with open PR):**
```
- [x] is-release-branch: false
- [x] has-open-pull-requests (PR for the same branch detected)
- [x] source-branch-name: feature/new-feature
- [x] target-branch-name: -
```

**For push events to release branch (without open PR):**
```
- [x] is-release-branch: true
- [ ] has-open-pull-requests (no PR for the same branch detected)
- [x] source-branch-name: main
- [x] target-branch-name: -
```

## Dependencies

This action requires:
- `gh` (GitHub CLI) - for GraphQL API queries during push events
- `bash` - standard shell with regex support
- GitHub Actions environment variables (`GITHUB_REF`, `GITHUB_REPOSITORY`, etc.)
- `github.token` - for API authentication (automatically provided by GitHub Actions)

Hosted GitHub runners come pre-installed with `gh` and have a scoped `GITHUB_TOKEN` available, so no additional setup is required. For self-hosted runners:
- Install `gh` CLI ([installation guide](https://cli.github.com/))
- Ensure `GITHUB_TOKEN` or `GH_TOKEN` is available in the environment
- The token must have `repo` scope to query pull request information

## Use Cases

1. **Avoid duplicate test runs**: When both `push` and `pull_request` events are configured, the same commit can trigger tests twice. Use this action to skip tests on push events when a PR already exists.

2. **Conditional deployment**: Deploy to staging only when PRs target specific branches (e.g., `main` or `develop`).

3. **Branch-specific workflows**: Execute different steps based on source or target branch names.

4. **PR-only operations**: Run expensive operations (e.g., security scans, performance tests) only during PR events, not on every push.

5. **Release branch detection**: Execute deployment or tagging jobs only when pushing to release branches (e.g., `main`, `release/*`).

6. **Event-based logic**: Use `github.event_name` combined with `is-release-branch` to differentiate between PR events, pushes to release branches, and pushes to feature branches.

7. **Workflow optimization**: Make intelligent decisions about which jobs to run based on the Git state, reducing CI costs and execution time.
