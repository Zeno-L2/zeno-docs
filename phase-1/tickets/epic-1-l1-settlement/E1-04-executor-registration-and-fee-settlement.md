# Ticket E1-04 — Executor registration + fee settlement

**Epic:** 1 — L1 Settlement · **Phase:** 1B · **Order:** E1-04

## Goal
Bind the Phase-1 executor to a pillar, and collect, account and pay developer
fees. End state: the network executor set holds exactly one pillar, chosen by the
administrator with a time challenge; its executor address is the only one that can
submit; fees collected at the L1 receive accrue per `(epoch, pillar)` at batch
acceptance and settle through a no-argument `Update` — 30% burned, 70% credited to
the pillar's `RewardWithdrawAddress` and paid by `CollectReward` from Settlement's
own balance. No bond exists anywhere.

## SPEC refs
SPEC §22 (executor model — identity, registration, attribution, fee settlement),
§6.2 (network executor set, `SINGLE`), §10.7 (developer fees), §5.3
(`RegisterExecutor`, `Update`, `CollectReward`, `SetPeripheryConfig` fee prices),
§5.4 (executor-set and fee storage), §23 (admin gate, `AdministratorDelay`,
`PAUSE_RELAY`), §25 (`FeeBurnShare`, fee-price bounds).

## Depends on
Tickets E1-01 (storage — reworked here), E1-02 (admin gate, pause), E1-03 (domain exists).

## Key work
- **Storage rework (reverses part of Ticket E1-01).** Remove the bond records keyed by
  `(domainId, executorAddress)` and the per-address QSR deposit balances; remove
  `executors` from `domainRecord`. Add the network executor set (one record per
  pillar: name, executor address, registration height, status), the per-input fee
  on the assigned-index record, per-`(epoch, pillar)` pending amounts, the queue of
  epochs holding pending amounts with each epoch's highest batch finalization
  height, the four fee counters (`totalFeesCollected`, `totalFeesBurned`,
  `totalFeesCredited`, `totalRewardsCollected`), and a ZNN `RewardDeposit` per
  reward address. No per-epoch burn total is stored. Executor status follows SPEC
  §27.11 (`ACTIVE = 0`, `LAPSED = 1`, `COOLDOWN = 2`). Phase-2 member fields (last
  accepted batch, missed turn, cooldown expiry) are not part of the member record;
  Phase 2 adds them under their own key prefix keyed by pillar name.
- **`RegisterExecutor(pillarName, executorAddress)`** — admin-only,
  `TimeChallenge(context, "RegisterExecutor", paramsHash,
  constants.MinAdministratorDelay)`; reject unless `pillarName` is an active pillar
  (`context.MomentumStore().GetActivePillars()`). Under `SINGLE` it replaces the
  sole member. The outgoing pillar keeps its pending amounts.
- **ACTIVE predicate for Ticket E1-07:** caller equals the member's executor address
  **and** the member's pillar is active at the receive. Expose the pillar name for
  `executorPillar` in the stored batch entry. Add both new `batchEntry` fields here,
  since the accrual hook reads them: replace `executorIndex` with `executorPillar`,
  and add `finalizeAtHeight` (SPEC §19, §21). Ticket E1-07 sets their values.
- **Accrual hook for Ticket E1-07:** on acceptance, sum the fees recorded with the
  batch's inputs and add them to `(E, pillar)` with `E = submittedAtHeight /
  MomentumsPerEpoch` (height-based, SPEC §22.4); append `E` to the pending-epoch
  queue when it is newer than the last entry, and raise `E`'s finalization height to
  the batch's `finalizeAtHeight` (fixed by Ticket E1-07 at acceptance, SPEC §21).
- **`Update()`** — no argument, `checkAndPerformUpdate` rate limit; settle at most
  the epoch at the head of the pending queue, only when it has ended and the
  frontier has reached its recorded finalization height. Empty epochs are never
  queued, so there is nothing to anchor and no backlog after quiet periods. Per
  pillar: `burn = floor(P × 30 / 100)`, credit `P − burn` to the pillar's current
  `RewardWithdrawAddress`, burn all of `P` for a pillar absent from
  `GetActivePillars()`; one descendant ZNN burn send per epoch. When the head is not
  settleable, the queue is empty, paused, or the guard would break, `Update`
  succeeds without settling and still consumes its rate limit. Add `types.SettlementContract` to
  `types.EmbeddedWUpdate` so pillars crank it automatically.
- **`CollectReward()`** — pay the caller's ZNN `RewardDeposit` from Settlement's
  balance with one descendant send; never mint.
- **Guard (always on):** never perform a settlement or collection that would make
  `totalFeesBurned + totalFeesCredited > totalFeesCollected` or
  `totalRewardsCollected > totalFeesCredited`.
- **Pause:** while `PAUSE_RELAY` is in force globally or for any domain, `Update`
  does not settle and `CollectReward` fails with an error; never alter pending or
  credited amounts.
- **Rename the `Update(domainId)` stub to `ReleaseWithdrawals(domainId)`** here,
  still returning `ErrNotImplemented`, because one ABI cannot hold two `Update`
  functions. Ticket E1-08 adds the release logic behind it.
- **Fee-price bounds (pre-mainnet placeholders):** floor 1 base unit, ceiling
  1 ZNN per byte, for both prices.
- **Fee prices:** add `znnPerByteOfBytecode` and `znnPerByteOfState` to
  `SetPeripheryConfig` with `IsZnnPerByteOfBytecodeInBounds` /
  `IsZnnPerByteOfStateInBounds` (non-zero floors) in `vm/constants/settlement.go`.
  Fee collection at the receive lands with the methods that carry fees: Ticket E1-06
  (`DeployContractFinalize`) and Ticket E1-09 (`UpgradeContract`, `BuyStorage`).

## Code references
- `vm/embedded/implementation/common.go` (`TimeChallenge`, `checkAndPerformUpdate`,
  `checkAndPerformUpdateEpoch`, `addReward`, `CollectRewardMethod` — whose mint
  path this ticket must **not** reuse) [EXISTS].
- `vm/embedded/implementation/pillars.go` (`UpdateEmbeddedPillarMethod`, the epoch
  crank precedent) [EXISTS].
- `chain/momentum/embedded.go` (`GetActivePillars`),
  `vm/embedded/definition/pillars.go` (`PillarInfo`) [EXISTS].
- `pillar/worker_updater.go`, `common/types` (`EmbeddedWUpdate`) [EXISTS].
- Ticket E1-01 storage, Ticket E1-02 admin gate and pause state [EXISTS on
  `epic/settlement`, reworked here].

## Acceptance criteria
- `RegisterExecutor` takes effect only after `AdministratorDelay`; naming an
  inactive or unknown pillar is rejected; a second member is impossible under
  `SINGLE`.
- A batch from the executor address is accepted only while the pillar is active;
  after the pillar is revoked it is rejected (verified end-to-end with Ticket E1-07).
- Fees accrue to the submitting pillar for the epoch of `submittedAtHeight`; after
  a replacement, the old pillar still receives what its batches earned.
- `Update` settles nothing until the head epoch's finalization height is reached;
  afterwards it burns exactly the burn share, credits the rest, and removes the head
  epoch. Epochs with no batches are skipped: two epochs with accruals separated by
  idle epochs settle in two calls. A not-settleable `Update` succeeds and consumes
  its rate limit.
- A pillar revoked before settlement has its whole pending amount burned.
- `CollectReward` pays from Settlement's balance; the ZNN total supply is unchanged
  by a collection.
- A forced accounting fault (settling more than collected) is blocked by the guard
  with the epoch left at the head of the queue.
- `PAUSE_RELAY` on any domain blocks settlement and collection; unpausing resumes
  in epoch order with amounts unchanged.
