# Ticket E2-25: Local-variable gas pricing

**Epic:** 2 (Executor) · **Phase:** 1C · **Order:** E2-25

## Goal
Price the work of zeroing a function's local variables, or cap how many a function may declare, so that gas bounds the time an input can take. End state: a 69-byte module that calls an 8,192-local function in a loop can no longer run for minutes on one `CallL2` or `Deposit`, and the gas table, the validator and the conformance artifact agree on the rule.

## The problem
The gas table charges a `call` a flat 5 and charges nothing for locals (`plugins/wasm/internal/module/wasm-gas-table-v1.json`). The pinned wazero interpreter (v1.8.2, zeno-executor `go.mod`) zeroes every declared local on each call: `(*compiler).compile` in `internal/engine/interpreter/compiler.go` emits one zero-constant operation per declared local (lines 379-381, through `emitDefaultValue`), and `(*callEngine).callNativeFunc` in `interpreter.go` runs them on every call. A function that declares 8,192 `i64` locals therefore costs 5 gas and thousands of memory writes.

This module passes `validate` and is 69 bytes: `(func $f (local i64 ×8192)) (func (export "run") (param i32 i32) (result i64) (loop $l (call $f) (br $l)) (i64.const 0))`.

Measured on an 8-core machine:
- One `CallL2` at the default `plasmaLimit` of 10,000,000 runs 24.8 s before `OUT_OF_GAS`. That is 0.40 M gas per second, about 50 times slower per gas than a `loop br 0` body.
- One input at `MaxInputGas` (1,000,000,000) would run about 2,500 s, about 42 minutes.
- A batch of such inputs stalls an honest executor for hours, and §10.1 says `MaxInputGas` exists to bound exactly this work.

E2-11's resolution rule treats an operation as unpriced when its measured cost at its consensus bound exceeds 2× the time bought by the least gas that triggers it. Local zeroing is such an operation, and E2-11's suite does not measure it.

## SPEC refs
SPEC §10.1 (`MaxInputGas` bounds the work a single input can compel), §10.2 (opcode gas table), §7 (runtime profile, structural caps), §24 (versions and the pre-activation rule), §25.1 (structural caps, `MaxInputGas`), §31 (conformance artifacts).

## Depends on
E2-11 (benchmark method and the 2× rule). Lands before the activation spork: SPEC §24 says a format change made before activation updates the format in place and bumps no version, so the fix edits `metering_version = 1` in place.

## Key work

### 1. Reproduce and measure
- Add the module above as a test and a benchmark in `plugins/wasm`. Record gas per second for 0, 64, 1,024 and 8,192 locals, for functions that declare only `i32` locals and for functions that declare only `i64` locals.
- Find the worst module the validator accepts: the cap on locals per function, the cap on functions, and call depth. Report the largest time one input can take at `MaxInputGas`.

### 2. Choose the rule (decision recorded in the ticket's PR)
Pick one, with the measurements as evidence:
- **Charge locals at function entry.** Add a per-local cost to each function's entry block in the basic-block model, for example `ceil(local_count / k)` for a measured `k`. Costs a table change and an instrumenter change.
- **Cap locals per function.** Add a structural cap to §25.1 and reject larger functions at deploy. Costs a validator change and a new deploy rejection reason, and it breaks no existing price.
- **Both**, with the cap set high enough that the charge is the binding limit.

Whichever is chosen must keep the 2× rule: the worst case at the consensus bound is at most 2× the time bought by the gas.

### 3. Implement
- Update the gas table (`wasm-gas-table-v1.json` and `.md`), the instrumenter and the validator as the chosen rule needs.
- Both zeno-executor copies of the gas table JSON must change together. The table disagrees with itself on the price of integer conversions (2 in `cost_principles`, 1 in the entries); settle that in the same change.
- Regenerate the conformance artifact once. Add vectors: a function at the cap, a function over the cap (if capped), and a call whose entry charge makes it run out of gas.
- Update every copy, as `../../../artifacts/README.md` lists them: the gas table JSON in zeno-executor `plugins/wasm/internal/module/` and its `testdata/` copy, and in zeno-docs `WASM Spec/` with `wasm-gas-table-v1.md`; the execution artifact and its README in zeno-executor `plugins/wasm/testdata/` and zeno-docs `WASM Spec/`. No other repository holds a copy of either.

### 4. Document
- SPEC §10.2 or §25.1 states the rule normatively. The CHANGELOG records it as a pre-activation correction.
- Re-check the 200,000,000 gas budget of `zeno-exec serve` against the new worst case. The child process that `serve` kills at its request timeout stays as defence in depth.

## Code references
- `plugins/wasm/internal/module/wasm-gas-table-v1.json` and the second copy under `plugins/wasm/internal/module/testdata/`.
- wazero v1.8.2 `internal/engine/interpreter/compiler.go:379-381` (`(*compiler).compile`, local zeroing through `emitDefaultValue`) and `interpreter.go:698` (`(*callEngine).callNativeFunc`, which runs it on every call).
- The instrumenter in `plugins/wasm/internal/module` and the validator.
- `../SPEC.md` §10.1, §10.2, §24, §25.1.

## Acceptance criteria
- The 69-byte module runs for at most 2× the time its gas buys, at the default `plasmaLimit` and at `MaxInputGas`, on the reference executor hardware. A test fails on the current table.
- A module whose function declares more locals than the rule allows is rejected at deploy, or is charged at entry, as the chosen rule says.
- Both zeno-executor gas table copies agree byte for byte, and a test compares them.
- Every copy of the gas table and the execution artifact listed in `../../../artifacts/README.md` has the source's SHA-256.
- The conformance artifact has the new vectors, regenerated once, and a second implementation can reproduce them from the SPEC text alone.
- No existing vector changes outside the rows that use many locals.

## Known gap
The `call` base price is re-measured on reference hardware in the closing benchmark ticket. Starting point: a call to a function with no locals ran at 13.0× the time its gas buys on Apple M1 Pro, which is over the 2× rule.
