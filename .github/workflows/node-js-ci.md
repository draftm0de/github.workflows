# Node.js CI Workflow

Reusable workflow that orchestrates a complete Node.js CI pipeline including testing, Docker builds, and semantic version validation. The workflow intelligently avoids duplicate test runs when both push and pull_request events trigger for the same branch, making it efficient for teams using PR-based development workflows.

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
| `node-version`      | string  | No       | `''`    | Node.js version to use (e.g., `'20'`, `'22'`). Required when `node-version-env` is not provided. |
| `node-version-env`  | string  | No       | `''`    | Path to env file exporting `NODE_VERSION` (e.g., `.env.build`). Overrides `node-version` when provided. |
| `enable-cache`      | boolean | No       | `true`  | Enable npm caching in `actions/setup-node`. Requires `package-lock.json` to be committed. |
| `lint-script`       | string  | No       | `''`    | npm script to run linting (e.g., `lint`). Leave blank to skip. |
| `prettier-script`   | string  | No       | `''`    | npm script to run Prettier in check mode (e.g., `format:check`). Leave blank to skip. |
| `badges-script`     | string  | No       | `''`    | npm script to refresh README/coverage badges. Leave blank to skip. |
| `test-script`       | string  | No       | `''`    | npm script to run test suite (e.g., `test`, `test:ci`). Leave blank to skip. |

### Tagging

| Name                  | Type    | Required | Default | Description |
|-----------------------|---------|----------|---------|-------------|
| `tag-source`          | string  | No       | `''`    | Version source type: `nodejs` (package.json), `flutter` (pubspec.yaml), or `target` (git tags). Leave blank to skip tagging jobs. |
| `tag-increment-patch` | boolean | No       | `false` | Auto-increment patch version when major.minor match latest tag. |

### Docker

| Name                       | Type    | Required | Default         | Description |
|----------------------------|---------|----------|-----------------|-------------|
| `docker-image-name`        | string  | No       | `''`            | Docker image name (e.g., `ghcr.io/org/app`). Leave blank to skip Docker builds. |
| `docker-build-context`     | string  | No       | `'.'`           | Docker build context path. |
| `docker-build-args-file`   | string  | No       | `''`            | Path to file with Docker build args. Leave blank to skip. |
| `docker-build-options`     | string  | No       | `'--no-cache'`  | Additional Docker build options. |
| `docker-build-reproducible`| boolean | No       | `true`          | Build reproducible Docker image with `SOURCE_DATE_EPOCH`. |
| `docker-tag-levels`        | string  | No       | `'patch,latest'`| Version levels to tag: `patch`, `minor`, `major`, `latest` (comma-separated). |

### Input Notes

**Testing:**
- **Node.js Version**: Provide either `node-version` OR `node-version-env`. The workflow will fail if both are empty.
- **Scripts**: All script inputs default to empty strings. When empty, the test action skips that step and logs it in the workflow summary.

**Tagging:**
- Tagging jobs only run when `tag-source` is provided.
- Use `nodejs` to read version from package.json, `flutter` for pubspec.yaml, or `target` to read from git tags.
- `tag-increment-patch` automatically increments patch version when major.minor match the latest tag.

**Docker:**
- Docker jobs only run when `docker-image-name` is provided.
- `docker-build-context` defaults to repository root (`.`).
- `docker-build-args-file` is optional - provide path to a file containing build args (one per line, format: `ARG=value`).
- `docker-build-options` defaults to `--no-cache` for clean builds.
- `docker-build-reproducible` uses `SOURCE_DATE_EPOCH` from git commit timestamp for reproducible builds.
- `docker-tag-levels` controls which version tags to create (defaults to `patch,latest`).

## Permissions

The workflow requires:
```yaml
permissions:
  contents: read
  pull-requests: read
```

Consuming repositories should use `secrets: inherit` to pass `GITHUB_TOKEN` and any registry credentials:

```yaml
jobs:
  node:
    uses: draftm0de/github.workflows/.github/workflows/node-js-ci.yml@main
    secrets: inherit
```

## Jobs

### 1. setup

**Purpose**: Detects Git state and branch information.

**Runs on**: `ubuntu-latest`

**Steps**:
1. Calls `git-state` action to detect:
   - Whether running in PR context (`is-pull-request`)
   - Whether an open PR exists for current branch (`ci-open-pull-request`)
   - Source branch name (`source-branch-name`)
   - Target branch name (`target-branch-name`)

**Outputs**:
- `is-pull-request`: `'true'` for PR events, `'false'` for push events
- `ci-open-pull-request`: `'true'` when open PR exists for the branch
- `source-branch-name`: Source/current branch name
- `target-branch-name`: Target/base branch name (empty for push events)

### 2. tests

**Purpose**: Runs linting, formatting, and test suite.

**Runs on**: `ubuntu-latest`

**Depends on**: `setup`

**Runs when**:
```yaml
is-pull-request == 'true' OR ci-open-pull-request == 'false'
```

This means:
- Always runs on PR events
- Only runs on push events when there's NO open PR for that branch
- Avoids duplicate test runs for the same branch

**Steps**:
1. Checkout repository
2. Calls `node-js-test` action with all test/lint/format script inputs

**Uses action**: `draftm0de/github.workflows/.github/actions/node-js-test@main`

### 3. build_on_pull

**Purpose**: Builds Docker image for pull requests.

**Runs on**: `ubuntu-latest`

**Depends on**: `setup`, `tests`

**Runs when**:
```yaml
is-pull-request == 'true' AND docker-image-name != ''
```

This means:
- Only runs for PR events
- Only when Docker image name is provided
- Only after tests pass

**Steps**:
1. Checkout repository
2. Build Docker image using `docker-build` action (no cache, reproducible)
3. Upload Docker image as artifact using `artifact-from-image` action

**Uses actions**:
- `draftm0de/github.workflows/.github/actions/docker-build@main`
- `draftm0de/github.workflows/.github/actions/artifact-from-image@main`

### 4. tag_on_pull

**Purpose**: Validates version and calculates next tag candidate for PRs.

**Runs on**: `ubuntu-latest`

**Depends on**: `setup`, `tests`

**Runs when**:
```yaml
is-pull-request == 'true' AND tag-source != ''
```

This means:
- Only runs for PR events
- Only when tagging mode is configured
- Only after tests pass

**Steps**:
1. Checkout repository with full history (`fetch-depth: 0`)
2. Read current version using `version-reader` action:
   - Type: `nodejs`, `flutter`, or `target` (from `tag-source` input)
   - Target branch: from `setup` job outputs
3. Build next version using `tag-builder` action:
   - Target branch: from `setup` job outputs
   - Current version: from version-reader step
   - Patch mode: from `tag-increment-patch` input

**Uses actions**:
- `draftm0de/github.workflows/.github/actions/version-reader@main`
- `draftm0de/github.workflows/.github/actions/tag-builder@main`

## Behavior

### Pull Request Guard

The workflow uses the `git-state` action to implement intelligent test execution:

**For pull_request events:**
- `is-pull-request` is `'true'`
- Tests always run
- Docker builds and version validation run (if configured)

**For push events:**
- `is-pull-request` is `'false'`
- `git-state` queries GitHub API to check for open PRs on the branch
- If `ci-open-pull-request == 'true'`: tests are skipped (PR workflow already running)
- If `ci-open-pull-request == 'false'`: tests run (no PR exists, or direct push to protected branch)

This prevents duplicate test runs when both events trigger for the same commit.

### Tag Source Modes

**`tag-source: nodejs`:**
- Reads version from `package.json` in repository root
- Validates version matches semantic version pattern
- Fails if package.json missing or version invalid
- Use for: Node.js projects with version in package.json

**`tag-source: flutter`:**
- Reads version from `pubspec.yaml` in repository root
- Validates version matches semantic version pattern
- Fails if pubspec.yaml missing or version invalid
- Use for: Flutter projects with version in pubspec.yaml

**`tag-source: target`:**
- Reads latest semantic version tag from target branch
- Discovers tags using git history merged into target branch
- Fails if no valid semantic version tags found
- Use for: Branch-based versioning with git tags

### Patch Auto-Increment

When `tag-increment-patch: true`:

1. Compares current version with latest tag from target branch
2. If major.minor are the same: increments patch (e.g., `v1.2.3` → `v1.2.4`)
3. If major or minor increased: uses version as-is with patch `0`
4. Prevents version drift and ensures monotonic versioning

When `tag-increment-patch: false`:
- Uses exact version from source
- Fails if version already exists or is older than latest tag

## Example Usage

### Basic Node.js CI

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  node:
    uses: draftm0de/github.workflows/.github/workflows/node-js-ci.yml@main
    with:
      node-version: '20'
      lint-script: 'lint'
      test-script: 'test:ci'
    secrets: inherit
```

### With Docker and Tagging

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop, release/*]
  pull_request:
    branches: [main]

jobs:
  node:
    uses: draftm0de/github.workflows/.github/workflows/node-js-ci.yml@main
    with:
      node-version: '20'
      lint-script: 'lint'
      prettier-script: 'format:check'
      test-script: 'test:ci'
      tag-source: 'nodejs'
      tag-increment-patch: true
      docker-image-name: ghcr.io/my-org/my-app
      docker-build-context: '.'
      docker-build-args-file: '.build.args'
      docker-tag-levels: 'patch,minor,major,latest'
    secrets: inherit
```

### Version from Branch Name

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, release/*]
  pull_request:
    branches: [main]

jobs:
  node:
    uses: draftm0de/github.workflows/.github/workflows/node-js-ci.yml@main
    with:
      node-version-env: '.env.build'
      lint-script: 'lint'
      test-script: 'test'
      tag-source: 'target'
      tag-increment-patch: false
    secrets: inherit
```

## Dependencies

### Required on Runners

**GitHub-hosted runners** (ubuntu-latest) include everything needed:
- Node.js (version specified via inputs)
- `gh` CLI
- `git`
- Docker

**Self-hosted runners** must provide:
- Node.js (or specify installation in consuming workflow)
- `gh` CLI for git-state action
- `git` with tag support
- Docker (if using Docker jobs)
- `GITHUB_TOKEN` or `GH_TOKEN` environment variable

### Required Actions

This workflow uses the following actions from the same repository:
- `git-state` - Detects PR state and branch information
- `node-js-test` - Runs Node.js tests and quality checks
- `docker-build` - Builds Docker images
- `artifact-from-image` - Uploads Docker images as artifacts
- `version-reader` - Reads versions from package.json or branch names
- `tag-builder` - Builds and validates semantic version tags

## Workflow State

**Current Status**: Partially implemented

**Implemented jobs**:
- ✅ `setup` - Git state detection
- ✅ `tests` - Test execution with PR guard
- ✅ `build_on_pull` - Docker builds for PRs
- ✅ `tag_on_pull` - Version validation for PRs

**Not yet implemented** (future work):
- ⏳ `build_on_push` - Docker builds and pushes for merge events
- ⏳ `tag_on_push` - Actual git tag creation on merge
- ⏳ Push/publish jobs for npm, Docker registry, etc.

The workflow currently handles the complete pull request validation pipeline. Push event jobs (building, tagging, publishing) will be added in future iterations.

## Notes

- The workflow enforces that either `node-version` or `node-version-env` is provided
- All test scripts are optional - skip by leaving blank
- Docker jobs are opt-in via `docker-image-name` input
- Tagging jobs are opt-in via `tag-source` input
- The PR guard prevents wasteful duplicate runs on the same branch
- Full git history (`fetch-depth: 0`) is required for version comparison in tagging jobs
- The workflow uses composite actions, making it easy to test and maintain individual components
