# Ticket E2-06 — Input pipeline: ledger follower, canonical ordering, cursor

**Epic:** 2 — Executor · **Phase:** 1A · **Order:** E2-06

## Goal
Follow the `znnd` confirmation frontier, reconstruct the canonical input stream
from confirmed momenta, assign `globalInputIndex`, filter to a `domainId`, and
maintain the per-domain input-sequence cursor — reproducing L1 order with zero
sequencing authority. End state: two independent runs derive identical input
sequences for identical momentum content. Settlement decode is stubbed (plugin-
decoded) until Ticket E2-17.

## SPEC refs
SPEC §4 (canonical input stream, ordering, global index, per-domain contiguity),
§4.3 (confirmed-momentum requirement); EXECUTOR.md §5.1, §8, §9.

## Depends on
Ticket E2-01. (Real Settlement address/ABI decode → Ticket E2-17; until then decode
behind a plugin stub.)

## Key work
- Advance to `frontier − confirmationDepth`; read confirmed momenta ascending.
- Canonical order = momentum height asc, then `AccountHeader.Bytes()` asc
  (`Address‖Height(8B BE)‖Hash`); reproduce with **no** alternative ordering.
- Assign `globalInputIndex` (`uint64`, zero-based, monotonic, never reset, halt
  on overflow) over the **global** stream before filtering.
- Filter content to the configured `domainId`; track each domain's last-consumed
  `globalInputIndex` **independently** — consecutive consumed indices may jump
  (domain A `{5,7,9}`); never require `next == prev+1` (§4.6).
- Identify Settlement-destined account-blocks and hand the plugin an opaque
  `{domainId, payload}` to `DecodeInput` (the six input kinds, §4.1). The
  envelope's encoding is fixed by Ticket E1-09; decode against it rather than
  inventing one.
- Assign an index on **acceptance** (§4.6): only an account-block whose
  addressed Settlement method accepted it is an input. A rejected send consumes
  no index and produces no receipt, so the executor's numbering matches
  Settlement's without a second predicate.
- Reorg-aware: detect momentum-hash change at a previously consumed height (hook
  for Ticket E2-08's reorg handling).

## Code references
- `rpc/api/ledger.go` `GetDetailedMomentumsByHeight` (~:446),
  `GetMomentumsByHeight` [EXISTS].
- `common/types/account_header.go` `Bytes()` (~:41) [EXISTS].
- `chain/nom/momentum_content.go` sort by `bytes.Compare(...) <= 0` (~:53)
  [EXISTS].
- Settlement address `common/types/address.go` + ABI
  `vm/embedded/definition/settlement.go` — [NEW], stubbed here, wired in 18.

## Acceptance criteria
- Two-implementation determinism: identical momentum content → identical
  `globalInputIndex` assignment and identical filtered subsequence.
- Per-domain jump test: interleaved A/B inputs yield A subsequence `{5,7,9}` with
  a cursor that advances 5→7→9, never tripping a raw-contiguity check.
- Inputs below the confirmation frontier are never consumed; overflow halts
  admission rather than wrapping.
