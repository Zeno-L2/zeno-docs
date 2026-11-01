# Ticket E2-10 — Close the stale pending-record ownership-capture path

**Epic:** 2 — Executor · **Phase:** 1A · **Order:** E2-10
**Size:** S — one halt check, one EXECUTOR amendment, one executable test, one test-only cosmetic fix.

## Goal
Ticket E2-09's contract-metadata change (`policy`, `owner`) widened an already-documented,
non-attacker-triggerable reorg hazard from a status divergence into a silent ownership capture.
Close it with a local halt check, and update EXECUTOR §15 to name the widened blast radius. A
one-line test-message fix from the same review rides along.

## SPEC refs
EXECUTOR §15 (failure handling, the "unrepaired path"). SPEC §11.3 (finalize's deployer
binding), §11.5 (pending-record expiry).

## Depends on
Ticket E2-09 (delivered). This hazard exists only because E2-09's metadata write reads fields from
the same node-local pending record whose `total_size`/`chunk_count`/`deployer` EXECUTOR §15
already documents as reorg-corruptible.

## Key work

### 1. EXECUTOR §15: name the widened blast radius
The "unrepaired path" paragraph says a stale pending record surviving rollback makes "a
canonical finalize assemble against the wrong `total_size`, `chunk_count` and deployer, and
commit a different status from a node that never reorged." Since Ticket E2-09, the same stale
record also supplies `upgrade_policy` and `owner`, with nothing binding them to the canonical
finalize's actual deployer. Add `upgrade_policy` and `owner` to that field list, and state the
consequence precisely: an unrelated second `DeployContractStart`'s sender becomes the finalized
contract's owner, who can then upgrade it to arbitrary code or transfer it. Keep the existing
operator remedy (stop, remove the domain and code-store directories, re-sync from genesis)
unchanged — this ticket adds a local halt, not a structural fix; EXECUTOR §15 should keep noting
that closing the class needs a plugin rollback seam.

### 2. `plugins/wasm/deploy.go`: halt on a deployer mismatch at finalize (consensus-safe)
`prepareModule` (`:76`) currently returns `(codeHash, metadata, ok, err)` and deliberately never
lets the pending record's `Deployer` leave the function, because using it as a consensus source
would let a stale local entry derive a different `ContractID` than honest nodes (`:56-59`,
`:183-188`). That reasoning does not extend to a local-only halt check, which every node reaches
independently: extend `prepareModule` to also return the record's `Deployer`, and update its
doc comment to say it may leave the function for a same-node comparison only, and must never
feed `contractID` or otherwise become consensus-visible.

In `applyDeployFinalize` (`:169`), after decoding `metadata` (`:178`), compare the returned
`Deployer` against `ctx.L1Sender`. On a mismatch, return `(domain.Effects{}, err)` — a halt, not
a status — with an error naming it a stale pending record requiring local recovery, mirroring
the existing `err != nil` halt convention at this call site (`:171-173`, "this node cannot see
what other nodes can"). Honest Settlement guarantees the two agree (§11.3), so the mismatch
branch is reachable only on the stale-record path this ticket targets.

`applyUpgrade`'s call site (`:251`) already discards `prepareModule`'s metadata and needs no
change: it has its own state-backed owner-authorization check, unrelated to this record.

### 3. Executable test for the hazard
`internal/recovery/boot.go`'s and `internal/batch/driver.go`'s reorg-rollback comments describe
the sequence (deploy start A, its record expires per §11.5, deploy start B reuses the same
`code_hash` with a different deployer, a canonical finalize for A then assembles against B's
surviving record) but nothing executes it. Add a `plugins/wasm` test that builds this sequence
directly against the code store and asserts the finalize now halts (returns a non-nil error)
rather than committing `SUCCESS` with B's deployer as owner. Cover the matching-deployer case
too, confirming it still finalizes normally.

### 4. Cosmetic: a misleading conformance-test failure message
`internal/codec/vectors_test.go`'s `testMeta1` formats a mismatched owner with `%x` on
`m.Owner`, a `types.Address` that implements `String()`; `%x` on a `Stringer` renders the hex of
its `String()` output, not the raw 20 bytes, so a real mismatch prints a misleading value on
failure. Change it to `hex.EncodeToString(m.Owner[:])`. No behaviour change; the check itself is
correct today.

## Out of scope
- A structural fix (a plugin rollback seam that repairs the pending-record store on reorg,
  closing the hazard class rather than halting on this one instance of it). EXECUTOR §15 already
  flags this as future work.
- Any other reorg-adjacent hazard. This ticket is scoped to the one escalation Ticket E2-09's
  review surfaced.

## Acceptance criteria
- `go test ./...` passes.
- The new `plugins/wasm` test proves both directions: a matching deployer still finalizes
  `SUCCESS`, and a stale-record deployer mismatch halts with a local-recovery error instead of
  committing a status.
- EXECUTOR §15's unrepaired-path paragraph names `upgrade_policy` and `owner` among the
  corrupted fields and states the ownership-capture consequence.
- `internal/codec/vectors_test.go`'s owner-mismatch failure message renders the raw address
  bytes, not `String()`'s hex.
