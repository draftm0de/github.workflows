# node-js-patch-package

Composite action that writes a provided semver into `package.json` so workflows can bump versions in-place.

## Inputs
| Name | Description | Required | Default |
| --- | --- | --- | --- |
| `package-json-path` | Location of the manifest to update. | No | `package.json` |
| `new-version` | Semver string (`x.y.z` with optional `+build`) to persist. | Yes | — |

## Outputs
| Name | Description |
| --- | --- |
| `updated_version` | Version that was written to the manifest. |
| `git_next_tag` | Prefixed tag (e.g., `v1.2.4`) returned from the internal `node-js-version-builder` step. |
| `git_next_tag_short` | Bare semver tag (e.g., `1.2.4`) emitted alongside `git_next_tag`. |

## Behavior
1. Validates the manifest path and new version format.
2. Uses Node.js to load the manifest, set `version`, and rewrites the file with 2-space indentation, preserving a trailing newline.
3. Summarizes the update for quick inspection in the workflow logs.

Pair it with `node-js-version-builder` (already invoked internally) or other release steps to bump the manifest before tagging or publishing artifacts.

## Usage
```yaml
- name: Patch package version
  id: patch
  uses: draftm0de/github.workflows/.github/actions/node-js-patch-package@main
  with:
    package-json-path: package.json
    new-version: 1.2.3
```
