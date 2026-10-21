---
name: github-ci-status-logs-mcp
description: Investigate GitHub CI failures using MCP status and gh API logs, validate Bitcoin Core fixes on copilot, and fold repairs into their introducing commits. Also use for read-only CI status and log retrieval.
---

# Skill Instructions

- **Agent behavior:**
    - Follow the steps below in order; only deviate if a step fails.
    - Prefer GitHub MCP tools for PR metadata, commit status, and check-run/job discovery.
    - Use `gh api` only for Actions job logs, because the current GitHub MCP surface does not expose a job-log download tool.
    - **Never** open or fetch GitHub Actions web URLs (e.g., `/actions/runs/.../job/...`). If the user gives one, parse the owner/repo, run id, job id, and optional PR number from the URL instead.
    - Do not run `gh auth` commands (e.g. `gh auth status` or `gh auth login`); rely on existing token-based auth and proceed directly with `gh api` only when logs are needed.
    - If a `gh api` command fails with an authentication error, report the error and ask the user to refresh the token.
    - Keep narration minimal: run the MCP calls/commands, then summarize results.

- **Shell usage:** keep local commands short, and store large logs in a repo-local ignored temp directory such as `./tmp/`.
    - Remove downloaded log files after summarizing unless the user asked to keep them.

- **Common workflow (PR or Actions URL → CI status/logs):**
    - Use MCP first whenever the PR number is known.
    - If the user gives an Actions URL, do not open it. Extract:
        - owner/repo from the path,
        - run id from `/actions/runs/<run-id>`,
        - job id from `/job/<job-id>`,
        - PR number from `?pr=<number>` when present.

    - **Step 0: Resolve PR context**

    - If the PR number is known, call GitHub MCP:

```
mcp__github__.pull_request_read({
  "owner": "<owner>",
  "repo": "<repo>",
  "pullNumber": <pr-number>,
  "method": "get"
})
```

    - For a checked-out branch without a PR number, first get the branch name locally:

```sh
git -C /abs/path/to/repo branch --show-current
```

    - Then search PRs with GitHub MCP, scoped to the expected repository and branch. If multiple PRs match, ask the user which one to inspect.

```
mcp__github__.search_pull_requests({
  "owner": "<owner>",
  "repo": "<repo>",
  "query": "head:<branch-name>",
  "perPage": 10
})
```

    - **Step 1: List check runs for the PR head**

```
mcp__github__.pull_request_read({
  "owner": "<owner>",
  "repo": "<repo>",
  "pullNumber": <pr-number>,
  "method": "get_check_runs",
  "perPage": 100
})
```

    - Identify failed or cancelled jobs by `conclusion`.
    - If a job id was supplied in an Actions URL, match it against `id` in the check-run list to get the job name and status.

    - **Step 2: Fetch logs for a specific job by `id`**
    - Current limitation: use `gh api` for this step; MCP can list check runs but does not download Actions job logs.
    - Store large logs in a repo-local ignored temp directory such as `./tmp/`.

```sh
gh api repos/<owner>/<repo>/actions/jobs/<job-id>/logs
```

    - Then search the saved log for failure terms with `rg`, or inspect the relevant tail:

```sh
rg -n -i -C 5 "error|fail|assert|fatal|traceback|cmake error" /abs/path/to/repo/tmp/<job-log>.log
```

    - For Bitcoin Core "test ancestor commits" jobs, also extract the failing
      commit hash and the exact per-commit command shape from the log before
      reproducing locally or remotely. These jobs commonly rebase with
      `--exec`, merge the target branch into each ancestor with
      `git merge --no-commit`, run the per-commit test script, then reset. Match
      that shape when reproducing the failure.
      Use [merge](../merge/SKILL.md) in temporary-test mode: record the starting
      tip, restore it after testing even if `--no-commit` fast-forwards, and
      never commit or publish the reproduction merge. Preserve the destination
      PR context before detaching for ancestor tests.

    - **Important:** Do NOT guess what went wrong without fetching actual logs first. Always retrieve the real error message before proposing fixes.
    - For successful jobs, still inspect relevant indicators the user asked about, such as cache hit/miss lines, tool versions, warnings, and final summary sections.

## Validate and fold CI fixes

Apply this section when repairing a failure is in scope. Requests only for
status, logs, or an explanation remain read-only.

- **Use copilot for Bitcoin Core IWYU and sanitizer fixes.** Try the failing
  check and the candidate fix there using the CI toolchain and configuration.
  A native macOS build or a different IWYU version is not sufficient validation
  of a Linux CI failure. Use [bitcoin-iwyu](../bitcoin-iwyu/SKILL.md) for IWYU and
  [bitcoin-core-tests](../bitcoin-core-tests/SKILL.md) for builds and tests.
- Read the checked-out CI scripts for compiler/IWYU versions, flags, mappings,
  and runtime settings. For sanitizers, use the repository's existing
  `ASAN_OPTIONS`, `LSAN_OPTIONS`, and `UBSAN_OPTIONS`, including CI's suppression
  files, before running tests. Do not add suppressions to conceal the reported
  failure. For IWYU, check the `fix_includes.py --nosafe_headers --dry_run` diff;
  the IWYU process exit status alone does not establish success.
- Reuse the PR's isolated verification directory on copilot, following the
  repository's worktree instructions; run long jobs in the PR-numbered tmux
  session with logs under `tmp/`. If local containerized lint fails because of
  the host/container environment, try `./ci/lint.py` on copilot too. If copilot
  is unavailable or cannot match CI, report the remaining validation limit.
- **Always fold each fix into the PR commit that introduced the failure**, not
  just for ancestor-commit failures. Trace the introducing change with Git
  history, preserve unrelated work and a backup ref, then follow
  [interactive-rebase](../interactive-rebase/SKILL.md) to amend or autosquash the
  fix into that commit. Do not finish with an uncommitted patch or a separate
  tip fixup unless the user explicitly requests that. A pre-existing upstream
  failure is not a reason to rewrite unrelated upstream history.
- Validate the amended commit in isolation and the final tip with the relevant
  checks; use `~/utils/verify_bitcoin_commits.sh` for commit/range verification.
  Inspect `git range-diff` to confirm only the intended fixes changed, and report
  the rewritten commit IDs, test results, and worktree state. Do not push unless
  explicitly requested.

- **Related helper:** if the task also involves PR review-thread triage (outside CI logs), use `~/utils/extract_threads.py` with `get_review_comments` JSON output.
