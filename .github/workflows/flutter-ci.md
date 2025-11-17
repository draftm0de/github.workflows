# Flutter CI Workflow

This workflow guards DraftMode Flutter packages by enforcing formatting, analyzer cleanliness, tests, and semantic version hygiene on every pull request and manual run.

## When It Runs
- `pull_request` events targeting `main` (opened, synchronized, reopened, or marked ready for review)
- Manual invocations via `workflow_dispatch` or other workflows using `workflow_call`
- Jobs auto-skip when invoked inside this catalog repo (`draftm0de/github.workflows`) or the `draftm0de/flutter.clone` mirror to avoid loops

## Job Overview
1. **verify-version-tag** — Checks out the repo with full history and runs the `flutter-auto-tagging` action to ensure the current `pubspec.yaml` version doesn’t lag behind existing git tags.
2. **tests** — Repeats the checkout, re-reads pubspec metadata for downstream consumers, then executes the `flutter-test` composite action to set up Flutter, enforce formatting/analyzer consistency, and run `flutter test`.

## Local Parity
Replicate the same gates before opening a PR:
```bash
flutter pub get
flutter gen-l10n   # when l10n.yaml exists
dart format --output=none --set-exit-if-changed .
dart analyze .
flutter test
```
If you need to dry-run the workflow logic itself, install [`act`](https://github.com/nektos/act) and execute:
```bash
act pull_request -W .github/workflows/flutter-ci.yml
```
Provide a sample `pubspec.yaml` so the version job can complete.
