# Ticket E1-07 — `SubmitBatch` acceptance + finality state machine

**Epic:** 1 — L1 Settlement · **Phase:** 1B · **Order:** E1-07

## Goal
Implement batch-commitment acceptance — the heart of Settlement's anchoring role
— and the `SUBMITTED → FINALIZED` finality timer. Core verifies ordering,
contiguity, chaining, sizing, version, and asset-flow correspondence, but **never
execution correctness** (Phase 1). End state: a contiguous, well-chained batch
from the entitled proposer is accepted and finalizes after the withdrawal delay.

## SPEC refs
SPEC §19 (batch commitments, `SubmitBatch`), §4.6 (per-domain contiguity), §21
(finality / withdrawal delay), §18.4 (AssetFlowSummary), §24 (version
rejection), §6.2 (entitled proposer — `SINGLE`).

## Depends on
Tickets E1-04 (executor registration, fee accrual), E1-05 (conservation/AssetFlowSummary hook), E1-01
(batch-chain + cursor storage), E1-02 (pause gate).

## Key work
- `SubmitBatch(commitment)` — decode the canonical commitment (§19 field order;
  version fields first; reject unknown `protocol_version` before other fields).
- Acceptance checks (all MUST): caller is the executor address of an **ACTIVE**
  member whose pillar is active (§22.1, Ticket E1-04) **and** the entitled proposer
  for `domainId` at the batch's `firstInputSeq` cursor (Phase 1 `SINGLE` ⇒ "is
  the single registered executor address, and its pillar is active"); `executorId`
  equals the caller; per-domain
  contiguity against the stored cursor **and** Settlement's assigned input
  indices (§4.6) — `firstInputSeq` is the domain's earliest
  assigned-but-unconsumed index, `lastInputSeq` is assigned to the domain and
  ≤ the highest index assigned, range count ≤ `MaxBatchInputs`, no
  gap/overlap/replay; commitment size ≤ `MaxOnChainCommitmentSize (15 KiB)`;
  `assetFlowSummary` deposit correspondence against the cumulative deposit
  totals over the assigned-index range (Ticket E1-05 hook); `DAMode` is in the
  domain's approved set (§20, Ticket E1-03's `SetApprovedDAModes`); `proofData` is the
  zero-length encoding (reject non-empty / trailing bytes);
  `preStateRoot == previous accepted postStateRoot`.
- Reject a commitment whose wire `submittedAtHeight` is not `0` (§19): the executor
  cannot know the acceptance height, and zeno-executor already sends `0`.
- On acceptance: store batch `SUBMITTED`; record `submittedAtHeight` as this receive's
  receive height (`context.GetFrontierMomentum().Height`, SPEC §4.6), never the wire
  value;
  `finalizeAtHeight` (`submittedAtHeight` + the domain's withdrawal delay in force
  now, §21); and `executorPillar` — the submitting pillar's name (§19, §22.3). Both
  `batchEntry` fields were added by Ticket E1-04; this ticket sets them. Acceptance
  heights never decrease, which Ticket E1-04's fee-epoch queue relies on. Advance the
  cursor (`consumedInputCount += n`, `nextBatchId += 1`) and call Ticket E1-04's
  fee-accrual hook, passing it the batch's assigned-index records so it can sum
  their fees (at most `MaxBatchInputs` additions).
- **Collecting a batch's inputs (SPEC v1.16.6, §4.6).** Change the shared
  `admitInput` (landed with Ticket E1-05) to assign the domain's next ordinal from a
  per-domain received-input count and make the assigned-index record reachable by
  `(domainId, ordinal)` in one lookup — keyed by it, or via a separate ordinal index
  (both conformant, SPEC §4.6); the record carries `globalInputIndex`, asset,
  amount and fee. Read ordinals
  `consumedInputCount … consumedInputCount + n − 1` by key: `firstInputSeq` and
  `lastInputSeq` must equal the first and last records' `globalInputIndex`, `n ≤
  MaxBatchInputs`, and the last ordinal must exist. No scan; at most 64 lookups.
- **Cursor (§5.4).** `{consumedInputCount, nextBatchId, recordsSweptThrough,
  sweptInputCount}`; a
  domain with no accepted batch has `nextBatchId = 0`. No sentinel index value.
  The executor's recovery anchor reads it.
- **`batchId` (§19).** Must equal the cursor's `nextBatchId`: `0`, then +1 per
  accepted batch. A stored batch can never be overwritten.
- **First batch's `preStateRoot` (§19).** Must equal the genesis root: for WASM the
  empty-tree root, the 32-byte zero hash.
- **Version fields (§19, §24).** Reject any of the nine that differs from the
  Phase-1 value, which is `1` for all nine (SPEC §24: versions only count changes
  after activation). zeno-executor encodes all nine as `1` (E2-14).
- **Assigned-index record sweep (§4.6).** After acceptance, examine at most
  `MaxRecordSweepBatchesPerSubmit = 2` of the domain's batches from
  `recordsSweptThrough`, stopping at the first not `FINALIZED` at the receive
  height; delete each finalized batch's records by ordinal and advance
  `recordsSweptThrough`. Never changes an accept/reject outcome.
- Finality: a batch becomes `FINALIZED` when the frontier reaches its stored
  `finalizeAtHeight` (height-based, not wall-clock). A later `SetDomainConfig`
  change to the withdrawal delay never moves an accepted batch's finality, and
  Ticket E1-04's epoch settlement relies on that.
  `FINALIZED` is the only time gate on withdrawal release (§21). Provide the
  read predicate consumed by `RelayMessage`/`ReleaseWithdrawals` (Ticket E1-08) and
  by epoch settlement (`Update`, Ticket E1-04).
- Respect `PAUSE_SUBMIT` (Ticket E1-02). Core MUST NOT verify `preStateRoot`/
  `postStateRoot` correctness or run WASM.

## Decisions behind the commitment bytes

The executor already encodes the commitment. `CV-COMMIT-1` in zeno-executor's
`internal/codec/testdata/serialization-conformance-v1.json` is the byte-level
contract: decode exactly those bytes. A divergence in the version block rejects
every `SubmitBatch`, and it would only surface at integration. Each width below was
left unstated at some point and then fixed; the reasoning is kept here because
nothing else in the tree records it.

- **All nine version fields, `u16` each, first, in §24 order.** The commitment
  commits to roots over receipts, outbox messages, state diffs and results, whose
  own version fields are otherwise invisible on-chain. §24's unknown-rejects rule
  has to cover every format the commitment depends on transitively. Otherwise a
  version mismatch in one of those structures would be accepted silently at the
  commitment layer. The block is an 18-byte prefix.
- **`executorId` is `Address(20)`, and the stored `executorPillar` is a different
  field.** Settlement compares the wire `executorId` with `sendBlock.Address`, a
  20-byte L1 address, so 20 bytes is the only width that type-checks against the
  comparison actually made. The `batchEntry` storage variable holds
  `executorPillar`, the submitting pillar's name. It is resolved from the verified caller and never crosses the
  wire (§19).
- **`domainId` is `u32` in the `outboxId` preimage.** It is `u32` everywhere else.
  `outboxId` is a consensus preimage every Settlement instance and every executor
  derives, so the width is stated rather than left to convention (§27.4). Ticket E1-08
  verifies relay proofs against it.
- **`firstInputSeq`, `lastInputSeq` and `submittedAtHeight` are `u64`.** The first
  two bound a `globalInputIndex` and the third is a momentum height; both are `u64`
  throughout the spec.
- **`proofData` is present and empty, never omitted.** Its zero-length `Bytes`
  encoding keeps the commitment to exactly one valid encoding, so a trailing byte
  after it is a rejection, not an extension point (§19).

## Code references
- Ticket E2-03 commitment codec [NEW]; Ticket E1-01 cursor/batch storage [NEW].
- `vm/embedded/implementation/common.go` height-window helpers (`TimeChallenge`
  pattern for the delay) [EXISTS].

## Acceptance criteria
- A batch whose `batchId` is not the cursor's `nextBatchId` is rejected; a resubmission
  of an accepted `batchId` cannot overwrite it.
- A domain's first batch with a non-zero `preStateRoot` is rejected.
- A commitment with any of the nine version fields off its Phase-1 value is
  rejected.
- With another domain's inputs interleaved, a batch's inputs are read by ordinal
  with no scan, and a wrong `firstInputSeq`/`lastInputSeq` is rejected.
- Once a batch is `FINALIZED`, a later `SubmitBatch` deletes its assigned-index
  records (at most two batches per submit); records of unfinalized batches stay.
- Contiguous, correctly-chained batch from the registered executor is accepted;
  cursor advances.
- Rejections: non-proposer; gap/overlap/replay; non-`+successor`
  `firstInputSeq`; `lastInputSeq` beyond the highest Settlement-assigned index
  (receive-lag race, §4.6); `assetFlowSummary` not matching the cumulative
  deposit totals; `preStateRoot` mismatch; unknown `protocol_version`;
  non-empty `proofData`; oversize commitment; count > `MaxBatchInputs`.
- Finality timer: no withdrawal releasable while `SUBMITTED`; transitions to
  `FINALIZED` exactly at `submittedAtHeight + WithdrawalDelay`.
- Replacement-executor first batch with wrong `preStateRoot` rejected (closes the
  Ticket E1-04 loop).
