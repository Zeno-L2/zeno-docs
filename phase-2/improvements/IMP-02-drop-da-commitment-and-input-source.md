# Ticket IMP-02 — Drop the on-chain DA commitment and `inputSource`

**Status:** Under review, leaning to keep both mechanisms · **Repos:** zeno (SPEC), go-zenon (Settlement), zeno-executor
**Related:** `../SPEC.md` §4.5 S4

> **Review note.** The bundle and `inputSource` machinery is built and costs little to keep. Keeping it reserves a data-availability path for future domains and for inputs that do not originate on L1, where L1 order and availability do not apply. The draft below stands as the case for removal.

## Problem
Every input of every domain is an L1 account-block sent to Settlement (SPEC §4.1,
§6.1). L1 therefore already provides canonical order, input data availability and
completeness. Two mechanisms restate that guarantee and carry no load of their own:

1. **The DA commitment.** Each batch commits `DAHash` and `DAMode`; each domain has
   an approved `DAMode` set (§5.2, §19, §20). Only `DAMode = 0` exists, and
   Settlement never checks availability.
2. **`inputSource`.** Every domain declares `L1_NATIVE` or `L1_RELAYED` (§6.1). Both
   values give the same order, DA and censorship resistance, and Settlement and
   the executor treat them the same.

## Why neither is needed
- **State is reconstructable from L1 alone.** Inputs, including deploy chunks, are
  account-blocks. Replaying them from the domain's genesis root rebuilds every
  state, with no bundle.
- **No on-chain party consumes the bundle.** The DA commitment serves a challenger
  re-executing one disputed batch from its pre-state. Under `ZK_VALIDITY`,
  Settlement verifies a proof instead, so the witnesses are needed only off-chain.
- **No input source bypasses L1.** The only off-chain source (code `2`) is retired
  (§27). The DA commitment and `inputSource` both exist to describe that source.
- **`L1_RELAYED`'s remaining meaning lives elsewhere:**

| `L1_RELAYED` meaning | Carried by |
|---|---|
| The STF validates foreign data (headers, confirmation depth) | `runtimeKind` / `stfSpecHash` |
| Trust disclosure: the domain depends on a foreign chain | `foreignProfile` |
| Relaying is permissionless | A rule on every domain with `foreignProfile ≠ NONE` |

## Proposal
1. Remove `DAHash`, `DAMode` and `damode_version` from the batch commitment and the
   stored batch entry (§19, §24).
2. Remove the approved-`DAMode` set, `SetApprovedDAModes` and its time challenge
   (§5.2, §5.3, §23), and `MaxDAChunkSize` (§25).
3. Remove `inputSource` from the domain record, `RegisterDomain`, the
   `stfSpecHash` fold (§24) and the §27 enum. §6.1 becomes one normative rule:
   every input is an L1 account-block sent to Settlement, ordered by L1.
4. Move the permissionless-relay requirement to §6.4, applied to every domain
   with `foreignProfile ≠ NONE`.

The execution bundle stays as an off-chain object: the prover's witness input
and the data the gateway serves. It stops being a consensus object.

## Trade-offs
- The gateway loses its single content address for a bundle. A client verifies a
  bundle piecewise: inputs against `inputRoot`, results against `receiptRoot`,
  and each witness against `preStateRoot` (open question 1).
- Reintroducing an off-chain input source later needs a new commitment field and
  a spork. §6.1 forbids such a source today, so this costs nothing until that rule
  changes.

## Open questions
1. Is piecewise verification enough for gateway clients, or does the commitment
   need a cheap anchor over the bundle (for example, folding a bundle hash into
   `receiptRoot`)? A hash folded into an existing root keeps a content address
   without a separate field.
2. Is `domainClass` redundant against `foreignProfile` by the same argument?
   `BRIDGE` versus `EXECUTION` mostly restates whether a foreign profile is set.
   Assess separately before folding it in.
3. Is a `DomainRecord` already stored on testnet? If yes, removing `inputSource`
   needs a storage migration at the spork; if no, the encoding changes in place.

## SPEC edits
§4.1, §5.2, §5.3, §5.4, §6, §6.1, §6.4, §19, §20, §23, §24, §25, §27 (commitment
wire format, `inputSource` and `DAMode` enums), §29 (disclosure text).

## Code touch points
- **go-zenon Settlement:** commitment decode and `SubmitBatch` checks; stored batch
  entry; `SetApprovedDAModes` and its storage prefix; `inputSource` in
  `DomainRecord` and `RegisterDomain` validation.
- **zeno-executor:**
  - `internal/codec/commitment.go`, `internal/codec/constants.go` — commitment
    fields and `DAModeVersion`.
  - `internal/batch/builder.go`, `internal/batch/driver.go` — `DAHash` computation
    moves off the commitment path; the bundle is still built for the gateway and
    the prover.
  - `internal/gateway/api.go` — `findDAHash` lookup and the `DAHash` match at
    `api.go:308`; re-key bundles by `(domainId, batchId)` and verify per open
    question 1.
  - `internal/settlement/anchor.go`, `internal/recovery/*` — stored commitment
    fields.
  - `domain/domain.go` — the `Source` doc comment names `inputSource`; key it on
    `foreignProfile`.
