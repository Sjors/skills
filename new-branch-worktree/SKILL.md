---
name: new-branch-worktree
description: >-
  Create a new Git branch using the user's dated branch names and a matching
  sibling worktree when starting from main or master. Use when the user asks
  to create a new branch or start an assignment on a new branch, in any
  repository. Also use after opening a PR, or when a PR number becomes known,
  to replace the worktree's x placeholder with that number. Understand the
  assignment before choosing the topic name.
---

# New branch and worktree

When the user requests a new branch and the current branch is `main` or
`master`, create the branch in a separate worktree using this personal
convention. This applies across repositories.

## Understand the assignment first

Read the request, relevant linked issues or PRs, repository guidance, and
enough code to identify the actual change before choosing a name. Keep this
initial investigation read-only; make the implementation changes in the new
worktree. Choose a short, descriptive topic for the intended result, such as
`ntime-start`, rather than a generic name like `fix` or `task`.

If the assignment is not yet known, ask what the branch will be for before
creating it. An explicit branch name, worktree path, or base from the user
takes precedence over these defaults.

## Naming and location

- Branch: `YYYY/MM/<topic>`, using the current local year and two-digit month.
  This is the user's preferred branch convention; do not add `codex/`.
- Worktree: `<primary-repo-parent>/<repo>-pr/<PR>-<topic>`.
  Derive the primary repository path from Git's worktree/common-directory
  information, rather than treating an existing worktree's basename as the
  repository name. Inspect existing worktrees for the established location.
- Use lowercase words separated by hyphens for new topic names.
- Until this work has a PR number, use a run of lowercase `x` characters
  for `<PR>`. Infer the number of characters from this repository's other
  worktrees: follow existing placeholders, or the digit count of recent
  numbered worktrees (for example, `xxx` for three digits or `xxxxx` for
  five). This is an approximate local convention; do not look up the latest
  opened PR number just to choose the placeholder length. If there are no
  examples, use `xxx`.
  A linked reference PR, especially one in another repository, is not the
  PR number for this work. Do not open a PR just to obtain a number.

For example, if the other `<repo>-pr` worktrees have five-digit PR
numbers, work during September 2026 could use branch `2026/09/peer-timeout`
and worktree `<primary-repo-parent>/<repo>-pr/xxxxx-peer-timeout`.

## Create and continue

Inspect the current branch, working-tree status, and existing branches and
worktrees. Default to the current `main` or `master` commit as the base unless
the assignment specifies a different base. Leave the original checkout and
its uncommitted changes intact; do not automatically move or stash them.

Create the branch and worktree together with
`git worktree add -b <branch> <worktree-path> <base>`. Do not force or overwrite
an existing branch or directory. If a name is already occupied by unrelated
work, choose a more specific topic; reuse an existing worktree only when the
request is clearly to continue that work.

Run all subsequent edits, builds, tests, and commits for the assignment from
the new worktree. Report the branch and absolute worktree path. Creating this
worktree is part of the requested branch creation, so no separate confirmation
is needed beyond any required tool permission handling.

When already on a feature branch or detached HEAD, do not automatically
apply the main/master workflow or infer a new base. Follow the user's stated
branch/base intent and ask only if the choice is consequential and unclear.

## Replace the placeholder after opening a PR

Once a PR is opened for the branch, including a draft PR, move its
worktree from its x-placeholder name to `<PR>-<topic>` in the same parent
directory, regardless of how many x characters the placeholder contains.
Apply this when opening the PR yourself as part of an authorized task, or
when the user reports that the PR now exists. This step applies on the
feature branch; it does not require being on `main` or `master`.

Resolve the PR in the target repository and verify its head repository and
branch match this work. Use its assigned number, not a linked reference PR.
Keep the branch name and topic unchanged. If the worktree already has the
correct numbered path, no move is needed.

Check the worktree status, destination, and any ongoing work before moving.
Do not move it while builds, tests, or other agents are actively using it;
coordinate or finish that work first. Preserve uncommitted and untracked
files. Do not overwrite an occupied destination or force a locked worktree.

From outside the worktree, use `git worktree move <old-path> <new-path>` so
Git updates its administrative paths. Verify `git worktree list`, the branch,
HEAD, and working-tree status at the new location. Continue all commands
from the new path and report it to the user.

Generated build files may contain absolute paths. Refresh affected build
configuration before the next build, using the repository's required
options; for CMake, a fresh configuration may be needed. Preserve source and
user files, and do not leave a compatibility symlink at the old path by
default. Moving the worktree does not authorize opening, pushing, or editing
any PR beyond the user's existing request.
