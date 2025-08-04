# Extract Called Workflow Ref

A GitHub Action that extracts the ref (branch/tag/SHA) used to call a reusable workflow or action. This solves a common problem where reusable workflows need to know which version of themselves was called.

## The Problem

When a reusable workflow is called, GitHub's context variables (`github.ref`, `github.sha`, etc.) refer to the **caller's** repository, not the called workflow. This makes it difficult for reusable workflows to checkout their own code at the correct version.

For example:
```yaml
# In caller workflow
uses: owner/repo/.github/workflows/reusable.yml@v1.2.3
```

Inside `reusable.yml`, there's no built-in way to know it was called with `@v1.2.3`.

## The Solution

This action parses the caller's workflow file to extract the ref used in the `uses:` statement.

## Usage

### Basic Example

```yaml
- name: Extract ref
  id: extract-ref
  uses: Open-Athena/extract-called-ref@v1
  with:
    target_repository: owner/repo

- name: Checkout at extracted ref
  uses: actions/checkout@v4
  with:
    repository: owner/repo
    ref: ${{ steps.extract-ref.outputs.ref }}
```

### Complete Example in a Reusable Workflow

```yaml
name: Reusable Workflow
on:
  workflow_call:
    inputs:
      ref_override:
        description: 'Override the automatic ref detection'
        required: false
        type: string

jobs:
  setup:
    runs-on: ubuntu-latest
    steps:
      # Use explicit ref if provided, otherwise extract it
      - name: Determine ref
        id: determine-ref
        run: |
          if [ -n "${{ inputs.ref_override }}" ]; then
            echo "ref=${{ inputs.ref_override }}" >> $GITHUB_OUTPUT
          else
            echo "needs_extraction=true" >> $GITHUB_OUTPUT
          fi

      - name: Extract ref from caller
        if: steps.determine-ref.outputs.needs_extraction == 'true'
        id: extract-ref
        uses: Open-Athena/extract-called-ref@v1
        with:
          target_repository: ${{ github.repository }}
          default_ref: main

      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          ref: ${{ steps.determine-ref.outputs.ref || steps.extract-ref.outputs.ref }}
```

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `target_repository` | The repository to search for (e.g., `owner/repo`) | Yes | - |
| `default_ref` | Ref to use if extraction fails | No | Target repo's default branch |
| `fail_on_multiple_refs` | Whether to fail if multiple different refs are found | No | `true` |

## Outputs

| Output | Description | Example |
|--------|-------------|---------|
| `ref` | The extracted ref or default | `v1.2.3`, `main`, `abc123` |
| `extraction_method` | How the ref was determined | `extracted`, `default`, `error` |

## How It Works

1. Extracts the workflow filename from `github.workflow_ref`
2. Fetches the caller's workflow file using the GitHub API
3. Parses the YAML to find `uses:` statements matching the target repository
4. Extracts the ref portion after the `@` symbol
5. If extraction fails and no `default_ref` is provided, fetches the target repository's default branch
6. Returns the extracted ref or falls back to the default

## Limitations

- Requires `GH_TOKEN` or `GITHUB_TOKEN` to fetch workflow files
- Cannot extract refs from private repositories unless the token has access
- If multiple jobs use different refs of the same repository, it will fail by default (configurable)
- Only works within GitHub Actions workflow context

## Examples

### Action in Public Repository

```yaml
# Caller workflow
- uses: actions/setup-node@v4

# In extract-called-ref
- uses: Open-Athena/extract-called-ref@v1
  with:
    target_repository: actions/setup-node
# Output: ref=v4
```

### Reusable Workflow

```yaml
# Caller workflow
jobs:
  call-workflow:
    uses: owner/repo/.github/workflows/build.yml@feature/new-stuff

# In extract-called-ref
- uses: Open-Athena/extract-called-ref@v1
  with:
    target_repository: owner/repo
# Output: ref=feature/new-stuff
```

### With Default Fallback

```yaml
- uses: Open-Athena/extract-called-ref@v1
  with:
    target_repository: owner/repo
    default_ref: develop
    fail_on_multiple_refs: false
# Output: ref=develop (if extraction fails or multiple refs found)
```

## License

MIT