# Node JS CI Workflow

Protects DraftMode Node repositories by enforcing semantic version checks, linting, Prettier formatting, badge regeneration, and tests on every pull request/manual run.

## When It Runs

- `push` events to `main` branch
- `pull_request` events (opened, synchronized, reopened, ready for review)
- Manual invocations via `workflow_dispatch`
- Reusable via `workflow_call` so other repos can `uses:` it directly
- Jobs auto-skip inside `draftm0de/github.workflows` (template repo) and `draftm0de/flutter.clone`

## Job Overview

1. **verify-version-tag** — Checks out the repository with full history and executes `node-js-auto-tagging` action to compare the `package.json` version against existing tags.
2. **tests** — Repeats the checkout in a fresh runner, then calls `node-js-test` composite action to:
   - Setup Node.js with optional npm caching
   - Verify `package-lock.json` exists (required for reproducible builds)
   - Install dependencies using `npm ci`
   - Run lint script (if configured)
   - Run Prettier format check (if configured)
   - Run badges refresh script (if configured)
   - Run test suite (if configured)

## How to Use

Reference the workflow from your repository so you inherit updates automatically:

```yaml
# .github/workflows/node-js-ci.yml inside your repo
name: Node JS CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  draftmode-node-ci:
    uses: draftm0de/github.workflows/.github/workflows/node-js-ci.yml@main
    secrets: inherit
    with:
      node-version: '22'
      enable-cache: 'true'
      lint-script: 'lint'
      prettier-script: 'format:check'
      badges-script: ''         # Skip if not needed
      test-script: 'test'
```

Override any `with` inputs to match the npm scripts your project exposes. Leave a script blank (`''`) to skip that gate entirely.

## Inputs

- `node-version` _(default `22`)_ — Node.js version forwarded to `actions/setup-node`.
- `enable-cache` _(default `true`)_ — Enable npm dependency caching (requires package-lock.json). Set to `false` to disable.
- `lint-script` _(default `lint`)_ — npm script name used for linting. Set to `''` (empty) to skip.
- `prettier-script` _(default `format:check`)_ — npm script for Prettier check mode. Set to `''` to skip.
- `badges-script` _(default `badges`)_ — npm script that refreshes README/coverage badges. Set to `''` to skip.
- `test-script` _(default `test`)_ — npm script that executes the main test suite. Set to `''` to skip.

## Prerequisites

**Required files:**
- `package.json` - Node.js project manifest
- `package-lock.json` - Dependency lockfile (must be committed)

If `package-lock.json` is missing, the CI will fail. Generate it with:
```bash
npm install
git add package-lock.json
git commit -m "Add package-lock.json"
```

## Local Parity

```bash
npm ci              # Requires package-lock.json
npm run lint
npm run format:check
npm run badges      # Optional, skip if not configured
npm test
```

Use [`act`](https://github.com/nektos/act) with `node-js-ci.yml` to dry-run the workflow. Bump `package.json` with `npm version --no-git-tag-version 0.0.1`, add throwaway tags (`git tag v0.0.1`) so the tag gate has realistic data, and clean them up afterwards.
