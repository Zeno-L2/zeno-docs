# Ticket E2-15 — Key the WASM plugin's pending deploy records by deployer

**Epic:** 2 — Executor · **Phase:** 1A · **Order:** E2-15

## Goal
Match SPEC v1.16.4's deploy-record key. Settlement keys every pending deployment
record by `(domainId, deployer, code_hash)` (§11), so two deployers can stage the
same `code_hash` at once. The WASM plugin's node-local store keys its pending record
and chunk entries by `code_hash` alone (`plugins/wasm/wasm.go`, `deployChunkKey`),
so a second deployer's `DeployContractStart` would clear and overwrite the first
deployer's entries. End state: the plugin keys both by deployer and `code_hash`, and
two concurrent deployments of the same module both finalize correctly.

## SPEC refs
SPEC §11.1–§11.5 (record key, duplicate rule, chunk/finalize lookup, expiry), §12
(`UpgradeContract` consumes the caller's own record), §4.6 (the executor's acceptance
predicate must match Settlement's), §20 (bundle `deployChunkData` — unchanged).

## Depends on
Ticket E2-04 (WASM plugin), E2-10 (stale deploy record hazard).

## Key work
- Key the pending record and every chunk entry by `(deployer, code_hash)`, the
  deployer being the `DeployContractStart` sender (`L1Sender`). Chunk, finalize and
  upgrade inputs look up with their own sender.
- A `DeployContractStart` clears only its own deployer's entries for that
  `code_hash`, never another deployer's.
- Revisit the stale-pending-record halt (EXECUTOR §15): a finalize can no longer
  read a record whose deployer differs from its sender, because the key includes
  the deployer. What remains is a repeated start by the same deployer. Update the
  §15 failure-table row and the reorg paragraph to match, and record the change in
  `CHANGELOG.md`.
- E2-17 reads Settlement's input assignment from L1 state, so the executor makes no expiry or duplicate checks of its own; Settlement's checks
  use the same key.

## Code references
- `plugins/wasm/wasm.go` (`deployChunkKey`, `encodePendingRecord`, the
  `DeployStartArgs` clearing loop) [EXISTS].
- `internal/recovery/boot.go`, `internal/batch/driver.go` (stale-record notes)
  [EXISTS].

## Acceptance criteria
- Two deployers stage and finalize the same `code_hash` in interleaved order; both
  contracts deploy, each from its own chunks.
- A `DeployContractStart` by deployer B does not disturb deployer A's pending record
  or chunks for the same `code_hash`.
- No committed byte changes: input encodings, receipts, roots and `DAHash` are
  identical to the previous key layout for single-deployer flows, and every
  conformance suite passes unmodified.
- `go test ./...` passes.
