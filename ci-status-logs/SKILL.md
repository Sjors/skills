---
name: github-ci-status-logs-mcp
description: Retrieve CI status and logs via gh CLI/API without using web URLs
---

# Skill Instructions

- **Agent behavior:**
    - Follow the steps below in order; only deviate if a step fails.
    - Prefer the gh CLI + API over web URLs for Actions logs and status.
    - **Never** open or fetch GitHub Actions web URLs (e.g., `/actions/runs/.../job/...`).
    - Do not run `gh auth` commands (e.g. `gh auth status` or `gh auth login`); rely on existing token-based auth and proceed directly with `gh`/`gh api`.
    - If a `gh` command fails with an authentication error, report the error and ask the user to refresh the token.
    - Keep narration minimal: run the commands, then summarize results.

- **Common workflow (checked-out branch → CI status/logs):**
    - Assumptions: `gh pr view` can resolve the PR for the current branch.
    - Run commands from an interactive shell with the repo checked out.
    - **Step blocks are standalone:** each block below recomputes the minimal context it needs.
    - **Run each line as a separate command** (no chained `&&`).

    - **Step 0 (standalone): Resolve PR context (local branch → PR number + URL)**

```sh
git -C /abs/path/to/repo branch --show-current
```

```sh
cd /abs/path/to/repo; gh pr view --json number --template '{{.number}}'
```

```sh
cd /abs/path/to/repo; gh pr view --json url --template '{{.url}}'
```

```sh
echo "<pr_url>" | cut -d/ -f4
```

```sh
echo "<pr_url>" | cut -d/ -f5
```

    - If `gh pr view` cannot resolve the PR, stop and ask the user for the PR URL/number.

    - **Step 1: List recent workflow runs for the PR branch**

```sh
gh run list --repo <owner>/<repo> --limit 10 --json databaseId,status,conclusion,headBranch,displayTitle
```

    - Pick the run that matches the PR branch and the relevant workflow name. If ambiguous, ask the user which run to inspect.

    - **Step 2: List jobs for the selected run**

```sh
gh api repos/<owner>/<repo>/actions/runs/<run-id>/jobs --jq '.jobs[] | {id, name, status, conclusion, started_at, completed_at}'
```

    - Identify failed or cancelled jobs by `conclusion`.

    - **Step 3: Fetch logs for a specific job by `id`**

```sh
gh api repos/<owner>/<repo>/actions/jobs/<job-id>/logs 2>&1 | grep -i -A5 -B5 "error\|fail\|assert"
```

    - Or fetch the tail of the logs:

```sh
gh api repos/<owner>/<repo>/actions/jobs/<job-id>/logs 2>&1 | tail -100
```

    - **Important:** Do NOT guess what went wrong without fetching actual logs first. Always retrieve the real error message before proposing fixes.

- **Related helper:** if the task also involves PR review-thread triage (outside CI logs), use `~/utils/extract_threads.py` with `get_review_comments` JSON output.

