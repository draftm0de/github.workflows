# Load Docker Image from Artifact Action

This GitHub Action enables loading a previously saved Docker image from an artifact file, facilitating the use of Docker images across different workflow steps or pipelines.

## Features

- Downloads a Docker image artifact from GitHub Actions storage
- Loads the image into Docker, making it available for use
- Outputs the loaded image reference for subsequent steps

## Inputs

| Name       | Description                                                                  | Required | Default |
|------------|------------------------------------------------------------------------------|----------|---------|
| `artifact` | Fully qualified artifact file path (e.g., `artifact-name/path/to/file.tar`). | Yes      |         |

## Outputs

| Name    | Description                |
|---------|----------------------------|
| `image` | Name of the loaded image.  |

## How It Works

1. Extracts the artifact name and file path from the provided input.
2. Downloads the specified artifact using `actions/download-artifact`.
3. Loads the Docker image from the artifact file into Docker using `docker load`.
4. Outputs the loaded image's name for subsequent use (for example `docker-push`).

## Usage

### Basic Example

```yaml
jobs:
  push:
    runs-on: ubuntu-latest
    steps:
      - name: Load Docker image from artifact
        id: download
        uses: draftm0de/github.workflows/.github/actions/artifact-to-image@main
        with:
          artifact: myrepo-myimage-1-0/image.tar

      - name: Use loaded image
        run: docker images ${{ steps.download.outputs.image }}
```

### With artifact-from-image

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      artifact: ${{ steps.upload.outputs.artifact }}
    steps:
      - name: Build image
        run: docker build -t myrepo/myimage:1.0 .

      - name: Upload to artifact
        id: upload
        uses: draftm0de/github.workflows/.github/actions/artifact-from-image@main
        with:
          image: myrepo/myimage:1.0

  push:
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Download from artifact
        id: download
        uses: draftm0de/github.workflows/.github/actions/artifact-to-image@main
        with:
          artifact: ${{ needs.build.outputs.artifact }}

      - name: Push to registry
        uses: draftm0de/github.workflows/.github/actions/docker-push@main
        with:
          image: ${{ steps.download.outputs.image }}
          tags: v1.0.0 latest
          registry: ghcr.io
          password: ${{ secrets.GITHUB_TOKEN }}
```

### In node-js-ci Workflow

```yaml
jobs:
  docker_build:
    outputs:
      artifact: ${{ steps.upload.outputs.artifact }}
    steps:
      - name: Build
        id: build
        uses: draftm0de/github.workflows/.github/actions/docker-build@main
        with:
          image: ghcr.io/org/app

      - name: Upload
        id: upload
        uses: draftm0de/github.workflows/.github/actions/artifact-from-image@main
        with:
          image: ${{ steps.build.outputs.image }}

  docker_push:
    needs: docker_build
    steps:
      - name: Download
        id: download
        uses: draftm0de/github.workflows/.github/actions/artifact-to-image@main
        with:
          artifact: ${{ needs.docker_build.outputs.artifact }}

      - name: Push
        uses: draftm0de/github.workflows/.github/actions/docker-push@main
        with:
          image: ${{ steps.download.outputs.image }}
          tags: v1.2.3 latest
```

## Input Format

The `artifact` input must be in `name/path` format:
- `name`: The artifact name (as uploaded by `actions/upload-artifact`)
- `path`: The file path within the artifact (typically `image.tar`)

Example: `myrepo-myimage-1-0/image.tar`

## Notes

- The artifact must have been uploaded in a previous job using `artifact-from-image` or `actions/upload-artifact`
- The action extracts the image reference from `docker load` output
- The action writes a summary to `$GITHUB_STEP_SUMMARY` showing the artifact and loaded image
- Artifacts are automatically cleaned up after workflow completion (configurable in GitHub settings)
