# Clients

The client side of Zeno is everything a user, a contract author or an operator runs outside the node and the executor. Four pieces cover it: two TypeScript libraries, one Rust SDK and one web app. None of them holds a key.

**Normative source:** [SPEC](../phase-1/SPEC.md) for Settlement, [EXECUTOR](../phase-1/EXECUTOR.md) for the gateway API.

```
console ─> client SDK ─> Settlement SDK ─> znnd
   │
   └─> gateway

contract SDK ─> module + interface file (encoded by the client SDK)
```

The console reaches a znnd node through the two SDKs and a gateway through the client SDK. The contract SDK sits beside them: it produces the WASM module a user deploys and the interface file the client SDK encodes calls against.

| Client | Repository | Owns | Doc |
|---|---|---|---|
| The Settlement SDK | `digitalSloth/znn-typescript-sdk`, branch `feature/settlement` | Everything go-zenon implements: the Merkle state root and the Settlement contract | [settlement-sdk.md](settlement-sdk.md) |
| The client SDK | `Zeno-L2/zeno-client-ts` (npm `zeno-client`) | Everything the executor, the gateway and the runtimes define: L2 wire formats, derived identifiers, the gateway client, contract interfaces | [client-sdk.md](client-sdk.md) |
| The contract SDK | `Zeno-L2/zeno-contract-sdk-rs` (Rust, `cargo zeno`) | Writing, building and testing a WASM contract, and its interface file | [contract-sdk.md](contract-sdk.md) |
| The console | `Zeno-L2/zeno-console` | Reading every domain with verification; contracts, withdrawals and the administrator area through a browser wallet | [console.md](console.md) |

## The boundary rule

A helper belongs in the Settlement SDK when go-zenon's Go code implements or enforces it. It belongs in the client SDK when only the executor, the gateway or a runtime defines it.

The test is mechanical: search go-zenon for the rule. `AssetIdFromZTS` is in `vm/embedded/definition/settlement.go`, so asset ids live in the Settlement SDK. `chain_id`, `caller` and `ContractID` appear nowhere in go-zenon, so they live in the client SDK.

The client SDK never re-exports the Settlement SDK. An app imports Settlement builders, the state verifier and `SettlementStorage` from `znn-typescript-sdk` directly.

## Shared rules

- Neither SDK holds keys. Both build unsigned `AccountBlockTemplate`s or read data. Signing stays with the caller: a key pair in Node, or the Zenon web wallet's `window.zenon` provider in a browser.
- A verified value names the root it was verified against. Neither SDK upgrades an unverified value to a verified one.
- A new runtime changes the client SDK's interface encoder and path derivation, and nothing else on this page. Settlement's contract inputs carry opaque bytes and check no `runtimeKind`, so the builders and the console's send path serve every runtime unchanged.
