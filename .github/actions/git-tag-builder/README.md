# git-tag-builder

Composite action that proposes the next semantic version tag for a pull request. It can derive the version directly from the branch naming convention or from the package metadata, and applies guardrails so git history never outruns the declared version.

## What it does
1. Determine the source version:
   - `tagging-mode=branch`: use the provided `branch-name` (e.g. `release/v1.2.3`).
   - `tagging-mode=file`: use the provided `source-version` (usually read via `.github/actions/node-js-version-reader`).
2. Split any non-numeric prefix (`v`, `release/`, etc.) from the `x.y.z` payload and validate that the base version is semver.
3. Fetch tags, detect the latest semver value that either matches the prefix, `v<x.y.z>`, or a bare `<x.y.z>` tag.
4. Fail when the git tag is ahead of the source version (major/minor/patch drift) so the repo cannot silently skip bumps.
5. Build `next-tag-short`:
   - Start from the source version.
   - If `tagging-patch` is `true` and git already has the same major/minor, bump the patch beyond the latest git tag instead of reusing the same number.
6. Ensure the prefixed version, `v<next-tag-short>`, and `<next-tag-short>` do not already exist.
7. Emit `last_*` / `next_*` outputs plus a GitHub summary line so downstream workflows can decide whether to create a tag or update manifest files.

## Inputs
| Name | Description | Required | Default |
| --- | --- | --- | --- |
| `tagging-mode` | Strategy used to derive the source version. Accepts `branch` or `file`. | Yes | — |
| `branch-name` | Branch to parse when `tagging-mode=branch`. Usually comes from `.github/actions/git-state`. | Conditionally (branch mode) | — |
| `source-version` | Version string (`<optional-prefix>x.y.z`) to parse when `tagging-mode=file`. | Conditionally (file mode) | `''` |
| `tagging-patch` | When `true`, bump the patch when git already has the same major/minor to avoid duplicate tags. | No | `false` |

## Outputs
| Name | Description |
| --- | --- |
| `last-tag` | Latest git tag including the detected prefix (if any). |
| `last-tag-short` | Latest semver git tag (x.y.z) without prefix. |
| `next-tag` | Next git tag including the detected prefix (if any). |
| `next-tag-short` | Next tag candidate (x.y.z) without prefix. |

Use `next-tag` for the actual git tag to create (`git tag ${{ steps.tags.outputs['next-tag'] }}`) and `next-tag-short` when updating files that store bare versions such as `package.json`.
