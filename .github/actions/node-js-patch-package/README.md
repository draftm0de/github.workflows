# node-js-patch-package

Composite action that writes a provided semver into `package.json` so workflows can bump versions in-place.

## Inputs
- `package-json-path` (optional, default `package.json`): Location of the manifest to update.
- `new-version` (required): Semver string (`x.y.z` with optional `+build`) to persist.

## Outputs
- `updated_version`: Echoes the version that was written to the manifest.

## Behavior
1. Validates the manifest path and new version format.
2. Uses Node.js to load the manifest, set `version`, and rewrites the file with 2-space indentation, preserving a trailing newline.
3. Summarizes the update for quick inspection in the workflow logs.

Pair it with `node-js-version-guard` (or any other version-calculation step) to bump the manifest before tagging or publishing artifacts.
