# Zeno

Zeno is the off-chain execution layer for the Zenon Network. Pillar-run executors run smart contracts off-chain and settle the results to Zenon L1 through the Settlement embedded contract. L1 orders the inputs and anchors the results; it never executes a contract.

**Consensus orders. Executors compute. Settlement anchors.**

The end state is a ZK rollup: every batch of execution carries a validity proof, Settlement verifies the proof before it accepts the batch, and acceptance is finality. Settlement is runtime-agnostic. Adding a runtime (an EVM, a Bitcoin verifier) adds a prover program and a verifying key, not Settlement code.

## Status

| Phase | Network | How a batch is trusted | Status |
|---|---|---|---|
| 1 — attested executor | Devnet today; public testnet in preparation. Never mainnet | One administrator-registered pillar executor attests to each batch; Settlement enforces aggregate conservation and a withdrawal delay | Complete. Feedback and improvements only |
| 2 — validity proofs | Mainnet | Settlement verifies a ZK validity proof for every batch; acceptance is finality | Planning |

Phase 1 has no challenge window and no watcher role by design. Both would be removed when validity proofs land, so Phase 1 stays on test networks and Phase 2 is the mainnet release. There is no optimistic phase.

## How it works

1. A user sends an ordinary L1 account block to Settlement: a contract call, a deposit, a deploy chunk or a storage purchase. Settlement accepts it and assigns it the next input index for its domain.
2. The executor reads accepted inputs in L1 order and runs the domain's state transition function (STF) off-chain. It has no sequencing authority: anyone can replay the same inputs and reach the same state.
3. The executor submits a batch commitment with `SubmitBatch`: the state roots before and after, the receipt, event and outbox roots, and the asset flows. In Phase 2 the commitment carries a validity proof over the same batch.
4. Settlement checks the batch and records it. Withdrawals a contract emitted are released from Settlement's custody once the batch is final, and never beyond what was deposited.
5. Clients read L2 state through the gateway and verify every value against the batch root Settlement holds, so a gateway cannot lie about state.

## Documents

| Document | Covers |
|---|---|
| [architecture.md](architecture.md) | The two layers, every component, the transaction flow, domains, trust model |
| [settlement.md](settlement.md) | The Settlement embedded contract: inputs, batch acceptance, custody, withdrawals, fees, administration |
| [executor.md](executor.md) | The executor: STF worker, prover, input pipeline, batches, recovery, gateway |
| [demos.md](demos.md) | What runs today on a local devnet, and what Phase 2 adds |
| [clients/](clients/README.md) | The Settlement SDK, the client SDK, the contract SDK and the console |
| [phase-1/](phase-1/README.md) | Phase 1 as built: the normative spec, the executor spec, the changelog, the activation runbook and the delivered tickets |
| [phase-2/](phase-2/README.md) | Phase 2: aim, decisions, design direction, research items and open questions |
| [wasm-vm/](wasm-vm/README.md) | The WASM runtime: profile, gas table, how to build a contract, conformance vectors |
| [artifacts/](artifacts/README.md) | Runtime-agnostic conformance material: SMT vectors, proof format, serialization vectors, and the registry of every cross-repo copy |
| [research/](research/README.md) | Research by topic: proving, bridges, other chains and runtimes, the L1 state root |

The root documents describe Zeno in its finished form and are informative. `phase-1/SPEC.md` and `phase-1/EXECUTOR.md` are normative for what runs today.

## Repositories

| Repository | Holds |
|---|---|
| [go-zenon](https://github.com/digitalSloth/go-zenon) (`feature/settlement`) | The node: the Settlement embedded contract, the `common/trie` sparse Merkle tree, the L1 state root |
| [zeno-executor](https://github.com/Zeno-L2/zeno-executor) | The executor, the WASM domain worker, the gateway and `zeno-exec` |
| [zeno-contract-sdk-rs](https://github.com/Zeno-L2/zeno-contract-sdk-rs) | The Rust contract SDK and `cargo zeno` |
| [znn-typescript-sdk](https://github.com/digitalSloth/znn-typescript-sdk) (`feature/settlement`) | The TypeScript SDK for go-zenon, with Settlement and state-root support |
| [zeno-client-ts](https://github.com/Zeno-L2/zeno-client-ts) | The TypeScript client for everything Zeno defines above go-zenon (npm `zeno-client`) |
| [zeno-console](https://github.com/Zeno-L2/zeno-console) | The web app: explore domains, deploy, call and manage contracts, administer Settlement |
| [zeno-devnet](https://github.com/Zeno-L2/zeno-devnet) | One command brings up a local devnet with nodes, executor, gateway and console |
| [zeno-docs](https://github.com/Zeno-L2/zeno-docs) | This repository |
