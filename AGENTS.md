# Repository Guidelines

## Project Structure & Module Organization
The repo intentionally stays lean: documentation and licensing live at the root, while everything production-grade sits under `.github`. Detailed docs accompany their subjects—workflows keep inline comments plus `.md` siblings (for example `.github/workflows/flutter-ci.yml` ⇢ `.github/workflows/flutter-ci.md`), and composite actions carry per-action `README.md` files inside `.github/actions/<name>/`. Reusable actions (`flutter-auto-tagging`, `flutter-test`) provide the core logic, and each workflow is merely a thin orchestration layer on top.

## Build, Test, and Development Commands
Authoring YAML alone is not enough—exercise the commands CI will run. Follow the per-workflow docs for nuance, but in every DraftMode Flutter package run:
- `dart format --output=none --set-exit-if-changed .` to ensure formatting parity with the formatter step.
- `dart analyze .` for the analyzer parity check.
- `flutter test` to mirror the CI test stage.
- `flutter gen-l10n` (if `l10n.yaml` exists) before committing so localization verification stays green.
To dry-run workflow logic, install `act` and execute `act pull_request -W .github/workflows/flutter-ci.yml` from this repo; provide a sample `pubspec.yaml` to satisfy auto-tagging.

## Coding Style & Naming Conventions
Keep YAML two-space indented, order keys logically (`name`, `on`, `jobs`, `permissions`), and maintain the explanatory comments introduced in this update. Composite action directories stay kebab-case, step IDs snake_case, and GitHub summary lines use checklist bullets just like the existing scripts. Any shell snippets should be POSIX-compliant `bash` blocks with defensive checks (`set -euo pipefail` when adding new scripts) and explicit logging so workflow output stays skimmable.

## Testing Guidelines
The auto-tagging action relies on git tags; when modifying it, create throwaway tags such as `v1.2.3` to confirm the guard rails fire. For flutter-test updates, validate inside a real DraftMode package by running the formatter/analyzer/test trio above and ensuring no untracked files appear after `flutter gen-l10n`. Prefer creating lightweight fixtures rather than editing production repos just to test CI changes, and document notable scenarios inside each action/workflow README so future maintainers know how to reproduce them.

## Commit & Pull Request Guidelines
Follow Conventional Commits (`ci:`, `chore:`, `fix:`) so downstream release tooling can auto-tag. Describe the affected workflow or action in the subject (`ci: make flutter-test cache pub packages`). Every PR should link the consuming repo or issue, include `act` output or screenshots when touching workflows, and confirm whether version bumps or tag cleanups are required.

## Security & Configuration Tips
Workflows require write access to create tags; never hard-code tokens. Use repository variables for package-specific knobs and prefer GitHub secrets for anything sensitive. Keep dependency pins (actions/checkout@v4, subosito/flutter-action@v2) current to receive upstream fixes without surprises.
