# node-js-auto-tagging Action

Composite action that parses `package.json`, validates existing git tags, and emits both the current and next semantic versions for downstream workflows.

## Inputs
- `package-json-path` (optional, default `package.json`) — Override when the manifest lives elsewhere.

## Outputs
- `version` / `version_with_build` — The exact semver parsed from `package.json` (minus the optional `v` prefix).
- `build_metadata` — Any `+build` suffix that followed the base semver.
- `new_version` / `new_version_with_build` — The computed patch bump that preserves build metadata when present.

## How It Works
1. Parses the provided `package.json`, extracting the `version` field and stripping an optional leading `v`.
2. Ensures the resulting semver matches `x.y.z` (with optional `+build`), emitting helpful errors if it does not.
3. Fetches repo tags and ensures no published major/minor exceeds the declared version.
4. Increments the patch portion and appends build metadata (if any) so workflows can update files and tags safely.

## Local Testing
Use [`act`](https://github.com/nektos/act) to trigger `.github/workflows/node-js-ci.yml` or another workflow that calls this action:
```bash
npm version --no-git-tag-version 1.2.3
act workflow_dispatch -W .github/workflows/node-js-ci.yml --input node-version=22
```
Drop throwaway tags (`git tag v9.9.8`) prior to running the workflow to confirm the guard rails behave as expected without polluting real history. Delete those tags when you finish (`git tag -d v9.9.8`).
