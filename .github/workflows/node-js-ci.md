# Node.js CI Workflow

Reusable workflow (`node-js-ci.yml`) that orchestrates DraftMode's Node.js CI pipeline. It installs dependencies, runs the repo-defined lint/format/badge/test scripts, and optionally builds/upload Docker images. Because it is `workflow_call`-only you must author an event-driven workflow (for example `.github/workflows/ci.yml`) in the consuming repo that invokes it on `push`/`pull_request`.

## Triggers
- `workflow_call`: reusable workflow that must be invoked from another workflow file.

## Inputs
| Name              | Default | Description |
|-------------------|---------|-------------|
| `node_version`    | `''`    | Node.js version forwarded to `actions/setup-node`. Required when `node_version_env` is blank. |
| `node_version_env`| `''`    | Optional path to an env file that exports `NODE_VERSION`. When set it overrides `node_version`. |
| `enable_cache`    | `'true'`| Toggles npm caching inside `actions/setup-node`. Requires committed `package-lock.json`. |
| `lint_script`     | `''`    | npm script name executed by `.github/actions/node-js-test`. Leave blank to skip linting. |
| `prettier_script` | `''`    | npm script name for Prettier checks. Leave blank to skip. |
| `badges_script`   | `''`    | npm script that refreshes README / coverage badges. Leave blank to skip. |
| `test_script`     | `''`    | npm script that runs the test suite. Leave blank to skip (not recommended). |
| `docker_image`    | `''`    | Image tag passed to the Docker build/upload jobs when a pull request is under test. Leave blank to skip those jobs. |

Provide either `node_version` or `node_version_env`. The composite action fails fast when both are empty so callers do not accidentally run with whatever Node version happens to be on the runner. Script inputs default to empty strings, which causes the composite action to skip each step after writing a checklist entry to the workflow summary—configure the scripts explicitly for every repo so CI matches your package scripts.

## Secrets & Permissions
Callers should inherit secrets so the workflow can read `${{ secrets.GITHUB_TOKEN }}` for the pull-request guard and any private registry credentials.
```yaml
jobs:
  node:
    uses: draftm0de/github.workflows/.github/workflows/node-js-ci.yml@main
    secrets: inherit
```
The workflow itself requests `contents: read` and `pull-requests: read`. Hosted runners already satisfy the requirements for the guard action (`gh` + authenticated token). Self-hosted runners must provide `gh`, `jq`, and ensure `GH_TOKEN`/`GITHUB_TOKEN` are readable.

## Job Overview
- `setup`: Runs `.github/actions/git-state` to detect whether the ref is already a pull request or whether a push branch has an open PR. The job outputs `is-pull-request` and `ci-open-pull-requests` booleans that downstream jobs use to avoid duplicate test runs.
- `tests`: Always runs on pull requests and only runs on pushes when there is no open PR for the same branch. It checks out the repo and executes `.github/actions/node-js-test`, which handles installing dependencies plus lint/format/badge/test scripts.
- `build_on_pull`: Optional Docker build/upload flow for pull requests. It runs only when `docker_image` is non-empty so repositories can opt in job-by-job.
- `tag_on_pull`: Builds version/tag metadata for PR validation using `.github/actions/node-js-version-builder`. It fetches the entire git history (`fetch-depth: 0`) so semantic versioning logic has the tags it needs.

## Example Integration
```yaml
# .github/workflows/ci.yml in a consuming repository
name: Node CI

on:
  push:
    branches: ["main", "release/*"]
  pull_request:
    branches: ["main"]

jobs:
  node:
    uses: draftm0de/github.workflows/.github/workflows/node-js-ci.yml@main
    with:
      node_version: '20'
      lint_script: 'lint'
      prettier_script: ''        # record a skip in the summary
      test_script: 'test:ci'
      docker_image: ghcr.io/my-org/my-app:${{ github.sha }}
    secrets: inherit
```

In this setup the reusable workflow automatically skips redundant push jobs when the same branch already has an open pull request. Pull requests keep the canonical test run plus optional Docker build/upload. Repositories that centralize their Node version in `.env.build` files can pass `node_version_env: '.env.build'` instead of hard-coding versions in multiple places.

## Pull Request Guard
`.github/actions/git-state` inspects `GITHUB_REF` to short-circuit during pull_request events and calls the GitHub GraphQL API (via `gh`) on push events to see whether the branch already has an open PR. Its outputs are consumed by the `tests` job condition:
```yaml
if: needs.setup.outputs.is-pull-request == 'true' ||
    needs.setup.outputs.ci-open-pull-requests == 'false'
```
This ensures a branch's push workflow does not waste cycles running the same suite while the PR workflow is already in progress, but still runs on branches without open PRs (for example release branches or first-time pushes).
