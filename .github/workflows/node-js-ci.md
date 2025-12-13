# Node.js CI Workflow

Reusable workflow (`node-js-ci.yml`) that orchestrates DraftMode's Node.js CI pipeline. It installs dependencies, runs the repo-defined lint/format/badge/test scripts, and optionally builds/upload Docker images. Because it is `workflow_call`-only you must author an event-driven workflow (for example `.github/workflows/ci.yml`) in the consuming repo that invokes it on `push`/`pull_request`.

## Triggers
- `workflow_call`: reusable workflow that must be invoked from another workflow file.

## Inputs
| Name              | Default | Description |
|-------------------|---------|-------------|
| `node-version`    | `''`    | Node.js version forwarded to `actions/setup-node`. Required when `node-version-env` is blank. |
| `node-version-env`| `''`    | Optional path to an env file that exports `NODE_VERSION`. When set it overrides `node-version`. |
| `enable-cache`    | `true`  | Toggles npm caching inside `actions/setup-node`. Requires committed `package-lock.json`. |
| `lint-script`     | `''`    | npm script name executed by `.github/actions/node-js-test`. Leave blank to skip linting. |
| `prettier-script` | `''`    | npm script name for Prettier checks. Leave blank to skip. |
| `badges-script`   | `''`    | npm script that refreshes README / coverage badges. Leave blank to skip. |
| `test-script`     | `''`    | npm script that runs the test suite. Leave blank to skip (not recommended). |
| `docker-image`    | `''`    | Image tag passed to the Docker build/upload jobs when a pull request is under test. Leave blank to skip those jobs. |
| `tagging-mode`    | `''`    | Auto-tagging strategy for PRs. Use `branch` to derive the tag from the branch name or `file` to use the package version. Leave blank to disable tagging. |
| `tagging-patch`   | `false` | When `true`, bumps the patch version beyond the latest git tag when major/minor match. Ignored when `tagging-mode` is blank. |

Provide either `node-version` or `node-version-env`. The composite action fails fast when both are empty so callers do not accidentally run with whatever Node version happens to be on the runner. Script inputs default to empty strings, which causes the composite action to skip each step after writing a checklist entry to the workflow summary—configure the scripts explicitly for every repo so CI matches your package scripts. Tagging inputs are optional; leave `tagging-mode` blank when a repo does not need semver/tag validation.

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
- `setup`: Runs `.github/actions/git-state` to detect whether the ref is already a pull request, whether a push branch has an open PR, and to surface the normalized branch name via `branch-name`. It also validates that `tagging-mode` is blank, `branch`, or `file` before downstream jobs continue.
- `tests`: Always runs on pull requests and only runs on pushes when there is no open PR for the same branch. It checks out the repo and executes `.github/actions/node-js-test`, which handles installing dependencies plus lint/format/badge/test scripts.
- `build_on_pull`: Optional Docker build/upload flow for pull requests. It runs only when `docker-image` is non-empty so repositories can opt in job-by-job; the job depends on both `setup` and `tests` so Docker builds only happen after the quality gates pass.
- `tag_on_pull`: Builds version/tag metadata for PR validation using `.github/actions/node-js-version-builder` and `.github/actions/git-tag-builder`. It fetches the entire git history (`fetch-depth: 0`) so semantic versioning logic has the tags it needs and pulls the branch name from `setup` when `tagging-mode == 'branch'`.

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
      node-version: '20'
      lint-script: 'lint'
      prettier-script: ''        # record a skip in the summary
      test-script: 'test:ci'
      docker-image: ghcr.io/my-org/my-app:${{ github.sha }}
      tagging-mode: 'file'
      tagging-patch: true
    secrets: inherit
```

In this setup the reusable workflow automatically skips redundant push jobs when the same branch already has an open pull request. Pull requests keep the canonical test run plus optional Docker build/upload. Repositories that centralize their Node version in `.env.build` files can pass `node-version-env: '.env.build'` instead of hard-coding versions in multiple places.

## Pull Request Guard
`.github/actions/git-state` inspects `GITHUB_REF` to short-circuit during pull_request events and calls the GitHub GraphQL API (via `gh`) on push events to see whether the branch already has an open PR. Its outputs are consumed by the `tests` job condition:
```yaml
if: needs.setup.outputs.is-pull-request == 'true' ||
    needs.setup.outputs.ci-open-pull-requests == 'false'
```
This ensures a branch's push workflow does not waste cycles running the same suite while the PR workflow is already in progress, but still runs on branches without open PRs (for example release branches or first-time pushes).

## Auto Tagging
When `tagging-mode` is set, pull requests run the `tag_on_pull` job to compare the package version against the git history and calculate the next tag candidate:

- `branch`: treat the branch name as the source version. This is useful for release branches named like `release/v1.2.3`, ensuring the branch naming convention directly drives the tag candidate.
- `file`: read the version from `package.json` via `.github/actions/node-js-version-reader`.

Setting `tagging-patch: true` keeps the tag monotonic within the same major/minor by patch-bumping when the latest git tag already matches the source major/minor. Use this when multiple PRs can merge without bumping the version file but you still want unique tags. Leave `tagging-mode` blank to skip the job entirely (for example infrastructure repos that do not publish artifacts).
