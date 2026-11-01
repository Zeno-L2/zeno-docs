# Ticket E2-17 — Input path: Settlement's assignment, canonical decode, withdrawal caps

**Epic:** 2 — Executor · **Phase:** 1C · **Order:** E2-17

## Goal
Replace `UnwiredFilter` with Settlement's own input assignment. End state: each
worker's input stream is exactly the inputs Settlement accepted for its domain, in
`globalInputIndex` order, read from proven L1 state, with no second acceptance
predicate. Step 5a and the batch builder use the real per-(domain, asset)
`MaxBatchWithdrawal`.

## SPEC refs
SPEC §4.6 (assignment, per-domain ordinal, the executor reads the assignment), §5.4,
§10.6 step 5a, §27.5 (canonical inputs), §27.9 (cap reference point, unconfigured
asset); EXECUTOR §5.1.

## Depends on
E2-16 (read path), E1-10 (ordinal entry carries the send hash; exported key
builders), E2-06, E2-07, E2-08.

## Key work
Delivered in five `/develop` runs on one branch, in this order, squashed to one commit.

### 1. Decode
- A pure function from an ABI send plus its ordinal entry to the §27.5 canonical
  input. The method is unpacked with `domainId` first; `Deposit` and payable `CallL2`
  take asset and amount from the send; chunks set `SourceData`. A method that does
  not decode, or a `domainId` other than the worker's, is a consistency fault (HALT,
  SPEC §4.6). Tested against every `CV-INPUTROOT-1` leaf.

### 2. Assignment reader (supervisor)
- A momentum whose content holds no `SettlementContract` account-block has an empty
  assignment and costs no proof.
- Otherwise, per request (one momentum: its height and hash), the supervisor checks
  the node's momentum at that height has that hash (otherwise unavailable), proves
  the domain's received-input count at the momentum, and proves the ordinal entries
  from the worker's next expected ordinal up to that count. Each entry it serves
  carries `(ordinal, globalInputIndex, sourceAccountBlockHash, sendMomentumHeight)`;
  the send's momentum height comes from the node's account-block lookup.
- A momentum below version 3 holding a Settlement account-block is a consistency
  fault (SPEC §5.1 orders the sporks so this does not happen).
- New IPC request and reply types; `serve` answers them.
- Golden fixture: extend the go-zenon generator with one momentum holding accepted
  and rejected sends for one domain, and proofs at every height the tests read. The
  generator is test-only and goes into go-zenon under E1-10.

### 3. Follower model (worker)
- The follower drops `SettlementFilter`, its local index counter and
  `StartGlobalIndex`. It requests the assignment for each momentum holding a
  Settlement account-block and emits that momentum's inputs in `globalInputIndex`
  order.
- The worker finds each send by hash in the momentum the entry names, fetching that
  momentum through the existing momentum request when it has not followed it. A send
  not found is a consistency fault. Run 5 hash-links a fetched momentum older than the
  follower's window.
- The cursor's persisted next global index becomes the next expected domain ordinal.
  An assignment that does not start there is a consistency fault; it rolls back with
  the cursor on a reorg.
- `InputContext.MomentumHeight` stays the send's momentum height (SPEC §9). Resume
  and rollback stay momentum-granular.
- Rewire `runtime`, `worker`, `cmd/worker-wasm`, `internal/testbin/worker`,
  `internal/recovery/store.go`, and every filter-based test helper. Follower tests
  use synthetic in-memory sources.

### 4. Withdrawal caps
- The input context carries a cap lookup bound to the domain and the momentum
  confirming the input's receive. Step 5a calls it for each asset it withdraws; the
  worker memoises answers; the supervisor proves `AssetConfig` at that momentum and
  caches answers in a fixed-size cache (oldest evicted, no config knob). An absent
  `AssetConfig` is cap 0, so a non-zero total in that asset faults; a zero amount
  passes (SPEC §27.9).
- A lookup the supervisor cannot serve is a source outage: the worker ends and is
  restarted, never halts.
- The batch builder bounds each asset's aggregate by the highest cap step 5a used
  for it among the batch's inputs. Remove `codec.MaxBatchWithdrawal`.
- The execution vector file gains a per-vector cap table (an absent asset has cap 0).
  The existing withdrawal vector gets a cap above its amount; add vectors within the
  cap, above it, and in an unconfigured asset. Copy the regenerated file to zeno's
  `wasm-vm/`.

### 5. Hardening
- Every znnd JSON-RPC response body is read through a 32 MiB bound. An over-limit body
  is a source error, never a halt.
- A send momentum older than the follower's recorded window is hash-linked before the
  worker searches it. The follower fetches the heights from the send's momentum up to
  the oldest momentum it holds, in chunks that keep a full response under the body
  bound. Each momentum's `PreviousHash` must equal the hash of the one below it, and
  the top of the range must equal the held hash at that height. A range that does not
  link is a source error (`ErrSourceRead`), not a halt. A walked range serves every
  send in the same read. Sends inside the window keep the run 3 path.
- An input's cap lookups travel in one IPC request and reply. The supervisor still
  proves each asset and caches each answer; the worker memo is unchanged. A contract can
  cause at most 16 proof reads per input (`MaxAssetsPerBatch`; the footprint check runs
  before step 5a).
- Execution vectors `CV-CAP-5` to `CV-CAP-7` pin per-asset independence and the order of
  step 5a before step 6. Copy the regenerated file and README to zeno's `wasm-vm/`.
- The `wasip1` build of `cmd/worker-wasm` is not a supported target. The Dockerfile
  builds `./cmd/...` natively.

## Code references
- zeno-executor `internal/ledger/settlement.go` (`SettlementFilter`, `UnwiredFilter`),
  `internal/ledger/follower.go`, `domain/domain.go` (`InputContext`),
  `plugins/wasm/effects.go` (step 5a), `internal/batch/builder.go`,
  `internal/codec/constants.go` (`MaxBatchWithdrawal`), `internal/ipc/` [EXISTS].
- go-zenon `vm/embedded/definition/settlement.go` (ordinal entry, counters, asset
  config) [EXISTS after E1-10].

## Acceptance criteria
- Every `CV-INPUTROOT-1` leaf is reproduced from its ABI send plus the ordinal entry.
- A momentum holding accepted and rejected sends for one domain yields exactly the
  accepted ones, in global-index order; rejected sends consume nothing.
- A reorg below `C` rolls back and replays to the same input stream.
- A momentum with no Settlement account-block costs no proof; an assignment not
  starting at the next expected ordinal halts.
- A send older than anything the worker has followed (after a restart or a long
  receive lag) is found through the momentum the entry names.
- Step 5a: a withdrawal within the cap passes; above it, or a non-zero amount in an
  unconfigured asset, yields `RUNTIME_FAULT` with full rollback and refund; a zero
  amount in an unconfigured asset passes. New execution vectors cover each.
- A cap-source outage restarts the worker without a halt.
- A node response over 32 MiB fails as a source error for every RPC method.
- A send older than the window resolves only through a hash-linked range. A range whose
  `PreviousHash` chain breaks, or whose top differs from the held hash, is a source
  error.
- `CV-CAP-5`: two assets with different caps, each within its cap, give `SUCCESS`.
- `CV-CAP-6`: one asset over its cap and another within give `RUNTIME_FAULT` with full
  rollback and refund.
- `CV-CAP-7`: an input over a cap that would also run out of gas at the step 6 effect
  charge gives `RUNTIME_FAULT` from step 5a.
- One input's cap lookups cost one IPC round trip.
- `go test ./...` passes; replay determinism holds.
