# Version Reader Action

Composite action that reads version information from different sources. It supports reading versions from Node.js package.json files, extracting version patterns from branch names, or reading the latest version tag from a target branch. The action outputs both the full version (with optional postfix) and a clean semantic version without postfix.

## Inputs

| Name                | Description                                                                 | Required | Default |
|---------------------|-----------------------------------------------------------------------------|----------|---------|
| `type`               | Source type to read version from. Supported values: `nodejs`, `branch`, `target` | Yes      | -       |
| `source-branch-name` | Source branch name to extract version from (required when `type=branch`)   | No       | -       |
| `target-branch-name` | Target branch name to read latest tag from (required when `type=target`)   | No       | -       |

## Outputs

| Name             | Description |
|------------------|-------------|
| `version`        | Full version string with optional postfix (e.g., `v1.0.1+1`, `1.0.1`). |
| `version-short`  | Clean semantic version without postfix (e.g., `v1.0.1`, `1.0.1`). |

## Usage

### Reading from package.json

```yaml
steps:
  - uses: actions/checkout@v4

  - name: Read Node.js version
    id: version
    uses: draftm0de/github.workflows/.github/actions/version-reader@main
    with:
      type: nodejs

  - name: Use version
    run: |
      echo "Version: ${{ steps.version.outputs.version }}"
      echo "Version Short: ${{ steps.version.outputs.version-short }}"
```

### Extracting version from branch name

```yaml
steps:
  - name: Extract version from branch
    id: version
    uses: draftm0de/github.workflows/.github/actions/version-reader@main
    with:
      type: branch
      source-branch-name: v1.2.3

  - name: Use version
    run: |
      echo "Branch version: ${{ steps.version.outputs.version }}"
      echo "Short version: ${{ steps.version.outputs.version-short }}"
```

### Reading from target branch

```yaml
steps:
  - uses: actions/checkout@v4
    with:
      fetch-depth: 0  # Required to fetch all tags

  - name: Read latest tag from main
    id: version
    uses: draftm0de/github.workflows/.github/actions/version-reader@main
    with:
      type: target
      target-branch-name: main

  - name: Use version
    run: |
      echo "Latest version on main: ${{ steps.version.outputs.version }}"
      echo "Short version: ${{ steps.version.outputs.version-short }}"
```

### Using with git-state action

```yaml
steps:
  - name: Get Git state
    id: git_state
    uses: draftm0de/github.workflows/.github/actions/git-state@main

  - name: Extract version from source branch
    id: version
    uses: draftm0de/github.workflows/.github/actions/version-reader@main
    with:
      type: branch
      source-branch-name: ${{ steps.git_state.outputs.source-branch-name }}

  - name: Use version
    run: |
      echo "Branch contains version: ${{ steps.version.outputs.version }}"
```

## Behavior

### Type: nodejs

When `type: nodejs`:

1. The action searches for `package.json` in the repository root
2. Reads the `version` field from the JSON file using Node.js
3. Validates the version matches one of the supported patterns:
   - `v1.0.1` - semantic version with `v` prefix
   - `1.0.1` - plain semantic version
   - `v1.0.1+1` - semantic version with prefix and postfix
   - `1.0.1+1` - semantic version with postfix
4. Outputs:
   - `version`: Exact version string from package.json
   - `version-short`: Version with postfix stripped (if present)

**Error conditions:**
- If `package.json` is not found → exits with error
- If `package.json` has no `version` field → exits with error
- If version doesn't match supported patterns → exits with error

### Type: branch

When `type: branch`:

1. Validates that `source-branch-name` input is provided (required)
2. Attempts to extract a semantic version pattern from the branch name using regex
3. Searches for version patterns in the branch name:
   - `v1.0.1` - exact semantic version with `v` prefix
   - `1.0.1` - exact semantic version without prefix
   - `release/v1.0.1` - version with path prefix
   - `v1.0.1-feature` - version with suffix
   - `feature/v1.0.1/something` - version anywhere in the path
   - Supports optional postfix: `v1.0.1+1`, `1.0.1+build`
4. If only major.minor is found (e.g., `v1.0`), sets patch to `0` (becomes `v1.0.0`)
5. Extracts and validates the version matches supported patterns
6. Outputs:
   - `version`: Extracted version with original prefix and postfix
   - `version-short`: Version with postfix stripped (if present)

**Error conditions:**
- If `source-branch-name` input is not provided → exits with error
- If no version pattern found in branch name → exits with error
- If extracted version doesn't match semantic version pattern → exits with error

**Branch name examples:**
| Branch Name           | Extracted version | version-short | Notes |
|-----------------------|-------------------|---------------|-------|
| `v1.2.3`              | `v1.2.3`          | `v1.2.3`      | Exact match |
| `1.2.3`               | `1.2.3`           | `1.2.3`       | No prefix |
| `release/v1.2.3`      | `v1.2.3`          | `v1.2.3`      | With path prefix |
| `v1.2.3+build-10`     | `v1.2.3+build-10` | `v1.2.3`      | With postfix |
| `feature/v1.2.3-dev`  | `v1.2.3`          | `v1.2.3`      | Version extracted, suffix ignored |
| `v1.2`                | `v1.2.0`          | `v1.2.0`      | Patch defaulted to 0 |
| `release/1.5`         | `1.5.0`           | `1.5.0`       | Patch defaulted to 0 |
| `feature/new-feature` | ERROR             | ERROR         | No version pattern |
| `main`                | ERROR             | ERROR         | No version pattern |

### Type: target

When `type: target`:

1. Validates that `target-branch-name` input is provided (required)
2. Fetches all tags from the repository
3. Filters tags merged into `origin/<target-branch>`
4. Identifies valid semantic version tags with optional prefix and postfix:
   - `v1.0.1`, `1.0.1`, `v1.0.1+1`, `1.0.1+build-123`, etc.
5. Sorts tags by semantic version ordering (not chronological)
6. Selects the latest semantic version tag
7. If no tags found, defaults to `0.0.0`
8. Outputs:
   - `version`: Full tag name (e.g., `v1.2.3+10`)
   - `version-short`: Tag without postfix (e.g., `v1.2.3`)

**Error conditions:**
- If `target-branch-name` input is not provided → exits with error
- If target branch doesn't exist → emits warning and defaults to `0.0.0`

**Use case:**
This type is useful for feature branches without embedded version information. Instead of requiring a version in the branch name or package.json, the action reads the latest version from the target branch (e.g., `main`) and tag-builder can increment from there.

**Example:**
- Branch: `feature/new-feature` (no version anywhere)
- Target: `main` (latest tag: `v1.0.2`)
- Result: version-reader returns `v1.0.2`
- tag-builder with `patch: true` → builds `v1.0.3`

### Unsupported Types

When `type` is not `nodejs`, `branch`, or `target`:

1. The action emits an error message
2. Exits with status code 1

This ensures that workflows fail early with clear error messages when an invalid type is provided, preventing downstream actions from receiving invalid input.

### Version Pattern Validation

All versions must match one of these patterns:

```
v1.0.1         # Semantic version with 'v' prefix
1.0.1          # Plain semantic version
v1.0.1+1       # Semantic version with 'v' prefix and postfix
1.0.1+build    # Semantic version with postfix
v1.0           # Major.minor only (patch will be set to 0)
1.0            # Major.minor only (patch will be set to 0)
```

The regex pattern used: `^([vV]?)([0-9]+)\.([0-9]+)(?:\.([0-9]+))?(\+.+)?$`

Where:
- `([vV]?)` - Optional `v` or `V` prefix
- `([0-9]+)\.([0-9]+)` - Major.Minor numbers (required)
- `(?:\.([0-9]+))?` - Optional Patch number (defaults to 0 if missing)
- `(\+.+)?` - Optional postfix after `+` (build metadata)

### Postfix Handling

Postfixes (characters after `+`) are used to carry build metadata:
- **Preserved** in the `version` output
- **Stripped** in the `version-short` output

Examples:
| Source Version     | version output     | version-short output |
|--------------------|--------------------|---------------------|
| `v1.0.1+1`         | `v1.0.1+1`         | `v1.0.1`            |
| `1.2.3+build-123`  | `1.2.3+build-123`  | `1.2.3`             |
| `v2.0.0`           | `v2.0.0`           | `v2.0.0`            |
| `1.0.0`            | `1.0.0`            | `1.0.0`             |
| `v1.2`             | `v1.2.0`           | `v1.2.0`            |

### Example Scenarios

| Type     | Source                    | source-branch-name | target-branch-name | version output | version-short output | Notes |
|----------|---------------------------|--------------------|--------------------|--------------  |---------------------|-------|
| `nodejs` | package.json: `1.2.3`     | (ignored)          | (ignored)          | `1.2.3`        | `1.2.3`             | No prefix, no postfix |
| `nodejs` | package.json: `v1.2.3+1`  | (ignored)          | (ignored)          | `v1.2.3+1`     | `v1.2.3`            | With prefix and postfix |
| `branch` | -                         | `v1.5.0`           | (ignored)          | `v1.5.0`       | `v1.5.0`            | Exact version match |
| `branch` | -                         | `release/v1.5.0`   | (ignored)          | `v1.5.0`       | `v1.5.0`            | Extracted from path |
| `branch` | -                         | `v1.5`             | (ignored)          | `v1.5.0`       | `v1.5.0`            | Patch defaulted to 0 |
| `branch` | -                         | `feature/test`     | (ignored)          | ERROR          | ERROR               | No version pattern |
| `target` | Latest tag: `v1.5.0`      | (ignored)          | `main`             | `v1.5.0`       | `v1.5.0`            | Latest from main |
| `target` | Latest tag: `v1.5.0+10`   | (ignored)          | `main`             | `v1.5.0+10`    | `v1.5.0`            | Postfix preserved |
| `target` | No tags found             | (ignored)          | `develop`          | `0.0.0`        | `0.0.0`             | Default fallback |
| `nodejs` | package.json missing      | (ignored)          | (ignored)          | ERROR          | ERROR               | File not found |
| `branch` | -                         | (not provided)     | (ignored)          | ERROR          | ERROR               | Required input missing |
| `target` | -                         | (ignored)          | (not provided)     | ERROR          | ERROR               | Required input missing |
| `other`  | -                         | (any)              | (any)              | ERROR          | ERROR               | Unsupported type |

## Dependencies

This action requires:

### For type: nodejs
- `node` - Node.js runtime for reading package.json
- `package.json` - Must exist in repository root with valid `version` field
- Standard bash shell with regex support

### For type: branch
- Standard bash shell with regex support
- No external dependencies

### For type: target
- `git` - For fetching and filtering tags
- Standard bash shell with regex support
- Repository must be checked out with tags fetched (`fetch-depth: 0` or `git fetch --tags`)
- Access to the target branch (usually via prior checkout action)

### Common Requirements
- Bash shell with regex pattern matching support

## Use Cases

1. **Node.js projects**: Read version from package.json for tagging, releases, or Docker image tags
2. **Branch naming validation**: Ensure feature/release branches follow version naming conventions
3. **Version-based workflows**: Extract version from branch name to determine deployment targets
4. **Release automation**: Validate that release branches contain valid semantic versions
5. **PR validation**: Check that source branch names follow versioning standards
6. **Multi-environment deployments**: Route deployments based on version extracted from branch names
7. **Feature branches without versions**: Use target branch's latest version as base for patch increments

## Notes

- The action does **not** modify any files - it only reads version information
- For `type: branch`, the action is lenient in extracting versions from complex branch names
- For `type: target`, ensure repository is checked out with `fetch-depth: 0` to fetch all tags
- The `v` prefix in versions is preserved from the source (package.json, branch name, or git tags)
- Version comparison should be done on the `version-short` output to ignore build metadata
- When patch version is missing in branch names (e.g., `v1.2`), it defaults to `0` (becomes `v1.2.0`)
- Unsupported types cause the action to exit with error, ensuring workflows fail early with clear messages
- The `source-branch-name` and `target-branch-name` inputs can be combined with the git-state action's outputs for dynamic extraction
