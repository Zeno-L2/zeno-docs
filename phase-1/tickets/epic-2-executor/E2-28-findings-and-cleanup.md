# Ticket E2-28 — Findings and cleanup

**Epic:** 2 — Executor · **Phase:** 1C · **Order:** E2-28

## Goal
Fix what the Ticket E2-27 devnet run found, and the open items other tickets record
here. End state: every item below and every finding in E2-27's findings list is fixed,
or recorded as accepted with a reason, and the activation runbook no longer carries a
workaround for a fixed finding.

This ticket collects items. Before work starts, review the list and either split it
into tickets or plan one `/develop` run per group of related items.

## Depends on
E2-27.

## Status
Done on 2026-10-29: every open item and every E2-27 finding is fixed or accepted with a reason, and the devnet pass of that day passed every step it ran. The push-time steps under "Resolution" remain, and the human runs them before pushing the repositories.

## Open items

### Deploy and upgrade records are never deleted
Found during E2-26 seam 2. The executor keeps the local pending record and the staged
chunks of every deployment and upgrade after its finalize or upgrade input runs
(zeno-executor `plugins/wasm/wasm.go:122-157`), so the code store grows with every
deployment. It is not exploitable, because every deployment pays a fee, but the store
has no bound.
- Delete a deployment's record and chunks only once the batch holding its finalize or
  upgrade input is finalized. Recovery from a finalized anchor can re-run that input,
  and it needs the chunks.
- Decide what happens to a deployment that is staged and never finalized: whether
  Settlement expires it, and when the executor may delete its chunks.

### E2-27 findings
Every finding in `E2-27-findings.md`, each with its resolution there.

### zeno-client-ts skips the serialization suite's refusal cases
Its tests read only the vectors they name, and its execution-result decoder does not check the input-number rule, the version, or event and message numbering. Fix: the conformance test iterates every vector in `serialization-conformance-v1.json`, accepting each well-formed case and refusing each broken one. The decoder enforces the rules those cases pin.

### zeno-console's fixture copies are unpinned
No test pins a SHA-256 for any file in zeno-console's `test/fixtures/` folders, including the gateway recording, the vault interface and module, and the Settlement RPC recording. Fix: each test that reads a copied fixture asserts its SHA-256, as zeno-client-ts does.

## Key work
Each fix carries a test that fails before it. Any fix that changes a runbook step
updates `phase-1/runbooks/ACTIVATION.md` in the same change.

## Acceptance criteria
- Every open item and every finding is fixed with a test, or accepted with a recorded
  reason.
- The flows a fix touches are re-run on the devnet and pass.

## Resolution

### Open items
- **Deploy and upgrade records are never deleted**: fixed in zeno-executor 7df3dff, on `dev` 4d4df30. After each commit the runtime hands the plugin the lower of the finalized anchor and the oldest retained snapshot (zeno-executor `runtime/loop.go:430-458`). The plugin deletes the record and chunks of every deployment whose finalize or upgrade input is at or below that bound, and keeps the compiled code, which calls need (`plugins/wasm/wasm.go:188`). Every restored position first drops the consumed marks above it (`plugins/wasm/wasm.go:218`, `runtime/loop.go:131-132`), so a finalize that a reorg undid never deletes a live deployment. Before returning to a pruning build after running an older build on the same storage, delete `wasm-code/consumed-deploy/`. Deployments finalized before this build keep their staged chunks until a redeploy or a re-sync.
- **A staged deployment that never finalizes**: accepted. Settlement expires it after 2,880 momentums (go-zenon `vm/constants/settlement.go:14`), it holds at most 256 KiB (`MaxCodeSize`, zeno-executor `plugins/wasm/limits.go:21`), and a new start of the same (deployer, code_hash) clears it.
- **E2-27 findings**: closed as `E2-27-findings.md` records, 22 fixed and 8 accepted.
- **zeno-client-ts skips the serialization suite's refusal cases**: fixed in zeno-client-ts 35610d2. `test/conformance.spec.ts` walks every vector in `serialization-conformance-v1.json`, accepting each well-formed case and refusing each broken one. It skips `CV-COMMIT-1` and `CV-COMMIT-2` by name with their reason (`:39-40`) and fails on any other vector it did not run. The execution-result decoder (`src/decode.ts`) enforces the version, the input number, and event and outbox numbering.
- **zeno-console's fixture copies are unpinned**: fixed in zeno-console 1cb0a8b. `test/fixtures.test.ts:10` pins the SHA-256 of every file in `test/fixtures/`: the gateway and Settlement recordings and the vault interface and module.

### Found in the devnet pass
N-1 (zeno-console, defect): a deploy journal was keyed `domainId:deployer:codeHash` with no chain identity, and a finished journal stayed saved. After the devnet wipe, deploying `vault.wasm` from index 8 resumed the wiped chain's finished journal: the page showed every step as published and sent nothing. The same happened on any network when one address deployed the same code twice. Fixed in zeno-console 1cb0a8b: a stored journal whose last step is published is never resumed, so the deploy starts a new journal (`src/core/composables/useDeploy.ts:111,229`), and the Deploy page can discard a stale pending journal. A resumed unfinished journal still re-sends every step when the chain holds no deploy record for it.

### `stfSpecHash` derivation (F-08)
The testnet `stfSpecHash` is the SHA3-256 hash of zeno-docs `phase-1/SPEC.md`, the STF spec document, at the zeno-docs release commit:

```
git show <release commit>:phase-1/SPEC.md | openssl dgst -sha3-256
```

The release computes the value with this rule and records the commit and the value in the testnet run record. The devnet registers the placeholder `00…01` (zeno-devnet `src/devnet.ts:21`). The placeholder is devnet only and is never registered on testnet.

### Push-time steps
zeno-executor has no remote, and every repository stays local until every E2-28 fix is merged. The human runs these steps, in order, before pushing:
1. zeno-executor (F-22): `.github/workflows/ci.yml` checks out this repository at `zeno/zeno-executor` and go-zenon from `digitalSloth/go-zenon` at `GO_ZENON_REF` (`938fca3`, `feature/settlement`) at `zenon-network/go-zenon`, the layout the `go.mod` `replace` expects. The `replace` stays, because the fork's `go.mod` declares the canonical module path.
2. zeno-executor: tag the pushed `main` commit `zeno-exec-v<version>`. The release workflow publishes `zeno-exec-<os>-<arch>` for `darwin` and `linux` on `amd64` and `arm64`, and `SHA256SUMS`.
3. zeno-contract-sdk-rs: set `version`, `revision` (the commit the tag names) and the four checksums from `SHA256SUMS` in `crates/cargo-zeno/zeno-exec.pin`. Then run `cargo zeno build --check`. It downloads the pinned release, checks its checksum, and fails with Z0205 when the binary's revision differs from the pin. The SDK's CI runs the same download.
4. zeno-contract-sdk-rs (F-14): tag the commit of step 3 `v0.1.0`. The vault artifact is then `vault.wasm` `code_hash` `f7de3e6f84434ce1ade0c45d91c0720c36d2892db3eef80c1e742717753569ff`, `instrumented_code_hash` `29f032c42a1c22e66389f27ac64f0ab90da09e7fa86b1c7a16a09b8e56d0405f`.
5. Push go-zenon before zeno-executor, and zeno-executor with its release tag before zeno-contract-sdk-rs, because the executor's CI checks out go-zenon and the SDK's CI downloads the executor's release.
6. Run runbook G.2 against the pushed zeno-executor commit. A `success` conclusion closes the "in the executor's CI" criterion of E2-27.

### Phase 2
Bundle byte growth (SPEC OD-5(e)) is phase 2 work, with no ticket. A valid bundle has no byte bound, and the receipt verifier no longer reads bundles: it rebuilds each input from the node's copy of the send block.
