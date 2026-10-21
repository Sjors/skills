---
name: cherry-pick
description: Apply selected Git commits from another branch or PR, preserving authorship and reviewing conflicts and commit messages. Reduce main-repository PR pingbacks when the destination PR targets the user's fork. Use backport for downstream adaptation and attribution conventions.
---

# Cherry-pick

## Establish context

Read repository instructions and inspect the working tree, destination branch,
remotes, and source PR/commits. Preserve user work and record the starting tip.
Fetch the intended source, verify its association with the source PR, and
record an explicit ordered commit list. Do not infer the destination PR's
base repository from the source PR or the current head's fork owner.

Read [PR reference formatting](../merge/references/pr-references.md) before
applying commits. This policy covers inherited subjects and bodies as well as
new attribution. Rebuilding old local commits follows the same rule when the
destination PR targets the user's fork.

Use [backport](../backport/SKILL.md) for downstream adaptation and its
`Backport of` attribution convention. Verification replays and tutorial
rebuilds retain their caller's message conventions; they do not automatically
gain backport attribution. Add `-x` only when the caller's traceability policy
calls for it; backport explicitly omits that trailer.

## Apply and review each commit

Apply the selected commits in order. For a merge commit, inspect its parents
and intended patch before choosing a mainline parent; do not guess `-m 1`.
Stop at the first failure or conflict, including when using a replay helper.

```bash
GIT_EDITOR=true git cherry-pick <source-sha>
```

Resolve conflicts using the source intent and destination behavior, preserving
unrelated changes. Stage only the resolved files, inspect the diff, and use
`GIT_EDITOR=true git cherry-pick --continue`. Inspect empty commits before
deciding whether to skip or retain them.

Inspect each resulting full message. When formatting or caller-required
attribution is needed, write the complete revised message to an ignored
repo-local file and amend that commit before starting the next pick:

```bash
GIT_EDITOR=true git commit --amend --file <message-file>
```

Keep the index clean for a message-only amend. Preserve the original author,
author date, useful explanation, and author trailers. Do not use `--reset-author`.
If picking a sequence through a helper, ensure it applies this same message
policy or normalize the newly created messages before publishing. Use
[interactive-rebase](../interactive-rebase/SKILL.md) for edits to earlier picks.

## Verify

Inspect the complete picked range, not just its tip: verify order, patches,
authors/dates, caller-required attribution, and every full commit message.
Use a range-diff against the selected source sequence where applicable and
explain adaptations or dropped commits. Rebuild and run targeted checks when
code changes; message-only edits require tree and metadata verification.

Verification-only replay branches remain local. Report the selected commits,
resulting tip, adaptations, checks, and reference formatting. Push only under
the user's existing authorization.
