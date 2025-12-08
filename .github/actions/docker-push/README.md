# Push Docker Image Action

Composite action that loads a Docker image (either from the local daemon or a saved artifact), retags it if needed, logs into a registry, and performs `docker push`.

## Inputs
| Name | Description | Required | Default |
| --- | --- | --- | --- |
| `image` | Local Docker image reference to push. | No | — |
| `artifact` | Artifact handle (`name/path`) created by [`artifact-from-image`](../artifact-from-image). | No | — |
| `target` | Optional tag applied prior to pushing (useful for promoting digests). | No | — |
| `docker-username` | Explicit registry username. Falls back to parsing the target reference. | No | — |
| `docker-registry` | Registry domain (leave blank for Docker Hub). | No | — |
| `secret-docker-token` | Password or token passed to `docker/login-action`. | Yes | — |

> Either `image` or `artifact` must be supplied.

## Behavior
1. Optional artifact download via [`artifact-to-image`](../artifact-to-image) loads the Docker image into the runner.
2. The action validates local availability, infers a username (unless provided), and logs into the requested registry.
3. If `target` is set, it retags the image before pushing; if `docker-registry` is set, it further namespaces the tag (for example `ghcr.io/org/app:sha`).
4. Runs `docker push` and surfaces any failures immediately.

## Registry Notes (GHCR)
- Enable `Read and write` workflow permissions plus `packages: write` when pushing to `ghcr.io` with `GITHUB_TOKEN`.
- Organization repos must toggle **Settings → Actions → General → Workflow permissions** accordingly.
- Built images appear under `https://github.com/<user_or_org>?tab=packages`.

## Usage
```yaml
permissions:
  contents: read
  packages: write

jobs:
  push-docker-image:
    runs-on: ubuntu-latest
    steps:
      - name: Push Docker image
        uses: draftm0de/github.workflows/.github/actions/docker-push@main
        with:
          artifact: ${{ needs.build.outputs.artifact }}
          target: ghcr.io/draftm0de/app:${{ github.sha }}
          docker-registry: ghcr.io
          secret-docker-token: ${{ secrets.GITHUB_TOKEN }}
```

Use the `artifact` output from the build action/workflow to avoid rebuilding before push jobs.
