# Ticket E2-16 — L1 read path: znnd client, Settlement storage reader, recovery anchor

**Epic:** 2 — Executor · **Phase:** 1C · **Order:** E2-16

## Goal
Give the supervisor a verified read path to L1. End state: the supervisor follows a
real `znnd` node over JSON-RPC and serves momenta to every worker, reads any
Settlement storage entry through `ledger.getProof` verified against the momentum
header's state root, and answers the workers' recovery-anchor requests from the
domain's finalized batches. Workers boot from a real anchor instead of the genesis
fallback.

## SPEC refs
EXECUTOR.md §1.4 (allowed imports), §5.1, §5.5 (read path), §14 (recovery anchor),
§17 (configuration, node requirements), §18 (P-6, P-8); SPEC §5.4 (storage
responsibilities), §13.1 (path-native verification), §19, §21 (finality).

## Depends on
E2-08 (anchor interface), E2-13 (supervisor, IPC). E1-10 for the exported key
builders; until it lands, derive keys from the documented prefix table and switch
to the builders when they exist.

## Key work
- **znnd client.** A thin JSON-RPC client over `net/http` in the supervisor, decoding
  into `chain/nom` types. It imports no `rpc/*` package (EXECUTOR §1.4). It serves
  `ledger.MomentumSource` (confirmed momenta and their content), the frontier
  momentum, `ledger.getProof`, `ledger.getStateRoot` and the frontier account block.
  New required config field `znndEndpoint` (EXECUTOR §17); transient failures use the
  supervisor's bounded backoff, never a halt.
- **Storage reader.** For a Settlement storage key `k`, the state-tree key is
  `0x03 ‖ SettlementContract(20) ‖ 0x04 ‖ k`. Request `getProof` at a confirmed
  height, check the returned root against that momentum header's `StateRoot`, and
  verify with `VerifyProofByPath(root, types.NewHash(rawKey), …)`. A mismatch is a
  consistency fault (HALT). Decode values with `ABISettlement.UnpackVariable`;
  processed-outbox membership is the literal byte `0x01`.
- **Typed reads** used by later tickets: domain record and status, input-sequence
  cursor, batch entry, domain received-input counter, ordinal entry, asset config,
  global and domain pause state, executor-set member, processed-outbox membership.
- **Recovery anchor.** Replace `unwiredAnchorSource`. The anchor is the last batch of
  the domain's contiguous finalized prefix (`height ≥ finalizeAtHeight` for every
  batch from 0), read at the confirmed frontier (node frontier minus
  `confirmationDepth`). No accepted batch, or none finalized, is the genesis anchor.
- **Node checks at boot.** The node serves version-3 momenta (`StateRootSpork`
  active) and returns a proof at the confirmed frontier; otherwise the supervisor
  reports the node as unusable and retries.

## Code references
- zeno-executor `cmd/executor/main.go` (`unwiredMomentumSource`, `unwiredAnchorSource`),
  `internal/supervisor/` (config, serve), `internal/recovery/boot.go`
  (`AnchorSource`) [EXISTS].
- go-zenon `rpc/api/ledger.go` (`GetStateRoot`, `GetProof`,
  `GetDetailedMomentumsByHeight`), `chain/state_tree.go` (retention),
  `common/trie/proof.go` (`VerifyProofByPath`), `vm/embedded/definition/settlement.go`
  (prefix table, key builders) [EXISTS].

## Acceptance criteria
- Against a `httptest` JSON-RPC fake serving golden data captured from a go-zenon
  node, the supervisor serves momenta to a worker and reads every typed entry above.
- A tampered proof, value or root fails verification and halts; a node outage
  retries without halting.
- The anchor is the end of the finalized prefix, including the case where a later
  batch finalizes before an earlier one; genesis when nothing is finalized.
- A worker boots from a real anchor and reconciles; an anchor mismatch halts
  (closes E2-08 against the real path).
- `go test ./...` passes; `scripts/check-trie-imports.sh` passes.
