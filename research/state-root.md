# Momentum state root

**Status:** Research and design notes — informative. Implemented in go-zenon behind `StateRootSpork`.

**Normative source:** [phase-1/SPEC.md](../phase-1/SPEC.md) §13 (the shared SMT) and §4.6 (the executor's L1 read path); the go-zenon code in `common/trie`, `chain/state_tree.go` and `verifier/momentum.go`.

The momentum state root is a 32-byte hash of all go-zenon consensus state, written into every momentum header and signed by the producing pillar. It lets anyone prove that a key holds a value, or holds nothing, at a given height, with no trust in the node that answers. The executor and every Zeno client read Settlement storage through it.

## 1. What it is and the problem it solves

Ask a node for an address balance or a Settlement batch record, and it returns a value. Without a state root you cannot check that value except by replaying the whole chain. The header commits to each momentum's write-set through `ChangesHash`, but a diff cannot prove one current fact.

The state root fixes this. Every node keeps all consensus state in one hash tree, and the hash at the top is a fingerprint of the entire state: change any value and the root changes. From momentum version 3 the producing pillar writes that root into the header it signs. Two parties who agree on a root agree on the whole state, and a node can hand out a small proof for any key that verifies against the header alone.

The root is history-independent: replay from genesis and a build from a synced database give the same root.

## 2. What a proof proves and how a client checks one

A proof says "key `K` has value `V` at height `H`", or "key `K` is absent at height `H`". It carries `V` plus the sibling hashes from `K`'s leaf to the root, about 1–1.4 KB, so it fits inside an account-block.

A client checks a proof in three steps.

1. Fetch the root with `ledger.getStateRoot(height)` and the proof with `ledger.getProof(height, key)`, which returns `{value, proof, root}`.
2. Compare `root` with the `StateRoot` field of the signed momentum header at that height.
3. Hash `value` into a leaf, fold it up through the siblings, and confirm the result equals that root. `trie.VerifyProof` and `trie.VerifyAbsence` do this with no database and no node trust.

An absence proof has the same shape and ends at an empty slot. `cmd/prove-balance` shows the flow; the wire layout is fixed in [proof-format.md](../artifacts/proof-format.md).

## 3. How it is built and wired into consensus

**The tree.** `common/trie` is a versioned, full-depth-256 binary sparse Merkle tree with SHA3-256 throughout. A leaf sits at `path = sha3(dbkey)` and hashes as `sha3(path ‖ value)`. An interior node hashes as `sha3(left ‖ right)`, and an empty subtree is the constant zero hash at every level. `trie.NodeTree` persists nodes keyed by `(version, path)`, so a reorg can undo a version and historical heights stay readable.

**What it commits to.** The tree folds exactly the write-set that `ChangesHash` commits to, minus the three momentum-frontier metadata keys. `trie.FoldFilter` fixes that rule for every build path.

**The header field.** `StateRoot` is a 32-byte momentum field in `chain/nom/momentum.go`, folded into `Momentum.ComputeHash()` from momentum version 3 (`StateRootMomentumVersion`). It follows the version-2 bytes, so version-2 momentums hash and serialize unchanged.

**Producer and verifier.** The producer in `pillar/worker_momentum.go` stamps version 3 and sets `StateRoot`. The verifier in `verifier/momentum.go` requires a zero root before activation; after it, it recomputes the root with `ComputeStateRoot(previous, changes)` and rejects a mismatch with `ErrMStateRootInvalid`.

**Node integration.** In `chain/state_tree.go`, `Init` catches the tree up to the chain frontier, `UpdateStateTree` folds one momentum per insert, and `TruncateStateTreeTo` undoes versions on a reorg. A default node prunes versions older than the rollback window plus about two days; `StateTreeArchive: true` keeps every version. Pruning never changes a root.

**The spork.** `StateRootSpork` gates the feature through the standard create, activate and enforce flow. Every node maintains the tree from genesis; the root enters headers, and the verifier enforces it, only once the spork is active. `OffchainExecutionSpork` must not activate first, because the executor reads Settlement storage only from version-3 momenta.

## 4. The shared core with Zeno's executor

One SMT algorithm serves both layers. The frozen core in `common/trie` is `LeafHash`, `InternalHash`, `RootOfLeaves`, `ProveByPath`, `VerifyProofByPath` and `VerifyAbsenceByPath`; the incremental `CompactTree` matches `RootOfLeaves` and `ProveByPath` byte for byte. Two adapters sit on top. The L1 adapter (`NodeTree`) hashes database keys into paths and folds an empty value to a deletion. The L2 executor adapter uses `sha3(cid ‖ local_key)` as the path, deletes only on the explicit `EMPTY` sentinel, and keeps a zero-length value as a present-empty leaf distinct from absence.

The executor must not reuse the L1 adapter. `NodeTree` pre-hashes its key, so an L2 path would be hashed twice, and its empty-to-delete fold would erase the present-empty leaves L2 requires. The executor therefore calls only the path-native API, never `NodeTree` or `VerifyProof`. One vector suite, `smt-v1-test-vectors.json`, governs both; the [artifacts registry](../artifacts/README.md) records its source of record, its copies and the implementation checklist.

## 5. Why Zeno needs it

Settlement keeps its batches, domains and custody counters in go-zenon storage. With the state root, a client proves any of those entries against a signed momentum header, so it trusts neither the gateway nor the node. The gateway's own checks read `postStateRoot`, `receiptRoot` and `DAHash` this way.

The executor's L1 read path depends on it. For each confirmed momentum it reads each domain's received-input count and `(domainId, ordinal)` entries through `ledger.getProof`, verifies them with `VerifyProofByPath` against the header's `StateRoot`, and treats a mismatch as a consistency fault. Its recovery anchor, the last finalized batch's `postStateRoot`, comes the same way. See [EXECUTOR.md](../phase-1/EXECUTOR.md).

## 6. Build at activation

A node's first start on a binary that carries the state tree builds the tree to the chain frontier inside `chain.Init`, synchronously, and is offline meanwhile. It is the chosen design because it has no concurrency: `Init` runs before any insert or reorg driver, so nothing can race it. Rolling pillar upgrades keep the network producing while individual nodes rebuild.

Build time on a mainnet copy is the release-gating measurement. If it shows unacceptable downtime, two alternatives are ready. An offline pre-build command builds the tree against a database snapshot while the old binary keeps running, so the upgraded node catches up only a small delta; this is the preferred pivot. A concurrent background build keeps the node responsive, but must hold the chain-wide insert lock around every batch it folds, or it folds a half-rolled-back momentum.

Whether the SMT keeps SHA3-256 or moves to a ZK-friendly hash is Phase 2 research item R3 in [phase-2/SPEC.md](../phase-2/SPEC.md), tracked in [R3-smt-hash.md](proving/R3-smt-hash.md). The decision is due before `StateRootSpork` reaches mainnet; the hash sits behind one interface so a migration touches one component.
