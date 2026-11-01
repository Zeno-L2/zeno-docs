# Phase 2 — Validity Proofs on Mainnet

> **Status:** Outline. It records the aim, the decisions taken, the current design direction, the
> research items and the questions still open. A full specification in the shape of `../phase-1/SPEC.md`
> follows once the research and the improvement drafts are decided; tickets follow the specification.
> Nothing here is normative.
> **Improvement drafts:** `improvements/` holds one ticket per change (`IMP-NN`). A section that names
> a ticket summarises it; the ticket holds the detailed design.

## 1. Aim

Run Zeno on mainnet with every batch's state transition **proven valid before Settlement accepts it**.

- No executor, committee or watcher is trusted for correctness.
- Settlement never executes a runtime. It verifies one proof per batch, whatever the runtime.
- Adding a runtime (EVM, a bridge verifier) needs no Settlement code change.

Phase 1 (testnet, single attested executor) is the working system this phase proves.

## 2. The roadmap

| Phase | Network | Execution profile | Finality | Correctness rests on |
|---|---|---|---|---|
| 1 | Testnet only | `ATTESTATION` | Withdrawal delay | The registered executor |
| 2 | Mainnet | `ZK_VALIDITY` | Proof verified at `SubmitBatch` | The proof system and the guest program |

There is no optimistic phase. No dispute game, replay referee, watcher role, challenger payout or
slashing is built. Phase 1 has no challenge window for the same reason: it would be deleted when
proofs land, so Phase 1 stays on test networks and Phase 2 is the mainnet release.

## 3. Decisions taken

| # | Decision |
|---|---|
| DT-1 | Two phases. Phase 1 stays testnet only; Phase 2 is the mainnet target |
| DT-2 | go-zenon and the executor shell stay in Go. Only the provable state transition is Rust |
| DT-3 | One Rust STF crate runs both natively, behind the executor, and inside the zkVM guest |
| DT-4 | All L1 data reaches a domain only as Settlement inputs. A new node replays any domain from L1 alone |
| DT-5 | Settlement verifies proofs and never executes a runtime: no WASM, EVM or other VM on L1 |
| DT-6 | One verifying key per domain, held in the verifier registry. A new runtime is a crate, a guest and a key registration |
| DT-7 | No watcher role and no challenger mechanics |
| DT-8 | Executors are pillars; developers pay deploy and storage fees in ZNN; 30% burned, 70% to the operator. Calls stay free to users (subject to Q-2) |

## 4. Design direction

This section is the current best design. Every part of it is open to change by the research in §5.

### 4.1 Components

| Component | Repo | Language | Owns |
|---|---|---|---|
| Settlement | go-zenon | Go | Input acceptance, input hash chain, SNARK verification, outbox and withdrawals, fees |
| `common/trie` | go-zenon | Go | L1's momentum state root |
| Executor shell | zeno-executor | Go | L1 sync, batch cut points, submission, persistence, supervisor, gateway |
| STF crate | `zeno-stf` (new) | Rust | Codec, executor-state SMT, driver, runtimes, commitments, read-API proofs |
| Prover | zkVM SDK | Rust | Runs the STF crate as a guest and wraps the result into a SNARK |

### 4.2 STF crate layout

```
zeno-stf/
  core/           codec, SMT, driver, commitments, Runtime trait
  runtime-wasm/   interpreter, gas table, structural limits, host functions
  runtime-evm/    (later) revm
  runtime-btc/    (later) Bitcoin SPV ChainVerifier, no interpreter
  worker/         native binary behind the supervisor; owns the state database
  guest/          zkVM entry point; witness-backed state
```

- Each domain builds `core` plus its runtime twice: as a native worker and as a zkVM guest.
- The worker owns executor state. The Go shell never reads or writes it. Worker interface:
  `apply_batch(prev_root, inputs)`, `read(root, path)`, `prove(root, paths)`.
- `apply_batch` returns the witness the guest consumes.
- The gateway's state proofs come from the worker, so the executor side has one tree implementation.
- Rules for `core`: no floats, wall-clock time or randomness; no hash-map iteration on the consensus
  path; no I/O except through the state trait.

### 4.3 Transaction flow

1. A user sends a Settlement input (call, deposit, deploy chunk, `BuyStorage`). Settlement accepts it,
   assigns its index, and folds its canonical encoding into the domain's input hash chain.
2. The executor shell reads accepted inputs from L1 and passes them, in index order, to the worker.
3. The worker applies the batch and returns roots, receipts, the outbox and the witness.
4. The prover runs the guest over the witness and inputs, and produces a SNARK over the public outputs.
5. The shell submits the batch with the proof. Settlement verifies it against the domain's key and the
   input chain; on success the batch is final and its withdrawals are releasable.

### 4.4 Cross-language boundaries

Three encodings are written in Go on one side and read in Rust on the other. Each is pinned by
conformance vectors.

1. **Input record.** Settlement encodes each accepted input and its context (momentum height, time
   and hash) into the input hash chain. The guest decodes the same bytes and recomputes the chain.
2. **Public outputs.** The guest commits to `preStateRoot`, `postStateRoot`, the input-chain value and
   `outboxRoot`, plus the submitter's executor address (S2) and whatever R8 and R9 add. Settlement
   rebuilds the same bytes.
3. **SMT format.** Go `common/trie` and the Rust SMT pass the same vectors (`common/trie/testdata`).

Withdrawals prove messages against `outboxRoot`, a binary Merkle root, so Settlement needs no SMT
verifier for executor state.

### 4.5 Simplifications adopted

| # | Simplification | Removes |
|---|---|---|
| S1 | Accept equals final: `SubmitBatch` verifies the proof, so acceptance is finality | The submitted/finalized split, the withdrawal delay as a safety mechanism, the fee wait for finality, the `RelayMessage` → `ReleaseWithdrawals` two-step |
| S2 | Round-robin primary per epoch and domain, plus an open fallback: once the oldest unbatched input has waited `T` momentums, any ACTIVE member may submit. Timed-out primaries leave the rotation until they submit again. Proofs bind to the submitter. Design: `improvements/IMP-03-round-robin-proposer.md` | Random proposer schedule, missed-turn records, lapse, activation grace |
| S3 | Permissionless fallback after `T₂ > T`: anyone with a valid proof may submit | The escape hatch and forced-withdrawal mechanics |
| S4 | *Under review.* No on-chain DA commitment and no `inputSource`: every input is an L1 account-block, so L1 already gives order and availability. Current leaning: keep both, since the machinery is built and is reserved for future domains and inputs that do not originate on L1. Design: `improvements/IMP-02-drop-da-commitment-and-input-source.md` | `DAHash`, `DAMode`, `damode_version`, the approved-`DAMode` set, `SetApprovedDAModes`, `MaxDAChunkSize`, `inputSource` |

S3 holds only if the prover runs without special hardware, or through an open prover network (R1).

### 4.6 Kept as bug containment

A validity proof cannot catch a bug in the guest program or the zkVM. These stay:

- the conservation guard and the fee conservation guard;
- `PAUSE_RELAY` and the emergency pause;
- one per-batch withdrawal cap (`MaxBatchWithdrawal`) on embedded-owned assets (ZNN, QSR); user-issued
  tokens are uncapped and bounded by per-asset conservation only (`improvements/IMP-01-uncapped-user-tokens.md`);
- the runtime-upgrade delay, as the users' exit window before a runtime or key change;
- verifying-key rotation under the administrator delay.

## 5. Research items

Each item ends in measured data or a decision record in its file under `../research/proving/`.

| # | Question | Method | Data to collect | Feeds | Confidence |
|---|---|---|---|---|---|
| R1 | Which zkVM? | Compare SP1, RISC Zero and others on the criteria below; prototype the guest on the top two | Rust support, precompiles (Keccak, SHA-256, BLS12-381, ed25519), SNARK wrapping, proof size, verifier cost, prover hardware, prover network, licence, maintenance | Everything | High |
| R2 | What does proving WASM cost? | Run wasmi in the guest over the execution conformance vectors | Cycles per gas unit; proving time and cost per vector and per worst-case batch (`MaxBatchInputs × MaxInputGas`) | Gas table, Q-2, Q-4 | Medium |
| R3 | Keep SHA3-256 for the SMT? | Benchmark depth-256 SMT updates and multiproofs in the guest | Cycles per key read and write; per-batch SMT share of total cycles | Hash decision before `StateRootSpork` reaches mainnet | Medium-high |
| R4 | Can an embedded contract verify the SNARK? | Prototype Groth16 or PLONK verification in `vm/embedded` (e.g. gnark) | Verify time, plasma price, proof and public-input size against the 16 KiB account-block cap | `SubmitBatch` | High |
| R5 | Who pays for proving? | Combine R2's cost with expected call volume | Cost per batch and per call; break-even fee levels | Q-2, Q-4 | Medium-low |
| R6 | How does Ethereum learn Zenon finality? | Design a Zenon consensus proof (pillar signatures plus Settlement storage inclusion against the momentum state root) | Signature scheme cost in the zkVM; proof size and EVM verify gas | ETH bridge only; not on the Zeno mainnet path | Medium |
| R7 | Can the guest verify Ethereum? | Evaluate SP1 Helios or RISC Zero Helios, verified recursively in the guest | Recursion cost; header cadence | ETH bridge | High |
| R8 | Can the verifying key replace `stfSpecHash` and Settlement's version checks? | List every client that relies on Settlement rejecting the eight non-`protocol_version` version fields, and check whether the domain's verifying key gives each the same guarantee | Each consumer of version rejection and the guarantee it needs | `stfSpecHash` becomes the key hash; a runtime upgrade becomes a key rotation under the upgrade delay; the fields stay in the encodings for off-chain clients | High |
| R9 | Can the input hash chain replace per-input assignment records? | Move the batch fee total and per-asset deposit credits into the public outputs; size them for a worst-case batch | Public-output size plus R4's proof size against the 16 KiB account-block cap; Settlement storage removed per input | Settlement storage keeps only `(source_account_block_hash, chainValue)` per input; reworks landed E1-04, E1-05 and E1-07 storage and the record sweep | Medium |

### 5.1 Leads

- Succinct reports real-time Ethereum block proving with SP1 Hypercube (R1, R2 baseline).
- SP1 Helios and RISC Zero Helios are audited ZK Ethereum light clients (R7).
- zkWasm (Delphinus Lab) is a dedicated WASM prover, an alternative to an interpreter in a RISC-V
  zkVM (R2).

## 6. Open design questions

### 6.1 Must answer before the build starts

| # | Question | Current leaning |
|---|---|---|
| Q-1 | How does a proof bind to L1 inputs? | Settlement keeps a running input hash per domain, updated at each receive, over each input's canonical encoding and its momentum height, time and hash. This puts the canonical input encoding in go-zenon |
| Q-2 | What revenue pays for proving? | Open until R5. Candidates: a ZNN gas price on calls; a proving surcharge on deploy and storage fees |
| Q-3 | Which SMT hash? | Keep SHA3-256 if R3 allows; otherwise migrate before mainnet |
| Q-4 | Prove every batch, or prove on challenge? | Prove every batch. Fall back to §7 if R2 and R5 say proving is too expensive |
| Q-5 | Who runs provers? | Executors run them, or buy proofs from a prover network. Must stay open to anyone for S3 |

### 6.2 Can wait for the build

| # | Question | Current leaning |
|---|---|---|
| Q-9 | Batch size and proof aggregation: one proof per batch, or aggregate several batches or domains? | Measure first (R2, R4) |
| Q-10 | Values for `T` and `T₂` in S2 and S3 | Set from measured proving latency |
| Q-11 | Gas: keep a static table priced in proving cycles, or meter by cycles? | Static table, re-derived from R2; cycle counts may vary across zkVM versions |
| Q-12 | Testnet-to-mainnet transition | Mainnet starts from a fresh genesis; no Phase 1 state carries over |

## 7. Fallback: prove on challenge

If proving every batch costs too much (R2, R5), batches are accepted optimistically and a validity
proof is required only when someone challenges a batch.

- The executor must answer a challenge with a proof within a window; failing that, the batch is rejected.
- It needs no bisection game and no per-runtime referee.
- It reuses the STF crate, the zkVM and Settlement's verifier unchanged. Only finality and a
  lightweight challenge path change.
- Cost: finality waits for a challenge window, and a simple watcher role returns.

The Phase 2 investment carries over to either mode, so the choice can follow the measurements.

## 8. Risks and unknowns

| Risk | Effect | Mitigation |
|---|---|---|
| WASM proving cost (R2) | Proving too slow or expensive for real use | Gas limits; prove on challenge (§7); an EVM or native runtime first |
| Prover economics (R5) | Operators are not paid for the cost that grows with use | Settle Q-2 before mainnet |
| Guest or zkVM bug | An invalid state is accepted | §4.6 containment; audits; key rotation |
| Two STF implementations during the port | Go executor and Rust crate disagree | Conformance vectors as the byte-equality gate; the Rust crate replaces the Go STF, it does not run beside it long term |
| External zkVM dependency | Licence, maintenance or soundness issues upstream | Choose with R1's maintenance and licence criteria; keep the guest portable across zkVMs |
| Capacity | Rust port, prover integration and bridge work compete for one developer | Keep R6 off the Zeno mainnet path |

## 9. Sequencing

Current thinking, not a plan of record. The improvement drafts come first and are general Phase 1 improvements from feedback and from the second-domain work; the Phase 2 tickets cover the validity-proof work itself.

1. **Runtime-neutral groundwork first.** Conformance and normalisation work that lets a second runtime join as a domain without per-runtime branches: Settlement inputs shared by every runtime, runtime-neutral encodings and identities, one verifying key per domain. This is also the work a second runtime's developers can build against in parallel.
2. **The STF crate.** Port the WASM state transition to `zeno-stf` and gate it on the Phase 1 conformance vectors.
3. **The guest and prover.** Run the crate in the zkVM R1 selects; measure R2 and R3.
4. **Settlement verification.** `SubmitBatch` verifies the proof (R4); S1–S3 land with it, and S4 if it survives review.

ZVM, an EVM rollup on NoM, is the candidate second domain. The aim is for Phase 2 to ship with ZVM support, with its runtime wrapped as a Zeno domain and its applications unchanged. Its integration tests the multi-domain changes and the shared settlement before the prover lands.

## 10. Out of scope

- The ETH bridge and its `ReleaseAdapter` (R6 and R7 are tracked here because they share the zkVM).
- The EVM runtime build.
- Any optimistic or fraud-proof mechanism beyond the §7 fallback.

## 11. Carried over from Phase 1

Items left open at the close of Phase 1. This section owns their resolution; `../phase-1/SPEC.md` §30 and §32 point here.

| Item | What is open | Resolution |
|---|---|---|
| Pre-mainnet parameters | Fee prices and their Core bounds, `MaxBatchWithdrawal` and its ceiling, the network administrator address, the runtime-upgrade delay length. The withdrawal delay leaves with S1 | Set before mainnet, inside the Core hard bounds |
| Deploy-time pricing | `DeployContractFinalize` and `UpgradeContract` run assembly, validation, instrumentation and two engine compiles with no `plasma_limit`; the bytecode fee is the only price | Set `znnPerByteOfBytecode` and its bounds from the pipeline's measured per-byte cost before mainnet |
| OD-4 — returned-blob decode | The returned blob is bounded by `MaxResultBlob`, but neither its decode nor the `StateDiffEntry` key derivation that follows has a gas term, so a blob at the bound costs the same gas as none | Measure at the bound against the initial-memory charge; add a length-proportional term if the resolution rule says so |
| OD-5 — unpriced per-call work | Node-local work with no gas term: compiling a module on a cache miss; shape-dependent instantiation (globals, functions, data and element segments); witness retention for absent-key reads; re-reading and re-verifying stored module bytes; DA bundle bytes, which only gas bounds. None forks consensus; each is a cost-shifting exposure against honest executors | Measure each; add terms where the rule says so. A per-input witness-byte charge is the only closure for bundle bytes |
| Landing-zone balance | An optional alternative to the automatic refund of an unclaimed deposit: a per-(account, asset) balance the depositor reclaims at will, at the cost of a protocol sub-ledger and a `WithdrawDeposit` method | Recorded so the choice is explicit; not needed for safety |
| `MaxBatchMint` bounds and the `AssetRegistry` schema | A bounds predicate symmetric to `MaxBatchWithdrawal`'s, and the full reserved per-(domain, asset) registry entry (custody mode, foreign binding, four counters) | Needed only before a `MINTED` asset is registered; bridges are out of scope (§10) |
| Administrator migration | The administrator gate compares the sender address only, so a governance-contract administrator sends contract-sends, and a rotation must select the authority by the height it takes effect or pre-rotation calls re-execute to a failure | Settle both when the network administrator address moves to a contract or multisig |
| Escape hatch | Forced withdrawal against the last finalized root on domain failure | Closed by S3: anyone with a valid proof may submit after `T₂` |

**Resolution rule for OD-4 and OD-5.** Measure each operation at its consensus bound against the time bought by the least gas an input must pay to trigger it, at a nanoseconds-per-gas calibration from metered straight-line code on the reference hardware. An operation costing more than twice that time gains a term in `metering_version = 2`; one costing less is recorded as an accepted bounded cost with its ratio. Because Q-11 re-derives the gas table from proving cycles (R2), both items fold into that pass.
