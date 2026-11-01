# Ticket E1-10 — Settlement client surface: send hash in the ordinal entry, exported keys

**Epic:** 1 — L1 Settlement · **Phase:** 1B · **Order:** E1-10

## Goal
Let the executor read Settlement's input assignment and storage without duplicating
go-zenon internals. End state: each per-domain ordinal entry names the send it
assigned, the executor-facing storage keys are exported from
`vm/embedded/definition`, and a test proves a Settlement storage entry through
`ledger.getProof`.

## SPEC refs
SPEC §4.6 (per-domain ordinal, the executor reads the assignment), §5.4 (storage),
§27.3 (`MerkleProof`), §27.5 (`source_account_block_hash`).

## Depends on
E1-07 (ordinal entries), E1-08 (proof verifier).

## Key work
- **Ordinal entry.** `DomainInputOrdinal` (`0x20 ‖ domainId ‖ ordinal`) stores
  `{globalInputIndex, sourceAccountBlockHash}`. `admitInput` writes the send's
  account-block hash. The sweep deletes it as before. No version field changes: the
  format is fixed before activation.
- **Exported key builders and ABI variable names** in
  `vm/embedded/definition/settlement.go` for every entry the executor reads:
  input-sequence cursor, domain received-input counter, ordinal entry, batch entry,
  asset config, global and domain pause state, executor-set member, processed
  outbox, and release-queue meta (`0x09`). Export the variable-name constant of each
  ABI-packed entry, so the executor decodes with `ABISettlement.UnpackVariable` and
  no literal names. Keep the prefix table as the single source; the other names
  stay unexported.
- **State-tree key test.** In `vm/embedded/tests`, prove a Settlement storage entry
  with `ledger.getProof` at a version-3 momentum, using the raw key
  `0x03 ‖ SettlementContract ‖ 0x04 ‖ key`, and verify it with `VerifyProofByPath`.
- **Frozen proof vector.** A test-only `MerkleProof` builder and a test that inlines,
  for every `CV-INPUTROOT-1` element, its proof hex. Each proof verifies through
  `outboxProofRoot` to the root `TestOutboxTree_ReproducesConformanceRoot` pins, and
  at least one proof crosses a promoted odd node. These bytes are the vector E2-19
  adds to zeno-executor's serialization suite.

## Code references
- `vm/embedded/definition/settlement.go` (prefix table, ABI variable names, `DomainInputOrdinal`),
  `vm/embedded/implementation/settlement.go` (`admitInput`, sweep, proof verifier),
  `vm/embedded/tests/` (spork helpers) [EXISTS].

## Acceptance criteria
- Every accepted input's ordinal entry carries its send hash; rejected sends write
  none; the sweep removes entries as before.
- Every exported key builder produces the bytes in the prefix table, and every
  exported variable name decodes its entry with `ABISettlement.UnpackVariable`.
- The inlined proof for every `CV-INPUTROOT-1` element verifies; a proof with one
  altered byte does not.
- The `getProof` test verifies a Settlement entry's value against the momentum state
  root.
- Existing Settlement tests pass; `TestDumpContractsABIMethods` unchanged.
