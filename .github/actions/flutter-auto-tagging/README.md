# flutter-auto-tagging Action

Composite action that parses `pubspec.yaml`, validates existing git tags, and emits both the current and next semantic versions for downstream workflows.

## Inputs
- `pubspec-path` (optional, default `pubspec.yaml`) — Override when the manifest lives elsewhere.

## Outputs
- `version` / `version_with_build` — The exact semver parsed from the manifest.
- `build_metadata` — Any `+build` suffix that followed the base semver.
- `new_version` / `new_version_with_build` — The computed patch bump that preserves build metadata when present.

## How It Works
1. Reads the supplied pubspec file, failing if the `version:` entry is missing or malformed.
2. Fetches repo tags and ensures no published major/minor exceeds the pubspec declaration.
3. Increments the patch portion and appends build metadata (if any).

## Local Testing
Use [`act`](https://github.com/nektos/act) to trigger `.github/workflows/flutter-ci.yml` or another workflow that calls this action:
```bash
act workflow_dispatch -W .github/workflows/flutter-ci.yml
```
Seed the repo with throwaway tags (for example `git tag v9.9.8`) to confirm the guard rails behave as expected without polluting real history.
