# Ticket E2-12 — State store scaling

**Epic:** 2 — Executor (spans go-zenon `common/trie`) · **Phase:** 1A · **Order:** E2-12

## Goal
Make the state store's per-input and per-witnessed-key costs independent of total domain state.
Two costs grow with total leaf count:
1. `Store.Root` and `Store.Prove` rebuild and hash the whole tree on every call. `Prove` runs
   once per witnessed key an input observes.
2. `Store.Accessor` copies every leaf into a fresh map once per input, even for an input that
   reads nothing.

The ticket keeps one incremental `CompactTree` beside the leaf map, so a root or proof costs
`O(log N)`. It also gives each input a shared, generation-tracked view of the leaf map instead
of a copy. No root, proof, witness or receipt byte changes.

The work spans two repositories. The go-zenon half adds the path-native incremental prover to
`common/trie`. The executor half rebuilds `Store` and `Accessor` on it.

## SPEC refs
SPEC §13.1 (shared `common/trie` core; enumerated path-native API, including `NewCompactTree`
and `CompactTree`'s `Insert`, `Delete`, `Root`, `ProveByPath`), §13.4 (SHA3-256 hashing,
constant-zero empty subtree), §13.5 (incremental root equals full `RootOfLeaves` recompute),
§13.6 (present-empty is distinct from absent), §13.7 (`smt-v1-test-vectors.json`), §10.1
(snapshot construction is not a charged surface). EXECUTOR §18.

## Depends on
E2-02 (state store and applier) and E2-07 (the STF driver's call sites in
`internal/batch/driver.go`). E2-11 supplies the benchmark shapes and the state sizes 10³, 10⁵
and 10⁶ leaves. No L1 consensus dependency: the go-zenon half adds functions to `common/trie`
and changes no momentum root, so it needs no spork.

## Key work

### go-zenon half: path-native incremental prover (`common/trie`)
This half lands on go-zenon `feature/merkle-state-root`, which owns `common/trie`.
- `NewCompactTree()` is the exported constructor. `Insert`, `Delete` and `ProveByPath` take a
  `Path`, never a `types.Hash` key, so the §13.1 path/key distinction is explicit at each call.
- `(*CompactTree).ProveByPath(path)` walks from the root and fills the 256-entry sibling array
  the way `proveLeaves` in `compute.go` does. It uses `padSubtree` for each off-path child and
  `emptyHash` along single-child chains. It encodes with the unexported `encodeProof`.
- The walk follows `readCtx.proveFromRoot` in `nodestore.go`. `NodeTree` itself stays L1-only,
  because its entry points hash keys.
- `Insert` with a zero-length value stores a present-empty leaf. Only `Delete` removes a leaf,
  and deleting an absent path is a no-op.
- `conformance_test.go` drives `CompactTree` from the SMT vectors. A differential test compares
  it against `RootOfLeaves` and `ProveByPath` over random insert, update and delete sequences.

### Incremental tree in `Store` (`internal/state/store.go`)
- `Store.leaves` stays the source of truth for values and for the write-ahead log. `Store.tree`
  is a `*trie.CompactTree` that mirrors it and changes only under `mu.Lock`, together with
  `leaves`.
- `newStore` builds the tree once from the replayed leaf set, in `O(N log N)` once per process.
  `Open` and `openFresh` (`internal/state/snapshot.go`) both construct through `newStore`.
- `Apply` appends the diff to the log first, so a failed append leaves the `Store` unchanged.
  It then applies each entry: a nil `Value` calls `Delete`, any other value calls `Insert`. It
  returns `tree.Root()`.
- `Root` returns `tree.Root()`. `Prove` returns `tree.ProveByPath(path)`. Both hold only the
  read lock.
- `go.mod` resolves go-zenon through `replace github.com/zenon-network/go-zenon => ../go-zenon`
  until the go-zenon half merges and zeno-executor re-pins it.

### Shared accessor view (`internal/state/store.go`, `internal/state/accessor.go`)
- `Store.gen` numbers the leaf map generation `leaves` refers to. `Store.live` counts the
  unreleased accessors sharing that generation.
- `Store.Accessor()` takes `mu.Lock`, increments `live`, and returns an `Accessor` holding the
  current map, its generation and an empty witness map. It copies nothing.
- `Apply` clones `leaves` with `maps.Clone` only when `live > 0`. The clone starts a new
  generation with `live = 0`. An unreleased accessor keeps reading its own generation's map.
- `Accessor.Release()` marks the accessor released and decrements `live` only when the
  accessor's generation is still current. `Release` is idempotent.
- After `Release`, `Read` and `Len` return `ErrAccessorReleased`. `Witnesses` keeps working.

### Driver (`internal/batch/driver.go`)
`computeRecord` calls `defer acc.Release()` right after `d.cfg.Store.Accessor()`. The defer
fires when `computeRecord` returns, before `finishApply` calls `Store.Apply`. The driver path
therefore never clones the leaf map.

### Import check (`scripts/check-trie-imports.sh`)
The forbidden pattern covers `VerifyProof(`, `VerifyAbsence(`, `Tree.Update` and
`stagedApplier`. `CompactTree` is permitted, because it is part of the §13.1 path-native API.

## Decisions
- **The incremental tree lives in go-zenon, not the executor.** §13.1 requires the executor to
  use only the shared path-native API. `encodeProof` is unexported, so an executor-side tree
  would need a second proof codec.
- **The tree is in memory.** `CompactTree` holds every leaf, which suffices at 10⁶ leaves. A
  disk-backed path-native tree is needed only if domain state outgrows memory.
- **Scaling costs are removed in the store, not priced.** A cost that grows with total domain
  state cannot be charged fairly to one input. Snapshot construction stays uncharged (§10.1).
- **The accessor copy is removed because it fails the decision rule.** A(n) is the cost of
  a full-copy `Accessor()` at n leaves, and W is one witnessed read at 10⁶ leaves. With G =
  A(10⁶)/A(10³) and D = A(10⁶)/W, the store must change when G ≥ 100 and D > 1.
- **The rule's inputs came from a non-reference machine.** They gave G = 2610 and D = 1270,
  recorded in `wasm-vm/wasm-gas-table-v2-draft.md` §8. With full copies, one
  `MaxBatchInputs` batch at 10⁶ leaves spends 5.88 s on snapshots.
- **Sharing needs a generation count, not a plain shared reference.** `Apply` mutates `leaves`
  in place, and test callers hold an accessor across `Store.Apply`. A plain shared map would
  let them see later writes, and a concurrent map read and write is fatal in Go.
- **A copy-on-write map that always clones under a live accessor only moves the cost.** Every
  input takes an accessor and every successful input applies. The release call lets the driver
  skip the clone entirely.
- **An unreleased accessor costs one clone, never correctness.** It keeps full isolation for its
  generation. A read after `Release` fails loudly rather than observing later writes.

## Code references
- `internal/state/store.go` [EXISTS] — `Store` with `tree`, `gen`, `live`; `newStore`,
  `Open`, `Root`, `Prove`, `Accessor`, `Apply`.
- `internal/state/accessor.go` [EXISTS] — `Accessor`, `Release`, `ErrAccessorReleased`.
- `internal/state/snapshot.go` [EXISTS] — `openFresh` through `newStore`.
- `internal/batch/driver.go` [EXISTS] — `computeRecord`'s `defer acc.Release()`.
- `internal/state/differential_test.go`, `internal/state/accessor_test.go`,
  `internal/state/bench_test.go`, `internal/batch/driver_test.go` [EXISTS].
- go-zenon `common/trie/compact.go`, `common/trie/conformance_test.go` — the go-zenon half.

## Acceptance criteria
**go-zenon half**
- `CompactTree.Root` and `ProveByPath` reproduce every root and proof byte of
  `smt-v1-test-vectors.json` unmodified, driven from `conformance_test.go`.
- A differential test shows byte-identical roots and proofs against `RootOfLeaves` and
  `ProveByPath`. It covers inserts, updates, deletes, absent-path deletes, present-empty writes
  and absent-key proofs.
- The key-hashing `VerifyProof` rejects a `CompactTree` proof given the path as a key, and
  `VerifyProofByPath` accepts it. The existing `Tree` and `NodeTree` tests pass unchanged.

**Executor half**
- `TestStoreMatchesFullRecompute` (0, 1, 64, 512 leaves) checks every root and proof against
  a full recompute. It also checks the leaf map against the reference set, before and after a
  reopen.
- `TestStoreApplyFailedAppendLeavesStateUnchanged` passes.
- These accessor tests pass: `TestAccessorUnreleasedReadsPreStateAcrossApply`,
  `TestAccessorUnreleasedWitnessesAfterApply`, `TestAccessorStaleReleaseKeepsCurrentGenerationIsolated`,
  `TestAccessorDoubleReleaseKeepsSiblingIsolated` and `TestAccessorDriverSequenceDoesNotClone`.
- `TestAccessorReadAfterReleaseFails`, `TestAccessorConcurrentReadDuringApply` and
  `TestDriverReleasesAccessor` pass.
- `BenchmarkOpen` reports build time, `heap-MB` and `tree-heap-MB` at 10³, 10⁵ and 10⁶ leaves.
  `BenchmarkAbsentKeyWitness` shows roughly flat per-key `Prove` cost across the three sizes.
- `BenchmarkAccessor` and `BenchmarkAccessorApply` show per-input snapshot cost and B/op that
  do not grow with leaf count.
- `scripts/check-trie-imports.sh`, `go test ./...` and `go test -race ./...` pass. Every SMT,
  execution and serialization vector passes unmodified.
