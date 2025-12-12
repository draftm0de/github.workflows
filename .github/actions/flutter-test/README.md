# flutter-test Action

Composite action that mirrors the DraftMode Flutter CI gate locally: set up Flutter, ensure generated localizations are committed, enforce formatting/analyzer cleanliness, and run tests.

## Steps Performed
1. Checkout repo SHA so jobs operate on the correct sources.
2. Install Flutter via `subosito/flutter-action`, using the channel/version declared in `pubspec.yaml`.
3. Cache `~/.pub-cache` and `.dart_tool` directories to speed repeat runs.
4. Execute `flutter pub get`.
5. If `l10n.yaml` exists, run `flutter gen-l10n` and fail when new files appear.
6. Run `dart format --output=none --set-exit-if-changed .` and `dart analyze .`.
7. Finish with `flutter test`.

## Usage
```yaml
- name: Run Flutter gate
  uses: draftm0de/github.workflows/.github/actions/flutter-test@main
```
This action assumes the calling repo already contains a valid Flutter app or package. Add any additional quality gates upstream (e.g., code generation) before invoking it.

## Inputs
| Name | Description | Required | Default |
| --- | --- | --- | --- |
| `pub-cache-key` | Override for the cache key used with `~/.pub-cache` and `.dart_tool`. When omitted, the action hashes any `pubspec.lock` and `pubspec.yaml` files so packages without a checked-in lockfile still get deterministic caches. | No | — |

When you set `pub-cache-key`, include every signal that should invalidate the cache (e.g., lockfile hash, Flutter channel, branch). A simple pattern is:
```yaml
- name: Run Flutter gate
  uses: draftm0de/github.workflows/.github/actions/flutter-test@main
  with:
    pub-cache-key: ${{ runner.os }}-pub-${{ github.ref_name }}-${{ hashFiles('**/pubspec.yaml') }}
```
If the key stays static while dependencies change, GitHub Actions may restore stale packages, so make sure the expression reflects the inputs that matter in your workflow.

## Local Testing
Mirror the same steps manually before pushing a branch:
```bash
flutter pub get
flutter gen-l10n   # when l10n.yaml exists
dart format --output=none --set-exit-if-changed .
dart analyze .
flutter test
```
If you need to test the action itself, rely on [`act`](https://github.com/nektos/act) with `.github/workflows/flutter-ci.yml` so the composite action runs exactly as it would in GitHub Actions.
