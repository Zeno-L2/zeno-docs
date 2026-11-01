# Ticket E2-09 — Conform the executor to SPEC v1.14.0 / EXECUTOR v0.7.0

**Epic:** 2 — Executor · **Phase:** 1A · **Order:** E2-09
**Size:** M–L — four consensus-visible changes (step 5a, contract metadata, input kind 6, witness values), a regeneration of both vector suites, and comment re-pointing across ~16 files.

## Goal
zeno-executor matches SPEC v1.14.0 and EXECUTOR v0.7.0 exactly. This covers the step-5a
deposit rule, the contract-metadata transport and `TransferContractOwnership`, witness values
in the bundle, the conformance cases §31 now requires, and code comments that cite
normative text instead of the removed `UPSTREAM-AMENDMENTS.md` files. When it lands, no
consensus-visible behaviour differs from the spec, and no comment cites a document that no
longer exists.

## SPEC refs
SPEC §10.6 (step 5a and the per-status table), §27.9 (per-input bound, value in force at the
input's momentum), §20 (`deployChunkData`), §31 (the step-5a cases), §11.1/§11.3/§12/§27.12 (contract metadata, upgrade policy and owner), §12/§15/§27.5 (`TransferContractOwnership`, `input_kind = 6`), §20 (`WitnessEntry.value`). EXECUTOR §14 (`AWAIT_DA` defaults), §15 (znnd retry, reorg rollback and
plugin-local stores), §16 (bundle retention).

## Depends on
Tickets E2-07 and E2-08 (delivered). It needs the zeno branch that carries SPEC v1.14.0 merged
first, since the vector files it regenerates record that version. Settlement's side of the
metadata argument and kind 6 is Tickets E1-06 and E1-09.

## Key work

### 1. Step 5a: exclude the deposit amount from the per-asset total (consensus-visible)
- `plugins/wasm/effects.go` step 5a (`:123-156`): stop adding `s.DepositAmount` to its asset's
  amount total. Keep `s.DepositAsset` in the distinct-asset footprint whenever
  `s.DepositAmount` is non-zero. Rewrite the step's comment to state the rule as §27.9 does.
- This changes a committed status. The shipped rule gives `RUNTIME_FAULT` for an input whose
  deposit alone, or deposit plus withdrawals in that asset, exceeds the cap. The corrected rule
  lets that input run.
- **u256 invariant:** a lone input's `withdrawalDebit` is its withdrawals plus the refund. That
  is at most `MaxBatchWithdrawal + deposit`, which is ≤ 2^255 + (2^255 − 1) = 2^256 − 1, because
  `TokenMaxSupplyBig` is 2^255 − 1 (go-zenon `vm/constants/embedded.go:75`). So
  `internal/batch/builder.go`'s `breach()` never rejects a lone input for overflow. State this
  next to `codec.MaxBatchWithdrawal`, because any placeholder above 2^255 breaks it.
- Tests in `plugins/wasm/effects_test.go`:
  - `DepositAloneOverCap` (`:816`) moves to `TestSettleStep5aAdmits`, and must now settle
    `SUCCESS`.
  - Add a case where a withdrawal exactly at the cap and a deposit share one asset, and the
    input is admitted.
  - Keep `SixteenAssetsPlusDepositSeventeenth` unchanged.
  - Add a `internal/batch` test: a lone input carrying a worst-case deposit and a
    withdrawal at the cap in the same asset is batchable alone.

### 2. `codec.MaxBatchWithdrawal`: document the value it stands in for
- `internal/codec/constants.go:45-53`: the comment names the §27.9 rule. Step 5a reads the
  Periphery value in force at the momentum containing the input. The batch builder splits
  against the value it expects Settlement to apply when it registers the batch's withdrawals.
  2^255 remains a placeholder until the Periphery read is wired.
- Do not wire the read here. Add a Key-work bullet to ticket E2-17 saying so: step 5a must read the
  value at the input's momentum, not at submission.

### 3. Conformance cases §31 now requires
- **Execution suite** (`plugins/wasm/testdata/execution-conformance-v1.json`): append two
  vectors, following the suite's id convention and append-only rule:
  1. Contract-emitted withdrawals across `MaxAssetsPerBatch + 1` distinct assets produce
     `RUNTIME_FAULT`, with metered gas and no effect charge.
  2. A payable input whose withdrawals name `MaxAssetsPerBatch` assets, and whose deposit asset
     is one more, produces the same result.
  - Update the README's Coverage table.
  - Replace the README's `../UPSTREAM-AMENDMENTS.md`, D32/D33 and A12 citations (`:22`,
    `:96-97`) with SPEC §14.3, §15 and §30 `OD-4`.
- **Serialization suite** (`internal/codec/testdata/serialization-conformance-v1.json`):
  - Append `CV-BUNDLE-2`, a bundle holding a kind-3 record with `deployChunkData` followed by a
    non-kind-3 record. It fixes the §20 framing byte for byte.
  - Set `"spec"` to v1.14.0.
  - Replace the `notes` line that cites `UPSTREAM-AMENDMENTS.md` with the zeno path the copy
    syncs to.
  - In the same change, add `CV-BUNDLE-2` to SPEC §31's serialization list, which says the suite
    holds "exactly" the ids it names.
- Re-sync all three copies into zeno's `wasm-vm/` and `artifacts/` byte for byte, and confirm the
  hashes match.

### 4. Reorg rollback comments: the reachable hazard, cited normatively
- `internal/recovery/boot.go:574-586` and `internal/batch/driver.go:158-171` describe a stale
  *chunk* slot surviving rollback. That path can't happen: Settlement accepts a finalize only when
  every chunk is present (SPEC §11.3), and replay rewrites every such chunk.
- Rewrite both comments to describe the reachable path in EXECUTOR §15. The retracted branch sees
  a pending record expire and accepts a second `DeployContractStart` for the same `code_hash`.
  That record survives rollback, and a canonical finalize for the first record then assembles
  against it.
- Cite EXECUTOR §15 in place of the removed `internal/recovery/UPSTREAM-AMENDMENTS.md §4`.
- Optional: a `plugins/wasm` test that builds that sequence against the code store, so the
  hazard is executable rather than only described.

### 5. Re-point every decision-id citation to normative text
- Comments cite `D1`–`D35`, `OD-7` and removed-file paths in about 70 places:
  - `plugins/wasm`: `limits.go`, `deploy.go`, `effects.go`, `wasm.go`, `call.go`, `keys.go`,
    `codestore.go`, `internal/engine/engine.go` and their tests.
  - `internal/codec/bundle.go:25` and `bundle_test.go:148`.
- Replace each one with the SPEC or EXECUTOR section that now states the rule. The mapping is in
  the removed file's Basis column: `git show 57f595b:plugins/wasm/UPSTREAM-AMENDMENTS.md` (and the
  `internal/codec` and `internal/recovery` files at the same commit).
- `rederive_test.go`'s `A1:`/`A2:` prefixes are local assertion labels, not amendment ids. Leave
  them.
- A row whose Basis cites no normative section has no home in zeno. Report it rather than
  dropping it.

### 6. Verify, no change expected
- The `RetryPolicy` defaults in `internal/recovery/boot.go:61-69` equal EXECUTOR §14/§15: 5
  attempts at 200 ms for `AWAIT_DA`, and 5 attempts at 1 s for the source retry.
- `BundleStore` exposes no delete (EXECUTOR §16 retention).
- The bundle encoder emits `deployChunkData` only for kind 3 (`internal/codec/bundle.go:61-65`).

### 7. Contract metadata and upgrade policy (consensus-visible)
- `internal/codec`: `DeployStartArgs` gains a trailing `Metadata []byte`; the kind-2 tail
  becomes `code_hash || metadata_hash || total_size || chunk_count || metadata:Bytes` (§27.5).
- `plugins/wasm/wasm.go` `DecodeInput`: store `metadata` in the pending record. The record is
  no longer exactly 64 bytes, so update the width check that treats any other length as a
  halt (`deploy.go`, the EXECUTOR §15 code-store row) to the new layout.
- `plugins/wasm/deploy.go` finalize: validate `metadata` per §27.12 — version `1`, exactly 23
  bytes, policy `0` or `1`, owner zero ⟺ `IMMUTABLE`. Any failure is `VALIDATION_FAILED` with
  no contract. On success write the policy byte, and for `OWNER_UPGRADEABLE` the owner leaf
  `SHA3-256("L1_CALLER:" || owner)`, in place of today's unconditional `IMMUTABLE` write
  (`deploy.go:185`). Drop the `MetadataHash has no consumer` comment (`deploy.go:29`).
- `applyUpgrade` keeps the policy and owner leaves unchanged and ignores the upgrade record's
  metadata. Its no-code and unauthorized paths already match §10.6 and §15.

### 8. `TransferContractOwnership`, `input_kind = 6` (consensus-visible)
- `internal/codec`: a kind-6 input, `contract_id:bytes32 || new_owner:Address(20)`, and
  `UNSUPPORTED_OPERATION` now starts at kind 7.
- `plugins/wasm`: an `applyTransferOwnership` that reads the `CodeHash`, policy and owner leaves
  unconditionally and in that order. It returns `VALIDATION_FAILED` when there is no code, and
  `UNAUTHORIZED` when the policy isn't `OWNER_UPGRADEABLE` or the owner leaf isn't `caller`.
  Otherwise it returns `SUCCESS` with a `StateDiff` that either sets the owner leaf or, for a
  zero `new_owner`, writes `IMMUTABLE` and deletes the owner leaf. `gas_used = 0`, no events, no
  outbox messages, and a receipt `contractId`/`codeHash` per §15.
- These writes target reserved keys from the runtime, not from a contract blob, so the step-3
  reserved-key scan does not apply to them.

### 9. Witness values in the bundle (`WitnessEntry.value`)
- `internal/codec` `WitnessEntry` gains `Value`, encoded between `derivedKey` and `proof`: the
  value as `Bytes`, `L = 0x00000000` for present-empty, and the `EMPTY` sentinel for absent (§20).
- `internal/batch/driver.go:337` already receives `present, value` from `Store.Prove` and
  discards them; carry them into the entry.
- No `damode_version` change. No production bundle exists yet.

### 10. Regenerate both vector suites
Items 7–9 change committed bytes that existing vectors freeze, so the suites are regenerated
rather than only appended:
- **Serialization:** `CV-INPUTROOT-1` (the kind-2 tail) and `CV-BUNDLE-1` (the witness layout)
  change. Add a kind-6 input to `CV-INPUTROOT-1`'s mixed batch, and a `CV-META-1/2` pair
  (accept a well-formed version-1 encoding; reject each malformed shape).
- **Execution:** every deploy vector's `canonical_input_hex` and receipt change with the kind-2
  tail. Add vectors for:
  - an `OWNER_UPGRADEABLE` deploy followed by an authorized upgrade;
  - a transfer, a renounce, and an upgrade after a renounce (`UNAUTHORIZED`);
  - a transfer by a non-owner;
  - a transfer naming no deployed contract;
  - each malformed-metadata `VALIDATION_FAILED` case.
- List every new id in SPEC §31 in the same change, and re-sync the zeno copies.

### 11. Remaining `EXTERNAL_OBSERVED` reference
`domain/domain.go:140`'s `Source` comment names `EXTERNAL_OBSERVED`; only `L1_RELAYED` domains
implement `Source` now (SPEC §6.1).

## Out of scope
- The benchmark behind `OD-4`/`OD-5` (Ticket E2-11).
- `MINTED`, the verifier registry and timeouts (SPEC §5.5, §17.4, §18.4a). All are reserved and
  have no Phase-1 executor behaviour.
- The Periphery read itself (ticket E2-17).

## Acceptance criteria
- `go test ./...` passes. The step-5a tests assert the corrected rule, including a deposit above
  the cap settling `SUCCESS`.
- Both vector files carry the new cases, each vector passes, and the three zeno copies
  hash-match their sources.
- `grep -rnE "UPSTREAM-AMENDMENTS|\bD[0-9]{1,2}\b|OD-7"` over `*.go` and `testdata/` returns
  nothing.
- SPEC §31 lists `CV-BUNDLE-2`, `CV-META-1/2` and every new execution vector id.
- A deploy with `OWNER_UPGRADEABLE` metadata, followed by an owner's upgrade and then a transfer,
  runs end to end through the batch driver, and its bundle replays from DA alone using the
  witness values.
