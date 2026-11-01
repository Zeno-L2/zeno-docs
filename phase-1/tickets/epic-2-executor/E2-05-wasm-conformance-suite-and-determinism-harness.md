# Ticket E2-05 — WASM conformance suite and determinism harness

**Epic:** 2 — Executor · **Phase:** 1A · **Order:** E2-05

## Goal
Freeze the WASM runtime byte-for-byte in `execution-conformance-v1.json` and prove the frozen
bytes are independent of local cache warmth and CI platform. The artifact is generated from
the real `Plugin.DecodeInput`/`Plugin.Apply` seam over real modules, checked by an independent
re-derivation runner, and guarded by a whole-file digest. The ticket also lands the runtime
bounds and receipt encoding the frozen vectors depend on: the pre-decode `MaxResultBlob`
check, the `MaxInputGas` halt, the receipt `abort_code` field, and the 33rd outbox slot that
exempts the runtime deposit refund from the contract outbox cap.

## SPEC refs
SPEC §31 and §13.7 (conformance suites as normative companions), §7.2 (prohibited
capabilities), §7.3 and §20 (byte-identical results; cache-independent observed-key set),
§10.1 (`MaxInputGas`), §10.6 (ordered post-execution validation, step 0 and step 5), §14.3
(failure discard, `MaxResultBlob`), §15 and §27.7 (receipt fields, `abortCode`,
`stateDiffHash`/`returnHash` preimages), §18.5 and §27.4 (deposit refund, outbox bounds),
§25.1 (constants), §27.5 (canonical input encoding). EXECUTOR.md §2 Principle 3
(memoization tier).

## Depends on
E2-04 (all six input kinds execute through `Apply`), E2-03 (`internal/codec`, authoritative
for every encoding the vectors contain), E2-02 (`internal/state` `Store`/`Accessor`, the
witness backend).

## Key work

### Vector schema (`plugins/wasm/vectors_test.go`)
- Go structs `vectorFile`, `vectorJSON`, `vectorInput`, `vectorReceipt` declare the schema.
  The file header carries the version fields, `metering_version`, `network_name` and `chain_id`.
- Four shapes, selected by `shape`: `call` (kinds 0/1, full pipeline), `deploy` (kinds 2–5,
  code-store seeding), `validation_rejection` (module never runs) and `unsupported_kind`
  (kind 6). `input_frame_hex`, `module_wasm_hex` and `source_data_hex` are optional per shape.
- Each vector carries `canonical_input_hex` (the §27.5 `inputHash` preimage, distinct from the
  input frame), an explicit `batch_id`, `code_hash` and `instrumented_code_hash`, metered
  `gas_used`, and `witness_keys` as the ascending read ∪ written union.
- Each vector also carries `execution_result_blob_hex` (rebuilt through
  `codec.ExecutionResult.Encode`), `receipt_hex`, and a decoded receipt view including
  `abort_code`, `state_diff_hash` and `return_hash`.
- Kind-specific input fields that can be zero or empty are pointer types, so a zero value and
  an absent field stay distinguishable.

### Generator and tripwire
- `TestGenerateVectors` rebuilds the whole artifact and byte-compares it with
  `testdata/execution-conformance-v1.json`; `-update` rewrites the file instead.
- Each call vector validates and instruments its module, seeds a fresh code store, and opens a
  `state.Store` on a log under `t.TempDir()`. It passes that store's `state.Accessor` to
  `Plugin.Apply`, never the `mapState` test helper.
- `assertUniqueIdentities` fails the run on a duplicate vector ID, `SourceAccountBlockHash`, or
  `(batch_id, global_input_index)` pair. `newIdentity` allocation is append-only.
- `TestVectorFileDigest` compares the stdlib SHA-256 of the whole file with the hand-recorded
  `vectorFileDigest` literal. A mismatch fails with a "consensus change" message.
- `plugins/wasm/fixtures_test.go` provides `buildWasmModule`: arbitrary types, arbitrary
  `(module, name)` imports, arbitrary bodies, one memory and active data segments.

### Reference implementation (`plugins/wasm/reference_test.go`)
- Test-only receipt assembly: `refStateDiffHash`, `refReturnHash`, `refEventRoot`,
  `refOutboxRoot`, `refWitnessKeys` and `refDerive`. It carries `AbortCode` into the receipt.
- It adds no helper to `internal/codec`, which stays consume-only.

### Independent re-derivation (`plugins/wasm/rederive_test.go`)
- `TestConformanceVectors` re-derives every committed value without the reference
  implementation, and without `internal/codec` helpers for the value under test.
- It derives `post_state_root` through `trie.RootOfLeaves` and the event and outbox roots
  through `inlineMerkleRoot`. It assembles `receipt_hex` byte by byte with `encoding/binary`.
- `checkPathGas` re-derives `gas_used` from the gas table without calling `effects.go`.
- Per-shape checks (`checkCallFrame`, `checkCallRefund`, `checkDeploy`,
  `checkValidationRejection`, `checkUnsupportedKind`) cover each shape's fields.
- `namedInvariants` binds vector-specific properties, labelled `INV-n`, to named vectors.

### Vector set (`callvectors_test.go`, `deployvectors_test.go`)
- Spine: `EXEC-001` (pure return), `EXEC-002` (single write plus an absent-key `storage_len`
  observation), `EXEC-FAIL-OUTBOX-1` (failed input with a deposit, refund at `outbox_index` 0).
- Call path: `EXEC-003`…`EXEC-012`, `EXEC-DEP-1`…`EXEC-DEP-5`, `CV-OUTBOX-1`, `CV-ROOTS-1`,
  `CV-DUPKEY-1`, `CV-UNAUTH-1`, `CV-RTL-RETURN-1`, `CV-RTL-RESULT-1`, `CV-RTL-BLOB-1`/`-2`.
- Engine edges: `CV-OOG-1`…`CV-OOG-4` (all four `OUT_OF_GAS` forms), `CV-READ-2`,
  `CV-OOB-1`, `CV-GROW-1`, `CV-DEPTH-1`, and the four `CV-SEG-*` segmentation cases.
- Deploy: `CV-DEPLOY-1`/`-2` (single-chunk sizing), `EXEC-DEPLOY-1`, `EXEC-UPGRADE-1`,
  `CV-UPGRADE-UNDEPLOYED-1`, `CV-UPGRADE-UNAUTH-1`, and one `CV-REJECT-*` vector per §7.2
  class. `rejectionReasons` maps each `CV-REJECT-*` vector to its validator error.
- `EXEC-UNSUP-1`: `input_kind = 6` yields `UNSUPPORTED_OPERATION`, `gas_used = 0`, no outbox,
  and the 33-byte prefix as `canonical_input_hex`.

### Artifact README (`plugins/wasm/testdata/execution-conformance-v1-README.md`)
- States the schema, encoding rules, regeneration command, and a Coverage table naming every
  vector. It carries no STALE banner.
- A Provenance section names the open items whose resolution would re-date frozen bytes
  (`OD-4`'s possible blob-length gas term), the receipt preimage and discard rules, and
  `EXEC-UNSUP-1`'s encoding.
- It names the pins the bytes depend on: the exact `wazero` version in `go.mod` and
  `wasm-gas-table-v1` at its `metering_version`.
- It records that the copy under the zeno docs `wasm-vm/` is synced by hand.

### Determinism harness (`plugins/wasm/determinism_test.go`)
- `TestDeterminismCacheMatrix` runs every vector `determinismRuns` (3) times in each of four
  `cacheCells`: {warm, cold} compiled-module cache × {warm, cold} code store.
- A cold compiled-module cache is a fresh `Plugin` per run. A cold code store means the
  instrumented tier is absent while the assembled tier is present.
- Each run byte-compares status, `gas_used`, the `ContractEffects`, `ExecutionResult` and
  receipt blobs, `post_state_root` and the witness set against the frozen row.
- `TestDeterminismWitnessBackendTripwire` reruns `EXEC-002` over `mapState`. It checks status
  and `gas_used` match, then shows the witness set is strictly smaller.
- `Plugin.Close()` in `plugins/wasm/wasm.go` delegates to `engine.Engine.Close()`. The
  generator and harness close every `Plugin` they construct.

### CI matrix (`.github/workflows/ci.yml`)
- Job `test` runs build, vet, test and `scripts/check-trie-imports.sh` on `ubuntu-latest` and
  `macos-latest`, with `fail-fast: false`.
- Job `build` (`needs: test`, `if: always()`) is green only when every matrix leg succeeded.
  It keeps the check name branch protection requires.

### Runtime bounds (`plugins/wasm/limits.go`, `effects.go`, `call.go`)
- `MaxResultBlob` (64 KiB) bounds the returned `ContractEffects` blob. Step 0 of `validate`
  checks its length before decode and returns `RESULT_TOO_LARGE` with metered `gas_used`.
- An over-long blob is `RESULT_TOO_LARGE` whether or not it is also malformed
  (`CV-RTL-BLOB-2`).
- `MaxInputGas` bounds a `CallL2` or `Deposit` `plasma_limit`, below `2^63 − 1`. `applyCall`
  returns an `error` (halt) for a larger value before seeding the budget, and never clamps.

### Receipt and outbox encoding (`internal/codec`)
- `codec.Receipt` gains `AbortCode int32`, encoded as a big-endian two's-complement `i32`
  between `gas_used` and `state_diff_hash`. `receipt_version` changes with it; E2-14 sets the
  version fields.
- `DecodeReceipt` rejects any `receipt_version` other than `codec.ReceiptVersion` with
  `ErrUnknownVersion`.
- `MaxOutboxPerResult = MaxOutboxPerExecution + 1`. `writeOutboxList`/`readOutboxList` take the
  bound: `ContractEffects` uses `MaxOutboxPerExecution`, `ExecutionResult` uses
  `MaxOutboxPerResult`.
- `settle` step 5 checks the refund against `MaxOutboxPerResult`. A contract emitting 32
  messages with a non-zero remainder succeeds with the refund at `outbox_index` 32
  (`EXEC-DEP-5`).
- A zero remainder appends no refund and no padding.

### Companion suites
- `internal/codec/testdata/serialization-conformance-v1.json` is the source copy of the
  serialization suite, published byte-identical under the zeno docs `artifacts/`. Its
  receipt-bearing vectors and version block carry the receipt encoding above.
- `TestArtifactDrift` in `plugins/wasm/internal/module/gastable_test.go` holds the embedded gas
  table equal to `internal/module/testdata/wasm-gas-table-v1.json`. It runs on every
  `go test ./...` and never skips.
- `TestApplyMixedPresentEmptyAndDelete` in `internal/state/conformance_test.go` covers §31
  `CV-APPLY-1`: one diff with a present-empty write and a delete, checked against
  `trie.RootOfLeaves`.

## Decisions
- **The vectors outrank the reference implementation.** `reference_test.go` is test-only and
  subordinate to the frozen bytes. E2-07's production receipt assembler must reproduce the
  vectors; where they disagree, the vectors win.
- **Witness sets come from a `state.Store`-backed `Accessor`.** `mapState` records only `Read`,
  so it would drop every absent-key `storage_len` observation from a consensus-committed field.
- **The digest covers the whole file.** Digesting only module bytes would let `-update` launder
  a hand-edited `gas_used` or `receipt_hex`. The literal sits where `-update` cannot write.
- **Re-derivation shares no code with generation.** Sharing the reference implementation or
  codec helpers would make the check self-consistency rather than independent derivation.
- **Vector identities are append-only.** Inserting a vector mid-sequence renumbers published
  vectors' `global_input_index`/`source_account_block_hash` and changes their bytes. New
  vectors take the next unused identity.
- **"Cold code store" means the instrumented tier is absent.** With both tiers absent, module
  loading halts, which is not a vector outcome.
- **The `ubuntu-latest`/`macos-latest` matrix is a proxy.** It does not discharge §31's
  requirement for a second independent implementation.
- **Step 5's refund-capacity check stays although unreachable.** Step 1's decode already caps
  contract messages. The check mirrors the `MaxOutboxPerResult >= MaxOutboxPerExecution + 1`
  invariant that `ExecutionResult.Encode` relies on. If that invariant broke, a contract input
  would get `RUNTIME_FAULT` instead of a halt.
- **The codec does not identify the refund slot.** `ExecutionResult` decode bounds the outbox
  at `MaxOutboxPerResult` only; attributing slot 33 to the refund is the runtime's invariant.
- **`engine.Run` keeps its internal i64 budget clamp** as a defensive guard on a value
  `applyCall` no longer admits.
- **The two artifact copies are synced by hand.** The zeno docs copy lives in another
  repository, so no drift test compares it. The gas table's vendored copy is in this repository,
  so `TestArtifactDrift` checks it.

## Code references
- `plugins/wasm/vectors_test.go`, `fixtures_test.go`, `reference_test.go`,
  `rederive_test.go` [EXISTS] — schema, generator, tripwire, reference implementation,
  re-derivation.
- `plugins/wasm/callvectors_test.go`, `deployvectors_test.go` [EXISTS] — vector specs.
- `plugins/wasm/determinism_test.go` [EXISTS] — cache matrix and witness-backend tripwire.
- `plugins/wasm/testdata/execution-conformance-v1.json`, `-README.md` [EXISTS] — the artifact.
- `plugins/wasm/limits.go`, `effects.go`, `call.go`, `wasm.go` [EXISTS] — `MaxResultBlob`,
  `MaxInputGas`, step 0 and step 5, `Plugin.Close()`.
- `internal/codec/receipt.go`, `primitives.go`, `outbox.go`, `effects.go`, `constants.go`
  [EXISTS] — `AbortCode`, `i32`, outbox bounds.
- `internal/codec/testdata/serialization-conformance-v1.json`,
  `plugins/wasm/internal/module/testdata/wasm-gas-table-v1.json`,
  `plugins/wasm/internal/module/gastable_test.go`, `internal/state/conformance_test.go`
  [EXISTS] — companion suites.
- `.github/workflows/ci.yml` [EXISTS] — platform matrix.

## Acceptance criteria
- `TestGenerateVectors` regenerates a file byte-identical to the checked-in artifact, and
  `TestConformanceVectors` re-derives every vector green.
- `TestVectorFileDigest` fails on any hand-edit of the JSON file.
- `assertUniqueIdentities` passes over the full spec list.
- The README Coverage table lists every vector in the file, and the file has no STALE banner.
- Every `CV-REJECT-*` vector maps to its validator error in `rejectionReasons`.
- `EXEC-002`'s `witness_keys` contains its absent-key `storage_len` path.
- `TestDeterminismCacheMatrix` passes in all four cells.
- `TestDeterminismWitnessBackendTripwire` shows the `mapState` substitution yields a strictly
  smaller witness set.
- CI is green on both `ubuntu-latest` and `macos-latest`; `build` fails if either leg fails.
- `TestSettleOverLongBlobRejectedBeforeDecode`, `TestSettleBlobAtBoundAccepted`,
  `CV-RTL-BLOB-1` and `CV-RTL-BLOB-2` show step 0 precedes decode.
- `TestApplyPlasmaLimitAboveMaxInputGasHalts` halts on a kind-0 and a kind-1 input above
  `MaxInputGas`.
- `TestReceiptAbortCodeRoundTrip`, `TestReceiptTruncated`, `TestExecutionResultOutboxAtMaxPerResultAccepted`
  and `TestContractEffectsOutboxOverCapRejected` pass.
- `TestSettleOutboxAtCapacityAppendsRefund`, `TestSettleOutboxNoPadOnZeroRemainder`,
  `TestSettleReceiptAbortCode` and `EXEC-DEP-5` pass.
- `TestArtifactDrift` and `TestApplyMixedPresentEmptyAndDelete` pass without skipping.
- Three §31 cases stay outside this artifact: a payable `CallL2` with a partial claim plus the
  §18.4 batch identity; `claimed_deposit` omitted; a `Deposit` to an undeployed contract.
- `go build ./... && go vet ./... && go test ./...` green.
