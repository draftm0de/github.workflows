# Save Docker Image to Artifact Action

This GitHub Action saves a specified Docker image as a `.tar` file and uploads it as an artifact, enabling seamless image storage and retrieval across workflows.

## Features

- Exports Docker images as `.tar` files.
- Automatically generates artifact names based on the Docker image name and tag.
- Uploads the saved image as an artifact for later use.

## Inputs

| Name    | Description                                                                          | Required | Default |
|---------|--------------------------------------------------------------------------------------|----------|---------|
| `image` | Docker image name (e.g., `repo/image:tag`). If no tag is provided, `latest` is used. | Yes      |         |

## Outputs

| Name       | Description                                      |
|------------|--------------------------------------------------|
| `artifact` | Reference to the uploaded artifact (`name/path`) |

## How It Works

1. Extracts the image name and tag from the input and prepares an artifact name.
2. Saves the specified Docker image as a `.tar` file using `docker save`.
3. Uploads the `.tar` file as an artifact using `actions/upload-artifact`.

Pair this action with [`artifact-to-image`](../artifact-to-image) or the `docker-push` composite to move the built image across workflow boundaries.

## Usage

Below is an example of how to use this action in your workflow:

```
jobs:
  save-docker-image:
    runs-on: ubuntu-latest
    steps:
      - name: Save and Upload Docker Image
        uses: ./
        with:
          image: myrepo/myimage:1.0
```
