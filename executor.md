# Executor

**Status:** Overview — informative.
**Normative source:** [phase-1/EXECUTOR.md](phase-1/EXECUTOR.md) for the Phase 1 binary; [phase-2/SPEC.md](phase-2/SPEC.md) §4 for the STF crate and prover.

The executor is the off-chain program that runs Zeno's domains. It reads each domain's inputs from Settlement, applies the domain's state-transition function (STF), proves the result and submits the batch with its proof. Consensus orders. Executors compute. Settlement anchors.

## 1. Role

Executors are pillars. A pillar registers with `RegisterExecutor` and names a submission address of its own choosing. That address signs `SubmitBatch` and may differ from the pillar's block-producing address, so the executor never holds the producer key.

Every executor runs every active domain. The executor has no sequencing authority: it reads inputs in the order L1 fixed and never reorders, skips or inserts one. Settlement decides acceptance. In Phase 1 one administrator-registered executor runs under the `SINGLE` policy on test networks; in Phase 2 a set of pillar executors shares the work under the proposer rule in section 6.

## 2. Layered runtime

The executor is a Go shell around a Rust worker. The shell owns L1 sync, batch cut points, submission, persistence, the supervisor and the gateway. The Rust STF crate, `zeno-stf`, owns everything the proof covers: codec, SMT, driver, runtime and commitments. Each domain builds it twice, as a native worker behind the supervisor and as a zkVM guest for the prover. The worker owns the domain's state; the shell never reads or writes it. The worker interface has three calls: `apply_batch(prev_root, inputs)` returns the roots, receipts, outbox and witness; `read(root, path)` reads one leaf; `prove(root, paths)` returns a state proof. In Phase 1 the worker is a Go process running the WASM runtime on wazero inside the generic Go pipeline; in Phase 2 the Rust crate replaces that Go STF rather than running beside it.

## 3. Domain plugins

A domain is a runtime linked into the STF core. `zeno-stf/core` holds the codec, SMT, driver, commitments and the `Runtime` trait; `runtime-wasm` holds the interpreter, gas table, structural limits and host functions, and a later runtime such as `runtime-evm` adds one crate. A runtime contributes the STF, input decoding and genesis; ordering, witness capture, roots, batching and Settlement communication belong to the shell and the core. Adding a runtime is a crate, a guest and a verifying-key registration; Settlement code does not change.

Every domain runs in its own worker process, so a worker's crash, hang or memory exhaustion reaches neither the supervisor nor another domain. A worker holds no key material and talks only to the supervisor over a local channel; it hands over an unsigned commitment, and the supervisor signs and sends. In Phase 1 the plugin is a Go package behind the `Domain` interface (`Genesis`, `DecodeInput`, `Apply`) linked into `worker-wasm`; in Phase 2 it is a Rust runtime crate linked into the worker and the guest.

## 4. Input pipeline and ordering

Inputs come only from Settlement account-blocks. A user sends `CallL2`, a deposit, a deploy chunk or `BuyStorage` to Settlement; Settlement accepts it, assigns its index and folds its canonical encoding into the domain's input hash chain. The supervisor follows the `znnd` frontier `confirmationDepth` momenta behind the tip and proves each domain's new ordinal entries from Settlement storage against the momentum state root. The worker finds each named send in the momentum content and applies it in index order.

Indices are assigned over the global stream before domain filtering, so one domain's consecutive inputs can jump in index; the pipeline keeps a cursor per domain. Because every input is an L1 account-block, any node replays any domain from L1 alone. In Phase 1 a domain declares an `inputSource`; whether Phase 2 keeps the field is open (S4).

## 5. Execution loop and batch building

The loop runs per domain. The shell passes the next inputs to `apply_batch`; the worker applies each in turn through a witnessing state accessor, records every path read and written, and returns `preStateRoot`, `postStateRoot`, the receipts, the outbox and the witness. A batch is contiguous over the domain's own subsequence up to `MaxBatchInputs`, and the worker stops sealing once `maxPendingBatches` sit above Settlement's `nextBatchId`.

The STF core is deterministic by rule: no floats, no wall-clock time, no randomness, no hash-map iteration on the consensus path, and no I/O except through the state trait. An internal inconsistency such as a root mismatch halts the worker instead of submitting. The driver also builds a DA bundle and checks its `DAHash` round-trip before committing; in Phase 2 the witness feeds the prover as well.

## 6. Proving and submission

The prover runs `zeno-stf` as a zkVM guest over the inputs and the witness `apply_batch` returned. The guest decodes the input records Settlement encoded, recomputes the input hash chain, re-applies the batch against witness-backed state and commits to the public outputs: `preStateRoot`, `postStateRoot`, the input-chain value, `outboxRoot` and the submitter's executor address. The prover wraps that result in a SNARK.

The supervisor checks that the batch's `batchId` is the domain's `nextBatchId` and that its `preStateRoot` is the last accepted `postStateRoot`, then signs `SubmitBatch` with the executor address and sends the commitment with the proof. Settlement verifies the proof against the domain's verifying key and the input chain; acceptance is finality, and the batch's withdrawals are releasable. The supervisor logs every send before publishing it and follows an unresolved send to its outcome after a restart.

Each epoch has one primary per domain, chosen round-robin over the ACTIVE members. Once the oldest unbatched input has waited `T` momentums, any ACTIVE member may submit, and a timed-out primary leaves the rotation until it submits again. After `T₂ > T`, anyone with a valid proof may submit. The proof binds to the submitter, so a copied proof cannot claim the fees. In Phase 1 there is no prover: the single executor submits each batch with an empty `proofData` and the batch finalizes after the withdrawal delay; in Phase 2 the proof is the acceptance condition.

## 7. State, persistence and recovery

Each worker owns `<storagePath>/<domainId>/` and holds an exclusive `worker.lock` on it. The state store is a cache of derivable state: snapshots tagged with the input position and `stfSpecHash` are its only on-disk copy, and the atomically written `cursor` names the position, root, momentum and `nextBatchId`.

On boot a worker loads the snapshot at its cursor, reads the domain's last accepted batch from Settlement with a verified proof, recomputes its local root at that position and compares. A match replays forward over the canonical stream; a mismatch halts. A reorg within `confirmationDepth` rolls the cursor and store back to the surviving height; a deeper reorg halts. The supervisor restarts a failed worker with bounded backoff and never restarts one that halted on a consistency fault. In Phase 1 the anchor is the last `FINALIZED` batch and replay may wait on the worker's own DA bundle; in Phase 2 accept equals final, so the anchor is the last accepted batch.

## 8. Gateway

The gateway is a separate read-only binary that serves L2 data to wallets, dApps, explorers and other gateways. It holds no key and never connects to the supervisor; it reads an executor's `batches/`, `bundles/` and `receipts/`, or fetches from upstream gateways in `REMOTE` mode. It serves JSON-RPC 2.0 over HTTP and WebSocket in the `zeno` namespace: batches, receipts, inputs, state, proofs, events, outbox, domain views through `zeno_read`, `zeno_simulate` and subscriptions.

Reads take a tag: `pending`, `latest`, `finalized` or a `batchId`, and every response names the batch it was read at and whether that batch passed the checks against Settlement. `zeno_getProof` returns values with proofs against the named batch's `postStateRoot`, the root Settlement holds. `zeno_simulate` runs an input against state at a tag without committing anything, in the domain's worker binary started in simulate mode as a child process; it covers contract reads and gas estimates. In Phase 1 the gateway rebuilds state from bundle `StateDiff`s, checks `SHA3-256(bundle) = DAHash` and serves bundles over HTTP by `DAHash`; in Phase 2 the gateway's state proofs come from the worker's `prove`, and the bundle route stays while the DA commitment does (S4).

## 9. Configuration and operation

The repository ships four programs: `executor` (the supervisor), `worker-wasm`, `gateway` and `zeno-exec`. The supervisor takes one JSON file and rejects any field it does not know.

| Field | Purpose |
|---|---|
| `pillarName`, `executorAddress`, `networkName` | Identity; the pillar's executor-set member must name `executorAddress` |
| `znndEndpoint`, `confirmationDepth` | The L1 node and how far behind its frontier to read |
| `storagePath`, `maxPendingBatches` | The storage root and the sealing limit |
| `submissionKeySource` | A 32-byte ed25519 seed from a `FILE` or an `ENV` variable |
| `workers[]` | Per domain: `domainId`, `stfSpecHash`, `binary`, `memoryLimit`, `cpuLimit` |

A seed whose address is not `executorAddress` fails boot, and no worker receives the key or its location. The gateway has its own file: `listenAddress`, `allowedOrigins`, simulation limits and a `domains` list. In Phase 2 an executor also runs a prover or buys proofs from a prover network; the zkVM, and so what the prover needs, is still being selected.

`zeno-exec` is the local developer tool. It validates and runs contract modules with the real runtime over in-memory state, with no node, executor or chain; `cargo zeno` in the Rust contract SDK drives it.
