# The Settlement SDK

**Repository:** `digitalSloth/znn-typescript-sdk`, branch `feature/settlement`. **Package:** `znn-typescript-sdk`.

The Settlement SDK is the Settlement support in the Zenon TypeScript SDK. It gives a TypeScript client the two go-zenon features Zeno adds to L1: the Merkle state root and the Settlement embedded contract.

**Normative source:** [SPEC](../phase-1/SPEC.md).

## Why it exists

Every L2 action a user takes is an L1 send to Settlement, an embedded contract in go-zenon. A client therefore needs Settlement's ABI, a builder for each method, its read RPC, and a way to check Settlement's stored records against the momentum `StateRoot` instead of trusting a node. The TypeScript SDK already wraps go-zenon's other embedded contracts, so this support follows the same split: ABI definition in `src/embedded/`, read RPC and block-template builders in `src/api/embedded/`, models in `src/model/`.

A helper lives here when go-zenon's Go code implements or enforces it. Anything only the executor, the gateway or a runtime defines lives in [the client SDK](client-sdk.md), which depends on this package and never re-exports it.

## What it does

| Part | Where | Content |
|---|---|---|
| Constants and enumerations | `src/settlement/constants.ts` | `SETTLEMENT_ADDRESS`, the fixed limits Settlement reads (`MaxCallPayload`, `MaxCodeSize`, `MaxInputGas` and the rest) and the enumerations it accepts (`ExecProfile`, `UpgradePolicy`, `DomainStatus` and the rest) |
| Settlement ABI | `src/embedded/settlement.ts` | A byte-identical copy of go-zenon's `jsonSettlement` definition, kept in sync by a script and a test |
| Block-template builders | `src/api/embedded/settlement.ts` | One builder per user method (`deposit`, `callL2`, the three deploy methods, `upgradeContract`, `transferContractOwnership`, `buyStorage`), per permissionless method (`releaseWithdrawals`, `relayMessage`, `update`, `collectReward`) and per administrator method (`registerDomain`, `registerExecutor`, `pause`, `setPeripheryConfig` and the rest). `settlementParamsHash` and `settlementChallengeKey` match a proposal to its pending time challenge |
| Read RPC | `src/api/embedded/settlement.ts` | The `embedded.settlement` namespace with typed models: domains, deploy records, batches, pause state, periphery config, executor set, fees, assets, release queue and pending time challenges |
| Deploy planner | `planDeploy` | Splits a module into chunks and computes the fee, plasma and send count |
| State root | `src/state/` | `ledger.getStateRoot`, `ledger.getProof`, the SMT proof verifier (`verifyProofByPath`, `verifyAbsenceByPath`) and `balanceKey` |
| Storage reader | `SettlementStorage` | Point reads of any Settlement record through `ledger.getProof`, verified by path against the momentum `StateRoot` and decoded with the ABI's variable definitions |
| Withdrawal helpers | `outboxId`, `decodeOutboxWithdrawal`, `outboxProofRoot` | The values `relayMessage` takes, and a check of an inclusion proof against a batch's `outboxRoot` before a client spends a send |
| Asset ids | `assetIdFromZts`, `ztsFromAssetId` | The 32-byte asset id of a ZTS and back |

The contract builders take a `domainId` and carry opaque bytes. Settlement's contract inputs check no `runtimeKind`, so the same builders serve any runtime's domain, and the runtime-specific payload comes from the client SDK's interface encoder. Builders validate what they can check locally, such as sizes and `chunkHash = SHA3-256(chunkData)`, and throw on failure. Fees are arguments, never fetched, so every builder is synchronous.

## What it does not do

- It holds no keys. Every builder returns an unsigned `AccountBlockTemplate`. The caller signs with `zenon.send(block, keyPair)` in Node or `window.zenon.sendAccountBlock(block.toJson())` in a browser.
- It has no `SubmitBatch` builder. Only the executor sends it, and the executor is Go. The reserved methods have no builders either.
- It mirrors go-zenon and nothing else: no gateway client, no contract interfaces, no L2 wire formats, no receipt or event decoders. It decodes only the kind 1 withdrawal that Settlement registers.
- It does not run the WASM runtime or replay batches.
- The storage reader serves point lookups only. Listing deploy records or batches goes through the read RPC.
- It carries no network administrator address. That value differs per network, so callers pass it in configuration.

## Where to read more

In the `znn-typescript-sdk` repository: `docs/state-root.md` for the Merkle state root, state proofs and verification, and `docs/settlement.md` for Settlement contract reads, builders and proof-verified storage.

In these docs: [SPEC](../phase-1/SPEC.md) for Settlement's methods and storage, and [the client SDK](client-sdk.md) for everything built on top of this package.
