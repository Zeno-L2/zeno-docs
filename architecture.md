# Architecture

**Status:** Overview — informative.
**Normative source:** [phase-1/SPEC.md](phase-1/SPEC.md) and [phase-1/EXECUTOR.md](phase-1/EXECUTOR.md) for Phase 1 as built; [phase-2/SPEC.md](phase-2/SPEC.md) for the validity-proof design. [phase-1/ARCHITECTURE.md](phase-1/ARCHITECTURE.md) is the Phase 1 design companion with the go-zenon integration map.

Zeno puts execution off-chain and verification on-chain. This document describes the finished system: what each component does, how a transaction moves through it, and what a user has to trust. Where Phase 1 differs from the end state, the difference is stated inline.

## 1. The two-layer model

Programmability has failed on this architecture by landing on the wrong layer. Execution on account chains destroys atomicity. Execution on the momentum chain forces every node to re-run everything. Zeno keeps execution off both and returns only commitments to L1.

- **L1 — the Zenon network.** Account chains carry feeless asynchronous sends; the momentum chain orders them and anchors state. For Zeno, L1 is a verification surface: it orders inputs, holds custody and records commitments.
- **L2 — off-chain execution.** Pillar-operated executors run each domain's state transition function (STF) off consensus and submit batch commitments to Settlement.

**Consensus orders. Executors compute. Settlement anchors.** Each concern belongs to exactly one layer. L1 owns the order and finality of inputs. The executor owns computation. Settlement owns custody and commitment recording. Because no layer does another's job, moving from attestation to validity proofs adds a verifier; it does not rework the layers.

## 2. Components

| Component | Where it runs | Language | Owns |
|---|---|---|---|
| Settlement | Every go-zenon node, as a spork-gated embedded contract | Go | Input acceptance and indexing, the per-domain input hash chain, batch acceptance and proof verification, custody and conservation, outbox and withdrawals, executor registration and fees, administration |
| `common/trie` | go-zenon | Go | The sparse Merkle tree (SMT) behind the L1 momentum state root, and the path-native core the executor's state tree shares |
| Executor shell | A pillar's infrastructure | Go | L1 sync, batch cut points, submission, persistence, the supervisor that isolates one worker per domain, the gateway |
| STF crate (`zeno-stf`) | Inside the executor as a native worker, and inside the zkVM as the guest | Rust | Codec, the executor state SMT, the driver, the runtimes, commitments, read-API proofs |
| Prover | Beside the executor, or a prover network | Rust | Runs the STF crate as a zkVM guest over a batch's witness and wraps the result in a SNARK |
| Gateway | Beside the executor | Go | Read-only JSON-RPC for a domain's batches, state, receipts, events and outbox, with proofs against the Settlement batch root, and input simulation |
| Clients | Browser and Node | TypeScript, Rust | The Settlement SDK, the client SDK, the contract SDK and the console ([clients/](clients/README.md)) |

Settlement is split into a **domain-agnostic core** and one **domain handler** per runtime. The core is the only code that moves value: custody, conservation, batch chaining, withdrawals, fees, pause. A handler carries a runtime's input vocabulary, such as the WASM handler's chunked deployment. In Phase 2 a handler is thin, because the proof carries execution correctness and a verifying key identifies the runtime.

In Phase 1 the executor's WASM worker is Go on wazero and there is no prover or STF crate; `phase-1/EXECUTOR.md` is its specification.

## 3. Transaction flow

```
User ──account block──▶ Settlement       accept, assign input index, fold into the domain's input hash chain
                            │
                            ▼  (L1 order)
                        Executor shell ──▶ STF worker     apply_batch(prev_root, inputs) → roots, receipts, outbox, witness
                            │                              │
                            │                              ▼
                            │                           Prover       guest re-runs the STF over the witness → SNARK
                            ▼
                        SubmitBatch{commitment, proof} ──▶ Settlement   verify proof against the domain key and input chain
                                                                        batch final; withdrawals releasable
```

1. A user sends a Settlement input as an ordinary L1 account block: `CallL2`, `Deposit`, a deploy chunk, `UpgradeContract` or `BuyStorage`. Settlement accepts it, assigns the domain's next input index, and folds the input's canonical encoding and its momentum context into the domain's input hash chain. Canonical order is the committed momentum order, so the executor derives nothing and can neither reorder nor censor.
2. The executor shell reads accepted inputs from L1 and passes them, in index order, to the domain's STF worker. The worker applies the batch and returns the state roots, receipts, outbox and the witness the guest needs.
3. The prover runs the same STF crate as a zkVM guest over the inputs and witness and produces a SNARK whose public outputs are the pre- and post-state roots, the input-chain value, the outbox root and the submitter.
4. The shell submits the batch with the proof. Settlement verifies the proof against the domain's verifying key and its own input chain. On success the batch is final.
5. A withdrawal is an outbox message. Once its batch is final, anyone proves the message against the batch's outbox root and Settlement releases the asset from custody, inside the per-(domain, asset) conservation bound.

In Phase 1 the commitment carries no proof. Settlement checks contiguity, that `preStateRoot` equals the previous `postStateRoot`, and the asset-flow summary; finality is a withdrawal delay, and release is the two-step `RelayMessage` then `ReleaseWithdrawals`.

## 4. Domains and runtimes

Settlement anchors **domains**. A domain is a runtime, its STF, its input vocabulary and its verifying key, with its own state-root lineage and its own custody. Every counter is keyed by `domainId`, so a batch for one domain can never reach another's custody.

- Every registered executor runs every active domain, each in its own worker process. Running a domain's plugin is a network decision pillars support, not a per-operator choice.
- Adding a runtime means a new runtime module in the STF crate, a new guest program and a verifying-key registration. Settlement's code does not change, and no VM of any kind runs on L1. Settlement inputs are shared by every runtime; only the runtime-specific payload differs.
- Cross-domain interaction, when it exists, is asynchronous through the outbox. There are no synchronous cross-domain calls.
- Phase 1 registers one domain: WASM, under the `ATTESTATION` execution profile. Phase 2 adds the `ZK_VALIDITY` profile and the verifier registry. Candidate second domains are a bridge verifier and an EVM; see [phase-2/SPEC.md](phase-2/SPEC.md).

## 5. State and proofs

One SMT algorithm serves both layers, and the same vector suite pins both. The L1 momentum state root commits to all consensus state, including Settlement's storage, in every momentum header; a client proves "key K has value V at height H" against a signed header. The executor's state tree uses the same path-native core (`RootOfLeaves`, `ProveByPath`, `VerifyProofByPath`) with its own key mapping, so L1 and L2 roots and proofs come from one implementation. The hash is SHA3-256; whether it stays is research item R3.

A client reads L2 state from the gateway with a proof against the batch root Settlement holds, and reads Settlement storage from a node with a proof against the momentum state root. Neither the gateway nor the node has to be trusted. Withdrawals prove against the batch's outbox root, a binary Merkle root, so Settlement needs no SMT verifier for executor state.

## 6. Data availability and replay

Every input is an on-chain account block, so L1 already gives order and availability. A new executor replays any domain from L1 alone, and a user can reproduce any batch. The executor holds bulk state and bulk execution data off-chain; L1 stores only roots and bounded commitment metadata, so L1 cost is constant per batch.

In Phase 1 the gateway also serves an execution data bundle per batch (inputs, witnesses, per-input results) addressed by `DAHash`, best-effort. Whether Phase 2 keeps the on-chain DA commitment is open: the proof covers what the bundle let a reader check, but the bundle machinery is already built and is reserved for future domains and inputs that do not originate on L1.

## 7. Trust model

| | Phase 1 (devnet, then testnet) | Phase 2 (mainnet) |
|---|---|---|
| Who attests a batch | One administrator-registered pillar executor | The proof; no executor, committee or watcher is trusted for correctness |
| Enforced on-chain, independent of the executor | Aggregate custody solvency per (domain, asset), domain isolation, input order, batch chaining, per-batch withdrawal cap | The same, plus the state transition itself |
| Finality | Withdrawal delay | Proof verified at `SubmitBatch` |
| Residual trust | Per-account balances rest on executor honesty | A bug in the guest program or the zkVM |

Phase 1 is testnet-only because no bond backs the executor and no challenge mechanism exists. Both are deliberate: a challenge window and a watcher role would be deleted when proofs land. What stays in Phase 2 is bug containment: the conservation guards, pause, a per-batch cap on ZNN and QSR withdrawals, the runtime-upgrade delay as the users' exit window, and verifying-key rotation under the administrator delay.

## 8. Economics

Executors are pillars and post no bond; a pillar registers a submission address of its choosing and rewards go to the pillar. Calls are free to users. Developers pay ZNN for deployment and for storage, as a prepaid quota bought with `BuyStorage`. Fees settle per epoch: 30% is burned and 70% is credited to the submitting pillar. What pays for proving in Phase 2 is open (research item R5 and question Q-2).

## 9. Roadmap

| Phase | Network | Delivers |
|---|---|---|
| 1 — complete | Devnet today; public testnet in preparation | The Settlement contract, the WASM executor and gateway, the contract SDK, the client SDK and the console; a single attested executor |
| 2 — planning | Mainnet | The Rust STF crate, the zkVM guest and prover, proof verification in Settlement, the proposer rotation and the simplifications that a proof allows |

Everything in Phase 2 is additive to Phase 1's surface: the `proofData` field in the batch commitment is the forward hook, and inputs stay on-chain account blocks.
