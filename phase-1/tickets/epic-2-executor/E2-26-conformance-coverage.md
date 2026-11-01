# Ticket E2-26 — Conformance coverage: module behaviour, deploy, undeployed targets

**Epic:** 2 — Executor · **Phase:** 1C · **Order:** E2-26

## Goal
Freeze in the execution conformance artifact the per-input rules that the artifact
does not yet pin and that a second implementation could break while reproducing
every committed byte. End state: each rule below has a vector, or a recorded reason
why it cannot have one.

## SPEC refs
SPEC §7.4, §7.5, §8.1, §11.3, §12, §18.2, §20, §27.1, §27.1a, §27.12, §31.

## Depends on
E2-25, which changes the gas table and regenerates the artifact first. E2-18
carries the pipeline-ordering, blob, limit, state, quota and deposit rows and the
batch-root tests.

## Key work
One `/develop` run per seam, each regenerating the artifact once. One rule per row;
each row's named invariant proves its condition is present.

### 1. Module behaviour (15 to 18 rows)
- Traps, aborts and `r < 0` handling (beyond E2-20's revert-data rows), frame and memory rules, and the multi-leaf
  event tree with an odd promotion.
- The `storage_read` out-buffer order (§8.1).
- Optional: deposit refunds on a trap, a run-time `OUT_OF_GAS` and `r < 0`.

- Two SUCCESS rows the E2-18 checker review asked for:
  - three distinct events, which pins event order in the result and odd-node
    promotion in the per-input `event_root`;
  - a deposit of 1000 with claim 0 and no events or messages, so the refund is the
    only message, at `outbox_index` 0.

### 2. Deploy lifecycle (6 to 8 rows)
- Assembly and hash failures, finalize and upgrade outcomes.
- Upgrade decision order (§12): authorization before the §11.3 pipeline.
- An upgrade to a module whose `CodeHash` equals the current one is
  `VALIDATION_FAILED` (§12). zeno-executor returns `SUCCESS` today
  (`plugins/wasm/deploy.go:246-268`). Add the unit test first, then fix it: compare
  the prepared `CodeHash` with the current one after `prepareModule`.

### 3. Deploy validation (23 to 26 rows)
- Structural caps, segment fit, import and export rules, and the §11.3 rejections:
  start section, mutable global export, `_start`, `__zenon_*`.
- Measure whether a module under `MaxCodeSize` can instrument past
  `MaxInstrumentedCodeSize`. If it cannot, record why and add no row.
- The fixture builder gains custom and start sections.

### 4. Serialization suite cases
Rules no contract can trigger, so the per-input artifact cannot pin them. They go in
the serialization suite (`internal/codec/testdata/serialization-conformance-v1.json`):
- `ExecutionResult` decoder rejections: trailing bytes, field order, a `StateDiff` not
  sorted by derived key.
- An `ExecutionResult` with `MaxOutboxPerExecution + 1` messages accepted, and one with
  `MaxOutboxPerExecution + 2` rejected (§27.4).
- The `MerkleProof` rules and odd-node promotion (§27.3).

### 5. Undeployed target (2 to 3 rows)
- A `Deposit` and a payable `CallL2` to an undeployed target: `VALIDATION_FAILED`,
  full refund, and the two-leaf witness set (§18.2).
- A README shape row and a generator path with no module or frame.

### 6. Copies in other repositories
Each run updates every copy of the artifacts it changes, in the same change, as
`../../../artifacts/README.md` lists them:
- `execution-conformance-v1.json` and its README: zeno-executor `plugins/wasm/testdata/`
  (source, pinned by `vectorFileDigest`) and zeno-docs `wasm-vm/`.
- `serialization-conformance-v1.json` (seam 4): zeno-executor `internal/codec/testdata/`
  (source), zeno-docs `artifacts/`, and `zeno-client-ts` `test/fixtures/` with `SOURCE_SHA256` in
  `test/conformance.spec.ts`. go-zenon holds individual vectors as hex literals; a run
  that changes an existing vector updates them, and a run that only adds vectors leaves
  go-zenon untouched.
- `znn-typescript-sdk` needs nothing: it holds only the SMT suite and the Settlement
  fixtures dumped by go-zenon.

Other cross-repository fixtures to resolve in this ticket:
- `zeno-client-ts` `test/fixtures/gateway/gateway-go.json` is dumped by zeno-executor
  `internal/gateway/sdk_fixture_test.go` (`GATEWAY_SDK_FIXTURE_OUT`), which exists only on
  the unmerged branch `feature/gateway-sdk-fixture`. Adapt that dump test from the branch
  onto `dev` so the fixture can be regenerated, then regenerate it and check
  `zeno-client-ts`'s copy.
- `zeno-client-ts` `test/fixtures/{vault,counter,counter-v2}.interface.json` are copies of
  zeno-contract-sdk-rs's example artifacts, pinned by `FIXTURE_SHA256` in
  `test/interface-files.spec.ts`. After the SDK's phase 5 rebuilds the vault, copy any
  changed interface file and update its pin.
- List both in `../../../artifacts/README.md`, or in whichever index the reorganised zeno
  directories use, with their source, readers and pins.

## Code references
- zeno-executor `plugins/wasm/{callvectors,deployvectors,vectors,rederive}_test.go`,
  `plugins/wasm/deploy.go`, `plugins/wasm/internal/module/` [EXISTS].

## Acceptance criteria
- Every rule above has a row, or a README note saying why none is possible.
- The `CodeHash`-equality upgrade is `VALIDATION_FAILED` with the unchanged
  `CodeHash` in the receipt and no state change.
- SPEC §31's required-cases list carries no case the artifact now covers.
- `go test ./...` passes and the vector digest is updated.
- Every copy of each changed artifact listed in `../../../artifacts/README.md` has the
  source's SHA-256.
- After the last run, every checksum check in `../../../artifacts/README.md` passes for all
  four artifacts, so every repository holds the current copy of what it reads.
