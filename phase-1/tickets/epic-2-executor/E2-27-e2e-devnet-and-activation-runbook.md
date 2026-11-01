# Ticket E2-27 — End-to-end run on a local devnet and the activation runbook

**Epic:** 2 — Executor (operational; spans Epic 1) · **Phase:** 1C · **Order:** E2-27

## Goal
Run the whole system end to end on a local devnet: register the WASM domain and the
pillar executor, set the pre-mainnet parameters, run the deposit → call → batch →
finalize → withdraw flow, the fee flow and the failure flows, and drive the user-facing
flows through the console. Write each step down as it is run. End state: every flow
green on the devnet, and a draft activation runbook, `phase-1/runbooks/ACTIVATION.md`,
whose steps were all run on the devnet.

Testnet activation is not part of this ticket. A later ticket takes the runbook to
testnet. Any defect the run finds goes to Ticket E2-28, not into this ticket's scope.
The Phase-1 profile does not activate on mainnet (SPEC §1): mainnet activation waits
for Phase 2's validity proofs.

## SPEC refs
SPEC §29-1C (build sequencing), §25.2 (pre-mainnet parameters), §5.1 (activation),
§4 / §18 / §19 / §21 (the end-to-end path); EXECUTOR.md §20 (phase mapping).

## Depends on
All prior tickets (E2-01–E2-26 and Epic 1, including E1-12), and zeno-console
with the SDKs it builds on. The flows
run against the Ticket E2-24 reference vault, zeno-contract-sdk-rs's
`examples/vault/artifacts/vault.wasm` at a release tag.

## Key work
- **Devnet.** Start the devnet from zeno-devnet with `scripts/devnet.sh up`. It runs
  go-zenon's devnet (`docker/devnet/`, znnd on `feature/settlement`) from a fresh
  genesis that activates `StateRootSpork` and `OffchainExecutionSpork` at height 0,
  the executor and its gateway, and zeno-console against them. The bring-up
  registers the domain and the executor, approves the DA mode, sets the ZNN asset
  limit and the byte prices, and fuses QSR.
- **Bring-up:** `RegisterDomain(WASM, L1_NATIVE, SINGLE, minExecutors = 1)`;
  `RegisterExecutor(pillarName, executorAddress)` for a devnet pillar (§22.2);
  fuse QSR for the executor address and for deployers, since Settlement sends pay
  the plain-send data price (§10.1); deploy the vault artifact (Ticket E2-24) as the
  first contract and buy its storage quota with `BuyStorage` (§13.8).
- **Parameters.** Set the pre-mainnet parameters (§25.2) within Core hard bounds, each
  through its scoped setter (`RegisterDomain` initial delays, `SetDomainConfig`,
  `SetAssetConfig` — which also makes the asset custodyable — `SetApprovedDAModes`,
  `SetPeripheryConfig`): withdrawal-delay duration, `znnPerByteOfBytecode` /
  `znnPerByteOfState` and their bounds, `MaxBatchWithdrawal` and its ceiling,
  runtime-upgrade delay length, and `MaxInputGas`, sized from the batch, since an
  honest executor's worst case is `MaxBatchInputs × MaxInputGas`. Record each value
  with the reason for it.
- **E2E flows** (`architecture.md` §4): payable `CallL2` deposit-and-call with
  partial-claim refund; pure `Deposit`; chunked deploy + finalize; L2→L1 withdrawal
  via outbox → batch `FINALIZED` → `RelayMessage` registration →
  `ReleaseWithdrawals` release; deploy and `BuyStorage` fees → batch accrual →
  `Update` epoch settlement (30% burned, 70% credited) → `CollectReward`; a write
  past quota ⇒ `STORAGE_QUOTA_EXCEEDED`; force-inclusion; pause/unpause, including
  `PAUSE_RELAY` holding fee settlement; executor replacement preserving lineage;
  a domain worker crash that leaves the supervisor running (EXECUTOR.md §1.5);
  crash-recovery against a finalized anchor. Each flow is its own script in
  zeno-devnet that runs against a running devnet, so any flow can be replayed
  alone.
- **Assert through the client path.** Each E2E flow reads its outcome the way a
  client does: the receipt by send hash from the gateway (Ticket E2-22), contract
  state through the `storage` view with a verified proof, and Settlement
  state through `embedded.settlement` (Ticket E1-11). No flow reads executor or
  Settlement storage directly, so the run also proves the client surface.
- **Console run.** Drive the user-facing flows through zeno-console with a real
  nom-webwallet build on the devnet: chunked deploy and finalize of the reference
  vault, `BuyStorage`, a pure `Deposit`, a payable call, a withdrawal, then
  `RelayMessage` and `ReleaseWithdrawals` from the Withdrawals view, and pause and
  unpause from the Admin view. Record the run against the console's manual smoke
  checklist.
- **Findings.** Record every defect, workaround and surprise in a findings list, each
  with the step that hit it. The list is Ticket E2-28's input.
- **Activation runbook** (`phase-1/runbooks/ACTIVATION.md`). One ordered list of the
  steps run on the devnet, each with its command or call, its expected result and how
  to check it. It covers bring-up, parameters and their values, vault deployment, the
  smoke flows, and the rollback or pause step for each stage. Steps that only exist on
  a public network are written out and marked "not run on devnet":
  - the spork release flow: placeholder → governance `CreateSpork` → real id in
    `common/types/spork.go` → coordinated binary release → `ActivateSpork` →
    enforcement, with `StateRootSpork` active before `OffchainExecutionSpork` (SPEC §5.1);
  - confirming the network administrator address carried in the released node binary,
    a build-time value with no on-chain setter;
  - standing up the public gateway (Ticket E2-22);
  - publishing the trust-model disclosure (SPEC §1): Phase 1 is unbonded
    single-executor attestation on testnet, not trustless execution.

## Code references
- `common/types/spork.go` release-flow comments (placeholder → activate)
  [EXISTS].
- `vm/embedded/tests/` (e.g. `state_root_test.go`, `dp_test.go`,
  `z_bridge_test.go`) as integration-test patterns with spork override [EXISTS].

## Acceptance criteria
- Every E2E flow is green on the local devnet: deposit-and-call (with refund), deploy,
  withdrawal release, force-inclusion, pause behaviour, executor replacement, crash
  recovery.
- The console's manual smoke checklist passes on the devnet for every console-run
  flow, with a real nom-webwallet build.
- Conservation holds across the whole run:
  `totalReleased + pendingWithdrawalReserve ≤ totalDeposited` per (domain, asset)
  at every step.
- Fee conservation holds across the run: `totalFeesBurned + totalFeesCredited ≤
  totalFeesCollected` and `totalRewardsCollected ≤ totalFeesCredited`.
- For every E2E input, the gateway serves its receipt at `pending`, `latest` and
  `finalized` in turn, and every proof it returns verifies against the Settlement
  batch root.
- All §31 conformance suites green in the executor's CI and the node's trie/
  embedded tests.
- `phase-1/runbooks/ACTIVATION.md` exists. Each devnet step in it was run as written,
  and each public-network step is marked "not run on devnet".
- The findings list exists, and each finding names the step that hit it.
