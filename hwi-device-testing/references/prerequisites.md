# System prerequisites on Ubuntu

## Preferred: install missing system packages

Run `hwi-check-prereqs.py` from the dev-utils checkout before a new native build.
It reads dpkg state and local apt metadata without installing or downloading
anything. Use `--device NAME` repeatedly to scope the check, or no flags for all
ten devices. Compare its baseline with the selected branch's build-sim action,
setup script, firmware sources, and SDK workflows; those are authoritative when
requirements change.

The checker covers the shared C/C++ build and Python bootstrap packages, then
adds device-specific dependencies: SDL2/SDL2_image and protobuf for Trezor,
libclang for Trezor T, GCC 12 for Trezor One, GCC 13 for Coldcard/KeepKey, the ARM
cross compiler and QEMU for Speculos, and SDL/libslirp plus QEMU build libraries
for Jade. BitBox02 and Ledger app builds still use their pinned SDK images.
`uv`, Poetry, SCons, Rust, and firmware-specific Python packages belong in
user-owned environments; do not install them with sudo pip.

For missing packages, ask the user to run the checker's generated command as
root (or with sudo). It prints only missing packages with candidates in local apt
metadata. If the metadata is stale, refresh it and rerun the checker. An absent
candidate, particularly an older versioned GCC on a newer Ubuntu release, calls
for the workaround section rather than an apt command that cannot succeed.
Recheck after installation and record local paths/tool versions in the ignored
`AGENTS.md`.

The Python development/venv packages support the distro's Python. A supported
user-managed Python may already provide those capabilities, so these are baseline
recommendations rather than proof that an existing environment cannot build.
Likewise an explicit local protoc can replace the distro executable. Confirm
such alternatives and avoid redundant installation requests.

## Verify versions and runtime libraries

Package presence alone does not establish build compatibility. Inspect the tools
actually selected by the build's `PATH`, not just `/usr/bin`:

```sh
cmake --version
protoc --version
gcc-12 --version
gcc-13 --version
```

Run only the probes relevant to the selected device. Recent Ubuntu releases can
ship CMake 4 while legacy firmware needs CMake 3. KeepKey's validated recipe needs
protoc 22 with its include tree; Ubuntu's protobuf-compiler can be another version.
Select compatible versions locally even after installing system prerequisites.
For Rust bindgen, a versioned `libclang-<N>-dev` can satisfy the development
package requirement without the `libclang-dev` metapackage; set `LIBCLANG_PATH`
when its loader does not discover that installation.

Ubuntu releases differ in whether `qemu-arm-static` comes from `qemu-user-static`
or `qemu-user`. The checker accepts either installed package, chooses a candidate
available in the local apt metadata, and also checks for the actual executable
required by the current Speculos recipe. Do not substitute the dynamic
`qemu-arm` blindly for a hardcoded static executable.

For an existing simulator, inspect its ELF dependencies with `ldd` and smoke-test
it in the environment the harness uses. Install only missing runtime libraries
when no rebuild is needed. For example, Trezor T can need
`libsdl2-image-2.0-0` even when Trezor One already works with SDL2. Development
packages pull in those runtime libraries for native builds. Package names and
ABI suffixes can vary across Ubuntu releases; use the current apt metadata and
actual missing library instead of copying an older release's runtime package list.

If installation is unavailable or declined, continue with
[workarounds without system package installation](building.md#workarounds-without-system-package-installation).
