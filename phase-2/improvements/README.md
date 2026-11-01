# Phase 2 — Improvement Drafts

General improvements to the Phase 1 design, from feedback and from the work to bring a
second domain. One draft ticket per idea. A draft states
the problem, the proposed design and its trade-offs. `../SPEC.md` and `../../phase-1/SPEC.md`
govern on any conflict; a draft becomes normative only once its SPEC edits land.

The improvements come before the Phase 2 tickets, so that a second domain can build against
the changed surface while the validity-proof work proceeds. The Phase 2 tickets cover the
validity-proof work itself.

File names: `IMP-NN-<short-slug>.md`, numbered in order of creation.

| Ticket | Title | Status |
|---|---|---|
| IMP-01 | Uncapped custody for user-issued tokens | Draft |
| IMP-02 | Drop the on-chain DA commitment and `inputSource` | Under review, leaning to keep both |
| IMP-03 | Round-robin proposer with an open fallback | Draft |

## Candidates not yet drafted

In the order they decide the design:

1. Per-domain execution limits in `profileConfig`
2. Runtime-neutral chunked input replacing the deploy triple
3. `genesisStateRoot` in `RegisterDomain`
4. SPEC §9 identity derivations scoped to the WASM runtime
5. `InputContext` additions: `MomentumInputOrdinal`, `MomentumInputCount`, `UsedPlasma`
6. Developer-fee policy per runtime
