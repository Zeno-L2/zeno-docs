# Ticket E2-22 — Gateway: unified read API and HTTP bundle serving

**Epic:** 2 — Executor · **Phase:** 1C · **Order:** E2-22

## Goal
Serve L2 data to clients through one API for every domain, from a process that
holds no key. End state: `cmd/gateway` rebuilds each served domain's state from DA
bundles and receipt sidecars, checks every batch against Settlement, and serves the
`zeno_*` JSON-RPC API and HTTP bundle serving (EXECUTOR.md §21); the WASM read
adapter serves contracts; the devnet runs a gateway by default.

## SPEC refs
EXECUTOR.md §21 (gateway), §1.5 (process isolation), §7 (`Reader`), §16 (read
surface), §17 (gateway configuration); SPEC §13.5.1 (canonical applier), §13.6
(absent vs present-empty), §15/§27.7 (receipts), §16 (event inclusion proofs from
the bundle), §20 (bundle layout, `MaxDAChunkSize`), §21 (finality), §27.3
(`MerkleProof`), §27.5 (canonical input encoding, send hash).

## Depends on
E2-16 (proof-verified Settlement reads), E2-19 (accepted batches, devnet harness), E2-20 (revert data in the `REVERT` `ExecutionResult`),
E2-21 (the `MerkleProof` builder, reused for outbox proofs).

## Key work
Seven `/develop` runs on one gateway feature branch, in this order: receipt sidecar;
gateway core; indexes and generic API; bundle serving and `REMOTE`; views; simulation;
deployment and the devnet run. Bundle serving runs with `REMOTE`, because `REMOTE`
fetches from it. The seams below give each run's content.

### 1. Receipt sidecar
- The batch driver writes `receipts/<daHash>.receipts` right after the bundle, when
  it builds the batch and before the batch record (EXECUTOR.md §16). The file holds
  exactly the batch's receipt encodings as a `List<Bytes>`, ascending by
  `globalInputIndex`, with no header; its extension is its format version. Retained
  like bundles.
- A note in `internal/recovery` that `batches/`, `bundles/` and `receipts/` are a
  read interface: a format change a reader cannot parse is a new version.
- Read grant: files in the three read-interface directories are written 0644 and the
  directories 0755; the domain directory is 0711; every other file, the state WAL
  included, is 0600. All modes are set explicitly, never left to umask. Executor data
  written before the sidecar has none and needs a re-sync.

### 2. Gateway core
- `cmd/gateway` with the §17 configuration plus a required `confirmationDepth` and
  `networkName`, validated at boot like the executor's. A gateway `storagePath` that
  overlaps `executorStoragePath` is rejected. An import deny-list test keeps the
  supervisor, worker runtime, batch builder, `internal/ipc`, the WASM engine and the
  wallet out of the gateway binary.
- `LOCAL` source: read batch records, bundles and sidecars read-only from the
  executor's domain directory. Batch records carry only `BUILT`; acceptance comes from
  Settlement.
- Apply each input's `StateDiff` with `internal/state`'s applier. Versions share
  structure through the immutable `CompactTree` and never change once derived; every
  value is copied before insert.
- Before serving a batch at `latest`, run the four §21.2 checks, plus: each receipt
  names its own input, input hash and batch; and the commitment's `firstInputSeq`,
  `lastInputSeq`, `inputRoot`, `eventRoot` and `outboxRoot`, and each receipt's event
  and outbox counts, match the bundle. Settlement checks none of these. Mark the
  domain `INCONSISTENT` on a failed check.
- Re-read each accepted entry once, when its batch finalizes; a changed entry marks
  the domain `INCONSISTENT`, and finality follows the re-read `FinalizeAtHeight`.
- Keep state versions from the oldest non-finalized batch through `pending`, plus
  `finalized` (§21.6). Losing pending batch N's record, or a change to its `daHash`,
  retracts N and every pending batch above it.
- The gateway serves only domains with an empty genesis state, since it loads no
  plugin; a seeded domain goes `INCONSISTENT` with that reason.
- Bundles are read up to a 1 GiB memory ceiling (`maxBundleBytes`, shared by `LOCAL`
  reads and `REMOTE` fetches). It is not a protocol bound: no rule bounds bundle bytes
  (SPEC OD-5(e)). After a batch passes its checks the gateway keeps no raw bundle
  bytes and no witnesses, only each record's input data, `ExecutionResult` and the
  version; raw bytes are re-read from disk on demand.
- The `finalized` state and the indexes persist under the gateway's `storagePath` as
  a cache, checked against Settlement at boot and rebuilt from bundles when it fails.

### 3. Indexes and generic API
- Indexes: send hash → input, receipts, events, outbox messages, receipt
  `contractId` → inputs, L1 sender → inputs, and each input's L1 sender read from
  `znnd` by its send hash (§21.6).
- JSON-RPC 2.0 over HTTP and WebSocket: every generic method in §21.3, tag
  resolution, `verified` and `batchId` on every response, the §21.3 error codes,
  and `zeno_subscribe` for `heads`, `events` and `receipt`.
- `zeno_getOutboxProof` builds the §27.3 proof with E2-21's builder and reports
  `processedOutbox` from Settlement.
- A small JSON-RPC server on `net/http` and `gorilla/websocket`: batches of at most
  100 requests, 1 MiB request bodies, 100 results per page, 64 paths per proof
  request, 16 subscriptions per connection. A WebSocket upgrade with an `Origin`
  must match `allowedOrigins` exactly; one without is accepted.

### 4. Bundle serving and `REMOTE`
- `GET /bundles/<daHash>` and `GET /receipts/<daHash>` with range requests (§21.5),
  extending the write deadline per chunk so a whole-file transfer is never cut.
- `REMOTE` source: read batch entries from Settlement, fetch bundles and sidecars
  from `upstreams`, run the checks, skip an upstream that fails them.

### 5. Views and profiles
- `zeno_read` and the `Reader` interface (EXECUTOR.md §7, §21.4): view and
  profile declaration, parameter validation, profiles and views in
  `zeno_chainInfo`, and the paths-and-proofs rule for state-backed views.
- The WASM `Reader` in its own package, `plugins/wasm/reader`, so the WASM engine
  stays out of the gateway: input decoding for display and the `contracts/1` profile
  with its feature flags (`nativeAddresses` unset). Linked into the gateway, never
  into a worker.
- `module` takes the as-deployed `code_hash`. `contract` returns `ownerCaller` (the
  owner leaf, with its proof) and `owner`, the L1 address from the latest successful
  deploy (the `DeployContractStart` metadata) or `TransferContractOwnership` naming
  the contract, returned only after `SHA3-256("L1_CALLER:" || owner) = ownerCaller`
  holds.
- `allowedOrigins` CORS handling.

### 6. Simulation
- Simulate mode in the generic `worker` package: no domain directory, no cursor, no
  persistence, state through a read-only accessor served by the gateway over a local
  channel (EXECUTOR.md §1.5, §21.8).
- `zeno_simulate` in the gateway: a per-domain pool of worker processes, the
  `simulation` limits, `-32006` on a crash, timeout or over-limit request, and
  process replacement after a crash. A per-domain `workerBinary` setting names the
  executable. Only `CallL2` and `Deposit` inputs are simulated.
- A simulator has no domain directory, so it asks the gateway for contract code by
  `code_hash`, verifies it against the `code_hash` and `CodeHash` leaves, and caches
  instrumented modules in memory up to 32 MiB.

### 7. Deployment
- A `gateway` service in `docker-compose.yml`, on by default, `LOCAL` over the
  executor's volume mounted read-only. It listens on all interfaces inside its
  container, is published only on the host's loopback, and runs as a non-root user.
- Operator docs: configuration, exposure, the read-access grant for `LOCAL`, and
  that the gateway checks consistency with Settlement but is not a watcher.
- One devnet run in total, extending the existing devnet test.

### Documentation, after the runs
EXECUTOR.md: §16 (read-interface file modes), §17 (`confirmationDepth`,
`networkName`, `workerBinary`), §21.2 (the commitment binding, the finalization
re-read and the empty-genesis limit), §21.4 (`module` takes the as-deployed
`code_hash`; the `owner` and `ownerCaller` fields), and §21.6 (the bundle memory
ceiling).

## Code references
- zeno-executor `internal/state` (`Store.Apply`, `Store.Prove`, snapshots),
  `internal/codec` (`DecodeBundle`, `Receipt`, `merkle.go`), `internal/recovery`
  (`Layout`, `ListBatchRecords`, `BundleStore`), `internal/settlement` (`reader.go`),
  `internal/znnd` [EXISTS].
- zeno-executor `internal/batch/builder.go` (the bundle is written when the batch is
  built) [EXISTS].
- go-zenon `common/trie` path-native API [EXISTS].

## Acceptance criteria
- On the devnet, a `LOCAL` gateway serves a `CallL2`'s receipt by send hash at
  `pending`, then `latest`, then `finalized`, with `verified = true` from `latest` on.
- A `REMOTE` gateway syncing from the `LOCAL` one returns byte-identical `latest` and
  `finalized` responses, and serves no `pending`.
- A tampered bundle, sidecar or batch root marks the domain `INCONSISTENT`; nothing
  past the last good batch is served.
- A retracted pending batch disappears from `pending` and from every index.
- Every returned proof verifies with `VerifyProofByPath` / `VerifyAbsenceByPath`
  against the root the response names; absent and present-empty values stay distinct.
- `zeno_getInputs` filtered by a contract returns its finalize and every later
  input naming it, failed ones included, newest first.
- The `contract` view reports a deployed contract's policy, owner and quota, and
  its returned paths and proofs reproduce that result; the `module` view returns
  bytes hashing to `code_hash`.
- `zeno_read` on a domain without a `Reader` returns `-32005`; every generic method
  still serves it.
- `zeno_simulate` of a `CallL2` returns the same status, `gasUsed`, `ReturnData` or
  revert data, and `StateDiff` as the real input when nothing else lands between
  them; a `plasmaLimit` above `maxGas` is `-32006`; killing a simulator mid-call
  returns `-32006` and the next call succeeds.
- The gateway reads nothing in the supervisor's directory and opens no connection to
  a domain worker or the supervisor; its only child processes are simulate-mode
  workers; E2-13's key-isolation tests still pass.
- `go test ./...` passes.
