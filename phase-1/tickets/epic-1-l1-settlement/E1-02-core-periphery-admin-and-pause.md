# Ticket E1-02 — Core/Periphery, administration, emergency pause

**Epic:** 1 — L1 Settlement · **Phase:** 1B · **Order:** E1-02

## Goal
Implement the administration layer: a single global network administrator
address gating Settlement's admin-only methods, Core hard-bound rejection of
out-of-bounds Periphery values, and the two-scope emergency pause. End state:
Periphery is admin-tunable within Core bounds; Core is unreachable by the
admin; pause works global and per-domain.

## SPEC refs
SPEC §5.2 (Core/Periphery), §23 (administration, hard bounds, escape hatch,
emergency pause), §27.10 (relay under pause).

## Depends on
Ticket E1-01. Provides the admin gate consumed by Tickets E1-03, E1-04, E1-05, E1-07, E1-08.

## Key work
- Admin-gated methods verify `sendBlock.Address == types.NetworkAdmin`
  (`common/types/address.go`) at receive time and otherwise reject with
  `ErrPermissionDenied`. Settlement stores no administrator address of its
  own and exposes no method that changes the network administrator; the
  address is a build-time value in the node binary, rotated by a node
  release plus an activation spork, not by an on-chain call.
- No guardian recovery exists for this address. There is no on-chain
  break-glass and no on-chain rotation ceremony; Settlement stores nothing
  to recover.
- Periphery-parameter changes are time-challenged with the shared
  `TimeChallenge` helper (`vm/embedded/implementation/common.go`): soft
  parameter changes (`SetPeripheryConfig`) take effect only after
  `SoftDelay`, the escape hatch and `RegisterExecutor` only after
  `AdministratorDelay`. Both delays are fixed Core constants —
  `MinSoftDelay` (`MomentumsPerEpoch`) and `MinAdministratorDelay`
  (`2×MomentumsPerEpoch`) — applied directly by the method, with no
  administrator override.
- Periphery setters: `DAMode` set, withdrawal-delay/challenge window,
  `MaxBatchWithdrawal`, runtime-upgrade delay, per-domain executor identity
  (`RegisterExecutor`) + `valueCaps`, reserved Phase 2/3 verifier
  registries. Periphery carries no address fields — no guardian set, no
  pause-authority address; the same network administrator address exercises
  both administration and emergency pause. **Core MUST reject** any
  out-of-bounds value with `ErrForbiddenParam` rather than clamping it to a
  bound — even a compromised administrator cannot exceed the immutable hard
  bounds (Ticket E1-01 holders).
- `Pause` method: scopes `PAUSE_SUBMIT` (halts `SubmitBatch`) and `PAUSE_RELAY`
  (halts `RelayMessage` registration and `Update` release), settable global or
  per-domain; each call names exactly one scope — `PAUSE_RELAY` is never
  implied by `PAUSE_SUBMIT` and is reserved for an active asset-safety
  exploit. Both scopes at both granularities are durable, publicly readable
  on-chain records. Pause never rewrites history.
- Administrator prohibitions (§23) enforced: cannot alter finalized
  roots/receipts/balances/order/conservation/SMT/serialization formats.
- Domain escape hatch (§23 carve-out) — implement the method signature,
  admin gate, `AdministratorDelay` time-lock and a durable, publicly
  readable escape-hatch record identifying the domain and the momentum
  height it was initiated at, now; the forced-withdrawal mechanics
  themselves are a tracked follow-up (§32).

## Code references
- `vm/embedded/implementation/common.go` (`TimeChallenge` ~:439),
  `common/types/address.go` (`NetworkAdmin`).

## Acceptance criteria
- Admin-gated methods accept only `sendBlock.Address == types.NetworkAdmin`
  and reject every other sender with `ErrPermissionDenied`, checked at
  receive time before any state write.
- `SetPeripheryConfig` takes effect only after `SoftDelay`; a pending change
  is observable before it activates.
- A Periphery value outside a Core hard bound is rejected with
  `ErrForbiddenParam`, never clamped (test the `MaxBatchWithdrawal` ceiling
  and the runtime-upgrade-delay floor).
- `PAUSE_SUBMIT` blocks `SubmitBatch` but withdrawals from `FINALIZED` batches
  still register and release; `PAUSE_RELAY` requires its own call and is
  recorded as its own durable, publicly readable pause-state field,
  independent of `PAUSE_SUBMIT`; deactivation restores releasability in
  unchanged registration order.
- `InitiateEscapeHatch` takes effect only after `AdministratorDelay` and
  writes a durable, publicly readable escape-hatch record naming the domain
  and the momentum height it was initiated at.
