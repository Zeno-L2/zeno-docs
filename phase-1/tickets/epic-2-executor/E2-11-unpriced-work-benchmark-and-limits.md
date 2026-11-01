# Ticket E2-11 — Unpriced-work benchmark and limits

**Epic:** 2 — Executor · **Phase:** 1A · **Order:** E2-11

## Goal
Build the benchmark suite that SPEC §30's resolution rule for `OD-4` and `OD-5` needs. Set the
measurement method for every operation those open designs name. Stage the results table and the
`metering_version = 2` amendment in `wasm-vm/wasm-gas-table-v2-draft.md`.

The ticket delivers the suite, the method decisions and the draft. The `-count 10` pass on
reference executor hardware fills the draft's `[TBD]` cells. `OD-4` and `OD-5` stay open in
SPEC §30 until that pass classifies every operation. The state-size scaling the suite exposes
is removed in the store by E2-12, not priced in the gas table.

## SPEC refs
SPEC §10.1 (pre-execution charging), §10.2 (gas-table amendment procedure), §10.6 (where each
charge falls), §14.3 (`MaxResultBlob`), §25.1 (structural caps, `MaxInstrumentedCodeSize`),
§30 `OD-4`, `OD-5` and their resolution rule, §31 (conformance artifacts).

## Depends on
E2-09 (SPEC conformance). The decision numbers are meaningful only on the reference executor
hardware, the machine class that runs testnet.

## Key work

### Resolution rule
An operation gains a `metering_version = 2` term when its measured cost at its consensus bound
exceeds **2×** the time bought by the least gas that triggers it. Otherwise it is recorded as an
accepted bounded cost under `metering_version = 1`, with its measured ratio. The time bought is
gas × `ns_per_gas`, from the calibration below.

### Calibration (`plugins/wasm/bench_test.go`)
`BenchmarkCalibrateNsPerGas` runs four children, `cost-1`, `cost-2`, `cost-5` and `cost-10`.
Each loops one opcode pattern `calibrationReps` (32768) times in a warm compiled module. The
gas each child reports excludes `module.InitialMemoryCost(1)`. Each child reports `gas/op` and
`ns/gas`; `ns_per_gas` is the median `ns/gas`.

### Operations at their bound (`plugins/wasm/bench_test.go`)
| Item | Benchmark | Bound |
|---|---|---|
| `OD-4` | `BenchmarkResultBlobAtTrueBound/min-entries` (primary), `BenchmarkResultBlobAtBound/min-entries` | 2-page module returning exactly `MaxResultBlob`; 1-page module returning `MaxResultBlob − len(frame)` |
| `OD-5`(a) | `BenchmarkCompileMissAtBound/unreachable-chain` (primary), `/many-functions` | 16-module pool at `MaxInstrumentedCodeSize`, cycled past the engine's 8-entry cache budget |
| `OD-5`(b) | `BenchmarkInstantiateAtCaps/{globals,functions,data-segments,element-entries,table-entries,all,data-bytes}` | Each §25.1 cap separately, all combined, and data bytes to `MaxCodeSize` |
| `OD-5`(d) | `BenchmarkCodeStoreReloadAtBound/warm-page-cache` | `loadInstrumented` read and re-hash of a `MaxInstrumentedCodeSize` module |

Every benchmark checks the run outcome (`Returned` or `Trapped`) in a warm-up call before it
times anything. Each one reports its trigger gas as `gas/op`.

The OD-4 benchmarks pack the blob with the most distinct minimum-size `LocalStateDiffEntry`
values that fit. They skip reserved local keys and time `settlement.settle`, which decodes the
blob and derives every key.

### Module shapes
- `boundModules` binary-searches the longest `unreachable` chain that validates and instruments
  within `MaxInstrumentedCodeSize`. Variants differ by one immediate, so each has its own
  `CodeHash`.
- `manyFunctionsModules` spreads the same budget over `MaxFunctions/2` (4096) functions. It
  must reach at least 95% of `MaxInstrumentedCodeSize`, because per-function framing costs size.
- `plugins/wasm/fixtures_test.go` `buildWasmModule` encodes tables (`TableMin`), mutable i32
  globals (`Globals`) and active element segments (`Elements`). The structural-cap modules use
  these fields.

### Absent-key witness (`internal/batch/bench_test.go`)
`BenchmarkAbsentKeyWitness` covers `OD-5`(c). At 10³, 10⁵ and 10⁶ leaves it times
`Accessor.Len` plus `Store.Prove` for a fresh absent path. It reports `proof-B/op` beside the
time. The trigger is the fixed 100-gas `storage_len` host cost.

### Working draft (`wasm-vm/wasm-gas-table-v2-draft.md`)
The draft holds the method decisions, the proposed `MinCallGas` term and the results table
template. It also holds the amendment template, the sync checklist and the SPEC §30 closing
text. The draft is not normative; nothing in it binds an implementation.

When the reference pass prices any operation, one amendment applies these changes together:
1. Regenerate `execution-conformance-v1.json` as a `v2` suite, because every frozen `gas_used`
   changes (SPEC §10.2, §31).
2. Update the published gas table in zeno, the `go:embed` copy and the `testdata` copy that
   `TestArtifactDrift` checks. The zeno copy is kept in sync by hand.
3. Replace SPEC §30's `OD-4` and `OD-5` entries with the per-operation classification.

## Decisions
- **Zero-gas triggers get a SPEC-wide floor, `MinCallGas`.** Compile-miss, six of the seven
  instantiation children and code-store reload trigger through a 0-page module. Their 2× ratio
  divides by zero. One floor is easier to verify than five unconditional terms. It also covers
  any future zero-gas operation.
- **`MinCallGas` is charged once per L2-accounted input, before instantiation.** It applies
  where `CallL2` and `Deposit` pay the initial-memory charge. The five §10.1-exempt deploy and
  ownership kinds stay exempt. The value is sized so the cheapest zero-gas operation lands at
  or under 2×.
- **Every ratio uses the floor-adjusted least gas.** The floor raises the trigger cost of every
  operation. The results table therefore records the floor beside `ns_per_gas`, so any row can
  be reproduced.
- **OD-4 reports both blob shapes; the 64 KiB 2-page shape is primary.** The resolution rule
  measures "at its consensus bound", and only the 2-page shape reaches `MaxResultBlob` exactly.
- **OD-5(a) reports both compile shapes.** The `unreachable` chain maximises basic blocks per
  byte, and `many-functions` maximises function boundaries. On a non-reference machine the two
  were within run-to-run noise, so neither is treated as the sole worst case.
- **State-size scaling is removed in the store, not priced.** `Store.Prove` and the per-input
  `Store.Accessor` copy grow with total domain state, which no input pays for and no cap bounds.
  E2-12 owns both store changes. The `Accessor` copy is outside `OD-4` and `OD-5`, so the rule does not
  classify it.
- **The deploy-time pipeline is out of scope.** It has no gas budget to charge against. SPEC
  §32 tracks it under executor economics.

## Code references
- `plugins/wasm/bench_test.go` [EXISTS] — calibration and the `OD-4`, `OD-5`(a), (b), (d)
  benchmarks.
- `plugins/wasm/fixtures_test.go` [EXISTS] — `buildWasmModule` table, global and element
  support, and `TestBuildWasmModuleEncodesTableGlobalsElements`.
- `internal/batch/bench_test.go` [EXISTS] — `BenchmarkAbsentKeyWitness`.
- `plugins/wasm/limits.go` [EXISTS] — `MaxResultBlob`, `MaxCodeSize`,
  `MaxInstrumentedCodeSize`, `MaxFunctions`.
- `plugins/wasm/internal/module/wasm-gas-table-v1.json`,
  `plugins/wasm/internal/module/testdata/wasm-gas-table-v1.json`, and
  `plugins/wasm/internal/module/gastable_test.go` `TestArtifactDrift` [EXISTS].

## Acceptance criteria
- `go test -run '^$' -bench . ./plugins/wasm/ ./internal/batch/` runs every benchmark above,
  and each warm-up outcome check passes.
- `TestBuildWasmModuleEncodesTableGlobalsElements` passes: a module with a table, globals and
  element segments passes `module.Validate`.
- `BenchmarkResultBlobAtTrueBound` returns a blob of exactly `MaxResultBlob` bytes and reports
  about 2000 `gas/op`. `BenchmarkResultBlobAtBound` reports about 1000 `gas/op`.
- `manyFunctionsModules` reaches at least 95% of `MaxInstrumentedCodeSize`, and its variants
  carry distinct `CodeHash` values.
- `wasm-gas-table-v2-draft.md` records a stated decision for each method question: zero-gas
  rows, the OD-4 shape, the OD-5(a) shape and the `Accessor` copy. Its results table has one
  row per `OD-4` and `OD-5`(a)–(d) operation.
- Each results row has columns for median, spread, floor-adjusted least gas, `ns_per_gas`,
  ratio against 2×, machine and classification.
