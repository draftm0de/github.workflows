# node-js-version-builder

Composite action that combines package version discovery with git tag planning so workflows can safely update `package.json` and create tags in a single step.

## How it works
1. Checks out the repository (full history) so git metadata and `package.json` are available.
2. Reads the version from `package.json` via `node-js-version-reader`.
3. Invokes `git-tag-builder` using that version, which enforces drift rules and emits the next tag candidates (with and without prefixes).

## Inputs
This composite has no user-configurable inputs—it always reads `package.json` at the repository root via `node-js-version-reader`.

## Outputs
| Name | Description |
| --- | --- |
| `version` | Bare semver (x.y.z) read from `package.json`. |
| `last_tag` | Latest git tag including the detected prefix (e.g., `v1.2.3`). |
| `last_tag_short` | Latest tag stripped to bare semver (`1.2.3`). |
| `next_tag` | Next git tag to create, including prefix. |
| `next_tag_short` | Next tag as bare semver. |

Use `next_tag_short` when updating `package.json` (or other source files) and `next_tag` when running `git tag` / publishing releases.
