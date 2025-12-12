# Node.js CI Workflow

Reusable workflow (`node-js-ci.yml`) that orchestrates linting, formatting, testing, and optional Docker builds for Node.js projects. It exposes the same knobs as the composite actions underneath so downstream repos can opt in to caching, badge refreshes, or Docker artifacts. Because the workflow is `workflow_call`-only, you must create an event-driven workflow (see `example.yml`) in the consuming repo to trigger it on `push`/`pull_request`.

## Triggers
- `workflow_call`: reusable workflow that must be invoked from another workflow (see `example.yml`).

## Inputs
| Name               | Default        | Description                                                                                                          |
|--------------------|----------------|----------------------------------------------------------------------------------------------------------------------|
| `node-version`     | `22`           | Node.js version passed to `actions/setup-node`.                                                                      |
| `node-version-env` | `''`           | Optional env file path that defines `NODE_VERSION`, overriding `node-version` (useful when sharing `.env`/`.nvmrc`). |
| `enable-cache`     | `true`         | Whether to enable npm caching in `setup-node`.                                                                       |
| `lint-script`      | `lint`         | npm script name for linting (`''` to skip).                                                                          |
| `prettier-script`  | `format:check` | npm script that runs Prettier in check mode (`''` to skip).                                                          |
| `badges-script`    | `badges`       | npm script that refreshes README/coverage badges (`''` to skip).                                                     |
| `test-script`      | `test`         | npm script that runs the test suite (`''` to skip).                                                                  |
| `docker-image`     | `''`           | When set, enables Docker build / artifact jobs for pull requests.                                                    |

## Secrets
When invoked via `workflow_call`, inherit the caller’s secrets so the workflow can use `${{ secrets.GITHUB_TOKEN }}` for the `git-state` guard and any package-registry credentials you rely on. If secrets are not inherited the action defaults to `exists=false`, meaning tests will always run on pushes.
```yaml
uses: draftm0de/github.workflows/.github/workflows/node-js-ci.yml@main
secrets: inherit
```
Ensure the caller grants `pull-requests: read` so the custom action can check for existing PRs.

## Example Integration
```yaml
# .github/workflows/ci.yml in a consuming repository
name: Reuse node-js-ci

on:
  push:
    branches: ["main", "release/*"]
  pull_request:
    branches: ["main"]

jobs:
  node:
    uses: draftm0de/github.workflows/.github/workflows/node-js-ci.yml@main
    with:
      node-version: '20'
      lint-script: 'lint'
      prettier-script: ''        # skip prettier step
      test-script: 'test:ci'
      docker-image: ghcr.io/my-org/my-app:${{ github.sha }}
    secrets: inherit
```

In this setup the reusable workflow automatically skips the push tests whenever a branch already has an open pull request (so the PR workflow remains the single source of truth), but still executes the PR and optional Docker jobs whenever the caller supplies `docker-image`. Callers that keep the canonical Node version inside a dotenv-style file can pass `node-version-env: '.env.build'` to avoid duplicating version declarations (the composite action sources the file with `set -a; source <file>; set +a`, so only reference trusted env files).

## Pull Request Guard

The workflow delegates branch-deduplication to `.github/actions/git-state`, which reads `${{ secrets.GITHUB_TOKEN }}` to query the pull request API via the GitHub CLI (`gh`). Hosted runners ship with `gh` preinstalled. Self-hosted runners must provide the CLI and set `GITHUB_TOKEN`/`GH_TOKEN` environment variables so the action can authenticate. The action exposes outputs `exists`, `count`, and `ci-open-pull-requests` for backwards compatibility.
