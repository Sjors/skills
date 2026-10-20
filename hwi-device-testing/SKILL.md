---
name: hwi-device-testing
description: Build, install, cache, and run HWI hardware-wallet simulators locally on Ubuntu. Use for device tests, simulator setup, full device sweeps, or selected hardware-wallet test methods.
---

# HWI device testing on Ubuntu

Run locally as the current user. Read the checkout's local `AGENTS.md` for
machine-specific paths and available tooling. Keep those choices separate from
this skill: Python installations, repository locations, storage mounts, and
container stores differ between machines.

The shared helpers live in [dev-utils](https://github.com/Sjors/dev-utils#hwi-simulator-helpers).
Use `${HWI_UTILS_ROOT:-$HOME/dev-utils}` for its local checkout. Skill-owned
executable helpers belong there, alongside their tests and documentation.

## Resolve the environment and assets

1. Identify the actual HWI worktree and preserve its uncommitted changes.
   Read `pyproject.toml`, `test/run_tests.py --help`, `test/setup_environment.sh`,
   `.github/actions/build-sim/action.yml`, and `.github/actions/install-sim/action.yml`.
   For Ledger also read both app-builder workflows. Derive firmware commits,
   SDK versions, Python constraints, and required patches from this branch.
2. Check for active test runners and occupied simulator endpoints before starting.
   The fixed endpoints are listed in [parallel and targeted tests](references/parallel-and-targeted.md).
   An unrelated Bitcoin node is not itself a conflict; the harness uses an
   independent regtest datadir and dynamic node ports.
3. Choose user-owned locations for build trees, caches, and installed simulators.
   Honor local configuration. Useful portable defaults are:

   ```sh
   export REPO_ROOT="$(git rev-parse --show-toplevel)"
   export HWI_CACHE_ROOT="${HWI_CACHE_ROOT:-${XDG_CACHE_HOME:-$HOME/.cache}/hwi}"
   export HWI_BUILD_ROOT="${HWI_BUILD_ROOT:-$HWI_CACHE_ROOT/build}"
   export HWI_SIM_STORE="${HWI_SIM_STORE:-${XDG_DATA_HOME:-$HOME/.local/share}/hwi/simulators}"
   ```

4. Resolve an existing exact build before installing or rebuilding anything.
   Store assets under `$HWI_SIM_STORE/<device>/<version-id>/`, preserving the
   runtime's relative layout. Replace `/` in tag names when naming directories.
   Ledger's key must identify both Speculos and the app version; record the SDK
   too. Keep build-input hashes/provenance so changed HWI patches are noticed.
   If necessary, add a build-input suffix instead of overwriting a validated asset.
5. Verify candidates using their source commit, applicable HWI patches, native
   executables/resources, architecture, shared libraries, and a smoke test.
   A directory name or `simulator.py` alone does not establish compatibility.
   Inventory the current user's older build trees before rebuilding. Keep the
   source, submodules, compilers/SDKs, and generated runtime resources available
   locally. Reuse validated fixes in those trees.

Prefer native builds. Before installing or rebuilding simulators, follow the
[system package sanity check](#system-package-sanity-check) below. Read
[building and installing](references/building.md) for the relevant device.
Build from source when setting up local tooling; CI artifacts are useful when
explicitly reproducing an artifact-specific failure, not a substitute for that setup.

Some HWI setup targets reset sources, reapply patches, delete virtual environments,
or remove tool directories. Use a fresh per-device/version build workspace for
initial setup and resume the needed build step after a failure. Preserve verified
builds before changing a shared staging path. When copying HWI into a workspace,
include the current working-tree changes and test data; exclude `.git`, `.venv`,
`build/`, and `test/work/` rather than copying another user's installation.

## System package sanity check

For a native build, run the read-only dev-utils checker for the selected devices:

```sh
"${HWI_UTILS_ROOT:-$HOME/dev-utils}/hwi-check-prereqs.py" --device trezor-t
# Omit --device for a full installation sweep; repeat it for multiple devices.
```

If recommended packages are missing, show the generated apt command and ask the
user to install them as root, unless installation is already authorized. Do not
silently choose a container solely to avoid asking. The checker recognizes
versioned libclang packages and Ubuntu's QEMU packaging alternatives. Its exit
status is 0 for installed prerequisites, 1 for missing prerequisites, and 2 for a
check failure. Check failures need diagnosis rather than an empty install list.
Read [system prerequisites](references/prerequisites.md) for the preferred path
and version checks. Recheck after installation.

If root packages cannot or will not be installed, use the separate
[workarounds](references/building.md#workarounds-without-system-package-installation).
Existing user toolchains can already satisfy missing distro packages; confirm
that before installing replacements. Record that choice locally and avoid
repeating an installation prompt already answered. Continue independent tests
while waiting for a root installation. When reusing a cached simulator, check
its actual runtime libraries with `ldd` and its Python imports; do not require the
whole native build package set just to run tests.

## Python and Bitcoin Core

Choose a Python supported by this HWI branch and the selected simulator. Use a
checkout-specific HWI venv and separate build environments when dependencies
conflict. `uv` can supply an appropriate Python without changing the system
interpreter. Reinstall HWI after source edits; a normal install is sufficient.

```sh
export PATH="$REPO_ROOT/.venv/bin:$PATH"
uv pip install --python "$REPO_ROOT/.venv/bin/python" "$REPO_ROOT"
```

Several harnesses launch a bare `python3`, so the venv must be first in `PATH`.
Install the simulator's runtime Python requirements in this same environment.
A venv created inside a container belongs to that interpreter and container path
layout; use a separate host venv for native tests.

Select an explicit, verified `BITCOIND` path. Probe with
`-version -nosettings -conf=/dev/null` and an explicit temporary `-datadir`.
Even a version query can otherwise try writing default settings. Check that its
RPC/descriptor features satisfy the branch's tests. Use an existing suitable
user-owned binary before building another. Development tests can require
features absent from a recent release. The setup script may follow an unpinned
Core branch, so record the binary's exact version. Let the harness start its own
node; any manual probe must use an explicit temporary regtest datadir.

## Run the requested coverage

All non-device tests are selected by the runner without device flags:

```sh
cd "$REPO_ROOT/test"
python run_tests.py
```

For a device, pass its explicit verified simulator path, `--device-only`, the
selected interface, and `--bitcoind`. For ordinary source-tree validation use
`--interface=library`. Run another interface when requested or reproducing its CI
failure. Distribution validation follows the branch's test-dist action.

```sh
python run_tests.py --coldcard-edge --device-only --interface=library \
  --coldcard-edge-path="$SIMULATOR" --bitcoind="$BITCOIND"
```

For a full sweep, invoke each device separately and retain a result for each.
Run suites one at a time by default. `--all` can short-circuit after a failure,
and default build locations are not a multi-version simulator cache.

Use independent writable runtime copies for simulators that save state. Never
clear registered policies or flash images in a shared validated asset. The
Coldcard helper and other runtime layouts are described in the build reference.

Capture runner output and emulator traces. Several harnesses write traces into
the current test directory and replace them on the next run. For large traces,
choose a suitable test workspace containing the current test code and `data/`,
then use absolute Python/simulator/node paths and `--device-only`. Preserve or
compress completed traces between suites. Confirm expected signing and
version-specific tests actually ran; a green empty or reduced suite is insufficient.

For selected methods, read [parallel and targeted tests](references/parallel-and-targeted.md).
For CI-only failures, read [matching the CI environment](references/ci-container.md).
Report failures, expected skips, and the scope tested. After an installation sweep,
measure allocated space with `du` and remaining filesystem capacity with `df`;
include runtimes, build trees, toolchains, images, caches, and retained logs without
double-counting shared image layers. Record local paths in a Git-ignored `AGENTS.md`
or existing machine configuration, and keep publishing within the user's instructions.
