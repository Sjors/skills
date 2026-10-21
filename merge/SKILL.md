---
name: merge
description: Merge Git branches or PR dependencies, including temporary test merges and message handling for subtree or recreated merges. Reduce main-repository PR pingbacks when the destination PR targets the user's fork. Use rebase for moving a branch onto a new base.
---

# Merge

## Establish context

Read repository instructions, inspect status and remotes, and identify the
destination branch and exact source ref/SHA. Preserve user work and record
the starting tip before a persistent merge. Fetch the intended source rather
than relying on a stale remote-tracking ref. Confirm which repository the
destination PR targets; a fork-owned head alone does not identify that target.

Before preparing messages, read [PR reference formatting](references/pr-references.md).
Apply it to the entire new merge message, including copied subjects, PR
descriptions, shortlogs, and attribution. The same policy applies when another
workflow recreates merge commits or generates subtree squash/merge messages.

## Persistent merge

Follow the requested topology. A fast-forward creates no new message; do not
force a merge commit merely to edit references. When a merge commit is needed,
prepare its full message in a file under the repository's ignored temp directory
and stop before committing so the final message can be inspected:

```bash
git merge --no-ff --no-commit <source-sha>
```

Resolve conflicts from the source PR's intent and destination changes, hunk by
hunk. Do not discard a whole side to make the merge pass. Check the staged
diff, run relevant checks, and commit using the prepared message:

```bash
git diff --cached --check
GIT_EDITOR=true git commit --file <message-file>
```

Verify the resulting parents, final tree/diff, and full message. Inspect every
new merge message, rather than just the branch tip. Source commits brought in
by the merge keep their original hashes and messages; this procedure controls
newly created messages and does not guarantee removal of references already in
incoming history. Do not rewrite that history as an incidental cleanup.

## Temporary test merge

When reproducing CI, match the logged merge semantics in a clean disposable
checkout. `--no-commit` alone still permits a fast-forward; use
`--no-ff --no-commit` when the test must leave an uncommitted merge. Record the
starting tip and restore that exact state after testing, including after a
fast-forward. Discard only the reproduction's own changes. Never commit or
publish this temporary merge.

## Subtree and recreated merges

Keep the specialized caller's history structure. For subtree updates, use
[subtree-update](../subtree-update/SKILL.md); inspect both generated messages
and preserve all `git-subtree-*` metadata and the squash/merge relationship.
Prefer a prepared `--message` where supported; if a generated message still
needs adjustment, use the caller's supported replay/reword workflow and verify
the tree and metadata remain intact. Apply formatting before fixing hashes for
reuse across destination branches.

For merges recreated by `--rebase-merges`, use
[interactive-rebase](../interactive-rebase/SKILL.md) for message surgery and
verify the resulting topology and range-diff. Do not rewrite unchanged
dependency commits outside the requested operation.

Report the source, resulting tip, conflict resolutions, checks, and any message
formatting applied. Pushing requires the user's existing authorization.
