# Version Reader - Deployment Guide

This document provides the pseudo-code logic for the version-reader action to allow reconstruction.

## Action Structure

### Inputs
- `type`: Required. Values: `nodejs` | `flutter` | `branch`
- `branch`: Required when type=branch

### Outputs
- `version`: Full version with optional postfix (e.g., `v1.0.1+1`)
- `version-short`: Version without postfix (e.g., `v1.0.1`)
- `source-title`: Project title from package.json name field (nodejs only)
- `source-description`: Project description from package.json description field (nodejs only)

## Implementation Logic

### Main Flow

```
IF type == "nodejs" THEN
    CALL read_nodejs_version()
ELSIF type == "flutter" THEN
    CALL read_flutter_version()
ELSIF type == "branch" THEN
    CALL read_branch_version()
ELSE
    ERROR "Unsupported type: {type}"
    SUPPORTED_TYPES = ["nodejs", "flutter", "branch"]
    EXIT 1
END IF

CALL validate_version_pattern(VERSION)
CALL extract_version_short(VERSION)
CALL output_results(VERSION, VERSION_SHORT)
CALL write_step_summary(TYPE, VERSION, VERSION_SHORT, BRANCH)
```

### read_nodejs_version()

```
FUNCTION read_nodejs_version():
    IF NOT exists("package.json") THEN
        ERROR "package.json not found in repository root"
        EXIT 1
    END IF

    VERSION = jq -r '.version // empty' package.json

    IF VERSION is empty THEN
        ERROR "No version field found in package.json"
        EXIT 1
    END IF

    TITLE = jq -r '.meta.title // .name // empty' package.json
    DESCRIPTION = jq -r '.meta.description // .description // empty' package.json

    NOTICE "Read version from package.json: {VERSION}"

    IF TITLE is not empty THEN
        OUTPUT "title={TITLE}"
        NOTICE "Read title from package.json: {TITLE}"
    END IF

    IF DESCRIPTION is not empty THEN
        OUTPUT "description={DESCRIPTION}"
        NOTICE "Read description from package.json: {DESCRIPTION}"
    END IF

    RETURN VERSION
END FUNCTION
```

### read_flutter_version()

```
FUNCTION read_flutter_version():
    IF NOT exists("pubspec.yaml") THEN
        ERROR "pubspec.yaml not found in repository root"
        EXIT 1
    END IF

    # Extract version field using grep
    VERSION = grep -E "^version:" pubspec.yaml
              | sed 's/version:[[:space:]]*//'
              | tr -d '\r'
              | xargs

    IF VERSION is empty THEN
        ERROR "No version field found in pubspec.yaml"
        EXIT 1
    END IF

    TITLE = yq eval '.meta.title // ""' pubspec.yaml
    DESCRIPTION = yq eval '.meta.description // ""' pubspec.yaml

    NOTICE "Read version from pubspec.yaml: {VERSION}"

    IF TITLE is not empty THEN
        OUTPUT "source-title={TITLE}"
        NOTICE "Read title from pubspec.yaml: {TITLE}"
    END IF

    IF DESCRIPTION is not empty THEN
        OUTPUT "source-description={DESCRIPTION}"
        NOTICE "Read description from pubspec.yaml: {DESCRIPTION}"
    END IF

    RETURN VERSION
END FUNCTION
```

### read_branch_version()

```
FUNCTION read_branch_version():
    BRANCH = input.branch

    IF BRANCH is empty THEN
        ERROR "branch input is required when type=branch"
        EXIT 1
    END IF

    NOTICE "Reading latest version tag from branch"
    NOTICE "Branch: {BRANCH}"

    # Fetch tags quietly
    git fetch --tags --force (suppress output)

    LATEST_TAG = ""

    IF git rev-parse "origin/{BRANCH}" exists THEN
        RAW_TAGS = git tag --merged "origin/{BRANCH}"

        SEMVER_TAGS = []
        FOR EACH TAG in RAW_TAGS DO
            IF TAG is empty THEN CONTINUE

            # Match semver pattern: ^([vV]?)([0-9]+)\.([0-9]+)\.([0-9]+)(\+.+)?$
            IF TAG matches semver pattern THEN
                EXTRACT TAG_PREFIX, TAG_MAJOR, TAG_MINOR, TAG_PATCH
                TAG_BASE = "{TAG_MAJOR}.{TAG_MINOR}.{TAG_PATCH}"
                ADD TAG_BASE to SEMVER_TAGS
            END IF
        END FOR

        IF SEMVER_TAGS is not empty THEN
            # Sort by version (semantic, not lexical)
            LATEST_BASE = sort SEMVER_TAGS by version | get last

            # Find original tag with this base version
            LATEST_TAG = find first tag matching "^[vV]?{LATEST_BASE}"
        END IF
    ELSE
        WARNING "Branch origin/{BRANCH} not found"
    END IF

    IF LATEST_TAG is empty THEN
        VERSION = "0.0.0"
        NOTICE "No tags found on branch, using default: {VERSION}"
    ELSE
        VERSION = LATEST_TAG
        NOTICE "Latest tag from branch: {VERSION}"
    END IF

    RETURN VERSION
END FUNCTION
```

### validate_version_pattern()

```
FUNCTION validate_version_pattern(VERSION):
    # Pattern: ^([vV]?)([0-9]+)\.([0-9]+)\.([0-9]+)(\+.+)?$
    IF NOT VERSION matches pattern THEN
        ERROR "Version does not match supported pattern: {VERSION}"
        ERROR "Expected format: [v]X.Y.Z[+postfix]"
        EXIT 1
    END IF
END FUNCTION
```

### extract_version_short()

```
FUNCTION extract_version_short(VERSION):
    # Pattern: ^([vV]?)([0-9]+\.[0-9]+\.[0-9]+)(\+.+)?$
    IF VERSION matches pattern THEN
        VERSION_SHORT = PREFIX + BASE_VERSION  # without postfix
    ELSE
        VERSION_SHORT = VERSION
    END IF

    RETURN VERSION_SHORT
END FUNCTION
```

### output_results()

```
FUNCTION output_results(VERSION, VERSION_SHORT):
    WRITE to GITHUB_OUTPUT:
        "version={VERSION}"
        "version_short={VERSION_SHORT}"

    NOTICE "Version: {VERSION}"
    NOTICE "Version Short: {VERSION_SHORT}"
END FUNCTION
```

### write_step_summary()

```
FUNCTION write_step_summary(TYPE, VERSION, VERSION_SHORT, BRANCH):
    APPEND to GITHUB_STEP_SUMMARY:
        "### version-reader"
        "- Type: {TYPE}"

        IF TYPE == "target" THEN
            "- Branch: {BRANCH}"
        END IF

        "- Version: {VERSION}"
        "- Version Short: {VERSION_SHORT}"
END FUNCTION
```

## Version Pattern Details

### Supported Patterns

- `v1.0.1` - Semantic version with 'v' prefix
- `1.0.1` - Plain semantic version
- `v1.0.1+1` - Semantic version with 'v' prefix and postfix
- `1.0.1+build` - Semantic version with postfix

### Regex Pattern

```
^([vV]?)([0-9]+)\.([0-9]+)\.([0-9]+)(\+.+)?$
```

**Groups:**
1. `([vV]?)` - Optional 'v' or 'V' prefix
2. `([0-9]+)` - Major version number
3. `([0-9]+)` - Minor version number
4. `([0-9]+)` - Patch version number
5. `(\+.+)?` - Optional postfix (build metadata)

## Error Handling

### Type: nodejs
- `package.json` not found → EXIT 1
- No `version` field in package.json → EXIT 1
- Version doesn't match pattern → EXIT 1

### Type: flutter
- `pubspec.yaml` not found → EXIT 1
- No `version` field in pubspec.yaml → EXIT 1
- Version doesn't match pattern → EXIT 1

### Type: target
- `branch` not provided → EXIT 1
- Branch not found → WARNING, default to `0.0.0`
- No valid tags found → default to `0.0.0`

### General
- Unsupported type → EXIT 1
- Invalid version pattern → EXIT 1

## Dependencies

### Type: nodejs
- Node.js runtime
- Bash shell with regex support

### Type: flutter
- Bash shell with grep, sed, xargs
- pubspec.yaml file

### Type: target
- Git CLI
- Bash shell with regex support
- Repository with fetched tags (fetch-depth: 0)

## Output Examples

| Input Type | Source                   | Output version | Output version-short |
|------------|--------------------------|----------------|----------------------|
| nodejs     | package.json: `1.2.3`    | `1.2.3`        | `1.2.3`              |
| nodejs     | package.json: `v1.2.3+1` | `v1.2.3+1`     | `v1.2.3`             |
| flutter    | pubspec.yaml: `1.2.3`    | `1.2.3`        | `1.2.3`              |
| flutter    | pubspec.yaml: `v1.2.3+1` | `v1.2.3+1`     | `v1.2.3`             |
| flutter    | + meta.title: "App"      | (also outputs source-title)           ||
| flutter    | + meta.description: "X"  | (also outputs source-description)     ||
| target     | Latest tag: `v1.5.0`     | `v1.5.0`       | `v1.5.0`             |
| target     | Latest tag: `v1.5.0+10`  | `v1.5.0+10`    | `v1.5.0`             |
| target     | No tags found            | `0.0.0`        | `0.0.0`              |
