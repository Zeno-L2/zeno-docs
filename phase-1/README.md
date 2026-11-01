# Phase 1 — Attested single executor

**Status:** Complete. Runs on devnet today; a public testnet is in preparation. Never mainnet.

Phase 1 is Zeno as it runs today: one administrator-registered pillar executor runs the WASM domain off-chain and attests to each batch; Settlement records the commitment, enforces aggregate conservation per (domain, asset) and releases withdrawals after a delay. There is no challenge window and no watcher role by design: both would be removed when validity proofs land in [Phase 2](../phase-2/README.md), which is the mainnet release. Phase 1 exists for feedback and improvements on test networks.

| File | What it is |
|---|---|
| [SPEC.md](SPEC.md) | The on-chain protocol: trust model, canonical input stream, Settlement, domain model, runtime profile, host ABI, gas, deployment, state, commitments, finality, executor model, administration, wire formats, conformance. Normative; governs on any conflict |
| [EXECUTOR.md](EXECUTOR.md) | The executor binary: roles, layered runtime, domain plugin interface, execution loop, recovery, storage, configuration, gateway. Normative for zeno-executor |
| [ARCHITECTURE.md](ARCHITECTURE.md) | The design companion: rationale, component architecture, end-to-end flows and the go-zenon integration map. Informative |
| [CHANGELOG.md](CHANGELOG.md) | Revision history of the document set. The documents themselves are forward-reading |
| [runbooks/ACTIVATION.md](runbooks/ACTIVATION.md) | The ordered activation procedure for a shared network, with the devnet run record |
| [tickets/](tickets/README.md) | The delivered implementation tickets: `epic-1-l1-settlement/` (go-zenon, E1-NN) and `epic-2-executor/` (zeno-executor, E2-NN), with the status table |

Conformance material the spec cites lives in [../artifacts/](../artifacts/README.md) (runtime-agnostic) and [../wasm-vm/](../wasm-vm/README.md) (WASM domain). Items left open at the close of Phase 1 are carried in [../phase-2/SPEC.md](../phase-2/SPEC.md) §11.
