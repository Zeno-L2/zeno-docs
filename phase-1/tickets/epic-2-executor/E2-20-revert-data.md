# Ticket E2-20 — Revert data: `abort(code, data_ptr, data_len)`

**Epic:** 2 — Executor · **Phase:** 1C · **Order:** E2-20

## Goal
Let a contract refuse with data, in the one form every runtime shares. End state: a
`REVERT` receipt commits its abort code and its revert data through `returnHash`, the
WASM host's `abort` takes a data buffer, and the execution conformance artifact pins
the new behaviour.

## SPEC refs
SPEC §7.5 (aborts), §8.1 (`abort` signature and resolution order), §10.3 (`abort`
gas), §10.6 (`REVERT` and the new `RESULT_TOO_LARGE` row), §14.3 (the `REVERT`
exception to the discard rule), §15 (`abortCode` and revert data, `returnHash`),
§25.1 (`MaxReturnData`), §27.1 (`r < 0`), §27.1a (the `REVERT` `ExecutionResult`);
`wasm-vm/wasm-gas-table-v1.md` §6.

## Depends on
E2-04 (runtime), E2-18 (the conformance artifact and its generator). Lands before
E2-22, which serves revert data, and before E2-26, which adds abort rows on top of it.

## Key work
- **Host import.** `abort(code: i32, data_ptr: i32, data_len: i32)` in
  `internal/engine/host.go`. Charge `1 + 10 * ceil(data_len / 32)` first, then
  resolve per §8.1: `data_len > MaxReturnData` is `RESULT_TOO_LARGE` without reading
  memory; an out-of-memory range traps; otherwise `REVERT` with the copied bytes.
  `data_len` and `data_ptr` are read as unsigned.
- **Validation.** The `abort` import signature becomes `(i32, i32, i32) -> ()` in
  `internal/module/validate.go`; a module importing the one-parameter form is
  rejected like any other wrong signature.
- **Gas table.** `abort.per_32_bytes = 10` in both zeno-executor copies of
  `wasm-gas-table-v1.json`; zeno's published copy already carries it.
- **Result and receipt.** On `REVERT`, assemble the `ExecutionResult` per §27.1a with
  `ReturnData` = revert data, and set the receipt's `returnHash` to its `SHA3-256`, or
  zero when empty. `r < 0` keeps empty revert data. Every other non-`SUCCESS` status
  keeps `returnHash = 0`.
- **Conformance.** Regenerate `execution-conformance-v1.json` with rows for: an abort
  with data; an abort with `data_len = 0` and a garbage `data_ptr`; `data_len =
  MaxReturnData` (accepted); `MaxReturnData + 1` (`RESULT_TOO_LARGE`, memory not
  read); an out-of-memory range (`RUNTIME_FAULT`); an abort whose data charge is
  unaffordable (`OUT_OF_GAS`); and a reverting payable call, whose refund outbox
  survives beside the revert data. Update existing `REVERT` rows built against the
  one-parameter `abort`. Copy the artifact to zeno's `wasm-vm/`.

## Code references
- zeno-executor `plugins/wasm/internal/engine/{host,outcome,engine}.go`,
  `plugins/wasm/internal/module/{validate,gastable}.go`, `plugins/wasm/effects.go`,
  `internal/codec/receipt.go`, `internal/batch/builder.go`,
  `plugins/wasm/{vectors,callvectors}_test.go` [EXISTS].

## Acceptance criteria
- Each row above reproduces byte for byte, and the artifact digest is updated in both
  repos.
- A `REVERT` receipt's `returnHash` equals `SHA3-256` of the revert data in its bundle
  `ExecutionResult`; `stateDiffHash`, `eventRoot` and `abortCode` follow §15.
- A module importing `abort(i32)` fails validation with `VALIDATION_FAILED` at deploy.
- `TestArtifactDrift` passes with the new gas table; `go test ./...` passes.
