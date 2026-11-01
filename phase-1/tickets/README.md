# Phase 1 — Implementation Epics

Phase-based implementation plan for the Phase 1 off-chain WASM execution layer
(single-executor attestation, testnet only), derived from `../SPEC.md` (v1.17.10,
normative), `../EXECUTOR.md` (v0.12.3, executor binary), and `../ARCHITECTURE.md` (design
companion). Those documents govern on any conflict — these tickets are the
delivery plan, not a re-specification.

> **Scope note (SPEC v1.6.0).** The spec now carries the generalized bridge
> framework (domain classes, execution/foreign-fact profiles, custody modes,
> `chainBinding`) as first-class schema, but **all of it ships reserved** — Phase 1
> implements only the single active point `(EXECUTION, L1_NATIVE, ATTESTATION,
> NONE, ESCROW, SINGLE)`. These tickets are therefore unchanged in scope; the only
> impact is that the `DomainRecord` storage encoding (E1-01) and `RegisterDomain`
> validation (E1-03) must carry the new fields and **clamp them to their Phase-1
> defaults** (schema-shaped from day one, so later activation is a spork, not a
> storage migration). `MESSAGING`/`BRIDGE` domains, `MINTED` custody, the profile
> ladders, and the `MintFlowSummary` reconciliation are all out of Phase 1.

## Two epics

| Epic | Folder | Scope |
|---|---|---|
| **Epic 1 — L1 Settlement** | `epic-1-l1-settlement/` | The spork-gated Settlement embedded contract inside `go-zenon`: domain registry, custody/conservation, pillar executor registration and fee settlement, batch acceptance, deployment state machine, relay/withdrawal, admin/pause. Tickets **E1-01–E1-12**. |
| **Epic 2 — Executor** | `epic-2-executor/` | The standalone off-chain executor binary (separate repo / Go module that pins `go-zenon`): generic runtime, SMT store, serializer, WASM plugin, input pipeline, batch builder, recovery, gateway, two-phase reconciliation, reference vault contract. Tickets **E2-01–E2-29**. |

## Delivery status

Updated when a ticket merges. This table is the status of record; ticket files
themselves are left unedited once delivered, as a historical record.
`EXECUTOR.md` §18's prerequisite table carries the cross-repo view of what each
side is still waiting on.

| Ticket | Epic | Phase | Status |
|---|---|---|---|
| E2-01 Repo scaffold + domain interface | 2 | 1A | Delivered |
| E2-02 SMT state store + applier | 2 | 1A | Delivered |
| E2-03 Canonical serialization library | 2 | 1A | Delivered |
| E2-04 Deterministic WASM runtime plugin | 2 | 1A | Delivered |
| E2-05 WASM conformance suite + determinism harness | 2 | 1A | Delivered |
| E2-06 Input pipeline + ordering | 2 | 1A | Delivered |
| E2-07 STF driver + batch builder | 2 | 1A | Delivered |
| E2-08 Persistence, recovery, failure | 2 | 1A | Delivered |
| E2-09 SPEC v1.14.0 conformance | 2 | 1A | Delivered |
| E2-10 Stale pending-record hazard | 2 | 1A | Delivered |
| E2-11 Unpriced-work benchmark + limits | 2 | 1A | Delivered |
| E2-12 State store scaling | 2 | 1A | Delivered |
| E2-13 Supervisor + per-domain worker processes | 2 | 1A | Delivered |
| E2-14 Storage quota, `BuyStorage`, version fields | 2 | 1A | Delivered |
| E2-15 Deploy records keyed by deployer | 2 | 1A | Delivered |
| E1-01 Settlement scaffold, spork, address, storage layout | 1 | 1B | Delivered |
| E1-02 Core/Periphery, admin, pause | 1 | 1B | Delivered |
| E1-03 Domain registry + lifecycle | 1 | 1B | Delivered |
| E1-04 Executor registration + fee settlement | 1 | 1B | Delivered |
| E1-05 Deposits, custody, conservation | 1 | 1B | Delivered |
| E1-06 Chunked deployment state machine | 1 | 1B | Delivered |
| E1-07 `SubmitBatch` acceptance + finality | 1 | 1B | Delivered |
| E1-08 `RelayMessage` + withdrawal release | 1 | 1B | Delivered |
| E1-09 `CallL2` / upgrade input surface | 1 | 1B | Delivered |
| E1-10 Settlement client surface | 1 | 1B | Delivered |
| E1-11 Settlement RPC API | 1 | 1B | Delivered |
| E1-12 Settlement reconciliation | 1 | 1C | Delivered |
| E2-16 L1 read path | 2 | 1C | Delivered |
| E2-17 Input path | 2 | 1C | Delivered |
| E2-18 Conformance vectors + batch roots | 2 | 1C | Delivered |
| E2-19 Submission | 2 | 1C | Delivered |
| E2-20 Revert data | 2 | 1C | Delivered |
| E2-21 Withdrawals | 2 | 1C | Delivered |
| E2-22 Gateway: read API + bundle serving | 2 | 1C | Delivered |
| E2-23 Two-phase reconciliation | 2 | 1C | Delivered |
| E2-24 Reference vault contract | 2 | 1C | Delivered |
| E2-25 Local-variable gas pricing | 2 | 1C | Delivered |
| E2-26 Conformance coverage | 2 | 1C | Delivered |
| E2-27 E2E on devnet + activation runbook | 2 | 1C | Delivered |
| E2-28 Findings and cleanup | 2 | 1C | Delivered |
| E2-29 Reference-hardware gas calibration | 2 | 1C | Open: waits on reference hardware |

## Numbering and implementation order

Each epic numbers its own tickets, and each repository's history carries one
squashed commit per delivered ticket. Executor tickets are `E2-NN` and map to
zeno-executor commits on `dev`. Settlement tickets are `E1-NN` and map to go-zenon
commits on `feature/settlement`, which builds on `feature/merkle-state-root`. A
ticket number is its landing order within its repository.

```
Phase 1A — Executor core, no chain changes        (Epic 2)   E2-01 → E2-15
Phase 1B — Spork-gated Settlement contract         (Epic 1)   E1-01 → E1-11
Phase 1C — Wiring, gateway, reconciliation, vault, gas pricing, conformance, devnet E2E   (Epic 2)   E2-16 → E2-29
                                                                                          (Epic 1)   E1-12
```

Within 1C the order is E2-23 (two-phase reconciliation, with E1-12), E2-24 (reference vault),
E2-25 (local-variable gas pricing, after the vault), E2-26 (conformance coverage), E2-27 (E2E on
a local devnet and the activation runbook), E2-28 (findings and cleanup) and E2-29
(reference-hardware gas calibration, before the activation spork).

Rationale (SPEC §29): the executor runtime and conformance suite (1A) have **no
on-chain dependency** and retire the highest-risk item (deterministic
instrumentation) first. The Settlement contract (1B) ships dark on testnet. The
wiring (1C) connects the two and activates the spork.

The phases are the intended order, not a hard gate. A ticket whose **Depends on**
list is satisfied may land early, and tickets E1-01–E1-03 carry no Epic 2 dependency.
The gates that bind are the per-ticket **Depends on** lists and these cross-epic
edges:

| Ticket | Waits on |
|---|---|
| Epic 1 Settlement (E1-01–E1-09) | E2-03 (shared canonical encodings), E2-04 (the STF the deploy and upgrade paths assume) |
| E2-12 (go-zenon half) | go-zenon `common/trie` `CompactTree` prover |
| E1-10 Settlement client surface | E1-07, E1-08 |
| E1-11 Settlement RPC API | E1-01–E1-10 |
| E2-16 L1 read path | E2-08, E2-13; E1-10 for the exported key builders |
| E2-17 Input path | E2-06–E2-08, E2-16; E1-10 |
| E2-18 Conformance vectors + batch roots | E2-17 |
| E2-19 Submission | E2-16, E2-17, E2-18 |
| E2-20 Revert data | E2-04, E2-18 |
| E2-21 Withdrawals | E2-16, E2-19; E1-10 |
| E2-22 Gateway | E2-16, E2-19, E2-20, E2-21 |
| E2-23 Two-phase reconciliation | E2-22; part B also waits on E1-12 |
| E1-12 Settlement reconciliation | none; E2-23 part B waits on it |
| E2-24 Reference vault contract | E2-04; E2-27 consumes it |
| E2-25 Local-variable gas pricing | E2-11; lands after E2-24 |
| E2-26 Conformance coverage | E2-25, E2-18 |
| E2-27 E2E on devnet + activation runbook | E2-01–E2-26, all of Epic 1 (including E1-12), and zeno-console |
| E2-28 Findings and cleanup | E2-27 |
| E2-29 Reference-hardware gas calibration | E2-28, E2-11, E2-25; lands before the activation spork |

## Ticket format (lean)

Each ticket carries: **Goal**, **SPEC refs**, **Depends on**, **Key work**,
**Code references**, **Acceptance criteria**. Every ticket ends with a tested,
verified, independently-mergeable increment.

## Status legend for code references

- **[EXISTS]** — present in current `go-zenon` source, build against it.
- **[NEW]** — created by this plan.
- **[ARTIFACT]** — a companion file already in `../../artifacts/` or `../../wasm-vm/` (gas table,
  proof-format, SMT/execution vectors).
