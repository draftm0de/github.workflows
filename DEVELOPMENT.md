# Development Guide

This document outlines the development standards, conventions, and workflow for maintaining GitHub Actions and reusable workflows in this repository.

## Philosophy

**Documentation-First Development**: Every action and workflow must have comprehensive documentation that is both human-readable and detailed enough to recreate the implementation from scratch. READMEs serve as the source of truth for behavior and implementation.

## Documentation Standards

### README Structure

Every action and workflow MUST have a README following this structure:

1. **Title** - Clear name (e.g., `# Git State Action`, `# Node.js CI Workflow`)

2. **Introduction Paragraph** - Brief overview explaining:
   - What the action/workflow does
   - Why it exists
   - Key features or benefits

3. **Triggers** (workflows only) - How to invoke the workflow

4. **Inputs** - Table format:
   ```markdown
   | Name | Description | Required | Default |
   |------|-------------|----------|---------|
   ```

5. **Outputs** - Table format:
   ```markdown
   | Name | Description |
   |------|-------------|
   ```

6. **Usage** - Multiple examples showing:
   - Basic usage
   - Advanced scenarios
   - Integration with other actions
   - Real code snippets

7. **Behavior** - Detailed sections explaining:
   - Logic for each type/mode/scenario
   - Step-by-step process
   - Error conditions
   - Edge cases

8. **Example Scenarios** - Table showing inputs → outputs with notes

9. **Dependencies** - What's required:
   - Tools (git, gh, node, etc.)
   - Environment variables
   - Permissions
   - External actions

10. **Use Cases** - Real-world applications numbered list

11. **Notes** - Important reminders, caveats, best practices

### README Requirements

- **Human-readable**: Clear, concise language
- **Complete**: Anyone should be able to rebuild the action from the README alone
- **Accurate**: Must match implementation exactly
- **Updated**: When code changes, README MUST be updated immediately
- **Consistent**: Follow the same structure across all actions/workflows

### Documentation Examples

Reference these as templates:
- `.github/actions/git-state/README.md`
- `.github/actions/tag-builder/README.md`
- `.github/actions/version-reader/README.md`
- `.github/workflows/node-js-ci.md`

## Code Standards

### General Principles

1. **No Comments**: Code should be self-documenting. Only add comments if explicitly requested.

2. **Follow Conventions**:
   - Read existing implementations first
   - Match coding style, patterns, and structure
   - Use existing libraries and utilities

3. **Error Handling**:
   - Always validate inputs
   - Provide clear error messages using `::error::`
   - Include context in error messages
   - Fail fast with meaningful exit codes

4. **Logging**:
   - Use `::notice::` for successful operations
   - Use `::warning::` for non-fatal issues
   - Use `::error::` for failures
   - Always add job summaries via `GITHUB_STEP_SUMMARY`

### Naming Conventions

#### Actions and Workflows
- Use **kebab-case** for all names
- Be descriptive and clear
- Examples: `git-state`, `tag-builder`, `version-reader`, `node-js-ci`

#### Inputs and Outputs
- Use **kebab-case**
- Be consistent across related actions
- Examples: `source-branch-name`, `target-branch-name`, `version-short`

#### Consistency Rules
- If `git-state` outputs `source-branch-name`, use the same name in `version-reader` input
- Maintain naming consistency across the entire action ecosystem

### Action Implementation Pattern

```yaml
name: Action Name
description: Brief description

inputs:
  input-name:
    description: "Clear description"
    required: true/false
    default: ''

outputs:
  output-name:
    description: "Clear description"
    value: ${{ steps.step-id.outputs.output_name }}

runs:
  using: "composite"
  steps:
    - name: Descriptive step name
      id: step-id
      shell: bash
      run: |
        set -euo pipefail

        # Input validation
        # Core logic
        # Output generation
        # Job summary
```

### Error Handling Example

```bash
# Temporarily disable exit on error for API calls
set +e
RESULT=$(command_that_might_fail 2>&1)
STATUS=$?
set -e

if [[ $STATUS -ne 0 ]] || [[ -z "$RESULT" ]]; then
  echo "::warning::Operation failed, using fallback"
  RESULT="default_value"
fi
```

### Job Summary Pattern

```bash
{
  echo "### action-name"
  echo "- Input: $INPUT_VALUE"
  echo "- Output: $OUTPUT_VALUE"
} >> "$GITHUB_STEP_SUMMARY"
```

## Development Workflow

### 1. Planning Phase

- Understand the requirement
- Review existing similar implementations
- Check for related actions that might be affected
- Plan the integration points

### 2. README First

- Write or update the README completely
- Include all sections per structure above
- Add examples and scenarios
- Get conceptual clarity before coding

### 3. Implementation

- Read README to understand spec
- Implement action.yml following patterns
- Match README behavior exactly
- Test logic mentally against examples

### 4. Synchronization Check

- Verify README and action.yml match
- Check all inputs/outputs are documented
- Ensure examples would actually work
- Validate error conditions are covered

### 5. Integration

- Update dependent actions/workflows
- Update their READMEs if affected
- Test integration points
- Maintain consistency across components

## Action Ecosystem

### Current Architecture

```
git-state → version-reader → tag-builder
```

### Actions

#### git-state
- **Purpose**: Detect PR state and branch information
- **Outputs**: `is-pull-request`, `ci-open-pull-request`, `source-branch-name`, `target-branch-name`
- **Key Feature**: Prevents duplicate test runs

#### version-reader
- **Purpose**: Read version from different sources
- **Types**: `nodejs`, `branch`, `target`
- **Outputs**: `version`, `version-short`
- **Use Cases**: Get current version for tagging

#### tag-builder
- **Purpose**: Build and validate semantic version tags
- **Inputs**: `target-branch`, `current-version`, `patch`
- **Outputs**: `version`, `version-short`
- **Key Feature**: Prevents version drift

### Workflows

#### node-js-ci.yml
- **Status**: Partially implemented (PR validation complete)
- **Pending**: Push event jobs (build_on_push, tag_on_push)
- **Pattern**: Uses git-state → version-reader → tag-builder

## Version Management

### Version Types

1. **nodejs**: Read from `package.json`
   - Use for: Node.js projects with package.json
   - Required: package.json with version field

2. **branch**: Extract from branch name
   - Use for: Version-based branch names (release/v1.2.3)
   - Supports: Complex patterns, defaults patch to 0

3. **target**: Read latest from target branch
   - Use for: Feature branches without version
   - Behavior: Gets latest tag, defaults to 0.0.0

### Patch Mode

- `patch: true` - Auto-increment patch from latest tag
- `patch: false` - Use exact version (strict validation)

### Version Patterns

Supported formats:
- `v1.0.1` - With v prefix
- `1.0.1` - Without prefix
- `v1.0.1+1` - With postfix
- `1.0` - Major.minor only (patch defaults to 0)

## Best Practices

### DO

- ✅ Update README when changing code
- ✅ Validate all inputs
- ✅ Add error handling
- ✅ Use job summaries
- ✅ Follow naming conventions
- ✅ Be consistent across actions
- ✅ Test integration points
- ✅ Read existing code first

### DON'T

- ❌ Add comments (unless requested)
- ❌ Commit without updating README
- ❌ Assume libraries are available
- ❌ Use inconsistent naming
- ❌ Leave README out of sync
- ❌ Forget error handling
- ❌ Skip input validation

## Tools and Commands

### Running Actions Locally

Actions use composite steps, so they require GitHub Actions environment. Testing approaches:

1. **Mental verification**: Walk through logic against README examples
2. **Integration testing**: Use in actual workflows
3. **Example validation**: Ensure examples in README would work

### Useful Git Commands

```bash
# Check action usage in workflows
grep -r "uses:.*git-state" .github/workflows/

# List all actions
ls -la .github/actions/

# Find action references
git grep "draftm0de/github.workflows"
```

## Maintenance

### When Updating Actions

1. Read current README to understand existing behavior
2. Plan changes - what inputs/outputs/behavior changes?
3. Update README with new behavior
4. Update action.yml implementation
5. Check dependent workflows/actions
6. Update their READMEs if affected
7. Verify examples still work

### When Adding Actions

1. Review similar existing actions
2. Follow the same patterns
3. Create comprehensive README first
4. Implement action.yml
5. Update this DEVELOPMENT.md if needed
6. Consider integration with existing actions

## Getting Started for New Developers

1. Read this DEVELOPMENT.md
2. Study existing action READMEs (git-state, tag-builder, version-reader)
3. Examine corresponding action.yml files
4. Note the patterns and structure
5. Follow the workflow when making changes

## Questions or Issues?

When working with an AI assistant:
- Ask it to read this DEVELOPMENT.md first
- Reference specific sections when requesting changes
- Remind it to update READMEs when modifying code
- Ask for consistency checks across related components
