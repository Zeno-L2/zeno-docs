# Ticket E1-08 — `RelayMessage` registration + `ReleaseWithdrawals` withdrawal release

**Epic:** 1 — L1 Settlement · **Phase:** 1B · **Order:** E1-08

## Goal
Implement the two-method withdrawal path: permissionless `RelayMessage`
registration of a proven, finalized outbox message, and the permissionless,
rate-limited `ReleaseWithdrawals` crank that releases queued withdrawals under the
conservation, replay, and fan-out bounds. End state: a finalized L1-withdrawal
outbox message can be registered exactly once and released by the next `ReleaseWithdrawals`,
with aggregate solvency preserved and `FINALIZED` as the only time gate.

## SPEC refs
SPEC §17 (outbox/messaging, replay, `RelayMessage`, relay rejections), §18.2/§18.3
(release queue, `ReleaseWithdrawals`, empty queue, conservation), §21 (finality — the
only time gate), §27.3 (`MerkleProof`), §27.9 (`MaxWithdrawalsPerUpdate`, fan-out
bounds, refund bound), §25.1 (`MaxMerkleProofDepth`), §5.4 (storage), §27.10 (pause), §5.3
(`RelayMessage`, `ReleaseWithdrawals`), §28 (`ReleaseWithdrawals` crank precedent).

## Depends on
Tickets E1-07 (FINALIZED batches + committed `outboxRoot`), E1-05 (conservation
counters), E1-01 (release queue + `lastUpdate` storage), E1-02 (pause gate).

## Key work
- `RelayMessage(domainId, batchId, message, inclusionProof)` — permissionless,
  **registration-only, releases no funds**. Verify: batch is `FINALIZED`;
  inclusion proof of `message` against the batch's committed `outboxRoot`
  (§27.3 `MerkleProof`: reject `step_count > MaxMerkleProofDepth` before reading
  steps, reject a `side` other than 0 or 1 and trailing bytes, and hash the leaf
  from the supplied message); the supplied `domainId`/`batchId` match the batch whose
  `outboxRoot` verifies the proof (reject mismatch);
  `outboxId = sha3(domainId‖batchId‖inputIndex‖outboxIndex)` not in
  `processedOutbox`. Record `outboxId` at registration (at-most-once — a message
  is registered exactly once, ever).
- Relay rejections (SPEC §17.3), each recording nothing so the `outboxId` stays
  free: a `kind = 0` message, a `kind = 1` message with `amount = 0`, and a
  `kind = 1` message whose `recipient_l1` is an embedded-contract address
  (`types.IsEmbeddedAddress`, `ErrForbiddenParam`). A release to an embedded
  address fails in `applySend` and would roll back the whole `ReleaseWithdrawals`.
- L1 withdrawal (`kind==1`) registration: increase
  `pendingWithdrawalReserve[domainId][asset]` and append to the domain's release
  queue in registration order. No additional delay — `FINALIZED` already gates.
- `ReleaseWithdrawals(domainId)` — permissionless, rate-limited by
  `ReleaseIntervalMomentums = 1` (SPEC §18.2, §25.1; the Accelerator-payout pattern),
  with the rate limit evaluated **per domain** so one domain's traffic cannot starve another's exits.
  Keep it in its own record keyed by `domainId` (new key prefix): the shared
  `checkAndPerformUpdate` record is one value for the whole contract and belongs to
  Ticket E1-04's fee crank `Update` (SPEC §18.2).
  Ticket E1-04 renamed the `Update(domainId)` stub to `ReleaseWithdrawals(domainId)`
  (the name `Update` belongs to the no-argument fee crank, SPEC §5.3); the stub
  already carries `domainId` and rejects a non-zero attached amount at send
  validation. This ticket adds the release logic behind it. Each
  accepted call drains that domain's release queue in registration order, at most
  `MaxWithdrawalsPerUpdate (16)` per call: per release, move the amount to
  `totalReleased`, decrease `pendingWithdrawalReserve`, transfer via a
  descendant `ContractSend`, re-check conservation (§18.3). An empty queue fails
  with `ErrNothingToWithdraw` and leaves the rate-limit record unchanged; a domain's
  first call always passes the rate limit.
- Two running totals per (domain, batch, asset), stored at registration (SPEC
  §27.9): contract-emitted withdrawals, rejected above the (domain, asset)
  `MaxBatchWithdrawal`; and refunds (`source_contract = bytes32(0)`), rejected
  above the batch's `depositCredit` for the asset from its stored
  `AssetFlowSummary` (an absent asset has `depositCredit = 0`).
- Pause interaction (§27.10): `PAUSE_SUBMIT` MUST NOT freeze registration or
  release for `FINALIZED` batches; `PAUSE_RELAY` halts both `RelayMessage`
  registration and `ReleaseWithdrawals` release and is separately authorized; on
  deactivation, queued withdrawals release again in unchanged registration
  order.

## Code references
- `vm/vm.go` (descendant `ContractSend` from embedded; unlimited embedded
  plasma) [EXISTS]; `pillar/worker_contract_generator.go` (auto-receive)
  [EXISTS].
- `vm/embedded/implementation/accelerator.go` (`UpdateEmbeddedAcceleratorMethod`)
  + `vm/embedded/implementation/common.go` (`checkAndPerformUpdate`) — the
  `ReleaseWithdrawals` crank pattern [EXISTS]; `vm/constants/embedded.go`
  (`UpdateMinNumMomentums` stays with the fee crank; `ReleaseIntervalMomentums`
  is new beside it) [EXISTS].
- zeno-executor `internal/codec` outbox encoding (E2-03) [EXISTS]; the `MerkleProof`
  encoding is new here, and E2-19 builds the executor's proof builder [NEW]; Ticket E1-05 conservation helper [NEW];
  Ticket E1-01 release queue [NEW].

## Acceptance criteria
- EXEC-DEP-3 end to end: a deposit to an undeployed `targetContract` yields
  `VALIDATION_FAILED`, and its full-refund outbox message is registered and released
  to the depositor (Ticket E1-05 tests the on-chain preconditions).
- Registration of a valid finalized withdrawal records `outboxId`, increases the
  reserve, and enqueues; the next `ReleaseWithdrawals` releases it (transfer + counters +
  conservation re-check). A relay against a `SUBMITTED` batch is rejected.
- Replay of the same `outboxId` rejected; a proof valid against a mismatched
  domain/batch rejected.
- Malformed proofs rejected: `step_count > MaxMerkleProofDepth`, a `side` outside
  {0, 1}, trailing bytes, a wrong sibling. A proof for a single-message batch has
  zero steps; promoted odd nodes contribute no step.
- `kind = 0`, zero-amount and embedded-recipient messages rejected with nothing
  recorded; the same `outboxId` stays unregistered.
- `ReleaseWithdrawals` on an empty queue fails with `ErrNothingToWithdraw` and does
  not delay the next release.
- `ReleaseWithdrawals` releases in registration order, stops at `MaxWithdrawalsPerUpdate`,
  and a second `ReleaseWithdrawals` for the same domain within
  `ReleaseIntervalMomentums` (the same momentum) is rejected; remaining
  queue entries release on the next accepted call.
- Per-batch contract-withdrawal total capped at `MaxBatchWithdrawal`; per-batch
  refund total capped at the batch's `depositCredit`, with refunds outside the
  `MaxBatchWithdrawal` total.
- Deposit-refund path (EXEC-FAIL-OUTBOX-1 / EXEC-DEP-1) releases the unclaimed
  remainder to the original depositor.
