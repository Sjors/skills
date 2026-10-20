# Serial, parallel, and targeted runs

Use one device suite at a time for ordinary installation sweeps or when requested.
The harness creates a separate bitcoind datadir with dynamic RPC/P2P ports, but
simulator endpoints are fixed in the current HWI harness:

| Family | Endpoint | Conflict |
|---|---|---|
| Coldcard / Coldcard Edge | `/tmp/ckcc-simulator.sock` | Any other Coldcard-family run |
| Ledger / Ledger legacy | TCP 9999 | Any other Ledger-family run |
| Trezor One / Trezor T | UDP 21324 | Any other Trezor model run |
| Jade | TCP 30121 | Another Jade run |
| BitBox02 | TCP 15423 | Another BitBox02 run |
| BitBox01 | UDP 35345 | Another BitBox01 run |
| KeepKey | UDP 11044 | Another KeepKey run |

If parallelism is requested, distinct rows can coexist with separate writable
runtime directories and suitable resources. Use isolated Python environments
for different branches so installing HWI cannot replace another run's package.
Check processes and endpoints after interrupted runs. Stop only the process
belonging to the affected run; do not use an unqualified `pkill` against a family.

## Selected methods

Probe `python run_tests.py --help` in the checkout's test environment. Prefer
its native repeatable `--test` option when available. The relevant HWI implementation
implies `--device-only`, rejects unmatched method names, and preserves the normal
simulator lifecycle. Confirm the exact branch's behavior before using it.

```sh
cd "$REPO_ROOT/test"
python run_tests.py --ledger \
  --test test_segwit_miniscript --test test_taproot_miniscript \
  --ledger-path="$SIMULATOR" --bitcoind="$BITCOIND" --interface=library
```

For branches without native filtering, use dev-utils from the HWI test directory:

```sh
python "${HWI_UTILS_ROOT:-$HOME/dev-utils}/hwi-run-targeted-device-tests.py" ledger \
  test_segwit_miniscript test_taproot_miniscript \
  --simulator="$SIMULATOR" --bitcoind="$BITCOIND" --interface=library
```

Use exact method names or quote method-name globs. The helper filters the
assembled suite, including device-specific Trezor/KeepKey classes, and rejects
every unmatched pattern before running tests. Existing constructors, setup,
teardown, and device-specific suite composition are preserved. This avoids
editing HWI suite-building functions for a local targeted run.

A method name can occur in multiple classes: inspect the reported test IDs and
count. Repeated Edge policy tests need a fresh runtime even when selecting just
one method. Selected tests do not replace the full device suite when full
validation was requested.
