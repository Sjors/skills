---
name: bitcoin-core-tests
description: Run Bitcoin Core functional and unit tests locally.
---

# Skill Instructions

## When to use
After modifying Bitcoin Core C++ or Python test code, including during and after a rebase.

## Environment setup

1. **Clear stale test cache** when encountering `AssertionError: not(0 == 199)`:
```bash
rm -rf build/test/cache
```

2. **macOS only — raise file descriptor limit** (default is too low; bitcoind needs at least 160 fds per node):
```bash
ulimit -n 10240
```

## Running functional tests

Always run from the build directory so `config.ini` is found:
```bash
build/test/functional/<test>.py
```

To run multiple tests, use ~2x CPU thread count for concurrency (e.g. `-j32` on a 16-thread machine):
```bash
build/test/functional/test_runner.py -j32 <test1>.py <test2>.py
```

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

- **Do not run functional and unit tests in parallel.** They share `build/test/cache` and will corrupt each other's state, causing spurious `0 == 199` failures.

## Common issues

| Symptom | Cause | Fix |
|---|---|---|
| `Not enough file descriptors available. -1 available, 160 required` | fd limit too low (common on macOS) | `ulimit -n 10240` |
| `AssertionError: not(0 == 199)` | Stale test cache | `rm -rf build/test/cache` |
| `FileNotFoundError: .../test/config.ini` | Running from source dir instead of build | Use `build/test/functional/<test>.py` |
| `SkipTest: no IPC support` | Missing Python capnp dependency | `pip install pycapnp` |
