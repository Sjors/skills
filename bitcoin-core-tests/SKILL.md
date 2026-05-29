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

## Environment setup

1. **macOS only — raise file descriptor limit** when running functional tests
   with several nodes:
```bash
ulimit -n 10240
```

2. **Prepare a test temp root.** Prefer the RAM disk for test tmp/cache
   paths. The helper clears it before use unless the user explicitly asks to
   preserve it, creates the standard 11 GiB RAM disk when absent and enough RAM
   is free, and falls back to the private temp directory when memory is tight.
```bash
TEST_TMP_ROOT="$(~/utils/bitcoin-test-temp-root.sh)"
```

Use the RAM disk only for test tmp/cache paths, not for the build tree or
build artifacts.

If preserving the existing RAM disk contents is intentional, use:
```bash
TEST_TMP_ROOT="$(~/utils/bitcoin-test-temp-root.sh --preserve)"
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

Derive paths from the physical checkout path to avoid symlink surprises:
```bash
REALPWD=$(pwd -P)
```

Invoke the built functional-test scripts from the repo root so
`build/test/config.ini` is found:
```bash
build/test/functional/<test>.py
```

For reliable sandbox/debug runs, prefer the main runner with stable paths
under `TEST_TMP_ROOT`:
```bash
rm -rf "$TEST_TMP_ROOT/testrunner"
env PWD="$REALPWD" TMPDIR="$TEST_TMP_ROOT" "$REALPWD/build/test/functional/test_runner.py" -j 1 --tmpdirprefix="$TEST_TMP_ROOT/testrunner" --cachedir="$TEST_TMP_ROOT/cache-functional" --failfast --combinedlogslen=200 <test>.py
```

When invoking a single test directly, use a fixed per-test tmpdir and the same
cache path. Delete that fixed tmpdir before each run; do not create numbered
siblings like `func-<test>-run2` unless the user explicitly asks to preserve
the failed directory for investigation:
```bash
rm -rf "$TEST_TMP_ROOT/func-<test>"
env PWD="$REALPWD" TMPDIR="$TEST_TMP_ROOT" "$REALPWD/build/test/functional/<test>.py" --tmpdir="$TEST_TMP_ROOT/func-<test>" --cachedir="$TEST_TMP_ROOT/cache-functional"
```

To run multiple tests outside sandbox triage, use appropriate concurrency for
the machine:
```bash
rm -rf "$TEST_TMP_ROOT/testrunner"
env PWD="$REALPWD" TMPDIR="$TEST_TMP_ROOT" "$REALPWD/build/test/functional/test_runner.py" -j <jobs> --tmpdirprefix="$TEST_TMP_ROOT/testrunner" --cachedir="$TEST_TMP_ROOT/cache-functional" <test1>.py <test2>.py
```

When tests need previous releases, remember that Bitcoin worktrees on this
host normally recreate the ignored local `releases` symlink through the shared
`post-checkout` hook. Do not infer that a verifier worktree is missing
`releases` just because it is ignored/untracked; check the actual worktree
after `git worktree add`/`git checkout`, or pass
`PREVIOUS_RELEASES_DIR=<path>` explicitly if the hook is unavailable.

## Running unit tests

Via ctest:
```bash
env TMPDIR="$TEST_TMP_ROOT/unit" ctest --test-dir build -R <test_name> --output-on-failure
```

Or directly:
```bash
env TMPDIR="$TEST_TMP_ROOT/unit" build/bin/test_bitcoin --run_test=<suite>/<case>
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
  fd limit too low. On macOS, run `ulimit -n 10240`.
- `AssertionError: not(0 == 199)`: stale test cache. Clear the active cache
  path, usually `build/test/cache` or `$TEST_TMP_ROOT/cache-functional`.
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
