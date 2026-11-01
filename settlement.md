# Settlement

**Status:** Overview — informative.
**Normative source:** [phase-1/SPEC.md](phase-1/SPEC.md) for Phase 1 as built; [phase-2/SPEC.md](phase-2/SPEC.md) for the validity-proof design.

Settlement is the embedded contract in go-zenon that anchors Zeno to Zenon L1. It accepts every input, holds every deposit, verifies one validity proof per batch and releases withdrawals; it never runs contract code. This page describes the Phase 2 contract and says where Phase 1 differs today.

## 1. Role

Consensus orders. Executors compute. Settlement anchors. Settlement holds custody, keeps the domain registry and records batch commitments. No WASM, EVM or other VM runs on L1. Its only correctness check is one proof per batch, so a new runtime is a Rust STF crate, a zkVM guest and a verifying-key registration, with no Settlement code change.

Settlement lives in `znnd` as a spork-gated embedded contract (`OffchainExecutionSpork`). Every value it writes folds into `ChangesHash` and then into the momentum hash, so anchoring a commitment is a state write with no momentum-format change. Inputs are ordinary account-blocks; pillars auto-receive them and send the no-argument `Update` crank automatically.

Core is node-binary logic — input acceptance, the input hash chain, proof verification, custody, the outbox, fees and pause — and changes only by a node release behind a spork. Periphery is a few numeric values the network administrator tunes within Core's hard bounds. See [architecture](architecture.md) for the component map.

## 2. Domains and execution profiles

A domain is one runtime plus its STF, input stream and state-root lineage, registered with `RegisterDomain` under a `domainId`. All custody, commitment and conservation state is keyed by domain, so one domain can never reach another's funds.

Each domain carries an execution profile. Under `ZK_VALIDITY` Settlement verifies a validity proof at `SubmitBatch` against the domain's verifying key; under `ATTESTATION` it records the executor's roots without checking them. Keys live in the verifier registry: Core carries a fixed table of built-in verifiers, and `RegisterVerifyingKey` adds a write-once entry under the administrator delay. Rotating a key means registering a new entry and repointing the domain under the runtime-upgrade delay. In Phase 1 the registry is reserved and exactly one WASM `EXECUTION` domain runs under `ATTESTATION`; Phase 2 activates `ZK_VALIDITY` with one key per domain. Whether the key also replaces `stfSpecHash` is open (R8).

## 3. Inputs: `CallL2`, deposits, deployment

Every input is an account-block sent to Settlement with a `{domainId, payload}` envelope. L1 fixes the order — momentum height, then the committed `MomentumContent` order — so the executor has no sequencing authority and force inclusion is native: an input in a confirmed momentum must appear in the domain's next batch, or Settlement rejects the batch.

Settlement assigns each accepted input its index at the receive and folds its canonical encoding, with the momentum height, time and hash, into the domain's input hash chain. A send that fails its method's checks is not an input: it takes no index, and the rollback refunds any attached value. The input vocabulary is shared by every contract runtime; Settlement never branches on the runtime kind.

| Method | Purpose |
|---|---|
| `CallL2` | A call into the domain; payable, so attached ZTS value reaches the target contract as a deposit |
| `Deposit` | Attached value delivered to a contract with no application payload |
| `DeployContractStart`, `DeployContractChunk`, `DeployContractFinalize` | Chunked deployment: a hash-committed record, chunks verified by hash at receive, and a finalize that pays the bytecode fee |
| `UpgradeContract`, `TransferContractOwnership` | Replace the code, or change the owner, of an owner-upgradeable contract |
| `BuyStorage` | Prepaid state quota for a contract, paid in ZNN |

In Phase 1 Settlement also stores a per-input assignment record that the executor reads with a state proof; whether the input hash chain replaces those records in Phase 2 is open (R9).

## 4. `SubmitBatch` and proof verification

An executor submits a batch commitment with `SubmitBatch`: the domain, the batch id, the input range, `preStateRoot` and `postStateRoot`, the input, receipt, event and outbox roots, the asset-flow summary, the submitter and the proof.

Settlement verifies the proof against the domain's verifying key, over public outputs it rebuilds itself: `preStateRoot`, `postStateRoot`, the input-chain value, `outboxRoot` and the submitter's executor address. It also checks that the submitter is entitled, that `preStateRoot` chains from the last accepted `postStateRoot`, that the input range follows the domain's cursor with no gap, and that the asset-flow summary matches recorded deposits. A batch whose proof verifies is final on acceptance; any other is rejected. In Phase 1 the proof field is empty and Settlement checks only contiguity, root chaining and the asset-flow summary, so finality is the withdrawal delay. Whether Phase 2 keeps `DAHash` and `DAMode` is open (S4): L1 already gives every L1 input order and availability, and the commitment is reserved for inputs that do not.

## 5. Assets, custody and conservation

Settlement holds one custody pot per (domain, asset). A `Deposit` or payable `CallL2` raises `totalDeposited` at the receive, unconditionally, because the L1 transfer has already settled. Settlement keeps no per-account ledger: contracts record who owns what in their own state.

Conservation holds per (domain, asset):

```
totalReleased + pendingWithdrawalReserve <= totalDeposited
```

Core re-checks it after every mutating operation and reverts any that would break it. The check does not depend on the executor or the proof system, and it stays in Phase 2 as bug containment. One per-batch cap, `MaxBatchWithdrawal`, bounds what a single batch may release of ZNN and QSR; user-issued tokens are uncapped and bounded by conservation alone. Developer fees sit outside these counters, so fee ZNN never backs a withdrawal and custody never pays a fee. In Phase 1 the cap applies to every custodyable asset, and setting it with `SetAssetConfig` is what makes an asset depositable for the domain.

## 6. Outbox, `RelayMessage` and withdrawals

A contract withdraws by emitting an outbox message that names an L1 recipient, asset and amount. The executor commits a batch's messages under `outboxRoot`, a binary Merkle root, so Settlement needs no verifier for executor state. The unclaimed remainder of any deposit becomes a runtime-emitted refund message, so a contract cannot strand a deposit.

Each message has a unique `outboxId`, and Settlement's processed set makes delivery at-most-once. In Phase 2 acceptance is finality: a withdrawal is proved against its batch's `outboxRoot` and released in one step, under conservation and `PAUSE_RELAY`; the shape of that single-step method is not yet fixed (S1). In Phase 1 `RelayMessage` registers the proven message from a `FINALIZED` batch into the domain's release queue, and the permissionless, rate-limited `ReleaseWithdrawals` crank pays out up to `MaxWithdrawalsPerUpdate` entries per momentum.

## 7. Executor registration and fees

Executors are pillars. Settlement holds one executor set, shared by every domain: each member is a pillar name plus the executor address that pillar chose to sign `SubmitBatch`. Members post no bond, and Core re-checks pillar activity at every submission. In Phase 2 a pillar registers its own executor address with `RegisterExecutor` from its block-producing address; in Phase 1 the administrator registers the single executor under the administrator delay. Proposers rotate round-robin per epoch and domain, with an open fallback after `T` momentums and a permissionless fallback after `T₂` (S2, S3); in Phase 1 the one executor is always entitled.

Calls cost users L1 plasma and nothing else. Developers pay in ZNN for the two things that cost executors lasting work: `DeployContractFinalize` and `UpgradeContract` carry `total_size × znnPerByteOfBytecode`, and `BuyStorage` carries `bytes × znnPerByteOfState` for prepaid storage quota. Fees are collected at the receive, held outside custody and never refunded. The `Update` crank settles one epoch per call: 30% is burned and 70% is credited to the submitting pillar's reward address, paid out by `CollectReward`. A fee conservation guard keeps payouts at or below collections. In Phase 1 an epoch settles only once its batches are final; Phase 2 has no finality wait. What revenue pays for proving is open (Q-2).

## 8. The L1 state root

Version-3 momentums carry a header state root, activated by `StateRootSpork`, over account state including Settlement storage. The tree is the `common/trie` sparse Merkle core the executor also uses for L2 state, with one conformance vector set governing both. A client can therefore verify any Settlement value — a batch entry, a domain record, a custody counter — with a proof against a momentum header instead of trusting a node. The executor reads its input assignment the same way, so `OffchainExecutionSpork` activates only after `StateRootSpork`. Whether the tree keeps SHA3-256 or moves to a ZK-friendly hash is settled before `StateRootSpork` reaches mainnet (R3, Q-3).

## 9. Administration

One global network-administrator address, `types.NetworkAdmin`, carried in the node binary, gates every admin method. Settlement stores no administrator of its own and exposes no method that changes it; moving the authority to a multisig or governance contract repoints the address outside Settlement.

The administrator controls Periphery only. The fee prices (`SetPeripheryConfig`), per-domain delays (`SetDomainConfig`) and `MaxBatchWithdrawal` per asset (`SetAssetConfig`) take effect after a fixed `SoftDelay`, and Core rejects any value outside its hard bounds rather than clamping it. Registering a verifying key, registering the Phase 1 executor or changing a domain's `minExecutors` waits for the longer `AdministratorDelay`. A runtime upgrade or key rotation runs under the runtime-upgrade delay, at least the withdrawal delay, as the users' exit window.

Emergency controls take effect immediately. `Pause` sets `PAUSE_SUBMIT` or `PAUSE_RELAY`, globally or per domain; `PAUSE_RELAY` also halts fee settlement. A domain's `PAUSED` status closes it to new inputs. The administrator can never alter a finalized root, reorder inputs, bypass a delay, or move custody or fees outside these rules. Phase 1 also has `InitiateEscapeHatch`, which Phase 2 drops (S3), and `SetApprovedDAModes`, which stays if the DA commitment does (S4).
