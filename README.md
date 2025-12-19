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
- [`git-tag-builder`](.github/actions/git-tag-builder/README.md) — Creates git tags for semantic versions, filtering out levels covered by version-like branches.
- [`docker-tag-builder`](.github/actions/docker-tag-builder/README.md) — Builds Docker image tags from semantic versions or custom tags (sha, edge, beta). Supports patch/minor/major levels and latest tag detection.

### Docker
- [`docker-build`](.github/actions/docker-build/README.md) — Builds Docker images with optional Buildx reproducibility. Defaults to short git SHA tag when no tag provided.
- [`docker-push`](.github/actions/docker-push/README.md) — Pushes Docker images to registries with username inference from image names, automatic tag stripping, and multi-tag support.

### Artifacts
- [`artifact-from-image`](.github/actions/artifact-from-image/README.md) — Saves a Docker image to an artifact for later workflows.
- [`artifact-to-image`](.github/actions/artifact-to-image/README.md) — Downloads a Docker image artifact and loads it back into the Docker daemon.
