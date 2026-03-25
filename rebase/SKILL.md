---
name: rebase
description: Rebase the current branch onto upstream and resolve conflicts safely.
---

# Skill Instructions

## Goal
Move a branch onto latest upstream history with minimal drift and clear verification.

## Critical Reminders

1. Set both editors before rebasing to avoid hangs/prompts:
```bash
export GIT_SEQUENCE_EDITOR=true
export GIT_EDITOR=true
```
2. Resolve conflicts hunk-by-hunk; do not use whole-file `--ours/--theirs` unless explicitly requested.
3. Suggest reusable approval prefixes for repeated git operations (for example `["git","add"]`, `["git","rebase"]`).
4. Avoid `$()` command substitution — run each command separately and use the literal values observed in subsequent commands.
5. When you need to explore the build or filesystem, use `ls <project-root>/` or `ls <project-root>/build/` so a single permission covers the whole tree. Do not request `ls` on deep specific paths.
6. For tests, pass the specific test names as arguments to the runner binary and request permission on the runner binary itself (not the full invocation). This way the same permission covers future test runs.

## Procedure

### 1. Gather context

Run these separately so no `$()` substitution is needed:
```bash
git branch --show-current
git rev-parse HEAD
git merge-base master HEAD
```
Then, using the merge-base hash observed above:
```bash
git rev-list --count <merge-base>..HEAD
git log --oneline -<num_commits_plus_3>
```

Record `PRE_REBASE_HEAD` and `NUM_COMMITS` from the output for use later.

Before rebasing:
- Find/read the corresponding PR for branch intent.
- Confirm target base branch (default: `origin/master`).

### 2. Fetch and start rebase

```bash
git fetch origin master
git log --oneline origin/master -5
git rebase origin/master
```

For stacked rebases, run the user-provided `--onto` command exactly.

### 3. Handle conflicts

Per conflict:

1. Identify files:
```bash
git diff --name-only --diff-filter=U
```
2. Inspect conflict hunks using the Grep tool (not bash grep) on the conflicted file.
3. Find likely upstream merge/PR causing churn:
```bash
git log --oneline --merges origin/master -- <conflicted_file> | head -5
```
4. Resolve minimally and intentionally:
- Keep unrelated upstream changes.
- Apply only branch-intended behavior.
5. Build and run targeted tests before continue. Use `nproc` for the core count. On macOS `nproc` may not exist; if so, set the alias first:
```bash
alias nproc="sysctl -n hw.physicalcpu"
cmake --build build -j$(nproc)
```

If exploring the build directory is needed, use `ls <project-root>/build/` rather than a deep specific path.

6. Run targeted tests by passing test names as arguments to the runner:
```bash
<test-runner-binary> <test-name-1> <test-name-2>
```
Request permission on `<test-runner-binary>` so it covers all future test invocations.

7. Continue:
```bash
git add <resolved_files>
git rebase --continue
```

### 4. Post-rebase verification

Using the literal values of `PRE_REBASE_HEAD` and `NUM_COMMITS` recorded in step 1:
```bash
git merge-base HEAD <PRE_REBASE_HEAD>
```
Then, using the base hash observed above:
```bash
git range-diff <base>...<PRE_REBASE_HEAD> HEAD~<NUM_COMMITS>...HEAD
```

Also provide this copy-paste command for manual inspection with values filled in:

```bash
PREV=<original-short-hash> N=<num-commits> && git range-diff `git merge-base --all HEAD $PREV`...$PREV HEAD~$N...HEAD
```

Minimum validation:
- Build succeeds.
- Targeted tests for touched areas pass.
- If fuzz code changed, build fuzz target(s) as well.

### 5. Final summary checklist

Include:
- Number of rebased commits.
- Conflict files and causing PRs.
- Conflict-resolution approach.
- Build/test results.
- Notable `range-diff` outcomes.
- The filled-in manual inspection command (`PREV=... N=...`).
