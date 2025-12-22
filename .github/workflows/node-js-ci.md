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
| `format-script`     | string  | No       | `''`    | npm script to run code formatting check (e.g., `format:check`). Leave blank to skip. |
| `badges-script`     | string  | No       | `''`    | npm script to refresh README/coverage badges. Leave blank to skip. |
| `test-script`       | string  | No       | `''`    | npm script to run test suite (e.g., `test`, `test:ci`). Leave blank to skip. |

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

### Docker

| Name                       | Type    | Required | Default         | Description |
|----------------------------|---------|----------|-----------------|-------------|
| `docker-image-name`        | string  | No       | `''`            | Docker image name (e.g., `ghcr.io/org/app`). Leave blank to skip Docker builds. |
| `docker-build-context`     | string  | No       | `'.'`           | Docker build context path. |
| `docker-build-args-file`   | string  | No       | `''`            | Path to file with Docker build args. Leave blank to skip. |
| `docker-build-options`     | string  | No       | `'--no-cache'`  | Additional Docker build options. |
| `docker-build-reproducible`| boolean | No       | `true`          | Build reproducible Docker image with `SOURCE_DATE_EPOCH`. |
| `docker-build-platform`    | string  | No       | `''`            | Target platform(s) for Docker build (e.g., `'linux/amd64'`, `'linux/amd64,linux/arm64'`). |
| `docker-tag-levels`        | string  | No       | `'patch,latest'`| Version levels to tag: `patch`, `minor`, `major`, `latest` (comma-separated). |
| `docker-registry`          | string  | No       | `'ghcr.io'`     | Docker registry URL (e.g., `'ghcr.io'`). Leave blank for Docker Hub. |
| `docker-registry-username` | string  | No       | `''`            | Docker registry username. If not provided, extracted from `docker-image-name` (e.g., `myuser/myapp` → `myuser`). |
| `docker-registry-password` | string  | No       | `''`            | Docker registry password. Defaults to `secrets.GITHUB_TOKEN` if not provided. |

### Input Notes

**Testing:**
- **Node.js Version**: Provide either `node-version` OR `node-version-env`. The workflow will fail if both are empty.
- **Scripts**: All script inputs default to empty strings. When empty, the test action skips that step and logs it in the workflow summary.

**Release Detection:**
- `ci-release-branch-patterns` defines which branches are considered "release" branches (defaults to `'main'`).
- Supports glob patterns like `'release/*'` or `'hotfix/*'`.
- Used by the `git-state` action to set `is-release-branch` output.

**Tagging:**
- Tagging jobs only run when `ci-tag-source` is provided.
- Use `nodejs` to read version from package.json, `flutter` for pubspec.yaml, or `target` to read from git tags.
- `ci-tag-increment-patch` automatically increments patch version when major.minor match the latest tag.
- `git-tag-levels` controls which git tags are created (default: `'patch'` for exact version only).
- Multi-level tags (`minor`, `major`) are automatically filtered if covered by version-like branches (e.g., `v1.2` branch skips `v1.2` multi-level tag).
- **Important**: When `docker-tag-levels` contains semantic versions (`patch`, `minor`, or `major`), `git-tag-levels` MUST include `'patch'` to maintain version consistency across git and Docker registry. This prevents version drift across builds.

**Docker:**
- Docker jobs only run when `docker-image-name` is provided.
- `docker-build-context` defaults to repository root (`.`).
- `docker-build-args-file` is optional - provide path to a file containing build args (one per line, format: `ARG=value`).
- `docker-build-options` defaults to `--no-cache` for clean builds.
- `docker-build-reproducible` uses `SOURCE_DATE_EPOCH` from git commit timestamp for reproducible builds.
- `docker-build-platform` enables multi-platform builds (e.g., `'linux/amd64,linux/arm64'`). Leave blank for default platform. Requires `docker-build-reproducible: true`.
- `docker-tag-levels` controls which version tags to create (defaults to `patch,latest`).
- `docker-registry` defaults to GitHub Container Registry (`ghcr.io`).
- `docker-registry-username` is inferred from `docker-image-name` if not provided (e.g., `myuser/myapp` → username: `myuser`).
- `docker-registry-password` defaults to `secrets.GITHUB_TOKEN` if not provided.
- Docker push requires both `docker-image-name` AND `ci-tag-source` to be configured.

## Permissions

The reusable workflow declares these permissions as its requirements:
```yaml
permissions:
  contents: write    # Required for git_push job
  pull-requests: read  # Required for git-state action
  packages: write      # Required for docker_push job
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
  packages: write      # Required for Docker push

jobs:
  node:
    uses: draftm0de/github.workflows/.github/workflows/node-js-ci.yml@main
    secrets: inherit
    with:
      node-version: '20'
      ci-tag-source: 'nodejs'
      docker-image-name: 'ghcr.io/org/app'
```

**Permission requirements by feature:**
- **All workflows**: `pull-requests: read` (for PR detection in git-state)
- **Git tagging** (`ci-tag-source` set): `contents: write`
- **Docker push** (`docker-image-name` set): `packages: write`

**Note:** If you only use testing features (no tagging/docker), you can use:
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

**Purpose**: Runs linting, formatting, and test suite.

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
2. Calls `test-node-js` action with all test/lint/format script inputs

**Uses action**: `draftm0de/github.workflows/.github/actions/test-node-js@main`

### 3. auto_tagging

**Purpose**: Calculates next version and creates git/docker tag lists.

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
4. Create docker tags (if `docker-tag-levels` provided) using `docker-tag-builder` action
   - Validates tag level consistency when `git-tag-levels` is also provided
   - Ensures semantic docker levels are subset of git levels
5. Create git tags (if `git-tag-levels` provided) using `git-tag-builder` action

**Outputs**:
- `next-version`: Next version with postfix (e.g., `v1.2.12+build`)
- `next-version-short`: Next version without postfix (e.g., `v1.2.12`)
- `is-latest-version`: `'true'` if this is the latest version on the target branch
- `docker-tags`: Space-separated Docker tags (e.g., `v1.2.12 latest`), empty if docker not configured
- `git-tags`: Space-separated git tags (e.g., `v1.2.12 v1.2`), empty if not created

**Uses actions**:
- `draftm0de/github.workflows/.github/actions/version-reader@main`
- `draftm0de/github.workflows/.github/actions/tag-builder@main`
- `draftm0de/github.workflows/.github/actions/docker-tag-builder@main`
- `draftm0de/github.workflows/.github/actions/git-tag-builder@main`

### 4. docker_build

**Purpose**: Builds Docker image and uploads as artifact.

**Runs on**: `ubuntu-latest`

**Depends on**: `setup`, `tests`

**Runs when**:
```yaml
docker-image-name != '' AND (
  github.event_name == 'pull_request' OR
  github.event_name == 'merge_group' OR
  (github.event_name == 'push' AND has-open-pull-requests == 'false')
)
```

This means:
- Only runs when Docker image name is provided
- Runs on PR events, merge queue events, or push events without open PRs
- Only after tests pass

**Steps**:
1. Checkout repository
2. Build Docker image using `docker-build` action
3. Upload Docker image as artifact using `artifact-from-image` action

**Outputs**:
- `image`: Docker image reference (e.g., `ghcr.io/org/app@sha256:abc123`)

**Uses actions**:
- `draftm0de/github.workflows/.github/actions/docker-build@main`
- `draftm0de/github.workflows/.github/actions/artifact-from-image@main`

### 5. git_push

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

### 6. docker_push

**Purpose**: Pushes Docker image to registry (release branches only).

**Runs on**: `ubuntu-latest`

**Depends on**: `setup`, `auto_tagging`, `docker_build`

**Runs when**:
```yaml
github.event_name == 'push' AND
is-release-branch == 'true' AND
docker-image-name != '' AND
docker-tags != ''
```

This means:
- Only runs on `push` events to release branches (e.g., `main`, `release/*`)
- Only when Docker image name is configured
- Only when docker tags were calculated by auto_tagging job
- Only after docker_build completes

**Steps**:
1. Checkout repository
2. Load Docker image from artifact using `artifact-to-image` action
3. Push Docker image to registry using `docker-push` action with:
   - `image`: Downloaded image from artifact
   - `tags`: Docker tags from auto_tagging job
   - `registry`: Registry input (defaults to `ghcr.io`)
   - `username`: Registry username input (inferred from image name if not provided)
   - `password`: Registry password input (defaults to `secrets.GITHUB_TOKEN`)

**Uses actions**:
- `draftm0de/github.workflows/.github/actions/artifact-to-image@main`
- `draftm0de/github.workflows/.github/actions/docker-push@main`

## Behavior

### Pull Request Guard

The workflow uses the `git-state` action to implement intelligent test execution:

**For pull_request events:**
- `github.event_name == 'pull_request'`
- Tests always run
- Docker builds and version validation run (if configured)

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
      ci-release-branch-patterns: 'main,develop'
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
      format-script: 'format:check'
      test-script: 'test:ci'
      ci-release-branch-patterns: 'main,release/*'
      ci-tag-source: 'nodejs'
      ci-tag-increment-patch: true
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
      ci-release-branch-patterns: 'main,release/*,hotfix/*'
      ci-tag-source: 'branch'
      ci-tag-increment-patch: false
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
- `test-node-js` - Runs Node.js tests and quality checks
- `version-reader` - Reads versions from package.json, pubspec.yaml, or git tags
- `tag-builder` - Builds and validates semantic version tags
- `docker-tag-builder` - Creates Docker image tags
- `git-tag-builder` - Creates git tags
- `docker-build` - Builds Docker images
- `docker-push` - Pushes Docker images to registry
- `git-push` - Pushes git tags to remote repository
- `artifact-from-image` - Uploads Docker images as artifacts

## Workflow State

**Current Status**: Fully implemented

**Implemented jobs**:
- ✅ `setup` - Git state detection
- ✅ `tests` - Test execution with PR guard and merge queue support
- ✅ `auto_tagging` - Version calculation and tag generation
- ✅ `docker_build` - Docker image builds (PRs, merge queue, and pushes)
- ✅ `git_push` - Git tag creation and push to remote (release branches only)
- ✅ `docker_push` - Docker image pushes to registry (release branches only)

## GitHub Configuration

When using automatic version patching (`ci-tag-increment-patch: true`), the repository requires specific settings:

### Required Workflow Permissions

The workflow already declares `contents: write` permission for version file updates and git tag pushes.

### Required Repository Settings

Enable "Allow GitHub Actions to create and approve pull requests":

1. Navigate to repository Settings → Actions → General
2. Scroll to "Workflow permissions"
3. Enable: ☑ Allow GitHub Actions to create and approve pull requests

See [GitHub Configuration](../../README.md#github) in the root README for detailed setup instructions.

## Notes

- The workflow enforces that either `node-version` or `node-version-env` is provided
- All test scripts are optional - skip by leaving blank
- Docker jobs are opt-in via `docker-image-name` input
- Tagging jobs are opt-in via `ci-tag-source` input
- **Version File Updates**: When `ci-tag-source` and `ci-tag-increment-patch: true` are set, the workflow automatically updates `package.json` (nodejs) or `pubspec.yaml` (flutter) with the new version and commits it to the release branch before creating git tags
- Docker tag calculation is conditional on `docker-tag-levels` input
- Git tag calculation is conditional on `git-tag-levels` input
- Git tags are pushed only when `git-tags` output is non-empty
- Docker push requires `docker-image-name` AND non-empty `docker-tags` output
- The PR guard prevents wasteful duplicate runs on the same branch
- Supports GitHub merge queue (`merge_group` events)
- Full git history (`fetch-depth: 0`) is required for version comparison in tagging jobs
- The workflow uses composite actions, making it easy to test and maintain individual components
- Git tags and Docker images are pushed only on `push` events to release branches (configured via `ci-release-branch-patterns`)
- **Tag Level Validation**: The `docker-tag-builder` action validates that semantic docker tag levels (`patch`, `minor`, `major`) are a subset of git tag levels. This ensures version consistency between git tags and Docker registry, preventing version drift where Docker images have semantic versions but git repository doesn't track them. For example, if you use `docker-tag-levels: 'latest,major,minor'`, then `git-tag-levels` must include both `'major'` and `'minor'`. Custom docker tags (`sha`, `edge`, `beta`, `latest`) are ignored in validation.
