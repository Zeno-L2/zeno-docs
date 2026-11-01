# Ticket IMP-01 — Uncapped custody for user-issued tokens

**Status:** Draft · **Repos:** zeno (SPEC), go-zenon (Settlement)

## Problem
An asset is custodyable for a domain only once the administrator sets its
`MaxBatchWithdrawal` through `SetAssetConfig` (SPEC §5.2, §18.5). An asset with no
configured cap has a cap of `0`: Settlement rejects deposits of it, and the
executor faults any non-zero withdrawal of it (§27.9 step 5a).

A developer who issues a ZTS for their own contract therefore cannot use it until
the administrator acts, and then waits a further `SoftDelay` for the time
challenge (§23). The gate also blocks any contract that wants to accept arbitrary
ZTS.

## Why the gate is not needed for user-issued tokens
Conservation is keyed per `(domain, asset)` (§18.3). A fraudulent or buggy batch
in a user-issued token can pay out only that domain's deposits of that same token.
It can never reach ZNN, QSR or any other asset's custody. The exposure falls only
on holders who chose to deposit that token.

## Proposal
Classify each asset by its ZTS owner:

1. **Embedded-owned** — the token's owner is an embedded contract address (ZNN and
   QSR, owned by the token contract). Custody stays administrator-configured,
   exactly as today.
2. **User-issued** — every other ZTS. Custody is uncapped: its
   `MaxBatchWithdrawal` is the maximum `U256`, and conservation is its only bound.

### Mechanism: Settlement writes the record on first deposit
On the first accepted deposit of a user-issued token into a domain, Settlement
writes that `(domain, asset)`'s `AssetConfig` with `MaxBatchWithdrawal = MaxU256`.

- The executor's cap read (`zeno-executor/internal/settlement/caps.go`) proves the
  record exactly as it does today. The record exists at the momentum confirming
  the deposit's receive, so step 5a needs no token-ownership proof and the
  executor is unchanged.
- The classification is taken once, at first deposit. A later `UpdateToken`
  ownership transfer does not move an asset between classes.
- The monotonicity rule (§27.9) holds unchanged: no `SetAssetConfig` can lower
  the cap of a user-issued asset.
- An asset never deposited into a domain has no record and keeps a cap of `0`, so
  a withdrawal of it still faults rather than strands.

### Unchanged
- The refund bound against `depositCredit` (§27.9).
- `AssetFlowSummary` correspondence (§18.4).
- Conservation (§18.3) and pause (§23).

## Trade-offs
- A user-issued token has no per-batch loss bound. A bug in the guest program or
  zkVM — the risk Phase 2 keeps `MaxBatchWithdrawal` for (`../SPEC.md` §4.6) —
  can drain one domain's full deposits of that token in one batch.
- Embedded-owned tokens, which hold the most value, keep their caps.

## Open questions
1. Which embedded contract owns the bridge-minted wrapped tokens? If it is an
   embedded address, those tokens fall in class 1 automatically, which is the
   intended outcome. Confirm against `go-zenon` genesis and the bridge contract.
2. Does `SetAssetConfig` reject a user-issued asset outright, or accept only
   `MaxU256`? Rejecting it is simpler and leaves no admin lever over the class.

## SPEC edits
§5.2 (custodyable definition), §18.5 (deposit admission), §23 (`SetAssetConfig`
scope), §27.9 (step 5a cap source).

## Code touch points
- go-zenon Settlement: deposit receive (`admitInput` path) writes the record for a
  user-issued token; `SetAssetConfig` validation per open question 2.
- zeno-executor: none expected.
