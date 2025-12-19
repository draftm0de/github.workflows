# Docker Tag Builder Action

Builds Docker image tags based on version, target branch, and latest version detection. Outputs comma-separated tags for Docker build/push operations.

## Inputs

| Name                | Description                                                  | Required | Default  |
|---------------------|--------------------------------------------------------------|----------|----------|
| `version`           | Version to tag (e.g., `v1.2.12`, `1.2.12`)                   | Yes      | -        |
| `is-latest-version` | Whether this is the latest version globally (`true`/`false`) | Yes      | -         |
| `tag-levels`        | Version levels to tag: `patch`, `minor`, `major`, `latest` (comma-separated) | No | `'patch'` |

## Outputs

| Name          | Description                                                       |
|---------------|-------------------------------------------------------------------|
| `docker-tags` | Comma-separated list of Docker tags (e.g., `v1.2.12,v1.2,latest`) |
| `exact-tag`   | Exact version tag (e.g., `v1.2.12`)                               |
| `latest-tag`  | Latest tag (`latest`), empty if not created                       |

## Usage

### With Tag Builder

```yaml
- name: Build next version
  id: tag_builder
  uses: draftm0de/github.workflows/.github/actions/tag-builder@main
  with:
    target-branch: v1.2
    current-version: v1.2.11
    patch: 'true'

- name: Build Docker tags
  id: docker_tags
  uses: draftm0de/github.workflows/.github/actions/docker-tag-builder@main
  with:
    version: ${{ steps.tag_builder.outputs.next-version-short }}
    is-latest-version: ${{ steps.tag_builder.outputs.is-latest-version }}
    tag-levels: 'patch,minor'

- name: Build and push Docker image
  run: |
    IFS=',' read -ra TAGS <<< "${{ steps.docker_tags.outputs.docker-tags }}"
    for tag in "${TAGS[@]}"; do
      docker tag myimage:build myimage:$tag
      docker push myimage:$tag
    done
```

### Main Branch Example (Full Tagging)

```yaml
- name: Build Docker tags
  id: docker_tags
  uses: draftm0de/github.workflows/.github/actions/docker-tag-builder@main
  with:
    version: v3.2.2
    is-latest-version: 'true'
    tag-levels: 'patch,minor,major,latest'

# Output: docker-tags="v3.2.2,v3.2,v3,latest"
```

### Maintenance Branch Example

```yaml
- name: Build Docker tags
  id: docker_tags
  uses: draftm0de/github.workflows/.github/actions/docker-tag-builder@main
  with:
    version: v1.2.12
    is-latest-version: 'false'
    tag-levels: 'patch,minor'

# Output: docker-tags="v1.2.12,v1.2"
```

## How It Works

**Tag Levels:**

Control which version tags to create using `tag-levels` (comma-separated):
- `patch`: Exact version (e.g., `v1.2.12`)
- `minor`: Minor version (e.g., `v1.2`)
- `major`: Major version (e.g., `v1`)
- `latest`: Latest tag (only if `is-latest-version: true`)

**Examples:**
- `tag-levels: 'patch'` → `v1.2.12`
- `tag-levels: 'patch,minor'` → `v1.2.12,v1.2`
- `tag-levels: 'patch,minor,major'` → `v1.2.12,v1.2,v1`
- `tag-levels: 'patch,minor,major,latest'` + is-latest: true → `v1.2.12,v1.2,v1,latest`

**Postfix Handling:**
- Postfixes (e.g., `+build`) are always stripped
- Example: Input `v1.2.12+build` → Output `v1.2.12`

## Example Scenarios

| Tag Levels | Version | Is Latest | Output Tags | Notes |
|------------|---------|-----------|-------------|-------|
| `patch` | `v1.2.12` | `false` | `v1.2.12` | Exact version only (default) |
| `patch,minor` | `v1.2.12` | `false` | `v1.2.12,v1.2` | Patch + minor |
| `patch,minor,major` | `v3.2.2` | `false` | `v3.2.2,v3.2,v3` | All version levels |
| `patch,minor,major,latest` | `v3.2.2` | `true` | `v3.2.2,v3.2,v3,latest` | Full tagging |
| `patch,latest` | `v3.2.2` | `true` | `v3.2.2,latest` | Simple latest tagging |
| `minor,major` | `v1.2.12` | `false` | `v1.2,v1` | Skip patch level |

## Notes

- Output is comma-separated for easy use with Docker tag commands
- Postfixes are stripped (Docker tags should be clean semver)
- Branch tags are "floating" - they move to the latest patch
- `:latest` tag only added when this is globally the newest version
- Invalid tag levels will cause action to fail with exit code 1
- See [DEPLOYMENT.md](DEPLOYMENT.md) for implementation details
