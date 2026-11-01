# Ticket E1-05 — Deposit custody, conservation invariant, AssetFlowSummary

**Epic:** 1 — L1 Settlement · **Phase:** 1B · **Order:** E1-05

## Goal
Implement the value layer: the `Deposit` method, the aggregate per-(domain,
asset) custody counters, the on-chain conservation invariant enforced on every
mutating path, the canonical `AssetID` encoding, and `AssetFlowSummary` deposit
correspondence. End state: aggregate over-withdrawal is impossible on-chain,
independent of executor honesty.

## SPEC refs
SPEC §18.1 (AssetID), §18.1a (no protocol ledger), §18.2 (deposits/withdrawals),
§18.3 (conservation), §18.4 (AssetFlowSummary), §18.5 (deposit delivery/claim/
refund — the on-chain custody half); §5.3 (`Deposit`).

## Depends on
Tickets E1-03 (domain), E1-01 (counters storage).

## Key work
- `Deposit(domainId uint32, targetContract hash, plasmaLimit uint64)` — a typed ABI
  method with `domainId` first, the envelope shape every input method uses (SPEC
  §4.1); user send carrying a ZTS value. Send validation rejects a `Deposit` with
  no value (`Amount <= 0`, `ErrInvalidTokenOrAmount`, §18.2). The receive rejects a
  deposit to a domain whose `status` is `PAUSED` (§4.1);
  the receive rejects a non-custodyable asset for the domain — one with no
  `MaxBatchWithdrawal` configured for (domain, asset) (§18.5, Ticket E1-03's
  `SetAssetConfig`); this needs storage, so it cannot sit in `ValidateSendBlock`.
  On embedded receive, increase `totalDeposited[domainId][asset]` **unconditionally and
  finally** (L1 transfer already settled). Enqueue the deposit as a canonical
  input delivered to `target_contract` (empty payload); off-chain runtime handles
  claim/refund.
- Input-admission helper (shared, §4.6): on every valid input receive, assign
  the next global input index, record it against the domain, and update the
  per-(domain, asset) cumulative deposit totals at that index. An index is
  assigned on **acceptance**: a send the method rejects for any reason — an
  undecodable envelope, an unregistered domain, or a failed semantic check —
  consumes no index and carries no receipt (§4.6). This helper is consumed by
  every input method (this ticket's
  `Deposit`, plus Tickets E1-06 and E1-09) and is what Ticket E1-07 verifies batch bounds
  against.
- Canonical `AssetID` (§18.1): 32-byte ZTS encoding (`[0:22]` zero, `[22:32]`
  ZTS); reject non-zero high bytes (`UNSUPPORTED_OPERATION` on user input,
  `VALIDATION_FAILED` on `SubmitBatch`) and non-canonical alignment. One custody
  pot per real asset.
- Track only aggregates: `totalDeposited`, `pendingWithdrawalReserve`,
  `totalReleased` — **no per-account ledger**.
- Conservation check (§18.3) as a shared helper run after **every** mutating
  deposit/withdrawal/release path: revert if
  `totalReleased + pendingWithdrawalReserve > totalDeposited`
  (`ErrSettlementConservationViolated`). This is the **`ESCROW`** custody-mode
  invariant — the only mode in Phase 1. The `MINTED` predicates and their
  reconciliation (§18.3, OD-1) are reserved and out of scope here.
- **`MaxBatchMint` bounds predicate (§32).** `MaxBatchMintFloor`/`Ceiling`
  already exist with no predicate reading them. Add one symmetric to
  `IsMaxBatchWithdrawalInBounds`; it is unreachable while `ESCROW` is the only
  custody mode, and must exist before any `MINTED` asset can be registered.
- `AssetFlowSummary` verification hook (consumed by Ticket E1-07): on `SubmitBatch`,
  verify each summary `depositCredit` against the per-(domain, asset) cumulative
  deposit totals over the batch's assigned-index range (§4.6, §18.4 — incl.
  payable `CallL2` deposits). `depositCredit` = on-chain deposited amount,
  independent of contract claim outcome.
- Undeployed `target_contract` ⇒ the input fails `VALIDATION_FAILED` and the full
  `deposit_amount` is auto-refunded via the §18.5 path (runtime-emitted refund;
  on-chain side honours it through `RelayMessage` registration + `ReleaseWithdrawals`
  release, Ticket E1-08).

## Code references
- `vm/vm.go` (token burn/mint, descendant `ContractSend`, embedded
  auto-receive observing `sendBlock.TokenStandard`/`Amount`) [EXISTS].
- `vm/embedded/implementation/liquidity.go` (`LiquidityStakeMethod` — value +
  call in one send) [EXISTS].
- `common/types/tokenstandard.go` (ZTS) [EXISTS]; serializer AssetID (Ticket E2-03
  shares the encoding) [NEW].

## Acceptance criteria
- Deposit increases `totalDeposited`; a non-custodyable asset (no `MaxBatchWithdrawal`
  configured for the (domain, asset)) is rejected at the receive and is not an input.
- Input admission assigns strictly increasing indices in receive order across
  mixed input kinds; a send the method rejects consumes no index; cumulative deposit
  totals reproduce each deposit's amount at its assigned index.
- CV-ASSET-1/2 (round-trip; reject non-canonical) pass on the L1 side.
- Conservation helper reverts any operation that would breach the invariant
  (unit test each path).
- EXEC-DEP-3 preconditions: Settlement accepts a deposit to any `targetContract` (L1
  cannot see whether it is deployed), and the conservation helper admits a
  `pendingWithdrawalReserve` equal to the full deposit but not one unit more. The
  end-to-end refund (`VALIDATION_FAILED`, refund registered and released) is tested
  in Ticket E1-08.
- A zero-value `Deposit` fails send validation; a `Deposit` to a `PAUSED` domain is
  rejected at the receive, consumes no index, and its value is returned.
