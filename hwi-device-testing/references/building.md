# Building and installing simulators

Use the pins and recipes in the selected HWI checkout. The locations below are
relative layouts, not assumptions about the login name, repository location, or
storage hardware. Keep build-only files separate from installed runtimes.

| Runner flag | Pin/build inputs | Required runtime layout |
|---|---|---|
| `--bitbox01` | `BITBOX01_VERSION`, HWI CMake flags | `mcu/build/bin/simulator` plus `mcu/tests/sd_files/` |
| `--coldcard` | `COLDCARD_VERSION`, HWI multisig/MicroPython patches | `firmware/unix/`, native `coldcard-mpy`, and dereferenced `firmware/shared/` |
| `--coldcard-edge` | `COLDCARD_EDGE_VERSION`, Edge fixtures/patches | Same layout as Coldcard, separate build and state |
| `--keepkey` | `KEEPKEY_VERSION`, HWI KeepKey/googletest/nanopb patches | `keepkey-firmware/bin/kkemu` |
| `--trezor-1` | `TREZOR_VERSION`, legacy build settings | `trezor-firmware/legacy/firmware/trezor.elf` |
| `--trezor-t` | `TREZOR_VERSION`, matching Rust nightly, HWI build patch | `core/emu.sh`, `core/src/`, and `core/build/unix/trezor-emu-core` |
| `--ledger` | `SPECULOS_VERSION`, modern app/SDK workflow pins | Speculos Python package/resources, `speculos.py`, `apps/btc-test.elf` |
| `--ledger-legacy` | Same Speculos pin, legacy app/SDK workflow pins | Same layout, `apps/btc-test-legacy.elf` |
| `--jade` | `JADE_VERSION`, firmware Dockerfile's ESP-IDF pins | `qemu-system-xtensa`, `flash_image.bin`, `qemu_efuse.bin`, `pc-bios/` |
| `--bitbox02` | `BITBOX02_VERSION`, `.containerversion` | `bitbox02-firmware/build-build-noasan/bin/simulator` |

The runner has one `--ledger-path` option for either Ledger mode. Other path
options follow the corresponding device flag; check `--help` for the branch.

## Native builds

Follow the [system package check and preferred installation path](prerequisites.md)
before choosing a workaround. The build-sim action lists branch prerequisites.
Legacy CMake projects can reject CMake 4; use a compatible CMake 3 in a user build environment, or a narrowly
applicable policy minimum when appropriate. Avoid changing the global Python or
compiler defaults merely to build one simulator.

## Workarounds without system package installation

Use these when the user declines installation, has no root access, a package has
no candidate on this Ubuntu release, or installed versions remain incompatible.
Check existing user-owned toolchains first. Keep a fallback scoped to the affected
build or test rather than changing global defaults.

| Missing or unsuitable dependency | Workaround |
|---|---|
| SDL/libslirp/libusb or other C development headers | Build in `hwi-build-env`; Python wheels do not supply general C headers. |
| SDL runtime libraries | Coldcard can use `pysdl2-dll` in its runtime venv. For Trezor T and other native binaries, use a matching container with the libraries; an outer `LD_LIBRARY_PATH` may be stripped by the harness. |
| Python venv support or headers | Use a supported user-managed Python via `uv`, including its own headers, or the builder image. Keep all pip packages in venvs. |
| protobuf compiler/descriptors | Install the branch-pinned protoc archive, including `include/`, in a user tool directory. Set `PROTOC_BINARY` and `PATH` explicitly. |
| libclang or compatible GCC | Select an existing versioned compiler/library explicitly, or use the builder image. Keep Rust's matching nightly in user-owned toolchains. |
| CMake 4 rejects legacy projects | Install CMake 3 in a separate user build venv, or use the builder image. |
| Rootless container support is unavailable too | Use native tools and library prefixes with the build's supported include/library options. If that cannot satisfy the pinned recipe, report the specific blocker and request the relevant host setup; do not replace local builds with downloaded CI artifacts. |

The reusable container fallback is `hwi-build-env` in dev-utils:

```sh
"${HWI_UTILS_ROOT:-$HOME/dev-utils}/hwi-build-env" --build-image
"${HWI_UTILS_ROOT:-$HOME/dev-utils}/hwi-build-env" \
  --repo "$REPO_ROOT" --work "$HWI_BUILD_ROOT/trezor-1" \
  -- ./setup_environment.sh --trezor-1
```

It supplies Ubuntu 24.04's GCC 12/13, SDL development libraries, protobuf
**descriptor headers**, libclang, Python build tooling, and Rust's launcher.
The repo is read-only at `/hwi`, while `--work` is writable at `/hwi/test/work`.
Caches and installed Rust/Python toolchains persist outside the container.
Use its `--help` and the dev-utils README for configurable cache, toolchain,
image-store, and runtime-state locations. Large image pulls also use temporary
space: the launcher defaults `TMPDIR` to its selected cache unless already set.
Use the same setting for direct SDK image pulls.

Keep this generic build image and each device's required SDK images locally.
The generic image does not replace the pinned BitBox02 or Ledger app builders.
Use rootless containers so mounted build outputs remain manageable by the current
user. Retain the actual source checkout as well as the resulting executable.

## Coldcard and Coldcard Edge

Resolve each exact pin independently; both HWI setup targets use the staging
name `work/firmware`. Preserve a validated build before switching that path.
Prefer HWI's setup recipe over the firmware's `make setup`: HWI includes the
required submodule/libffi/MicroPython workarounds.

Use a compatible compiler throughout sub-makes. GCC 13 is known to run these
simulators; GCC 14/15 builds can compile and then die with signal 11. A successful
compile is not a smoke test. Make sure the native `coldcard-mpy` target exists
and runs, not just the Python front end.

For a simulator build/run, its `unix/requirements.txt` supplies PySDL2 and Pillow;
`pysdl2-dll` can supply SDL runtime libraries. The top-level firmware requirements
also include unrelated legacy development/smartcard tools. When those fail,
follow HWI's build steps with the simulator requirements in a disposable build
workspace. Edge already uses that narrower set in current HWI. Mirror the
branch's install-sim action when reproducing CI, rather than assuming all modes
use the same environment.

After building, use the runtime helper rather than copying the full source or
sharing hardlinked mutable state:

```sh
"${HWI_UTILS_ROOT:-$HOME/dev-utils}/hwi-coldcard-runtime.py" \
  "$BUILT_FIRMWARE" "$STORED_FIRMWARE"
"${HWI_UTILS_ROOT:-$HOME/dev-utils}/hwi-coldcard-runtime.py" \
  "$STORED_FIRMWARE" "$RUN_FIRMWARE"
```

Each destination must be new. The helper copies `unix`, dereferenced `shared`
files, and the native binary, records provenance, and creates empty
`unix/work/{MicroSD,VirtDisk,debug,settings}` directories. Pass
`$RUN_FIRMWARE/unix/simulator.py` with the appropriate path flag. Check the full
Edge output includes transaction signing and Taproot address display.

Edge persists BIP388/Miniscript registrations in `unix/work/settings/*.aes`.
Reusing a policy name can stall on an "already exists" screen and later surface
as the misleading "handling another request" (`-13`). Create another runtime
copy for repeated policy runs. Deleting `work` without recreating the skeleton
also prevents startup. Firmware `--segregate` changes the socket name and needs
corresponding harness support.

Edge's policy path differs from standard Coldcard's legacy multisig enrollment.
For policies reusing a device fingerprint/account family across Taproot leaves,
display/registration only verifies policy structure. A script-path signing test
must select that path, satisfy its sequence/locktime, finalize without the Core
key-path participant when appropriate, and broadcast.

## Trezor One and Trezor T

Use HWI's pin for the entire firmware checkout, including the legacy model.
Trezor One uses its GCC 12/headless/debug-link setup. SDL development headers and
protobuf descriptors (`google/protobuf/descriptor.proto`) are build inputs;
installing just the `protoc` executable package may omit those descriptors.

Trezor T additionally needs libclang for Rust bindgen. Read its source's
`shell.nix` and HWI's setup recipe for the expected Rust nightly and components.
Use a scoped `RUSTUP_TOOLCHAIN`/override, or container-scoped Rust configuration,
so the host default is preserved. Apply the branch's Trezor build patch where
required. The rolling nightly can reject this source even when a pinned nightly
builds. Resume the relevant `uv`/make step rather than rerunning a destructive
setup target on a validated checkout.

Retain the legacy ELF for One. For T, keep the original relative `emu.sh`, `src`,
and `build/unix` layout. Use writable per-run copies for flash/profile state.
Inspect `ldd`: T can require `libSDL2_image` even when One runs with just SDL2.
The HWI Trezor harness supplies a minimal emulator environment, so merely setting
`LD_LIBRARY_PATH` on the outer test runner may not reach it. A matching local
container with the required libraries is a straightforward fallback. Keep the
node inside that test container too, with a compatible explicit binary.

Trezor T's signing and mnemonic/key-vector tests can take many minutes. Check
emulator log activity and process state before treating a quiet runner line as
a hang; keep a bounded test timeout.

## KeepKey

Follow the branch's three build patches: KeepKey, googletest, and nanopb. The
legacy generator uses nanopb 0.3.9.4 in current HWI. Keep its Python build deps
separate from HWI's runtime: protobuf 3.20.3 plus requests works with this
legacy source. Use the version appropriate to another branch if its recipe changes.

Use the protoc version from the build-sim action (22.0 for the validated recipe),
including its `include` tree. Install it in a user tool directory, put its `bin`
in `PATH`, and pass its actual absolute path as `PROTOC_BINARY`; HWI's script may
assume `/usr/local/bin/protoc`. Generate nanopb's proto module before configuring
KeepKey, and put `nanopb/generator` in `PATH` for CMake. A compatible CMake 3 and
GCC/G++ 13 build the emulator natively. Retain the source/submodules for rebuilding;
only `bin/kkemu` is needed in the runtime copy. Its writable emulator image should
be created outside the stored asset.

Git's shallow recursive clone can fail on old Savannah lwIP URLs. When needed,
use Git's GitHub lwIP mirror rewrite scoped to the build command/configuration,
following HWI CI. Avoid changing unrelated global Git configuration.

## BitBox01

Use the pinned `mcu` source and HWI's CMake simulator flags. With CMake 4, old
nested projects may need `CMAKE_POLICY_VERSION_MINIMUM=3.5`; a compatible CMake 3
also works. Keep `build/bin/simulator` and `tests/sd_files` in their original
relative layout: the harness passes `../../tests/sd_files/` from the binary's
directory. A binary copied alone is an incomplete runtime.

## Ledger and Ledger legacy

Build Speculos at `SPECULOS_VERSION`. Its native launcher uses the ARM cross
compiler, OpenSSL, and blst; retain the generated
`speculos/resources/launcher`. Install its declared Python runtime dependencies
in the HWI test environment. Keep the Speculos Python package/resources alongside
the `speculos.py` entry point, or install a matching built package. Python package
installation itself can invoke CMake; account for that build environment. Preserve
`.git` when installing from a tree that needs setuptools_scm version detection.

On GCC 15, old blst can fail on `unterminated-string-initialization` or
`maybe-uninitialized` warnings promoted to errors. Export the narrowly applicable
`-Wno-error=...` flags through **CFLAGS for configure and build**, preserving its
normal optimization/PIC flags. `CMAKE_C_FLAGS` alone does not reach blst's shell
build. A failed build may leave an empty archive marked as built; rebuild that
dependency and relink, rather than trusting the old stamp. Keep adaptations in
the disposable simulator build, or use the compatible container compiler.
When building dependencies in parallel, ensure their shared install directory
exists before blst's install step.

Build each app with its own workflow's repository, ref, builder image, SDK, model,
and flags. Current modern HWI uses the `app-bitcoin` repository; the legacy
workflow uses `app-bitcoin-legacy`. Clone sources into retained user build trees
and mount them into the pinned official builder image instead of cloning only
into an ephemeral container. The workflows select `NANOX_SDK` for modern Ledger
and `NANOSP_SDK` for legacy; check them again when the pin changes.

The HWI harness locates `apps/btc-test.elf` or `apps/btc-test-legacy.elf` relative
to the entry point. Speculos itself has no Coldcard-style persistent policy state,
so a verified runtime can be used in place. Run modern and legacy suites serially.

For a review-screen stall followed by `0x6901`, inspect the last
`automation: getting actions for` titles in `ledger-emulator.stderr` and compare
`test/data/speculos-automation.json`. New app informational titles can require
an automation update. Confirm the actual screen before adding a rule and rerun
the full affected suite. `0x6a82` can indicate the wrong or incompatible app.
Speculos debug traces can be large; preserve/compress them between suites.

## Jade

Read ESP-IDF's branch/commit from the pinned firmware's Dockerfile. Retain its
source, downloaded tools, and Python environment. HWI's fresh setup target may
remove the tools directory; after a failure resume the required install/build
step instead of downloading everything again.

**Bootstrap ESP-IDF with a base Python, outside a venv.** Selecting the HWI venv's
interpreter makes `install.sh` refuse to create its own environment. Obtain the
underlying interpreter through `sys.base_prefix` or `uv python find`, select it
first in `PATH`, and unset inherited `VIRTUAL_ENV` for bootstrap. Keep an explicit
`IDF_TOOLS_PATH`, then source ESP-IDF's `export.sh` for the actual build. Fetch the
exact IDF commit if a shallow branch checkout does not contain it.

Install `qemu-xtensa`, build with `configs/sdkconfig_qemu.defaults`, and merge the
firmware/bootloader/partition/OTA binaries exactly as HWI's recipe specifies.
`flash_image.bin` is a build output; `qemu_efuse.bin` alone is insufficient.
Copy the four runtime components from the table and check QEMU's shared-library
requirements. Both flash and efuse files must be writable in a per-run copy.
For a port-forward error, inspect the existing QEMU owner/process before stopping
anything; do not broadly kill another run.

## BitBox02

Use the firmware pin and the exact `.containerversion` image from that source.
The version generator needs reachable signed firmware, firmware-btc-only, and
bootloader tags. A shallow firmware-tag-only clone can fail with "No signed tags
found with prefix bootloader". Fetch the needed history/tags, preserving HEAD,
instead of bypassing version checks.

Build `make simulator` in the pinned `shiftcrypto/firmware_v2` image using a
user-owned source mount and an appropriate job limit. Fully qualify registry
names when the container tool requires it. Mirror CI's scoped safe-directory
setting inside the container for the mounted source. Retain both the image and
source; the runtime is `build-build-noasan/bin/simulator`. Verify its libraries
and use a separate writable runtime copy for tests.
