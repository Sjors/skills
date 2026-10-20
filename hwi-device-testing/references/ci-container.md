# Matching a CI-only failure

Read the selected branch's `device-test.yml`, install-sim action, test-dist
action, and simulator/app build workflows. Match the failing Python image,
interface, firmware/app pins, dependencies, and install method. CI's official
Python container can have a different distribution, glibc, and dependency set
from the Ubuntu host or the generic local build image.

Run both the HWI harness and its regtest node inside the test container. Mount
the current HWI tree, a verified simulator runtime, and a compatible explicit
bitcoind binary. Install HWI and the branch's runtime requirements there. The
harness writes traces into its working test directory, which must be writable.
A host binary can require a newer `GLIBC_*` than the image supplies; use a
compatible release/depends build or build against the target environment.

Keep container engines, image stores, cache locations, and toolchain paths in
local configuration. Rootless containers can provide the required system
packages without host sudo. Use the environment's configured proxy/CA trust
when applicable, retaining TLS verification.

For wheel/sdist/bindist failures, follow test-dist exactly. Install one artifact
at a time with the right interpreter and use the prescribed CLI/bindist mode.
A cached bindist builder may have the wrong pinned Python; rebuild its current
Dockerfile instead of patching the old image by hand. Ordinary source validation
usually needs just the library interface.

Compare the installed dependency and node versions rather than assuming a
fresh host install matches CI. HWI's configured development Core branch can
advance independently: compare a suitable release and the exact development
binary when signing behavior changes, including PSBT versions/locktime behavior.
Use the same temporary regtest setup for comparisons.

CI's exact simulator artifact can help isolate an artifact-specific regression
when requested. Record its originating run, source pins/build inputs, digest,
and libraries; do not infer provenance from its filename. Building and retaining
local source/toolchains remains the workflow for a local setup request.

If a generic assertion hides a device result, inspect that result and the
emulator trace. Keep temporary instrumentation out of the final HWI change.
