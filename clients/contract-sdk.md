# The contract SDK

**Repository:** `Zeno-L2/zeno-contract-sdk-rs` (Rust, MIT). **Tool:** `cargo zeno`.

The contract SDK lets a developer, or an agent, write a Zeno WASM contract in Rust. Every module it builds passes Settlement's deployment validation unchanged, and the build fails before a deployment would.

**Normative source:** [SPEC](../phase-1/SPEC.md) governs every byte a contract emits; the SDK governs only how those bytes are produced.

## Why it exists

A Zeno contract is a `no_std` WASM module with one export, `run(ptr, len) -> i64`, and three imports: `env.storage_len`, `env.storage_read` and `env.abort`. A call performs no writes. The module returns its writes, events and withdrawals as one encoded effects blob, and the runtime applies them when the call succeeds. A malformed blob ends in `RUNTIME_FAULT` with no abort code, so the SDK owns the encoding.

The SDK adds no consensus surface. It produces the module a user deploys and the interface file that describes it. Deploying and calling a contract happen through [the console](console.md) and [the client SDK](client-sdk.md), which encodes calls against that file.

## What it does

| Part | Content |
|---|---|
| `crates/zeno-contract` | The `no_std` runtime: the frame reader, the codec, `Ctx`, `Item<V>` and `Map<K, V>`, the effects encoder and the panic handler |
| `crates/zeno-contract-macros` | `#[contract]` on an inline module, with `#[method]`, `#[on_deposit]`, `#[contract_error]`, `#[event]` and `#[storage(0xNN)]` |
| `crates/zeno-testing` | `TestDomain`, a harness that deploys, calls, deposits, upgrades and buys storage through `zeno-exec run` and returns a `Receipt` |
| `crates/cargo-zeno` | `cargo zeno new`, `build`, `test` and `explain` |
| `examples/` | `counter`, the smallest contract, and `vault`, the reference vault |
| `templates/contract` | The `cargo zeno new` scaffold |

**The model.** `Ctx` is the only handle a method receives: the call context, the attached `deposit()`, `claim_deposit(n)`, `emit(event)` and `withdraw(to, asset, n)`. A read sees the state from before the call plus the call's own earlier writes. Value arrives as a deposit and leaves as a withdrawal; there are no protocol balances and no transfers. There is no view entry point, so every method call is an input, and clients read state through the gateway's `storage` view.

**Dispatch and errors.** The payload is a 4-byte selector, `SHA3-256(method_name)[0:4]`, followed by the arguments in the canonical codec. An empty payload routes to `#[on_deposit]`. Contract error codes are positive and carry the variant's fields as revert data; SDK abort codes are negative.

**The interface file.** `cargo zeno build` writes `artifacts/<name>.interface.json` beside `<name>.wasm`: `interface_version: 1`, `encoding: "zeno-selector-v1"`, the module's `code_hash`, and its methods, deposit handler, events, errors and storage layout. The client SDK encodes calls and decodes state, events and return data from this file. An agent reads it to learn a deployed contract without its source.

**Written for agents.** `AGENTS.md` and `llms.txt` give the model, the deployability rules and the commands in a form an LLM can work from alone. Every error has a stable `Zxxxx` code and a one-line fix, `cargo zeno explain <code>` prints the long form, and `--json` writes machine-readable results.

**The reference vault.** `examples/vault` is the testnet demo contract for deposits and withdrawals and the production-quality pattern for custody-holding contracts: DEXes, escrows and bridge pools. `ctx.withdraw` is unbounded, so the contract's own ledger, debited with `checked_sub` before the withdrawal, is the only guard. Only `vault.*` is ever deployed; `vault-testhook.*` exists for tests. It has no mainnet deployment.

## What it does not do

- It has no Rust client. No crate deploys, calls or administers a contract.
- It is not a second runtime. `cargo zeno test` runs contracts against zeno-executor through `zeno-exec`.
- It allows no synchronous cross-contract calls, floats, `alloc`, or any import beyond the three.
- A plain host `cargo build` does not produce a deployable module. Only `cargo zeno build` sets the target, link flags and strip step.

## Where to read more

In the `zeno-contract-sdk-rs` repository:

- `AGENTS.md` and `llms.txt`: the model, rules and commands for agents.
- `docs/guide.md`: writing a contract from the scaffold, step by step.
- `docs/api.md`: attributes, `Ctx`, storage, types, the interface file and limits.
- `docs/testing.md`: the `zeno-testing` harness and `cargo zeno test`.
- `docs/errors.md`: the cause and fix of every `Zxxxx` code.
- `examples/vault`: the reference vault and its committed artifacts.
