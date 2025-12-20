# Flutter CI Workflow

Reusable workflow that orchestrates a complete Flutter CI/CD pipeline including testing and semantic version validation with git tagging. The workflow intelligently avoids duplicate test runs when both push and pull_request events trigger for the same branch, making it efficient for teams using PR-based development workflows.

**Note**: Publishing to pub.dev is not yet implemented but will be added soon.

## Triggers

This is a `workflow_call` reusable workflow that must be invoked from event-driven workflows in consuming repositories.

```yaml
on:
  workflow_call:
```

Consuming repositories should create a workflow file (e.g., `.github/workflows/ci.yml`) that triggers on `push` and/or `pull_request` events and calls this reusable workflow.

## Inputs

### Testing

| Name                | Type    | Required | Default | Description |
|---------------------|---------|----------|---------|-------------|
| `pub-cache-key`     | string  | No       | `''`    | Optional cache key passed to the flutter-test action for pub dependency caching. |

### Release Detection

| Name                          | Type    | Required | Default | Description |
|-------------------------------|---------|----------|---------|-------------|
| `ci-release-branch-patterns`  | string  | No       | `'main'`| Comma-separated branch patterns to identify release branches (e.g., `'main,release/*'`). Used to determine if a PR targets a release branch or a push is to a release branch. |

### Tagging

| Name                     | Type    | Required | Default | Description |
|--------------------------|---------|----------|---------|-------------|
| `ci-tag-source`          | string  | No       | `''`    | Version source type: `nodejs` (package.json), `flutter` (pubspec.yaml), or `branch` (git tags). Leave blank to skip tagging jobs. |
| `ci-tag-increment-patch` | boolean | No       | `false` | Auto-increment patch version when major.minor match latest tag. |
| `git-tag-levels`         | string  | No       | `''` | Git tag levels to create: `patch`, `minor`, `major` (comma-separated). Filters out levels covered by version-like branches. `'latest'` not allowed. Empty by default (no multi-level tags). |


### Input Notes

**Testing:**
- **Pub Cache**: Optional cache key for Flutter pub dependencies. When empty, the test-flutter action uses default caching behavior.

**Release Detection:**
- `ci-release-branch-patterns` defines which branches are considered "release" branches (defaults to `'main'`).
- Supports glob patterns like `'release/*'` or `'hotfix/*'`.
- Used by the `git-state` action to set `is-release-branch` output.

**Tagging:**
- Tagging jobs only run when `ci-tag-source` is provided.
- Use `nodejs` to read version from package.json, `flutter` for pubspec.yaml, or `branch` to read from git tags.
- `ci-tag-increment-patch` automatically increments patch version when major.minor match the latest tag.
- `git-tag-levels` controls which git tags are created (default: `'patch'` for exact version only).
- Multi-level tags (`minor`, `major`) are automatically filtered if covered by version-like branches (e.g., `v1.2` branch skips `v1.2` multi-level tag).


## Permissions

The reusable workflow declares these permissions as its requirements:
```yaml
permissions:
  contents: write      # Required for git_push job
  pull-requests: read  # Required for git-state action
```

**Important:** When calling this workflow with `workflow_call`, the **caller's permissions override** these defaults. You must explicitly grant permissions in the calling workflow:

```yaml
# .github/workflows/ci.yml (caller)
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  contents: write      # Required for git tags
  pull-requests: read  # Required for PR detection

jobs:
  flutter:
    uses: draftm0de/github.workflows/.github/workflows/flutter-ci.yml@main
    secrets: inherit
    with:
      pub-cache-key: 'flutter-deps'
      ci-tag-source: 'flutter'
```

**Permission requirements by feature:**
- **All workflows**: `pull-requests: read` (for PR detection in git-state)
- **Git tagging** (`ci-tag-source` set): `contents: write`

**Note:** If you only use testing features (no tagging), you can use:
```yaml
permissions:
  pull-requests: read
```

## Jobs

### 1. setup

**Purpose**: Detects Git state and branch information.

**Runs on**: `ubuntu-latest`

**Steps**:
1. Calls `git-state` action to detect:
   - Whether an open PR exists for current branch (`has-open-pull-requests`)
   - Source branch name (`source-branch-name`)
   - Target branch name (`target-branch-name`)
   - Whether branch matches release patterns (`is-release-branch`)

**Outputs**:
- `has-open-pull-requests`: `'true'` when open PR exists for the branch
- `source-branch-name`: Source/current branch name
- `target-branch-name`: Target/base branch name (empty for push events)
- `is-release-branch`: `'true'` when target/current branch matches `ci-release-branch-patterns`

**Note**: Use `github.event_name` to check the event type (`'pull_request'`, `'push'`, etc.).

### 2. tests

**Purpose**: Runs Flutter testing, linting, formatting, and analyzer checks.

**Runs on**: `ubuntu-latest`

**Depends on**: `setup`

**Runs when**:
```yaml
github.event_name == 'pull_request' OR
github.event_name == 'merge_group' OR
(github.event_name == 'push' AND has-open-pull-requests == 'false')
```

This means:
- Always runs on `pull_request` events
- Always runs on `merge_group` events (GitHub merge queue)
- Only runs on `push` events when there's NO open PR for that branch
- Avoids duplicate test runs for the same branch

**Steps**:
1. Checkout repository
2. Calls `test-flutter` action with pub-cache-key input

**Uses action**: `draftm0de/github.workflows/.github/actions/test-flutter@main`

### 3. auto_tagging

**Purpose**: Calculates next version and creates git tag lists.

**Runs on**: `ubuntu-latest`

**Depends on**: `setup`

**Runs when**:
```yaml
ci-tag-source != '' AND (
  github.event_name == 'pull_request' OR
  github.event_name == 'merge_group' OR
  (github.event_name == 'push' AND has-open-pull-requests == 'false')
)
```

This means:
- Only runs when `ci-tag-source` is configured
- Runs on PR events, merge queue events, or push events without open PRs
- Runs in parallel with tests (doesn't depend on tests)

**Steps**:
1. Checkout repository with full history (`fetch-depth: 0`)
2. Read current version using `version-reader` action
3. Build next version using `tag-builder` action
4. Create git tags (if `git-tag-levels` provided) using `git-tag-builder` action

**Outputs**:
- `next-version`: Next version with postfix (e.g., `v1.2.12+build`)
- `next-version-short`: Next version without postfix (e.g., `v1.2.12`)
- `is-latest-version`: `'true'` if this is the latest version on the target branch
- `git-tags`: Space-separated git tags (e.g., `v1.2.12 v1.2`), empty if not created

**Uses actions**:
- `draftm0de/github.workflows/.github/actions/version-reader@main`
- `draftm0de/github.workflows/.github/actions/tag-builder@main`
- `draftm0de/github.workflows/.github/actions/git-tag-builder@main`

### 4. git_push

**Purpose**: Creates and pushes git tags to remote repository (release branches only).

**Runs on**: `ubuntu-latest`

**Depends on**: `setup`, `auto_tagging`, `tests`

**Runs when**:
```yaml
github.event_name == 'push' AND
is-release-branch == 'true' AND
git-tags != ''
```

This means:
- Only runs on `push` events to release branches (e.g., `main`, `release/*`)
- Only when git tags were calculated by auto_tagging job
- Only after tests pass

**Steps**:
1. Checkout repository with full history (`fetch-depth: 0`)
2. Push git tags to remote using `git-push` action

**Uses actions**:
- `draftm0de/github.workflows/.github/actions/git-push@main`

## Behavior

### Pull Request Guard

The workflow uses the `git-state` action to implement intelligent test execution:

**For pull_request events:**
- `github.event_name == 'pull_request'`
- Tests always run
- Version validation runs (if `ci-tag-source` configured)

**For push events:**
- `github.event_name == 'push'`
- `git-state` queries GitHub API to check for open PRs on the branch
- If `has-open-pull-requests == 'true'`: tests are skipped (PR workflow already running)
- If `has-open-pull-requests == 'false'`: tests run (no PR exists, or direct push to protected branch)

This prevents duplicate test runs when both events trigger for the same commit.

**Note**: When using `workflow_call`, `github.event_name` reflects the calling workflow's trigger event, not `'workflow_call'` itself.

### CI Tag Source Modes

**`ci-tag-source: nodejs`:**
- Reads version from `package.json` in repository root
- Validates version matches semantic version pattern
- Fails if package.json missing or version invalid
- Use for: Node.js projects with version in package.json

**`ci-tag-source: flutter`:**
- Reads version from `pubspec.yaml` in repository root
- Validates version matches semantic version pattern
- Fails if pubspec.yaml missing or version invalid
- Use for: Flutter projects with version in pubspec.yaml

**`ci-tag-source: branch`:**
- Reads latest semantic version tag from target branch
- Discovers tags using git history merged into target branch
- Fails if no valid semantic version tags found
- Use for: Branch-based versioning with git tags

### Patch Auto-Increment

When `ci-tag-increment-patch: true`:

1. Compares current version with latest tag from target branch
2. If major.minor are the same: increments patch (e.g., `v1.2.3` → `v1.2.4`)
3. If major or minor increased: uses version as-is with patch `0`
4. Prevents version drift and ensures monotonic versioning

When `ci-tag-increment-patch: false`:
- Uses exact version from source
- Fails if version already exists or is older than latest tag

## Example Usage

### Basic Flutter CI

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  flutter:
    uses: draftm0de/github.workflows/.github/workflows/flutter-ci.yml@main
    with:
      pub-cache-key: 'flutter-deps'
      ci-release-branch-patterns: 'main,develop'
    secrets: inherit
```

### With Git Tagging

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop, release/*]
  pull_request:
    branches: [main]

jobs:
  flutter:
    uses: draftm0de/github.workflows/.github/workflows/flutter-ci.yml@main
    with:
      pub-cache-key: 'flutter-deps'
      ci-release-branch-patterns: 'main,release/*'
      ci-tag-source: 'flutter'
      ci-tag-increment-patch: true
      git-tag-levels: 'patch,minor,major'
    secrets: inherit
```

### Version from Git Tags

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, release/*]
  pull_request:
    branches: [main]

jobs:
  flutter:
    uses: draftm0de/github.workflows/.github/workflows/flutter-ci.yml@main
    with:
      ci-release-branch-patterns: 'main,release/*,hotfix/*'
      ci-tag-source: 'branch'
      ci-tag-increment-patch: false
    secrets: inherit
```

## Dependencies

### Required on Runners

**GitHub-hosted runners** (ubuntu-latest) include everything needed:
- Flutter SDK (setup-flutter action handles installation)
- `gh` CLI
- `git`

**Self-hosted runners** must provide:
- Flutter SDK (or specify installation in consuming workflow)
- `gh` CLI for git-state action
- `git` with tag support
- `GITHUB_TOKEN` or `GH_TOKEN` environment variable

### Required Actions

This workflow uses the following actions from the same repository:
- `git-state` - Detects PR state and branch information
- `test-flutter` - Runs Flutter tests, analyzer, formatting checks
- `version-reader` - Reads versions from pubspec.yaml or git tags
- `tag-builder` - Builds and validates semantic version tags
- `git-tag-builder` - Creates git tags
- `git-push` - Pushes git tags to remote repository

## Workflow State

**Current Status**: Fully implemented

**Implemented jobs**:
- ✅ `setup` - Git state detection
- ✅ `tests` - Flutter test execution with PR guard and merge queue support
- ✅ `auto_tagging` - Version calculation and git tag generation
- ✅ `git_push` - Git tag creation and push to remote (release branches only)
- 🚧 `pub_publish` - Publishing to pub.dev (coming soon)

## Notes

- `pub-cache-key` is optional for pub dependency caching
- Flutter tests include analyzer, formatting, and test suite checks
- Tagging jobs are opt-in via `ci-tag-source` input
- Git tag calculation is conditional on `git-tag-levels` input
- Git tags are pushed only when `git-tags` output is non-empty
- The PR guard prevents wasteful duplicate runs on the same branch
- Supports GitHub merge queue (`merge_group` events)
- Full git history (`fetch-depth: 0`) is required for version comparison in tagging jobs
- The workflow uses composite actions, making it easy to test and maintain individual components
- Git tags are pushed only on `push` events to release branches (configured via `ci-release-branch-patterns`)
- Publishing to pub.dev is planned but not yet implemented
