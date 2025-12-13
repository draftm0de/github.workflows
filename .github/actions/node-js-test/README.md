# node-js-test Action

Composite action that enforces the Node.js quality gates DraftMode expects in CI. It installs dependencies with `npm ci`, runs repo-defined lint/format/badge/test scripts, and records each result in the GitHub step summary so downstream workflows remain easy to audit.

## Inputs
| Name | Default | Description |
|------|---------|-------------|
| `node_version` | `''` | Node.js version forwarded to `actions/setup-node`. Required when `node_version_env` is blank. |
| `node_version_env` | `''` | Path to an env file that exports `NODE_VERSION`. The file is sourced with `set -a` / `source <file>` / `set +a`, so only reference trusted files. Overrides `node_version` when present. |
| `enable_cache` | `'true'` | Toggles npm caching inside `actions/setup-node`. Requires checked-in `package-lock.json`. |
| `lint_script` | `''` | npm script that runs linting. Leave blank to skip and record a checkbox entry in the summary. |
| `prettier_script` | `''` | npm script that runs Prettier in check mode. Leave blank to skip. |
| `badges_script` | `''` | npm script that refreshes README / coverage badges. Leave blank to skip. |
| `test_script` | `''` | npm script that runs the test suite. Leave blank to skip (not recommended). |

## Behavior
1. Validates the Node version source. When `node_version_env` is set the action sources the file and reads `NODE_VERSION`; otherwise it uses the explicit `node_version` input. Missing values cause the action to fail fast so runners never use an unintended system default.
2. Installs Node via `actions/setup-node@v6`, optionally enabling npm caching based on `enable_cache`.
3. Runs `npm ci` (failing when `package-lock.json` is missing) to ensure reproducible dependency installs.
4. Executes the configured lint, Prettier, badge, and test scripts. Each step always runs so the GitHub job summary shows either a completed checkbox or a "skipped" entry. The action errors when a non-empty script name is not defined inside `package.json`.

## Local Parity
Before opening a pull request, mirror the workflow locally with the same script names you pass to the action:
```bash
npm ci
npm run lint           # replace with your lint_script
npm run format:check   # replace with your prettier_script
npm run badges         # replace with your badges_script when applicable
npm test               # replace with your test_script
```
Use [`act`](https://github.com/nektos/act) pointed at `.github/workflows/node-js-ci.yml` if you need to dry-run the full workflow that consumes this action.
