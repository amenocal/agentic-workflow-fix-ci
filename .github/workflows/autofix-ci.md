---
description: Reproduce a failing CI run on a pull request branch and open a stacked fix PR.
engine: copilot
on:
  workflow_dispatch:
    inputs:
      branch:
        description: "Failing branch to reproduce and fix"
        required: true
        type: string
  workflow_run:
    workflows: ["CI"]
    types: [completed]
if: >
  github.event_name == 'workflow_dispatch' ||
  (github.event_name == 'workflow_run' && github.event.workflow_run.conclusion == 'failure')
permissions:
  contents: read
network: defaults
steps:
  # Branch names are attacker-controllable, so gh-aw blocks the
  # github.event.workflow_run.head_branch expression. Read it from the event
  # payload file instead and expose it as a step output for checkout.
  - name: Resolve failing branch
    id: resolve-branch
    env:
      DISPATCH_BRANCH: ${{ github.event.inputs.branch }}
    run: |
      set -euo pipefail
      BRANCH=""
      if [ -n "${GITHUB_EVENT_PATH:-}" ] && [ -f "$GITHUB_EVENT_PATH" ]; then
        BRANCH="$(jq -r '.workflow_run.head_branch // empty' "$GITHUB_EVENT_PATH")"
      fi
      if [ -z "$BRANCH" ]; then
        BRANCH="${DISPATCH_BRANCH:-}"
      fi
      if [ -z "$BRANCH" ]; then
        echo "Unable to determine the failing branch." >&2
        exit 1
      fi
      echo "branch=$BRANCH" >> "$GITHUB_OUTPUT"
  - name: Checkout failing branch
    uses: actions/checkout@v7
    with:
      fetch-depth: 0
      persist-credentials: false
      ref: ${{ steps.resolve-branch.outputs.branch }}
safe-outputs:
  create-pull-request:
    draft: false
    preserve-branch-name: true
    allowed-base-branches:
      - "*"
      - "!main"
      - "!master"
    # Only source files may land in the PR; test edits are stripped from the
    # patch so the agent cannot "fix" CI by rewriting or deleting tests.
    allowed-files:
      - "app/src/**"
    excluded-files:
      - "app/test/**"
---

# Autofix CI

A CI run failed on a pull request branch. Reproduce the failure and open a minimal,
stacked fix pull request against the failing branch.

## Context

- The runner has already checked out the failing branch for you, so the buggy code is
  in your workspace. Determine the failing branch name with `git branch --show-current`.
  On automatic runs this is the CI `workflow_run` head branch; on manual runs it is the
  `workflow_dispatch` branch input. Confirm with `git status`; do not fetch or switch
  branches yourself.

## Task

1. Read `notes/ci-fix-guide.md` before changing any code.
2. Determine the failing branch name with `git branch --show-current` (the runner already
   checked it out from the `workflow_run` head branch or the `workflow_dispatch` branch input).
3. Reproduce the failure by running `cd app && npm test`.
4. Read the failing assertion carefully. Find the smallest source change needed in
   `app/src/domain.mjs` to restore the intended behavior. Do not make unrelated changes
   or touch generated files.
5. **Only edit files under `app/src/`.** Never modify, delete, or weaken tests under
   `app/test/` to make CI pass — that is not a valid fix. The pull request is restricted
   to `app/src/**` and any `app/test/**` changes are automatically excluded from the patch,
   so a test edit will silently disappear and cannot land.
6. Re-run `cd app && npm test` to confirm the fix passes.

## Safe Outputs

- Open the fix as a stacked pull request whose **base is the failing branch, never `main`**,
  using the `create-pull-request` safe output with `base` set to the failing branch
  (permitted by `allowed-base-branches`).
- Name the fix branch `<failing-branch>-fix`.
- Keep the pull request description friendly and mention the `cd app && npm test` command you ran.
- If no source change is needed, call `noop` with a short explanation instead of opening a PR.
