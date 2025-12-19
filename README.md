# DraftMode Workflow Library

DraftMode curates the workflows and composite GitHub Actions we reach for most often. The goal is simple: provide commonly used automation that anyone interested in DraftMode (or workflows in general) can drop into their repositories. Expect this catalog to grow continuously—and essentially without limit—as we codify more scenarios; each addition is documented separately so the root README stays focused on navigation.

## Growing Catalog
- Centralize automation primitives (test gates, tagging logic, etc.) behind reusable actions.
- Keep documentation close to the files they describe to make audits easy.
- Encourage contributions from anyone who spots an automation gap—open an issue or PR when a new workflow belongs here.

## Flutter Workflows
- [`flutter-ci.yml`](.github/workflows-kept/flutter-ci.md) — Pull-request guardrail that checks semantic version drift, formatting, analyzer warnings, localization artifacts, and `flutter test` results before merges.
- [`flutter-release.yml`](.github/workflows-kept/flutter-release.md) — Push-triggered release helper that bumps `pubspec.yaml`, commits the change, and creates a matching git tag so packages ship consistently.

## Node.js Workflows
- [`node-js-ci.yml`](.github/workflows/node-js-ci.md) — Pull-request/reusable workflow that validates `package.json` versions, runs lint/Prettier/badge scripts, and executes the Node test suite.

## Docker Workflows
- [`docker-build.yml`](.github/workflows/docker-build.md) — Reusable build wrapper that outputs both the digest and an artifact for later jobs.
- [`docker-push.yml`](.github/workflows/docker-push.md) — Reusable push workflow that rehydrates an artifact (or local image) and authenticates with the requested registry.

## Shared Actions
- [`flutter-auto-tagging`](.github/actions/flutter-auto-tagging/README.md) — Parses `pubspec.yaml`, validates existing tags, and emits both the current and next semantic versions.
- [`test-flutter`](.github/actions/test-flutter/README.md) — Sets up Flutter, verifies generated localizations, enforces formatting/analyzer rules, and runs the project's test suite.
- [`node-js-auto-tagging`](.github/actions/node-js-auto-tagging/README.md) — Reads `package.json`, enforces semver ordering vs. git tags, and emits both current and next patch versions for downstream workflows.
- [`test-node-js`](.github/actions/test-node-js/README.md) — Installs dependencies, runs lint/Prettier/badge npm scripts, and executes the Node test suite while emitting GitHub summary checklists.
- [`docker-build`](.github/actions/docker-build/README.md) — Builds Docker images (with optional Buildx reproducibility) and uploads them as artifacts.
- [`docker-push`](.github/actions/docker-push/README.md) — Loads an image (or artifact), retags if needed, logs into a registry, and pushes.
- [`artifact-from-image`](.github/actions/artifact-from-image/README.md) — Saves a Docker image to an artifact for later workflows.
- [`artifact-to-image`](.github/actions/artifact-to-image/README.md) — Downloads a Docker image artifact and loads it back into the Docker daemon.
