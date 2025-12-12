# Save Docker Image to Artifact Action

This GitHub Action saves a specified Docker image as a `.tar` file and uploads it as an artifact, enabling seamless image storage and retrieval across workflows.

## Features

- Exports Docker images as `.tar` files.
- Automatically generates deterministic artifact names from the image name/tag.
- Uploads the saved image as an artifact for later jobs.
- Appends a short summary to the workflow run so consumers can quickly see the artifact that was produced.

## Inputs

| Name    | Description                                                                          | Required | Default |
|---------|--------------------------------------------------------------------------------------|----------|---------|
| `image` | Docker image name (e.g., `repo/image:tag`). If no tag is provided, `latest` is used. | Yes      |         |

## Outputs

| Name       | Description                                      |
|------------|--------------------------------------------------|
| `artifact` | Reference to the uploaded artifact (`name/path`) |

## How It Works

1. Extracts the image name and tag from the input and prepares an artifact name/path pair.
2. Saves the specified Docker image as a `.tar` file using `docker save`.
3. Uploads the `.tar` file as an artifact with `actions/upload-artifact` and writes a short summary section.

Pair this action with [`artifact-to-image`](../artifact-to-image) or the `docker-push` composite to move the built image across workflow boundaries.

## Usage

Below is an example of how to use this action in your workflow:

```yaml
jobs:
  save-docker-image:
    runs-on: ubuntu-latest
    steps:
      - name: Save and upload Docker image
        uses: draftm0de/github.workflows/.github/actions/artifact-from-image@main
        with:
          image: myrepo/myimage:1.0
```

The resulting artifact name is exposed via the `artifact` output (in `name/path` format) and echoed inside the workflow summary, making it easy to pass to [`artifact-to-image`](../artifact-to-image) or other publishing steps.
