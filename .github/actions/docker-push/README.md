# Docker Push Action

Pushes a Docker image to a registry with multiple tags in a single operation.

## Inputs

| Name       | Description                                                            | Required | Default |
|------------|------------------------------------------------------------------------|----------|---------|
| `image`    | Local Docker image name to push                                        | Yes      | -       |
| `tags`     | Space-separated list of tags to push (e.g., `v1.2.12 v1.2 latest`)    | Yes      | -       |
| `registry` | Registry hostname (e.g., `ghcr.io`, leave blank for Docker Hub)        | No       | -       |
| `username` | Registry username (inferred from first tag if not provided)            | No       | -       |
| `password` | Registry password or token                                             | Yes      | -       |

## Outputs

| Name          | Description                                                |
|---------------|------------------------------------------------------------|
| `tags-pushed` | Space-separated list of fully-qualified tags that were pushed |

## Usage

### With Docker Tag Builder

```yaml
- name: Build Docker tags
  id: docker_tags
  uses: draftm0de/github.workflows/.github/actions/docker-tag-builder@main
  with:
    version: v1.2.12
    is-latest-version: 'true'
    tag-levels: 'patch,minor,major,latest'

- name: Push Docker image
  uses: draftm0de/github.workflows/.github/actions/docker-push@main
  with:
    image: myapp:build
    tags: ${{ steps.docker_tags.outputs.docker-tags }}
    registry: ghcr.io
    password: ${{ secrets.GITHUB_TOKEN }}
```

### Push to Docker Hub

```yaml
- name: Push to Docker Hub
  uses: draftm0de/github.workflows/.github/actions/docker-push@main
  with:
    image: myapp:build
    tags: myorg/myapp:v1.2.12 myorg/myapp:latest
    username: myorg
    password: ${{ secrets.DOCKER_HUB_TOKEN }}
```

### Push to GHCR

```yaml
permissions:
  contents: read
  packages: write

jobs:
  push:
    runs-on: ubuntu-latest
    steps:
      - name: Push to GitHub Container Registry
        uses: draftm0de/github.workflows/.github/actions/docker-push@main
        with:
          image: myapp:build
          tags: ghcr.io/myorg/myapp:v1.2.12 ghcr.io/myorg/myapp:latest
          registry: ghcr.io
          password: ${{ secrets.GITHUB_TOKEN }}
```

## How It Works

1. **Validation**: Checks that `image` exists locally and `tags` is provided
2. **Username inference**: If `username` not provided, extracts from first tag (e.g., `ghcr.io/myorg/app` → `myorg`)
3. **Registry login**: Uses `docker/login-action@v3` with provided credentials
4. **Tag and push**: For each tag in space-separated list:
   - Tag local image with `<registry>/<tag>` (or just `<tag>` for Docker Hub)
   - Push tagged image to registry
5. **Summary**: Writes list of all pushed tags to workflow step summary

## Registry Notes

**GitHub Container Registry (ghcr.io):**
- Requires `packages: write` permission
- Use `GITHUB_TOKEN` as password
- Username can be user or organization name
- Images appear at `https://github.com/<user_or_org>?tab=packages`

**Docker Hub:**
- Leave `registry` input blank
- Provide `username` explicitly (cannot be inferred for Docker Hub)
- Use Docker Hub access token as `password`

## Example Output

When pushing `myapp:build` with tags `v1.2.12 v1.2 latest` to `ghcr.io`:

```
Tags pushed:
- ghcr.io/myorg/myapp:v1.2.12
- ghcr.io/myorg/myapp:v1.2
- ghcr.io/myorg/myapp:latest
```

## Notes

- All tags must be provided in a space-separated list
- Registry prefix is automatically added to each tag
- Username inference works for registry-prefixed tags (e.g., `ghcr.io/user/app`, `docker.io/user/app`)
- For Docker Hub without registry prefix, provide `username` explicitly
- Action fails if image doesn't exist locally or if any push fails
- See [DEPLOYMENT.md](DEPLOYMENT.md) for implementation details
