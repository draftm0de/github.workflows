# Load Docker Image from Artifact Action

This GitHub Action enables loading a previously saved Docker image from an artifact file, facilitating the use of Docker images across different workflow steps or pipelines.

## Features

- Downloads a Docker image artifact from GitHub Actions storage.
- Loads the image into Docker, making it available for use.

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

Below is an example of how to use this action in your workflow:

```yaml
jobs:
  load-docker-image:
    runs-on: ubuntu-latest
    steps:
      - name: Load Docker Image
        uses: draftm0de/github.workflows/.github/actions/artifact-to-image@main
        with:
          artifact: myartifact/image.tar
```
