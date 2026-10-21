# PR references in merge and cherry-pick messages

Use this policy for messages created or rewritten by merging, cherry-picking,
or their specialized callers. Its purpose is to reduce noise on the main
upstream repository while working on a PR that targets the user's fork.

## Determine the destination and reference repositories

Resolve the destination PR's **base repository**, not just the repository
hosting its head branch. Use the user's stated destination or verified PR
metadata. For example, query the branch's PR in the intended repository with
`gh pr view <branch-or-pr> --repo <owner/repo> --json url,headRefName,baseRefName,headRepository`.
The repository in the returned PR URL is the base repository. Check the head
matches the branch being changed. Do not infer the destination from `origin`,
the directory's PR number, or the fact that a contributor uses a fork.

Identify the main upstream repository from repository/PR metadata and the
user's context. For Bitcoin Core it is `bitcoin/bitcoin`; other projects have
their own upstream. If context is still unresolved after inspection, clarify
the destination before choosing reference formatting.

## Formatting rule

When the destination PR targets the user's fork, replace the `#` with a space
in references to PRs on the main upstream repository. Apply this throughout
the subject and body, including attribution and copied merge summaries.

For a destination PR on `Sjors/bitcoin` and main upstream `bitcoin/bitcoin`:

| Reference | Result |
| --- | --- |
| `bitcoin/bitcoin#36442` | `bitcoin/bitcoin 36442` |
| `Merge bitcoin/bitcoin#36442: descriptor changes` | `Merge bitcoin/bitcoin 36442: descriptor changes` |
| `Backport of bitcoin/bitcoin#33593.` | `Backport of bitcoin/bitcoin 33593.` |
| `#36442` known to refer to `bitcoin/bitcoin` | ` 36442` (keep natural surrounding spacing) |
| `achow101/bitcoin#123` | Unchanged: another fork. |
| `Sjors/bitcoin#91` | Unchanged: the user's fork. |
| `#91` known to refer to `Sjors/bitcoin` | Unchanged. |

Resolve bare `#N` references from their source context. A message copied from
an upstream commit or PR may use upstream-local numbers even though the new
commit belongs to a fork. If useful, qualify the plain reference as
`bitcoin/bitcoin 36442` to retain that context. Do not guess the meaning of
an ambiguous bare number, and do not blindly replace every `#`.

Keep `#` for PRs on other forks, including dependencies from another fork.
Outside the fork-targeting condition, retain normal PR-reference formatting;
an upstream PR whose head is on the user's fork does not activate this rule.
Do not add a main-repo PR URL as a replacement for a suppressed reference.
When a copied message already contains such a URL, render that PR reference
as the same plain `owner/repo N` form under the fork-targeting condition.

Change only PR-reference text. Preserve authorship, coauthor trailers, source
SHAs, code examples and scripted-diff bodies, subtree metadata, and unrelated
hash characters. If reference-like text is executable rather than prose,
leave it intact and report the limitation. Review the final full messages
before publishing; do not expand the task into rewriting pre-existing history.
