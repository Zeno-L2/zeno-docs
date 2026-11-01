# Ticket E2-29: Reference-hardware gas calibration

**Epic:** 2 (Executor) · **Phase:** 1C · **Order:** E2-29

## Goal
Run the gas benchmark suite on a VPS of the class that runs testnet, then price every operation that breaks the 2× rule on it. End state: on the reference executor hardware, no measured operation runs for more than 2× the time its gas buys, a plain `call` included. The worst-case batch fits a stated wall-clock budget, and SPEC §30's `OD-4` and `OD-5` are closed.

## The problem
Every gas ratio measured so far comes from a developer laptop (Apple M1 Pro, darwin/arm64). SPEC §30's resolution rule asks for the reference executor hardware, so none of those numbers can settle a price.

The laptop numbers already show one operation far over the rule. A call to a function with no locals ran at 13.0× the time its gas buys (E2-25). A loop of plain calls costs about 40 ns per gas against a calibration of about 3 ns per gas. So one input at `MaxInputGas` runs about 40 s, and a worst-case batch (`MaxBatchInputs` × `MaxInputGas`) runs about 43 minutes.

A VPS core is usually slower per thread than the laptop, and a shared vCPU adds steal time. Execution within a domain is sequential, so extra cores do not help. Gas is never paid in tokens, so an attacker who deploys a call-loop module once pays only L1 plasma per `CallL2` to stall every honest executor on the domain.

## SPEC refs
SPEC §10.1 (`MaxInputGas` bounds the work one input can compel), §10.2 (opcode gas table), §24 (versions and the pre-activation rule), §25 (`MaxBatchInputs`, `MaxInputGas`), §30 (`OD-4`, `OD-5` and their resolution rule), §31 (conformance artifacts).

## Depends on
E2-28. The devnet run and its cleanup settle the build that is measured. E2-11 (benchmark suite, method decisions, `wasm-gas-table-v2-draft.md`) and E2-25 (locals entry charge, `call` known gap) supply the method and the starting numbers.

Lands before the activation spork. SPEC §24 says a format change made before activation updates the format in place and bumps no version. Every price this ticket sets therefore edits `metering_version = 1` and `wasm-gas-table-v1.json` in place, including the terms the v2 draft stages for `metering_version = 2`.

## Key work

### 1. Fix the reference hardware
- Choose the VPS class that runs testnet, preferring dedicated vCPUs. Record the provider, instance type, CPU model, architecture, vCPU count and whether vCPUs are dedicated, plus RAM, OS and kernel, the Go version and the pinned wazero version.
- That record is the definition of "reference executor hardware" for SPEC §30 and for every ratio in this ticket.
- On a shared-vCPU class, record CPU steal (for example `vmstat 1` during the run) beside every result.

### 2. Run the suite
- Build zeno-executor at the E2-28 commit and run every benchmark with `-count 10`: `BenchmarkCalibrateNsPerGas`, the E2-11 `OD-4` and `OD-5` benchmarks in `plugins/wasm` and `internal/batch`, `BenchmarkLocalsZeroing` and `BenchmarkLocalsLoopAtBudget`.
- Add one benchmark per control-transfer price that the laptop numbers put in doubt: `call`, `call_indirect` and `return`, each in its least-gas loop shape, at the default `plasmaLimit` and at `MaxInputGas`.
- Keep the raw logs with the ticket's PR.

### 3. Apply the 2× rule
- Compute each row's ratio against the reference `ns_per_gas`. Fill every `[TBD]` cell in `wasm-vm/wasm-gas-table-v2-draft.md`, adding rows for the control-transfer benchmarks and the locals charge.
- Classify each row as priced (needs a new or higher price) or accepted (a bounded cost within 2×, with its ratio).

### 4. Price
- Raise the base price of each control-transfer opcode over 2× until its least-gas loop is within 2×.
- Re-check the locals charge `function_entry.locals_per_gas` (2 on the laptop). Change it only if the reference run requires it.
- Set `MinCallGas` and any `OD-4` or `OD-5` term the rule requires, as the v2 draft stages them.
- Regenerate the conformance artifact once. Every vector that makes a call changes, and that is expected here.

### 5. Bound the batch
- State the wall-clock budget one worst-case batch may take on the reference hardware, and record the reason for it.
- Check that `MaxBatchInputs` × `MaxInputGas` × the worst post-change `ns/gas` fits that budget. If it does not, lower `MaxInputGas` in this ticket. SPEC §25 says that lowering it after activation breaks the validity-proof program.
- Re-check the 200,000,000 gas budget of `zeno-exec serve` against its request timeout.

### 6. Document
- SPEC §10.2 and §25 state the new prices and limits. SPEC §30 replaces `OD-4` and `OD-5` with the per-operation classification. The CHANGELOG records the change as a pre-activation correction.
- Update every copy that `artifacts/README.md` lists: both zeno-executor gas table copies, the zeno-docs gas table JSON and `.md`, and the execution artifact and its README in both repositories.
- Close the E2-25 known gap by naming this ticket's result.

## Code references
- `plugins/wasm/bench_test.go`: calibration, the E2-11 benchmarks, `BenchmarkLocalsZeroing`, `BenchmarkLocalsLoopAtBudget`.
- `internal/batch/bench_test.go`: `BenchmarkAbsentKeyWitness`.
- `plugins/wasm/internal/module/wasm-gas-table-v1.json` and its `testdata/` copy, `gastable.go`, `instrument.go`.
- `plugins/wasm/limits.go`: `MaxInputGas`. `internal/codec/constants.go`: `MaxBatchInputs`.
- `wasm-vm/wasm-gas-table-v2-draft.md`: the results table and amendment template.

## Starting numbers (Apple M1 Pro, darwin/arm64, go1.23.12, wazero v1.8.2)
These come from E2-25 and are not reference numbers. They show where to look first.
- `ns_per_gas` calibration: 3.08 to 3.32 ns.
- Call to a function with no locals: 40.1 ns per gas, a ratio of 13.04.
- 69-byte locals module after the E2-25 charge: 1.57 at `plasmaLimit`, 1.60 at `MaxInputGas`.
- One input at `MaxInputGas` in a plain-call loop: about 40 s.

## Acceptance criteria
- The hardware record names the instance type and CPU, and every result row names that machine.
- `wasm-gas-table-v2-draft.md` has no `[TBD]` left, and each row has median, spread, least gas, `ns_per_gas`, ratio and classification.
- On the reference hardware, every measured operation runs within 2× the time its gas buys. That covers a plain `call` with no locals, and the 69-byte locals module at the default `plasmaLimit` and at `MaxInputGas`.
- The worst-case batch time on the reference hardware is stated in the PR and fits the stated budget.
- SPEC §30 no longer lists `OD-4` or `OD-5` as open.
- The conformance artifact is regenerated once, every copy has the source's SHA-256, and a second implementation can reproduce the new prices from the SPEC text alone.
