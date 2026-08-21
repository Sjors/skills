---
name: rebase
description: Rebase a Git branch onto its intended base and resolve conflicts safely. Use for ordinary or stacked rebases; use the interactive-rebase skill when the request is to split, squash, reorder, or reword commits.
---

# Rebase

Move a branch onto its intended upstream history with minimal patch drift and
clear verification. Follow repository-local instructions for building and
testing; this skill is used across projects with different build systems.

## Invariants

- Preserve user work. Do not autostash, reset, or replace a branch tip without
  first understanding and preserving its unique commits.
- Resolve conflicts hunk-by-hunk. Do not take an entire conflicted file from one
  side unless the user explicitly requests that result.
- Run shell queries separately and reuse the literal refs and SHAs they return.
  Avoid command substitution in later commands.
- Prefix every rebase invocation with both editor settings because environment
  state may not persist between tool calls:
  ```bash
  GIT_SEQUENCE_EDITOR=true GIT_EDITOR=true git rebase <arguments>
  ```
- Do not push unless the user asks.

## 1. Establish the source and target

Read the repository's `AGENTS.md` and relevant build or CI instructions before
acting. Do not assume a build command, test runner, remote name, or default
branch from another repository.

Gather the current state:

```bash
git branch --show-current
git rev-parse HEAD
git status --short --branch
git remote -v
```

Stop if tracked or untracked work could be overwritten or makes the rebase
ambiguous. A detached `HEAD` has no `@{u}`; identify the intended PR head or
destination branch before proceeding, and create a temporary local branch if
needed to preserve the result.

Determine the target from the user's requested `--onto` arguments or the PR's
actual base branch. If neither exists, inspect the relevant remote's default
branch instead of assuming `origin/master`:

```bash
git symbolic-ref --short refs/remotes/<remote>/HEAD
```

Fetch the chosen target and record its immutable SHA:

```bash
git fetch <target-remote> <target-branch>
git rev-parse <target-remote>/<target-branch>
```

Record that output as `TARGET_SHA`. Read the corresponding PR when one exists
to understand branch intent and confirm the base.

### Tracking-branch safety

If the current branch has a tracking branch, identify and fetch it, then inspect
the relationship:

```bash
git rev-parse --abbrev-ref --symbolic-full-name '@{u}'
git fetch <tracking-remote> <tracking-branch>
git rev-list --left-right --count HEAD...'@{u}'
```

- Behind only and no local commits: fast-forward before rebasing:
  ```bash
  git merge --ff-only '@{u}'
  ```
- Ahead only: preserve and rebase the local tip.
- Ahead and behind: inspect patch-equivalent and unique commits before choosing
  a source:
  ```bash
  git log --oneline --left-right --cherry-pick HEAD...'@{u}'
  ```
  Do not assume the tracking branch is authoritative. A previously rebased but
  unpushed local stack naturally appears diverged from its old tracking branch.
  Preserve that local stack when it contains the latest verified resolutions;
  use the tracking tip when it contains newer author changes. If both sides
  contain distinct intentional changes and the correct source is unclear, ask
  the user.

Before switching away from or moving a branch tip, create a unique backup using
the literal pre-rebase SHA, for example:

```bash
git branch codex/backup-<short-pre-rebase-sha> <full-pre-rebase-sha>
```

If that name already exists, choose another explicit suffix. When rebasing a
fetched tracking tip instead of the current local tip, switch to it detached,
perform and verify the rebase, then move the intended local branch to the new
tip and switch back.

### Record the old range

After choosing the source tip, record these literal values:

```bash
git rev-parse HEAD
git merge-base <TARGET_SHA> HEAD
```

Call them `PRE_REBASE_HEAD` and `OLD_BASE`. Inspect the stack:

```bash
git rev-list --count <OLD_BASE>..<PRE_REBASE_HEAD>
git log --oneline <OLD_BASE>..<PRE_REBASE_HEAD>
git rev-list --merges <OLD_BASE>..<PRE_REBASE_HEAD>
git diff --name-only <OLD_BASE>..<PRE_REBASE_HEAD>
```

If the stack contains merge commits whose structure matters, use
`--rebase-merges` or get direction instead of silently flattening them. Inspect
upstream changes between `OLD_BASE` and `TARGET_SHA` in the touched areas before
starting.

## 2. Rebase

Use the recorded target SHA so the base cannot move during the operation:

```bash
GIT_SEQUENCE_EDITOR=true GIT_EDITOR=true git rebase <TARGET_SHA>
```

For a stacked rebase, preserve the user's `--onto` semantics exactly while
still prefixing the command with both editor settings.

## 3. Resolve each stop intentionally

Identify conflicts and inspect their context with `rg` and Git:

```bash
git diff --name-only --diff-filter=U
rg -n -C 30 '^(<<<<<<<|=======|>>>>>>>)' <conflicted-file>
git diff -- <conflicted-file>
```

Look for both merge and direct commits in the upstream range that caused the
interaction; avoid unrelated historical results and unnecessary pipelines:

```bash
git log --oneline --merges -5 <OLD_BASE>..<TARGET_SHA> -- <conflicted-file>
git log --oneline -20 <OLD_BASE>..<TARGET_SHA> -- <conflicted-file>
```

Map the causing commit to its PR when applicable, using the available GitHub
tools or `gh api repos/<owner>/<repo>/commits/<sha>/pulls`.

Resolve minimally: retain unrelated target changes and apply only the source
branch's intended behavior. Then check the result:

```bash
rg -n '^(<<<<<<<|=======|>>>>>>>)' <resolved-files>
git diff --check
```

Rebuild and run proportionate targeted tests before continuing when the
conflict affects code, build metadata, or tests. Use the repository's own
documented commands, presets, CI scripts, and any applicable repository-specific
test skill. Do not copy a build or test invocation from another project.

Stage only resolved files and continue with editor settings applied again:

```bash
git add <resolved-files>
GIT_SEQUENCE_EDITOR=true GIT_EDITOR=true git rebase --continue
```

Repeat for later stops. Do not blindly skip an empty or failing commit; inspect
why it became empty or failed first.

## 4. Verify the completed rebase

Record `POST_REBASE_HEAD`, then verify ancestry and compare explicit ranges:

```bash
git rev-parse HEAD
git merge-base --is-ancestor <TARGET_SHA> <POST_REBASE_HEAD>
git rev-list --count <TARGET_SHA>..<POST_REBASE_HEAD>
git range-diff <OLD_BASE>..<PRE_REBASE_HEAD> <TARGET_SHA>..<POST_REBASE_HEAD>
```

The explicit ranges remain valid if the rebase drops a patch already present
upstream. Investigate every non-equivalent range-diff change; do not explain it
away solely because the rebase completed.

Using repository-local guidance, perform the final build and targeted tests at
the completed tip. Build fuzz targets when the source changes fuzz code. Also
run:

```bash
git diff --check <TARGET_SHA>...<POST_REBASE_HEAD>
git status --short --branch
```

An ahead/behind tracking annotation can be expected after rewriting history,
but the worktree itself must be clean. Refresh the target ref after long
validation; if it advanced beyond `TARGET_SHA`, either rebase again when the
request requires the latest tip or report the newer base clearly.

## 5. Report

Include:

- The recorded target ref/SHA and number of rebased commits.
- Conflict files, causing commits or PRs, and the resolution approach.
- Additional upstream interactions found during build or test diagnosis.
- Build and targeted test results.
- Every notable range-diff change, including dropped or newly empty commits.
- Final worktree state and whether anything was pushed.
