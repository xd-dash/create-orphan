# create-orphan

A small composite GitHub Action that creates and pushes an empty orphan branch in the repository running the workflow.

It uses the same sequence as the xd-dash automation workflows:

1. Validate the requested branch name with `git check-ref-format`.
2. Optionally append a random 8-character hexadecimal suffix.
3. Refuse to overwrite an existing remote branch.
4. Run `git switch --orphan`.
5. Remove inherited tracked and untracked workspace content.
6. Create an empty root commit.
7. Push the new branch to `origin`.

## Usage

The calling job must grant `contents: write` because an action cannot elevate the permissions of `GITHUB_TOKEN` itself.

```yaml
name: Create orphan branch

on:
  workflow_dispatch:
    inputs:
      branch:
        description: Branch to create
        required: true
        type: string

permissions:
  contents: write

jobs:
  create:
    runs-on: ubuntu-latest
    steps:
      - id: orphan
        uses: xd-dash/create-orphan@main
        with:
          branch: ${{ inputs.branch }}

      - run: echo "Created ${{ steps.orphan.outputs.branch }}"
```

No separate `actions/checkout` step is required; this action checks out the calling repository itself.

### Random suffix

```yaml
- id: orphan
  uses: xd-dash/create-orphan@main
  with:
    branch: automation
    append-random-suffix: "true"
```

This creates a branch such as `automation-a1b2c3d4` and exposes the final name as the `branch` output.

### Custom token

For cases where `GITHUB_TOKEN` is not suitable, pass another token explicitly:

```yaml
- uses: xd-dash/create-orphan@main
  with:
    branch: automation
    token: ${{ secrets.REPO_TOKEN }}
```

The token needs permission to push repository contents.

## Inputs

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `branch` | yes | — | Base branch name to create. |
| `append-random-suffix` | no | `false` | Append an 8-character hexadecimal suffix. |
| `token` | no | `github.token` | Token used for checkout and push. |
| `commit-message` | no | `Initialize orphan branch` | Empty root commit message. |

## Output

`branch` is the final branch name that was pushed.

## Behavior

The action intentionally fails if the target branch already exists. It never force-pushes or rewrites an existing branch.

Because producing a truly empty orphan branch requires cleaning the checked-out working tree, run this action in a dedicated job rather than after build steps whose generated files you still need.
