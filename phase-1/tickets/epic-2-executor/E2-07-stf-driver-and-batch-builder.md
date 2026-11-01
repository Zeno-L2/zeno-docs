# Ticket E2-07 — STF driver, batch builder, commitment & DA bundle

**Epic:** 2 — Executor · **Phase:** 1A · **Order:** E2-07

## Goal
The execution loop that feeds decoded inputs to `plugin.Apply` one at a time with
the witnessing accessor, collects effects, computes `pre`/`postStateRoot`,
charges effect gas, slices the domain subsequence into contiguous batches, and
assembles the batch commitment + DA bundle (`DAHash`). End state: a full batch
commitment and bundle are produced and self-verified against the WASM plugin, with
the `DAHash` round-trip enforced.

## SPEC refs
SPEC §13.5 (apply-on-success), §14 (results), §15 (receipts), §16 (events),
§17 (outbox), §18.4 (AssetFlowSummary), §19 (commitment), §20 (DA bundle);
EXECUTOR.md §5.2, §5.4, §9.

## Depends on
Tickets E2-02 (store/witness), E2-03 (serializer), E2-04 (WASM plugin), E2-06 (ordered
inputs). Ticket E2-04 has shipped, so this ticket drives the real `Apply` across
all six input kinds — there is no stub path to build.

`gas_used` per status and the ordered post-execution validation are fixed in
§10.6, so the committed `Receipt` this ticket assembles has a defined value for
every field on every path. The receipt carries `abortCode` (§15, §27.7) at
`receipt_version = 2`.

## Key work
- Per input: `DecodeInput → Apply (witnessed) → stage effects → receipt`;
  compute `preStateRoot` (before) and `postStateRoot` (after applying the
  staged `StateDiff` on success only).
- Batch slicing: contiguous over **this domain's subsequence** up to
  `MaxBatchInputs (64)`; a batch begins at the successor of the domain's own
  cursor, never `globalIndex+1`; ≥1 input per batch.
- Assemble roots: `inputRoot`, `receiptRoot`, `eventRoot`, `outboxRoot` (§27.3
  construction, correct orderings), and `AssetFlowSummary` (deposit credit /
  withdrawal debit per asset, incl. refund debits, ≤ `MaxAssetsPerBatch`).
- Build the canonical DA bundle (`BundleInputRecord` per input, ascending
  `globalInputIndex`, witnesses ascending by derived key, no duplicates);
  `DAHash = sha3(bundle)`.
- **Fail-closed `DAHash` round-trip**: reconstruct from the persisted bundle,
  confirm equality before any submit; mismatch → halt (EXECUTOR.md §5.4, §15).
- Enforce per-batch `MaxBatchWithdrawal` aggregation rules and the refund
  exemption (§27.9) at build time so batches are never built unsubmittable.
- **Production `witness_keys` (`D14`).** The union of keys read and keys written
  has no production assembler: only the conformance harness builds it today, from
  the accessor's observations plus the input's writes. This ticket owns the
  version the DA bundle commits.
- **`deployChunkData` in `BundleInputRecord` (`OD-7`).** §20's bundle layout has
  no field for a kind-3 chunk's bytes, so a DA-only replayer cannot reconstruct a
  deployed module. Adding one is a bundle-layout increment and this ticket is
  where it lands.
- **Old-`CodeHash` retention (`A6`).** §12 requires the previous `CodeHash` be
  retained in historical state. `internal/state` keeps only current leaves, so
  retention rests on the bundle carrying every deploy and upgrade input with its
  `ExecutionResult`; this ticket owns the bundle side and Ticket E2-08 the retention
  window.

## Code references
- Tickets E2-02/E2-03 packages [NEW].
- SPEC §20 bundle layout; §27.3 Merkle [spec-defined].

## Acceptance criteria
- Replay determinism: same inputs → identical roots, `ExecutionResult`s, and
  `DAHash` across runs.
- `DAHash` round-trip verified; a deliberately corrupted bundle halts before
  submit.
- A batch with one absent read produces the CV-BUNDLE-1 `DAHash`.
- Batch never exceeds `MaxBatchInputs`; per-domain contiguity holds across a
  multi-batch slice.
