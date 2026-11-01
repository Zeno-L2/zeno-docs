# Executor Binary — Architecture Specification

**Document status:** Architecture specification — normative for the executor binary, grounded against go-zenon and zeno-executor source.
**Version:** 0.12.3
**Upstream:** `SPEC.md` v1.17.3 (the on-chain protocol; governs on any conflict).

> This document specifies the **off-chain executor binary**: one generic runtime that serves any number of execution domains as plugins in the **executor** role (the **relay** role is reserved for externally sourced domains), and the **gateway** that serves its data to clients (§21). It does not redefine any on-chain rule; where it touches the on-chain surface it cites `SPEC.md`. It is written so a developer can build from it, so every external dependency is graded **[EXISTS]** (located in source), or **[PREREQUISITE]** (must be built first).

The governing invariants are inherited from `SPEC.md`:

> **Consensus orders. Executors compute. Settlement anchors.**
> **The runtime is generic. The domain is a plugin. State is always replayable from L1-anchored data.**

---

## Conventions

RFC 2119 / RFC 8174 keywords as in `SPEC.md`. Tag convention:

- **[CORE]** — a requirement imposed by `SPEC.md`; the binary **MUST** satisfy it regardless of this document.
- **[BINARY]** — a requirement this document adds for the executor component; local to the binary, touches no consensus byte.
- **[EXISTS]** — grounded in existing source (repository and path cited).
- **[PREREQUISITE]** — an upstream dependency that does **not** yet exist in source, or exists only in part, and **MUST** be completed before the dependent capability functions; where a row carries a Status cell, that cell states which. The binary **MUST NOT** assume it exists.

`>`-blockquotes are *Informative*.

---

## 1. Scope

### 1.1 What the binary IS

A deterministic, off-chain execution engine for Project Zenon off-chain execution domains. One binary:

- consumes the canonical L1 input stream from a `znnd` source and reconstructs each domain's input sequence (`SPEC.md` §4);
- executes a domain's deterministic state-transition function (STF) through a **domain plugin**, maintaining an SMT via the path-native trie API (`SPEC.md` §13);
- produces `ExecutionResult`s, DA bundles, `DAHash`, and batch commitments (`SPEC.md` §14, §19, §20);
- runs the **executor** role (§4); the relay role is reserved;
- recovers deterministically after a crash, reconciling against finalized on-chain roots (§14).

### 1.2 What the binary is NOT

- It is **not** the Settlement embedded contract (that is L1, `SPEC.md` §5).
- It does **not** define protocol behaviour. Ordering, conservation, finality, and wire formats are `SPEC.md`'s.
- It does **not** produce validity proofs. Mainnet requires `ZK_VALIDITY` (`SPEC.md` §1, §6.3), whose design is in `../phase-2/`.
- Unikernel packaging is **optional**; a plain Linux build is fully conformant. Hardened packaging is a deployment concern, not architecture.

### 1.3 Relationship to upstream

| Upstream | Relationship |
|---|---|
| `SPEC.md` v1.17.3 | Defines every consensus-visible byte. This binary implements §4 (ordering / input sequence), §6.1 (input source), §6.2 (executor set / proposer), §7–§8 (runtime), §13 (SMT, including the §13.8 storage quota), §14/§27 (serialization), §19 (commitment), §20 (DA), §22 (executor model). **Core spec governs on any conflict.** |
| `common/trie` | The shared SMT core. The binary uses the **path-native API** (`RootOfLeaves`, `ProveByPath`, `CompactTree`, `VerifyProofByPath`, `VerifyAbsenceByPath`) and **MUST NOT** call the key-hashing L1 verifiers. **[EXISTS]** — see §18. |

### 1.4 Packaging, code sharing, and plugin linking

**[BINARY] The executor is a separate repository and Go module** (e.g. `github.com/zenon-network/zenon-executor`), built and operated on its own node, on its own release cadence. It is **not** a `cmd/` entry point inside go-zenon. It pulls go-zenon in as a **pinned module dependency** (`require github.com/zenon-network/go-zenon vX.Y.Z`) and imports only the consensus-frozen code it must not reimplement:

- `common/trie` — the shared SMT core, via the path-native API (§5.3);
- `common/types` — `Address`, `Hash`, `AccountHeader`, and related primitives (with `chain/nom` for `AccountBlock`/`Momentum`);
- `vm/embedded/definition` (Settlement) — the ABI (`ABISettlement`, `vm/embedded/definition/settlement.go`) to decode inputs and encode `SubmitBatch`/`RelayMessage`, and the exported storage-key builders the read path uses (§5.5);
- `wallet` and `common/crypto` — the executor key pair and account-block signing (§5.5);
- `vm/constants` and `dp`: the Settlement protocol limits (the gateway, the supervisor and the WASM reader) and the plasma price the supervisor uses to fuse plasma for a send.

It imports no `rpc/*` package: its JSON-RPC client is its own thin client over `net/http`, decoding into `chain/nom` types, so the node's server and API packages stay out of the executor build.

It reads momenta and Settlement storage from a `znnd` node over JSON-RPC (§5.1, §5.5); it does not embed or run L1 consensus.

"Import directly" means *use the same compiled package*, not co-locate. This is how the executor meets the shared-library requirement of `SPEC.md` §13.1: L1 momentum roots and L2 executor roots are byte-identical because both sides run identical `common/trie` code. `common/trie` therefore lives in go-zenon, which needs it for the momentum state root, and the executor is a *consumer* of it. A second implementation, such as the Rust SMT in the validity-proof program mainnet requires, is conformant only if it reproduces every vector of `smt-v1-test-vectors.json` (`SPEC.md` §13.1). The shared seam is the path-native API (§5.3); the executor **MUST NOT** import the L1 maintenance adapter (`nodestore.go`/`momentum_fold.go`), whose momentum-specific empty→delete fold and key pre-hashing differ from L2 (`SPEC.md` §13.6). The `common/trie` conformance vectors (`smt-v1-test-vectors.json`, gated by `conformance_test.go`) are the contract that keeps the two layers identical; the executor runs them in its own CI (§19).

**[BINARY] The dependency direction is one-way and load-bearing: `executor → go-zenon`, never the reverse.** go-zenon **MUST NOT** import the executor or any domain plugin. Keeping `common/trie` and the Settlement contract in go-zenon creates no back-dependency — they are L1 components — whereas a back-dependency *is* the real separation violation, and Core review **MUST** reject one.

**[BINARY] Domains are static, compile-time plugins.** A plugin is a Go package implementing the `Domain` (and, for external sources, `Source`) interface of §7. The domain-specific code lives outside L1 entirely — in particular the **WASM runtime** (interpreter, instrumentation, host ABI) lives inside the WASM plugin, never in go-zenon. Each plugin is linked into its own **domain worker** binary (§1.5), whose `main` runs the generic pipeline with exactly that one plugin:

```go
// cmd/worker-wasm/main.go — one worker binary per domain plugin
worker.Main(newPlugin, newSimulator)      // the WASM L1_NATIVE domain
// cmd/worker-bitcoin: worker.Main(newPlugin, newSimulator)  — a future L1_RELAYED domain
```

**A release builds a worker for every domain the network has adopted, and every executor runs all of them** (`SPEC.md` §6.2). Adding a domain is a release plus its activation spork, and then a `workers` entry for it in each executor's configuration (§17); the supervisor does not compare that list with Settlement's domain registry. Workers are built statically, so the binary stays type-safe, deterministic, and conformance-testable, with no dynamic-loading fragility. Each plugin **MAY** be its own module/repo; Phase 1 may start with the WASM plugin as a package in the executor repo and extract it when a second domain appears — the boundary that matters is the `Domain` interface, not the repo line.

> *Informative.* Pinning go-zenon as a dependency pulls in its module graph and couples the executor to go-zenon's package layout; this is acceptable (Go compiles only what is imported). If the coupling later becomes a burden, extract a lean shared module (`zenon-trie`/`zenon-types`) imported by both — a refactor to reach for only if needed, never pre-emptively. Dynamic plugin loading (`.so` or wasm-sandboxed) is out of scope until third parties must ship domains without a release (Phase 2), and trades determinism guarantees for that flexibility. Process isolation (§1.5) needs none of it: each worker is a statically built binary.

### 1.5 Process isolation

Every executor runs every domain (`SPEC.md` §6.2). Without isolation, one faulty plugin — a panic, a leak, a hang — would stop every domain on every executor at once. Each domain therefore runs in its own operating-system process.

- **[BINARY]** The binary **MUST** run as one **supervisor** process and one **domain worker** process per domain. A plugin **MUST NOT** run inside the supervisor or inside another domain's worker.
- **[BINARY]** The **supervisor** follows the `znnd` frontier and serves confirmed momenta to every worker on request, holds the single-instance lease (§13), holds the executor key, and sends every Settlement transaction (§5.5). The supervisor **MUST NOT** serve a client API, and it opens no listener reachable from outside the host: bundle serving and client-facing reads are the gateway's (§5.6, §21).
- **[BINARY]** A **domain worker** is the generic pipeline (§5.1–§5.4) linked with exactly one plugin. It owns its domain's state store, bundles and cursor (§16). Its STF runs in-process with its plugin, so state access never crosses a process boundary.
- **[BINARY]** A worker **MUST NOT** hold key material or be able to read it. The supervisor never passes a worker the key, its source or its location, and where the host allows it, runs each worker as its own user, so the worker cannot read the key file or the supervisor's memory. A worker hands the supervisor an unsigned commitment and a reference to its persisted bundle; the supervisor signs and sends.
- **[BINARY]** Workers talk only to the supervisor, over a local channel such as a Unix-domain socket. Nothing a worker sends is consensus-visible until the supervisor submits it, and the supervisor **MUST** reject a malformed message rather than repair it.
- **[BINARY]** A worker's crash, hang or resource exhaustion **MUST NOT** affect the supervisor or any other worker. The supervisor tells the ends of a worker apart by its exit status. A worker that halts (§14) is not restarted: its domain stays stopped until operator action, and every other worker continues. A worker that rejects its configuration is not restarted. A worker that exits because the supervisor could not serve its input, or because another process holds its domain directory, is restarted with bounded exponential backoff for as long as the condition lasts. Any other end is a **failure**: another exit status, termination by a signal, a resource-limit kill, a missed liveness deadline, or a malformed message. The supervisor **MUST** restart a failed worker with bounded exponential backoff, and the restarted worker resumes through the recovery state machine (§14). The supervisor **MUST** stop restarting a worker that fails repeatedly without progress, meaning it neither commits a new position nor stays up for a set period, and report its domain as `HALTED`. It **SHOULD** enforce per-worker memory and CPU limits.

- **[BINARY]** The **gateway** (§21) is a separate binary, neither a worker nor supervised by the supervisor. It holds no key, writes nothing in a worker's directory, and its crash or overload **MUST NOT** affect the supervisor or any worker.
- **[BINARY]** A domain worker binary also runs in **simulate mode** as a child of the gateway (§21.8). In that mode it is not a domain worker: it opens no domain directory, persists nothing and never talks to the supervisor.

> *Informative.* Keeping the key in the supervisor means a compromised or buggy plugin cannot sign anything, and a stalled domain costs its executor only that domain's fees (`SPEC.md` §22.4). Keeping client traffic out of the supervisor means an exposed endpoint cannot reach the process that holds the key.

---

## 2. Principles

1. **[BINARY] Generic runtime, plugin domain.** Everything except the STF is written once. A domain contributes a pure STF, input decoding, a genesis state, and (for external sources) a relay source adapter — nothing else (§7).
2. **[CORE] L1-anchored replay.** L2 state is a pure function of L1-anchored data (canonical inputs + commitments + DA bundles). The binary **MUST NOT** read a live external source to *reconstruct* state; external reads exist only to *propose* (§8, §10).
3. **[CORE] Purity of the STF.** `Apply(state, input)` is deterministic and side-effect-free with respect to consensus: no clock, no RNG, no network, no ambient environment. Every external fact arrives as a decoded input (`SPEC.md` §7.3, §9). A plugin **MAY** read from and write to a node-local, content-addressed store as a memoization tier, provided all of the following hold: every entry read is verified against a hash taken from state or from the input before use, and a miss or a verification failure is an `error` (halt), never a receipt status; every entry written is a pure deterministic function of state and input; and no entry is ever observable in `Effects`, in a root, or in the witness set. Under these conditions the store changes only *how fast* `Apply` runs, never *what it returns*, so replay from L1-anchored data alone (§10) is unaffected.
4. **[CORE] Uniform commitment envelope.** The committed batch format (`SPEC.md` §19) is identical across domains; the STF differs, the wire format does not. This keeps Settlement runtime-agnostic.
5. **[BINARY] Fail-closed.** Any internal consistency fault (root mismatch, `DAHash` mismatch, corruption) halts the affected domain's worker rather than submitting (§15); other domains' workers continue (§1.5).

---

## 3. System context

```
   L1 (Zenon, via znnd)                Executor binary (this doc)             Off-chain
   ┌──────────────────────┐  inputs   ┌──────────────────────────┐  bundle  ┌──────────────┐
   │ Settlement contract  │◀────────▶ │  generic runtime         │────────▶ │ Gateway      │
   │ (SPEC.md §5)         │  Submit   │   + domain plugin(s)     │  DAHash  │ HTTP bundles │
   │  registry · custody  │  Batch    │   one worker per domain  │          └──────────────┘
   │  commitments         │◀────────  │                          │
   └──────────────────────┘  reads    └──────────────────────────┘
        ▲   getDetailedMomentumsByHeight / getStateRoot / getProof   (rpc/api/ledger.go)
        └── relay role posts external data as Settlement account-blocks (L1_RELAYED domains)
```

---

## 4. Roles

There are two roles. The executor runs every domain the network has adopted, one worker per `workers` entry (§1.4, §17). The relay is a separate, lighter path that only `L1_RELAYED` domains need, and it is reserved. The binary has no role setting: it always runs the executor role.

| Role | Does | Registration | Runs the STF? | Needed for |
|---|---|---|---|---|
| **Relay** | Watches an external source and posts its data to L1 as Settlement account-blocks | None (permissionless) | No — validation happens later, inside the STF | `L1_RELAYED` domains only (reserved) |
| **Executor** | Consumes the input sequence, runs the STF, builds and submits batch commitments | A registered pillar, serving every domain (`SPEC.md` §6.2, §22) | Yes | Every domain |

**[BINARY]** One binary serves every domain, each in its own worker process (§1.5). Protocol-level domain isolation is guaranteed on-chain by per-`(domain, asset)` keying (`SPEC.md` §6); process isolation is fault containment, not a consensus guarantee.

> An `L1_NATIVE` WASM domain (Phase 1) has no relay: users post inputs to Settlement directly. The relay role exists only once `L1_RELAYED` domains are activated (`SPEC.md` §6.1, reserved in Phase 1).

---

## 5. Layered runtime

The runtime is a stack of domain-agnostic layers; the domain plugin attaches only at the STF driver and (optionally) the relay path.

### 5.1 Input pipeline
**[CORE]** Follows the `znnd` confirmation frontier, reads confirmed momenta (the supervisor reads them from `znnd` and serves them to every worker, §1.5), filters content to the worker's `domainId`, reconstructs the domain's input sequence in the order fixed by L1 (`SPEC.md` §4.4), and maintains the per-domain input-sequence cursor (`SPEC.md` §4.5).

- **[EXISTS]** `LedgerApi.GetDetailedMomentumsByHeight(height, count)` (`rpc/api/ledger.go:456`) returns momentum content `{Momentum, []*AccountBlock}`; `GetMomentumsByHeight` returns headers (with the v3 `StateRoot`).
- **[EXISTS]** Canonical order is L1-verifiable with no new derivation: `AccountHeader.Bytes() = Address‖Height‖Hash` (`common/types/account_header.go:50`) and momentum content is sorted by `bytes.Compare(a.Bytes(), b.Bytes()) <= 0` (`chain/nom/momentum_content.go:58`) — note the comparator is `<= 0`; account headers are distinct, so the ordering is total — matching `SPEC.md` §4.4.
- **[EXISTS]** The Settlement contract **address** (`types.SettlementContract`, `common/types/address.go`, via `parseEmbedded` and listed in `EmbeddedContracts`) identifies Settlement-destined account-blocks, and the **ABI** (`ABISettlement`, `vm/embedded/definition/settlement.go`) decodes them.
- **[CORE]** The pipeline reads Settlement's own assignment rather than re-evaluating acceptance (`SPEC.md` §4.6). For each confirmed momentum the supervisor proves each served domain's received-input count and the `(domainId, ordinal)` entry of every new ordinal — `{globalInputIndex, sourceAccountBlockHash}` — and serves them to the worker. The worker finds each named send in the momentum content it has followed, ABI-decodes it and re-encodes the canonical input (`SPEC.md` §27.5). A named send it cannot find or decode, or one naming another domain, halts the worker. Nothing here is filter state: everything comes from L1 state at the momentum, so resume and reorg rollback stay momentum-granular (§16).
- **[BINARY]** Assignment reads, in detail:
  - A momentum whose content holds no `SettlementContract` account-block adds no input; neither side proves anything for it.
  - Otherwise the supervisor proves each domain's received-input count at that momentum, and the entry of each ordinal from the worker's next expected ordinal up to that count. The worker's cursor persists the next expected ordinal per domain and rolls back with it on a reorg; an assignment that does not start there halts.
  - Each entry the supervisor serves also names the height of the momentum holding the send, from the node's account-block lookup. The worker fetches that momentum if it has not followed it and finds the send by hash in its hash-checked content, so the height adds no trust in the node.
  - A momentum below version 3 that holds a Settlement account-block halts: it cannot be proven (`SPEC.md` §5.1 orders the sporks so this never happens).
- **[BINARY]** Withdrawal caps: step 5a looks up each withdrawn asset's cap at the momentum confirming the input's receive through a lookup bound to that momentum; the worker memoises answers and the supervisor caches them in a fixed-size cache. The batch builder bounds each asset's aggregate by the highest cap step 5a used for it among the batch's inputs (`SPEC.md` §27.9).
- **[EXISTS]** The ABI declares every `SPEC.md` §5.3 method, each input method with `domainId uint32` first. The ordinal entry names its send (`DomainInputOrdinal.SourceAccountBlockHash`, `vm/embedded/definition/settlement.go:773`), and the key builders the read path uses are exported.

For `L1_NATIVE`/`L1_RELAYED` domains the input sequence number equals the `globalInputIndex` (`SPEC.md` §4.5); the pipeline hands the plugin the canonical input encoding (`SPEC.md` §27.5) and an `InputContext` to decode.

**[CORE]** `globalInputIndex` is assigned over the **global** L1 stream *before* domain filtering; filtering does not renumber. A domain's batch is a contiguous slice of **that domain's subsequence**, so consecutive consumed indices may jump (e.g. domain A sees `{5, 7, 9}`, `SPEC.md` §4.6). The pipeline **MUST** track each domain's last-consumed `globalInputIndex` independently and **MUST NOT** require `nextGlobal == previousGlobal + 1`.

### 5.2 STF driver
**[CORE]** The execution loop. Feeds decoded inputs to the plugin's `Apply` one at a time, supplies a **witnessing** state accessor, collects the returned effects, computes `preStateRoot`/`postStateRoot`, and slices **each domain's subsequence** into batches that are contiguous **within that subsequence** up to `MaxBatchInputs` — a batch begins at the successor of the domain's own cursor, never at `globalIndex + 1` (`SPEC.md` §4.6, §10, §14).

- **[CORE]** The driver **MUST** decode and apply each input in turn and **MUST NOT** batch-decode a bundle ahead of `Apply`, because a plugin's `DecodeInput` may mutate node-local state that a not-yet-applied earlier input still reads (D30).
- **Witnesses are captured here automatically** — the accessor witnesses every read, including absent observations; the driver forms the input's witness set as the union of the paths read through the accessor and the paths written in the returned `Effects.StateDiff` (`SPEC.md` §13.6, §20). Writes do not pass through `StateAccess`, which is read-only.
- **[EXISTS]** The deterministic WASM runtime — validator, instrumentation, gas table, engine, and three-import host ABI, with an `Apply` for input kinds 0–7 (`plugins/wasm/`).
- **[EXISTS]** The storage quota (`SPEC.md` §13.8: the reserved `0x0005` leaf, §10.6 step 6a, `STORAGE_QUOTA_EXCEEDED`) and `BuyStorage` (`input_kind = 7`), at `statediff_version = 1` and `receipt_version = 1` (`plugins/wasm/quota.go`).

### 5.3 State store
**[CORE]** A Sparse Merkle Tree over the path-native trie API (`SPEC.md` §13.1), plus persistence and checkpointing.

- **[EXISTS]** `RootOfLeaves` (`common/trie/pathapi.go:39`), `ProveByPath` (`pathapi.go:49`), `VerifyProofByPath` (`common/trie/proof.go:235`), `VerifyAbsenceByPath` (`proof.go:254`), `LeafHash`/`InternalHash` (`hash.go:47`/`54`). The path-native functions are the primitives; the key-hashing `VerifyProof`/`VerifyAbsence` are thin wrappers over them. `NewCompactTree` and `CompactTree`'s `Insert`/`Delete`/`Root`/`ProveByPath` (`common/trie/compact.go`) are the incremental form. The store keeps one tree across writes, so a root or proof costs `O(log N)` rather than a rebuild.
- **[CORE]** The binary **MUST** use only the path-native functions and **MUST NOT** call any key-hashing or L1-maintenance entry point: the key-hashing `VerifyProof`/`VerifyAbsence` (`proof.go:270`/`276`, thin wrappers that pre-hash the key via `types.NewHash`), the `NodeTree` write-set methods `Update`, `AccumulateFrom` and `ComputeRoot` (`common/trie/nodestore.go:254`, `:352`, `:445`), or the `stagedApplier` write-set applier (`common/trie/momentum_fold.go:55`, which folds empty→delete). These are L1-momentum-specific and would either double-hash an already-derived path or collapse a present-empty leaf to a deletion — L1-only, `SPEC.md` §13.1, §13.5.1, §13.6.
- **[BINARY]** The store is a **cache** of deterministically derivable state; a snapshot is the cursor's only durable image, and boot halts when it is missing (§14, §16).

### 5.4 Commitment & bundle builder
**[CORE]** Assembles the canonical `ExecutionResult` per input, the per-batch `inputRoot`/`receiptRoot`/`eventRoot`/`outboxRoot`, the `AssetFlowSummary`, and the DA bundle hashed to `DAHash` (`SPEC.md` §14–§20). Format is uniform across domains; the plugin supplies only semantic content (which deposits/withdrawals/events occurred). **[BINARY]** The builder **MUST** verify the `DAHash` round-trip (reconstruct from the persisted bundle, confirm equality) before committing; mismatch → halt.

### 5.5 Settlement client
**[CORE]** Runs in the supervisor (§1.5). Builds and sends `SubmitBatch` / `RelayMessage` / `ReleaseWithdrawals` (the withdrawal-release crank, `SPEC.md` §18.2), and reads Settlement state (`SPEC.md` §19).

- **[EXISTS]** Read paths: `LedgerApi.GetProof(height, key)` (`rpc/api/ledger.go:371`) returns `{value, proof, root}` for reading Settlement storage via L1 state proofs; `GetStateRoot(height)` (`:359`) for the recovery anchor (§14). Both need version-3 momenta (`StateRootSpork` active). The state-tree key of Settlement storage key `k` is `0x03 ‖ SettlementContract ‖ 0x04 ‖ k`; the supervisor checks the returned root against the momentum header's `StateRoot` and verifies with `VerifyProofByPath(root, types.NewHash(rawKey), …)`, never `VerifyProof`. Every Settlement storage entity except processed-outbox membership is ABI-packed and decoded with `ABISettlement.UnpackVariable`; processed-outbox membership is the literal byte `0x01` under its key, not an ABI blob.
- **[EXISTS] (pattern)** Account-block construction/signing exists in `wallet/` (keys) and the embedded send pattern. The executor signs with its own executor address, never the pillar's producer key (`SPEC.md` §22.1). Fee settlement (`Update`, `SPEC.md` §22.4) needs nothing from the binary: pillars crank it automatically.
- **[EXISTS]** `types.OffchainExecutionSpork` (`common/types/spork.go`) gates activation, and the Settlement storage key layout is documented in the key-prefix table of `vm/embedded/definition/settlement.go`, and the same file exports the key builders the read path uses.
- **[BINARY]** Recovery anchor: the last batch of the domain's contiguous finalized prefix (every batch from `0` has `height ≥ finalizeAtHeight`), read at the confirmed frontier; nothing finalized is the genesis anchor.
- **[BINARY]** Signing: `submissionKeySource` yields a 32-byte ed25519 seed (hex); the supervisor derives the key pair with `wallet`, checks it against `executorAddress`, and zeroes the seed after use. One account chain serves every domain, so sends are serialised; the supervisor reads its account frontier before every send and never reuses an account height.
- **[BINARY]** Before any send, the supervisor checks that `pillarName` is an active pillar whose executor-set member names `executorAddress`; otherwise it warns and does not submit.
- **[BINARY]** `SubmitBatch`: submit only when the batch's `batchId` is the domain's `nextBatchId`, its first input is ordinal `consumedInputCount`, and its `preStateRoot` is the previous accepted `postStateRoot`. Acceptance happens at the receive, so the supervisor confirms it by re-reading the batch entry after the receive confirms. A chain batch at that `batchId` that differs from the local record is a consistency fault (§15).
- **[BINARY]** Withdrawals: the worker hands each committed batch's `kind = 1` messages to the supervisor with `Committed`. After the batch is `FINALIZED`, the supervisor sends one `RelayMessage` per message not yet in `processedOutbox` (skipping messages `SPEC.md` §17.3 rejects), then `ReleaseWithdrawals` for its own domains at most once per momentum while the queue is non-empty.
- **[BINARY]** Every send is recorded in an append-only submission log under `<storagePath>/supervisor/` and replayed against chain state at boot.

### 5.6 DA layer
**[BINARY]** The supervisor publishes no bundles. Bundle availability is the gateway's HTTP serving of bundles and receipt sidecars by `DAHash` (§21.5, `SPEC.md` §20, `DAMode = 0`): best-effort and not consensus-critical.

### 5.7 Role controller
**[BINARY]** Reserved. Phase 1 has one role (the executor, §4) and one proposer (`SINGLE`, §13), so the binary has no role controller.

---

## 6. Settlement-contract activation (registration pattern)

> *Informative — grounding for the on-chain side the binary talks to.* The Settlement contract is registered exactly as HTLC is: `GetEmbeddedMethod` (`vm/embedded/embedded.go`) builds the active contract map and applies spork-gated diff layers — `applySettlementDiffs` gated by `IsOffchainExecutionSporkEnforced()` (`vm/vm_context/spork.go`), mirroring `applyHtlcDiffs` / `IsHtlcSporkEnforced()`. The spork id follows the `HtlcSpork` placeholder-hash flow (`common/types/spork.go`), and the address follows `HtlcContract` (`common/types/address.go`). Administration gates on the single network administrator address (`types.NetworkAdmin`, `common/types/address.go`) by direct comparison and time-locks changes with the shared `TimeChallenge` helper (`vm/embedded/implementation/common.go`) at the fixed Core delays `MinSoftDelay` / `MinAdministratorDelay`; it uses no `SecurityInfo` and no guardian set. Registration and administration are shipped; what remains **[PREREQUISITE]** is the method surface the binary depends on (§5.5, §18).

---

## 7. Domain plugin interface

The plugin is the only code written to add a domain: a pure function plus decoding. It never sees Settlement, never computes a root, never builds a witness, never opens a socket inside `Apply`.

```go
// The execution contract every domain implements.
type Domain interface {
    Genesis() StateInit                                            // initial state (empty for WASM; SPV genesis for BTC)
    DecodeInput(ctx InputContext, payload []byte) (Input, error)   // opaque L1 payload + per-input context → typed, structurally-valid input
    Apply(state StateAccess, in Input) (Effects, error)            // pure STF; runtime supplies the witnessing accessor
}

// Per-input facts the runtime observes on L1 that the canonical input encoding does not carry.
type InputContext struct {
    GlobalInputIndex uint64
    MomentumHeight   uint64
    MomentumTime     uint64
    MomentumHash     Hash       // hash of the momentum at MomentumHeight; its producer chose its content, so it is not unbiased randomness
    L1Sender         Address    // raw L1 sender of the source account-block
    SourceData       []byte     // domain-defined source bytes the canonical encoding omits (e.g. deploy chunk bytes)
    Caps             CapLookup  // this input's withdrawal-cap lookup; a domain that emits no L1 withdrawal never calls it
}

// Optional: only L1_RELAYED domains implement this (the relay role).
type Source interface {
    Watch(ctx context.Context) (<-chan RelayPayload, error)
}

// Optional: the gateway's read adapter for this domain (§21.4). Pure decoding; never executes.
type Reader interface {
    ChainID() any                                                 // JSON-encodable chain_id
    DescribeInput(inputData []byte) (any, error)                  // decode a canonical input encoding (SPEC.md §27.5) for display
    Profiles() []Profile
    Views() []View                                                // named, typed reads served through zeno_read (§21.4)
}
```

`InputContext` is in-process only and commits nothing. A `Reader` is linked into the gateway, never into a worker, and its methods read only gateway state; a domain without one is still served by every generic gateway method (§21.3). `SourceData` is load-bearing — it is the only channel by which deploy chunk bytes reach a plugin.

Runtime-supplied types the plugin consumes but does not implement: `StateAccess` (`Read`/`Len`, backed by the SMT, witnessing every access including absences, read-only — writes do not pass through it); `Effects` — the shipped field set: `Status`, `GasUsed`, `ContractID`, `CodeHash`, `AbortCode`, `ClaimedDeposit`, `StateDiff`, `Events`, `OutboxMessages`, `ReturnData` (`domain/domain.go:46-69`). `AbortCode` carries the §7.5 abort code into the receipt's committed `abortCode` (`SPEC.md` §15).

| Concern | Owner |
|---|---|
| STF (`Apply`), input decode, input-canonicity predicate, genesis, SMT key layout, `ChainVerifier` choice (bridge domains) | **Plugin** |
| External source watching (`Source` / `SourceAdapter`) — external domains only | **Plugin** |
| Ordering, input-sequence cursor, witness capture, roots, batching, commitment/bundle, Settlement comms, proposer scheduling, profile verification | **Runtime / Settlement** |

**[BINARY]** The WASM specifics (`ExecutionInputFrame` write, `ContractEffects` validation, deposit refund `SPEC.md` §18.5) **MUST** live behind `Domain.Apply`/`DecodeInput`, even while WASM is the only implementation, so the generic runtime emerges from day one rather than being refactored out later.

**[BINARY] Bridge and messaging domains reuse this exact four-method interface.** A `BRIDGE`/`MESSAGING` domain (`SPEC.md` §6.5) is a `Domain` plugin whose `Apply` delegates to a shared **`BridgeLedger`** state machine (mint/burn/pool/conservation/peg-out/replay/fraud) with a per-chain **`ChainVerifier`** injected — the foreign-fact verifier implementing the domain's `foreignProfile` (`SPEC.md` §6.4, §6.6). `Genesis` returns the light-client anchor (`chainBinding.genesisAnchor`); `DecodeInput` decodes relayed `Header`/`Deposit`/`Redeem`/`Settle`/`Fraud` payloads; `Source.Watch` is the per-chain **`SourceAdapter`** (relay role). The foreign-side **`ReleaseAdapter`** that consumes a finalized `kind=1` peg-out authorization lives **on the foreign chain**, not in this binary (`SPEC.md` §6.6). Adding a chain is therefore a new `ChainVerifier` + `SourceAdapter` + `ReleaseAdapter` + asset wiring; the runtime, the `BridgeLedger`, and Settlement are untouched. The `ChainVerifier` **MUST** be pure (no live foreign reads inside `Apply`, §10) so replay reproduces state from L1-anchored data alone. Interface sketches and the shared `BridgeLedger` pseudocode are in the guide `../research/bridges.md` §4; reserved until the profiles activate (`SPEC.md` §6).

**[BINARY]** A plugin's SMT key layout **MUST** be expressed as a pure derivation to a 32-byte SMT path (`SPEC.md` §13.3 — `sha3(contract_id‖local_key)` for WASM); the runtime consumes **already-derived paths** and **MUST NOT** key-hash. This keeps the double-hash hazard (§5.3) out of the runtime as new domains are added.

---

## 8. Input-source handling

A domain declares its `inputSource` on-chain (`SPEC.md` §6.1). The binary handles each kind:

- **`L1_NATIVE`** — inputs are Settlement account-blocks; the input pipeline (§5.1) is the whole story. No relay. **Phase 1.**
- **`L1_RELAYED`** — external data is posted to Settlement as account-blocks by the relay role (§12); to the input pipeline these are ordinary L1 inputs with a `globalInputIndex`, so DA and ordering are L1-inherited and the STF validates canonicity. Reserved.

**[CORE]** For both kinds the executor **MUST NOT** reorder, skip, or privately insert inputs (`SPEC.md` §4.7). Every input is an L1 account-block, so completeness is the same per-domain input-sequence contiguity that gives force-inclusion (`SPEC.md` §4.6, §6.1). The binary never selects inputs from an external source itself.

---

## 9. Execution loop

The per-domain crank, driven by the L1 confirmation frontier. Every step is a pure function of its inputs except I/O (znnd reads, store writes, L1 submission). The architecture is:

```
advance to confirmation frontier (frontier − confirmationDepth)
  → read confirmed momenta in ascending height; on a momentum-hash change at a
    previously-consumed height → reorg handling (§14)
  → read Settlement's assignment for domainId: each input's global index and ordinal
  → for each input in order: plugin.DecodeInput → plugin.Apply (witnessed) → stage effects → receipt
  → close a batch contiguous over THIS domain's subsequence at MaxBatchInputs (never over raw global indices)
  → build ExecutionResult set, roots, DA bundle (DAHash); verify DAHash round-trip
  → ── the supervisor, the single executor (§13) ──▶ pre-check on-chain cursor; SubmitBatch
     ── role:relay ────────────────────────────────▶ (separate path, §12) post source data to L1
  → on finalization: relay outbox messages / withdrawals; send ReleaseWithdrawals to release due withdrawals
  → persist cursors atomically; repeat
```

**[BINARY]** Every branch detecting an internal inconsistency terminates in `HALTED(reason)`, never silent continuation (§15).

---

## 10. Determinism and replay

**[CORE]** `Apply` may consume only its `state` accessor and its decoded `input`. The binary **MUST** be able to reconstruct any domain's state from genesis (or a checkpoint) by replaying canonical inputs alone. For external domains, the executor reads the external chain only in the relay/`Source` path to decide what to propose; the consumed data is then committed and published, so a genesis syncer replays it through `Apply` to byte-identical state **without** external-network access (`SPEC.md` §13.5.1, §20). This is the property that lets any replayer, and the validity-proof program mainnet requires, reproduce the executor exactly.

---

## 11. (Unassigned)

This section number is unassigned and holds no requirement.

---

## 12. Relay role

**[BINARY]** For `L1_RELAYED` domains, the relay watches the external source via the plugin's `Source` adapter and posts the data the STF will need (e.g. BTC headers + transaction inclusion proofs) to Settlement as L1 account-blocks.

- Relaying is **permissionless** and carries **no trust**: the STF validates every relayed datum and rejects anything non-canonical, so a dishonest relayer can only *withhold*, which is defeated by relaying being open to anyone (force-inclusion / completeness, `SPEC.md` §4.2, §6.1).
- The relay does **not** run the STF or hold state; it is a thin source-to-L1 pump. It **MAY** be co-hosted with an executor for liveness or run standalone.
- **[PREREQUISITE]** Settlement input methods + ABI to carry relayed payloads (§18).

> Worked example (BTC `L1_RELAYED`): user sends BTC to the bridge address tagging the L2 recipient → relay posts the header(s) and the deposit's Merkle proof → the executor's `Apply` validates PoW/linkage/confirmations, verifies the branch, credits wrapped-BTC in the plugin's SMT state → standard batch commit. Replay reads the relayed headers/proofs from L1 and re-runs `Apply`; no Bitcoin access. BTC *withdrawal* (threshold-signing a Bitcoin transaction) is a key-custody problem outside this binary's scope.

---

## 13. Multiple executors and proposer selection

**[CORE]** Phase 1 has one executor, serving every domain, under `proposerPolicy = SINGLE` (`SPEC.md` §6.2): **no proposer schedule is computed**. A multi-executor set and its proposer rule are specified with Phase 2 (`../phase-2/`).

**[BINARY] Single-instance lease (distinct concern).** Independently of the set, the binary **MUST** prevent two copies of the *same* executor identity running concurrently (equivocation), via a single-active-instance lease acquired at boot (file lock or external lease); a non-holder **MUST** exit without entering the execution loop. This is orthogonal to proposer selection: the lease guards one identity; the proposer policy picks among identities.

---

## 14. Recovery and state machine

**[BINARY]** On startup the binary reconciles local state against the most recent *finalized* on-chain root before resuming.

```
BOOT ─ load+validate config, acquire the domain lock (§16) ─▶ LOAD_SNAPSHOT
LOAD_SNAPSHOT ─ load the snapshot at the cursor's position (or genesis) ─▶ FIND_RECOVERY_ANCHOR
FIND_RECOVERY_ANCHOR ─ read latest FINALIZED batch for the domain ─▶ RECONCILE_FINALIZED_ROOT
RECONCILE_FINALIZED_ROOT ─ recompute local root at anchor.lastInputSeq; compare ─▶ match: REPLAY_POST_ANCHOR | mismatch: HALTED
REPLAY_POST_ANCHOR ─ re-derive post-anchor state from the canonical stream ─▶ READY
REPLAY_POST_ANCHOR ─ required DA bundle missing ─▶ AWAIT_DA ─ retry exhausted ─▶ HALTED
READY ─ the supervisor submits each committed batch (§5.5) and the worker stays READY
any state ─ consistency fault ─▶ HALTED (terminal until operator action)
```

- **Recovery anchor:** the most recent `FINALIZED` batch's `postStateRoot` + `lastInputSeq`, read from Settlement storage via `GetProof` against the L1 state root (`GetStateRoot`, §5.5). Fallback (no finalized batch): genesis (empty SMT, with the executor's own cursor at "nothing consumed"). The on-chain representation of "nothing consumed" is a Settlement concern (`SPEC.md` §32) and **MUST NOT** be assumed to be a signed `-1`. The Settlement storage read path is built: the supervisor reads each entity through `GetProof` with the key builders exported by `vm/embedded/definition/settlement.go` (§5.5).
- **[BINARY]** Transient faults (a failed batch-commit write, OOM, network, a worker crash) **MAY** auto-restart with bounded backoff; in the supervisor/worker binary the supervisor performs the restart (§1.5). Consistency faults (root mismatch, `DAHash` mismatch, corruption) **MUST** stay `HALTED`: the worker exits with its halted status and the supervisor does not restart it. A failed local write outside the batch commit also halts.
- **[CORE]** Forward replay from the anchor advances by walking the domain's subsequence (the next `globalInputIndex` *belonging to this domain*), never by incrementing the global index by 1 (§5.1).
- **`AWAIT_DA` (new state).** If post-anchor replay needs its own pre-finalization DA bundle and it is not present, enter `AWAIT_DA` with bounded retry → `HALTED` on exhaustion. A missing local bundle for an un-finalized batch is a **defined fault**, never silent. **[BINARY]** Phase 1 retries its own bundle 5 times, 200 ms apart: a bundle this process wrote is not subject to network latency, so a miss that survives a few quick retries is local corruption or loss, and waiting longer only delays the halt. The count and interval are local liveness knobs fixed in the worker; exhausting them always halts, so neither affects correctness.
- **Anchor ahead of local frontier (new edge).** If Settlement shows a `FINALIZED` batch at a height this node never produced (a restored old disk), `RECONCILE_FINALIZED_ROOT` **MUST** distinguish *local-absent* (→ re-derive forward from the anchor, `REPLAY_POST_ANCHOR`) from *root mismatch at a height local did produce* (→ `HALTED`).
- **Intermediate-finalized mismatch (new fail-closed edge).** If forward replay disagrees with **any** intermediate finalized batch root between the anchor and the local frontier → `HALTED`.
- **Reorg deeper than `confirmationDepth`** → `HALTED` (L1 fault). The executor **MUST NOT** have submitted a batch whose inputs lie below an unconfirmed reorg point; `confirmationDepth` (the open question of §17) is precisely what guarantees this.

---

## 15. Failure handling

| Failure | Detection | Behaviour | Transition |
|---|---|---|---|
| Malformed `ContractEffects` | Blob validation (`SPEC.md` §14.3, §27.1a) | `RUNTIME_FAULT`, full rollback, failure receipt | stays `READY` |
| Trie root mismatch (incremental ≠ recompute) | post-apply check (`SPEC.md` §13.5) | internal corruption | → `HALTED` |
| `DAHash` round-trip mismatch | §5.4 | bundle serialization/persistence wrong | → `HALTED` |
| Recovery-anchor root mismatch | §14 | local state corrupt/substituted | → `HALTED` |
| Reorg (consumed momentum retracted) | momentum-hash change at known height | roll back cursor, state store and recovery tree to the surviving height, re-derive; a plugin's node-local store is not rolled back (below); depth > `confirmationDepth` → L1 fault | → `READY` / `HALTED` |
| Crash mid-batch | worker exit or signal | supervisor restarts the worker with backoff (§1.5); full recovery state machine | → `BOOT` |
| Disk full / OOM | write failure / OOM kill / memory-limit kill | fail-closed; no partial submit. A failed batch-commit write, an OOM kill or a memory-limit kill ends the worker, and the supervisor restarts it with backoff (§1.5). Any other failed local write halts (§14). | → `BOOT`; → `HALTED` after repeated failures without progress (§1.5), or at once for a write outside the batch commit |
| znnd unavailable, or its frontier behind the recorded momentum window | RPC timeout or read failure | the worker retries at a fixed interval (5 attempts, 1 s apart) without advancing its cursor, then exits; the supervisor restarts it with backoff until the source recovers (§1.5) | stays; → `BOOT` after max retries |
| Assignment consistency fault | worker decode: a named send not found, undecodable, or naming another domain; an assignment not starting at the next expected ordinal; a Settlement account-block in a momentum below version 3 | halt; never a receipt status | → `HALTED` |
| Withdrawal-cap read unavailable | supervisor cannot serve a cap lookup during `Apply` | a source outage, not a halt: the worker ends and the supervisor restarts it with backoff (§1.5); the input is re-executed from its committed position | → `BOOT` |
| Submission rejected (contiguity) | account-block rejected | the supervisor retries the batch with backoff and reports an operator error after 5 consecutive rejections | → `READY` |
| Duplicate instance | lease acquisition fails | exit immediately | never leaves `BOOT` |
| Malformed worker message | supervisor decode | message discarded; worker killed and restarted with backoff (§1.5) | → `BOOT`; → `HALTED` after repeated failures without progress (§1.5) |
| Code-store read/write failure on a deploy or upgrade | plugin store layer — absent record, a record whose layout check fails, an absent chunk, a checksum failure, an I/O fault, or a failed memoizing write | halt; never a receipt status | → `HALTED` |
| Deploy/upgrade pipeline failure | assembly, hash/length mismatch, validation, instrumentation, or compile (`SPEC.md` §11.3) | `VALIDATION_FAILED` receipt, no contract created; never a halt | stays `READY` |

**[BINARY]** The worker's znnd retry uses a fixed interval rather than a growing one: a local node catching up, or a transient RPC failure, is not a condition a longer wait helps with, and a fixed interval keeps the time to each worker restart predictable for an operator. The supervisor's restart delay grows instead (§1.5), because it spaces the restarts of a worker whose source stays unavailable, and such an outage never halts the domain. Like the `AWAIT_DA` schedule (§14), the count and interval are fixed in the worker and never affect correctness. `confirmationDepth` (§17) is safety-relevant and stays a required configuration field with no built-in default.

**[BINARY] What a reorg rollback does not reach.** A rollback restores the cursor, the state store and the recovery tree (§16). A plugin's node-local store (§2 Principle 3) is not rolled back, and the `Domain` interface has no rollback hook. For the WASM plugin that store holds the pending deploy records and chunk bytes its `DecodeInput` writes. The store keys each pending record and chunk by `(deployer, code_hash)`, where the deployer is the input's `L1Sender`, so another deployer's `DeployContractStart` cannot reach a record or chunk. A finalize or upgrade reads only its own sender's entries. Replay repairs almost all of the rest: a replayed `DeployContractStart` clears and rewrites its `(deployer, code_hash)` pending record and every chunk slot, and a replayed chunk rewrites its own slot. Settlement accepts a finalize only once every chunk is present (`SPEC.md` §11.3), so every chunk a canonical finalize assembles is rewritten on replay. One path is not repaired: the retracted branch saw a pending record expire (`SPEC.md` §11.5) and accepted a second `DeployContractStart` from the same deployer for the same `code_hash`, while the canonical chain finalizes the first record before it expires. The second start's pending record survives the rollback, so a canonical finalize reads the wrong `total_size`, `chunk_count`, `upgrade_policy` and `owner`. The runtime does not detect this: it can commit a different status, or a `SUCCESS` carrying the `upgrade_policy` and `owner` of the same deployer's retracted start, from a node that never reorged. The `owner` is always one the deployer itself chose. It needs a reorg spanning the `DeploymentExpiryWindow` boundary of a deployment in flight. The Phase-1 remedy is an operator re-sync: stop the executor, remove the domain directory (`<storagePath>/<domainId>/`, which holds the plugin's store under `wasm-code/`), and re-sync from genesis, which re-derives every entry from the canonical stream. Closing it in code needs a plugin rollback seam.

---

## 16. Storage layout

**[BINARY]** Property-based; no specific KV engine prescribed. Each domain worker owns its own directory, and no other process writes it (§1.5). A worker holds an exclusive lock on its directory (`worker.lock`) for its lifetime, so two workers never write one domain. The supervisor keeps its lease and submission log under `<storagePath>/supervisor/`. Per domain:

```
<storagePath>/<domainId>/
├── snapshots/         # content-addressed, tagged with inputSeq + stfSpecHash; a snapshot tagged with another stfSpecHash is never loaded: boot halts, and a rollback skips it
├── bundles/           # DA bundles keyed by DAHash; Phase 1 retains every bundle (see below)
├── receipts/          # receipt sidecars keyed by DAHash: the batch's receipts, the receiptRoot leaves (§21.2)
├── wasm-code/         # WASM plugin's node-local code store (§2 Principle 3); not rolled back on reorg (§15)
├── cursor             # atomic writes: lastInputSeq, postStateRoot, momentumHeight, momentumHash, nextOrdinal, nextBatchId
├── momentum_hashes/   # height → hash, for reorg detection
├── batches/           # built batch metadata { batchId, status, firstInputSeq, lastInputSeq, preStateRoot, postStateRoot, daHash, commitment }
├── worker.lock        # the worker's exclusive lock on this directory
└── meta.json          # { domainId, executorId }, written once at first boot and compared on every later boot
```

The state store holds the domain's leaves in memory and has no `state/` directory. A snapshot in `snapshots/` is its only on-disk copy: boot loads the snapshot at the cursor's position whose root equals the cursor's `postStateRoot`, halts if there is none, and follows the ledger from there (§14).

`cursor` writes **MUST** be atomic (rename-over); a crash must recover to the last atomically persisted cursor.

**[BINARY] Gateway read surface.** `bundles/`, `receipts/` and `batches/` are the gateway's read interface to a worker (§21.2). Their file formats are versioned like any wire format: a change that a reader cannot parse is a new version, never an edit in place. A worker writes a batch's receipt sidecar atomically, with its bundle, before the batch record, so a batch record always names a bundle and a sidecar that exist.

**Bundle retention.** Phase 1 retains every bundle indefinitely, with no garbage collection. Two requirements depend on it. A bundle is the only durable copy of a superseded `CodeHash` (`SPEC.md` §12), since the state store holds only current leaves (`SPEC.md` §13.1). And an un-finalized batch's own bundle is required by `REPLAY_POST_ANCHOR` (§14) on every restart before that batch is `FINALIZED`. Garbage collection is out of Phase 1 scope; when it lands, it **MUST NOT** collect a bundle before its batch is `FINALIZED`. Storage growth is an accepted Phase-1 operating cost.

---

## 17. Configuration

**[BINARY]** All fields validated at boot; the binary **MUST** halt on any malformed field, and on any missing field that has no stated default. A binary requires every field it consumes that has no stated default, and **MUST** reject any field it does not consume as unknown, so no setting is silently ignored. Supervisor:

```
pillarName, executorAddress, networkName             # identity (SPEC §9.1, §22.1); networkName: network name, each plugin derives `chain_id` from it (SPEC §9.1)
znndEndpoint                                         # L1 connection; the Settlement address is types.SettlementContract
confirmationDepth                                    # momenta behind frontier treated as confirmed
maxPendingBatches     : 1 to 64, default 8           # batches a worker lets sit above Settlement's nextBatchId before it stops sealing
storagePath
submissionKeySource   : { kind: FILE | ENV, path, name } # FILE takes path, ENV takes name, KMS is not implemented; a 32-byte ed25519 seed, hex; injected at boot, never embedded, never passed to a worker
workers               : one entry per domain worker (below)
```

Per domain worker:

```
domainId, stfSpecHash                                # identity (SPEC §6)
binary                                                # the worker executable linked with this domain's plugin (§1.4)
memoryLimit, cpuLimit                                # bytes and CPUs; enforced by the supervisor (§1.5)
```

**[BINARY]** The supervisor hands each worker its own entry's `domainId` and `stfSpecHash`, plus `executorAddress`, `networkName`, `confirmationDepth`, `storagePath` and `maxPendingBatches`, over the worker channel (§1.5). A worker receives no other configuration, and never `submissionKeySource`.

**[BINARY] Node requirements.** The `znnd` node at `znndEndpoint` **MUST** serve version-3 momenta (`StateRootSpork` active) and retain state-tree versions for at least as far back as the executor reads. A non-archive node keeps about two days beyond the rollback window; an executor that lags further, or re-syncs a domain from genesis, needs an archive node.

> `confirmationDepth` has no spec-fixed value (`SPEC.md` §4.3 says "confirmed momentums" without a depth); it is an operational parameter requiring a Phase-1 default — an open question for DS.

Gateway (§21), a separate configuration:

```
znndEndpoint, networkName, confirmationDepth        # L1 connection; must serve version-3 momenta, like the executor's
storagePath                                         # the gateway's own store; never an executor's storagePath
listenAddress         : string                      # default "127.0.0.1:35999"; HTTP and WebSocket
allowedOrigins        : [string]                    # CORS; empty allows none
simulation            : { enabled, maxGas, timeoutMs, maxConcurrent, uid }   # §21.8; defaults true, 100_000_000, 2000, 2; uid is an optional dedicated uid for the simulator processes
domains               : one entry per served domain (below)
```

Per served domain:

```
domainId
source                : LOCAL | REMOTE
executorStoragePath   : string | null               # LOCAL only: the executor's storagePath, read-only
upstreams             : [url]                       # REMOTE only: other gateways' base URLs
workerBinary          : string                      # absolute path of the domain's worker binary; required when simulation is enabled (§21.8)
```

---

## 18. Codebase grounding and prerequisites

**[EXISTS] — can build against now:**

| Capability | Source |
|---|---|
| Momentum content / headers / state root / proof | `rpc/api/ledger.go` `GetDetailedMomentumsByHeight:456`, `GetMomentumsByHeight`, `GetStateRoot:359`, `GetProof:371` |
| Settlement contract: every `SPEC.md` §5.3 method, its ABI, address and spork | `vm/embedded/implementation/settlement.go`, `vm/embedded/definition/settlement.go`, `common/types/address.go`, `common/types/spork.go` (go-zenon `feature/settlement`) |
| Account-block publishing and signing | `rpc/api/ledger.go` `PublishRawTransaction`, `wallet/keypair.go`, `vm/plasma.go` `GetBasePlasmaForAccountBlock` |
| Canonical ordering primitives | `common/types/account_header.go:50` (`Bytes()`), `chain/nom/momentum_content.go:58` (sort) |
| Path-native SMT API | `common/trie/pathapi.go:39,49`, `compact.go`, `proof.go:235,254`, `hash.go:47,54` |
| Embedded-contract + spork registration pattern | `vm/embedded/embedded.go` (`GetEmbeddedMethod`, `applyHtlcDiffs`, `applySettlementDiffs`), `vm/vm_context/spork.go` (`IsHtlcSporkEnforced`, `IsOffchainExecutionSporkEnforced`), `common/types/spork.go` (`HtlcSpork`, `OffchainExecutionSpork`), `common/types/address.go` (`HtlcContract`, `SettlementContract`) |
| Pillar registry read + time-locked admin pattern | `chain/momentum/embedded.go` (`GetActivePillars`), `vm/embedded/definition/pillars.go` (`PillarInfo.RewardWithdrawAddress`), `vm/embedded/implementation/common.go` (`TimeChallenge`) |
| Deterministic WASM runtime (validation, instrumentation, gas metering, three-import host ABI, `Apply` for every §27.5 input kind) | `plugins/wasm/` (zeno-executor) |
| Node-local content-addressed code store (memoization tier, §2 Principle 3) | `plugins/wasm/codestore.go` (zeno-executor) |
| Settlement storage read path: exported key builders, and ordinal entries that name their send | `vm/embedded/definition/settlement.go` (`DomainInputOrdinal.SourceAccountBlockHash`, the `*Key` builders), `rpc/api/embedded/settlement.go` (go-zenon `feature/settlement`) |

**Upstream dependencies and their build status — each is required before the capability it blocks:**

| # | What | Status | Where it lives | Blocks |
|---|---|---|---|---|
| P-9 | Settlement input methods and ABI to carry relayed payloads | not built: the relay role is reserved | `vm/embedded/definition/settlement.go` | §12 |

Numbers are stable; no surviving row is renumbered. A gap means that prerequisite is satisfied and its capability is listed in the **[EXISTS]** table above.

---

## 19. Conformance

**[CORE] (preconditions):** SMT vectors green (14/14); trie adversarial tests green incl. the double-hash trap; path-native API vectors (CV-PATH-1/2) and applier vectors (CV-APPLY-1/2) green; the binary uses **only** the path-native API — a grep for `VerifyProof(`/`VerifyAbsence(`/`NodeTree`/`stagedApplier` outside the L1 adapter returns zero; the L2 `StateDiff` applier preserves present-empty and never folds empty→delete (`SPEC.md` §13.5.1, §13.6); every vector of `execution-conformance-v1.json` reproduced byte-for-byte, on every CI architecture (`SPEC.md` §31).

**[BINARY] (component):** input-sequence derivation is identical across two independent implementations for identical momentum content; replay determinism (same inputs → identical roots, `ExecutionResult`s, `DAHash`); `DAHash` reproducibility; batch-commitment round-trip (present-but-empty `proofData`; unknown `protocol_version` rejected); a malformed `ContractEffects` blob → `RUNTIME_FAULT`; a malformed canonical input encoding → halt.

**[BINARY] (operational):** crash recovery reconciles against the finalized anchor with no reorder/skip/duplicate; reorg rollback; equivocation (lease) — non-holder exits; snapshot substitution → halt.

**[BINARY] (gateway):** a `LOCAL` and a `REMOTE` gateway over the same domain return identical `latest` and `finalized` responses; a corrupted bundle, sidecar or batch root marks the domain `INCONSISTENT` and nothing past it is served; a retracted pending batch disappears from `pending`; every proof the gateway returns verifies with `VerifyProofByPath` / `VerifyAbsenceByPath` against the root it names.

---

## 20. Phase mapping

| Phase | Roles active | Domains | This binary |
|---|---|---|---|
| Phase 1 (testnet) | Executor | One WASM `L1_NATIVE`, `SINGLE` proposer | Supervisor + one worker, single-instance lease, recovery state machine; gateway on by default |
| Phase 2 (mainnet) | Executor | Specified in `../phase-2/` | Validity proofs (`ZK_VALIDITY`), verified by Settlement (`SPEC.md` §6.3) |

The generic-runtime / pure-STF-plugin / uniform-commitment seam does not change across phases; new domains, new executors and the relay role are additive.

---

## 21. Gateway

The gateway serves L2 data to wallets, dApps, explorers and other gateways through one API for every domain. It runs no STF: it rebuilds each served domain's state by applying the `StateDiff`s recorded in DA bundles, and checks what it built against the roots Settlement holds.

### 21.1 Process and trust

- **[BINARY]** The gateway is its own binary (`cmd/gateway`), configured separately (§17). It holds no key material, never connects to a worker or the supervisor, and only reads an executor's storage.
- **[BINARY]** It **MUST** apply `StateDiff`s with the canonical L2 applier (`SPEC.md` §13.5.1) over the path-native trie API (§5.3), under the same prohibitions as the state store. It **MUST NOT** load a domain plugin into its own process: its state comes from bundles, and simulation (§21.8) runs the domain's worker binary as a child process.
- **[BINARY]** It does not re-execute. It checks that published data matches the commitment, not that execution was correct: an executor that commits a wrong but self-consistent batch passes the gateway's checks.

> *Informative.* Every domain's per-input effects have one shape (`Effects`, §7), and every bundle has one layout (`SPEC.md` §20). That is what lets one gateway serve every domain without running any of them.

### 21.2 Sources

A batch's data is its bundle, its receipt sidecar and its commitment.

- **Receipt sidecar.** The canonical encodings of the batch's receipts (`SPEC.md` §27.7), ascending by `globalInputIndex`, as a `List<Bytes>` (§27.0). They are exactly the `receiptRoot` leaves. A bundle does not carry receipts (`SPEC.md` §20), and a receipt's status, `gasUsed` and `abortCode` are not derivable from the bundle, so the worker writes the sidecar beside the bundle (§16).
- **`LOCAL`.** The gateway reads `batches/`, `bundles/` and `receipts/` under the executor's domain directory, read-only. Batch records give it built-but-unsubmitted batches, so a `LOCAL` gateway serves `pending`. The operator grants the gateway's user read access to those three directories and no others.
- **`REMOTE`.** The gateway reads batch entries from Settlement (§5.5) and fetches each batch's bundle and sidecar by `DAHash` from its `upstreams` (§21.5). A `REMOTE` gateway serves `latest` and `finalized` only.

**[BINARY] Checks, per batch, before anything from it is served at `latest` or `finalized`:**

1. `SHA3-256(bundle) = DAHash`.
2. `MerkleRoot(sidecar leaves) = receiptRoot` (`SPEC.md` §27.3).
3. The root after applying the batch's `StateDiff`s to the previous batch's post-state equals `postStateRoot`.
4. Each receipt's `stateDiffHash`, `returnHash`, `eventRoot` and `outboxRoot` match the bundle's `ExecutionResult` for the same input (`SPEC.md` §15).

`DAHash`, `receiptRoot` and `postStateRoot` are read from the Settlement batch entry with a verified proof (§5.5). A failed check marks the domain **`INCONSISTENT`**: the gateway stops advancing it, keeps serving data up to the last good batch, reports the status in `zeno_getHead` and logs an operator alarm. A pending batch gets checks 2–4 against its own batch record and is served unverified.

**Retraction.** A `LOCAL` gateway re-reads batch records as it follows. A pending batch whose record disappears or changes (a worker rollback, §14) is retracted, and `pending` falls back to the last batch still recorded.

### 21.3 API

**[BINARY]** JSON-RPC 2.0 over HTTP and WebSocket at `listenAddress`, namespace `zeno`. The methods that read a batch or its state (`zeno_getBatch`, `zeno_getStateAt`, `zeno_getProof`, `zeno_read` and `zeno_simulate`) take a **tag**, and the list methods take it as `toBatch`: `"pending"`, `"latest"` (accepted on Settlement), `"finalized"` (`height ≥ finalizeAtHeight`, `SPEC.md` §21), or a `batchId`. Every response that carries L2 data names the `batchId` it was read at and a `verified` flag: `true` when the batch passed §21.2's checks against Settlement, `false` for `pending`.

| Method | Returns |
|---|---|
| `zeno_chainInfo(domainId)` | `networkName`, `chain_id` (`SPEC.md` §9.1), `runtimeKind`, `stfSpecHash`, the profiles the domain implements with their features, and its views (§21.4) |
| `zeno_getHead(domainId)` | The `pending`, `latest` and `finalized` batch, each with `batchId`, `lastInputSeq`, `postStateRoot`; the domain status (`SYNCING`, `OK`, `INCONSISTENT`) |
| `zeno_getBatch(domainId, tag)` | The commitment fields and the batch's inputs, each with `globalInputIndex`, L1 send hash and receipt status |
| `zeno_getReceipt(sendHash)` | The domain, `globalInputIndex`, `batchId`, the tag the batch has reached, the receipt, `ReturnData`, events, outbox messages, the L1 sender, and the domain's decoded input |
| `zeno_getInputs(filter)` | Input history: inputs by domain, filtered by receipt `contractId`, `sender`, `inputKind` and batch range (`fromBatch`, `toBatch`), each with its send hash, sender, batch, receipt summary and decoded input; paged, newest first |
| `zeno_getStateAt(domainId, path, tag)` | The value at a 32-byte SMT path: present, present-empty or absent (`SPEC.md` §13.6) |
| `zeno_getProof(domainId, paths, tag)` | Values plus `proof-format.md` V1 proofs against the named batch's `postStateRoot` |
| `zeno_getEvents(filter)` | Events by domain, contract, topics and batch range; paged |
| `zeno_getOutbox(filter)`, `zeno_getOutboxProof(outboxId)` | Outbox messages by domain, recipient and batch range; one message's `RelayMessage` inclusion proof (`SPEC.md` §27.3) and its `processedOutbox` state |
| `zeno_getBundle(daHash)`, `zeno_getReceiptSidecar(daHash)` | Raw bundle and receipt-sidecar bytes, hex |
| `zeno_read(domainId, view, params, tag)` | A domain view (§21.4) |
| `zeno_simulate(domainId, input, tag)` | A would-be receipt for an input run against the state at `tag`, never committed (§21.8) |
| `zeno_subscribe(kind, params)`, `zeno_unsubscribe(id)` | WebSocket only: `heads`, `events` (with a filter), or `receipt` (one send hash) |

The method set is the same for every domain. Domain-specific reads go through `zeno_read`, and domain-specific fields appear only inside generic responses: the decoded `input` of a receipt and the result of a view.

**Errors.** `-32001` unknown domain; `-32002` not available at the tag (no `pending` from a `REMOTE` gateway, or a version older than retained); `-32003` domain `INCONSISTENT` past the requested batch; `-32004` not found; `-32005` the domain has no such view; `-32006` simulation unavailable or over its limits.

`zeno_getInputs` is generic: every receipt carries a `contractId`, zero when the input concerns no contract (`SPEC.md` §15), and every input has an L1 sender. A contract's history is every input whose receipt names it, from its `DeployContractFinalize` on: calls, deposits, upgrades, ownership transfers and storage purchases, successful or not.

> *Informative.* The shape follows Ethereum's JSON-RPC: tags for `pending`/`latest`/`finalized`, receipts by transaction hash, storage reads with proofs, log filters, subscriptions. `zeno_simulate` covers both `eth_call` and `eth_estimateGas`. There is no send method: inputs are L1 account-blocks published through `znnd`.

### 21.4 Domain views

Each domain's state means something only under that domain's key layout and vocabulary: a WASM domain has contracts, a relayed-chain domain has headers. A **view** is a named, typed read that interprets a domain's state or indexes. Views are the only domain-specific reads, and they all go through the one method `zeno_read`.

- **[BINARY]** A domain's `Reader` (§7) is linked into the gateway, as its plugin is linked into its worker (§1.4). It declares the domain's views; `zeno_chainInfo` lists each view's name, parameters and result fields.
- **[BINARY]** A view reads only gateway state and indexes, and never executes. A view built on state **MUST** return, with its result, every path it read and a proof of each against the root the response names, so a client can re-derive the result from verified values instead of trusting the gateway's interpretation.
- **[BINARY]** A domain without a `Reader` is served by every generic method; `zeno_read` on it is error `-32005`.

**Profiles.** A **profile** is a named, versioned set of views with fixed parameters and result shapes, which any domain may implement. Clients build features against a profile, never against a `runtimeKind`, so a new runtime that implements a profile works in every client that supports it. `zeno_chainInfo` lists each profile a domain implements and the profile's feature flags. A domain **MAY** declare views outside any profile; clients reach them through `zeno_read` without built-in support.

**The `contracts/1` profile** is for runtimes with deployable contracts called through Settlement's contract inputs (`SPEC.md` §27.5 kinds 0–7).

| View | Params | Returns |
|---|---|---|
| `contract` | `contractId` | `codeHash` (executable), `sourceCodeHash` (as deployed), `upgradePolicy`, `owner`, `ownerCaller` (the owner leaf's caller hash), `quota` `{purchased, used}`, `nativeAddress` (`SPEC.md` §9.2), with the paths and proofs they come from. A field the runtime does not have is `null` |
| `storage` | `contractId`, `key` | The value under the runtime's storage key for that contract, with its path and proof. `key` is the runtime's contract-local key: a `local_key` for WASM, a 32-byte slot for a slot-addressed runtime |
| `contractsByDeployer` | `address`, optional `cursor` | Contracts created by a successful deploy from that L1 address, newest first, each with its creating send hash, 100 per page with a `nextCursor` |
| `module` | `sourceCodeHash` | The deployed bytes reassembled from `deployChunkData` across bundles, returned only when `SHA3-256(bytes) = sourceCodeHash` |
| `resolveNativeAddress` | `address` | The `contractId` whose native form is `address`. Only when `nativeAddresses` is set |
| `nativeCaller` | `l1Address` | The canonical `caller` of that L1 address and its native form. Only when `nativeAddresses` is set |

Feature flags, each `true` or `false`: `upgrade` (`UpgradeContract` can succeed), `ownershipTransfer`, `storageQuota` (`BuyStorage` meaningful), `payableCalls`, `deposits`, `nativeAddresses` (the runtime exposes native forms, `SPEC.md` §9.2). A client shows and accepts native forms only through these views and never derives one itself. The profile also names `interfaceEncodings`: the contract-interface encodings this runtime's contracts use (for example `zeno-selector-v1` for the Rust contract SDK's convention), which tells a client which encoder builds a call payload.

The WASM domain implements `contracts/1` with every feature flag `true` except `nativeAddresses`, and `interfaceEncodings = ["zeno-selector-v1"]`. Its `storage` path is `SHA3-256(contractId ‖ local_key)` (`SPEC.md` §13.3), and `contract` reads the reserved leaves (`SPEC.md` §12, §13.8).

### 21.5 HTTP bundle serving

**[BINARY]** The gateway serves `GET /bundles/<daHash>` and `GET /receipts/<daHash>` on the same listener, with HTTP range requests so a client can fetch in chunks of at most `MaxDAChunkSize` (`SPEC.md` §20). A `REMOTE` gateway fetches from these endpoints and verifies every fetch against Settlement (§21.2) before use; an upstream that serves bytes failing a check is skipped for that batch.

### 21.6 Storage and retention

- **[BINARY]** The gateway keeps its own state store, separate from any executor's. It **MUST** serve state at every batch from the oldest batch not yet `finalized` through `pending`, and at `finalized`. Older versions **MAY** be discarded.
- **[BINARY]** Its indexes — send hash to input, receipts, events, outbox messages, deployer to contracts, receipt `contractId` to inputs, L1 sender to inputs, and native address to contract where the runtime has native forms — are append-only and retained for every batch it has indexed. A rollback of a pending batch removes that batch's entries.
- **[BINARY]** For each input it records the L1 sender of the source account-block, read from `znnd` by the send hash in the canonical input encoding (`SPEC.md` §27.5).

### 21.7 Exposure

**[BINARY]** The gateway is part of the default deployment and listens on `127.0.0.1` by default, like `znnd`'s HTTP default. An operator exposes it by changing `listenAddress` or by putting it behind their own proxy. `allowedOrigins` controls browser access; an empty list allows none.

### 21.8 Simulation

`zeno_simulate` runs an input against a committed or pending state without committing anything. It is how a client reads a contract through a method that only returns data, previews a call's outcome, and estimates its gas, for any runtime.

- **[BINARY]** Input: `kind` (`CallL2` or `Deposit`, `SPEC.md` §27.5 kinds 0 and 1), the L1 `sender`, `targetContract`, `payload` for `CallL2`, an optional `deposit` `{asset, amount}`, and `plasmaLimit`. The result is the would-be receipt fields — status, `gasUsed`, `abortCode`, `ReturnData` or revert data — plus events, outbox messages and the `StateDiff`, and the `inputIndex`, `momentumHeight`, `momentumTime` and `momentumHash` it ran under, marked `simulated: true`. A simulation is never `verified`.
- **[BINARY]** The gateway runs the domain's own worker binary in simulate mode as a child process, one pool per domain. In simulate mode the worker opens no domain directory, keeps no cursor, persists nothing and talks only to the gateway over a local channel. Its `Apply` reads state through a read-only accessor the gateway serves at `tag`'s version. A crash or timeout ends that simulation with `-32006` and the gateway replaces the process.
- **[BINARY]** Context: `caller` derived from `sender` (`SPEC.md` §9.2), the latest confirmed momentum's height, time and hash, and an `input_index` one past the domain's last known input. The runtime's post-execution rules (`SPEC.md` §10.6) apply unchanged, so the status is the one a real input would get against the same state and context.
- **[BINARY]** `plasmaLimit` is capped at `simulation.maxGas`; a larger request is error `-32006`, never clamped. `timeoutMs` and `maxConcurrent` bound the work an anonymous caller can cause. `enabled = false` turns the method off.

> *Informative.* A simulated result can differ from the real one: other inputs may land first, and the momentum context moves on. It is a preview, not a promise.

