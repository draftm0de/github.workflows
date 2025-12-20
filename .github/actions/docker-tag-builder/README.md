# Docker Tag Builder Action

Builds Docker image tags based on version and latest version detection. Supports semantic version levels (patch, minor, major, latest) and custom tags. Outputs space-separated tags for Docker build/push operations.

## Inputs

| Name                | Description                                                  | Required | Default  |
|---------------------|--------------------------------------------------------------|----------|----------|
| `version`           | Version to tag (e.g., `v1.2.12`, `1.2.12`). Required for `major`/`minor`/`patch` levels. | No       | -        |
| `is-latest-version` | Whether this is the latest version globally (`true`/`false`) | Yes      | -         |
| `tag-levels`        | Tag levels: `patch`, `minor`, `major`, `latest`, or custom tags (e.g., `sha`, `edge`, `beta`). Comma-separated. | No | `'patch'` |
| `git-tag-levels`    | Git tag levels (optional). When provided with `ci-tag-source: branch`, validates that semantic docker tag levels (`patch`, `minor`, `major`) are subset of git tag levels. | No | - |
| `ci-tag-source`     | Version source type (`nodejs`, `flutter`, `branch`). Validation only enforced when set to `branch`. | No | - |

## Outputs

| Name          | Description                                                       |
|---------------|-------------------------------------------------------------------|
| `docker-tags` | Space-separated list of Docker tags (e.g., `v1.2.12 v1.2 latest`) |
| `exact-tag`   | Exact version tag (e.g., `v1.2.12`)                               |
| `latest-tag`  | Latest tag (`latest`), empty if not created                       |

## Usage

### With Tag Builder and Git Tag Validation

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
    git-tag-levels: 'patch,minor,major'

- name: Build git tags
  id: git_tags
  uses: draftm0de/github.workflows/.github/actions/git-tag-builder@main
  with:
    version: ${{ steps.tag_builder.outputs.next-version-short }}
    target-branch: v1.2
    git-tag-levels: 'patch,minor,major'

- name: Build and push Docker image
  run: |
    for tag in ${{ steps.docker_tags.outputs.docker-tags }}; do
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

# Output: docker-tags="v3.2.2 v3.2 v3 latest"
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

# Output: docker-tags="v1.2.12 v1.2"
```

## How It Works

**Tag Levels:**

Control which tags to create using `tag-levels` (comma-separated):

**Semantic version levels** (require `version` input):
- `patch`: Exact version (e.g., `v1.2.12`)
- `minor`: Minor version (e.g., `v1.2`)
- `major`: Major version (e.g., `v1`)

**Special levels:**
- `latest`: Latest tag (only if `is-latest-version: true`)

**Custom tags** (no `version` required):
- Any other string (e.g., `sha`, `edge`, `beta`, `pr-123`)
- Passed through as-is

**Examples:**
- `tag-levels: 'patch'` + version: `v1.2.12` → `v1.2.12`
- `tag-levels: 'patch,minor'` + version: `v1.2.12` → `v1.2.12 v1.2`
- `tag-levels: 'sha,edge'` (no version) → `sha edge`
- `tag-levels: 'patch,latest'` + version: `v1.2.12` + is-latest: true → `v1.2.12 latest`
- `tag-levels: 'beta,sha'` (no version) → `beta sha`

**Version Handling:**
- Version is optional unless using `major`, `minor`, or `patch` levels
- Postfixes (e.g., `+build`) are always stripped for semantic tags
- Example: Input `v1.2.12+build` → Output `v1.2.12`

### Custom Tags Example

```yaml
- name: Build custom Docker tags
  id: docker_tags
  uses: draftm0de/github.workflows/.github/actions/docker-tag-builder@main
  with:
    is-latest-version: 'false'
    tag-levels: 'sha,edge,beta'

# Output: docker-tags="sha edge beta"
# No version required for custom tags
```

### Mixed Semantic and Custom Tags

```yaml
- name: Build mixed Docker tags
  id: docker_tags
  uses: draftm0de/github.workflows/.github/actions/docker-tag-builder@main
  with:
    version: v1.2.12
    is-latest-version: 'true'
    tag-levels: 'patch,latest,edge'

# Output: docker-tags="v1.2.12 latest edge"
```

## Example Scenarios

| Tag Levels | Version | Is Latest | Output Tags | Notes |
|------------|---------|-----------|-------------|-------|
| `patch` | `v1.2.12` | `false` | `v1.2.12` | Exact version only (default) |
| `patch,minor` | `v1.2.12` | `false` | `v1.2.12 v1.2` | Patch + minor |
| `patch,minor,major` | `v3.2.2` | `false` | `v3.2.2 v3.2 v3` | All version levels |
| `patch,minor,major,latest` | `v3.2.2` | `true` | `v3.2.2 v3.2 v3 latest` | Full tagging |
| `patch,latest` | `v3.2.2` | `true` | `v3.2.2 latest` | Simple latest tagging |
| `minor,major` | `v1.2.12` | `false` | `v1.2 v1` | Skip patch level |
| `sha,edge` | (none) | `false` | `sha edge` | Custom tags only |
| `patch,sha` | `v1.2.12` | `false` | `v1.2.12 sha` | Mixed semantic + custom |

## Tag Level Validation

When `git-tag-levels` is provided **AND** `ci-tag-source: 'branch'`, the action validates that semantic docker tag levels are tracked in git:

**Rule**: Docker semantic tag levels (`patch`, `minor`, `major`) must be a subset of git tag levels.

**Examples:**
- `tag-levels: 'latest,major,minor'` + `git-tag-levels: 'major,minor'` + `ci-tag-source: 'branch'` ✅
- `tag-levels: 'patch,latest'` + `git-tag-levels: 'patch'` + `ci-tag-source: 'branch'` ✅
- `tag-levels: 'minor'` + `git-tag-levels: 'patch'` + `ci-tag-source: 'branch'` ❌ (git doesn't include `minor`)
- `tag-levels: 'latest,sha,edge'` + `git-tag-levels: ''` ✅ (no semantic docker tags)
- `tag-levels: 'patch'` + `git-tag-levels: 'patch'` + `ci-tag-source: 'nodejs'` ✅ (validation skipped for nodejs/flutter)

**Why**: Ensures git repository tracks at least the same version granularity as Docker registry when using branch-based versioning, preventing version drift.

**Note**:
- Custom tags (`sha`, `edge`, `beta`, `latest`) are ignored in validation
- Validation only runs for `ci-tag-source: 'branch'` - skipped for `nodejs` and `flutter` sources

## Notes

- Output is space-separated for easy use with Docker tag commands
- Postfixes are stripped from semantic version tags (Docker tags should be clean semver)
- Branch tags are "floating" - they move to the latest patch
- `:latest` tag only added when this is globally the newest version
- `version` is required only for `major`, `minor`, `patch` levels
- Custom tag levels (non-semantic) are passed through as-is
- Action fails if `major`/`minor`/`patch` requested without `version`
- `git-tag-levels` is optional - when omitted, no validation is performed
- See [DEPLOYMENT.md](DEPLOYMENT.md) for implementation details
