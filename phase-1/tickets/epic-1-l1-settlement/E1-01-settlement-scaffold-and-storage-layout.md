# Ticket E1-01 — Settlement contract scaffold, spork, address and storage layout

**Epic:** 1 — L1 Settlement · **Phase:** 1B · **Order:** E1-01

## Goal
Register the spork-gated Settlement embedded contract and define its storage
layout and protocol constants. End state: before the spork every Settlement
selector is rejected; after it the contract resolves and dispatches its methods.
Every Phase-1 storage entity has a collision-free key and a tested accessor, and
the §25.1 constants and §25.2 hard bounds live in `vm/constants/`.

## SPEC refs
SPEC §5.1 (activation), §5.2 (Core/Periphery split), §5.4 (storage
responsibilities), §6 (domain schema), §25.1 (fixed constants), §25.2
(pre-mainnet parameters and their bounds), §28 (codebase foundations);
EXECUTOR.md §6.

## Depends on
None. This is the first Settlement ticket and has no Epic 2 dependency.

## Key work

### Spork, address and dispatch
- `OffchainExecutionSpork` in `common/types/spork.go`, with a placeholder hash in
  `ImplementedSporksMap` (placeholder, then `CreateSpork`, then `ActivateSpork`).
  It activates a contract and bumps no momentum version.
- `IsOffchainExecutionSporkEnforced()` in `vm/vm_context/spork.go`, mirroring
  `IsHtlcSporkEnforced()`.
- `SettlementContract` in `common/types/address.go` via `parseEmbedded`, appended
  to `EmbeddedContracts`.
- `vm/embedded/embedded.go`: `applySettlementDiffs` adds the Settlement method
  map. Settlement is its own bit (`settlementSporkBit`) in the precomputed
  embedded variant table, so `buildEmbeddedVariants` covers every combination
  of the five sporks and `GetEmbeddedMethod` sets the bit when the spork is
  enforced. `getAllEmbedded` in `embedded_test.go` applies it for tests.
- `vm/embedded/definition/settlement.go`: the Settlement ABI and method-name
  constants. `vm/embedded/implementation/settlement.go`: `Method` implementations
  (`GetPlasma`, `ValidateSendBlock`, `ReceiveBlock`).

### Storage layout
- Domain-keyed where domain-scoped, each entity under its own key prefix:
  - `DomainRecord` by `domainId`, encoding the full §6 schema (`domainClass`,
    `runtimeKind`, `stfSpecHash`, `inputSource`, `execProfile`,
    `foreignProfile`, `profileConfig`, `chainBinding`, `executors`,
    `minExecutors`, `proposerPolicy`, `valueCaps`, `finalityModel`, `status`).
    Phase 1 writes the defaults (E1-03 clamps them), so a later profile
    activation is a spork, not a storage migration.
  - Batch entries by `(domainId, batchId)` and the per-domain input-sequence
    cursor.
  - The global received-input counter, the per-domain assigned-index records and
    the per-(domain, asset) cumulative deposit totals (§4.6, §18.4).
  - Per-(domain, asset) `totalDeposited`, `pendingWithdrawalReserve` and
    `totalReleased`; the per-domain release queue in registration order.
  - Pending deploy records, the `processedOutbox` set, executor registration
    state, and Periphery configuration. E1-04, E1-06 and E1-07 set the final
    shape of the executor, deploy and batch records.
- Reserved entities (schema present, unused in Phase 1, §5.4): the per-(domain,
  asset) `AssetRegistry`, the per-`outboxId` `MessageState`, and the Periphery
  verifier registries. Their key prefixes are reserved so no later entity
  collides with them.

### Constants
- `vm/constants/settlement.go`: the §25.1 table (`MaxBatchInputs = 64`,
  `MaxOnChainCommitmentSize = 15 KiB`, `MaxCallPayload`, `MaxReturnData`,
  `MaxCodeSize`, `MaxDeployChunkData`, `MaxChunkCount`, `DeploymentExpiryWindow`,
  `MaxAssetsPerBatch`, `MaxWithdrawalsPerUpdate`, `MaxStateValueSize` and the
  rest). It reuses `MaxDataLength` (`vm/constants/plasma.go`) and
  `MomentumsPerEpoch` (`vm/constants/embedded.go`).
- Hard-bound holders for the §25.2 parameters: challenge and withdrawal-delay
  bounds, the `MaxBatchWithdrawal` floor and ceiling, `MinRuntimeUpgradeDelay ≥
  WithdrawalDelay`, the `valueCaps` window floor, and the reserved `MaxBatchMint`
  bounds.
- Settlement errors in `vm/constants/errors.go`.

## Code references
- `common/types/spork.go`, `common/types/address.go`, `vm/vm_context/spork.go`,
  `vm/vm_context/interfaces.go` [EXISTS].
- `vm/embedded/embedded.go` (`buildEmbeddedVariants`, `GetEmbeddedMethod`) and
  `vm/embedded/embedded_test.go` (`getAllEmbedded`, the ABI method dump) [EXISTS].
- `vm/embedded/definition/settlement.go`, `vm/embedded/implementation/settlement.go`,
  `vm/constants/settlement.go` [NEW].

## Acceptance criteria
- Before enforcement a call to any Settlement selector is rejected; after
  enforcement (test override of `ImplementedSporksMap`) the address resolves and
  methods dispatch.
- Every embedded-variant combination with the Settlement bit carries the
  Settlement contract, and none without it does. Existing embedded-contract tests
  still pass.
- Round-trip read/write tests for `DomainRecord`, the batch entry, the three
  per-(domain, asset) counters, the received-input counter, assigned-index
  records and cumulative deposit totals, the release queue (FIFO preserved), the
  deploy record and `processedOutbox` membership.
- No key prefix collides across entities. Constants match §25.1, and the
  hard-bound clamps reject out-of-range values.
