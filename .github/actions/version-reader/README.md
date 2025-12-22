# Version Reader Action

Reads version information from Node.js package.json, Flutter pubspec.yaml, or from the latest git tag on a branch.

## Inputs

| Name | Description | Required |
|------|-------------|----------|
| `type` | Source type: `nodejs`, `flutter`, or `branch` | Yes |
| `branch` | Branch name to read latest tag from (required when `type=branch`) | Yes |

## Outputs

| Name | Description |
|------|-------------|
| `version` | Full version with optional postfix (e.g., `v1.0.1+1`) |
| `version-short` | Version without postfix (e.g., `v1.0.1`) |
| `source-title` | Project title from package.json meta.title (fallback: name) or pubspec.yaml meta.title |
| `source-description` | Project description from package.json meta.description (fallback: description) or pubspec.yaml meta.description |

## Usage

### Reading from package.json

```yaml
- name: Read Node.js version
  id: version
  uses: draftm0de/github.workflows/.github/actions/version-reader@main
  with:
    type: nodejs
```

### Reading from pubspec.yaml

```yaml
- name: Read Flutter version
  id: version
  uses: draftm0de/github.workflows/.github/actions/version-reader@main
  with:
    type: flutter
```

### Reading from branch tags

```yaml
- uses: actions/checkout@v4
  with:
    fetch-depth: 0  # Required to fetch all tags

- name: Read latest tag from main
  id: version
  uses: draftm0de/github.workflows/.github/actions/version-reader@main
  with:
    type: branch
    branch: main
```

## How It Works

**Type: nodejs**
- Reads the `version` field from `package.json` in the repository root
- Reads `meta.title` as `source-title` output (fallback: `name`)
- Reads `meta.description` as `source-description` output (fallback: `description`)
- Returns the version as-is from the file

**Type: flutter**
- Reads the `version` field from `pubspec.yaml` in the repository root
- Reads `meta.title` as `source-title` output (if present)
- Reads `meta.description` as `source-description` output (if present)
- Returns the version as-is from the file

**Type: branch**
- Fetches all git tags from the specified branch
- Returns the latest semantic version tag
- Defaults to `0.0.0` if no tags found

Both types preserve the `v` prefix and handle postfixes (e.g., `+build`) by providing both full and short versions.

## Use Cases

- Read version from package.json or pubspec.yaml for tagging or releases
- Extract project metadata (title, description) for Docker labels or documentation
- Get the latest version from a branch for version bumping
- Works great with feature branches that don't have version information

## Notes

- For `type: branch`, use `fetch-depth: 0` to fetch all tags
- Supported version formats: `v1.0.1`, `1.0.1`, `v1.0.1+build`, `1.0.1+build`
- See [DEPLOYMENT.md](DEPLOYMENT.md) for detailed implementation logic
