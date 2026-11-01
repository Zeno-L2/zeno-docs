# Ticket E2-18 — Conformance vectors: identities, pipeline rules, batch roots

**Epic:** 2 — Executor · **Phase:** 1C · **Order:** E2-18

## Goal
Make the execution conformance artifact extensible and freeze the per-input rules a
second implementation is most likely to get wrong, and pin the batch roots E2-19
submits. End state: vector identities cannot collide, the pipeline-ordering, blob,
limit, state, quota and deposit rules each have a vector, and `eventRoot`,
`outboxRoot` and `receiptRoot` are tested against the SPEC ordering.

## SPEC refs
SPEC §10.6, §13.8, §14.3, §18.4, §18.5, §27.1a–§27.4, §27.9, §31.

## Depends on
E2-17.

## Key work
Delivered in five `/develop` runs, squashed to one commit. Plan of record: zeno-executor
`.pipeline/research/uncovered-conformance-rules/spec.md`. Run the batch-root tests
first: they gate E2-19 and change no digest. One rule per row; each row's named
invariant proves its condition is present.

### 1. Identities and cap vectors
- `source_account_block_hash = sha256("conformance-vector:" + ID)`; one append-only
  `global_input_index` block per generator; rows sorted by batch, then index. Vector
  IDs stay as stable names.
- Cap test assets constructed so one pair differs only in byte 22 and another only in
  byte 31. CV-CAP-6 uses identical withdrawals. CV-CAP-8 to CV-CAP-10 cover a u256
  wrap and sums above 2^64.
- `TestCapVectorsRejectWrongImplementations` covers realistic step 5a bugs: wrong
  cap pairing, missing totals, truncated asset keys and narrow or wrapping sums.

### 2. Pipeline ordering, footprint, deposit (seam 1a)
- §10.6 step order wherever the status differs: steps 2 and 3, the reserved and
  duplicate passes, steps 3, 4 and 5a, step 6 against steps 2 to 4, 6a and 7.
- Step 5a footprint and deposit rows. A carry across 64-bit limbs; rollback and
  refund on the wide-sum faults.

### 3. Blob content and limits (seam 1b)
- Decode rejections, including `claimed_deposit` omitted (§27.1a) and a
  non-canonical withdrawal `AssetID`.
- Every §27.2–§27.4 count and length bound, each with its boundary row.
- Event and outbox `input_index` equal to the blob's `globalInputIndex` (§27.1a).

### 4. State, quota, deposit (seam 1c)
- First, a pure-move refactor: the attribution, decode and limit rows leave
  `callVectors` for their own generator. Regenerating after it leaves the artifact
  byte-identical.
- Decode rows the seam 1b review found missing: a non-canonical `AssetID` at byte 21
  (the last byte that must be zero); an unknown outbox `kind` followed by a valid
  withdrawal branch; event and outbox indices starting at 1; `executionresult_version`
  0; a list count one above the elements present; the EMPTY sentinel followed by
  stray value bytes.
- A write of exactly `MaxStateValueSize` bytes succeeds.
- Delete of an absent key, rewrite of the current value, exact reserved-key match,
  the absent quota leaf and the deposit refund on each reachable failure path.
- A payable `CallL2` partial claim per input, with the §18.4 identity in an
  `internal/batch` test.

### 5. Batch roots (seam 1d)
- `internal/batch` tests re-derive `eventRoot`, `outboxRoot` and `receiptRoot` from
  the SPEC ordering, independent of production helpers: several inputs, an input with
  no events or messages, and a refund after its input's contract messages.

## Code references
- zeno-executor `plugins/wasm/{vectors,callvectors,deployvectors,rederive}_test.go`,
  `plugins/wasm/testdata/execution-conformance-v1*`, `internal/batch/` [EXISTS].

## Acceptance criteria
- Every row named in the plan of record exists, or the plan records why it dropped.
- A status zeno-executor produces that differs from the plan is a production finding,
  never an edited expectation.
- SPEC §31: `claimed_deposit` omitted and the per-input partial claim move to the
  cases the artifact carries.
- `go test ./...` passes; the digest is updated and the artifact is copied to zeno
  `wasm-vm/` after each regeneration.
