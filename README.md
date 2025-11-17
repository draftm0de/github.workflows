# DraftMode Workflow Library

DraftMode curates the workflows and composite GitHub Actions we reach for most often. The goal is simple: provide commonly used automation that anyone interested in DraftMode (or workflows in general) can drop into their repositories. Expect this catalog to grow continuously—and essentially without limit—as we codify more scenarios; each addition is documented separately so the root README stays focused on navigation.

## Growing Catalog
- Centralize automation primitives (test gates, tagging logic, etc.) behind reusable actions.
- Keep documentation close to the files they describe to make audits easy.
- Encourage contributions from anyone who spots an automation gap—open an issue or PR when a new workflow belongs here.

## Flutter Workflows
- [`flutter-ci.yml`](.github/workflows/flutter-ci.md) — Pull-request guardrail that checks semantic version drift, formatting, analyzer warnings, localization artifacts, and `flutter test` results before merges.
- [`flutter-release.yml`](.github/workflows/flutter-release.md) — Push-triggered release helper that bumps `pubspec.yaml`, commits the change, and creates a matching git tag so packages ship consistently.

## Shared Actions
- [`flutter-auto-tagging`](.github/actions/flutter-auto-tagging/README.md) — Parses `pubspec.yaml`, validates existing tags, and emits both the current and next semantic versions.
- [`flutter-test`](.github/actions/flutter-test/README.md) — Sets up Flutter, verifies generated localizations, enforces formatting/analyzer rules, and runs the project’s test suite.

## Related DraftMode Packages
These sibling repositories consume the workflows/actions above to keep tooling consistent across the ecosystem:

- [`draftmode_worker`](https://github.com/draftm0de/flutter.worker)
- [`draftmode_example`](https://github.com/draftm0de/flutter.example)
- [`draftmode_formatter`](https://github.com/draftm0de/flutter.formatter)
- [`draftmode_localization`](https://github.com/draftm0de/flutter.localization)
- [`draftmode_ui`](https://github.com/draftm0de/flutter.ui)
