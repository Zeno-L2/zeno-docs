# Ticket E2-21 — Withdrawals: `MerkleProof` builder, `RelayMessage`, `ReleaseWithdrawals`

**Epic:** 2 — Executor · **Phase:** 1C · **Order:** E2-21

## Goal
Carry finalized withdrawals to L1. End state: once a batch finalizes, the supervisor
registers each of its relayable withdrawals with `RelayMessage` and drains the
domain's release queue with `ReleaseWithdrawals`, and the executor's proofs verify
against the same vector as Settlement's verifier.

## SPEC refs
SPEC §17.3 (relay rules, rejections), §18.2 (release, empty queue), §21 (finality),
§25.1 (`MaxMerkleProofDepth`, `ReleaseIntervalMomentums`), §27.3 (`MerkleProof`),
§27.9 (per-batch totals, refund bound).

## Depends on
E2-16, E2-19; E1-10 (frozen proof vector, release-queue meta key).

## Key work
- **Proof builder.** `internal/codec`: build the §27.3 `MerkleProof` for element `i`
  of a list, over the same tree as `MerkleRoot` (promoted odd nodes contribute no
  step). Add the proof vector E1-10 froze in go-zenon (proof hex for every
  `CV-INPUTROOT-1` element) to `internal/codec/testdata/serialization-conformance-v1.json`,
  and copy the suite to zeno's `artifacts/`. The builder must reproduce those bytes
  exactly.
- **Outbox path.** After each `Committed` message, the worker sends the batch's
  `L1_WITHDRAWAL` messages with their inclusion proofs, in strictly ascending
  `(inputIndex, outboxIndex)` order, then an end frame. The supervisor verifies each
  proof against the batch's `outboxRoot` and the per-asset totals against the
  `AssetFlowSummary` withdrawal debit, then keeps them in a durable per-domain store
  until every one is relayed or skipped. It never reads the worker's directory.
- **Relay.** After a batch is `FINALIZED` (`height ≥ finalizeAtHeight`), send one
  `RelayMessage` per withdrawal not yet in `processedOutbox`. Skip messages
  Settlement rejects (`kind = 0`, zero amount, embedded-contract recipient).
- **Release.** For the executor's own domains, send `ReleaseWithdrawals` at most once
  per momentum while the release queue is non-empty. Respect `PAUSE_RELAY`.

## Code references
- zeno-executor `internal/codec/merkle.go`, `internal/codec/outbox.go` (`OutboxID`),
  `internal/batch/builder.go` (outbox order), `internal/ipc/` [EXISTS].
- go-zenon `vm/embedded/implementation/settlement.go` (`RelayMessage`,
  `ReleaseWithdrawals`, proof verifier) [EXISTS].

## Acceptance criteria
- Proofs for every element of lists of length 1 to 33 verify against `MerkleRoot`;
  the builder reproduces go-zenon's frozen vector byte for byte.
- On the devnet, a finalized withdrawal is registered once, released, and arrives at
  its recipient; a second relay of the same `outboxId` is not sent.
- A relay restart resumes from the submission log without double-sending.
- `go test ./...` passes.
