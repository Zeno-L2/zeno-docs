# WASM VM

The WASM runtime is Zeno's first execution domain. A contract is a WebAssembly module that the executor runs off-chain with deterministic gas metering; Settlement sees only the batch commitments. This folder holds the material specific to that runtime: how to build a contract, the gas table and the execution conformance vectors. Runtime-agnostic artifacts (the SMT vectors, the proof format, the serialization vectors) live in [../artifacts/](../artifacts/README.md), whose README is also the registry of every cross-repo copy, including the ones in this folder.

**Normative source:** [../phase-1/SPEC.md](../phase-1/SPEC.md) §7 (runtime profile), §8 (host ABI), §9 (execution context), §10 (gas), §11 (deployment), §13 (state), §14 (effects).

## 1. The runtime

| Property | Value |
|---|---|
| Profile | WebAssembly Core 1.0 MVP, integer subset: no floats, SIMD, threads, sign-extension or saturating truncation |
| Host imports | Exactly three: `storage_len`, `storage_read`, `abort`. A contract reads state through the host and returns its writes, events and withdrawals as one effects blob |
| Limits | Code 256 KiB before instrumentation, memory 4 MiB, 512 stack frames; the structural caps are in SPEC §25 |
| Gas | A static per-opcode table, `metering_version = 1`, applied by instrumentation at deploy; every call carries a `plasma_limit` |
| Deployment | Chunked through L1 account blocks, content-addressed by the module's SHA3-256; the executor validates, instruments and derives the contract id |
| State | A sparse Merkle tree per domain; a contract's keys live under its contract id, and storage is a prepaid quota bought with `BuyStorage` |

The executor's WASM worker runs this profile on wazero in Phase 1. In Phase 2 the same profile runs in the Rust STF crate, natively and inside the zkVM guest; the gas table is re-derived from proving cost (Phase 2 research item R2).

## 2. Building a contract

Contracts are written in Rust with [zeno-contract-sdk-rs](https://github.com/Zeno-L2/zeno-contract-sdk-rs). The SDK produces `no_std` modules that pass deployment validation unchanged, and `cargo zeno` builds, validates and tests them against the real runtime through `zeno-exec`, with no node or chain.

```sh
cargo install --git https://github.com/Zeno-L2/zeno-contract-sdk-rs --tag <tag> --locked cargo-zeno
cargo zeno new demo        # scaffold a contract crate
cd demo
cargo zeno build           # build, strip and validate the module into artifacts/
cargo zeno test            # run the crate's tests against zeno-exec
cargo zeno explain Z0211   # the cause and fix of an error code
```

A contract is one `#[contract]` module: `#[storage]` items and maps, `#[contract_error]` codes, `#[event]` structs and `#[method]` functions that take a `Ctx`. Deposits arrive as value attached to a call; the contract keeps its own ledger and sends value out with `ctx.withdraw`. `cargo zeno build` also writes the contract's interface file, which the [client SDK](../clients/client-sdk.md) and the [console](../clients/console.md) use to encode calls and read state.

Read, in order: the SDK's [`AGENTS.md`](https://github.com/Zeno-L2/zeno-contract-sdk-rs/blob/main/AGENTS.md) (the model and the rules that make a module deployable), [`docs/guide.md`](https://github.com/Zeno-L2/zeno-contract-sdk-rs/blob/main/docs/guide.md) (a contract from the scaffold, step by step), [`docs/api.md`](https://github.com/Zeno-L2/zeno-contract-sdk-rs/blob/main/docs/api.md), [`docs/testing.md`](https://github.com/Zeno-L2/zeno-contract-sdk-rs/blob/main/docs/testing.md) and [`docs/errors.md`](https://github.com/Zeno-L2/zeno-contract-sdk-rs/blob/main/docs/errors.md). The SDK is written so agents and LLMs can write contracts with it; `llms.txt` at its root is the entry point for them. The reference vault (`examples/vault`) is the production-quality example for any contract that holds custody. See [../clients/contract-sdk.md](../clients/contract-sdk.md) for what the SDK is and is not.

Deploy a built module from the console's Deploy page, or with the client SDK's `prepareDeploy`.

## 3. Artifacts

| File | What it is | Status |
|---|---|---|
| `wasm-gas-table-v1.json` | The opcode gas table, `metering_version = 1` (SPEC §10) | Normative; copy of zeno-executor's embedded table |
| `wasm-gas-table-v1.md` | Prose companion of the gas table | Normative |
| `execution-conformance-v1.json` | Per-input execution vectors: frame, receipt, gas, state diff, events, outbox (SPEC §31) | Normative; copy of zeno-executor's suite |
| `execution-conformance-v1-README.md` | Provenance, regeneration and coverage of the execution suite | Normative; copy of zeno-executor's README |

An implementation of the runtime is conformant when it reproduces every vector byte for byte. The sources of record, the regeneration commands and the checksum checks are in [../artifacts/README.md](../artifacts/README.md).
