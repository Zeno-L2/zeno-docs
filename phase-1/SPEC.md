# Phase 1 — Off-Chain WASM Execution with Pillar Attestation and On-Chain Conservation

**Document status:** Specification — normative.
**Version:** 1.17.10
**Phase scope:** Phase 1 (a single pillar-operated executor; attestation, not trustless execution; testnet only) is the single **active** point of a general model. The domain schema is generalised for multi-domain, multi-executor, external-input, and **bridge/messaging** operation across two per-domain security ladders (execution + foreign-fact) and two custody modes (§6, §17, §18), with Phase 1 as the degenerate case and every other configuration reserved/gated. How to build a bridge on this model — architecture, worked examples, testing and deployment — is the guide `../research/bridges.md`.
**Classification:** Normative requirements with *Informative* commentary and explicitly marked *Deferred* items.

**Revision history:** maintained in `CHANGELOG.md`.

---

## Conventions

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**, **SHOULD**, **SHOULD NOT**, **MAY**, and **OPTIONAL** are to be interpreted as in RFC 2119 and RFC 8174.

Three classes of content appear:

- **Normative** text defines binding requirements.
- **Informative** text (a leading `>` blockquote, or an explicit *Informative* label) explains rationale and carries no binding force.
- **Deferred** items are explicitly out of Phase 1 scope, recorded so Phase 1 does not foreclose them.

The architectural invariant of this protocol is:

> **Consensus orders. Executors compute. Settlement anchors.**

Every requirement derives from that invariant. This document is self-contained and normative: it is written to be handed directly to implementers. Design rationale, component architecture, end-to-end flows, and the go-zenon integration map live in the companion `ARCHITECTURE.md` (informative); the normative requirements for sharing the SMT core with L2 — including the path-native API the L2 executor MUST use and the key-hashing functions it MUST NOT call — are in §13.1. Companion machine-readable artifacts (`smt-v1-test-vectors.json`, `proof-format.md`, `wasm-gas-table-v1.json`/`.md`, `execution-conformance-v1.json`, `execution-conformance-v1-README.md`, `serialization-conformance-v1.json`) are normative where they specify exact bytes, ordinals, or costs.

---

## 1. Trust model

> **Phase 1 activates a single point on a general trust surface: attestation by one administrator-registered pillar, with no external inputs and no bond. It is not trustless execution, and it runs on testnet only.** This is normative context for the entire specification and **MUST** be communicated to users and integrators without qualification or obfuscation.

A domain's trust is defined by two **independent** profile dimensions, each a ladder that strengthens additively (§6.3, §6.4). Phase 1 occupies the base rung of each:

- **Execution profile** — how a batch's state transition is secured: `ATTESTATION` (the executor attests, with no on-chain correctness check; Phase 1) → `ZK_VALIDITY` (Settlement verifies a validity proof; required on mainnet). This is the two-phase roadmap (§2.2) lifted to a per-domain field.
- **Foreign-fact profile** — how the truth of an external-chain event consumed as input is established: `NONE` (no external input — Phase 1) → `COMMITTEE` (attester quorum) → `LIGHT_CLIENT` (verified in the STF) → `ZK_CONSENSUS` (succinctly proven). Only `BRIDGE`/`MESSAGING` domains use a non-`NONE` value (§6).

**A domain is exactly as trustless as the weaker of its two profiles, and no more.** There is no single global trust assumption; each domain carries its own pair and its own trust model, which **MUST** be surfaced to users.

Three permanent safety rails hold in every profile:

- **Rail 1 — STF-verifiable is not Settlement-verified.** Under `ATTESTATION`, Settlement records `preStateRoot`/`postStateRoot` but **MUST NOT** verify them; correctness rests on the registered executor. Execution is on-chain-verified only under `ZK_VALIDITY`.
- **Rail 2 — a foreign fact commits what was observed, not what is true.** A commitment binds the inputs the executor consumed; whether those reflect a real foreign event depends on the foreign-fact profile.
- **Rail 3 — commitment is not availability.** `DAHash` commits a bundle; it does not guarantee retrievability (§20). Under `ATTESTATION`, an unavailable bundle reduces the affected batch to executor honesty.

The chain cannot pay out more than was deposited/backed in aggregate per (domain, asset) — enforced on-chain, independent of executor honesty for the `ESCROW` custody mode; for `MINTED` the guarantee is profile-dependent (§18.3). Per-account balance correctness relies on the registered executor's honesty until the execution profile removes it.

| Guaranteed on-chain (executor-independent) | Established by the profile, not free |
|---|---|
| Aggregate ESCROW solvency: `totalReleased + pendingWithdrawalReserve ≤ totalDeposited` | Per-account / per-message correctness (executor honesty until `ZK_VALIDITY`) |
| Domain isolation: an executor can never reach another domain's custody | On-chain verification of `preStateRoot`/`postStateRoot` (only `ZK_VALIDITY`) |
| Canonical input order fixed by L1 (`L1_NATIVE`/`L1_RELAYED`); no reorder/skip/insert | Truth of a foreign fact (the foreign-fact profile) |
| Batch chaining, contiguous ranges, finality window, per-batch caps, public commitments | Aggregate MINTED solvency (profile-dependent, §18.3) and DA availability |

**Normative:**

- The Settlement contract **MUST NOT** execute any STF, and **MUST NOT** verify execution correctness except where the execution profile requires it (a validity proof under `ZK_VALIDITY`).
- The Settlement contract **MUST** record batch commitments and **MUST** enforce custody, finality, batch caps, and the per-(domain, asset) conservation invariant (§18).
- Aggregate over-withdrawal (`ESCROW`) and over-mint / over-release (`MINTED`, at its profile floor) **MUST** be prevented on-chain by Settlement conservation accounting (§18), within the guarantees each custody mode's profile provides.
- **Phase 1 active point (normative):** `execProfile = ATTESTATION`, `foreignProfile = NONE`, `custodyMode = ESCROW`. Every other value is reserved and Core **MUST** reject it while the Phase-1 profile is in force (§6.1, §6.5). At this point the safety mechanisms are exactly the withdrawal delay, the per-batch cap, the conservation invariant, and public commitment visibility. No bond backs the executor (§22).
- **Mainnet gate (normative).** The Phase-1 profile point **MUST NOT** be activated on mainnet. Mainnet activation **MUST** wait for `ZK_VALIDITY` (Phase 2, `../phase-2/`), under which Settlement verifies a validity proof for every batch, because with no bond the Phase-1 point has no economic backstop against a dishonest executor.

> The honest framing: at the Phase-1 point the chain cannot pay out more than was deposited in aggregate (enforced on-chain), but within the withdrawal delay and bounded by the per-batch cap it can misattribute balances among accounts if the executor is dishonest. No bond covers that loss, which is why the Phase-1 point is testnet-only. That residual trust is removed by `ZK_VALIDITY` (validity proofs); the foreign-fact profile removes the analogous trust on external inputs. The generalized bridge model layered on this base — profiles, custody modes, and the per-chain seams — is normative here (§6, §17, §18); its worked examples, comparative mappings, and testing/deployment strategy live in the guide `../research/bridges.md`.

---

## 2. Goal and scope

Run smart contracts off the consensus layer and settle their results on Zenon L1, without any momentum-format change and without L1 ever executing contract code. Phase 1 is the smallest deterministic WASM execution layer that can later be hardened with validity proofs (Phase 2) additively: no rewrite and no state migration, unless the §13.2 hash decision replaces the state-tree hash.

### 2.1 Non-goals

- Phase 1 does **NOT** execute WASM on L1.
- Phase 1 does **NOT** verify execution correctness on-chain (no validity proofs).
- Phase 1 does **NOT** provide trustless execution (§1).
- Phase 1 does **NOT** define synchronous cross-contract calls (§17).
- Phase 1 does **NOT** define a multi-executor set, native L2 token issuance, or zkVM circuit design.
- Phase 1 does **NOT** define L1 consensus mechanics; the Zenon momentum chain is consumed as given.

### 2.2 Phase roadmap

| Phase | Executor model | Correctness model | Finality model |
|---|---|---|---|
| Phase 1 (testnet only) | Single administrator-registered pillar executor | Attestation (no on-chain correctness verification, no bond) | Withdrawal delay |
| Phase 2 (mainnet) | Specified in `../phase-2/` | Validity proofs (`ZK_VALIDITY`), verified by Settlement | Proof-verified (`PROOF_VERIFIED`) |

**Normative:** Phase 1 design decisions **MUST NOT** create structural barriers to Phase 2. If the §13.2 hash decision replaces the state-tree hash, the change **MUST** be implemented as a one-time versioned migration.

---

## 3. Architecture

Settlement is a **runtime-agnostic anchor**: it holds escrow, maintains a **domain registry**, and records batch commitments — it never interprets contract code. The unit it manages is an **execution domain** (a runtime + STF spec + state-root lineage, served by the network's executor set). Phase 1 ships the Settlement core plus **exactly one domain handler — WASM** — and registers **one pillar-operated executor running the WASM runtime** at launch. The schema is domain-shaped so additional runtimes need no migration; opening domain creation further is an administrator decision in Phase 1 (later, a governance/multisig decision — §6, §23).

```
   L1 (Zenon)                                   Off-chain                    Off-chain (best-effort)
   ┌────────────────────────────────┐        ┌──────────────────┐         ┌────────────────────┐
   │ Settlement embedded contract   │ inputs │  WASM Executor   │ bundles │ Gateway: HTTP      │
   │ (spork-gated)                  │◀──────▶│ (single pillar,  │────────▶│ bundle serving by  │
   │  domain-agnostic CORE:         │ Submit │  per WASM domain)│ DAHash  │ DAHash             │
   │   registry · escrow/custody    │ Batch  │  runtime + SMT   │         └────────────────────┘
   │   per-domain commitments       │◀───────│  batch builder   │
   │  domain HANDLERS: [ WASM ]     │        └──────────────────┘
   └────────────────────────────────┘  commitments anchored via ChangesHash → momentum (no header change)
```

**Normative component roles:**

- **L1 consensus (Zenon momentum chain):** **MUST** provide canonical ordering of all inputs (§4) and anchor Settlement storage including batch commitment roots. **MUST NOT** execute WASM. It is the sole source of truth for canonical input order and for input finality.
- **Domain-agnostic core (one implementation, serves every domain):** the domain registry + lifecycle, per-(domain, asset) escrow/custody + conservation, executor registration (pillar-bound, shared by every domain), developer fees and their per-epoch settlement, batch-commitment chaining/contiguity keyed by `domainId`, withdrawal delay/finality, outbox replay protection, and pause.
- **Domain handler (one per runtime; Phase 1 = WASM only):** the runtime's input vocabulary plus any Settlement-side bookkeeping the runtime needs — for WASM, the chunked-deployment state machine and `code_hash` records. Compiled into `znnd`, gated by the spork. A future runtime adds another handler the same way (§6).
- **Executor:** operated by a pillar (§22), it serves every active domain. For each domain it **MUST** consume the canonical input stream filtered to that `domainId` in canonical order, compute deterministic WASM state transitions off-chain, produce execution artifacts, and submit batch commitments. It **MUST NOT** reorder, skip, censor, or privately insert inputs; it has no sequencing authority.
- **Browser clients:** **SHOULD** download execution data bundles (§20) and reproduce results independently. This is advisory: Settlement has no on-chain channel to act on a detected divergence.
- **Gateway:** serves bundles addressed by `DAHash` over HTTP, best-effort (§20, `EXECUTOR.md` §21.5); it is not consensus-critical.

**On-chain vs off-chain (normative):** L1 **MUST** store only roots and bounded commitment metadata; it **MUST NOT** store bulk execution-layer state or bulk execution data. Bulk state (the SMT) and bulk data (input data, witnesses, per-input results) are held by the executor and published off-chain.

> This separation is the load-bearing decision. It keeps L1 cost bounded and constant per batch and is what lets the WASM layer be added with no momentum-structure change.

---

## 4. Canonical input stream

### 4.1 Origination

**Normative:**

- All inputs **MUST** originate as account-blocks sent to the Settlement embedded contract on L1, carrying a `{domainId, payload}` envelope (data + plasma only; pillars order them, never execute them). Data availability is inherited from L1.
- **Envelope encoding.** Each input method is a typed Settlement ABI method whose first argument is `domainId : uint32`; its remaining arguments are the payload. For example, `Deposit(domainId uint32, targetContract hash, plasmaLimit uint64)`. The ABI encoding is the transport only: the executor decodes it and re-encodes the input in its canonical form (§27.5), which is what `inputRoot`, receipts and the DA bundle commit to.
- The valid input types are exactly the eight kinds of §27.5: `CallL2`, `Deposit`, `DeployContractStart`, `DeployContractChunk`, `DeployContractFinalize`, `UpgradeContract`, `TransferContractOwnership`, `BuyStorage`. (All are domain-scoped by `domainId`; core methods are in §5.3.)
- **Paused domains.** Settlement **MUST** reject at its receive every input method addressed to a domain whose `status` is `PAUSED` (§6). A rejected call is not an input (§4.6), and the standard embedded rollback returns any attached value.
- An input becomes part of the canonical stream when, and only when, the account-block carrying it is included in a confirmed momentum (§4.3).
- The executor **MUST NOT** process inputs from any source other than the canonical stream and **MUST NOT** treat any privately observed transaction as an input until it appears in a confirmed momentum.

### 4.2 Force inclusion is native

**Normative:**

- Force inclusion requires no additional mechanism. A user who submits any valid input to Settlement and has the account-block included in a confirmed momentum has force-included it.
- The executor **MUST** process that input in canonical order. A batch omitting a canonical input fails contiguity verification at Settlement (§4.6) and **MUST** be rejected.

> The Settlement contract *is* the force-inclusion mechanism. The only residual censorship surface is L1 censorship — preventing an account-block from entering a momentum at all — an L1 consensus concern outside this scope.

### 4.3 Confirmed-momentum requirement

**Normative:**

- The executor **MUST** consume inputs only from momenta at or below the node's confirmation frontier, so ordering cannot be altered by an L1 reorganization.
- The executor **MUST NOT** include in a batch any input whose containing momentum is not yet confirmed.

### 4.4 Canonical ordering

**Normative:** the canonical order of inputs **MUST** be:

1. **Momentum height ascending.**
2. **Within a momentum, the existing `MomentumContent` order committed by go-zenon** — the ascending lexicographic order of `AccountHeader.Bytes()`, where

   ```
   AccountHeader.Bytes() = Address (20B) || Height (8B big-endian) || Hash (32B)
   ```

- The executor **MUST** reproduce this exact order and **MUST NOT** apply any alternative (by account-block hash alone, fee, plasma, input type, or originator).

> This reflects go-zenon reality and is L1-verifiable with no new derivation: `common/types/account_header.go` defines `Bytes()` as `Address ‖ Height ‖ Hash`, and `chain/nom/momentum_content.go` sorts content by `bytes.Compare(a.Bytes(), b.Bytes())`. That ordering is already committed inside the momentum hash via `MomentumContent.Hash()`.

### 4.5 Global input index

**Normative:**

- Each input **MUST** be assigned a `globalInputIndex`: a `uint64`, zero-based, strictly monotonically increasing across the entire stream, never reset, assigned in canonical order.
- Assignment **MUST** be deterministic: any party reconstructing the canonical stream **MUST** compute identical values.
- `globalInputIndex` is bounded by `2^64 − 1`. Reaching that value is outside Phase 1 operational scope; an implementation **MUST** halt input admission (it **MUST NOT** wrap to `0`) if the next index would overflow, and the condition **MUST** be treated as a protocol-version migration trigger, never a reset.
- For generality across input sources (§6.1), a domain's ordered inputs form its **input sequence**, whose ordinal is the **input sequence number**; batch bounds and the per-domain cursor are named over it (`firstInputSeq`/`lastInputSeq`, §4.6, §19). Every input source is L1-anchored (§6.1), so the input sequence number **MUST** equal the `globalInputIndex` defined above.

> `globalInputIndex` is assigned over the global stream; per-domain filtering does not renumber. A domain's batch covers a contiguous slice of the global index (§4.6); indices belonging to other domains simply do not appear in this domain's batches, and contiguity is checked per domain against that domain's own cursor.

### 4.6 Batch contiguity and momentum splitting

**Normative:**

- A batch **MUST** be a contiguous slice of its domain's canonical input subsequence: the inputs for `domainId` with `globalInputIndex` values strictly increasing, beginning at the successor of that domain's previous batch's last index, with at most `MaxBatchInputs` (§25) inputs.
- A batch **MUST** contain at least one input.
- Batch boundaries **MAY** fall between two inputs originating in the same momentum. Momentum splitting across batches is permitted; there is no atomic-per-momentum requirement.
- Settlement **MUST** verify, per domain at acceptance, that `batch.firstInputSeq` is the successor of the domain's stored input-sequence cursor over that domain's inputs, with no gap, overlap, replay, non-canonical ordering, or count exceeding `MaxBatchInputs`. (For L1-sourced domains `firstInputSeq == firstGlobalInputIndex`; §4.5.)
- **Settlement assigns the input sequence on-chain, on acceptance.** An account-block sent to Settlement **MUST** be assigned the next global input index at its embedded receive, and recorded against its domain together with per-(domain, asset) cumulative deposit totals (§18.4) and any developer fee the send paid (§10.7), if and only if the addressed method **accepts** it: the envelope decodes to a valid `{domainId, payload}` for a registered domain (§4.1) **and** every check that method performs at receive passes. A send the method rejects — an undecodable envelope, an unregistered domain, or a failed semantic check such as a duplicate, absent, expired or already-consumed deployment record — is **not** an input: it consumes no index, produces no receipt, and contributes no `inputRoot` leaf. The embedded receive fails and any attached value is refunded by the standard embedded rollback. The executor does not re-evaluate this predicate: it reads Settlement's assignment from L1 state (**Executor reads the assignment**, below), so the two numberings coincide by construction.
- **Receive height (normative).** Every height-dependent check or record Settlement makes at an embedded receive — deployment-record start height and expiry (§11), the fee price in force (§10.7), a batch's `submittedAtHeight` (§19), a `MaxBatchWithdrawal` value in force (§27.9) — **MUST** use the **receive height**: the height of the frontier momentum in the receive's VM context, which is the height of the receive block's `MomentumAcknowledged` (`context.GetFrontierMomentum()`, `vm/supervisor.go`). It is not the height of the momentum that later contains the receive block, which the receive cannot know, nor of the momentum containing the send. The executor evaluates none of these checks itself; it reads their outcome from Settlement's assignment.
- Because acceptance governs assignment, every input the executor is asked to execute has already passed its method's receive-time checks. A runtime that observes an input whose on-chain precondition does not hold — a deploy finalize naming a record or chunk it cannot read — differs from the network rather than from the protocol, and **MUST** halt rather than commit any receipt status for it. An `input_kind` outside the eight §27.5 kinds is not such an input: its receipt is `UNSUPPORTED_OPERATION` (§15, §27.5). A failure of the §11.3 pipeline (assembly, hash or length mismatch, validation, instrumentation) is not such an input either: its receipt is `VALIDATION_FAILED` (§11.3, §12). Settlement **MUST** verify batch bounds against its own assignment: `firstInputSeq` **MUST** be the domain's earliest assigned-but-unconsumed index; `lastInputSeq` **MUST** be an index assigned to that domain and **MUST NOT** exceed the highest index Settlement has assigned; and the count of the domain's assigned indices in `[firstInputSeq, lastInputSeq]` **MUST NOT** exceed `MaxBatchInputs`.
- **Per-domain ordinal (normative).** Alongside its global index, every accepted input **MUST** receive the next per-domain ordinal (`0, 1, 2, …`), and its assigned-index record — carrying its `globalInputIndex`, deposit asset and amount, and fee (§10.7) — **MUST** be reachable by `(domainId, ordinal)` with one key lookup. The entry stored under `(domainId, ordinal)` **MUST** also carry the input's `source_account_block_hash` (§27.5), so the assignment names the send it accepted. Keying the record by the ordinal, or keeping it under its global index with a separate ordinal index, are both conformant. A batch's inputs are then the ordinals `[consumedInputCount, consumedInputCount + n)` for some `n` in `[1, MaxBatchInputs]`: `firstInputSeq` **MUST** equal the `globalInputIndex` of ordinal `consumedInputCount`, and `lastInputSeq` that of ordinal `consumedInputCount + n − 1`. Settlement reads at most `MaxBatchInputs` records per batch, by key, without scanning the global index range, which other domains' inputs can make arbitrarily long.
- **Executor reads the assignment (normative).** For each confirmed momentum `C`, the executor **MUST** read each of its domains' received-input count from Settlement state at `C`, and for every ordinal the count adds, the `(domainId, ordinal)` entry, both proven against `C`'s state root. Those entries are the domain's inputs, in `globalInputIndex` order. A count changes only at a Settlement receive, so a momentum whose content holds no Settlement account-block adds no input and needs no read. The executor keeps, per domain, the next ordinal it expects; the ordinals a momentum adds **MUST** start there, and anything else is a consistency fault. A momentum that holds a Settlement account-block but carries no state root (below version 3) cannot be read and is a consistency fault. The executor takes each input's data from the send the entry names, found by hash in hash-checked momentum content (the momentum holding the send may precede `C` by any distance), and re-encodes the canonical input (§27.5). A named send it cannot find or decode, or one whose envelope names another domain, is a consistency fault and **MUST** halt the executor. Reading at `C` needs the node to retain state for `C`: an executor that lags its node, or re-syncs from genesis, needs a node that retains state that far back (an archive node for genesis).
- **Assigned-index record lifecycle (normative).** An assigned-index record is kept until its batch is `FINALIZED`. After each accepted `SubmitBatch`, Core **MUST** examine at most `MaxRecordSweepBatchesPerSubmit = 2` of the domain's batches, starting at `recordsSweptThrough`, stopping at the first that is not `FINALIZED` at the receive height. For each finalized batch it **MUST** delete the assigned-index records (and any ordinal-index entries) of the batch's ordinals, which start at `sweptInputCount`, then advance `recordsSweptThrough` by one and `sweptInputCount` by the batch's input count. Every node deletes identically, since Settlement storage feeds `ChangesHash` (§5.1). The sweep never changes an acceptance outcome: a consumed ordinal is never read again once its batch is final. Storage therefore holds records only for inputs not yet in a finalized batch.

> *Informative.* Assigning on acceptance rather than on decodability is what keeps a deploy pipeline safe to serve. Under the alternative, any account could submit a call naming another deployer's `code_hash` and turn that deployer's finalize into a network-wide halt, because every executor would be obliged to produce a receipt for an input whose record it was never party to. Acceptance removes the class: a call Settlement rejects never reaches the executor at all.

> *Informative.* Embedded receives replay the committed send order — the per-contract sequencer is FIFO over `MomentumContent` order (`chain/account/sequencer.go`, consumed by `pillar/worker_contract_generator.go`) — so the receive-time assignment reproduces the §4.5 derivation exactly. This also closes the receive-lag race: a batch covering inputs whose receives have not yet been processed refers to indices Settlement has not assigned and is rejected; the executor resubmits once the receives land.

> **Contiguity is per-domain over the domain's subsequence, not over the raw global index.** When two domains (A and B) are both active, the global stream might carry inputs at indices: 5(A), 6(B), 7(A), 8(B), 9(A). Domain A's subsequence is `{5, 7, 9}`. A valid first batch for domain A covering indices 5 and 7 has `firstInputSeq = 5`, `lastInputSeq = 7` (equal to `firstGlobalInputIndex`/`lastGlobalInputIndex` for this L1-sourced domain) — the raw jump from 5 to 7 is correct and expected. Domain A's stored cursor advances to 7; the next batch must start at the domain A input whose `globalInputIndex` immediately follows 7 in domain A's subsequence (i.e., 9). Implementations **MUST NOT** check raw `globalInputIndex` contiguity; they **MUST** track each domain's last-consumed index independently and verify only that the incoming batch begins at the next index in that domain's own sequence.

> Momentum boundaries are irrelevant to contiguity. Allowing momentum splitting prevents a large momentum from forcing an oversized batch.

### 4.7 No executor sequencing authority

**Normative:** the executor **MUST NOT** reorder, skip, privately insert, or otherwise sequence inputs outside the canonical stream. There is no skip mechanism — omission produces a non-contiguous batch rejected on-chain.

---

## 5. Settlement embedded contract

### 5.1 Activation

**Normative:**

- Settlement **MUST** be a Zenon embedded contract gated behind a dedicated **`OffchainExecutionSpork`**.
- `OffchainExecutionSpork` **MUST NOT** be activated before `StateRootSpork`: the executor reads Settlement's input assignment through state proofs (§4.6), which exist only for version-3 momenta.
- Before activation, all Settlement methods **MUST** reject inputs; after activation they become live per this spec. Before activation the Settlement address resolves to no contract, so calls fail with `constants.ErrContractDoesntExist` — distinguishable from a method-level rejection.

> This uses the established extension pattern. A new embedded contract is registered by adding a base entry plus a spork-gated diff layer, exactly as the HTLC contract was added. In `vm/embedded/embedded.go`, `GetEmbeddedMethod` builds the active contract map by applying `applyXDiffs` layers gated by `context.IsXSporkEnforced()`. A new `applySettlementDiffs(...)` layer, gated by a new `IsOffchainExecutionSporkEnforced()` in `vm/vm_context/spork.go`, mirrors `applyHtlcDiffs` / `IsHtlcSporkEnforced`:
>
> ```go
> // vm/vm_context/spork.go — new accessor, mirroring IsHtlcSporkEnforced
> func (ctx *accountVmContext) IsOffchainExecutionSporkEnforced() bool {
>     active, err := ctx.cacheStore.IsSporkActive(types.OffchainExecutionSpork)
>     common.DealWithErr(err)
>     return active
> }
>
> // vm/embedded/embedded.go — new gate inside GetEmbeddedMethod
> if context.IsOffchainExecutionSporkEnforced() {
>     applySettlementDiffs(contractsMap)
> }
> ```
>
> The contract address is `z1qxemdeddedxsettlementxxxxxxxxxxqtmqrx9` (`types.SettlementContract`), added to `EmbeddedContracts` in `common/types/address.go` (via `parseEmbedded`, alongside `HtlcContract`). The spork id follows the placeholder-hash release flow in `common/types/spork.go` (precedents: `HtlcSpork`, `DynamicPlasmaSpork`, `StateRootSpork`): the symbol `types.OffchainExecutionSpork` carries the placeholder hash `…0004` and is registered in `ImplementedSporksMap`.

This is a **contract-activation spork** (like `HtlcSpork`), **not** a momentum-version bump (like `StateRootSpork`, which advances momentums to version 3 to carry a header `StateRoot`). Commitments anchor through the existing `ChangesHash` path, so the momentum header is unchanged: any value an embedded contract writes via `context.Storage()` is folded into `ChangesHash` (`block.ChangesHash = db.PatchHash(changes)` in `vm/supervisor.go`), committed into the momentum hash (`chain/nom/momentum.go`), signed by the producing pillar, and re-verified by every node. **No momentum or account-block format change; no hard fork beyond the spork.**

> *Informative.* The L2 root-sharing foundation is gated by its own independent spork: `chain/nom/momentum.go` defines `StateRootMomentumVersion = 3` with a `StateRoot` header field folded into `ComputeHash` from version 3; `verifier/momentum.go` enforces it (`verifyStateRoot`, `ErrMStateRootInvalid`, `ErrMStateRootMustBeZero`); `common/types/spork.go` carries `StateRootSpork` (placeholder hash, safe-by-default until governance activation); and `chain/state_tree.go` drives a `trie.NodeTree`. The Settlement spork (`OffchainExecutionSpork`) is independent of `StateRootSpork` and adds **no** momentum-version bump of its own; it anchors via `ChangesHash` as described above.

### 5.2 Core and Periphery

**Normative:**

- **SettlementCore** is node-binary embedded logic, changeable only by a coordinated node release gated behind a spork or hard fork. The administrator (and any Periphery method) **MUST NOT** be able to alter Core logic through any on-chain method or storage write. There is **no on-chain upgrade proxy** and no bytecode replacement; "immutable core" means binary logic only a sanctioned release can change.
- **SettlementPeriphery** is administrator-controlled configuration in contract storage, modified through dedicated admin methods (the network administrator, §23) and bounded by Core's immutable hard bounds (§23).
- Periphery **MUST NOT** write Core state directly; Core reads Periphery values as inputs to its own enforcement and **MUST** reject any value outside its hard bounds (`ErrForbiddenParam`) rather than silently clamping it.

**Core responsibilities (node-binary):** batch acceptance (canonical-ordering, contiguity, `AssetFlowSummary`, commitment-size checks); the domain registry + lifecycle; per-(domain, asset) deposit custody and conservation (§18); withdrawal registration, delay, release; the chunked-deployment state machine (§11); force-inclusion contiguity (§4.6); emergency pause (§23); hard bounds on all Periphery values.

> Maps onto existing patterns: time-windowed administration mirrors the `TimeChallenge` machinery used across the bridge and liquidity contracts (`vm/embedded/implementation/common.go`, `liquidity.go`); executor identity reads the pillar registry through `context.MomentumStore().GetActivePillars()` (precedent: `vm/embedded/implementation/accelerator.go`); per-epoch reward settlement mirrors `UpdateEmbeddedPillarMethod` / `checkAndPerformUpdateEpoch` and the `RewardDeposit` / `CollectReward` bookkeeping (`vm/embedded/implementation/common.go`).

**Periphery responsibilities (administrator-controlled storage), at three scopes:**

- **Network-wide:** the developer-fee prices `znnPerByteOfBytecode` and `znnPerByteOfState` (within hard bounds, §10.7), and the reserved verifier registry (§5.5).
- **Per domain:** the withdrawal delay (within hard bounds), the runtime-upgrade delay (within the §6 floor, evaluated against that domain's withdrawal delay), and the approved `DAMode` set (§20).
- **Per (domain, asset):** `MaxBatchWithdrawal` (within hard bounds). Configuring it is what makes an asset custodyable for the domain (§18.5).

 Periphery carries **no address fields**: administration and emergency pause are both exercised by the same single network administrator address (§23), which lives in the node binary, not in Settlement storage.

### 5.3 Method list

**Normative:** Settlement **MUST** expose exactly the following methods in Phase 1. Each implements the standard embedded-contract `Method` interface — `GetPlasma` / `ValidateSendBlock` / `ReceiveBlock` (`vm/embedded/embedded.go`). Every user/executor input carries `domainId`; core methods are domain-keyed except the network-wide `RegisterExecutor`, `SetPeripheryConfig`, `Update` and `CollectReward` (§22, §23).

**Contract inputs are shared by every runtime.** The rows whose scope is *contract runtime* — `Deposit`'s delivery, `CallL2`, the three deploy methods, `UpgradeContract`, `TransferContractOwnership` and `BuyStorage` — are the input vocabulary of every contract runtime, not of one. They carry opaque bytes (bytecode chunks, `metadata`, `payload`), and Settlement's checks on them are runtime-independent: sizes, hashes, fees, deploy records and custody. Settlement **MUST NOT** branch on `runtimeKind` when accepting them. The domain's STF gives the bytes their meaning (§27.5). A new contract runtime is therefore a new `runtimeKind` value and STF, registered through `RegisterDomain`, and **MUST NOT** add Settlement methods for inputs these rows already express.

| Method | Origin | Scope | Purpose |
|---|---|---|---|
| `RegisterDomain` | Admin | core | Register a domain, or reconfigure a registered one (§6): `domainId`, `domainClass`, `runtimeKind`, `stfSpecHash`, `inputSource`, `execProfile`, `foreignProfile`, `profileConfig`, `chainBinding`, `minExecutors`, `proposerPolicy`, `finalityModel`, status, and — at registration only — the initial `withdrawalDelay` and `runtimeUpgradeDelay`. Phase 1 registers only WASM `EXECUTION` domains at the base profile point |
| `RegisterExecutor` | Admin (Phase 1); the pillar's producer address (Phase 2, reserved) | core | Bind a pillar and the executor address it chose into the network executor set, which serves every domain; under `SINGLE`, replaces the sole member; time-challenged at `AdministratorDelay` (§22) |
| `Deposit` | User | core + contract runtime | Lock an L1 ZTS asset into custody (core) and deliver it to a named `target_contract` with no application payload, which credits it (§18.2, §18.5). Funds a contract / credits the depositor |
| `SubmitBatch` | Executor | core | Register a `domainId` batch commitment (§19); records the batch's developer fees against its pillar for the batch's epoch (§22.4) |
| `RelayMessage` | Permissionless | core | Relay a proven, finalized `domainId` outbox message; registers L1 withdrawals for release (§17, §18.2) |
| `ReleaseWithdrawals` | Permissionless | core | Rate-limited maintenance crank: releases due withdrawals from the named domain's release queue, ≤ `MaxWithdrawalsPerUpdate` per call (§18.2, §27.9) |
| `Update` | Permissionless (pillars call it automatically) | core | No-argument epoch crank: settles at most one epoch of developer fees — burns the burn share and credits operator rewards (§22.4) |
| `CollectReward` | Reward address | core | Pay out the caller's accrued ZNN reward from the settled fee pot (§22.4) |
| `Pause` | Admin | core | Emergency pause — global or per-domain (`PAUSE_SUBMIT` / `PAUSE_RELAY`, §23) |
| `SetPeripheryConfig` | Admin | core | Set the network-wide Periphery scalars `znnPerByteOfBytecode` and `znnPerByteOfState`, each rejected unless within its Core hard bound; time-challenged at `SoftDelay` (§23) |
| `SetDomainConfig` | Admin | core | Set a domain's `withdrawalDelay` and `runtimeUpgradeDelay`, within their Core bounds; time-challenged at `SoftDelay`, per domain (§23) |
| `SetAssetConfig` | Admin | core | Set `MaxBatchWithdrawal` for a (domain, asset), within its Core bounds and never below the value in force (§27.9); time-challenged at `SoftDelay`, per (domain, asset) (§23) |
| `SetApprovedDAModes` | Admin | core | Replace a domain's approved `DAMode` set; a non-empty set of listed `DAMode` values (§20, §27.11); time-challenged at `SoftDelay`, per domain (§23) |
| `InitiateEscapeHatch` | Admin | core | Initiate the domain escape hatch for a stalled domain or one whose executor set is below `minExecutors`; time-challenged at `AdministratorDelay`, recording a durable escape-hatch record (§23) |
| `CallL2` | User | contract runtime | Submit a call into the domain's input stream; **payable** — MAY carry attached ZTS value delivered to the target contract as a deposit (§18.5, §27.5) |
| `DeployContractStart` / `…Chunk` / `…Finalize` | User | contract runtime | Chunked deployment (§11); `…Finalize` carries the bytecode fee in ZNN (§10.7) |
| `UpgradeContract` | User | contract runtime | Replace code of an `OWNER_UPGRADEABLE` contract (§12); carries the bytecode fee in ZNN (§10.7) |
| `BuyStorage` | Anyone | contract runtime | Buy state-storage quota for a contract, paying the storage fee in ZNN (§10.7, §13.8) |
| `TransferContractOwnership` | User | contract runtime | Transfer or renounce ownership of an `OWNER_UPGRADEABLE` contract (§12) |

Runtime upgrades for a domain are a governed delayed `stfSpecHash` bump (§6), not a per-call method.

Reserved methods, not exposed while the Phase-1 profile is in force: `RegisterVerifyingKey` (admin, §5.5), `RelayForeignLock` (relayer, Core-verified foreign lock for a `MINTED` asset, §18.4a) and `TimeoutMessage` (permissionless, §17.4).

### 5.4 Storage responsibilities

**Normative:** Settlement **MUST** maintain at least, all keyed by `domainId` where domain-scoped:

- The domain registry: `DomainRecord` per `domainId` (§6).
- The batch commitment chain per domain: one entry per `(domainId, batchId)` carrying `protocolVersion` (§24 — the stored entry keeps only this one of the nine version fields) and the commitment's remaining fields, including `proofData`, with the submitting executor's pillar name in place of the wire commitment's `executorId` (§19) — plus the batch's stored state (`SUBMITTED`; a batch is `FINALIZED` at every height from its `finalizeAtHeight`, §21, §27.11).
- The per-domain input-sequence cursor: `consumedInputCount` (how many of the domain's inputs accepted batches have consumed), `nextBatchId` (the `batchId` the next accepted batch must carry), `recordsSweptThrough` (the number of the domain's batches whose assigned-index records have been deleted) and `sweptInputCount` (the number of the domain's inputs those batches covered, so the next swept batch's first ordinal needs no scan). A domain with no accepted batch has `nextBatchId = 0`; no index value is reserved as a sentinel (§4.6, §19).
- Per-(domain, asset) `totalDeposited`, `pendingWithdrawalReserve`, `totalReleased` (§18).
- Pending chunked-deployment records keyed by `(domainId, deployer, pre-instrumentation code_hash)`, each a fixed-size header; one chunk-hash entry per received chunk, keyed by `(domainId, deployer, code_hash, startHeight, chunk_index)`; and the FIFO start queue of `(record key, startHeight)` entries that drives the expiry sweep (§11).
- The processed-outbox set for replay protection (§17.3).
- The global received-input counter; a per-domain received-input count; an assigned-index record `{globalInputIndex, assetId, amount, fee}` reachable by `(domainId, ordinal)` with one key lookup — either keyed by it or through an ordinal index, whichever entry is stored under `(domainId, ordinal)` also carrying the send's `source_account_block_hash` (§4.6) — where `ordinal` is the input's zero-based position among the domain's own inputs; and, separately, a per-(domain, asset) cumulative deposit total (§4.6, §18.4).
- The per-domain withdrawal release queue: registered, not-yet-released withdrawals in registration order, maintained with a per-domain head/tail cursor pair that makes release strictly FIFO (§18.2).
- The per-domain `ReleaseWithdrawals` rate-limit record (§18.2), and per-(domain, batch, asset) totals of registered contract-emitted withdrawals and registered refunds (§27.9).
- The network executor set: one record per registered pillar — pillar name, executor address, registration height, status (§22).
- Developer-fee accounting (§22.4): the fee carried by each assigned input, recorded with its assigned-index record; per-(epoch, pillar) pending amounts; the queue of epochs that hold pending amounts, oldest first, and for each the highest finalization height of its batches; the network-wide counters `totalFeesCollected`, `totalFeesBurned`, `totalFeesCredited` and `totalRewardsCollected`; and a `RewardDeposit` per reward address.
- **(reserved, gated)** The per-(domain, asset) `AssetRegistry`: `custodyMode` (`ESCROW`/`MINTED`) and, for `MINTED`, the `(foreignChainId, foreignAssetRef, decimalsPolicy)` binding and the counters `provenForeignLocks`, `mintedSupply`, `burnedSupply`, `authorizedForeignRelease` (§18.3). Absent/`ESCROW`-only while the Phase-1 profile is in force.
- **(reserved, gated)** Per-`outboxId` `MessageState` for outbox messages carrying a timeout/ack policy (§17.4); fire-and-forget messages need only `processedOutbox` membership.
- **(reserved, gated)** The Periphery verifier registry: write-once `VerifierEntry` records keyed by `verifyingKeyId`, each naming a built-in verifier and a verifying key (§5.5).
- Global pause state: the `PAUSE_SUBMIT` and `PAUSE_RELAY` flags applying to every domain (§23).
- Per-domain pause state: the same two flags, per `domainId` (§23).
- Per-domain escape-hatch records: the momentum height at which the escape hatch was initiated for a domain (§23).
- Periphery configuration, carrying no address fields (§23): one network-wide record (`znnPerByteOfBytecode`, `znnPerByteOfState`); one record per domain (withdrawal delay, runtime-upgrade delay); one record per (domain, asset) (`MaxBatchWithdrawal`); and one approved-`DAMode` set per domain, under its own key prefix (§20).

### 5.5 Verifier registry and dispatch (reserved, gated)

Zenon L1 runs no user-deployed code: every contract is embedded Go logic in the node binary. A proof verifier therefore cannot be deployed on-chain the way an EVM verifier contract is; every verifier Settlement can call is compiled into the node.

**Normative:**

- **Built-in verifiers.** Core **MUST** carry a fixed table of verifier implementations, each identified by `verifierId : u16` and typed by `verifierKind` (§27.11): `VALIDITY`, `FOREIGN_CONSENSUS` or `FOREIGN_COMMITTEE`. Adding or changing a verifier is a node release activated by spork. No on-chain method installs verifier code.
- **Registry entries.** The Periphery registry holds write-once entries keyed by `verifyingKeyId : u32`, assigned sequentially from `0`:

  ```
  VerifierEntry := verifierId : u16 ; verifyingKey : Bytes (≤ MaxVerifyingKeySize) ; verifyingKeyHash : bytes32
  ```

  `verifyingKeyHash` **MUST** be `SHA3-256(verifyingKey)`. `RegisterVerifyingKey` (admin, §23) creates an entry, time-challenged at `AdministratorDelay`; Core **MUST** reject it when `verifierId` names no activated verifier or the verifier's own key check rejects `verifyingKey`. An entry **MUST NOT** be overwritten or deleted.
- **Binding and rotation.** A domain names its entries in `profileConfig` (§6.3, §6.4); every `*VerifierRef` and `verifyingKeyId` there is a `verifyingKeyId`, and the entry's `verifierKind` **MUST** match the profile arm naming it. Pointing a domain at a different entry changes `profileConfig`, so it is a `stfSpecHash` change under `MinRuntimeUpgradeDelay ≥ WithdrawalDelay` (§6). A key is rotated by registering a new entry and repointing, never by editing one.
- **Dispatch.** Core invokes a verifier as `Verify(verifierId, verifyingKey, publicInputs, proof) → accept | reject`. Every built-in verifier **MUST** be pure and deterministic, **MUST** declare a maximum `proof` size and a fixed plasma cost charged to the calling account-block, and **MUST** reject rather than error on malformed input. `publicInputs` is fixed per kind:
  - `VALIDITY`: `SHA3-256` of the batch's canonical commitment encoding (§19) with `proofData` as the zero-length `Bytes` encoding.
  - `FOREIGN_CONSENSUS` and `FOREIGN_COMMITTEE`: the relayed foreign fact's canonical encoding, fixed per chain by `chainBinding.chainVerifierId` (§6.6).

  The format of `proof` is fixed by each verifier's own specification (`../phase-2/` for `VALIDITY`), not here.

---

## 6. Domain model

Settlement anchors **domains**; a domain is the unit of runtime + STF + input source + execution profile + foreign-fact profile + proposer policy + state-root lineage + optional foreign-chain binding. Domains do not own executors: one network executor set, bound to pillars, serves every domain (§6.2, §22). **Phase 1 registers only WASM `EXECUTION` domains at the base profile point** (`ATTESTATION` / `NONE` / `ESCROW` / `SINGLE`, §1). The schema below is the general model; Phase 1 is its degenerate case, so no migration is needed to add profiles, bridges, or messaging later.

A domain carries a `domainClass` — `EXECUTION` (general computation, e.g. WASM), `BRIDGE` (moves value to/from a foreign chain, with custody), or `MESSAGING` (moves cross-chain data only, no custody). `EXECUTION` is the only class activated in Phase 1; `BRIDGE`/`MESSAGING` are reserved (§6.5).

**DomainRecord** (Settlement storage, keyed by `domainId`):

```
DomainRecord {
    domainId        // u32 identifier, assigned at RegisterDomain
    domainClass     // EXECUTION | BRIDGE | MESSAGING — informative classification (§6.5)
    runtimeKind     // WASM — the only kind with a handler in Phase 1
    stfSpecHash     // pins runtimeKind, the concrete execution-engine implementation and its exact
                    //   version (not merely the WebAssembly specification level, since the engine
                    //   decides which modules validate), genesis, input-envelope + commitment
                    //   format, AND domainClass/inputSource/execProfile/foreignProfile/chainBinding (§24)
    inputSource     // L1_NATIVE | L1_RELAYED — where inputs originate (§6.1)
    execProfile     // ATTESTATION | ZK_VALIDITY — secures the batch STF (§6.3)
    foreignProfile  // NONE | COMMITTEE | LIGHT_CLIENT | ZK_CONSENSUS — secures foreign facts (§6.4)
    profileConfig   // tagged union carrying each profile's parameters (§6.3, §6.4)
    chainBinding    // ChainBinding | null — non-null iff domainClass ∈ {BRIDGE, MESSAGING} (§6.6)
    minExecutors    // 1 in Phase 1; while the network executor set (§6.2) is smaller, the domain cannot finalize batches
    proposerPolicy  // SINGLE — entitled-proposer rule (§6.2)
    finalityModel   // DELAY | PROOF_VERIFIED — derived from execProfile (§6.3, §21)
    status          // ACTIVE / PAUSED — PAUSED closes the domain to new inputs (§4.1)
}

ChainBinding {                 // present only for BRIDGE / MESSAGING domains (§6.6)
    foreignChainId    // canonical foreign-chain identifier
    chainVerifierId   // pins the ChainVerifier implementation (foreign-fact verification, §6.6)
    genesisAnchor     // light-client checkpoint: header / sync-committee root / bank hash
    releaseAdapterRef // foreign-side ReleaseAdapter identifier (addr / script / graph id, §6.6)
    finalityParams    // per-chain confirmation depth / finality predicate (pre-mainnet, §6.4)
}
```

The numeric encoding of every enumerated field above — and of batch state, executor status, and pause scope — is normative in §27.11.

`profileConfig` and `chainBinding` are carried as byte strings; the block above is a field list, not a field-addressable schema. Their inner encodings are an open implementation item (§30).

> **Settlement Core change (gated).** `domainClass`, `execProfile`, `foreignProfile`, `profileConfig`, `chainBinding`, and `finalityModel` are semantics-defining Core additions folded into `stfSpecHash` (§24); activating any non-Phase-1 value requires a spork-gated `znnd` release. Until then Core **MUST** constrain them to the Phase-1 defaults below and reject any other value.

**Normative:**

- `RegisterDomain` is **admin-only** (the network administrator, §23) in Phase 1 and registers only domains with the Phase-1 profile (`domainClass = EXECUTION`, `runtimeKind = WASM`, `inputSource = L1_NATIVE`, `execProfile = ATTESTATION`, `foreignProfile = NONE`, `chainBinding = null`, `minExecutors = 1`, `proposerPolicy = SINGLE`, `finalityModel = DELAY`). `domainClass`, `inputSource`, `execProfile`, `foreignProfile`, `chainBinding`, `proposerPolicy`, and `finalityModel` are fixed at registration and changeable only by the governed mechanisms of §6.1–§6.6 (a change to any semantics-defining field is a runtime-semantics change and **MUST** be treated as a `stfSpecHash` bump under `MinRuntimeUpgradeDelay`).
- **Domain ids and reconfiguration.** `domainId` values are assigned sequentially from `1`; `0` names no domain. The caller passes `domainId`: a value equal to the next unassigned id registers a new domain, a lower value reconfigures that registered domain, and a higher value **MUST** be rejected. A reconfiguration **MUST** reject any change to a field fixed at registration (`domainClass`, `runtimeKind`, `inputSource`, `execProfile`, `foreignProfile`, `profileConfig`, `chainBinding`, `proposerPolicy`, `finalityModel`), so a repeated registration call fails rather than silently reconfiguring a domain. A reconfiguration identical to the stored record and delays changes nothing and **MUST** likewise be rejected, so an exact repeat fails too; this is the rule that catches a repeat while the Phase-1 clamp pins every fixed field. It may change only `minExecutors` (time-challenged at `AdministratorDelay`, §23), `status`, and `stfSpecHash` (under the runtime-upgrade delay); the initial `withdrawalDelay` and `runtimeUpgradeDelay` are accepted at registration only, and changed afterwards through `SetDomainConfig`: a reconfiguration whose delays differ from the domain's stored ones **MUST** be rejected.
- **Initial delays.** At registration Core **MUST** validate the initial `withdrawalDelay` (`IsWithdrawalDelayInBounds`) and the initial `runtimeUpgradeDelay` against it (`IsRuntimeUpgradeDelayValid`). Registration needs no time challenge: a new domain has no depositors to protect.
- **Phase 1:** `profileConfig` **MUST** be empty. Its contents are folded into `stfSpecHash`, and no Phase-1 profile defines any.
- **Domain `status`.** A `PAUSED` domain accepts no new inputs (§4.1). It is otherwise unaffected: inputs it has already accepted are still batched, and its withdrawals are still registered and released, under the two pause scopes of §23 like any domain. `status` is an administrator lifecycle setting, set through `RegisterDomain`; it is not an emergency pause scope.
- All conservation, commitment, deposit, and withdrawal accounting **MUST** be keyed by `domainId` (and, for asset accounting, by `(domainId, asset)`). This per-domain keying is what gives **domain isolation**: a faulty/dishonest domain can never pay out beyond its own deposits, and a batch for one domain can never reach another domain's custody.
- Routing: inputs reach a domain as a `{domainId, payload}` envelope sent to Settlement — never via an off-chain channel, so DA stays inherited. Value-bearing operations (`Deposit`, withdrawals) always route through Settlement because escrow lives there. The executor filters the confirmed momentum stream by `domainId`, once per domain it serves — which is every active domain (§6.2).

**What Settlement governs (domain lifecycle):** `RegisterDomain`; the network executor set (§22); **runtime upgrades as `stfSpecHash` bumps under a mandatory delay** (the L2-node-software upgrade path, giving users an exit window); domain pause; and the **escape hatch** for a stalled domain or one below `minExecutors` (forced withdrawals against the last finalized root, an administrator action in Phase 1).

**Runtime upgrade delay floor (normative):** the mandatory delay on a `stfSpecHash` bump **MUST** satisfy `MinRuntimeUpgradeDelay ≥ WithdrawalDelay`. A runtime upgrade changes the STF for all future roots and is the highest-impact change this protocol permits; users must have at least as long to observe and exit as the withdrawal delay gives them on any other finality boundary. Core **MUST** enforce this as a hard bound; the administrator **MUST NOT** configure a shorter upgrade delay regardless of Periphery settings. The floor is evaluated against the domain's own withdrawal delay in force after the configuring call — when both values are set in the same call (`RegisterDomain` at registration, or `SetDomainConfig`), against the withdrawal delay being set, not the one it replaces.

**What lives inside the domain's STF (not Settlement):** the meaning of `CallL2`/deploy/upgrade, the L2 contract registry, per-contract code/version/owner, and the SMT layout — all defined by the WASM runtime. "The executor ran the wrong code/version" is therefore ordinary state divergence (attested in Phase 1, proven under `ZK_VALIDITY`), not a special Settlement failure mode.

> **One WASM runtime, open to others.** Phase 1 builds and runs a single runtime under one pillar-operated executor. The domain-agnostic core does not preclude a third party proposing another runtime later: that is a new domain handler shipped into `znnd` and a new executor plugin, activated by governance/spork. Adopting a domain is a network decision, so every executor runs its plugin from the activation height, after the grace period of §22.6. Synchronous and cross-domain messaging are out of Phase 1 scope; cross-domain interaction, when it exists, is asynchronous (outbox → finalize → relay into the other domain's inbox).

### 6.1 Input-source descriptor

**Normative:** every domain **MUST** declare `inputSource` ∈ `{ L1_NATIVE, L1_RELAYED }`. The value is binding on how the executor and any verifier establish the canonical input stream, and on the data-availability and censorship guarantees the domain offers; it **MUST** be surfaced to users and integrators as part of the domain's trust profile.

| `inputSource` | Inputs originate as | Canonical order | Data availability | Censorship-resistance |
|---|---|---|---|---|
| `L1_NATIVE` | Settlement account-blocks (§4.1) | L1 momentum order (§4.4) | Inherited from L1 | Native force-inclusion (§4.2) |
| `L1_RELAYED` | External data submitted to Settlement as account-blocks | L1 momentum order of the relaying account-blocks | Inherited from L1 | Permissionless relay + force-inclusion |

- For `L1_NATIVE` and `L1_RELAYED`, the canonical order **MUST** be the L1 order of the originating account-blocks (§4.4); the executor **MUST NOT** apply any other order. For `L1_RELAYED`, the domain STF **MUST** establish input validity and canonicity (e.g. external-chain linkage, proof-of-work, confirmation depth) and **MUST** reject any relayed datum that does not extend the domain's canonical view; relaying **MUST** be permissionless.
- For both kinds the executor **MUST NOT** reorder, skip, or privately insert inputs (§4.7). Every input is an L1 account-block, so completeness is enforced by the same per-domain input-sequence contiguity as force-inclusion (§4.6).

**Normative (Phase 1):** only `L1_NATIVE` **MAY** be activated. `L1_RELAYED` is reserved; Core **MUST** reject a `RegisterDomain` selecting a reserved `inputSource` while the Phase-1 profile is in force.

Every input of every domain is therefore an L1 account-block, and canonical order, data availability and completeness are L1's in every profile. A foreign source whose raw data is too large to relay reaches a domain through a compact foreign-fact proof — a `ZK_CONSENSUS` proof or a `COMMITTEE` signature (§6.4) — relayed as an `L1_RELAYED` account-block. No input source lets the executor select inputs off-chain.

> *Informative.* `L1_RELAYED` makes an external domain a strict special case of an internal one: inputs are ordinary L1 account-blocks, so they receive a `globalInputIndex`, inherit L1 DA, and are ordered and committed by the §4/§19 machinery. The only new logic is inside the STF (e.g. SPV header validation). External bridges are light-client-shaped (headers + inclusion proofs, not whole chains), so the L1 data cost is small.

### 6.2 Executor set and proposer policy

**Normative:**

- Settlement **MUST** hold a single **network executor set**, shared by every domain. It is not a field of any `DomainRecord`. Each member is a pillar registered in the pillar contract, identified by its pillar name, together with the executor address that pillar chose (§22.1). A member posts no bond.
- Every member **MUST** serve every `ACTIVE` domain. Registration is not per domain, and a member **MUST NOT** be able to opt out of a domain. A domain whose foreign facts are relayed (`L1_RELAYED`, §6.1) imposes no extra infrastructure on executors: only the permissionless relay role reads the foreign chain, and executors consume the relayed L1 account-blocks (`EXECUTOR.md` §2).
- `RegisterExecutor` (§22.2) adds a member (Phase 2) or replaces the sole member (Phase 1). A replacement's first batch `preStateRoot` for each domain **MUST** equal that domain's last accepted `postStateRoot`, and registration changes **MUST NOT** alter finalized state.
- `proposerPolicy` is per domain and selects the **entitled proposer** for a given batch slot of that domain from the network executor set:
  - `SINGLE` — the set has exactly one member, always entitled. **Phase 1 MUST use this.**
  - Ordinals `1` and `2` are unassigned, and Core **MUST** reject them. The proposer rule for a multi-member set is specified with Phase 2 (`../phase-2/`).
- State lineage remains linear: at most one batch is accepted per cursor position (§19); a non-entitled or non-canonical submission **MUST** be rejected.

- A domain **MUST NOT** finalize batches while the network executor set has fewer members than its `minExecutors`.

**Normative (Phase 1):** the network executor set **MUST** contain exactly one member, `minExecutors` **MUST** be 1, and `proposerPolicy` **MUST** be `SINGLE` — observationally identical to the single-executor rule of §22. The stored batch entry records the submitting pillar (§19), so moving to a Phase-2 set requires no commitment-format change.

### 6.3 Execution-profile ladder (secures the batch STF)

`execProfile` determines how Settlement treats `preStateRoot`/`postStateRoot`, what `SubmitBatch` verifies, the `finalityModel`, and the contents of the reserved `proofData` field (§19). It is the two-phase roadmap of §2.2 lifted to a per-domain field.

```
profileConfig (execProfile arm) :=
  ATTESTATION  { attesterPolicy }                    // single executor (Phase 1) or quorum (reserved)
  ZK_VALIDITY  { validityVerifierRef, verifyingKeyId, publicInputSchema }
```

Every `*VerifierRef` and `verifyingKeyId` in `profileConfig` is a §5.5 registry `verifyingKeyId`.

| `execProfile` | `SubmitBatch` verifies | `proofData` carries | `finalityModel` | On-chain correctness |
|---|---|---|---|---|
| `ATTESTATION` | proposer entitlement, contiguity, size, `assetFlowSummary`, chaining — **not execution** | empty (single executor, Phase 1) | `DELAY` (§21) | none (Rail 1) |
| `ZK_VALIDITY` | as `ATTESTATION`, plus verifies a validity proof against `verifyingKeyId` before finalization | the SNARK over the batch transition | `PROOF_VERIFIED` (**MAY** be immediate) | full (the proof) |

**Normative:**
- A single batch transition **MUST** satisfy `postStateRoot = F(preStateRoot, input_data, code_hash)` in every profile; the profile changes only *who checks it and when*, never the function (§17.1). A validity proof **MUST** cover the identical canonical applier (§13.5.1), including the deposit-settlement step (§18.5), so "the executor ran the wrong code/version" is ordinary state divergence, not a special Settlement failure mode.
- Under `ZK_VALIDITY`, Settlement **MUST** verify the validity proof before a batch finalizes and **MUST** reject a batch whose proof does not verify against the registered `verifyingKeyId`.
- `proofData` **MUST** be the zero-length encoding under single-executor `ATTESTATION`; Core **MUST** reject a non-empty `proofData` outside the profiles/phases that define its contents (§19, §24).
- **Phase 1:** only single-executor `ATTESTATION` **MAY** be activated, and only on testnet (§1); `ZK_VALIDITY` requires the validity-verifier registry (§5.5), which is reserved. `OPTIMISTIC` (`1`) is a reserved ordinal: it names no profile, and Core **MUST** reject it (§27.11).
- **Mainnet:** `ZK_VALIDITY` is the execution profile mainnet requires (§1); its proof system is specified in `../phase-2/`.

### 6.4 Foreign-fact profile ladder (secures external-chain truth)

For `BRIDGE` and `MESSAGING` domains, `foreignProfile` determines how the `ChainVerifier` (§6.6) establishes that a foreign event happened and is final. This ladder is **orthogonal** to §6.3: it secures the *truth of a foreign fact consumed as input*, not the *correctness of the batch's computation*.

```
profileConfig (foreignProfile arm) :=
  NONE         { }                                   // non-bridge EXECUTION domains
  COMMITTEE    { attesterSet, threshold }            // relayer/oracle quorum
  LIGHT_CLIENT { clientKind, finalityRule }          // PoW | sync-committee | tower-bft | … (per chain)
  ZK_CONSENSUS { consensusVerifierRef, verifyingKeyId }
```

| `foreignProfile` | Foreign truth established by | Tolerable adversary |
|---|---|---|
| `COMMITTEE` | a quorum of foreign attesters signs the event; Core or the STF checks the threshold signature | `< threshold` colluding attesters |
| `LIGHT_CLIENT` | foreign consensus + inclusion verified from relayed headers/proofs inside the STF | `< foreign safety threshold` |
| `ZK_CONSENSUS` | a succinct proof of foreign consensus + inclusion | `< foreign safety threshold` + proof soundness |

**Normative:**
- `foreignProfile = NONE` iff `domainClass = EXECUTION`.
- Under `LIGHT_CLIENT` and `ZK_CONSENSUS`, the `ChainVerifier` **MUST** reject any relayed datum that does not extend the verified light-client view; a dishonest relayer **MUST** be unable to advance the foreign view to a state the foreign chain did not reach (Rail 2).
- `finalityRule` and `chainBinding.finalityParams` are pre-mainnet per chain and **MUST** be chosen so that no Zenon-side release ever finalizes over a re-organizable foreign event (§21).
- **Phase 1:** only `NONE` **MAY** be activated; `COMMITTEE`/`LIGHT_CLIENT`/`ZK_CONSENSUS` are reserved.

> The two ladders compose. A Bitcoin bridge is naturally `(execProfile = ZK_VALIDITY, foreignProfile = LIGHT_CLIENT)`: the BTC lock is SPV-verified inside the proven STF. An Ethereum bridge is `(ZK_VALIDITY, LIGHT_CLIENT)`; a high-assurance target is `(ZK_VALIDITY, ZK_CONSENSUS)`. Settlement mandates no pairing; §6.5 constrains only the legal combinations.

### 6.5 Domain classes, legal combinations, and custody modes

`domainClass` classifies a domain (informative, but load-bearing for the rules below): `EXECUTION` runs general computation; `BRIDGE` moves value across a foreign-chain boundary with custody; `MESSAGING` moves cross-chain data only, with no custody. Each `(domain, asset)` also carries a **custody mode** — `ESCROW` (a real ZTS held 1:1 in Settlement custody, §18.1) or `MINTED` (a representation asset backed by attested foreign locks, §18.3).

**Normative:** Core **MUST** reject a `RegisterDomain` violating any of:
1. `chainBinding = null` ⟺ `domainClass = EXECUTION` ⟺ `foreignProfile = NONE`.
2. `MESSAGING` domains **MUST** define no `custodyMode` for any asset and **MUST NOT** emit `kind=1` withdrawal authorizations (§17); they emit only `kind=0` cross-domain/cross-chain messages.
3. `custodyMode = MINTED` **MUST** satisfy the §18.3 floor (`execProfile = ZK_VALIDITY` and a `MaxBatchMint` cap) and **MUST** reconcile through `MintFlowSummary` (§18.4a), which requires `protocol_version = 2` (§19). A Core-verifiable `foreignProfile` (`COMMITTEE`, `ZK_CONSENSUS`) strengthens the mint side but does not replace the floor: burns and per-account balances are still enforced only by the validity proof.
4. `execProfile = ZK_VALIDITY` requires the validity-verifier registry (§5.5). Core **MUST** reject it until its registry entry exists.

**Migration** along either ladder is a `stfSpecHash` bump under `MinRuntimeUpgradeDelay ≥ WithdrawalDelay`, is a **strict strengthening**, and changes no commitment format and no custody accounting: the commitment already carries `proofData` (empty → validity proof) and `executorId` (single → set member), so no on-chain footprint changes except added Periphery verifier registries. A migration **MUST NOT** alter any finalized root, receipt, or conservation counter, and the replacement's first batch `preStateRoot` **MUST** equal the last accepted `postStateRoot`.

### 6.6 Bridge abstractions and the peg-out authorization

A `BRIDGE`/`MESSAGING` domain is still an ordinary domain served by the executor's four-method plugin (`EXECUTOR.md` §7); the bridge-specific logic lives entirely inside its STF and in two per-chain seams. Settlement is unaware which class a domain is — it anchors batches and enforces custody/conservation identically (Rail-independent, §3).

- **`ChainVerifier` (read seam, foreign → Zenon).** The per-chain foreign-fact verifier that implements `foreignProfile` inside the STF: verifies foreign headers/finality and event inclusion. Pure and deterministic (no live foreign reads inside the STF — `EXECUTOR.md` §10). Defined in `EXECUTOR.md` §7.
- **`SourceAdapter` (relay seam).** The per-chain reader the relay role uses to post foreign data (headers + inclusion proofs) to Settlement as `L1_RELAYED` account-blocks. Permissionless and trustless — the STF re-validates everything (§6.1, §4.2). Defined in `EXECUTOR.md` §7 (`Source`).
- **`BridgeLedger` (shared STF library).** The chain-neutral state machine (mint/burn/pool/conservation/peg-out/replay/fraud) a bridge domain's `Apply` delegates to; only the injected `ChainVerifier` and the asset differ between chains. A library, not a second Settlement.
- **`ReleaseAdapter` (write seam, on the foreign chain).** Consumes a finalized Zenon peg-out authorization and releases the real foreign asset. Settlement defines the authorization object and the obligations, **not** the adapter's code.

**Peg-out authorization (normative).** A bridge withdrawal is expressed as an ordinary `kind=1` outbox message (§17.2) — there is **no** per-chain withdrawal contract on Zenon. After finality the message is the authorization a `ReleaseAdapter` consumes on the foreign chain:

```
PegOutAuthorization := { domainId, outboxId, asset, amount, recipient(foreign), burnRef }
```

A conformant `ReleaseAdapter` deployment **MUST**: release at most once per `outboxId` (foreign-side replay protection mirroring `processedOutbox`, §17.3); verify, under every `execProfile`, a proof of Zenon consensus and of Settlement storage showing the authorization in the `outboxRoot` of a `FINALIZED` batch (under `ZK_VALIDITY` Settlement has already verified the batch's validity proof, so the adapter does not re-verify it); and release exactly `amount` of `asset` to `recipient`. The Zenon side **MUST NOT** depend on `ReleaseAdapter` liveness for safety — Settlement only emits the authorization; a withheld or buggy releaser delays but cannot over-release, since the authorization is one-per-`outboxId` and conservation-bounded (§18.3). Per-chain instantiations (BitVM graph / covenant script for BTC; verifier contract for EVM; program for SVM) and the exact proof object are out of scope here and specified per chain (`../research/bridges.md` §4.5).

---

## 7. Runtime profile

### 7.1 WebAssembly Core 1.0 only

**Normative:**

- The runtime **MUST** implement WebAssembly Core Specification 1.0 (the MVP feature set) and no more, interpreter profile.
- It **MUST NOT** enable any post-MVP proposal unless a future version of this spec enumerates it.
- The deployment validation pipeline (§11) **MUST** reject any module using out-of-set features.

### 7.2 Prohibited capabilities

**Normative:** the runtime **MUST NOT** expose, and validation **MUST** reject modules using:

- **WASI:** any import from `wasi_snapshot_preview1` or any WASI namespace.
- **Floating point:** any `f32.*`/`f64.*` instruction or float type, including in imported signatures.
- **Threads / atomics:** the threads proposal, shared memory, any atomic instruction.
- **SIMD:** any `v128` instruction.
- **Nondeterminism sources:** system clocks, OS randomness, uninitialized-memory-dependent behavior, growth-dependent layout exposure.
- **Post-MVP integer ops:** the sign-extension operators (`i32.extend8_s`, `i32.extend16_s`, `i64.extend8_s`, `i64.extend16_s`, `i64.extend32_s`) and saturating conversions (`*.trunc_sat_*`), which are outside strict Core 1.0 MVP. They are never metered (`wasm-gas-table-v1`) and **MUST** be rejected at validation.

> Floating point is prohibited because NaN canonicalization is platform-dependent. Determinism is the precondition for the validity proofs mainnet requires (`ZK_VALIDITY`).

### 7.3 Deterministic context

**Normative:**

- Linear memory **MUST** be deterministically initialized before contract entry: zero everywhere except the regions written by the module's own active data segments (Core 1.0, §7.1) and the input frame (§27.1). Table entries **MUST** be deterministically initialized.
- The only host imports are the three in §8. No other host function exists.
- Given the same execution context (§9), the same witnessed pre-state, and the same input data, every compliant runtime **MUST** produce a byte-identical `ExecutionResult` (§14).

### 7.4 Resource limits

**Normative:**

- Linear memory **MUST NOT** exceed the module's memory limit: the smaller of its declared maximum and `MaxMemory` (§25), or `MaxMemory` when it declares no maximum. A `memory.grow` that would exceed the limit **MUST** return `-1`, **MUST NOT** grow, and is charged only the `memory.grow` base cost (§10.2). Deployment validation **MUST** reject a module whose declared initial memory exceeds `MaxMemory` (§11.3).
- Call-stack depth **MUST NOT** exceed `MaxStackDepth` (§25), enforced as a fixed constant identical across implementations.
- These limits **MUST** be enforced by deterministic instrumentation injected at deployment (§11). *This instrumentation is the highest-risk Phase 1 item and is retired first (§29).*

### 7.5 Traps and aborts

**Normative:**

- All WASM traps (unreachable, integer division by zero, trapping-op overflow, OOB memory/table, indirect-call type mismatch, uninitialized table element, stack exhaustion) **MUST** canonicalize to receipt status `RUNTIME_FAULT` (§15) with full rollback for the affected input.
- A contract `abort` (§8) **MUST** halt immediately, record `REVERT`, and roll back the affected input. The receipt commits the abort code as `abortCode` and the revert data through `returnHash` (§15, §27.7). For `abort(code, data_ptr, data_len)` the code is `code` and the revert data is the `data_len` bytes at `data_ptr` (§8.1). For a `run` returning `r < 0` the code is `(-r)` truncated to `i32`, so `r = -2^63` yields `0`, and the revert data is empty. Every receipt whose status is not `REVERT` **MUST** carry `abortCode = 0`.
- Trap/abort handling **MUST** be deterministic.

---

## 8. Minimal host ABI

### 8.1 Allowed imports

**Normative:** the Phase 1 host ABI **MUST** expose exactly three imports; validation **MUST** reject any module importing any other symbol.

A WASM import is a `(module, name)` pair. The module string for all three **MUST** be `"env"`. Validation **MUST** reject a module importing any of the three symbols under any other module string, importing the same `(module, name)` pair twice, or declaring a fourth import. Each of the three **MUST** be imported as a function with exactly the signature below (`abort`'s `-> !` is a function type with no result). Validation **MUST** reject an import of any of the three names as a table, memory or global, or with any other signature.

```
// Byte length of the value at key as a non-negative i64 when present; -1 when absent.
// Present-empty (0) and absent (-1) are DISTINCT (§13.6, §27.8).
storage_len(key_ptr: i32, key_len: i32) -> i64

// Reads value into out buffer. Returns bytes written; -1 if absent; -2 if buffer too small.
// Every observed key (including absent observations) is witnessed (§20).
storage_read(key_ptr: i32, key_len: i32, out_ptr: i32, out_len: i32) -> i64

// Intentional contract-level failure: halts immediately, full rollback; the abort code and
// revert data are committed (§7.5, §15). Distinct from a WASM trap.
abort(code: i32, data_ptr: i32, data_len: i32) -> !   // no return
```

`storage_len` and `storage_read` **MUST** trap when `key_len` exceeds `MaxStateKeySize` or `[key_ptr, key_ptr + key_len)` lies outside linear memory. `storage_read` then resolves in this order, where `L` is the value's byte length:

1. Absent key: return `-1`.
2. `out_len < L`: return `-2`. The runtime does not check `[out_ptr, out_ptr + out_len)` against linear memory.
3. `[out_ptr, out_ptr + out_len)` outside linear memory: trap, even when `L` bytes at `out_ptr` would fit.
4. Otherwise write the `L` bytes at `out_ptr` and return `L`.

`abort` reads `data_ptr` and `data_len` as unsigned 32-bit values and, after its host charge (§10.3), resolves in this order:

1. `data_len > MaxReturnData`: halt with `RESULT_TOO_LARGE` and full rollback. Linear memory is not read.
2. `[data_ptr, data_ptr + data_len)` outside linear memory: trap.
3. Otherwise halt with `REVERT`, `code` as the abort code and the `data_len` bytes at `data_ptr` as the revert data. `data_len = 0` is an abort with empty revert data, whatever `data_ptr` holds.

A trap in a host call is a trap during `run` (§7.5, §27.1).

### 8.2 All other effects are returned, not called

**Normative:**

- All effects other than reads **MUST** be returned through the canonical result blob (§14): `StateDiff`, `Events`, `OutboxMessages`, `ReturnData`.
- The entry point **MUST** return a single packed `(pointer, length)` identifying the serialized `ContractEffects` in linear memory (exact ABI in §27.1). The runtime reads it, validates it, charges post-execution effect gas (§10), computes `preStateRoot`/`postStateRoot`, and applies it only on success.
- The following host functions **MUST NOT** exist in Phase 1: `state_write`, `transfer`, `withdraw`, `emit_event`, `send_message`, `set_return_data`, `sha3_256`.

> The declarative returned-blob model separates computation from state application (execution is a pure function from witnessed reads + input to an effects blob), removes write-ordering ambiguity (the contract declares the final value per key; the host applies in canonical key order), and minimizes what a validity proof must cover (prove "module + inputs ⇒ blob"; SMT application is a separate, simpler stage). This is the single most important ABI decision for keeping validity proofs tractable.

---

## 9. Execution context

**Normative:** each execution **MUST** be provided these deterministic context values, supplied by the runtime (none via a host call):

- `chain_id` (§9.1).
- `caller` (§9.2).
- `momentum_height`: height of the confirmed momentum that contains the input's send account-block (§4.6).
- `momentum_timestamp`: that momentum's `TimestampUnix`, a coarse reference the producing pillar can set within tolerance. Contracts **MUST NOT** use it as a high-precision clock or as security-critical entropy. It is non-decreasing across inputs in `globalInputIndex` order, because Settlement assigns indices in the canonical order of the sends (§4.4, §4.6). Contracts **MAY** rely on non-decrease but **MUST NOT** rely on strict increase, since multiple inputs in the same momentum share one timestamp.
- `momentum_hash`: that momentum's hash. The producing pillar chooses the momentum's content, so the hash is grindable: contracts **MUST NOT** use it as unbiased randomness.
- `contract_id` (§12).
- `code_hash`: the post-instrumentation `CodeHash` (§11, §12).
- `input_index`: the `globalInputIndex`.

The runtime **MUST NOT** allow a contract to set, spoof, or override any context value, in particular `caller`. Context is presented through runtime-populated entry-point parameters / a context region (§27.1), not host functions; there is no `get_caller`/`get_code_hash`.

### 9.1 chain_id

**Normative:** `chain_id` **MUST** be `SHA3-256("ZENON_WASM_L2:" || network_name)` (`network_name` ∈ `"mainnet"`, `"testnet"`, named devnet), a fixed 32-byte constant derived once from the configured network name, never changing for the life of the network. It **MUST** be incorporated into `ContractID` derivation (§12) and replay protection (§26).

### 9.2 caller derivation

**Normative:**

- For L1-originated inputs, `caller` **MUST** be `SHA3-256("L1_CALLER:" || raw_l1_address)`.
- For contract-originated executions (an outbox message delivered to a contract, §17), `caller` **MUST** be the source contract's `ContractID`, with no additional hashing.
- The two caller domains **MUST** be structurally distinguishable; the domain separator guarantees an L1 caller can never collide with a `ContractID`.
- **Native forms.** `caller` and `ContractID` (§12) are the canonical 32-byte identities, and every field this specification defines carries the canonical form. A runtime **MAY** expose a shorter native form of either to its contracts, for a contract model that expects one. A native form **MUST** be a deterministic function of the canonical identity alone, **MUST NOT** replace a canonical identity in any field this specification defines, and the runtime's specification **MUST** state its collision bound. The WASM runtime exposes no native form.

---

## 10. Plasma and gas

### 10.1 Two separate ledgers

**Normative:**

- **L1 plasma** pays for account-block admission and is governed entirely by Zenon L1 rules. It **MUST NOT** be conflated with L2 gas.
- **Settlement send plasma.** The required plasma of a send to the Settlement contract **MUST** be `max(P_method, AccountBlockBasePlasma + len(data) × ABByteDataPlasma)`, where `P_method` is the method's own `GetPlasma` value and `data` is the account-block's data field. A Settlement send therefore never costs less than a plain send carrying the same bytes. Other embedded contracts keep their flat per-method prices, which replace the per-byte charge (`GetBasePlasmaForAccountBlock`, `vm/plasma.go`); Settlement differs because its sends carry up to `MaxDataLength` bytes that stay in the momentum chain permanently. The rule is evaluated in `GetBasePlasmaForAccountBlock`, since `GetPlasma` does not see the block, and every path that computes a send's required plasma **MUST** apply it. It changes only blocks addressed to Settlement, which exist only once `OffchainExecutionSpork` is active (§5.1).
- **L2 execution gas** is a separate ledger metered and enforced by the executor off-chain; it never touches L1 plasma consensus.
- `CallL2.plasma_limit` is an **L2 execution budget** authorized by the submitter and enforced by the executor — **not** L1 fused plasma.
- Only `CallL2` and `Deposit` inputs carry a `plasma_limit` (§27.5); `DeployContractStart`, `DeployContractChunk`, `DeployContractFinalize`, `UpgradeContract`, `TransferContractOwnership` and `BuyStorage` carry no L2 execution budget. Under `metering_version = 1` these six input kinds are therefore not subject to L2 gas accounting in any form — no initial-memory charge, no host charge, and no §10.4 effect charge — and their receipts **MUST** carry `gasUsed = 0` (§15, §27.7).
- The L2 gas budget for a `CallL2` or `Deposit` input **MUST** be derived from that input's `plasma_limit` one-for-one — one unit of budget per unit of `plasma_limit`, with no scaling factor and no conversion constant.
- `plasma_limit` **MUST NOT** exceed `MaxInputGas` (§25.1). Settlement **MUST** reject a `CallL2` or `Deposit` whose `plasma_limit` exceeds it at send validation with `ErrForbiddenParam`, so such a send never becomes an input (§4.6) and carries no receipt. `MaxInputGas` bounds the work a single input can compel from an honest executor, and `MaxBatchInputs × MaxInputGas` bounds a whole batch.
- `MaxInputGas` **MUST** be below the representable range of a signed 64-bit counter, so every conformant budget fits one. No clamping, truncation or saturating behaviour is defined, and an implementation **MUST NOT** introduce one: a runtime that observes a budget above `MaxInputGas` differs from the network rather than from the protocol and **MUST** halt rather than commit any receipt status.

> Submitting a `CallL2` costs L1 plasma to enter a momentum; executing the resulting input costs L2 gas, accounted off-chain and never paid for in tokens. The only token-denominated charges are the developer fees of §10.7. L1 cannot meter WASM without running it, which this spec forbids. Embedded addresses have unlimited L1 plasma (`vm/vm.go`); a Settlement send costs at least `21000 + 68/byte` (the rule above), so a full 16 KiB input (≈1.14M plasma) exceeds the PoW-only ceiling (`MaxPoWPlasmaForAccountBlock`, 94,500) and requires fused QSR, while a small call stays at its method's flat price — L2 inputs are feeless-but-rate-limited exactly like all L1 traffic. The same holds for the executor's `SubmitBatch`, whose commitment can approach 15 KiB, so an executor needs fused plasma.

### 10.2 Opcode gas table

**Normative:**

- L2 gas **MUST** be metered by a static table mapping each permitted Core 1.0 opcode to a fixed non-negative integer cost, fixed for a given `metering_version` (§24) and unchangeable without an increment (because a validity proof proves execution under this exact table).
- Memory growth **MUST** be charged per page on `memory.grow` in addition to the base instruction cost. Declared **initial** linear memory **MUST** be charged at the same per-page rate before execution begins; if the L2 budget cannot cover it, the input fails `OUT_OF_GAS` before the contract runs. (The per-page rate and `memory.grow` base are in `wasm-gas-table-v1.json`.)
- The concrete table for Phase 1 is `metering_version = 1`, defined in `wasm-gas-table-v1.json` / `.md`. Implementations **MUST** reproduce those costs exactly, using the basic-block metering model defined there with the `br_table` and `memory.grow` surcharges.
- **Function-entry locals charge.** A locally defined function's entry segment runs from the first instruction of its body through the first `unreachable`, `loop`, `if`, `else`, `end`, `br`, `br_if`, `br_table` or `return`, inclusive. It costs its instructions' table costs plus `ceil(L / locals_per_gas)`. `L` is the function's declared locals: the sum of the counts in its code entry's locals vector. Parameters are not counted. `locals_per_gas` is `function_entry.locals_per_gas` in `wasm-gas-table-v1.json`, and `ceil(L / k) = (L + k − 1) // k`, so a function that declares no locals pays nothing extra. The charge is part of the entry segment's single decrement, so it is paid once per invocation, before the function's first instruction runs, whether the function is entered by `call`, by `call_indirect` or as `run`. No branch re-enters an entry segment: a branch lands after an `end` or at a loop header, an `if` whose condition is false lands after its `else`, and a `loop` that is a body's first instruction ends the entry segment, so its back-edges land in the next segment. The §7.4 stack-depth check for the new frame precedes the charge. Host imports have no body and pay no entry charge. If the remaining budget cannot cover the entry segment, the input fails `OUT_OF_GAS` (§10.5).

### 10.3 Host-function gas

**Normative:** charged at invocation, before the host runs; if insufficient L2 gas remains, execution **MUST** halt with `OUT_OF_GAS` before the host executes. The one exception is the value-proportional part of `storage_read`: the runtime charges it after reading the key and before the buffer checks of §8.1, so when it is unaffordable, execution halts with `OUT_OF_GAS` and the key stays observed (§20). Values (`metering_version = 1`, reproduce exactly): `storage_len = 100`; `storage_read = 200 + 10 * ceil(value_bytes / 32)` (absent key charges base only); `abort = 1 + 10 * ceil(data_len / 32)`, with `data_len` read as unsigned and charged before §8.1's size check.

### 10.4 Post-execution effect charge

**Normative:** after the contract returns its blob and before `StateDiff` is applied, the runtime **MUST** charge gas for declared effects from serialized sizes (`metering_version = 1`, reproduce exactly): `StateDiff` entry `500 + 20 * ceil(value_bytes / 32)` (deletion charges base only); event `100 + 10 * ceil(data_bytes / 32)`; outbox message `200 + 5 * ceil(payload_bytes / 32)`; `ReturnData` `10 + 2 * ceil(return_bytes / 32)`. If remaining gas is insufficient, the input **MUST** fail `OUT_OF_GAS` with full rollback; no partial effects.

### 10.5 Out-of-gas rollback

**Normative:** on `OUT_OF_GAS`, all state changes from the affected input **MUST** be discarded; gas accounting **MUST** be deterministic across implementations.

### 10.6 Ordered post-execution validation and `gas_used` per status

`gasUsed` is committed in every receipt (§15, §27.7), so it is defined for every status and not only for `OUT_OF_GAS`. Two implementations differing on any row below commit a different `receiptRoot` for an identical batch.

**Normative — ordering.** After `run` returns, the runtime **MUST** perform these steps in this order, and **MUST** stop at the first that fails:

0. **Blob length.** `len(ResultBlob) > MaxResultBlob` ⇒ `RESULT_TOO_LARGE` (§14.3). This precedes the decode, so no length-proportional work runs on an over-long blob.
1. **Decode** the returned blob.
2. **Input-index claim**, then effect attribution: every `Event.contract_id` and `OutboxMessage.source_contract` equals the executing `contract_id` (§27.1a).
3. **Reserved-key and duplicate-derived-key scan**, in blob order.
4. **Claim bounds** on `claimed_deposit` (§18.5).
5. **Outbox capacity** for the deposit refund (§27.4).
5a. **Per-input withdrawal bound** (§27.9): the contract-emitted `L1_WITHDRAWAL` amounts, summed per asset, **MUST NOT** exceed `MaxBatchWithdrawal` for any asset (an asset with no `MaxBatchWithdrawal` configured for the domain has a cap of `0`, so any non-zero total in it fails, §27.9), and the distinct assets those withdrawals name, together with `deposit_asset` when `deposit_amount > 0`, **MUST NOT** number more than `MaxAssetsPerBatch`.
6. **The §10.4 effect charge.**
6a. **Storage quota** (§13.8): the `StateDiff`'s net state-byte growth **MUST** fit the contract's remaining quota.
7. **`ReturnData` size** against `MaxReturnData`.
8. **`ExecutionResult` size** against `MaxExecutionResult`.

The effect charge is step 6, so it precedes both size checks and follows every content check. The quota check is step 6a, after the effect charge, so an input that exceeds its quota has paid for the effects it declared. "Metered" below means the pre-execution initial-memory charge plus every injected and host charge actually incurred before the condition was detected.

**Normative — `gas_used` per status:**

| Condition | Status | `gas_used` |
|---|---|---|
| normal return, every step passes | `SUCCESS` | metered + effect charge |
| `abort` within §8.1's bounds, or `run` returns `r < 0` | `REVERT` | metered (including `abort`'s own charge); no effect charge |
| `abort` with `data_len > MaxReturnData` | `RESULT_TOO_LARGE` | metered (including `abort`'s own charge); no effect charge |
| an injected, host, initial-memory or effect charge is unaffordable | `OUT_OF_GAS` | the whole budget |
| any trap during `run` | `RUNTIME_FAULT` | metered to the trap |
| step 0 rejects an over-long blob | `RESULT_TOO_LARGE` | metered; **no** effect charge, since step 6 has not run |
| any failure in steps 1–5a | `RUNTIME_FAULT` | metered through `run` |
| a `StateDiff` entry writes a reserved `local_key` (step 3) | `UNAUTHORIZED` | metered through `run` |
| step 6a: net state growth exceeds the contract's remaining quota | `STORAGE_QUOTA_EXCEEDED` | metered + effect charge |
| step 7 or step 8 rejects on size | `RESULT_TOO_LARGE` | metered + effect charge |
| kinds 0/1: `target_contract` not deployed | `VALIDATION_FAILED` | `0` — nothing executed, no module, no initial-memory charge |
| kinds 4/5: any deploy-pipeline failure (§11.3) | `VALIDATION_FAILED` | `0` (§10.1: these kinds carry no budget) |
| kinds 5/6/7: `contract_id` names no deployed contract | `VALIDATION_FAILED` | `0` |
| kind 7: `purchased + bytes` exceeds `2^64 − 1` | `VALIDATION_FAILED` | `0` |
| kinds 5/6: the contract is `IMMUTABLE`, or the caller is not its owner | `UNAUTHORIZED` | `0` |
| kinds 2–7: success | `SUCCESS` | `0` |
| `input_kind` outside `0..7` | `UNSUPPORTED_OPERATION` | `0` |

Every non-`SUCCESS` status discards `StateDiff`, `Events`, `ReturnData` and every contract-emitted outbox message, keeping only the runtime-emitted deposit refund (§14.3). A `REVERT`'s `ReturnData` is its revert data, which no returned blob supplies.

### 10.7 Developer fees

Users pay no token fee: `CallL2` and `Deposit` cost L1 plasma only (§10.1). Developers pay, in ZNN, for the two things that cost executors lasting work: validating and instrumenting bytecode, and holding contract state.

**Normative:**

- **Bytecode fee.** `DeployContractFinalize` and `UpgradeContract` **MUST** carry, as attached ZNN, exactly `total_size × znnPerByteOfBytecode`, where `total_size` is the one recorded in the pending record the call consumes (§11.1).
- **Storage fee.** `BuyStorage(domainId, contract_id, bytes)` **MUST** carry, as attached ZNN, exactly `bytes × znnPerByteOfState`, and buys `bytes` of state quota for `contract_id` (§13.8). `bytes` **MUST** be at least 1.
- The price is the Periphery value in force at the receive height (§4.6). Until `SetPeripheryConfig` has set a price, no fee can match it, so Settlement **MUST** reject every fee-carrying call; a zero fee is never accepted. Settlement **MUST** reject at its receive a send whose attached token is not ZNN or whose attached amount differs from the fee. A rejected call is not an input (§4.6), and the standard embedded rollback returns the attachment.
- **Fees are not deposits.** A fee **MUST NOT** enter `totalDeposited` or any other conservation counter of §18.3, **MUST NOT** appear in `AssetFlowSummary`, and is never delivered to a contract. On acceptance Core **MUST** add the fee to `totalFeesCollected` and record it with the input's assigned-index record (§4.6, §5.4).
- **Fees are not refunded.** A fee is kept whatever the input's receipt: a pipeline `VALIDATION_FAILED` on a finalize or upgrade, or a `BuyStorage` naming an undeployed contract, still pays the fee. Executors processed the input either way.
- `znnPerByteOfBytecode` and `znnPerByteOfState` are Periphery values set through `SetPeripheryConfig`, each within a Core floor and ceiling (`IsZnnPerByteOfBytecodeInBounds`, `IsZnnPerByteOfStateInBounds`, §23, §25.2). Both floors **MUST** be non-zero.
- Fees are split and paid per epoch by §22.4: `FeeBurnShare` (30%) is burned, and the remainder goes to the pillar whose accepted batch carried the input.

> *Informative.* Charging at the L1 receive keeps fees out of L2 state. Contracts hold no protocol balance (§18.1a), so a fee charged during execution would have nothing to draw on. A developer covers storage by buying quota for their contract. A contract that wants its users to cover storage can require them to call `BuyStorage` for it, since anyone may buy quota for any contract.

---

## 11. Contract deployment

**Normative — scope.** Settlement's half of §11 and §12 — chunking, the deploy record, expiry, the fee, record consumption — applies to every contract runtime unchanged. The runtime/STF half is stated here for WASM: assembly, Core 1.0 validation and instrumentation (§11.3), `CodeHash` as the post-instrumentation hash (§11.4), and the reserved policy and owner leaves (§12). Another runtime's STF specification defines its own validation pipeline and leaf layout, and **MUST** keep: assembly by ascending chunk index checked against `code_hash` and `total_size`; `VALIDATION_FAILED` with no contract created on any pipeline failure; `ContractID` derived per §12; the upgrade-policy and ownership semantics of §12, including their receipt statuses; and the per-kind receipt fields of §15.

> A non-trivial module exceeds the 16 KiB account-block data limit (`MaxDataLength`, `vm/constants/plasma.go`). Deployment is therefore chunked, committed by hash, and assembled deterministically before validation and instrumentation. Content-addressing by the pre-instrumentation `code_hash` removes all executor discretion over the assembled result. Deploy records are keyed by `(domainId, deployer, code_hash)`, where the deployer is the L1 sender of `DeployContractStart`. Including the deployer means no one can occupy another deployer's slot: a party who learns a `code_hash` and starts a record for it first only creates a record of its own.

### 11.1 DeployContractStart

`DeployContractStart(domainId, code_hash, metadata_hash, total_size, chunk_count, metadata)` — **Normative:**

- `code_hash` **MUST** be `SHA3-256` of the complete pre-instrumentation bytecode (the deployer's binding commitment). `metadata` is the contract's canonical metadata encoding (§27.12), which declares its upgrade policy and owner (§12); `metadata_hash` **MUST** equal `SHA3-256(metadata)`, and `len(metadata)` **MUST NOT** exceed `MaxContractMetadataSize` (§25.1). Settlement **MUST** reject the call at its receive if either check fails. Settlement does not decode `metadata`: its structure is validated by the runtime at finalize (§11.3), and the bytes reach the runtime in the input's canonical encoding (§27.5).
- `total_size` **MUST** be the exact byte length, **MUST** be at least 1, and **MUST NOT** exceed `MaxCodeSize` (§25); `chunk_count` **MUST** equal `ceil(total_size / MaxDeployChunkData)` and **MUST NOT** exceed `MaxChunkCount`.
- On acceptance, Core **MUST** create a pending record keyed by `(domainId, caller, code_hash)` recording `metadata_hash`, `total_size`, `chunk_count`, deployer `caller`, start height (the receive height, §4.6), and a received-chunk count of `0`. The record is a fixed-size header and reserves no space for chunk hashes: each chunk writes its own entry when it arrives (§11.2), paid for by that chunk's plasma. Core **MUST** then append `(record key, startHeight)` to the start queue (§11.5). Before creating the record, Core **MUST** run the expiry sweep (§11.5). The runtime records `metadata` in its own pending record for the same key.
- If a non-expired, non-finalized record already exists for the same `(domainId, caller, code_hash)`, Settlement **MUST** reject the call at its receive. Different deployers may hold records for the same `code_hash` at the same time. A rejected call is not an input (§4.6): it consumes no index and carries no receipt.

### 11.2 DeployContractChunk

`DeployContractChunk(domainId, code_hash, chunk_index, chunk_hash, chunk_data)` — **Normative:** the record `(domainId, caller, code_hash)` **MUST** exist (non-finalized, non-expired), so a chunk can only ever address its own sender's record; `chunk_index ∈ [0, chunk_count)` and not already present; `chunk_data` ≤ `MaxDeployChunkData`, all chunks except the last exactly `MaxDeployChunkData`, last exactly `total_size - (chunk_count-1)*MaxDeployChunkData`; `chunk_hash` **MUST** equal `SHA3-256(chunk_data)`, verified by Settlement at the receive — the chunk bytes are in hand there, and this is Settlement's only byte-level verification of code. When `chunk_count == 1` the single chunk **is** the last chunk and its size **MUST** equal `total_size` (not `MaxDeployChunkData`). Settlement **MUST** reject the call at its receive if any of these checks fails; a rejected call is not an input (§4.6). On success, Core **MUST** store `chunk_hash` under its own key `(domainId, deployer, code_hash, startHeight, chunk_index)` and increment the record's received-chunk count. Including `startHeight` keeps a restarted deployment (§11.5) from ever reading an earlier attempt's chunk hashes. Chunk bytes are published in the data bundle as that input's `deployChunkData` (§20), and only the hash is committed to state.

> `MaxDeployChunkData` (§25) is sized below `MaxDataLength` so a full chunk **plus the method's ABI envelope** (selector, `domainId`, `code_hash`, `chunk_index`, `chunk_hash`, dynamic-bytes framing — ~260 bytes) always fits a single account-block's data field. `MaxDAChunkSize` governs only off-chain DA bundle chunk serving (§20) and plays no role in on-chain deployment.

### 11.3 DeployContractFinalize

`DeployContractFinalize(domainId, code_hash)` — **Normative:**

- **Settlement (on-chain):** the record `(domainId, caller, code_hash)` **MUST** exist; all chunks **MUST** be present; and the call **MUST** carry the bytecode fee computed from the record's `total_size` (§10.7). Settlement **MUST** reject the call at its receive if the record is absent, expired, already consumed or incomplete — a call from any other address finds no record under its own key; a rejected call is not an input (§4.6), so the runtime never receives a finalize for a record it cannot read. All chunks are present when the record's received-chunk count equals `chunk_count`. On acceptance the record is consumed (finalized): Core **MUST** delete the record header and all of its chunk-hash entries. Settlement performs **no** assembly, validation, or instrumentation, and never stores assembled bytecode; its per-chunk hash checks (§11.2) are its only byte-level code verification.
- **Runtime/STF (off-chain, deterministic):** bytecode **MUST** be assembled by concatenating chunks in ascending index; `SHA3-256(assembled)` **MUST** equal `code_hash` and `len` **MUST** equal `total_size`, else the input's receipt is `VALIDATION_FAILED` and no contract is created.
- Validation **MUST** then run on the assembled bytecode (runtime/STF): Core 1.0 structural validation, with every LEB128 integer in its shortest encoding; rejection of WASI/floats/threads/atomics/SIMD and post-MVP integer ops (sign-extension, `trunc_sat`) per §7.2; host-import restriction to exactly `storage_len`/`storage_read`/`abort` under the `"env"` module string (§8.1); size checks; required exports (§27.1).
- Validation **MUST** reject a module with a start section, since a start function runs at instantiation, outside `run` (§27.1). It **MUST** also reject a module that exports a mutable global, an export named `_start`, or an export whose name begins with `__zenon_`.
- Validation **MUST** reject any custom section, including the name section, at the point the module is split into sections and before any section is parsed. `CodeHash` is therefore a function of the module's semantics alone, and no decoder reads a section whose contents validation does not constrain.
- Validation **MUST** reject a module exceeding any structural cap of §25.1. These bound counts that are LEB128-encoded and so bounded by nothing else, and each drives allocation or table writes at every instantiation — that is, on every call to the contract, for the life of the contract.
- Validation **MUST** check, at deploy, that every active data segment's `offset + length` fits the module's declared initial memory, and that every active element segment's `offset + length` fits the declared table minimum. Core 1.0 defers both checks to instantiation; deferring them here would make a module deployable that fails on its first call, so a module violating either **MUST** yield `VALIDATION_FAILED` at deploy. Every module that receives a `CodeHash` is therefore instantiable, which is what makes an instantiation failure at call time a node-local fault rather than a receipt status.
- The runtime **MUST** validate the pending record's `metadata` against §27.12 — for `DeployContractFinalize` only; `UpgradeContract` keeps the contract's existing policy and owner (§12) and ignores the upgrade record's metadata. A metadata failure is a pipeline failure: `VALIDATION_FAILED`, no contract created.
- Instrumentation **MUST** then run deterministically (runtime/STF): gas-metering injection, stack-depth limiter, and a memory-growth limiter that caps memory at `min(declared maximum, MaxMemory)`, or at `MaxMemory` when the module declares no maximum (§7.4). The instrumented size **MUST NOT** exceed the post-instrumentation ceiling (§25).
- Any failure at any stage of this pipeline — assembly, hash or length mismatch, validation, instrumentation, or a failure to prepare the instrumented module for execution — **MUST** yield a `VALIDATION_FAILED` receipt with no contract created; the pipeline is a pure deterministic function of the assembled bytes, so every conformant node reaches the same outcome and a failure **MUST NOT** be treated as a node-local fault.
- The executable `CodeHash` **MUST** be `SHA3-256(instrumented_bytecode)`, derived by the runtime. `ContractID` **MUST** be derived per §12 using the deployer address, this input's `globalInputIndex`, and `chain_id`. The deployment `ExecutionResult` **MUST** include both `code_hash` and `CodeHash`, carried in `ReturnData` (§27.1a) as `code_hash(32B) || CodeHash(32B)`, since §27.1a's grammar has no dedicated field for them.

> *Informative.* The split follows from what each side can verify: Settlement holds chunk bytes only transiently (one receive at a time) and commits only their hashes, so the whole-assembly check runs in the runtime — where it is deterministic, reproducible by anyone from L1-anchored data, and provable under `ZK_VALIDITY`. A `code_hash` mismatch therefore consumes the record on-chain but deterministically creates no contract; the deployer restarts with a fresh record.

### 11.4 Two hashes

**Normative:** `code_hash` = `SHA3-256(pre-instrumentation)` (developer-visible identity, what chunked assembly verifies against); `CodeHash` = `SHA3-256(post-instrumentation)` (executable identity stored in state, provided as context, used by the STF). They **MUST NOT** be conflated; because instrumentation is deterministic, any verifier can reproduce `CodeHash` from the pre-instrumentation bytecode.

### 11.5 Expiry

**Normative:** a pending record **MUST** expire if not finalized within `DeploymentExpiryWindow` (§25) heights of `DeployContractStart`: it is expired at every receive height `h` with `h ≥ startHeight + DeploymentExpiryWindow`, both heights being receive heights (§4.6), so it is live for exactly `DeploymentExpiryWindow` heights; an expired record **MUST** be treated as nonexistent and its `(domainId, deployer, code_hash)` key freed; plasma spent on start/chunk **MUST NOT** be refunded on expiry.

**Expiry sweep (normative).** Expired records are removed from storage so that an unfinalized start leaves no permanent state. Every node **MUST** perform exactly the same deletions, because Settlement storage is part of `ChangesHash` (§5.1).

- Each accepted `DeployContractStart` appends `(record key, startHeight)` to a per-contract FIFO start queue. Starts are accepted in receive-height order, so the queue is also in expiry order.
- At each accepted `DeployContractStart`, before creating its own record, Core **MUST** examine at most `MaxDeploySweepPerStart = 2` entries from the queue head, stopping at the first one that is not expired at the current receive height. For each expired entry it **MUST**: delete every chunk-hash entry under `(domainId, deployer, code_hash, startHeight, ·)`; delete the record under `(domainId, deployer, code_hash)` only if that record still exists **and** its start height equals the entry's `startHeight`; and pop the entry.
- An entry whose record was already consumed leaves nothing to delete but its queue slot. A record restarted after expiry carries a different start height, so the sweep of its predecessor's entry never deletes it.
- Each start adds one entry and removes up to two expired ones, so storage holds at most the records started within the last `DeploymentExpiryWindow` heights plus a bounded backlog, and the sender of each new start pays for the cleanup.

The sweep never changes an acceptance outcome: an expired record is already treated as nonexistent, and a consumed one is already deleted. An executor's acceptance predicate (§4.6) needs no knowledge of it.

---

## 12. Contract identity and upgrades

**Normative:**

- `ContractID` **MUST** be `SHA3-256(deployer_address(20B) || deployment_global_input_index(8B BE) || chain_id)`, where `deployer_address` is the raw 20-byte L1 address that submitted `DeployContractFinalize` (§27.0) — unique (by `globalInputIndex` monotonicity) and stable across upgrades.
- `CodeHash` is the post-instrumentation hash (§11.4), stored in state and provided as context; on upgrade it changes, `ContractID` does not.
- A runtime's native form of `ContractID`, if it has one, follows §9.2.
- Every contract **MUST** declare one upgrade policy at deployment, in the `metadata` its `DeployContractStart` carries (§11.1, §27.12): `IMMUTABLE` (an `UpgradeContract` **MUST** yield `UNAUTHORIZED`); `OWNER_UPGRADEABLE` (code **MAY** be replaced by the designated owner). On a successful `DeployContractFinalize` the runtime **MUST** write the policy under the reserved upgrade-policy key as exactly one byte (§27.11), and, for `OWNER_UPGRADEABLE` only, the owner under the reserved owner key as the §9.2 caller form of the metadata's `owner` address, `SHA3-256("L1_CALLER:" || owner)` (§13.3). An `IMMUTABLE` contract has no owner leaf. Reserved future policies **MUST NOT** be activated in Phase 1.
- `OWNER_UPGRADEABLE` contracts **MUST** specify an owner; `UpgradeContract` **MUST** be authorized by the owner (verified via `caller`, §9.2) and **MUST** produce a new `CodeHash`. An `UpgradeContract` whose `contract_id` names no deployed contract **MUST** yield `VALIDATION_FAILED`, matching §18.2's undeployed-target rule. An upgrade replaces code only: the policy and owner leaves are unchanged.
- **Upgrade decision order.** The runtime **MUST** read the contract's `CodeHash`, pre-instrumentation `code_hash`, upgrade-policy and owner leaves unconditionally and in that order, so the witness set does not depend on the outcome (§20). It then decides in this order: no deployed contract ⇒ `VALIDATION_FAILED`; not authorized (an `IMMUTABLE` policy, or an owner leaf that does not equal `caller`) ⇒ `UNAUTHORIZED`, without running the §11.3 pipeline; a §11.3 pipeline failure ⇒ `VALIDATION_FAILED`. A replacement whose `CodeHash` equals the contract's current `CodeHash` does not produce a new `CodeHash`, so it is a pipeline failure: `VALIDATION_FAILED`, with no code change. The comparison is on the post-instrumentation `CodeHash`, because distinct source modules can instrument to the same one.
- **Ownership transfer.** `TransferContractOwnership(domainId, contract_id, new_owner)` (`input_kind = 6`, §27.5) is the only way to change an owner, and **MUST** be authorized by the current owner. Settlement performs no check at its receive beyond the envelope (§4.6): ownership lives in L2 state. The runtime **MUST** read the contract's `CodeHash`, upgrade-policy and owner leaves unconditionally and in that order, so the witness set does not depend on the outcome (§20). It **MUST** yield `VALIDATION_FAILED` when `contract_id` names no deployed contract, and `UNAUTHORIZED` when the policy is not `OWNER_UPGRADEABLE` or the owner leaf does not equal `caller`. Otherwise it **MUST** yield `SUCCESS` with a `StateDiff` that, for a non-zero `new_owner`, sets the owner leaf to `SHA3-256("L1_CALLER:" || new_owner)`; and, for `new_owner` equal to the zero address, sets the policy leaf to `IMMUTABLE` and deletes the owner leaf, renouncing ownership permanently. It emits no event and no outbox message, and carries `gasUsed = 0` (§10.1).
- **Upgrade staging and finalization.** Replacement bytecode **MUST** be staged through the chunked mechanism (§11.1–§11.2; the single-chunk flow covers small modules). `UpgradeContract(domainId, contract_id, new_code_hash)` is the **finalizer** of an upgrade-destined record: a complete, non-expired pending record for `(domainId, caller, new_code_hash)` **MUST** exist, and the call **MUST** carry the bytecode fee computed from that record's `total_size` (§10.7); Settlement verifies these, including a received-chunk count equal to `chunk_count`, and consumes the record — deleting its header and chunk-hash entries (the §11.3 on-chain half), and **MUST** reject the call at its receive when they do not hold, in which case it is not an input (§4.6). The runtime/STF runs the full §11.3 assembly/validation/instrumentation pipeline: on success the contract's code is replaced (`CodeHash` changes, `ContractID` does not); on any pipeline failure the receipt is `VALIDATION_FAILED` with no code change. `DeployContractFinalize` and `UpgradeContract` are mutually exclusive consumers of a pending record — whichever is accepted first consumes it; `DeployContractFinalize` deploys a **new** contract (new `ContractID`), `UpgradeContract` re-codes an **existing** one. Expiry (§11.5) applies to upgrade-destined records identically.
- An upgrade **MUST NOT** reset or modify existing contract state; the protocol **MUST NOT** enforce ABI/state-layout compatibility (developer's responsibility). The upgrade `ExecutionResult` **MUST** include old and new `CodeHash`, carried in `ReturnData` (§27.1a) as `oldCodeHash(32B) || newCodeHash(32B)`, since §27.1a's grammar has no dedicated field for them; the old `CodeHash` **MUST** stay available for at least the `EXECUTOR.md` §16 bundle-retention window: execution state (§13.1) retains only current leaves, so the upgrade input's bundle record is the copy that survives.

---

## 13. State model

### 13.1 Off-chain SMT, shared pure layer

**Normative:**

- Execution-layer state **MUST** be a Sparse Merkle Tree maintained off-chain by the executor, with a fixed 256-bit key space and non-inclusion proofs for absent keys.
- L1 **MUST** store only SMT roots (within batch commitments); it **MUST NOT** store SMT contents or traverse the SMT.
- The SMT **MUST** be byte-for-byte deterministic across compliant implementations, verified against the conformance suite (§13.7).

**Shared library — scope (normative):** the L2 executor's SMT and the L1 momentum state root **MUST** be byte-identical for identical leaf sets and **MUST** be produced by the same consensus-frozen hashing/routing/proof core in `common/trie`: `LeafHash`, `InternalHash`, the constant-zero empty hash, and MSB-first routing (`hash.go`); the full-depth root computation (`compute.go`); and the deepest-first compressed-sparse proof codec `encodeProof`/`decodeProof` plus root reconstruction (`proof.go`). These bytes are conformance-bound by `common/trie/conformance_test.go`, which loads `smt-v1-test-vectors.json` and governs **both** layers from a single vector set. The incremental `CompactTree` (`compact.go`, listed below) hashes with the same `hash.go` primitives and encodes with the same `proof.go` codec. It **MUST** reproduce `compute.go`'s roots and proofs byte for byte for the same leaf set, and `conformance_test.go` gates it against the same vectors.

**The L2 executor operates on 32-byte paths directly and MUST NOT route through any key-hashing entry point.** The L2 derived key (§13.3) is *already* a 32-byte `SHA3-256` output and is the SMT path directly; passing it to any function that internally applies `types.NewHash` would compute `sha3(path) ≠ path` and **double-hash**, producing a divergent tree. The following existing functions hash a key into a path and are **L1-only** — the L2 executor **MUST NOT** call any of them: `VerifyProof(root, key, …)`, `VerifyAbsence(root, key, …)`, `NodeTree.Prove(id, key)`, and the write-set methods `NodeTree.Update`, `NodeTree.AccumulateFrom` and `NodeTree.ComputeRoot`, which hash each key of their patch that `foldKeep` admits through `stagedApplier` and skip every other key.

**Path-native public API (normative).** `common/trie` exports the following path-native functions, which perform **no** key hashing; the L2 executor **MUST** use only these for root computation, proof generation, and proof verification. Each reproduces the exact bytes locked by the conformance suite and adds only the path-vs-key distinction:

- `RootOfLeaves(paths []Path, values [][]byte) (Hash, error)` — the full-depth root over 32-byte paths and raw values; a repeated path returns `ErrDuplicatePath`.
- `ProveByPath(paths []Path, values [][]byte, targetPath Path) (present bool, value []byte, proof []byte, err error)` — a proof for a target supplied as a path, never a key.
- `VerifyProofByPath(root Hash, path Path, value, proof []byte) (bool, error)` — identical to `VerifyProof` except the supplied `path` is compared to the decoded proof's path directly, with no `types.NewHash` call.
- `VerifyAbsenceByPath(root Hash, path Path, proof []byte) (bool, error)` — the absence analogue, likewise path-native.
- `NewCompactTree() *CompactTree` — an empty in-memory tree that keeps its internal hashes across writes, so a root or proof costs `O(log N)` in the leaf count rather than a full recomputation.
- `CompactTree.Insert(path Path, value []byte)` — sets the leaf at `path`, which is the position itself and is never hashed. A zero-length `value` stores a present-empty leaf (§13.6) and is never folded into a deletion.
- `CompactTree.Delete(path Path)` — removes the leaf at `path`; deleting an absent path is a no-op.
- `CompactTree.Root() Hash` — equal to `RootOfLeaves` over the tree's current leaf set.
- `CompactTree.ProveByPath(path Path) (present bool, value []byte, proof []byte)` — byte-identical to `ProveByPath` over the tree's current leaf set.

The L1 maintenance adapter (`nodestore.go`, with its fold rule in `momentum_fold.go`) and the L1-only verifiers `VerifyProof` and `VerifyAbsence` are the only `common/trie` code that hashes a key, and the adapter alone applies the empty-value fold rule (§13.5.1, caveat 1).

> **What is NOT shared (normative caveat).** The L1 *maintenance* adapter (`common/trie/nodestore.go`: `NodeTree.Update`/`Commit`/`Prove`; and `momentum_fold.go`: `stagedApplier`) is L1-momentum-specific and **MUST NOT** be reused by the L2 executor, for two reasons:
>
> 1. **Empty-value-as-deletion.** The L1 adapter collapses an empty value to a deletion (`stagedApplier.Put`: `if len(value) == 0 { del }`) — correct for L1 storage, which has no present-empty values. The L2 model **requires** a *present-empty* leaf distinct from an absent key (§13.6). The L2 executor **MUST** implement its own `StateDiff` application per the canonical algorithm in §13.5.1, honoring the `EMPTY` deletion sentinel (§27.2) and never folding empty→delete. The core itself already supports a present-empty leaf: `LeafHash(path, [])` is `sha3(path)`, which is non-zero and distinct from the empty (absent) slot. (This has been executed against the source: present-empty root ≠ absent root; `LeafHash(path,[])` is non-zero.)
> 2. **Key pre-hashing.** The L1 adapter derives the leaf position by hashing the supplied db key (`types.NewHash(key)`), because L1 keys are arbitrary-length. The L2 derived key (§13.3) is *already* a 32-byte SHA3 output and is the path directly. The L2 executor **MUST** pass the derived key as the path into the path-native API (`RootOfLeaves`, `ProveByPath`, `CompactTree`, `VerifyProofByPath`, `VerifyAbsenceByPath`) and **MUST NOT** route through `NodeTree.Update` or any key-hashing verifier/prover, or it will double-hash.

**Second implementation (normative).** The shared-library requirement of this section binds the Phase 1 executor. A Rust implementation of the executor-state SMT, such as the one in the validity-proof program mainnet requires (`../phase-2/`), is conformant only if it reproduces every root, leaf hash and proof byte of `smt-v1-test-vectors.json` (§13.7, §31).

### 13.2 Hash function

**Normative:**

- The SMT **MUST** use `SHA3-256` for leaves, interior nodes, and roots. Leaves are `SHA3-256(path || value)`; interior nodes `SHA3-256(left || right)`. Empty subtrees are the 32-byte zero hash at every level (the constant zero, not a per-level computed default), subject to the both-empty short-circuit (§13.4).
- Whether the SMT keeps `SHA3-256` or migrates to a ZK-friendly hash **MUST** be settled before `StateRootSpork` reaches mainnet (§5.1, `../phase-2/`), because §13.1 makes the L1 momentum state root and the L2 SMT one tree format; no ZK-friendly hash, field, width or parameter is locked in Phase 1. The SMT hash **MUST** be isolated behind a single interface so a migration touches one component.
- All non-SMT commitments (`eventRoot`, `outboxRoot`, `receiptRoot`, `inputRoot`, `DAHash`, `CodeHash`, `code_hash`, `ContractID`, `metadata_hash`, `stateDiffHash`, `returnHash`) **MUST** use `SHA3-256` in all phases and are not subject to that decision.

> This matches the shipped `common/trie`: `LeafHash(path, value) = sha3(path‖value)`, `InternalHash(left,right) = sha3(left‖right)` with `emptyHash = 32 zero bytes`, gated by `conformance_test.go` against `smt-v1-test-vectors.json`.

### 13.3 State-key derivation

**Normative:**

- State keys **MUST** be derived as `SHA3-256(contract_id || local_key)`, where `local_key` is contract-defined, at most `MaxStateKeySize` bytes (§25).
- The **runtime**, not the contract, **MUST** perform this derivation, always using the executing contract's `contract_id`. A contract therefore cannot address another contract's state. The keys a contract passes to `storage_len`/`storage_read` and emits in its `StateDiff` are `local_key`s; the runtime derives the SMT key before touching the tree.
- The derived 32-byte key is used directly as the SMT path (§13.1, caveat 2). Canonical `StateDiff` ordering (§27.2) is over derived keys.
- Protocol-reserved per-contract values — the executable `CodeHash` (§11.4), the pre-instrumentation `code_hash` that locates the contract's module, the upgrade policy, and the owner (§12) — **MUST** be stored under reserved `local_key` values contracts **MUST NOT** write. A `local_key` is reserved by **exact match** against that set and never by prefix, so a contract keying its own state by a fixed-width integer is not locked out of part of its own key space. A `StateDiff` entry writing a reserved `local_key` **MUST** yield `UNAUTHORIZED`. The concrete byte values are fixed per runtime. For the WASM runtime the reserved set is exactly the five two-byte keys `0x0001` (executable `CodeHash`), `0x0002` (pre-instrumentation `code_hash`), `0x0003` (upgrade policy), `0x0004` (owner) and `0x0005` (storage quota, §13.8). Once the activation spork has fixed this set, a further reserved key **MUST NOT** be added without a `statediff_version` increment, since it removes a key a contract was free to write.

### 13.4 Tree structure and proof shape

**Normative (these govern routing and proof bytes and match `common/trie` exactly):**

- The SMT **MUST** be an uncompressed full-depth binary tree of exactly 256 levels; every key occupies a leaf at depth 256 positioned by its 32-byte path. Implementations **MAY** use a compressed in-memory representation but the committed root and all proofs **MUST** equal the full depth-256 tree.
- Descent **MUST** be most-significant-bit-first: at depth `d`, the selecting bit is `(key[d/8] >> (7 - (d mod 8))) & 1`. Bit `0` routes left, `1` routes right. Interior nodes are `SHA3-256(left || right)` with `left` the bit-0 subtree.
- An interior node both of whose children are the 32-byte zero hash **MUST** itself be the zero hash and **MUST NOT** be `SHA3-256(zero||zero)`. With exactly one empty child, the empty side is the zero hash.
- Proofs **MUST** be full-depth over all 256 levels, deepest-first (leaf-to-root). A zero-hash sibling is a real sibling counted at its level. The canonical byte encoding — a 256-bit non-zero-sibling bitmap (deepest-first, MSB-first within each byte), `sib_count`, then only the non-zero siblings — is defined in `proof-format.md` and implemented by `encodeProof`/`decodeProof`; implementations **MUST** reproduce it byte-for-byte and **MUST NOT** truncate levels.

### 13.5 StateDiff application

**Normative:**

- A `StateDiff` is a canonical list of `(key, new_value)` sorted ascending by derived key, exactly one valid encoding (§27.2). Deletion uses the `EMPTY` sentinel, distinct from a present empty value.
- State changes **MUST** be staged and applied to the SMT only after the input succeeds and the post-execution effect charge is covered. On any failure (`REVERT`, `OUT_OF_GAS`, `RUNTIME_FAULT`, `RESULT_TOO_LARGE`, `STORAGE_QUOTA_EXCEEDED`, validation failure) no entries are applied.
- After applying a `StateDiff`, the resulting root **MUST** equal a fresh full recomputation over accumulated state (incremental == full recompute).

### 13.5.1 Canonical L2 StateDiff applier

**Normative:** the L2 executor **MUST** compute the post-state leaf set exactly as defined here. Given a prior leaf set `S` (a map of 32-byte derived keys → raw values) and a canonical `StateDiff` whose entries are sorted ascending by derived key with no duplicates (§27.2), the post-state leaf set `S'` is computed as:

1. `S' := S`.
2. For each entry `(key, new_value)` in `StateDiff` order:
   - If `new_value` carries the `EMPTY` sentinel length prefix `0xFFFFFFFF` (§27.2): **remove** `key` from `S'`. Deleting a `key` that is not in `S'` is a well-formed no-op that leaves the root unchanged.
   - Otherwise `new_value` has a length `L` in `[0x00000000, MaxStateValueSize]`: **set** `S'[key] = new_value`. This **includes** the present-empty case `L = 0x00000000`, which stores a zero-length value whose leaf hash is `LeafHash(key, [])` = `SHA3-256(key)` — distinct from absence (§13.6). The applier **MUST NOT** fold a zero-length present value into a deletion.
3. `postStateRoot := RootOfLeaves(S')` using the path-native API (§13.1).

The empty→delete fold of the L1 adapter (`stagedApplier` in `momentum_fold.go`, which the `NodeTree` write-set methods use) is L1-only and **MUST NOT** be used here; the only deletion trigger at L2 is the explicit `EMPTY` sentinel. The post-state root produced by this incremental applier **MUST** equal a fresh `RootOfLeaves` recomputation over `S'` (the incremental == full-recompute invariant, §13.5, §13.7 SMT-014). On any input failure (`REVERT`, `OUT_OF_GAS`, `RUNTIME_FAULT`, `RESULT_TOO_LARGE`, `STORAGE_QUOTA_EXCEEDED`, validation failure) the applier **MUST** leave state unchanged (`S' := S`, no entries applied).

> *Informative.* This algorithm is the consensus-critical bridge between §27.2's wire-level sentinel discrimination and §13's tree. The two distinctions it locks — `0xFFFFFFFF` ⇒ delete vs. `0x00000000` ⇒ present-empty, and "delete-absent is a no-op" — are exactly the points two independent implementers would otherwise resolve differently, silently changing the post-state root. A validity proof under `ZK_VALIDITY` proves this identical applier.

### 13.6 Absent versus present-empty

**Normative:** the runtime **MUST** distinguish three observable conditions (see §27.8 for exact return semantics):

- **Absent** (non-inclusion): `storage_len → -1`; `storage_read → -1`, writes nothing.
- **Present-empty** (inclusion, zero-length value): `storage_len → 0`; `storage_read → 0`, writes zero bytes.
- **Present non-empty** (length `L>0`): `storage_len → L`; `storage_read → L` if buffer ≥ L (writes L), else `-2`.

A key deleted via the `EMPTY` sentinel is **absent** (`-1`), not present-empty. Every key observed (including absent observations) **MUST** be recorded in the input's witness set (§20).

> Conflating absent and present-empty would make a non-inclusion proof indistinguishable from an empty-value inclusion proof, letting a dishonest executor swap one for the other undetectably. This is a correctness requirement. (It is also exactly why the L2 executor cannot reuse the L1 adapter's empty==delete fold rule (`stagedApplier`) — §13.1.)

### 13.7 Conformance suite

**Normative:** the canonical SMT vector suite (`smt-v1-test-vectors.json` + `proof-format.md`) is a normative companion and **MUST** exist before Phase 1 implementation is considered complete. It **MUST** cover at minimum: empty-tree root; single-key insertion across the key space; long-shared-prefix keys; deletion of an existing key; deletion of a non-existent key (no-op); update to a new value; update to the current value (root unchanged); inclusion proofs; non-inclusion proofs through populated subtrees; present-empty inclusion (root distinct from absence); and batch `StateDiff` application equal to full recompute. Each vector specifies expected roots and proof bytes; an implementation **MUST** reproduce every value byte-for-byte. (The L1 pure layer is gated on this suite via `conformance_test.go`.)

The suite **MUST** additionally lock the L2-specific behaviors that the L1 layer does not exercise:

- **Path-native API equivalence (CV-PATH-1/2):** `VerifyProofByPath`/`VerifyAbsenceByPath`/`RootOfLeaves`/`ProveByPath` reproduce the exact roots and proof bytes of the SMT vectors when the SMT key is supplied as a **path** (no key hashing); the key-hashing `VerifyProof(root, path-as-key, …)` rejects the same path.
- **L2 applier (CV-APPLY-1/2):** a `StateDiff` mixing a present-empty write (`L=0x00000000`) and an `EMPTY` delete (`L=0xFFFFFFFF`) with expected post-root; and the incremental==full-recompute invariant of §13.5.1 applied step-by-step.

### 13.8 Storage quota

A contract may hold only as much state as has been bought for it with `BuyStorage` (§10.7). The quota is L2 state, kept by the runtime, so it is covered by the state root and reproducible from L1-anchored data like any other leaf.

**Normative:**

- **Quota leaf.** The runtime **MUST** keep each contract's quota under the reserved `local_key` `0x0005` (§13.3) as exactly 16 bytes, `purchased:u64 || used:u64`, big-endian. An absent leaf means `purchased = used = 0`. A newly deployed contract has no quota leaf.
- **Counted bytes.** A contract's `used` value is the sum, over its present leaves whose `local_key` is not reserved, of `StateLeafOverhead + len(value)` (§25.1). A present-empty leaf counts `StateLeafOverhead`. Reserved leaves are not counted, so deploy, upgrade, ownership transfer and the quota leaf itself need no quota.
- **Charging a write.** For a `CallL2` or `Deposit` that reaches §10.6 step 6a, the runtime **MUST** compute `delta` as the sum, over the contract's `StateDiff` entries, of `size(new value) − size(pre-state value)`, where `size` of an absent or deleted key is `0` and `size` of a present value is `StateLeafOverhead + len(value)`. Every pre-state value it reads is already an observed key of the input (§20). If `used + delta > purchased`, the input **MUST** fail `STORAGE_QUOTA_EXCEEDED` with full rollback and a full deposit refund (§18.5).
- **Reading the quota leaf.** The runtime **MUST** read the quota leaf at step 6a when, and only when, `delta ≠ 0`. When `delta = 0` the check cannot fail and nothing is recorded, so the quota leaf is not an observed key of the input (§20) and does not enter its witness set.
- **Recording a write.** When the check passes and `delta ≠ 0`, the runtime **MUST** add to the input's committed `StateDiff`, at its canonical position (§27.2), an entry setting the quota leaf to `purchased || used + delta`. `delta` may be negative: deleting or shrinking state frees quota for reuse. No ZNN is ever refunded.
- **`BuyStorage` (`input_kind = 7`).** The runtime **MUST** read the contract's `CodeHash` leaf and then its quota leaf, unconditionally and in that order, so the witness set does not depend on the outcome (§20). It **MUST** yield `VALIDATION_FAILED` when `contract_id` names no deployed contract, or when `purchased + bytes` exceeds `2^64 − 1`. Otherwise it **MUST** yield `SUCCESS` with a `StateDiff` setting the quota leaf to `purchased + bytes || used`. It emits no event and no outbox message, and carries `gasUsed = 0` (§10.1). The fee is kept on every outcome (§10.7).

> *Informative.* Quota is a stock of bytes, not a running charge: once bought it stays with the contract, and freed bytes can be reused. That keeps the rule deterministic and cheap to evaluate — one subtraction per `StateDiff` entry — and it never needs a token balance inside L2.

---

## 14. ExecutionResult and ContractEffects

### 14.1 Canonical binary serialization

**Normative:** `ExecutionResult` and all sub-structures **MUST** use a custom canonical binary format: integers fixed-width big-endian unsigned (`u8`/`u16`/`u32`/`u64`/`u256`); hashes/IDs/addresses/asset-ids fixed 32-byte (L1 `Address` is 20 bytes, used only in `caller` preimages, `ContractID` preimages (§12), withdrawal recipients, `executorId`, the `new_owner` field of §27.5 and the `owner` field of §27.12); fields in the exact order specified; `Bytes` = `u32` length prefix + raw bytes; lists = `u32` count + elements (sorted where required); exactly one valid encoding. JSON, protobuf, and SSZ **MUST NOT** be used. Full wire formats in §27.

### 14.2 Two objects

**Normative:** execution produces two related objects sharing the same field encodings:

- The contract returns a **`ContractEffects`** blob from `run` (§27.1; byte grammar §27.1a) — it cannot compute state roots. In order: `executionresult_version`, `globalInputIndex`, `StateDiff`, `Events`, `OutboxMessages`, `ReturnData`, `claimed_deposit`.
- The runtime assembles the canonical **`ExecutionResult`** (byte grammar §27.1a) by inserting `preStateRoot`, `postStateRoot` immediately after `globalInputIndex`: `executionresult_version`, `globalInputIndex`, `preStateRoot`, `postStateRoot`, `StateDiff`, `Events`, `OutboxMessages`, `ReturnData`, `claimed_deposit`.

`claimed_deposit` is a `u256` (§18.5): the amount of the input's `deposit_amount` the contract accepts into its own accounting. It **MUST** be `0` when the input carries no deposit, and **MUST NOT** exceed `deposit_amount` (else `RUNTIME_FAULT`). It is the only contract-returned field that is not a state/event/message effect; it is an instruction to the runtime's deposit-settlement step (§18.5).

The contract **MUST NOT** emit `preStateRoot`/`postStateRoot`; a `ContractEffects` blob that includes them **MUST** cause `RUNTIME_FAULT`. The runtime **MUST** compute `postStateRoot` itself and **MUST NOT** trust any root from the contract.

### 14.3 Failure semantics and size limits

**Normative:** on failure, `StateDiff`/`Events`/`OutboxMessages`/`ReturnData` **MUST** be discarded (empty) and `claimed_deposit` treated as `0`, with one exception: on `REVERT`, `ReturnData` is the revert data (§15), supplied by the runtime's abort path rather than by a returned blob. `ReturnData` is committed through `returnHash` (§15, §27.7), so an implementation retaining it on a failure path that holds a decoded blob (`RESULT_TOO_LARGE`, `UNAUTHORIZED`, a post-decode `RUNTIME_FAULT`) would commit a different `receiptRoot` for an identical batch. (The runtime-emitted deposit-refund outbox of §18.5 is appended independently of contract effects and is **not** subject to this discard rule.) The returned blob **MUST NOT** exceed `MaxResultBlob` (§25.1). The runtime **MUST** check its length **before** decoding it and **MUST** yield `RESULT_TOO_LARGE` with full rollback when the bound is exceeded, so no decode, no per-entry key derivation and no other length-proportional work is performed on an over-long blob. This check precedes every other check on the blob's contents; an over-long blob therefore carries `RESULT_TOO_LARGE` whether or not it is also malformed.

`STORAGE_QUOTA_EXCEEDED` (§13.8) likewise discards all four and keeps only the deposit refund.

A malformed returned blob within the bound (length-prefix inconsistency, trailing bytes, out-of-order fields, non-canonical list order) **MUST** cause `RUNTIME_FAULT` with full rollback; the runtime **MUST NOT** repair it. `ReturnData` **MUST NOT** exceed `MaxReturnData` and the total serialized `ExecutionResult` **MUST NOT** exceed `MaxExecutionResult`; violating **either** bound **MUST** yield `RESULT_TOO_LARGE` with full rollback (§25).

---

## 15. Receipts

**Normative:** each input **MUST** produce exactly one receipt (canonical encoding §27.7) containing, in order: `inputIndex`; `inputHash` (SHA3-256 of the canonical input encoding, §27.5); `batchId`; `contractId`; `codeHash`; `status`; `gasUsed`; `abortCode`; `stateDiffHash` (or zero if empty); `returnHash` (or zero); `eventRoot` (or zero); `eventCount`; `outboxRoot` (or zero); `outboxCount`.

**`contractId`/`codeHash` per input kind.** `contractId` is the contract the receipt concerns and `codeHash` is that contract's code identity as it stands when the receipt is assembled; both are zero when there is neither:

- `CallL2`/`Deposit` (kinds 0/1): `contractId` **MUST** be the input's `target_contract` on every outcome; `codeHash` **MUST** be that contract's current `CodeHash`, or zero if `target_contract` is not deployed.
- `DeployContractStart`/`DeployContractChunk` (kinds 2/3): both **MUST** be zero.
- `DeployContractFinalize` (kind 4): on `SUCCESS`, both **MUST** be the newly derived `ContractID`/`CodeHash` (§12); on `VALIDATION_FAILED`, both **MUST** be zero, since no contract is created (§11.3).
- `UpgradeContract` (kind 5): `contractId` **MUST** be the input's `contract_id` on every outcome; `codeHash` **MUST** be the new `CodeHash` on `SUCCESS`, the contract's unchanged current `CodeHash` on `UNAUTHORIZED` or a pipeline `VALIDATION_FAILED` against a deployed contract, and zero when `contract_id` names no deployed contract.
- `TransferContractOwnership` (kind 6): `contractId` **MUST** be the input's `contract_id` on every outcome; `codeHash` **MUST** be the contract's current `CodeHash`, or zero when `contract_id` names no deployed contract.
- `BuyStorage` (kind 7): the same rule as kind 6.

**`stateDiffHash`/`returnHash` preimages.** Both are `SHA3-256` (§13.2) over the corresponding committed value as it stands after §14.3, and `bytes32(0)` when that value is empty.

- `stateDiffHash` **MUST** be `SHA3-256` of the canonical `List<StateDiffEntry>` encoding of the `ExecutionResult`'s `StateDiff` (§27.1a, §27.2): §27.0's `u32` count prefix, then every entry in ascending derived-key order, each entry `key:bytes32 || new_value:Bytes` carrying §27.2's `EMPTY` sentinel (`L = 0xFFFFFFFF`, no bytes) for a deletion and `L = 0x00000000` for a present-empty value. It **MUST** be `bytes32(0)` when the list is empty. The local-key form of §27.2 **MUST NOT** be hashed: it is explicitly unordered, which would make a committed field depend on the contract's own emission order.
- `returnHash` **MUST** be `SHA3-256(ReturnData)` over the raw return bytes, with no length prefix and no framing. It **MUST** be `bytes32(0)` when `ReturnData` is empty.

Both rules apply to every receipt of every input kind. A non-`SUCCESS` status discards both values (§14.3), so every non-`SUCCESS` receipt **MUST** carry `stateDiffHash = bytes32(0)`, and `returnHash = bytes32(0)` except on `REVERT`, where `returnHash` is `SHA3-256` of the revert data, or `bytes32(0)` when it is empty.

**`abortCode` and revert data.** Every runtime reports a contract's refusal the same way: status `REVERT`, an `i32` `abortCode`, and revert data of at most `MaxReturnData` bytes, carried as the `ExecutionResult`'s `ReturnData` and committed through `returnHash`. Each runtime defines how a contract supplies the pair (§7.5 for WASM); their meaning is the contract's, and clients decode them through the contract's interface. Every receipt whose status is not `REVERT` **MUST** carry `abortCode = 0`. The code is committed beside the data so a refusal can be classified without decoding the data.

**Status codes (exhaustive; ordinals fixed, §27.7):** `SUCCESS=0`, `REVERT=1`, `OUT_OF_GAS=2`, `RUNTIME_FAULT=3`, `VALIDATION_FAILED=4`, `RESULT_TOO_LARGE=5`, `UNAUTHORIZED=6`, `UNSUPPORTED_OPERATION=7`, `STORAGE_QUOTA_EXCEEDED=8`. Once the activation spork has fixed this list, additional codes **MUST NOT** be introduced without a `receipt_version` increment. `UNSUPPORTED_OPERATION` is the status of an input whose `input_kind` is outside the eight defined in §27.5. Such a receipt **MUST** carry `gasUsed = 0`, zero `contractId` and `codeHash`, and no outbox message — no deposit refund is possible, because `deposit_asset` and `deposit_amount` live in the kind-specific tail the kind does not define (§27.5).

**Failed-input outbox invariant:** a non-`SUCCESS` receipt has `outboxCount = 0` and `outboxRoot = bytes32(0)` **except** for a single runtime-emitted deposit refund (§18.5, §27.4). When a deposit refund is emitted on a failed input, the receipt **MUST** carry `outboxCount = 1` and `outboxRoot` computed over that one message. This is the only outbox content permitted on a non-`SUCCESS` receipt; all contract-emitted outbox messages are discarded on failure (§14.3). The refund message **MUST** appear in the input's bundle entry and witness set on a par with any success-path outbox message.

The batch `receiptRoot` **MUST** be a binary Merkle root (§27.3 construction) over all receipts ordered by `globalInputIndex`, and **MUST** be in the batch commitment (§19).

---

## 16. Events

**Normative:**

- Events are execution metadata, **not** state: they **MUST NOT** influence the state root and **MUST NOT** be readable by contracts.
- Each event contains `contractId`, up to `MaxTopicsPerEvent` 32-byte topics, a `data` field ≤ `MaxEventDataSize`, the enclosing `inputIndex`, and a monotonic `eventIndex` (§27.3).
- Events from one input **MUST** appear in emission order; per-input count ≤ `MaxEventsPerExecution`. The per-input `eventRoot` is a Merkle root over the input's events; the batch `eventRoot` is the Merkle root over all batch events ordered by `(inputIndex, eventIndex)`, included in the batch commitment. Events are included on success, discarded on failure.
- Wallets/indexers **MUST** be able to build event inclusion proofs against the committed `eventRoot` from the data bundle.

---

## 17. Outbox and messaging

### 17.1 Asynchronous only

**Normative:** Phase 1 **MUST NOT** provide synchronous cross-contract calls; a contract **MUST NOT** invoke another contract within its own execution. Cross-contract interaction **MUST** be expressed as asynchronous outbox messages processed in a later batch. There is no host function that executes another contract (§8).

> Deferred, not banned. A future versioned upgrade **MAY** introduce synchronous calls only if they preserve the single-input transition invariant `postStateRoot = F(preStateRoot, input_data, code_hash)`, introduce no nondeterminism, and do not disturb canonical ordering.

### 17.2 Format and commitment

**Normative:** an outbox message (encoding §27.4) contains source `contractId`, `inputIndex`, monotonic `outboxIndex`, `kind` (0 = L2 / cross-domain / cross-chain **message** delivery, 1 = L1 / foreign **withdrawal (peg-out) authorization**), the kind-specific target (target `ContractID`, or recipient address + `AssetID` + amount, where "recipient" is an L1 address for a Zenon-native withdrawal or a foreign-chain address for a bridge peg-out, §6.6), and a `payload` ≤ `MaxOutboxPayloadSize`. The per-input `outboxRoot` and the batch `outboxRoot` (over all messages ordered by `(inputIndex, outboxIndex)`) are Merkle roots included in the batch commitment.

A `kind=1` message is the single authorization object for *every* withdrawal — a Zenon-native ZTS release (consumed on Zenon by `ReleaseWithdrawals`, §18.2) and a bridge peg-out (consumed on the foreign chain by a `ReleaseAdapter`, §6.6) are the same object with different consumers; there is no per-chain withdrawal path on Zenon. **`MESSAGING` domains MUST emit only `kind=0`; `BRIDGE` domains MAY emit both; `EXECUTION` domains emit `kind=0` for cross-contract/cross-domain delivery and `kind=1` for native ZTS withdrawals (§6.5).**

### 17.3 Replay protection and RelayMessage

**Normative:**

- Each outbox message **MUST** have a globally unique `outboxId = SHA3-256(domainId || batchId || inputIndex || outboxIndex)`. Settlement **MUST** maintain a `processedOutbox` set; a `RelayMessage` whose `outboxId` is already present **MUST** be rejected. Messages **MUST** be processed at most once.
- `RelayMessage` is permissionless and **registers** a message; it releases no funds itself (§18.2, §27.9). It **MUST** supply the message and an inclusion proof (`MerkleProof`, §27.3) against the committed `outboxRoot` of a `FINALIZED` batch (§21); Settlement **MUST** verify the proof and `FINALIZED` status before acting and **MUST** record the `outboxId` at registration — a message is registered at most once. **Settlement MUST additionally verify that the `domainId` and `batchId` supplied with the relay message match the domain and batch whose committed `outboxRoot` is used to verify the inclusion proof**; a relay presenting a valid proof against a mismatched domain or batch cursor **MUST** be rejected.
- For L1 withdrawals, registration enqueues the release (§18.2); release is performed by `ReleaseWithdrawals` under the conservation rules (§18.3), bounded per call (§27.9).
- `RelayMessage` **MUST** reject, and record nothing for, three kinds of message, so their `outboxId` stays unregistered:
  - a `kind = 0` message, because Phase 1 defines no L2 delivery path;
  - a `kind = 1` message whose `amount` is `0`;
  - a `kind = 1` message whose `recipient_l1` is an embedded-contract address (`types.IsEmbeddedAddress`). A descendant send cannot target one, so its release would fail and roll back the whole `ReleaseWithdrawals` call, blocking the domain's queue.

  The runtime does not stop a contract emitting these messages; they are inert in Phase 1. The value a rejected withdrawal names stays in custody, where the conservation invariant (§18.3) still counts it.

> Inter-batch outbox ordering (a total order across batches for L2-to-L2 delivery) is **Deferred**; replay protection and at-most-once processing are sufficient for Phase 1.

### 17.4 Timeouts and acknowledgements (reserved)

**Normative (reserved, gated):** Phase 1 outbox is fire-and-forget with replay protection only. A generalized messaging profile — used by `MESSAGING` domains and optionally by `BRIDGE`/`EXECUTION` cross-chain delivery — **MAY** attach a `timeout` (a destination height/time or source height deadline) and an `ackPolicy` (`NONE` / `ON_DELIVERY` / `ON_EXECUTION`) to a message, with a `MessageState` per `outboxId` (§5.4). When present: the destination **MUST** refuse delivery past the timeout, and the source becomes eligible for a **refund/rollback** on a proof of non-delivery by the deadline; the timeout path and delivery **MUST** be mutually exclusive (the `outboxId` is consumed by whichever lands first); an ack is itself a `kind=0` message replay-protected by its own `outboxId`. Timeout and ack windows are pre-mainnet per channel and **MUST** be ordered against source and destination finality so no refund races a valid delivery (§21). None of this is activated while the Phase-1 profile is in force. Lifecycle walkthrough in `../research/bridges.md` §5.3–§5.4.

**Proof of non-delivery (reserved, gated).** A timeout is proved against the destination's processed set, the same set that makes delivery at-most-once:

- **Zenon destination (another domain).** Settlement holds the destination's processed set (`processedOutbox`, §17.3), so no proof is needed. `TimeoutMessage(domainId, batchId, message, inclusionProof)` **MUST** verify the message's inclusion in a `FINALIZED` batch, that the current momentum height is at or past its source-height deadline, and that its `outboxId` is not in `processedOutbox`. On acceptance Settlement **MUST** add the `outboxId` to `processedOutbox` marked timed out, so a later delivery is rejected, and **MUST** deliver the timed-out message back to the source contract as an input so the source domain's STF performs the refund or rollback.
- **Foreign destination.** The `ReleaseAdapter` **MUST** keep its processed set as an authenticated map keyed by `outboxId` whose root is committed in foreign-chain state, and **MUST** refuse delivery past the deadline. Non-delivery is proved by relaying, as an `L1_RELAYED` input, a non-membership proof of the `outboxId` against that root at a finalized foreign block whose height or time is at or past the deadline; the source domain's `ChainVerifier` verifies it inside the STF, which performs the refund. Because the adapter refuses late delivery, absence at such a block is permanent, so delivery and timeout are mutually exclusive.

---

## 18. Asset model and conservation

### 18.1 Custody modes and asset identity

**Normative:** each `(domain, asset)` has a **custody mode** (§6.5):

- **`ESCROW`** — a real ZTS asset held 1:1 in Settlement custody. **The only mode activated in Phase 1.** Aggregate solvency is enforced on-chain independently of executor honesty (§18.3).
- **`MINTED`** (reserved, gated) — a representation asset (e.g. `zBTC`) whose supply is minted against attested foreign-chain locks and burned to authorize foreign release; it has no L1 deposit event behind it, so its conservation is enforced differently and its activation is gated on the §18.3 profile floor and the §18.4a reconciliation.

**Phase 1 therefore MUST NOT issue `MINTED` representation assets; all Phase-1 execution-layer assets MUST be `ESCROW`, backed 1:1 by L1 assets in Settlement custody.** An `AssetID` **MUST** be the canonical 32-byte encoding of a 10-byte ZTS identifier: bytes `[0:22]` (the leading 22 bytes) are zero, and bytes `[22:32]` are the ZTS identifier in its native byte order. This map is bijective and is the **only** valid `AssetID` encoding; it **MUST NOT** be a hash of the ZTS and **MUST NOT** use any alternative alignment (e.g. the ZTS bytes at `[0:10]`). Settlement **MUST** reject any `AssetID` with a non-zero byte in `[0:22]` (a `SubmitBatch`, `RelayMessage` or `SetAssetConfig` naming one is rejected at send validation) and **MUST** reject any non-canonical alignment. All conservation counters (§18.3) and `AssetFlowSummary` entries (§18.4, §27.6) are keyed by this canonical `AssetID`, so the single encoding guarantees one custody pot per real asset.

### 18.1a Balances are contract-managed (no protocol ledger)

**Normative:**

- Settlement custody is a single aggregate pot per (domain, asset). Settlement/Core **MUST** track only aggregate custody counters, such as `totalDeposited`, `pendingWithdrawalReserve` and `totalReleased` (§18.3); it **MUST NOT** maintain a per-account balance ledger of custodied assets.
- **Per-account balances are contract state.** Each contract records who owns what in its own SMT state (§13). There is **no protocol-level spendable balance** and **no balance or transfer host function** (the host ABI is exactly the three imports of §8). Value is moved between accounts only by contract logic writing its own `StateDiff`.
- Value enters L2 only by being **delivered to a contract** that credits it: a `Deposit` (value, no application call — §18.2) or a payable `CallL2` (value + call — §18.5). A withdrawal leaves L2 only by a contract emitting an L1-withdrawal outbox message (§17); the outbox carries the L1 recipient/asset/amount and has **no protocol source-account field**, because the emitting contract is the source of truth for whose balance it debited.

> This is the only model consistent with both the three-import host ABI and the outbox withdrawal format: per-account accounting lives in contracts, Settlement bounds only the aggregate. It is also what the trust model already states — aggregate solvency is enforced on-chain; per-account correctness relies on executor honesty until `ZK_VALIDITY` (§1, §18.3). The **reference vault contract** is the minimal example of this mechanism: a payable deposit credits `caller`, and a withdraw call debits it and emits the outbox. It is example code to build on and the testnet demo contract; it has no protocol role and no planned mainnet deployment.

### 18.2 Deposits and withdrawals

**Normative:**

- A `Deposit` is a user send of a ZTS asset to Settlement carrying `{domainId, target_contract}` (§27.5) — value with no application call. Settlement **MUST** reject a `Deposit` carrying no value at send validation (`ErrInvalidTokenOrAmount`): with nothing attached it would only duplicate a `CallL2` with an empty payload. On the embedded receive, Core **MUST** increase `totalDeposited[domainId][asset]` (final on L1, so safe). The deposit is a canonical input that the runtime delivers to `target_contract` by running it with an empty application payload and the deposit in the input frame (`deposit_asset`/`deposit_amount`, §27.1); the contract accepts it via `claimed_deposit` and credits the depositor (`caller`) — or its own treasury — in its own state, and any unclaimed amount is refunded automatically (§18.5). Token burn/mint and descendant `ContractSend` mechanics already exist (`vm/vm.go`). If `target_contract` resolves to no deployed contract (no code at that `ContractID`), the input **MUST** fail `VALIDATION_FAILED`, and the full `deposit_amount` **MUST** be auto-refunded to the depositor per the §18.5 failure path; a deposit therefore can never strand against a non-existent or undeployed target. The same rule applies to a payable `CallL2` whose `target_contract` is undeployed. For a `CallL2` or a `Deposit`, the runtime **MUST** read the target's executable `CodeHash` leaf (`0x0001`) and then its pre-instrumentation `code_hash` leaf (`0x0002`, §13.3), unconditionally and before deciding whether the target is deployed or looking up its code in the code store. An undeployed target's witness set (§20) is therefore exactly those two absent observations.
- A withdrawal is initiated by a contract emitting an L1-withdrawal outbox message (§17); the contract **MUST** debit the withdrawing account in its own state at emission (double-spend prohibited at the execution layer — the contract owns the per-account ledger, §18.1a).
- On `RelayMessage` of a withdrawal from a `FINALIZED` batch, Core **MUST** increase `pendingWithdrawalReserve[domainId][asset]` and append the withdrawal to the domain's release queue. `FINALIZED` status is the **only** time gate: registration and release add no further delay beyond §21. Release is performed by the permissionless, rate-limited **`ReleaseWithdrawals(domainId)`** method (§5.3): each accepted `ReleaseWithdrawals` **MUST** release queued withdrawals in registration order, at most `MaxWithdrawalsPerUpdate` per call (§27.9), and for each release **MUST** move the amount into `totalReleased[domainId][asset]`, decrease `pendingWithdrawalReserve`, transfer the asset via a descendant send, and re-check the conservation invariant. `ReleaseWithdrawals` follows the established embedded crank pattern (precedent `UpdateEmbeddedAcceleratorMethod`): permissionless, and rejected when called again for the same domain within `ReleaseIntervalMomentums` (§25.1). The interval is one momentum, so a domain releases up to `MaxWithdrawalsPerUpdate` withdrawals per momentum, and a batch's worst case of `MaxBatchInputs × (MaxOutboxPerExecution + 1)` withdrawals, deposit refunds included (§27.4), clears in `MaxBatchInputs × (MaxOutboxPerExecution + 1) / MaxWithdrawalsPerUpdate` momentums (132). A longer interval would let one batch of dust withdrawals delay every later exit on the domain for days. Its rate limit **MUST** be kept per domain, in its own record keyed by `domainId`: the shared `checkAndPerformUpdate` record (`vm/embedded/implementation/common.go`) is one value for the whole contract and belongs to the fee crank `Update` (§22.4). A `ReleaseWithdrawals` on an empty queue **MUST** fail with `ErrNothingToWithdraw` and **MUST NOT** update the rate-limit record, so an empty call never delays the next release. A domain with no rate-limit record has made no release, and its first call passes the rate limit.

### 18.3 Conservation invariant

**Custody mode `ESCROW` (Phase 1).** For every `ESCROW` (domain, asset), after every state-modifying Settlement operation the following **MUST** hold, and Core **MUST** revert any operation that would violate it:

```
totalReleased[domainId][asset] + pendingWithdrawalReserve[domainId][asset] <= totalDeposited[domainId][asset]
```

> Implementation sketch (Core, every mutating path):
>
> ```go
> // pseudocode — runs after applying any deposit/withdrawal/release effect
> rel := store.TotalReleased(domainId, asset)
> pend := store.PendingWithdrawalReserve(domainId, asset)
> dep := store.TotalDeposited(domainId, asset)
> if new(big.Int).Add(rel, pend).Cmp(dep) > 0 {
>     return constants.ErrSettlementConservationViolated // revert; refund attached value via embedded rollback
> }
> ```

- The per-batch withdrawal cap `MaxBatchWithdrawal` (§25, pre-mainnet) **MUST** be enforced per (domain, asset) so the maximum loss from a single fraudulent batch is bounded. With the refund bound (§27.9), a fraudulent batch can release at most `MaxBatchWithdrawal + depositCredit` per asset: its contract-emitted withdrawals, plus refunds up to the value deposited in that same batch. Custody from earlier batches is reached only through the `MaxBatchWithdrawal` part. No bond covers that loss; the bound limits exposure, and under `ZK_VALIDITY` Settlement rejects a batch whose validity proof does not verify (§1, §6.3).
- Developer fees (§10.7) are held outside these counters. They never raise `totalDeposited`, and paying or burning them never touches `totalReleased` or `pendingWithdrawalReserve`; their own guard is §22.4. Fee ZNN therefore can never back a withdrawal, and custody can never fund a fee payout.
- Aggregate over-withdrawal **MUST NOT** be possible and does not depend on executor honesty. Per-account correctness relies on executor honesty in Phase 1 and is removed by the validity proofs of `ZK_VALIDITY`.

**Custody mode `MINTED` (representation asset, reserved, gated).** For a `MINTED` (domain, asset), three coupled invariants **MUST** hold after every mutating operation:

```
(peg-in backing)     mintedSupply[d][a]             <= provenForeignLocks[d][a]
(peg-out authority)  authorizedForeignRelease[d][a] <= burnedSupply[d][a]
(L1 circulation)     totalReleased[d][a] + pendingWithdrawalReserve[d][a]
                         <= totalDeposited[d][a] + mintedSupply[d][a] - burnedSupply[d][a]
```

The third replaces the `ESCROW` invariant. A `MINTED` asset also circulates on L1 as a ZTS whose sole issuer is Settlement: a `kind=1` withdrawal to a Zenon address is released by `ReleaseWithdrawals` minting the ZTS rather than transferring it from custody, and a `Deposit` or payable `CallL2` carrying the ZTS burns it at the receive. Both legs use the `ESCROW` counters and `AssetFlowSummary` reconciliation unchanged (§18.4); the invariant bounds L1 circulation by the representation supply still live in L2.

`provenForeignLocks` increases only on a replay-checked, verified foreign lock (§18.4a); `mintedSupply` increases by a finalized batch's mint credit, never exceeding `provenForeignLocks`. `burnedSupply` increases by a finalized batch's burn debit; `authorizedForeignRelease` increases when `RelayMessage` registers a foreign peg-out authorization, never exceeding `burnedSupply`; a foreign release consumes exactly one `outboxId` (§17.3). Mint and burn **MUST** be gated on finalized batches only.

**Where the backing fact becomes Core-checkable.** `ESCROW` backing is an L1 `Deposit` event Core reconciles against natively (§18.4); a `MINTED` mint is backed by a *foreign* lock verified inside the STF, which Core never sees, so Core cannot reconcile a mint by deposit-event correspondence. The predicates above are well-defined safety properties, but the layer that *enforces* them depends on where the backing fact first becomes Core-checkable. Core **MUST** reject `custodyMode = MINTED` unless at least (a)+(b) hold:

- **(a) Economic floor.** A per-(domain, asset) `MaxBatchMint` cap (§25, Core hard-bounded, symmetric to `MaxBatchWithdrawal`) **MUST** bound per-batch mint exposure. No bond covers an over-mint; the cap bounds what one batch can mint, and (b) rejects a batch with an unbacked mint before it finalizes (§6.3, §18.4a).
- **(b) Proven execution.** `MINTED` **MUST** have `execProfile = ZK_VALIDITY` (§6.3): because the foreign-lock check is part of the STF, no validity proof exists for a batch with an unbacked mint, so Settlement rejects the batch. Pure `ATTESTATION` **MUST NOT** back a `MINTED` asset's aggregate solvency.
- **(c) Core-direct enforcement (strongest).** When `foreignProfile ∈ {COMMITTEE, ZK_CONSENSUS}` — a foreign fact Core can cheaply verify — Core **MUST** maintain `provenForeignLocks` itself by verifying the quorum signature or succinct proof when the lock is relayed (§18.4a), making the mint side honesty-independent, as `ESCROW` deposits are; burns remain STF state secured by (b). This is unavailable under `LIGHT_CLIENT` (too heavy for Core), which is why `LIGHT_CLIENT` `MINTED` relies on (a)+(b).

> The honest statement: **`MINTED` aggregate solvency is only as strong as the layer that makes the foreign backing fact checkable to Core** — cap-bounded and proven under `LIGHT_CLIENT` + `ZK_VALIDITY`; under `COMMITTEE`/`ZK_CONSENSUS` the mint side becomes honesty-independent while burns stay proven by the validity proof. The foreign legs reconcile through `MintFlowSummary` (§18.4a); the L1 legs through `AssetFlowSummary` (§18.4).

### 18.4 AssetFlowSummary

**Normative:** each batch commitment **MUST** include a bounded `AssetFlowSummary` of ≤ `MaxAssetsPerBatch` entries `(AssetID, depositCredit, withdrawalDebit)` (encoding §27.6). On `SubmitBatch`, Core **MUST** verify that deposit credits in the summary correspond to recorded L1 deposit events for that domain — including deposits carried by payable `CallL2` inputs (§18.5).

**`depositCredit` is the on-chain deposited amount** — the value locked into Settlement custody by the corresponding L1 `Deposit` or payable `CallL2` account-block, independent of whether the receiving contract claimed any or all of it. This definition reconciles against L1 deposit events (which is what Core verifies) and does not depend on contract behaviour or execution outcome. `withdrawalDebit` includes all outbox withdrawals from the batch, including runtime-emitted deposit refunds (§18.5). The per-asset conservation identity is:

```
depositCredit − withdrawalDebit(incl. refunds) = net amount that stayed in L2
```

This identity holds by construction across all deposit scenarios (success-partial, fail-full, partial-claim with refund) and **MUST** be stated explicitly in implementation documentation. Per-account flows are not verified on-chain in Phase 1.

### 18.4a MintFlowSummary (reserved, gated)

**Normative:** a batch commitment at `protocol_version = 2` (§19) carries a `MintFlowSummary` of ≤ `MaxAssetsPerBatch` entries `(AssetID, mintCredit, burnDebit)` (encoding §27.6a), one per `MINTED` asset whose foreign legs the batch touched. `mintCredit` is the representation the batch created in L2 state against proven foreign locks; `burnDebit` is the representation the batch destroyed in L2 state to back its foreign peg-out authorizations (`kind=1`, §6.6). A domain holding any `MINTED` asset **MUST** submit `protocol_version = 2` commitments. `AssetFlowSummary` keeps its meaning — L1 legs only — and **MAY** list a `MINTED` asset for its L1 deposits and withdrawals.

- **At `SubmitBatch`,** Core **MUST** verify that every entry names an asset registered `MINTED` for the domain, that `mintCredit` does not exceed that asset's `MaxBatchMint` (§25.2), and — under a Core-verifiable `foreignProfile` (`COMMITTEE`, `ZK_CONSENSUS`) — that the domain's cumulative accepted `mintCredit` does not exceed its cumulative Core-verified foreign locks, the `MINTED` counterpart of §18.4's deposit correspondence.
- **Core-verified locks.** Under a Core-verifiable `foreignProfile`, a foreign lock reaches the domain through `RelayForeignLock(domainId, asset, amount, foreignEventId, proof)`. Core **MUST** verify `proof` at the receive through the domain's registered verifier (§5.5), **MUST** reject a `foreignEventId` it has already recorded for the domain, and on acceptance **MUST** increase `provenForeignLocks` by `amount` and assign the account-block an input index like any input (§4.6). Under `LIGHT_CLIENT` the lock is verified only inside the STF, and `provenForeignLocks` increases by `mintCredit` when the batch finalizes; `MaxBatchMint` and the validity proof are then the only bounds, as (a) and (b) of §18.3 state.
- **At finalization,** Core **MUST** apply a batch's `MintFlowSummary` — `mintedSupply += mintCredit`, `burnedSupply += burnDebit` — before any operation reads that domain's `MINTED` counters, in ascending `batchId` order, lazily on the first such read after the batch's withdrawal delay elapses. `RelayMessage` of a foreign peg-out authorization **MUST** increase `authorizedForeignRelease` and **MUST** be rejected if the peg-out-authority invariant (§18.3) would fail.
- Foreign peg-out authorizations count toward `MaxBatchWithdrawal` exactly as L1 withdrawals do (§27.9).

### 18.5 Deposit delivery, claim, and refund

> Both value-entry methods deliver an attached ZTS value to a target contract. `Deposit` is the no-application-payload form (fund a contract / credit me, e.g. a contract owner funding their contract); payable `CallL2` is the with-payload form (invoke a method, optionally with value). Both are mechanically supported by the existing VM — an embedded `ReceiveBlock` already observes `sendBlock.TokenStandard` and `sendBlock.Amount` alongside the ABI call data (e.g. `LiquidityStakeMethod`), so one send can carry both value and a call.

**Normative:**

- A `Deposit` or `CallL2` account-block **MAY** carry an attached ZTS value (`sendBlock.TokenStandard`, `sendBlock.Amount`); a non-zero amount is a deposit for the input's `domainId` and `target_contract`. Settlement **MUST** reject at its receive an attached asset that is not custodyable for that domain: one with no `MaxBatchWithdrawal` configured for that (domain, asset) (§5.2, §27.9). A rejected call is not an input (§4.6).
- On the embedded receive, Core **MUST** increase `totalDeposited[domainId][asset]` by the attached amount. This custody increase is **unconditional and final** (the L1 transfer has already settled) and is independent of the outcome of the off-chain call.
- The runtime **MUST** populate the input frame's `deposit_asset`/`deposit_amount` (§27.1) from the on-chain send (`zero32`/`0` when none) and route execution to `target_contract`.
- **Explicit claim.** The contract declares, in its returned `ContractEffects`, a `claimed_deposit : u256` (§14.2) — how much of `deposit_amount` it accepts into its own accounting and **MUST** credit in its own `StateDiff`. The runtime **MUST** cause `RUNTIME_FAULT` if `claimed_deposit > deposit_amount`, or if `claimed_deposit > 0` when `deposit_amount == 0`. A contract that claims `0`, ignores the deposit, or fails claims nothing. `claimed_deposit` is always present in the effects blob; a contract that does not set it emits `0`, and a blob that omits the field is `RUNTIME_FAULT` (§27.1a).
- **Automatic refund of the unclaimed remainder.** After execution the runtime **MUST** refund `deposit_amount − claimed_deposit` (on `SUCCESS`) or the full `deposit_amount` (on any failure — the contract's effects, including any claim, are discarded per §14.3) to the depositor. The refund is a runtime-emitted `L1_WITHDRAWAL` outbox message (§27.4) with `recipient_l1 = the source account-block's L1 sender`, `asset_id = deposit_asset`, `amount = the unclaimed remainder`; it is appended to the input's outbox set and committed in `outboxRoot`, then registered by `RelayMessage` once its batch is `FINALIZED` and released by `ReleaseWithdrawals`, conservation-bounded (§18.3) and replay-protected like any withdrawal (§17.3). No refund outbox is emitted when the remainder is `0`.
- A contract therefore **cannot silently strand the unclaimed portion of a deposit**: anything it does not explicitly claim — because it failed, has no deposit logic, or claimed only part — is returned to the depositor automatically. The claimed portion is subject to the same per-account trust boundary as all Phase-1 balances (§1); the safe default (`claimed_deposit = 0`) is the do-nothing default.

> *Informative.* The L1 tokens never move during execution — they sit in Settlement custody throughout. "Claim" is bookkeeping: `claimed_deposit` is the amount the contract takes responsibility for (and must credit in its own state); the remainder has no owner in the contract-managed model (§18.1a) and is therefore refunded out of the custody it already occupies, once the refund outbox finalizes. The only residual trust is a contract that *explicitly* claims and then mishandles the funds — equivalent to any contract bug over money it has accepted, which no settlement-layer rule can prevent. A heavier alternative (a persistent per-account *landing-zone balance* that accrues unclaimed deposits for the depositor to reclaim at will, rather than auto-refunding per input) is recorded in §30 should it ever be wanted; the per-input auto-refund here is the lighter form and needs no protocol balance ledger.

> **Validity-proof scope note (normative forward hook).** The deposit-settlement step (custody increase, frame population, claim evaluation, refund outbox emission) is part of the runtime's STF, not the contract's effects blob. A validity proof under `ZK_VALIDITY` **MUST** cover this step: it is not sufficient to prove only "module + inputs ⇒ effects blob." The proof **MUST** additionally establish that `claimed_deposit` is within bounds, that the runtime correctly computed `deposit_amount − claimed_deposit`, and that the resulting refund outbox message (if any) is present at the correct `outbox_index` in the committed `outboxRoot`. All inputs to this check are available without new commitment structure: `deposit_asset`/`deposit_amount` are in the input frame (part of `input_data`), `claimed_deposit` is in the committed `ContractEffects`, and the refund outbox message is in the DA bundle.

---

## 19. Batch commitments

**Normative:** a batch commitment (canonical order) contains: `protocol_version`, `runtime_profile_version`, `metering_version`, `serialization_version`, `statediff_version`, `executionresult_version`, `receipt_version`, `outbox_version`, `damode_version` (all nine version fields of §24, `u16` each, in this order); `domainId`; `batchId` (`u64`); `firstInputSeq` (`u64`), `lastInputSeq` (`u64`) (the batch's input-sequence bounds, §4.5; equal to `firstGlobalInputIndex`/`lastGlobalInputIndex` for L1-sourced domains); `preStateRoot`, `postStateRoot`; `inputRoot` (Merkle root over the batch's canonical input encodings ordered by `globalInputIndex`); `receiptRoot`; `eventRoot`; `outboxRoot`; `DAHash`; `DAMode` (`u8`); `executorId` (`Address`, 20 bytes — §27.0); `submittedAtHeight` (`u64`); `assetFlowSummary`; at `protocol_version = 2` only, `mintFlowSummary` (§18.4a, §27.6a); `proofData` (reserved, **MUST** be empty in Phase 1 and Core **MUST** reject a non-empty value).

**Stored batch entry (normative).** An accepted commitment is stored as one entry per `(domainId, batchId)` (§5.4). `domainId` and `batchId` identify that entry rather than appearing inside it — they are its storage key — while the wire commitment above carries both inside the encoded structure. Of the nine version fields (§24) the wire commitment carries, the stored entry retains **only** `protocolVersion` (`u16`); the other eight are not stored. The entry otherwise carries the commitment's remaining fields — `firstInputSeq`, `lastInputSeq`, the six roots, `DAHash`, `DAMode`, `submittedAtHeight`, `assetFlowSummary`, `mintFlowSummary` when present, `proofData` — plus the batch's stored `state` (`SUBMITTED`, §27.11) and its `finalizeAtHeight` (§21), from which a batch is `FINALIZED`, and substitutes `executorPillar` — the submitting executor's pillar name (§22.1) — for the wire commitment's `executorId` (`Address`, §27.0). Core **MUST** resolve `executorPillar` at `SubmitBatch` from the caller address it has already verified. Recording the pillar rather than a position in the executor set keeps attribution exact however the set changes later: fee accrual and lapse (§22) read it directly. The stored entry is not a wire structure: only the encoding fixed under **Batch commitment wire format** below crosses the wire, and `executorPillar` never appears in it.

**Size/bounds:** a commitment **MUST** fit `MaxOnChainCommitmentSize` (15 KiB, §25 — sized so the commitment plus its ABI envelope fits `MaxDataLength`) — roots and bounded metadata only. A batch **MUST NOT** exceed `MaxBatchInputs` and **MUST** be contiguous per domain (§4.6).

**Batch commitment wire format (normative).** The commitment is encoded in the field order listed above using §27.0 primitives, with `assetFlowSummary` per §27.6 and `proofData : Bytes`. The version fields (§24) are encoded **first** in the structure; an unrecognized `protocol_version` **MUST** be rejected before any other field is interpreted. Phase 1 recognizes only `protocol_version = 1`, which has no `mintFlowSummary`; `protocol_version = 2` inserts it between `assetFlowSummary` and `proofData` and is reserved until a `MINTED` asset can be registered. In Phase 1, `proofData` **MUST** be the zero-length `Bytes` encoding (`u32` length `0x00000000`, no payload): it is a present-but-empty field, **never** omitted. Core **MUST** reject a Phase-1 commitment whose `proofData` length is non-zero, and **MUST** reject any commitment with trailing bytes after `proofData`. The commitment has exactly one valid encoding.

**SubmitBatch (normative):** only the executor address of an **ACTIVE** member of the network executor set (§22.1) that is the **entitled proposer** for the batch's `firstInputSeq` cursor position under the domain's `proposerPolicy` (§6.2) **MAY** submit. Core **MUST** verify: caller is the executor address of an ACTIVE member whose pillar is active, and that member is the entitled proposer for `domainId` at that cursor position; `executorId` equals the caller; the wire `submittedAtHeight` is `0`; `protocol_version` recognized; per-domain contiguity against the stored input-sequence cursor and Settlement's own assigned input indices (§4.6, incl. `lastInputSeq` not exceeding the highest assigned index); commitment size; `assetFlowSummary` deposit correspondence against the deposits recorded with the batch's assigned inputs (§4.6, §18.4); `proofData` is the zero-length encoding; and `preStateRoot == previous accepted postStateRoot` for that domain, or, for the domain's first batch, `preStateRoot` equal to the domain's genesis root — for `runtimeKind = WASM` the empty-tree root, the 32-byte zero hash (§13.2, §13.4), since the WASM genesis state has no leaves. Core **MUST** also verify that `batchId` equals the domain's `nextBatchId` (`0` for the first batch, then incremented by one per accepted batch), so no accepted batch can be overwritten, and that every one of the nine version fields equals its Phase-1 value (§24). On acceptance the batch enters `SUBMITTED`, the domain's cursor advances (`consumedInputCount` by the batch's input count, `nextBatchId` by one), the batch's fees accrue to its pillar (§22.4), and the assigned-index record sweep runs (§4.6). Core **MUST** store as the batch's `submittedAtHeight` the receive height of the `SubmitBatch` (§4.6), and derive `finalizeAtHeight` from it (§21). The executor cannot know that height when it encodes the commitment, so it sends `0`, and Core **MUST** reject any other wire value: an executor-chosen height could shorten the withdrawal delay. Acceptance heights never decrease, which is the ordering the fee-epoch queue relies on (§22.4). Core **MUST NOT** verify execution correctness in Phase 1. (Under the Phase-1 profile — `SINGLE`, a size-1 set — this reduces to "caller is the single registered executor address, and its pillar is active.")

---

## 20. Data availability

**Normative:**

- Phase 1 DA is commitment-only: Settlement **MUST** record `DAHash` and `DAMode` per batch and **MUST NOT** verify on-chain that the bundle is available. `DAHash` **MUST** be `SHA3-256` of the canonical execution data bundle, encoded exactly as defined below.
- The bundle for a batch **MUST** contain, per input: the canonical input data, the SMT witnesses for all keys read or written (including absent observations), and the per-input `ExecutionResult`. It **MUST NOT** be submitted on-chain; it is published off-chain and committed via `DAHash`.
- **Canonical bundle layout (normative).** The bundle is the concatenation, in **ascending `globalInputIndex` order**, of one `BundleInputRecord` per input in the batch, with no separators and no framing other than that defined here:

  ```
  BundleInputRecord :=
      globalInputIndex : u64
      inputData        : Bytes                          // canonical input encoding (§27.5)
      witnessCount     : u32
      witnesses        : witnessCount × WitnessEntry     // ascending by derivedKey, no duplicates
      executionResult  : Bytes                           // canonical ExecutionResult (§27.1a), u32-length-prefixed
      deployChunkData  : Bytes                           // input_kind = 3 only: the chunk's bytes (§11.2); absent otherwise

  WitnessEntry :=
      derivedKey : bytes32                               // the §13.3 derived SMT key
      value      : Bytes                                 // the key's value in the input's pre-state; the §27.2
                                                         // EMPTY sentinel (L = 0xFFFFFFFF, no bytes) when absent
      proof      : Bytes                                 // §13.4 proof bytes against the input's pre-state root:
                                                         // inclusion for a present key, non-inclusion for an absent one
  ```

  Witnesses **MUST** cover every key observed during the input — every key read or written, and every absent observation (§27.8) — sorted ascending by `derivedKey` with no duplicates. The set of observed keys **MUST** be a deterministic function of the input and the pre-state alone. An implementation **MUST NOT** let a local cache, memoization, or any other node-local state change which keys are observed: a read the runtime performs on a cold node **MUST** also be recorded as an observation on a warm one, or two honest nodes would commit different `DAHash` values for the same batch. `DAHash = SHA3-256(concat(BundleInputRecord for each input, in ascending globalInputIndex order))`. The bundle has exactly one valid encoding; any reorder of records, omission of a witnessed key, or duplicate witness yields a different `DAHash` and is non-conformant. `deployChunkData` is present if and only if the record's `inputData` carries `input_kind = 3`, and holds exactly the `chunk_data` whose `SHA3-256` is that input's `chunk_hash` (§11.2); every other record ends at `executionResult`. A decoder reads `input_kind` from `inputData` before reading past `executionResult`, so every record stays self-delimiting and a DA-only replayer can reassemble any deployed module from the bundles alone. Each `WitnessEntry` carries the value it proves: `L = 0x00000000` for a present-empty value (§13.6) and the `EMPTY` sentinel for an absent key (§27.8), so the three cases stay distinct. A replayer holding the bundle and the batch's `preStateRoot` can therefore re-execute every input from DA alone, verifying each value against its proof before use.
- `DAMode` **MUST** be an explicit `u8`. The only approved Phase 1 value is `DAMode = 0`: best-effort off-chain publication, content-addressed by `DAHash` (the gateway serves bundles over HTTP, `EXECUTOR.md` §21.5), retrievable in chunks ≤ `MaxDAChunkSize`. Settlement **MUST** verify `DAMode` is in the batch domain's approved `DAMode` set (§5.2).

> Limitation (**MUST** be disclosed): with no on-chain availability enforcement, an unavailable bundle under `ATTESTATION` reduces the affected batch to reliance on executor honesty.

---

## 21. Finality and withdrawal delay

**Normative:**

- A batch enters `SUBMITTED` on acceptance; no withdrawal **MAY** be released from a batch that is not yet `FINALIZED`. Its stored state stays `SUBMITTED`; it is `FINALIZED` at every height from its `finalizeAtHeight`: `submittedAtHeight` plus the domain's withdrawal delay in force when the batch is accepted, in L1 momentum heights, not wall-clock. Core **MUST** fix `finalizeAtHeight` at acceptance and store it with the batch (§19); a later `SetDomainConfig` change to the withdrawal delay applies only to batches accepted after it takes effect. Withdrawals **MAY** be released only from `FINALIZED` batches; `FINALIZED` is the only time gate — registration (§17.3) and the `ReleaseWithdrawals` crank (§18.2) add no additional delay.
- The withdrawal delay **MUST** be enforced by Core within hard bounds (§23). The exact duration is a pre-mainnet parameter (§25.2).

> In Phase 1 (testnet only) the delay protects through the bounded per-batch cap and public commitment visibility. Mainnet requires `ZK_VALIDITY`, whose finality model is `PROOF_VERIFIED` (§6.3, `../phase-2/`); the reserved `proofData` field is the forward hook.

---

## 22. Executor model

Executors are pillars. Pillars already run the infrastructure the network adopts, so executor identity, eligibility and pay hang off the pillar registry instead of a separately registered, bonded node type. Executors post no bond. Phase 1 is testnet-only (§1), and mainnet requires `ZK_VALIDITY`, under which Settlement rejects a batch whose validity proof does not verify (§6.3).

### 22.1 Identity

**Normative:**

- An executor is a pillar registered in the pillar contract, identified by its **pillar name**, together with an **executor address** the pillar chooses. The executor address signs `SubmitBatch`. It **MAY** differ from the pillar's `BlockProducingAddress`, so executor software never needs the producer key.
- One pillar holds at most one registration.
- A member is **ACTIVE** — eligible to submit — only while its pillar is active in the pillar contract and its registration has not been removed. Core **MUST** re-check pillar activity at every `SubmitBatch` (`context.MomentumStore().GetActivePillars()`), and **MUST** reject a batch from the executor address of a revoked pillar.
- Operator rewards are credited to the pillar's `RewardWithdrawAddress`, read from the pillar contract at settlement (§22.4). They are never credited to the executor address or the producer address.

### 22.2 Registration

**Normative:**

- **Phase 1.** `RegisterExecutor(pillarName, executorAddress)` is admin-only and time-challenged at `AdministratorDelay` (§23). Core **MUST** reject it unless `pillarName` names an active pillar. Under `SINGLE` it replaces the sole member. The outgoing pillar keeps every pending amount its accepted batches earned (§22.4).
- **Phase 2 (reserved).** `RegisterExecutor(executorAddress)` is sent by a pillar's current `BlockProducingAddress`. Core resolves the pillar from the sender and needs no administrator. The same call replaces that pillar's executor address.
- A registration change **MUST NOT** alter finalized state, and each domain's next batch **MUST** chain from that domain's last accepted `postStateRoot` (§6.2).

### 22.3 Submission and attribution

**Normative:**

- `SubmitBatch` **MUST** come from the executor address of an ACTIVE member that is the entitled proposer for the batch (§6.2, §19). The wire commitment's `executorId` **MUST** equal that address.
- Core **MUST** record the submitting pillar's name in the stored batch entry (§19). Fee attribution reads that field, so it stays exact across later registration changes.

### 22.4 Fee settlement

Developer fees (§10.7) are earned by the batch that carries the input, recorded per epoch, and paid only once every batch of that epoch is `FINALIZED`. Waiting for finality holds an executor's income for the withdrawal delay: there is no separate locked balance.

**Normative:**

- **Accrual.** On accepting a batch, Core **MUST** sum the fees recorded with the batch's inputs (§4.6, §10.7) and add the sum to the pending amount for `(E, pillar)`, where `E = submittedAtHeight / MomentumsPerEpoch` (integer division; a height-based epoch, independent of the time-based epochs of emission rewards) and `pillar` is the submitting pillar. Core **MUST** append `E` to the queue of pending epochs when `E` is newer than the queue's last entry, and raise `E`'s recorded finalization height to the batch's `finalizeAtHeight` (§21) when that is higher. `submittedAtHeight` is the acceptance height Core records (§19), and acceptance heights never decrease, so the queue stays in ascending epoch order without sorting.
- **The `Update` crank.** `Update()` takes no argument, is permissionless, and is rate-limited by `UpdateMinNumMomentums` like every embedded `Update`. Each accepted call settles at most one epoch: the epoch at the head of the pending queue. Epochs with no accruals are never queued, so periods without batches cost nothing to crank through. The head epoch `E` is settleable only when `E` has ended and the current height has reached `E`'s recorded finalization height, so that every batch submitted in `E` is `FINALIZED` (§21). An unsettleable head holds back every later epoch.
- **When `Update` does not settle.** If the head is not settleable, the queue is empty, a `PAUSE_RELAY` is in force, or the guard below would be broken, `Update` **MUST NOT** settle, and **MUST** still succeed and consume its rate limit. An error would roll the rate limit back and invite a resend every momentum.
- **Settling an epoch.** For each pillar with a pending amount `P` in `E`, Core **MUST** compute `burn = floor(P × FeeBurnShare / 100)`, credit `P − burn` to the `RewardDeposit` of that pillar's current `RewardWithdrawAddress`, and add `burn` to the epoch's burn total. When the pillar is not active at settlement (absent from `GetActivePillars()`), the whole of `P` is burned: a pillar that revokes forfeits its unsettled fees. Core **MUST** then burn the epoch's burn total with a single descendant ZNN burn send, delete `E`'s pending records and remove `E` from the queue. The burn total is computed and spent within the call and is not stored. The work is bounded by the size of the executor set (§6.2).
- **Collection.** `CollectReward()` **MUST** pay the caller's accrued ZNN `RewardDeposit` from Settlement's own balance with one descendant send, then zero it. It **MUST NOT** mint: these rewards are paid from fees already collected, unlike the emission rewards other embedded contracts mint.
- **Conservation guard (always in force).** Core **MUST** maintain `totalFeesCollected`, `totalFeesBurned`, `totalFeesCredited` and `totalRewardsCollected`, and **MUST NOT** perform any settlement or collection after which `totalFeesBurned + totalFeesCredited > totalFeesCollected` or `totalRewardsCollected > totalFeesCredited`. A blocked settlement leaves the epoch at the head of the queue for a later `Update`; a blocked collection fails with an error. Together with §18.3 this means fee payouts can never draw on custodied funds, whatever the administrator does.
- **Pause.** While `PAUSE_RELAY` is in force globally or for any domain (§23), `Update` **MUST NOT** settle and `CollectReward` **MUST** fail with an error. Pause **MUST NOT** cancel, reduce or redirect any pending or credited amount. Once no `PAUSE_RELAY` is in force, epochs settle in ascending order exactly as before.

> *Informative.* Pillars send `Update` to every contract in `types.EmbeddedWUpdate` automatically (`pillar/worker_updater.go`); listing Settlement there makes settlement run without anyone having to crank it. A bug that every implementation shares is invisible to cross-checks between them; the pause lets the administrator stop payouts and burns — both irreversible — while it is investigated. The guard is the backstop that needs no administrator: it catches any bug that would pay out more than was collected.

### 22.5 (Unassigned)

This section number is unassigned and holds no requirement.

### 22.6 (Unassigned)

This section number is unassigned and holds no requirement.

### 22.7 Phase 1 profile

**Normative:** the executor set has exactly one member, and nothing lapses. Downtime enforcement is an administrator action: the administrator replaces an unresponsive executor via `RegisterExecutor`, time-challenged at `AdministratorDelay` (§23) — the longer delay, because under the Phase-1 `ATTESTATION`/`SINGLE` profile a replacement executor controls the state root it submits, the same custody consequence the escape hatch carries. The only faults Settlement can detect in Phase 1 are process faults provable from its own storage (equivocation, double-submission).

---

## 23. Administration and emergency controls

> Zenon L1 has no on-chain multisig primitive in Phase 1, so a `k`-of-`n` authority cannot be expressed on-chain. Settlement's administrator is therefore a **single network administrator address** — the `types.NetworkAdmin` symbol carried in the node binary — the address Settlement gates its admin-only methods on today, and the address other administrator-gated contracts are intended to consolidate onto once the network-admin design lands there; that consolidation has not shipped, and bridge, liquidity and the spork contract still gate on their own separate addresses (§25.2). Settlement holds no administrator of its own: it gates directly on that global symbol from its first block, so it has no per-contract key to lose, compromise, or recover, and exposes no method of its own that installs, replaces, or recovers an administrator. **The stated goal remains to move this control to a multisig or governance contract once one exists**; because the authority is one address rather than per-contract storage, that migration repoints the address and needs no Settlement change.

**Normative:**

- Settlement Periphery **MUST** be controlled by the **network administrator address** (`types.NetworkAdmin`, `common/types/address.go`). Admin-gated methods **MUST** verify that the sending address equals the network administrator address and otherwise reject with `ErrPermissionDenied`. This authority check **MUST** be enforced at receive time, before any state write, and implementations **MUST NOT** move it after one. Settlement **MUST NOT** store an administrator address of its own, and **MUST NOT** expose a method that changes the network administrator. The administrator controls Periphery only; Core is node-binary and unreachable by the administrator (§5.2).
- Periphery-parameter changes **MUST** be time-challenged, using the shared `TimeChallenge` helper (`vm/embedded/implementation/common.go`), so users can observe a pending change and exit before it activates: soft parameter changes (`SetPeripheryConfig`, `SetDomainConfig`, `SetAssetConfig`, `SetApprovedDAModes`) take effect only after `SoftDelay`; the custody-touching escape hatch and `RegisterExecutor` — which, under the Phase-1 `ATTESTATION`/`SINGLE` profile, hands the incoming executor the same control over the submitted state root that makes the escape hatch custody-consequential (below) — take effect only after `AdministratorDelay`. A `RegisterDomain` call that changes `minExecutors` on an already-registered domain **MUST** likewise take effect only after `AdministratorDelay`, since `minExecutors` is the escape-hatch precondition (below); initial registration of a not-yet-active domain carries no prior depositors and is exempt. Both delays are **fixed Core constants** — `MinSoftDelay` (one epoch of momentum heights) and `MinAdministratorDelay` (twice that) — applied directly by the method. Settlement holds no Periphery delay field, offers no per-method delay configuration, and gives the administrator no lever over either value. Emergency pause is the single exception and **MUST** take effect immediately (below).

- A pending time challenge **MUST** be keyed by method and scope, so changes to different scopes never restart each other: the method name alone for network-wide methods (`SetPeripheryConfig`, `RegisterExecutor`, `InitiateEscapeHatch`); the method name and `domainId` for `SetDomainConfig`, `SetApprovedDAModes` and a `RegisterDomain` reconfiguration; the method name, `domainId` and `AssetID` for `SetAssetConfig`. Within one scope, at most one change is pending: re-invoking with different parameters restarts the challenge rather than queueing a second one.

> *Informative.* The shared helper keys a challenge by the string it is given (`timeChallengeKey(methodName)`, `vm/embedded/definition/common.go`), so a scoped key is a method name suffixed with its scope, for example `SetAssetConfig/<domainId>/<assetId>`.
- Core **MUST** enforce immutable hard bounds on Periphery values, independent of the administrator: a minimum and maximum withdrawal delay (`IsWithdrawalDelayInBounds`); a non-zero floor and a ceiling on `MaxBatchWithdrawal` (`IsMaxBatchWithdrawalInBounds`), and a monotonicity rule: a new `MaxBatchWithdrawal` for a (domain, asset) below the value in force **MUST** be rejected for as long as the domain is registered, whatever its `status` (§27.9); and a floor on the runtime-upgrade (`stfSpecHash`-bump) delay, which **MUST** be at least the domain's withdrawal delay in force — a relative floor evaluated as `IsRuntimeUpgradeDelayValid(upgradeDelay, withdrawalDelay)` (§6), not a fixed integer constant. The predicates and their bound holders live in `vm/constants/settlement.go`. An out-of-bounds value **MUST** be rejected at send validation with `ErrForbiddenParam`, never silently clamped. Concrete bound values are pre-mainnet (§25.2); the existence and enforcement of bounds are normative — **even a compromised administrator cannot exceed them**, and in particular cannot configure a runtime-upgrade delay shorter than the withdrawal delay in force. That floor is a relative predicate taking the active withdrawal delay as a parameter: `IsRuntimeUpgradeDelayValid`.
- The administrator **MUST NOT**: alter finalized roots/receipts/balances; alter canonical input order or `globalInputIndex`; bypass the withdrawal delay; move collected fees other than by the §22.4 settlement rules; or change SMT structure, key derivation, serialization formats, status-code semantics, or `ContractID` derivation (Core/protocol constants, changeable only by migration). These prohibitions hold by construction for `SetPeripheryConfig`, `SetDomainConfig`, `SetAssetConfig`, `SetApprovedDAModes`, `Pause` and `InitiateEscapeHatch`: those methods write only the Periphery configuration records, the approved-`DAMode` sets, the two pause-state records and the escape-hatch record. The administrator is also the sole caller of `RegisterDomain` and, in Phase 1, `RegisterExecutor` (§5.3, §22.2), which **DO** write the domain registry and the network executor set respectively — including `stfSpecHash` and `minExecutors` (`RegisterDomain`) and the pillar and executor address of the Phase-1 executor (`RegisterExecutor`), each within the bounds and delays this section and §6 state. No admin-gated method writes the batch chain, the input cursors and counters, the conservation counters, the release queue, the deploy records, the processed-outbox set, the fee accounting, or any `RewardDeposit`.

**Domain escape hatch (normative):** when a domain is stalled or falls below `minExecutors`, the administrator **MAY** initiate forced withdrawals against the last finalized root (§6). This is the single administrator action that moves custodied funds directly. Under the Phase-1 `ATTESTATION`/`SINGLE` profile, `RegisterExecutor` is a second path with custody consequence: because per-account correctness relies on executor honesty in that profile (§26), an administrator who replaces the executor controls the state root that executor submits for every domain, and therefore the balances that root implies — bounded only by `MaxBatchWithdrawal` and the withdrawal delay in force. The escape hatch itself **MUST** be subject to the `AdministratorDelay` time-lock, **MUST** record a durable, publicly readable on-chain escape-hatch record identifying the domain and the momentum height at which it was initiated, and **MUST NOT** alter any finalized root, receipt, or conservation counter. It is a last-resort governance action, not a routine operation. The forced-withdrawal mechanics themselves, including recording the triggering condition alongside the domain, remain deferred (§32).

**Emergency pause (normative):** an emergency pause **MUST** exist in Core, activatable by the network administrator (§23) — mirroring the bridge `Halt`/`Unhalt` pattern — and **MUST** take effect immediately, with no time-challenge. It has two independently settable scopes, applicable globally or per domain: `PAUSE_SUBMIT` (halts `SubmitBatch`) and `PAUSE_RELAY` (halts `RelayMessage` registration and `ReleaseWithdrawals` release; while set for any domain or globally, it also halts fee settlement by `Update` and `CollectReward`, §22.4). Each call names exactly one scope, one target (global, or a single `domainId`) and one activation state; a scope is in force for a domain when it is set globally **or** for that domain. `PAUSE_RELAY` **MUST NOT** be implied by `PAUSE_SUBMIT`: withdrawals from already-`FINALIZED`, past-delay batches **MUST** remain releasable until `PAUSE_RELAY` is explicitly set, which is reserved for an active asset-safety exploit (§27.10). Pause **MUST NOT** rewrite history or alter roots/receipts/balances/conservation counters/canonical order. Both scopes at both granularities **MUST** be durable, publicly readable on-chain records, so activation and deactivation are observable. Neither scope affects input admission; closing a domain to new inputs is its `PAUSED` status (§4.1, §6).

> *Informative — how the administrator address changes, and what bounds it meanwhile.* The network administrator address is a build-time value in the node binary, in the same class as the network's spork addresses. Rotating it — after key loss, after key compromise, or to hand control to a multisig or governance contract address — is a node release that changes the value plus an activation spork. Because the administrator gate is evaluated when an administrator call is executed, and execution results are re-derived when a node syncs, a rotation **MUST** select the authority by the height at which the new value takes effect, so that calls authorised by the previous address still re-execute to the same result. There is no on-chain break-glass and no on-chain rotation ceremony, which is why Settlement exposes no method that changes the address and stores nothing to recover. Until a migration lands, only one of the two constraints on the single key is genuinely custody-independent: the **Core hard bounds**, which no administrator value can exceed regardless of key custody. The **time-locks** (`AdministratorDelay` / `SoftDelay`) are not: `PAUSE_RELAY` takes effect with no delay (above) and halts `RelayMessage` registration and `ReleaseWithdrawals` release, so a compromised key can suspend the exit window the time-locks are meant to guarantee — and delay, though never redirect, executor pay (§22.4) — for as long as it holds `PAUSE_RELAY` set (`../research/bridges.md` §8.2). Neither rotation nor migration requires a Settlement storage migration: Settlement keeps no administrator record to migrate.

---

## 24. Versioning

**Normative:** the following components **MUST** carry explicit `u16` version fields, each first in its canonical encoding: `protocol_version` (batch commitments), `runtime_profile_version`, `metering_version` (Phase 1 = `1`, bound to `wasm-gas-table-v1.json`), `serialization_version`, `statediff_version`, `executionresult_version`, `receipt_version`, `outbox_version`, `damode_version`. **Every version field is `1` at activation.** Nothing is deployed before the activation spork, so a format change made before it updates the format in place and bumps no version; versions count changes after activation only. `proofData` (§19) is empty under single-executor `ATTESTATION`, carries a validity proof under `ZK_VALIDITY` (§6.3), and **MUST** be empty in Phase 1.

The semantics-defining domain fields of §6 (`domainClass`, `inputSource`, `execProfile`, `foreignProfile`, and the identity portion of `chainBinding`) are folded into `stfSpecHash`; changing any is a runtime-semantics change under `MinRuntimeUpgradeDelay` (§6). A commitment-format change that places a quorum attestation in `proofData` under an `ATTESTATION`-quorum sub-variant **MUST** be accompanied by a `protocol_version` bump and **MUST NOT** be activated while Phase-1 `proofData` rules are in force.

Settlement **MUST** reject a batch commitment whose `protocol_version` it does not recognize, and **MUST** reject one whose other eight version fields differ from the values this section fixes for the recognized `protocol_version` — under `protocol_version = 1`: `runtime_profile_version = 1`, `metering_version = 1`, `serialization_version = 1`, `statediff_version = 1`, `executionresult_version = 1`, `receipt_version = 1`, `outbox_version = 1`, `damode_version = 1`. The commitment commits to roots over receipts, state diffs and results, so a mismatch in any of them changes what the roots mean; clients and the executor **MUST NOT** silently interpret an unknown-version structure (unknown == reject). A version increment that changes any on-chain-committed format or circuit-relevant constant (the opcode gas table, the SMT hash function) **MUST** be accompanied by a migration specification; if the §13.2 hash decision replaces the SMT hash, that change is the principal anticipated migration and **MUST** be a one-time versioned state migration.

---

## 25. Constants

### 25.1 Fixed protocol constants

These are fixed for Phase 1 (changing any requires a major protocol version increment and is a breaking change to the validity-proof program). Settlement's constants live in `vm/constants/` and are enforced by Core as hard bounds where applicable; the runtime limits live in the executor (`plugins/wasm/limits.go`, `internal/codec/constants.go`).

| Constant | Value |
|---|---|
| `MaxDataLength` (L1 account-block data) | 16 KiB (`1024 * 16`) — already `vm/constants/plasma.go` |
| `MaxOnChainCommitmentSize` (sized so commitment + ABI envelope fits `MaxDataLength`) | 15 KiB (`1024 * 15`) |
| `MaxCallPayload` | 8 KiB |
| `MaxReturnData` | 4 KiB (also bounds revert data, §15) |
| `MaxExecutionResult` | 32 KiB |
| `MaxResultBlob` (returned `ContractEffects` blob, checked before decode — §14.3) | 64 KiB |
| `MaxBatchInputs` | 64 |
| `MaxDAChunkSize` (off-chain DA bundle chunk serving only, §20) | 16 KiB |
| `MaxDeployChunkData` (on-chain deploy chunk payload, §11.2 — sized so chunk + ABI envelope fits `MaxDataLength`) | 15 KiB (`1024 * 15`) |
| `MaxCodeSize` (pre-instrumentation, runtime-neutral) | 256 KiB |
| `MaxChunkCount` | 18 |
| `DeploymentExpiryWindow` | 2,880 momentum heights |
| `MaxContractMetadataSize` (`DeployContractStart.metadata`, §11.1, §27.12) | 256 bytes |
| `MaxMemory` | 4 MiB (64 WASM pages) |
| `MaxStackDepth` | 512 frames |
| `MaxLocalsPerFunction` (params + declared locals, per function; also the parameter count of each declared function type, whether or not a function uses it; declared locals are also charged at function entry, §10.2) | 8,192 |
| `MaxTotalLocals` (params + declared locals summed across every function body) | 1,048,576 (`2^20`) |
| `MaxTotalParams` (parameters summed across the type section) | 65,536 (`2^16`) |
| `MaxTypeCount` (declared function types) | 1,024 |
| `MaxFunctions` (locally defined functions) | 8,192 |
| `MaxGlobals` (declared globals) | 1,024 |
| `MaxDataSegments` (data-section segment count) | 1,024 |
| `MaxTableEntries` (declared table minimum) | 10,000 |
| `MaxTotalElementEntries` (element-segment entries summed across the element section) | 16,384 |
| `MaxInstrumentedCodeSize` (post-instrumentation size ceiling) | 1 MiB |
| `MaxStateKeySize` | 32 bytes |
| `MaxStateValueSize` | 16 KiB |
| `MaxEventsPerExecution` | 64 |
| `MaxEventDataSize` | 1 KiB |
| `MaxTopicsPerEvent` | 4 |
| `MaxOutboxPerExecution` | 32 |
| `MaxOutboxPayloadSize` | 8 KiB |
| `MaxAssetsPerBatch` | 16 |
| `MaxWithdrawalsPerUpdate` (per `ReleaseWithdrawals` call) | 16 |
| `ReleaseIntervalMomentums` (minimum momentums between a domain's accepted `ReleaseWithdrawals` calls, §18.2) | 1 |
| `MaxMerkleProofDepth` (steps in a §27.3 `MerkleProof`) | 32 |
| `MaxRecordSweepBatchesPerSubmit` (finalized batches whose assigned-index records one `SubmitBatch` deletes, §4.6) | 2 |
| `MaxDeploySweepPerStart` (expired start-queue entries swept per `DeployContractStart`, §11.5) | 2 |
| `StateLeafOverhead` (bytes counted per present state leaf, on top of its value length — §13.8) | 64 |
| `FeeBurnShare` (percentage of each developer fee burned; the rest goes to the operator — §22.4) | 30 |
| `MaxVerifyingKeySize` (reserved, §5.5 — sized so the key plus its ABI envelope fits `MaxDataLength`) | 15 KiB (`1024 * 15`) |
| `MaxInputGas` (ceiling on a `CallL2`/`Deposit` `plasma_limit`, hence on one input's L2 gas budget — §10.1) | Pre-mainnet (§25.2); **MUST** be below `2^63 - 1` so every conformant budget fits a signed 64-bit ledger |
| `MinRuntimeUpgradeDelay` (Core hard floor on the `stfSpecHash`-bump delay, §6, §23) | `≥ WithdrawalDelay` — relative to the active withdrawal delay, not a fixed integer; enforced by the predicate `IsRuntimeUpgradeDelayValid(upgradeDelay, withdrawalDelay)`, evaluated against the domain's withdrawal delay in force after the configuring call — when both values are set in the same call, against the withdrawal delay being set, not the one it replaces; there is no `MinRuntimeUpgradeDelay` integer constant. Core **MUST** reject any configured runtime-upgrade delay below the active `WithdrawalDelay` |

### 25.2 Pre-mainnet parameters

Not fixed by this spec; **MUST** be set before mainnet, bounded by Core hard bounds (§23) where noted. The bound holders and their predicates live in `vm/constants/settlement.go`: the floors and the existence and enforcement of the bounds are normative, while the concrete ceilings carried there today are placeholders to be finalized before mainnet.

| Parameter | Notes |
|---|---|
| Withdrawal-delay duration | Per domain, in momentum heights; within hard bounds (`WithdrawalDelayMin` / `WithdrawalDelayMax`). Natural floor reference: `MomentumsPerEpoch` (`vm/constants/embedded.go`), derived directly, not via `MinSoftDelay`. |
| `znnPerByteOfBytecode`, `znnPerByteOfState` | Developer-fee prices in ZNN (§10.7), Periphery values within Core floors and ceilings (`IsZnnPerByteOfBytecodeInBounds`, `IsZnnPerByteOfStateInBounds`); both floors non-zero. |
| `MaxBatchWithdrawal` | Within Core floor/ceiling (`MaxBatchWithdrawalFloor` / `MaxBatchWithdrawalCeiling`), per (domain, asset). |
| `MaxBatchMint` (reserved; `MINTED` only, §18.3) | Per-batch mint cap per (domain, asset), Core floor/ceiling (`MaxBatchMintFloor` / `MaxBatchMintCeiling`), symmetric to `MaxBatchWithdrawal`; bounds over-mint exposure. Not used while the Phase-1 profile is in force. |
| Network administrator address | The single global `types.NetworkAdmin` symbol — the address Settlement gates its admin-only methods on today, and the address every other administrator-gated contract is intended to consolidate onto once the network-admin design lands there; that consolidation has not shipped, and bridge, liquidity and the spork contract still gate on their own separate addresses (§23). Settlement stores no administrator of its own. It is a build-time value, so the address carried in the released binary **MUST** be confirmed before mainnet activation. The `AdministratorDelay` and `SoftDelay` windows are not parameters: they are fixed at the Core constants `MinAdministratorDelay` and `MinSoftDelay` (§23), with no administrator override. |
| Runtime-upgrade (`stfSpecHash`) delay length | Mandatory user-exit window. |
| `MaxInputGas` | Ceiling on one input's L2 gas budget (§10.1, §25.1). Size it from the batch, not the input: an honest executor's worst case is `MaxBatchInputs × MaxInputGas`, so choose the per-input value from the wall-clock time a worst-case batch may take. **MUST** be below `2^63 - 1`. It also bounds the per-input execution trace a validity proof must cover, so lowering it later is a breaking change to the validity-proof program. |

---

## 26. Security properties

**Normative:**

- **Sequencing integrity.** The executor **MUST NOT** reorder, skip, or privately insert inputs; canonical order is fixed by L1 (§4.4); omission yields a non-contiguous batch rejected on-chain (§4.6); a `CallL2` in a confirmed momentum is force-included (§4.2).
- **MEV.** Executor-level sandwiching/reordering is structurally prevented (no ordering discretion; non-canonical batches rejected). *Informative:* L1 inclusion-timing, application-level, oracle-timing, and cross-domain arbitrage MEV are out of scope (L1- or application-layer concerns).
- **Replay.** L1-originated inputs are non-replayable (tied to a unique confirmed account-block and `chain_id`, §27.5); outbox messages are non-replayable via unique `outboxId` + `processedOutbox` (§17.3).
- **Determinism.** Execution **MUST** be deterministic per §7 and §13; any implementation producing a different `postStateRoot` for the same input and witnessed pre-state is non-conformant.
- **DA / trust limits.** Phase 1 is commitment-only DA and unbonded single-executor attestation, on testnet only (§1); aggregate over-withdrawal is prevented on-chain (§18.3), per-account correctness relies on executor honesty until `ZK_VALIDITY` (Phase 2). These limits **MUST** be disclosed and **MUST NOT** be overstated.

---

## 27. Wire formats and tightened semantics

This section is fully normative. Where it and an earlier section overlap, this section governs the exact encoding; the earlier section governs intent.

### 27.0 Common primitives

`u8`/`u16`/`u32`/`u64`/`u256`: fixed-width big-endian unsigned, of exactly 1/2/4/8/32 bytes respectively. A `u256` is always exactly 32 bytes; it is **not** range-checked at the codec layer (a value larger than, e.g., total supply is a well-formed `u256` — domain range checks such as conservation are the responsibility of §18.3, not the decoder, which **MUST NOT** add codec-level range rejections beyond fixed width). `bytes32`/`Hash`/`ContractID`/`AssetID`/`CodeHash`: fixed 32-byte. `Address`: fixed 20-byte (only in `caller` preimages, `ContractID` preimages (§12's `deployer_address`), withdrawal recipients, `executorId`, the `new_owner` field of §27.5 and the `owner` field of §27.12; never substituted for a 32-byte field). `Bytes`: `u32` length `L` + `L` raw bytes. `List<T>`: `u32` count `N` + `N` encodings, no separators. Every structure has exactly one valid encoding; a decoder **MUST** reject trailing bytes, inconsistent length/count prefixes, or out-of-order fields.

### 27.1 Entry-point ABI

**Normative:**

- A module **MUST** export linear memory as `memory` and exactly one entry `run` with signature `(i32, i32) -> i64`. `run` **MUST** name a function the module defines, not an import.
- Before invoking `run`, the runtime **MUST** instantiate the module — applying its active data and element segments per Core 1.0 — then write a contiguous **input frame** at offset 0 and call `run(0, frame_len)`. The frame is written **after** segment initialization, so it overwrites any active data segment content in `[0, frame_len)`. A module whose declared initial memory is smaller than `frame_len` **MUST** yield `RUNTIME_FAULT`: the frame write fails, no `run` call is made, and the receipt carries only the pre-execution initial-memory charge. The runtime **MUST NOT** grow memory to fit the frame. Memory outside `[0, frame_len)` and outside the ranges covered by active data segments is zero. The frame is the canonical encoding of:

  ```
  ExecutionInputFrame :=
      context_version : u16          // = runtime_profile_version
      chain_id        : bytes32
      caller          : bytes32
      contract_id     : bytes32
      code_hash       : bytes32      // post-instrumentation CodeHash
      momentum_height : u64
      momentum_time   : u64          // TimestampUnix
      momentum_hash   : bytes32      // hash of the momentum at momentum_height
      input_index     : u64          // globalInputIndex
      deposit_asset   : AssetID      // attached deposit asset (payable CallL2 / Deposit); zero32 when none
      deposit_amount  : u256         // attached deposit amount; 0 when none
      payload         : Bytes        // input-type-specific (§27.5)
  ```

  Every field before `payload` is fixed-width: `momentum_hash` starts at byte 146 and the `payload` length prefix at byte 250, so a frame is 254 bytes plus its payload.

- `run` **MUST** return `i64 r`: if `r ≥ 0`, `result_ptr = (r >> 32) & 0xFFFFFFFF`, `result_len = r & 0xFFFFFFFF`; the runtime reads exactly `result_len` bytes at `result_ptr` as the `ContractEffects` blob (§14.2, grammar §27.1a), then assembles the `ExecutionResult`. `result_ptr + result_len` **MUST** be in-bounds, else `RUNTIME_FAULT`. If `r < 0`, treat as `abort` with code `(-r)` truncated to `i32` and empty revert data, status `REVERT`, full rollback. A trap during `run` **MUST** canonicalize to `RUNTIME_FAULT` regardless of any partial result region. The runtime **MUST NOT** call any function other than `run`; a module without `memory` and a conforming `run` **MUST** be rejected at deployment validation.

### 27.1a ContractEffects and ExecutionResult wire formats

**Normative.** The blob returned by `run` is exactly the following, in this field order, using §27.0 primitives:

```
ContractEffects :=
    executionresult_version : u16
    globalInputIndex        : u64
    StateDiff               : List<StateDiffEntry-local>   // §27.2 local-key form
    Events                  : List<Event>                  // §27.3
    OutboxMessages          : List<OutboxMessage>          // §27.4
    ReturnData              : Bytes
    claimed_deposit         : u256                         // ALWAYS present; 0 when deposit_amount == 0
```

- `claimed_deposit` is a fixed 32-byte field and **MUST** always be encoded, including the value `0`. A blob that omits it (i.e. ends after `ReturnData`) **MUST** cause `RUNTIME_FAULT`.
- Every `Event.contract_id` and every `OutboxMessage.source_contract` in the returned blob **MUST** equal the executing `contract_id` supplied in the input frame (§27.1); a blob containing any other value **MUST** cause `RUNTIME_FAULT` with full rollback. In particular a contract **MUST NOT** be able to emit a message with `source_contract = bytes32(0)`, which is reserved to mark a runtime-emitted deposit refund (§27.4, §18.5).
- Every `Event.input_index` and every `OutboxMessage.input_index` in the returned blob **MUST** equal the blob's `globalInputIndex`; a decoder **MUST** reject any other value, so the blob causes `RUNTIME_FAULT`. A foreign `input_index` would otherwise let a contract mint an `outboxId` belonging to another input (§17.3).
- The runtime **MUST** cause `RUNTIME_FAULT` if `claimed_deposit > deposit_amount`, or if `claimed_deposit > 0` when `deposit_amount == 0` (§18.5).
- The blob **MUST NOT** contain `preStateRoot` or `postStateRoot`; presence of either field (i.e. any decoding that yields a 9-field structure with roots after `globalInputIndex`) **MUST** cause `RUNTIME_FAULT` (§14.2).
- Every `asset_id` of an `L1_WITHDRAWAL` message in the blob **MUST** be a canonical `AssetID` (§18.1); a decoder **MUST** reject a non-canonical one, so the blob causes `RUNTIME_FAULT`. The runtime therefore never commits a withdrawal whose `asset_id` Settlement would reject at relay; §17.3 lists the withdrawals Settlement rejects on other grounds.
- A decoder **MUST** reject trailing bytes after `claimed_deposit`, any length/count-prefix inconsistency, out-of-order fields, or a non-canonical `Events` or `OutboxMessages` list order, or, in an `ExecutionResult`, a non-canonical `StateDiff` list order.

The runtime assembles the canonical **`ExecutionResult`** from a successful `ContractEffects` blob by (a) inserting `preStateRoot`,`postStateRoot` immediately after `globalInputIndex`, and (b) re-encoding `StateDiff` from local-key form into the derived-key `StateDiffEntry` form (§27.2), sorted ascending by derived key:

```
ExecutionResult :=
    executionresult_version : u16
    globalInputIndex        : u64
    preStateRoot            : bytes32
    postStateRoot           : bytes32
    StateDiff               : List<StateDiffEntry>         // §27.2 derived-key form, sorted
    Events                  : List<Event>                  // §27.3
    OutboxMessages          : List<OutboxMessage>          // §27.4, incl. runtime-emitted refund (§18.5)
    ReturnData              : Bytes (≤ MaxReturnData)      // §14.3; also bounds revert data (§15)
    claimed_deposit         : u256
```

On `REVERT`, the runtime assembles the `ExecutionResult` with `postStateRoot = preStateRoot`, empty `StateDiff` and `Events`, `OutboxMessages` holding only the deposit refund when one is due (§18.5), `ReturnData` equal to the revert data (§15), and `claimed_deposit = 0`.

`ExecutionResult` has exactly one valid encoding; a decoder **MUST** reject trailing bytes or any deviation from this field order. A decoder **MUST** reject an `ExecutionResult` whose `ReturnData` exceeds `MaxReturnData`. It **MUST** likewise reject an `ExecutionResult` in which any `Event.input_index` or `OutboxMessage.input_index` differs from its `globalInputIndex`. The total serialized `ExecutionResult` **MUST NOT** exceed `MaxExecutionResult` (§14.3, §25). `ContractEffects.ReturnData` carries no grammar bound: an over-long value decodes, and step 7 of §10.6 turns it into `RESULT_TOO_LARGE`.

### 27.2 StateDiff and the EMPTY deletion sentinel

**Normative (key encoding differs between the two blobs):**

- In a **`ContractEffects`** blob, each `StateDiff` key is a length-prefixed `local_key` (`u32 L ≤ MaxStateKeySize` + `L` bytes); entries need not be sorted (canonical order is over derived keys). The runtime derives `SHA3-256(contract_id || local_key)` and **MUST** reject a blob that, after derivation, has duplicate derived keys.
- In the canonical **`ExecutionResult`**, each `StateDiff` entry is:

  ```
  StateDiffEntry := key : bytes32 (derived SMT key) ; new_value : Bytes
  ```

  Entries **MUST** be sorted ascending by derived key with no duplicates; a decoder **MUST** reject duplicates or non-ascending order.
- **Deletion sentinel:** deletion is `new_value` with length prefix `L = 0xFFFFFFFF` followed by zero bytes (`EMPTY`); it removes `key` (a subsequent non-inclusion proof). A present empty value is `L = 0x00000000` (inclusion of the empty value). A present value **MUST NOT** have length `0xFFFFFFFF`; lengths in `[MaxStateValueSize+1, 0xFFFFFFFE]` **MUST** be rejected. Deleting a non-existent key is a well-formed no-op leaving the root unchanged.

### 27.3 Event encoding and Merkle construction

```
Event := contract_id : bytes32 ; input_index : u64 ; event_index : u32 ;
         topics : List<bytes32> (≤ MaxTopicsPerEvent) ; data : Bytes (≤ MaxEventDataSize)
```

Within an input, events appear in ascending `event_index` from 0 with no gaps, ≤ `MaxEventsPerExecution`. **Merkle construction (all roots in §27):** leaf hash `SHA3-256(0x00 || encoded_element)`, node hash `SHA3-256(0x01 || left || right)`; for an odd node count at a level, the last node is promoted unchanged (no duplication); the empty set has root `bytes32(0)`. (Distinct from the SMT, §13.) The batch `eventRoot` is over all batch events ordered by `(input_index, event_index)`.

**Inclusion proof (all roots in §27):**

```
MerkleProof := step_count : u32 ; steps : (side : u8 ; sibling : bytes32) × step_count
```

Steps run from the leaf level up. `side = 0` means the sibling is the left node, so the next node is `SHA3-256(0x01 || sibling || node)`; `side = 1` means it is the right node, so the next node is `SHA3-256(0x01 || node || sibling)`. Any other `side` **MUST** be rejected. A level at which the node is the promoted last node contributes no step. A verifier **MUST** reject `step_count > MaxMerkleProofDepth` (§25.1) before reading any step, and **MUST** reject trailing bytes. It **MUST** compute the leaf hash from the element it is given, `SHA3-256(0x00 || encoded_element)`, and **MUST NOT** accept a leaf hash from the caller. The proof verifies when the final node equals the committed root. The empty-set root admits no proof.

### 27.4 OutboxMessage encoding

```
OutboxMessage := source_contract : bytes32 ; input_index : u64 ; outbox_index : u32 ;
    kind : u8 (0=L2_DELIVERY, 1=L1_WITHDRAWAL) ;
    // kind==0: target_contract : bytes32
    // kind==1: recipient_l1 : Address(20) ; asset_id : AssetID ; amount : u256
    payload : Bytes (≤ MaxOutboxPayloadSize)
```

`kind` **MUST** be 0 or 1 (else `RUNTIME_FAULT`). A decoder selects the branch from `kind` alone: bytes that do not continue as that branch and the fields after it fail the decode under §27.1a's length and count rules. For `kind == 1` (L1 withdrawal), `payload` **MUST** be the zero-length `Bytes` encoding; a non-empty payload on a withdrawal **MUST** cause `RUNTIME_FAULT` (the withdrawal's effect is fully determined by `recipient_l1`/`asset_id`/`amount`, and `outboxId` does not commit to `payload`, so a non-empty withdrawal payload is unverifiable surface). Within an input, ascending `outbox_index` from 0, no gaps. `MaxOutboxPerExecution` bounds the messages a **contract** emits, and so bounds the `OutboxMessage` list of a returned `ContractEffects` blob. The runtime-emitted deposit refund (§18.5) is appended after that count and is **not** subject to it, exactly as §27.9 exempts it from the `MaxBatchWithdrawal` aggregate: a contract emitting `MaxOutboxPerExecution` messages alongside an unclaimed deposit remainder keeps both its messages and the depositor's refund. The `OutboxMessage` list of an `ExecutionResult` is therefore bounded at `MaxOutboxPerExecution + 1`, and a decoder **MUST** accept that count for an `ExecutionResult` and reject it for a `ContractEffects`. `outboxId = SHA3-256(domainId(u32) || batchId(u64) || input_index(u64) || outbox_index(u32))`; `domainId` and `batchId` are bound at batch construction by the runtime. Roots use the §27.3 construction, batch root ordered by `(input_index, outbox_index)`.

> **Foreign-recipient variant (reserved, gated).** The `kind==1` branch above encodes `recipient_l1 : Address(20)` — a Zenon-native withdrawal. A `BRIDGE` peg-out (§6.6) targets a foreign-chain address of chain-specific width; its encoding is a reserved, version-gated `kind==1` sub-variant (a `recipient : Bytes` form selected by the domain's `chainBinding`) accompanied by a `protocol_version`/`outbox_version` bump. It **MUST NOT** be emitted while the Phase-1 profile is in force; Phase-1 `kind==1` messages are Zenon-native `Address(20)` withdrawals only.

A **runtime-emitted deposit refund** (§18.5) is an `L1_WITHDRAWAL` message with `source_contract = bytes32(0)` (no contract source; the runtime is the emitter) and an empty `payload`. Its `outbox_index` **MUST** equal the number of retained contract outbox messages for the input — which is `0` on any non-`SUCCESS` (where contract messages are discarded), and equal to the count of contract-emitted messages on `SUCCESS`. It occupies the next `outbox_index` after any contract-emitted messages on success, or `outbox_index = 0` on a failed input. It is otherwise an ordinary outbox message for `outboxRoot`, relay, and replay purposes.

### 27.5 CallL2 / Deposit payload and input identity

```
CallL2Args  := target_contract : bytes32 ; plasma_limit : u64 ; payload : Bytes (≤ MaxCallPayload)
DepositArgs := target_contract : bytes32 ; plasma_limit : u64    // no application payload
```

(The `domainId` envelope is carried by the Settlement method call, §4.1.) Neither input carries an application-level nonce; replay protection is L1 account-block uniqueness + `chain_id` binding. The runtime routes execution to `target_contract`; `plasma_limit` is consumed by L2 gas accounting and is **NOT** placed in the contract-visible frame. For `CallL2` the frame `payload` is `CallL2Args.payload`; for `Deposit` the frame `payload` is empty. Any attached ZTS value on the account-block is a deposit delivered to `target_contract` (§18.5): the runtime sets the frame `deposit_asset`/`deposit_amount` from the on-chain send (`zero32`/`0` when none); the `Args` carry no value field. `caller` is derived per §9.2. `inputHash` (receipt) **MUST** be `SHA3-256(input_kind:u8 || source_account_block_hash:bytes32 || …kind-specific…)`, binding the receipt to the exact L1 account-block; the kind-specific tail is `target_contract:bytes32 || plasma_limit:u64 || deposit_asset:AssetID || deposit_amount:u256 || payload:Bytes` for `CallL2`, and `target_contract:bytes32 || plasma_limit:u64 || deposit_asset:AssetID || deposit_amount:u256` for `Deposit`.

**Canonical input encoding and `inputRoot` (normative).** The `inputRoot` (§19) leaf for an input is its **canonical input encoding**: `input_kind:u8 || source_account_block_hash:bytes32 || <kind-specific tail>`, byte-identical to the `inputHash` preimage above, defined for **all eight** input kinds (not only `CallL2`/`Deposit`) and, as a fallback, for any other `input_kind` (final bullet):

- `CallL2` (`input_kind = 0`): tail `target_contract:bytes32 || plasma_limit:u64 || deposit_asset:AssetID || deposit_amount:u256 || payload:Bytes`.
- `Deposit` (`input_kind = 1`): tail `target_contract:bytes32 || plasma_limit:u64 || deposit_asset:AssetID || deposit_amount:u256`.
- `DeployContractStart` (`input_kind = 2`): tail `code_hash:bytes32 || metadata_hash:bytes32 || total_size:u64 || chunk_count:u32 || metadata:Bytes` (§11.1 args in declared order).
- `DeployContractChunk` (`input_kind = 3`): tail `code_hash:bytes32 || chunk_index:u32 || chunk_hash:bytes32` (the `chunk_data` is committed by `chunk_hash`, not inlined).
- `DeployContractFinalize` (`input_kind = 4`): tail `code_hash:bytes32`.
- `UpgradeContract` (`input_kind = 5`): tail `contract_id:bytes32 || new_code_hash:bytes32` (§12), where `new_code_hash` is the pre-instrumentation `code_hash` of the replacement bytecode.
- `TransferContractOwnership` (`input_kind = 6`): tail `contract_id:bytes32 || new_owner:Address(20)` (§12).
- `BuyStorage` (`input_kind = 7`): tail `contract_id:bytes32 || bytes:u64` (§10.7, §13.8). No fee appears in any input encoding: Settlement fixes every fee at its receive, and fees do not affect execution.
- Any other `input_kind`: the canonical input encoding is the `input_kind:u8 || source_account_block_hash:bytes32` prefix alone, with no tail, since no tail is defined for the kind. This is the `inputHash` preimage and the `inputRoot` leaf for such an input; its receipt is `UNSUPPORTED_OPERATION` (§15). `source_account_block_hash` alone binds the receipt to the exact L1 account-block that commits the undecodable bytes.

The `input_kind` ordinals are fixed and **MUST NOT** be reused. `inputRoot` is the §27.3 Merkle root over these canonical input encodings ordered by `globalInputIndex`. The same encoding is the `inputData` field of the DA bundle record (§20).

### 27.6 AssetFlowSummary encoding

```
AssetFlowSummary := List<AssetFlowEntry> (≤ MaxAssetsPerBatch)
AssetFlowEntry := asset_id : AssetID ; deposit_credit : u256 ; withdrawal_debit : u256
```

Sorted by `asset_id` ascending, no duplicates; **MUST** include every asset with a non-zero deposit or withdrawal amount and **MUST NOT** include an asset with both fields zero. Exceeding `MaxAssetsPerBatch` means the executor **MUST** split activity across batches.

### 27.6a MintFlowSummary encoding (reserved, `protocol_version = 2`)

```
MintFlowSummary := List<MintFlowEntry> (≤ MaxAssetsPerBatch)
MintFlowEntry := asset_id : AssetID ; mint_credit : u256 ; burn_debit : u256
```

The same ordering and exclusion rules as §27.6, over `MINTED` assets with a non-zero foreign leg.

### 27.7 Receipt encoding

```
Receipt := receipt_version : u16 ; input_index : u64 ; input_hash : bytes32 ; batch_id : u64 ;
    contract_id : bytes32 (or 0) ; code_hash : bytes32 (or 0) ; status : u8 ; gas_used : u64 ;
    abort_code : i32 ; state_diff_hash : bytes32 (or 0) ; return_hash : bytes32 (or 0) ;
    event_root : bytes32 (or 0) ; event_count : u32 ; outbox_root : bytes32 (or 0) ; outbox_count : u32
```

`abort_code` is a fixed-width big-endian **signed** 32-bit integer, the one signed primitive in this specification; every other integer is unsigned (§27.0). Status ordinals fixed (§15); the batch `receiptRoot` uses the §27.3 construction over receipts ordered by `input_index`. `abort_code`, `state_diff_hash` and `return_hash` are fixed by §15.

### 27.8 storage_len: absent vs empty

**Normative:** over derived SMT keys (§13.3), the runtime **MUST** expose: absent → `storage_len = -1`, `storage_read = -1` (writes nothing); present-empty → `0` / `0` (writes zero bytes); present length `L>0` → `L` / `L` if buffer ≥ L else `-2`. A key deleted via `EMPTY` is absent (`-1`). Every observed key (incl. absent) **MUST** be witnessed (§20). A `key_len` exceeding `MaxStateKeySize` (§13.3, §25) is not a defined observation: the host function **MUST** trap, which canonicalizes to `RUNTIME_FAULT` with full rollback (§7.5). It **MUST NOT** be reported through a return value, and the key **MUST NOT** be witnessed.

### 27.9 Withdrawal fan-out per batch

**Normative:** `RelayMessage` releases no funds — it registers exactly one message per call (§17.3, §18.2). A single `ReleaseWithdrawals` **MUST** release at most `MaxWithdrawalsPerUpdate = 16` withdrawals (one descendant send each), draining the domain's release queue in registration order. Independently, a single batch **MUST NOT** register withdrawals whose aggregate per (domain, asset) exceeds `MaxBatchWithdrawal`. Both bounds **MUST** hold: the first bounds descendant-block fan-out per L1 action; the second bounds value at risk per batch.

**Per-input bound.** So that every input fits in a batch alone, the runtime **MUST** apply the batch bounds to each input in isolation, at §10.6 step 5a: an input whose contract-emitted withdrawals exceed `MaxBatchWithdrawal` for any asset, or whose withdrawal assets together with its deposit asset number more than `MaxAssetsPerBatch`, **MUST** yield `RUNTIME_FAULT` with full rollback and a full deposit refund. Without it, a contract could emit one withdrawal above the cap — Phase 1 checks no withdrawal against a balance (§18.3) — or withdrawals across more than `MaxAssetsPerBatch` assets, and §27.6's requirement to split activity across batches would have no satisfying assignment, stalling the domain. The deposit amount does not count toward the per-asset total, for the reason the refund is exempt below, so step 5a never limits a deposit's size. The deposit's asset does count toward the `MaxAssetsPerBatch` footprint, because every deposit contributes a `depositCredit` to the batch's `AssetFlowSummary` (§18.4, §27.6).

Step 5a **MUST** read the input's domain's `MaxBatchWithdrawal` for each asset (§25.2) from Settlement state at the end of the momentum that confirms the input's receive — the first momentum whose state holds the input's `(domainId, ordinal)` entry (§4.6) — so every node derives it from L1 state alone. That momentum precedes every registration of the input's withdrawals, and the value never decreases while the domain is registered (below), so the value Settlement applies at registration is never lower. An asset with no `MaxBatchWithdrawal` configured for the domain has a cap of `0`: Settlement rejects registering a withdrawal in it, so step 5a **MUST** fault any non-zero contract withdrawal of such an asset rather than let it strand. A zero amount passes: it strands nothing, and Settlement rejects relaying it (§17.3). Because the value never decreases, a batch builder that bounds a batch's per-asset aggregate by the highest value step 5a used for that asset among the batch's inputs never exceeds the value Settlement applies at registration. Settlement enforces the batch aggregate against the value in force when it registers the batch's withdrawals. Core **MUST** reject any `SetAssetConfig` that lowers a (domain, asset)'s `MaxBatchWithdrawal` for as long as the domain is registered, whatever its `status` (§23): pausing a domain does not lift the rule, so the value in force at registration is never below the value an input was bounded against: every input that passed step 5a stays batchable, and every withdrawal of a `FINALIZED` batch stays registrable. Pause (§23, §27.10) is the emergency lever for reducing at-risk value.

**Runtime-emitted deposit refunds (§18.5) are excluded from the `MaxBatchWithdrawal` aggregate.** The cap exists to bound fraud loss from contract-emitted withdrawals; a deposit refund returns value that entered custody in the same batch and therefore cannot increase at-risk value. Conservation (§18.3) still bounds all refund withdrawals. Without this exemption, an attacker can spray cheap failing deposits to push a domain's per-batch refund total over the cap and stall batch production indefinitely — blocking return of already-custodied funds.

**Refunds are bounded by the batch's deposits.** A `RelayMessage` registering a refund — a `kind = 1` message with `source_contract = bytes32(0)` (§27.4) — **MUST** be rejected when it would take the batch's registered refunds for its asset above the batch's `depositCredit` for that asset (§18.4); an asset absent from the batch's `AssetFlowSummary` has `depositCredit = 0`. Settlement therefore keeps two running totals per (domain, batch, asset): registered contract-emitted withdrawals, checked against `MaxBatchWithdrawal`, and registered refunds, checked against `depositCredit`. An honest runtime meets the refund bound by construction, because each refund returns part of a deposit made in the same batch and asset. Without the bound, an executor could label any withdrawal a refund and bypass `MaxBatchWithdrawal`, leaving only conservation to bound it. The bound does not tie a refund to its depositor: a fraudulent batch can still send up to its own `depositCredit` to recipients of its choosing, so its loss bound is `MaxBatchWithdrawal + depositCredit` per asset (§18.3).

### 27.10 RelayMessage under pause

**Normative:** `PAUSE_SUBMIT` halts `SubmitBatch`; `PAUSE_RELAY` halts `RelayMessage` registration and `ReleaseWithdrawals` release, and, while in force for any domain or globally, fee settlement by `Update` and `CollectReward` (§22.4). Each `Pause` call names exactly one scope, and `PAUSE_RELAY` is never implied by `PAUSE_SUBMIT`. `RelayMessage` and `ReleaseWithdrawals` for withdrawals from already-`FINALIZED` batches, and fee settlement and collection, **MUST** remain processable while only `PAUSE_SUBMIT` is active — halting submission **MUST NOT** freeze withdrawals already past the delay. `PAUSE_RELAY` **MUST** be a separately authorized escalation for an active asset-safety exploit, recorded as its own durable, publicly readable pause-state field, independent of `PAUSE_SUBMIT`. Neither scope **MUST** ever rewrite history or alter roots/receipts/balances/conservation counters/canonical order. On `PAUSE_RELAY` deactivation, queued withdrawals **MUST** again be releasable in unchanged registration order: the release queue is drained front-to-back through a per-domain head/tail cursor pair, and pause writes only pause state, so order is preserved by construction.

### 27.11 Enumerated field encodings

**Normative:** every enumerated field is a `u8` with the encoding below. Values not
listed **MUST** be rejected. Every Phase-1 default is `0`.

| Field | Encoding |
|---|---|
| `domainClass` (§6) | `EXECUTION = 0`, `BRIDGE = 1`, `MESSAGING = 2` |
| `runtimeKind` (§6) | `WASM = 0` |
| `inputSource` (§6.1) | `L1_NATIVE = 0`, `L1_RELAYED = 1`; `2` is retired and **MUST NOT** be reassigned |
| `execProfile` (§6.3) | `ATTESTATION = 0`, `OPTIMISTIC = 1` (reserved ordinal), `ZK_VALIDITY = 2` |
| `foreignProfile` (§6.4) | `NONE = 0`, `COMMITTEE = 1`, `LIGHT_CLIENT = 2`, `ZK_CONSENSUS = 3` |
| `proposerPolicy` (§6.2) | `SINGLE = 0`; `1` and `2` unassigned |
| `finalityModel` (§6.3, §21) | `DELAY = 0`, `CHALLENGE_WINDOW = 1` (reserved ordinal), `PROOF_VERIFIED = 2` |
| domain `status` (§6) | `ACTIVE = 0`, `PAUSED = 1` |
| batch `state` (§21) | `SUBMITTED = 0`, `FINALIZED = 1` (derived from `finalizeAtHeight`, never stored) |
| executor `status` (§22) | `ACTIVE = 0`; `1` and `2` unassigned |
| pause `scope` (§23) | `PAUSE_SUBMIT = 0`, `PAUSE_RELAY = 1` |
| `verifierKind` (§5.5) | `FRAUD_REFEREE = 0` (reserved ordinal), `VALIDITY = 1`, `FOREIGN_CONSENSUS = 2`, `FOREIGN_COMMITTEE = 3` (`1` to `3` reserved) |
| contract `upgrade_policy` (§12, §27.12) | `IMMUTABLE = 0`, `OWNER_UPGRADEABLE = 1` |
| `DAMode` (§20) | `0` = best-effort off-chain publication (§20); the only Phase-1 value |

Each ladder is numbered in the order its section presents it, so a higher value is
never a weaker rung.

A value marked *reserved ordinal* names nothing that any profile, model, status or kind uses: Core **MUST** reject it as it rejects an unlisted value, and it **MUST NOT** be reassigned.

### 27.12 Contract metadata encoding

```
ContractMetadata := metadata_version : u16 ; upgrade_policy : u8 ; owner : Address(20)
```

**Normative:** `metadata_version` **MUST** be `1`, and a version-1 encoding is exactly 23 bytes with no trailing bytes. `upgrade_policy` **MUST** be a §27.11 value. `owner` **MUST** be the zero address when `upgrade_policy = IMMUTABLE` and non-zero when it is `OWNER_UPGRADEABLE`. The runtime **MUST** check all four at `DeployContractFinalize` (§11.3); any failure yields `VALIDATION_FAILED` with no contract created. `metadata_hash = SHA3-256(ContractMetadata)` (§11.1). A later version **MAY** append fields; it **MUST** fit `MaxContractMetadataSize` and **MUST** keep the version-1 prefix.

---

## 28. Codebase foundations

Phase 1 reuses shipped, consensus-tested go-zenon patterns. Symbols (not line numbers) are cited so the references survive refactors.

- **Embedded-contract extension by spork layering** — `getOrigin()` + `applyXDiffs` gated by `context.IsXSporkEnforced()` in `vm/embedded/embedded.go` (`GetEmbeddedMethod`); the `Method` interface `GetPlasma`/`ValidateSendBlock`/`ReceiveBlock`. Precedent: `applyHtlcDiffs` / `IsHtlcSporkEnforced`.
- **Spork release flow** — placeholder hash → `CreateSpork` → real id → `ActivateSpork` → enforcement, with test/devnet override of `ImplementedSporksMap` (`common/types/spork.go`; precedents `HtlcSpork`, `DynamicPlasmaSpork`, `StateRootSpork`). `types.OffchainExecutionSpork` is added here as a contract-activation spork (no momentum-version bump).
- **Embedded addresses** — a new `z1qxemdeddedx…` value via `parseEmbedded`, listed in `EmbeddedContracts` (`common/types/address.go`).
- **Automatic anchoring** — embedded state writes fold into `ChangesHash` (`block.ChangesHash = db.PatchHash(changes)`, `vm/supervisor.go`) → momentum hash (`chain/nom/momentum.go`) → pillar signature → re-verified by every node, no format change.
- **Deterministic input ordering, auto-receive, descendant sends, unlimited embedded plasma** — `vm/vm.go`, `chain/momentum/ledger_store.go`, `pillar/worker_contract_generator.go`.
- **Canonical ordering source** — `AccountHeader.Bytes() = Address ‖ Height ‖ Hash` (`common/types/account_header.go`); `MomentumContent` sorts by `bytes.Compare(a.Bytes(), b.Bytes())` (`chain/nom/momentum_content.go`).
- **Pillar-bound executors + time-windows** — the pillar registry read through `context.MomentumStore().GetActivePillars()` (precedent `vm/embedded/implementation/accelerator.go`) for executor identity, `RewardWithdrawAddress` and activity (§22.1); `TimeChallenge` (`vm/embedded/implementation/common.go`) as the template for time-locking administrator changes.
- **Epoch reward settlement** — `Update` (§22.4) follows `UpdateEmbeddedPillarMethod` / `checkAndPerformUpdateEpoch`, and Settlement is listed in `types.EmbeddedWUpdate` so `pillar/worker_updater.go` cranks it automatically. Rewards reuse the `RewardDeposit` bookkeeping of `vm/embedded/implementation/common.go`, but `CollectReward` pays from Settlement's balance instead of minting through the token contract.
- **Permissionless rate-limited cranks** — the `ReleaseWithdrawals` method (§18.2) follows `UpdateEmbeddedAcceleratorMethod` / `checkAndPerformUpdate` (`vm/embedded/implementation/accelerator.go`, `common.go`): anyone may call, rejected within `ReleaseIntervalMomentums` of the domain's last accepted call, pays out via descendant sends.
- **Shared SMT core (consensus-frozen hashing/routing/proof bytes)** — `common/trie/hash.go` (`LeafHash`, `InternalHash`, constant-zero `emptyHash`, MSB-first `pathBit`), `compute.go` (full-depth root computation), and `proof.go` (`encodeProof`/`decodeProof`/`reconstructRoot`). The L2 executor uses the path-native API — `RootOfLeaves`, `ProveByPath`, `VerifyProofByPath`, `VerifyAbsenceByPath` (§13.1) — for all root computation, proof generation, and verification. The key-hashing verifiers `VerifyProof(root, key, …)`/`VerifyAbsence(root, key, …)` and the maintenance adapter `nodestore.go` (`NodeTree.Update`/`Commit`/`Prove`) are **L1-only** and **MUST NOT** be reused by L2 (§13.1). Conformance-gated by `conformance_test.go` against `smt-v1-test-vectors.json`.
- **Constants** — `vm/constants/plasma.go` (`MaxDataLength`), `vm/constants/embedded.go` (`MomentumsPerEpoch`, `MinSoftDelay`, `MinAdministratorDelay`, emission split). Settlement constants and hard-bound predicates live in `vm/constants/settlement.go`.

---

## 29. Build sequencing

- **1A — Off-chain executor + conformance (no chain changes).** Runtime + deterministic profile (§7), validation + instrumentation pipelines (§11), three-import host ABI (§8), the shared SMT core (§13) consumed **only** through the path-native API (§13.1) with the L2-specific application semantics (§13.5–§13.6) and the canonical applier (§13.5.1), canonical ordering off `MomentumContent` (§4.4), the canonical serializer (§14, §27, §27.1a); gated by the SMT (§13.7, incl. the path-native and applier vectors) and execution (`execution-conformance-v1.json`) vector suites. *Retires the deterministic-instrumentation risk first.*
- **1B — Spork-gated Settlement contract.** All §5.3 methods (domain-agnostic core + WASM handler), domain registry + lifecycle (§6), chunked deployment (§11), per-(domain, asset) custody + conservation (§18), delay + bounded descendant sends (§21, §27.9), Core/Periphery (§5.2), pause (§23). Ship dark; testnet.
- **1C — Executor wiring and the gateway, testnet activation.** `RegisterDomain` (WASM), register the pillar executor, wire `SubmitBatch`, stand up the gateway's HTTP bundle serving by `DAHash` (§20, `EXECUTOR.md` §21.5), activate the spork on testnet.

---

## 30. Open / deferred items

Items left open at the close of Phase 1 — the pre-mainnet parameters, the gas terms `OD-4` and `OD-5`, the pre-mainnet implementation tasks and the optional landing-zone-balance extension — are carried in `../phase-2/SPEC.md` §11, which owns their resolution. This section keeps only what stays reserved in the Phase 1 schema.

**Phase 2 (not in Phase 1):** mainnet requires `ZK_VALIDITY`; its design is in `../phase-2/`. The multi-executor set and inter-batch outbox ordering are also outside Phase 1.

**Bridge framework (schema present, activation reserved).** The generalized domain model (§6), the two profile ladders (§6.3, §6.4), custody modes and the `MINTED` conservation predicates (§18), the peg-out authorization + `ReleaseAdapter` obligations (§6.6), and the reserved timeout/ack messaging (§17.4) are normative here but constrained to their Phase-1 defaults until a spork-gated Core release activates them. The guide `../research/bridges.md` carries worked peg-in/peg-out examples per chain, comparative mappings (IBC/LayerZero/Wormhole/CCIP/Axelar/XCM/zkBridge), and the testing/formal-verification/deployment strategy. Activation prerequisites: registered verifiers for the profiles a domain uses (§5.5), `protocol_version = 2` commitments and representation-asset issuance for `MINTED` (§18.4a), and the per-chain `ChainVerifier`/`SourceAdapter`/`ReleaseAdapter` (the last off-Zenon).

---

## 31. Conformance vectors

An implementation is conformant when it reproduces the companion vector suites byte-for-byte. These are normative companion artifacts (§13.7 and the Conventions intro).

Three suites live in three files, each named with a path a reader can follow:

**SMT / trie (`common/trie/testdata/smt-v1-test-vectors.json`, `proof-format.md`):**

The vector file carries `SMT-001`…`SMT-014` and four requirement entries, `CV-PATH-1`, `CV-PATH-2`, `CV-APPLY-1` and `CV-APPLY-2`. An implementation **MUST** reproduce every root, leaf hash and proof byte in it. Each requirement entry states a property of an API or of the applier rather than of a single leaf set, so a test discharges it against the vector file or against the shared path-native core:

- **CV-PATH-1** — `VerifyProofByPath`/`VerifyAbsenceByPath` accept a vector key supplied as a path (no key hashing); the key-hashing `VerifyProof(root, path-as-key, …)` rejects the same path.
- **CV-PATH-2** — `RootOfLeaves(paths, values)` reproduces every `expected_root` in the file.
- **CV-APPLY-1** — one `StateDiff` carrying both a present-empty write (`L = 0x00000000`) and an `EMPTY` delete (`L = 0xFFFFFFFF`) yields the root of the leaf set that diff implies, and a present-empty leaf yields a different root from an absent one (§13.6).
- **CV-APPLY-2** — applying N diffs one at a time reaches the root of a single full recompute (§13.5.1); `SMT-014`'s `steps` field carries the sequence.

`../artifacts/smt-v1-test-vectors.json`, in the notes tree beside this document, is a byte-for-byte copy of the suite of record. The suite of record is `common/trie/testdata/smt-v1-test-vectors.json`, and that is the copy the conformance tests read.

**Execution (`../wasm-vm/execution-conformance-v1.json`, `../wasm-vm/execution-conformance-v1-README.md`).** The README is this suite's normative companion alongside the JSON, the same shape §13.7 gives `proof-format.md` beside the SMT suite. This suite's required coverage **is** the README's own coverage table (that file's **Coverage** section); the README's table is the complete coverage statement, and the ids this section names are required members of it. `CV-DEPLOY-1` (single-chunk deploy sizing, §11.2) lives in this suite, not the serialization suite.

Required cases the artifact carries:

- §10.6 step 5a: contract-emitted withdrawals across `MaxAssetsPerBatch + 1` distinct assets ⇒ `RUNTIME_FAULT` with metered gas and no effect charge; and a payable input whose withdrawals name `MaxAssetsPerBatch` assets and whose deposit asset is a further one ⇒ the same (`CV-ASSETS-1`, `CV-ASSETS-2`).
- Contract metadata, upgrade policy and ownership (§11.3, §12, §15, §27.12): an `OWNER_UPGRADEABLE` deploy followed by an authorized upgrade (`EXEC-DEPLOY-2`, `EXEC-UPGRADE-2`); a transfer, a renounce, and an upgrade after a renounce (`EXEC-TRANSFER-1`, `EXEC-TRANSFER-2`, `CV-UPGRADE-RENOUNCED-1`); a transfer by a non-owner (`CV-TRANSFER-UNAUTH-1`); a transfer naming no deployed contract (`CV-TRANSFER-UNDEPLOYED-1`); each malformed-metadata `VALIDATION_FAILED` case (`CV-DEPLOY-META-1`…`CV-DEPLOY-META-6`).
- A payable `CallL2` carrying a real deposit with a partial claim (`EXEC-DEP-2`, `EXEC-DEP-6`). The §18.4 batch-level `depositCredit`/`withdrawalDebit` identity for it belongs to the batch builder and is pinned there, not in this suite.
- `claimed_deposit` omitted from the blob ⇒ `RUNTIME_FAULT` (§27.1a, `CV-DECODE-5`).
- Storage quota (§13.8): a `CallL2` whose net state growth exceeds the quota ⇒ `STORAGE_QUOTA_EXCEEDED` with metered-plus-effect gas and a full deposit refund; a write within quota, whose committed `StateDiff` carries the updated `0x0005` leaf; a deletion that frees quota; a call whose writes net to `delta = 0`, whose witness set omits the `0x0005` leaf; a contract writing `0x0005` itself ⇒ `UNAUTHORIZED`; a write by a contract with no quota leaf; and `BuyStorage` success, undeployed target, and `purchased + bytes` overflow (`CV-QUOTA-1`…`CV-QUOTA-9`). Every vector whose contract writes state carries a quota leaf in its pre-state.
- Function-entry locals charge (§10.2): a function declaring `MaxLocalsPerFunction` locals pays its entry charge once (`CV-LOCALS-1`), and a budget one gas short of that entry segment ⇒ `OUT_OF_GAS` (`CV-LOCALS-2`).
- Memory limit (§7.4, §11.3): a module declaring a maximum of 1 page cannot grow; a module declaring no maximum, or one above `MaxMemory`, grows to exactly `MaxMemory`; a module declaring a maximum below `MaxMemory` grows to that maximum (`CV-GROW-2`…`CV-GROW-5`); and a module declaring more than `MaxMemory` initial pages is `VALIDATION_FAILED` at deploy (`CV-MEM-MIN-1`).
- Deploy validation (§8.1, §11.3, §25.1, §27.1): each structural cap one above its limit, and eight caps and both segment-fit bounds at their limit (`CV-STRUCT-*`); a data and an element segment past their bound (`CV-FIT-DATA-1`, `CV-FIT-ELEM-1`); the import rules, including an import's kind and signature (`CV-IMPORT-*`); a start section and custom sections (`CV-SECTION-*`); the export rules, including a `run` export naming an import (`CV-EXPORT-*`); a module under `MaxCodeSize` that instruments past `MaxInstrumentedCodeSize` (`CV-INSTR-SIZE-1`); and a module whose only defect is a LEB128 integer longer than its shortest encoding (`CV-LEB-PADDED-1`).
- Undeployed target (§10.6, §15, §18.2, §20): a `Deposit` and a payable `CallL2` naming a `target_contract` with no code-hash leaf ⇒ `VALIDATION_FAILED` at `gasUsed` 0 with a zero receipt `codeHash`, the full deposit refunded with `outboxCount = 1`, and a witness set of exactly the target's two absent code-hash leaves (`CV-DEPOSIT-UNDEPLOYED-1`, `CV-CALL-UNDEPLOYED-1`).

**Serialization / commitment / bundle (`../artifacts/serialization-conformance-v1.json`).** This suite is authored in zeno-executor's `internal/codec/testdata/serialization-conformance-v1.json`; the copy beside this document is a byte-identical snapshot of it, and no automated check enforces the equality. It holds exactly `CV-COMMIT-1/2`, `CV-INPUTROOT-1`, `CV-ASSET-1/2`, `CV-OUTBOX-1`, `CV-RESULT-1/2`, `CV-RESULT-OUTBOX-1/2`, `CV-MERKLE-1`, `CV-BUNDLE-1/2`, `CV-META-1/2`:

- **CV-BUNDLE-1** — two-input bundle, one absent read, fixed `DAHash` (§20).
- **CV-BUNDLE-2** — a kind-3 record carrying `deployChunkData` followed by a non-kind-3 record; fixes the §20 framing byte for byte.
- **CV-ASSET-1/2** — ZTS↔`AssetID` round-trip; reject a non-zero high byte / non-canonical alignment (§18.1).
- **CV-COMMIT-1/2** — batch commitment round-trip with empty `proofData`; reject non-empty `proofData` (§19).
- **CV-INPUTROOT-1** — mixed-kind batch (`CallL2` + `Deposit` + `Deploy*` + `Upgrade` + `TransferContractOwnership` + `BuyStorage`) `inputRoot` (§27.5); the `BuyStorage` leaf is a required addition. Its `proofs_hex` and `prefix_proofs_hex` pin the §27.3 `MerkleProof` of every leaf; `prefix_root` is a five-leaf tree whose last leaf is promoted at two levels, so its proof has one step.
- **CV-META-1/2** — accept a well-formed version-1 `ContractMetadata`; reject each malformed shape: unknown version, truncated, trailing bytes, an unlisted `upgrade_policy`, and each direction of the owner rule (§27.12).
- **CV-MERKLE-1** — reject a `MerkleProof` with a `side` of 2 at a right-sibling step and at a left-sibling step, one of `MaxMerkleProofDepth + 1` well-formed steps that folds to its root, one with a trailing byte, a leaf hash given in place of the element, and a proof against the empty-set root (§27.3).
- **CV-OUTBOX-1** — an `L1_WITHDRAWAL` message with a non-empty payload is rejected on decode (`non_empty_withdrawal_payload`, §27.4).
- **CV-RESULT-1/2** — accept a canonical `ExecutionResult`; reject it with a trailing byte, with its roots after `claimed_deposit` instead of after `globalInputIndex`, with its `StateDiff` in descending or duplicated derived-key order, and with an event's or outbox message's `input_index` other than its `globalInputIndex` (§27.0, §27.1a, §27.2). A swap of two equal-width fields re-parses as a different valid value, so no vector can pin it.
- **CV-RESULT-OUTBOX-1/2** — an `ExecutionResult` with `MaxOutboxPerExecution + 1` outbox messages, the last the deposit refund, is accepted; one with `MaxOutboxPerExecution + 2` is rejected (§27.4).

`CV-OUTBOX-1` is not unique across suites: it also exists in the execution suite, as the runtime-layer rejection of the same payload-bearing-withdrawal shape this suite rejects at the codec layer. Vector ids are unique within a suite, not across suites.

---

## 32. Implementation and deployment tasks

The companion artifacts (`proof-format.md`, `wasm-gas-table-v1.json`/`.md`, `execution-conformance-v1.json`) and the reference vault contract (§18.1a) are delivered. The remaining pre-mainnet tasks — parameters, deploy-time pricing, the `MaxBatchMint` bounds predicate, the reserved `AssetRegistry` schema, and the administrator-address migration rules — are carried in `../phase-2/SPEC.md` §11.
