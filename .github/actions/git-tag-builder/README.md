# git-tag-builder

Composite action that derives the next semver tag directly from the source version while ensuring git history is in sync.

## What it does
1. Reads the provided `source-version` argument (no file reads) and splits out any prefix (e.g. `v`, `release/`).
2. Fetches all git tags and determines the latest semver tag (accepts bare `x.y.z`, `vx.y.z`, or tags matching the detected prefix).
3. Fails if the repo tags are already ahead of the provided source version (major, minor, or patch drift).
4. Builds the next tag:
   - If the latest git tag shares the same major/minor, bump the git tag patch (`last patch + 1`).
   - Otherwise, reuse the source version.
5. Verifies that neither the prefixed candidate, `v<next_tag_short>`, nor `<next_tag_short>` already exists.

## Inputs
- `source-version`: Required version string (`<optional-prefix>x.y.z`).

## Outputs
- `last_tag`: Latest git tag including the detected prefix (if any).
- `last_tag_short`: Latest semver git tag (x.y.z) without prefix.
- `next_tag`: Next git tag including the detected prefix (if any).
- `next_tag_short`: Next tag to create (x.y.z) without prefix.

Use `next_tag_short` when updating source files that store bare versions and `next_tag` when creating annotated git tags.
