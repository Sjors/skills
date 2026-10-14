---
name: watch-pr-ci
description: Keep an eye on CI for the pull request of the CURRENT git branch and repair failures locally. Resolve the PR from the checked-out branch (works in any worktree), watch its checks until something fails or all pass, then fetch the failing job log, diagnose, reproduce locally, and fix on the branch without pushing. Use when asked to watch / keep an eye on / babysit / monitor the PR or its CI for the current branch or worktree.
---

# Watch PR CI for the current branch

Monitor CI for the PR that corresponds to the branch checked out in the current
working directory, surface failures the moment they appear, and repair them
locally on that branch. Designed to run across many worktrees — always resolve
the branch and PR fresh from the current directory; never assume a hard-coded PR
number.

## Guardrails

- **Never open or fetch GitHub Actions web URLs** (`/actions/runs/.../job/...`).
  Parse owner/repo/run-id/job-id from them if given, and use `gh api` for logs.
- **Do not push.** Fold fixes into their introducing commits as described in
  Step 5, unless the user explicitly requests a different commit workflow.
  Only push if explicitly told to.
- **Do not run `gh auth`** commands; rely on existing token auth. If a `gh`
  call fails with an auth error, report it and ask the user to refresh the token.
- Keep fixes tightly scoped to the CI failure. Don't opportunistically refactor.
- Keep narration minimal: run the commands, then summarize.

## Step 1 — Resolve the branch and its PR

```
git symbolic-ref --short HEAD          # current branch (fail loudly if detached)
gh pr view --json number,headRefName,headRefOid,state,url,title,headRepository,headRepositoryOwner
```

`gh pr view` with no argument uses the current branch's PR **in the default repo**
— which for a fork clone is usually the upstream (e.g. `upstream/project`), *not*
your fork. So a PR opened **against your own fork** (fork-internal, base+head both
on `example-user/project`) will report `no pull requests found` here even
though it exists. **Do not stop at the first miss** — fall through to the fork lookup:

1. Find the branch's push target from its tracking remote, and turn that remote's
   URL into an `owner/repo` slug:
   ```
   git rev-parse --abbrev-ref --symbolic-full-name @{u}   # e.g. fork/feature-branch
   # e.g. git@github.com:example-user/project.git
   git remote get-url <remote>
   ```
   Parse `owner/repo` from the URL (strip `.git`; handle both `git@host:owner/repo`
   and `https://host/owner/repo`). Call this `REPO`.
2. Look for the PR in that fork repo by head branch:
   ```
   gh pr list --repo <REPO> --head <branch> --state all \
     --json number,headRefName,headRefOid,state,url,title
   ```
   (If the branch has no upstream, also try `gh pr view --repo <REPO> <branch>`.)

Only after **both** the default-repo and the fork-repo lookups come up empty
should you conclude there is no PR and stop (nothing to watch).

Record the PR number, `headRefOid`, **and the `REPO` it lives in** (default repo
or the fork). **Every later `gh pr ...` / `gh api` call in this skill must target
that same repo** — pass `--repo <REPO>` to `gh pr checks`/`gh pr view`, and use
`<REPO>` in `gh api repos/<REPO>/...` log fetches. Getting this wrong silently
watches the wrong repo's checks (or none).

Sanity-check sync: compare `git rev-parse HEAD` to the PR's `headRefOid`.
- If they differ, the local branch is ahead/behind the pushed head. Tell the
  user which commits are unpushed (`git log --oneline <headRefOid>..HEAD`) so CI
  results are interpreted against the right commit. Watch the pushed head.

## Step 2 — Snapshot current CI status

```
gh pr checks <number> --repo <REPO>    # NAME <tab> STATE <tab> DURATION <tab> URL
```

States are `pass` / `fail` / `pending` (plus `skipping`, `cancelled`). Report a
one-line summary (e.g. `pass=3 pending=20 fail=0`) and name anything already
failed. For richer per-check detail, use an available CI status/log skill
that applies to the current repository.

## Step 3 — Monitor until something is actionable

For Codex, prefer one blocking `gh` watch call instead of agent-side polling or
periodic status wakeups. This keeps token use low while still returning as soon
as a check fails or all checks complete:

```
gh pr checks <number> --repo <REPO> --watch --fail-fast --interval 60
```

Run it as a single long-running command and wait on that session. Do not poll
the session every 30 seconds for status; only report when the command returns
with a failure or with all checks complete. If the command exits because of a
timeout or tool/session cap while checks are still pending, start the same
blocking watch again.

**Codex desktop:** Treat this as a watch for the current CI run. Do not create
a heartbeat automation or scheduled task unless the user explicitly asks for
scheduled monitoring. If a heartbeat was already created for this watch, pause
it before starting the blocking command. Redirect repetitive watch output to a
repo-local log. If the tool yields a running session, keep waiting on that same
session within the tool's allowed wait limits; do not replace it with repeated
CI snapshots or narrate unchanged status. A tool yield is not command completion.

For Claude Code, which may need a background process that survives turns and
does not have the same Codex session-wait behavior, use a background polling
loop that exits early on the first failure or when nothing is pending:

```
for i in $(seq 1 60); do
  out=$(gh pr checks <number> --repo <REPO> 2>/dev/null)
  fails=$(echo "$out" | awk -F'\t' 'tolower($2) ~ /fail/ {print $1}')
  pend=$(echo "$out" | awk -F'\t' 'tolower($2) ~ /pending/ {c++} END{print c+0}')
  echo "[$(date +%H:%M:%S)] pass=$(echo "$out"|awk -F'\t' 'tolower($2)~/pass/{c++}END{print c+0}') pending=$pend fail=$(echo "$fails"|grep -c .)"
  [ -n "$fails" ] && { echo "=== FAILURE ==="; echo "$out"; exit 0; }
  [ "$pend" -eq 0 ] && { echo "=== ALL COMPLETE ==="; echo "$out"; exit 0; }
  sleep 90
done
echo "=== window ended, still pending ==="; echo "$out"
```

Launch the Claude Code loop with `run_in_background: true` and `timeout: 570000`.
When either monitor returns:
- **failure(s)** → go to Step 4.
- **all complete, no fails** → report green and stop.
- **still pending** → re-launch the same monitor (re-arm). Don't burn turns
  faster than the monitor's own interval; nothing changes in between.

In a Claude Code `/loop` (self-paced) context, `ScheduleWakeup` may be used
to pace polls up to 1h apart. For Codex, keep the blocking `gh --watch` call as
the default.

## Step 4 — Diagnose a failure

1. Identify the failing job(s) from the `gh pr checks` output.
2. Pull the job log with `gh api` (not the web URL), using the job ID from the
   check URL:
   ```
   gh api repos/<REPO>/actions/jobs/<job-id>/logs
   ```
   An available CI status/log skill may provide more detailed job discovery.
   Store large logs in a repo-local ignored temp directory and delete after.
3. Consult the current repository's `AGENTS.md`, CI configuration, and applicable
   debugging or testing skills to reproduce and validate the failure. Follow
   those workflows for the required build host, toolchain, test commands, and
   runtime settings. Keep repository-specific reproduction instructions in
   those skills rather than in this monitoring skill.

## Step 5 — Repair on the branch

- Apply the minimal fix and always fold it into the PR commit that introduced
  the failure, following any applicable repository workflow. Use the
  `interactive-rebase` skill if available. This applies to every CI fix, not
  only per-commit build failures.
- Validate the amended commit and final tip, and inspect the range-diff.
- Report concisely: what failed, root cause, the fix, and that it's ready for the
  user to push. Do not push.
- Re-arm the monitor (Step 3) for the remaining pending checks — one fixed job
  doesn't mean the rest are green.

## Notes

- A single push can invalidate in-flight results; if the user pushes mid-watch,
  re-resolve the head (Step 1) and restart monitoring against the new commit.
