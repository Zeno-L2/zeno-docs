# Ticket IMP-03 — Round-robin proposer with an open fallback

**Status:** Draft · **Repos:** zeno (SPEC), go-zenon (Settlement), zeno-executor
**Related:** `../SPEC.md` §4.5 S2, Q-10; `../../phase-1/SPEC.md` §6.2, §19, §22

## Problem
Phase 1 runs one executor under `proposerPolicy = SINGLE` (SPEC §6.2). Ordinals `1`
and `2` are unassigned, and the multi-member proposer rule is left to Phase 2. Phase 2
needs a rule that:

- gives each batch slot one entitled proposer, so executors do not race to prove
  the same batch;
- keeps a domain live when the entitled proposer is offline;
- needs no missed-turn records, lapse or activation grace.

## Proposal

### 1. Primary proposer
`proposerPolicy = ROUND_ROBIN`. For domain `d` at height epoch
`E = height / MomentumsPerEpoch`:

```
eligible = ACTIVE members not marked timed-out, ordered by storage key ascending
primary(d, E) = eligible[(E + d) mod len(eligible)]
```

- The storage key is `hash(pillarName)` (`executorSetMemberPrefix`), so the order is
  fixed by L1 state and needs no extra record.
- The `+ d` offset gives each domain a different primary in the same epoch. Every
  member serves every domain (§6.2), so the offset spreads the proving work.
- The current height decides `E`, not the batch's inputs. A batch submitted in `E`
  belongs to `E`'s primary.

### 2. Open fallback
When the domain's oldest unbatched input has waited `T` momentums, any ACTIVE member
may submit. The first valid batch at the cursor position wins, as today (§19).

- Settlement keeps one field per domain, `backlogSinceHeight`. The input-admission
  helper (§4.6) sets it to the receive height when it admits an input to an empty
  backlog. `SubmitBatch` resets it to the acceptance height when inputs remain
  unbatched, and clears it when none remain.
- The fallback opens when `height ≥ backlogSinceHeight + T`.
- Measuring from the oldest unbatched input, not from the last accepted batch, keeps
  an idle domain's next input from opening the fallback immediately.

### 3. Timed-out primaries are skipped
When Settlement accepts a fallback batch, it marks the primary that was entitled at
that height as timed-out. That is one flag per member.

- A marked member is left out of `eligible`, so it no longer costs every domain `T`
  momentums of latency on each of its turns.
- Any batch the member submits, as primary or in the fallback, clears the mark. A
  recovered pillar rejoins the rotation without an administrator.
- The mark is network-wide, not per domain. A pillar that is down is down for every
  domain.

### 4. Proofs bind to the submitter
The proof's public outputs carry the submitter's executor address (`../SPEC.md` §4.4
item 2). `SubmitBatch` checks it equals the sender.

Without this, an observer can copy a proof from an unconfirmed account-block, submit
it from its own address and take the batch's fees.

### Unchanged
- Fee attribution to the submitting pillar (§22.3, §22.4).
- One accepted batch per cursor position; state lineage stays linear (§6.2, §19).
- `SINGLE` stays for a size-1 set.

## Trade-offs
- An offline primary delays its domain by up to `T` momentums once, until its first
  timeout marks it.
- A primary that loses its turn to a slow prover is marked timed-out. `T` must exceed
  worst-case proving time (Q-10), or healthy primaries are skipped.
- A fallback race wastes proving work for every member that loses it. The race
  happens only after a timeout.
- A set change mid-epoch changes `len(eligible)`, and with it the primary. This is
  deterministic, but a member can lose a turn it was already proving.

## Open questions
1. Is a mid-epoch set change acceptable, or should `eligible` be snapshotted at the
   epoch's first batch? A snapshot costs one record per domain per epoch.
2. Which ordinal does `ROUND_ROBIN` take? Ordinals `1` and `2` are unassigned. If
   they were ever assigned to an earlier policy, use `3`.
3. Does a timed-out mark expire on its own, or clear only on a submission? Clearing
   only on submission is simpler. A dead pillar then stays out until it is revoked.
4. `T` value (Q-10): set from measured proving latency (R2).

## SPEC edits
- `../../phase-1/SPEC.md` §6.2 (the policy), §4.6 (`backlogSinceHeight` in the
  admission helper), §19 (`SubmitBatch` entitlement check), §22.1 (timed-out flag),
  §27 (`proposerPolicy` enum).
- `../SPEC.md` §4.4 item 2 (submitter address in the public outputs).
- `../../phase-1/EXECUTOR.md` §13 (proposer schedule).

## Code touch points
- **go-zenon** `vm/embedded/definition/settlement.go`: the `ProposerPolicy`
  constants; `ExecutorSetMember` gains the timed-out flag; the domain record gains
  `backlogSinceHeight`.
- **go-zenon** `vm/embedded/implementation/settlement.go`: the `RegisterDomain`
  policy check (`:192`); the entitlement check in `SettlementSubmitBatchMethod`; the
  input-admission helper.
- **zeno-executor**: computing the primary and the fallback height from L1 state, and
  starting to prove only when this executor is entitled.
