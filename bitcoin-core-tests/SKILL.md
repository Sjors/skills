---
name: bitcoin-core-tests
description: >-
  Run Bitcoin Core functional and unit tests locally, including sandbox-aware
  temp/cache paths and common failure recovery.
---

# Skill Instructions

## When to use
After modifying Bitcoin Core C++ or Python test code, including during and
after a rebase.

## Sandbox escalation (Claude Code)

Builds and tests need to escape the sandbox: the shared ccache lives outside
the worktree, and `bitcoind`/`test_bitcoin` bind RPC and IPC sockets. In
Claude Code, escalate by setting `dangerouslyDisableSandbox: true` on the Bash
tool call — do **not** work around the sandbox by redirecting `CCACHE_DIR`,
disabling ccache, or reshaping the test command.

Commands that need escalation:
- `cmake --build --preset ...` (writes the shared ccache)
- `build/bin/test_bitcoin --run_test=...` (temp + socket access)
- `build/test/functional/<test>.py` and `test_runner.py ...` (RPC/IPC binds)

Keep escalated commands small and reusable so each can be approved once as a
standing allowlist rule, instead of re-prompting every run:
- **One action per call.** Don't chain `rm -rf`, `env VAR=...` preludes, or
  `&&` sequences into the escalated command — those make the command string
  unique and unmatchable. Run setup steps as their own (sandboxed) calls.
- **No session-specific paths.** Don't hardcode the worktree path (run from the
  repo root and use relative paths like `build/test/functional/<test>.py`) and
  don't pass a per-run `--tmpdir`/`--tmpdirprefix` — the value changes every run
  and defeats matching. Let the test framework pick its own temp dir unless you
  specifically need a stable path for triage.
- **`ulimit -n 10240` is generally no longer needed:** the test framework now
  caps each node's connections (`maxconnections=94` in the default config), so
  the default fd limit suffices. Only raise it if a run still fails with an fd
  exhaustion error, and escalate it as its own call (shell state does not
  persist between Bash calls, so it can't be a separate prelude to the test).
- **Pass tests as bare trailing args** (`test_runner.py <test>.py`) so one
  approved rule covers arbitrary test names.

## Environment setup

**File descriptor limit — usually unnecessary.** The test framework caps each
node's connections (`maxconnections=94` in the default config), so the default
fd limit suffices. Only if a run still fails with an fd exhaustion error, raise
it (macOS):
```bash
ulimit -n 10240
```

## Building

Use the repository's local presets so ccache settings are inherited:
```bash
cmake --preset dev-mode-no-gui
cmake --build --preset build-dev-mode-no-gui
```

Always use the shared ccache configured by the preset. Do not redirect ccache
to a repo-local cache with `CCACHE_DIR` or disable ccache to work around
sandbox errors. If a sandboxed build fails because ccache cannot write to the
shared cache, rerun the same build command with escalation so the shared cache
continues to be used.

## Running functional tests

Run the built scripts from the repo root so `build/test/config.ini` is found,
and let the framework pick its own temp dir (don't pin `--tmpdir` /
`--tmpdirprefix` — see the escalation notes above on keeping commands
reusable). A single test directly:
```bash
build/test/functional/<test>.py
```

One or more tests through the runner (bare trailing args; add `-j <jobs>` for
concurrency):
```bash
build/test/functional/test_runner.py <test1>.py <test2>.py
```

When a run misbehaves, the first thing to try is wiping the temp dir or
clearing the test cache (see Common issues), rather than pinning paths.

When tests need previous releases, remember that Bitcoin worktrees on this
host normally recreate the ignored local `releases` symlink through the shared
`post-checkout` hook. Do not infer that a verifier worktree is missing
`releases` just because it is ignored/untracked; check the actual worktree
after `git worktree add`/`git checkout`, or pass
`PREVIOUS_RELEASES_DIR=<path>` explicitly if the hook is unavailable.

## Running unit tests

Via ctest:
```bash
ctest --test-dir build -R <test_name> --output-on-failure
```

Or directly:
```bash
build/bin/test_bitcoin --run_test=<suite>/<case>
```

## Important warnings

- Do not run functional and unit tests in parallel. They share test cache
  state and can produce spurious `0 == 199` failures.
- If a sandboxed functional test fails with IPC/RPC bind permission errors,
  rerun the same command with escalation instead of changing the test shape.
- For commit-by-commit checks, run only tests present at that commit and
  return to the original branch/ref after detached-HEAD runs.

## Common issues

- `Not enough file descriptors available. -1 available, 160 required`:
  fd limit too low. Rare now that the framework sets `maxconnections=94`; if it
  still appears, raise the limit on macOS with `ulimit -n 10240`.
- `AssertionError: not(0 == 199)`: stale test cache. Clear the cache dir,
  `build/test/cache`.
- `FileNotFoundError: .../test/config.ini`: running from the wrong directory.
  Use `build/test/functional/<test>.py`.
- `SkipTest: no IPC support`: missing Python capnp dependency. Install
  `pycapnp`.
- `Unable to bind to IPC address ... Operation not permitted`: sandbox
  permission denial. Rerun the same command with escalation.
- `Unable to start HTTP server` or RPC bind failure in `debug.log`: sandbox
  network bind denial. Rerun the same command with escalation.
- `FileExistsError: ... tmpdir ... exists`: stale fixed tmpdir. Remove that
  exact tmpdir and rerun the same command; do not work around it by inventing
  a new numbered tmpdir.

## Remote build check

When local toolchains do not reproduce a CI compiler diagnostic, try a remote
build on `copilot`.
If syncing first, use `rsync -avL` when `CMakeUserPresets.json` is a local
symlink so the remote receives file contents rather than a broken symlink.

## Remote commit-by-commit verification

Use `copilot` when a CI ancestor-commit job fails, or when a rewritten stack
needs validation against a Linux/CI-like toolchain. Prefer one reusable remote
worktree for the PR path instead of creating numbered sibling directories, so
the build cache remains useful.

Recommended shape:

1. Fetch or transfer the exact candidate stack to the remote. For local-only
   rewrites, create a bundle for the range and fetch it into a temporary remote
   branch:
```bash
git bundle create ./tmp/<topic>.bundle <base>..HEAD
rsync -av ./tmp/<topic>.bundle copilot:<remote-repo>/tmp/
ssh copilot 'cd <remote-repo> && git fetch tmp/<topic>.bundle +HEAD:refs/heads/codex/<topic>'
```

2. Run the verifier in `tmux`, using the PR number as the session name when the
   repo path is `bitcoin-pr/<PR>-...`. Keep logs under the remote repo's
   `tmp/` directory:
```bash
ssh copilot tmux new-session -d -s <pr-number> '/bin/bash -lc "<remote-repo>/tmp/verify.sh 2>&1 | tee <remote-repo>/tmp/verify.log"'
```

3. For each commit under test, check it out detached, merge the current target
   branch without committing when reproducing GitHub's ancestor-commit job, then
   build and test:
```bash
git checkout --detach <commit>
git merge --no-commit origin/master
cmake -B ci_build_repro <ci-like-options>
cmake --build ci_build_repro -j <jobs>
ctest --output-on-failure --stop-on-failure --test-dir ci_build_repro -j <jobs>
```

4. Run any targeted functional tests from the build tree after `ctest`, with
   fixed tmp/cache paths under the remote repo's `tmp/` directory:
```bash
REALPWD=$(pwd -P)
env PWD="$REALPWD" TMPDIR="$REALPWD/tmp" "$REALPWD/ci_build_repro/test/functional/<test>.py" \
  --tmpdir="$REALPWD/tmp/func-<test>-<commit>" \
  --cachedir="$REALPWD/tmp/cache-functional"
```

5. Reset after each commit before advancing:
```bash
git reset --hard
```

If the remote host lacks an exact CI tool such as `mold`, document the omitted
flag in the final result and keep the rest of the CI-like compiler/warning
configuration intact.
