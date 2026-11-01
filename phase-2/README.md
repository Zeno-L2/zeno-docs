# Phase 2 — Validity proofs on mainnet

**Status:** Planning.

Phase 2 makes Zeno a ZK rollup: every batch carries a validity proof, Settlement verifies it at `SubmitBatch`, and acceptance is finality. No executor, committee or watcher is trusted for correctness, and Settlement never executes a runtime. Phase 2 is the mainnet release; Phase 1 stays on test networks.

| File | What it is |
|---|---|
| [SPEC.md](SPEC.md) | The outline: aim, roadmap, decisions taken, design direction, research items with the question each answers, open questions, the prove-on-challenge fallback, risks, sequencing, and the items carried over from Phase 1 |
| [improvements/](improvements/README.md) | One draft ticket per improvement to the Phase 1 design, from feedback and from the second-domain work; these come before the Phase 2 tickets |

A full specification in the shape of [../phase-1/SPEC.md](../phase-1/SPEC.md) follows once the research items and the improvement drafts are decided; implementation tickets follow the specification. The research the outline draws on lives in [../research/proving/](../research/proving/README.md).
