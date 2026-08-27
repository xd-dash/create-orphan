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
    branch: bootstrap
    append-random-suffix: "true"
```

This creates a branch such as `bootstrap-a1b2c3d4` and exposes the final name as the `branch` output.

## Push-file pseudo-dispatch example

An agent that can commit and push repository files, but cannot call the GitHub Actions dispatch API, can use a committed request file as a small input envelope.

For example, put this workflow in `.github/workflows/create-orphan-on-request.yml`:

```yaml
name: Create Orphan Branch From Request

on:
  push:
    paths:
      - ".github/requests/create-orphan.txt"

permissions:
  contents: write

jobs:
  create:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout request
        uses: actions/checkout@v6
        with:
          fetch-depth: 1

      - name: Parse request
        id: request
        shell: bash
        run: |
          set -euo pipefail

          request=.github/requests/create-orphan.txt
          branch="$(sed -n 's/^branch=//p' "$request" | tail -n1)"
          append="$(sed -n 's/^append_random_suffix=//p' "$request" | tail -n1)"

          [[ -n "$branch" ]] || {
            echo "::error::branch is required"
            exit 1
          }

          [[ -n "$append" ]] || append=true

          case "$append" in
            true|false) ;;
            *)
              echo "::error::append_random_suffix must be true or false"
              exit 1
              ;;
          esac

          git check-ref-format --branch "$branch"

          {
            echo "branch=$branch"
            echo "append=$append"
          } >> "$GITHUB_OUTPUT"

      - name: Create empty orphan branch
        id: orphan
        uses: xd-dash/create-orphan@main
        with:
          branch: ${{ steps.request.outputs.branch }}
          append-random-suffix: ${{ steps.request.outputs.append }}

      - run: echo "Created ${{ steps.orphan.outputs.branch }}"
```

The request file can be as small as:

```text
branch=bootstrap
append_random_suffix=true
```

With the random suffix enabled, the resulting branch will be something like `bootstrap-7f39b2a1` rather than requiring a fixed branch name such as `automation`.

That means an agent only needs ordinary Git write access to create a pseudo-dispatch job:

```bash
cat > .github/requests/create-orphan.txt <<'EOF'
branch=bootstrap
append_random_suffix=true
EOF

git add .github/requests/create-orphan.txt
git commit -m "request orphan branch"
git push
```

The push itself becomes the trigger, the request file becomes the workflow input, and this action performs the branch creation.

### Custom token

For cases where `GITHUB_TOKEN` is not suitable, pass another token explicitly:

```yaml
- uses: xd-dash/create-orphan@main
  with:
    branch: bootstrap
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
