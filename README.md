# DraftMode Workflow Library

DraftMode curates the workflows and composite GitHub Actions we reach for most often. The goal is simple: provide commonly used automation that anyone interested in DraftMode (or workflows in general) can drop into their repositories. Expect this catalog to grow continuously—and essentially without limit—as we codify more scenarios; each addition is documented separately so the root README stays focused on navigation.

## Growing Catalog
- Centralize automation primitives (test gates, tagging logic, etc.) behind reusable actions.
- Keep documentation close to the files they describe to make audits easy.
- Encourage contributions from anyone who spots an automation gap—open an issue or PR when a new workflow belongs here.

## Flutter Workflows
- [`flutter-ci.yml`](.github/workflows/flutter-ci.md) — Complete CI/CD reusable workflow with Flutter testing and semantic version validation. Supports version-based releases with automated git tagging. Publishing to pub.dev coming soon.

## Node.js Workflows
- [`node-js-ci.yml`](.github/workflows/node-js-ci.md) — Complete CI/CD reusable workflow with testing, semantic versioning, Docker builds, and automated tagging. Supports version-based releases with git tags and Docker registry pushes.

## Shared Actions

### Testing
- [`test-flutter`](.github/actions/test-flutter/README.md) — Sets up Flutter, verifies generated localizations, enforces formatting/analyzer rules, and runs the project's test suite.
- [`test-node-js`](.github/actions/test-node-js/README.md) — Installs dependencies, runs lint/Prettier/badge npm scripts, and executes the Node test suite while emitting GitHub summary checklists.

### Git & Workflow State
- [`git-state`](.github/actions/git-state/README.md) — Detects PR state, branch information, and prevents duplicate test runs when both push and pull_request events trigger.
- [`git-push`](.github/actions/git-push/README.md) — Pushes git tags to remote repositories.

### Versioning & Tagging
- [`version-reader`](.github/actions/version-reader/README.md) — Reads versions from package.json, pubspec.yaml, or git tags for semantic versioning workflows.
- [`tag-builder`](.github/actions/tag-builder/README.md) — Builds and validates semantic version tags with auto-increment support and version drift detection.
- [`version-patch`](.github/actions/version-patch/README.md) — Updates package.json or pubspec.yaml with new version and commits to main. Triggered when `ci-tag-source` and `ci-tag-increment-patch: true` are set. **Requires `contents: write` permission** and [Allow GitHub Actions to create and approve pull requests](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/enabling-features-for-your-repository/managing-github-actions-settings-for-a-repository#preventing-github-actions-from-creating-or-approving-pull-requests) enabled in repository settings.
- [`git-tag-builder`](.github/actions/git-tag-builder/README.md) — Creates git tags for semantic versions, filtering out levels covered by version-like branches.
- [`docker-tag-builder`](.github/actions/docker-tag-builder/README.md) — Builds Docker image tags from semantic versions or custom tags (sha, edge, beta). Supports patch/minor/major levels and latest tag detection.

### Docker
- [`docker-build-push`](.github/actions/docker-build-push/README.md) — Builds and pushes Docker images with automatic registry prefix handling, multi-tag support, OCI metadata labels, and BuildKit caching.

### Artifacts
- [`artifact-from-image`](.github/actions/artifact-from-image/README.md) — Saves a Docker image to an artifact for later workflows.
- [`artifact-to-image`](.github/actions/artifact-to-image/README.md) — Downloads a Docker image artifact and loads it back into the Docker daemon.

## GitHub

### Allow GitHub Actions to create and approve pull requests
1. Navigate to repository Settings → Actions → General
2. Section: Workflow permissions
3. [x] Allow GitHub Actions to create and approve pull requests
