# Flutter Release Workflow

This workflow automates the pubspec bump + git tagging flow whenever `main` advances, ensuring a predictable stream of patch releases for DraftMode consumers.

## When It Runs
- `push` events to `main`
- Direct invocations from other workflows via `workflow_call`
- Automatically skips execution when triggered from this catalog repository so we never tag the automation repo itself

## Job Overview
- **version-auto-tag** — Checks out the repo with full history, configures the GitHub Actions bot identity, and runs the shared `flutter-auto-tagging` action. If a new patch value is available, it rewrites `pubspec.yaml`, commits with `chore: auto-tagging version to x.y.z [skip ci]`, and pushes. When a plain semver increment is needed, it tags the commit as `vx.y.z` and pushes the tag upstream.

## How to Use
Add a workflow in your downstream repo that either calls this one via `workflow_call` or uses the file as a template. Calling keeps you on the canonical implementation:

```yaml
# .github/workflows/release.yml in your repo
name: Flutter Release

on:
  push:
    branches: [main]

jobs:
  draftmode-flutter-release:
    uses: draftm0de/github.workflows/.github/workflows/flutter-release.yml@main
    secrets: inherit
```
The release workflow needs `contents: write` permission to commit/tag, so ensure the calling repository’s default permissions allow it or set `permissions: contents: write` in your wrapper workflow. Pin to a tag or commit if you need reproducible behavior.

## Operating Notes
- The workflow requests `contents: write` permissions so it can push commits and tags—keep tokens scoped only to what is needed.
- To validate changes safely, run the `flutter-auto-tagging` action locally (via `act` or `bash`) against a throwaway repo with dummy tags such as `v1.2.3` to experience the guard rails.
- Any edits that alter how versions are computed should be accompanied by an update to `.github/actions/flutter-auto-tagging/README.md` so downstream repos know what to expect.
