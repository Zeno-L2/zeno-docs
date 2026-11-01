# Ticket E2-14 — Storage quota, BuyStorage and Phase-1 version fields

**Epic:** 2 — Executor · **Phase:** 1A · **Order:** E2-14

## Goal
Charge contract state against a prepaid byte quota, and set every version field to its Phase-1
value. End state:
1. The WASM plugin keeps each contract's quota in the reserved `0x0005` leaf.
2. A `CallL2` or `Deposit` whose net state growth exceeds the quota fails with
   `STORAGE_QUOTA_EXCEEDED`.
3. The plugin applies `BuyStorage` (`input_kind = 7`).
4. All nine version constants are `1` (SPEC §24).

Both suites are regenerated once, at the final version values.

## SPEC refs
SPEC §13.8 (storage quota), §13.3 (reserved keys), §10.6 (step 6a and the `gas_used` rows),
§10.7 (storage fee, `bytes` at least 1), §15 (status `8`, kind-7 `contractId`/`codeHash`), §19
(commitment version block), §24 (version fields), §25.1 (`StateLeafOverhead`), §27.5 (kind-7
encoding), §27.7 (receipt encoding), §31 (`CV-QUOTA-*`, `CV-INPUTROOT-1`, `CV-COMMIT-1/2`).

## Depends on
E2-03 (codec), E2-04 (WASM plugin), E2-07 (batch builder). Ticket E1-07 (Settlement) rejects a
commitment whose version fields are not all `1`, so integration with it needs this ticket.

## Key work

### Codec (`internal/codec`)
- `ReceiptVersion` is `1` in `constants.go`, so all nine version constants are `1`.
  `Receipt.Encode` and `DecodeReceipt` reject any other `receipt_version` with
  `ErrUnknownVersion`.
- `StatusStorageQuotaExceeded = 8` in `receipt.go`, with `String()` returning
  `STORAGE_QUOTA_EXCEEDED`. Encode and decode reject a status above `8` with `ErrInvalidStatus`.
- `BuyStorage InputKind = 7` and `BuyStorageArgs{ContractID, Bytes}` in `input.go`. The tail is
  `contract_id:bytes32 || bytes:u64`.
- `Input.Encode` and `DecodeInput` reject `Bytes == 0` with `ErrZeroStorageBytes` (§10.7).

### Quota leaf and reserved key (`plugins/wasm/keys.go`, `plugins/wasm/quota.go`)
- `keyQuota = 0x0005` joins `reservedKeys`. A contract `StateDiff` entry writing it is
  `UNAUTHORIZED` at step 3.
- The leaf is `purchased:u64 || used:u64`, big-endian, 16 bytes (`quotaLeafLen`). An absent
  leaf decodes as `0/0`.
- `leafSize(len)` is `StateLeafOverhead + len`. `StateLeafOverhead = 64` in
  `plugins/wasm/limits.go`.

### Step 6a (`plugins/wasm/effects.go`, `plugins/wasm/quota.go`)
`settlement.validate` calls `chargeQuota` after the step-6 effect charge and before step 7.
`applyCall` passes the input's `StateAccess` into `settlement.State`, for both `CallL2` and
`Deposit`.
- `chargeQuota` sums `delta` over the derived entries. It adds `leafSize(len(new))` for each
  present value and subtracts `leafSize(len(pre))` for each present pre-state value, read with
  `State.Len`.
- When `delta = 0`, it reads no quota leaf and adds no entry. The `0x0005` leaf stays out of
  the witness set.
- When `delta > 0` and `used + delta` exceeds `purchased`, the input fails
  `STORAGE_QUOTA_EXCEEDED` at metered plus effect gas. It has full rollback and a full deposit
  refund.
- Otherwise it appends the updated quota entry to the diff. `sortStateDiff` then places it at
  its canonical position.

### BuyStorage (`plugins/wasm/quota.go`, `plugins/wasm/wasm.go`)
`Plugin.Apply` dispatches `codec.BuyStorageArgs` to `applyBuyStorage`. `DecodeInput` accepts
kinds `0..7`; any other kind is `UNSUPPORTED_OPERATION`.
1. It reads the target's `keyCodeHash` leaf, then its `keyQuota` leaf, before any branch.
2. An absent code-hash leaf yields `VALIDATION_FAILED`.
3. `purchased + bytes` above `2^64 − 1` yields `VALIDATION_FAILED`, carrying the unchanged
   `CodeHash`.
4. Otherwise it returns `SUCCESS` with one quota entry, `purchased + bytes` and `used`
   unchanged.

Every outcome has `gasUsed = 0`.

### Conformance suites
- The execution suite seeds quota (`seedQuota`) for every vector that writes contract state.
  It adds the `buy_storage` shape. The unsupported-kind vector uses kind `8`.
- The execution suite adds `CV-QUOTA-1`…`CV-QUOTA-8`, documented in
  `execution-conformance-v1-README.md`. `rederive_test.go` re-derives each quota outcome
  independently.
- The serialization suite adds a `BuyStorage` leaf to `CV-INPUTROOT-1`. It carries
  `receipt_version` `1` in the `CV-COMMIT-1/2` version block.
- zeno `wasm-vm/` and `artifacts/` hold byte-identical copies of `execution-conformance-v1.json`,
  `execution-conformance-v1-README.md` and `serialization-conformance-v1.json`.

## Decisions
- **Quota invariant violations halt the executor; they are not a status.** A quota leaf whose
  width is not 16, or a `delta` that frees more than `used`, returns an error. Neither can
  arise from consensus-valid state, so a status would hide corruption.
- **`BuyStorage` reads both leaves before it branches.** The witness set is then the `0x0001`
  and `0x0005` leaves on every outcome. A verifier needs no outcome-specific witness rule.
- **`bytes = 0` is a codec error, not a receipt status.** Settlement enforces `bytes ≥ 1`
  (§10.7) before an input exists. A zero reaching the executor is an internal fault.
- **The version reset and the quota land together.** A version change rewrites every receipt
  and every `receiptRoot`. Landing both regenerates the suites once.

## Code references
- `internal/codec/constants.go`, `receipt.go`, `input.go`, `errors.go` [EXISTS].
- `plugins/wasm/quota.go` [EXISTS] — `decodeQuota`, `encodeQuota`, `leafSize`,
  `chargeQuota`, `applyBuyStorage`.
- `plugins/wasm/effects.go`, `keys.go`, `limits.go`, `wasm.go`, `call.go` [EXISTS].
- `plugins/wasm/testdata/execution-conformance-v1.json`,
  `internal/codec/testdata/serialization-conformance-v1.json` [EXISTS].

## Acceptance criteria
- `TestChargeQuota` passes: a write within quota commits the contract's entries plus the
  updated `0x0005` entry, and a write beyond it fails with no state change.
- `TestChargeQuotaDeleteThenReuse` passes: a delete lowers `used`, and a later write reuses
  the freed bytes. `TestChargeQuotaZeroDeltaLeavesQuotaOutOfWitnessSet` passes.
- `TestApplyBuyStorage` covers success, undeployed target and overflow, each at `gasUsed = 0`.
  `TestApplyBuyStorageHaltsOnBadLeafWidth` passes.
- `TestReceiptVersion1AcceptedVersion2Rejected` and `TestReceiptStorageQuotaExceededRoundTrip`
  pass. A status byte of `9` is rejected.
- `CV-QUOTA-1`…`CV-QUOTA-8`, `CV-INPUTROOT-1` and `CV-COMMIT-1/2` pass. The zeno copies in
  `wasm-vm/` and `artifacts/` are byte-identical to zeno-executor's.
- `go test ./...` passes.
