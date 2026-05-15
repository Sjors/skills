---
name: github-ci-status-logs-mcp
description: Retrieve GitHub CI status with MCP and Actions job logs with gh API when needed
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

- **Terminal safety:** when running any terminal commands as part of this workflow, follow the separate terminal skill: `~/.copilot/skills/terminal/SKILL.md`.

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

    - **Important:** Do NOT guess what went wrong without fetching actual logs first. Always retrieve the real error message before proposing fixes.

- **Related helper:** if the task also involves PR review-thread triage (outside CI logs), use `~/utils/extract_threads.py` with `get_review_comments` JSON output.
