# The client SDK

**Repository:** `Zeno-L2/zeno-client-ts`. **Package:** npm `zeno-client`, ESM for Node and browsers, with `znn-typescript-sdk` as a peer dependency.

The client SDK is the TypeScript client for everything Zeno defines on top of Settlement: the L2 wire formats, the identifiers a runtime derives, the gateway's `zeno_*` API and contract interfaces. It reads, decodes and verifies. It holds no keys and sends nothing.

**Normative source:** [SPEC](../phase-1/SPEC.md) for the wire formats, [EXECUTOR](../phase-1/EXECUTOR.md) for the gateway API.

## Why it exists

Settlement stores roots, not structures. The receipt, the execution result, the state diff, the event, the outbox message and the canonical input are defined by the executor and the runtime, and go-zenon never decodes them. The same holds for `chain_id`, `caller` and `ContractID`, and for the gateway that serves L2 state. A client needs one library that knows these formats and checks every gateway answer against the roots Settlement accepted.

A helper lives here when only the executor, the gateway or a runtime defines it. A helper that go-zenon implements or enforces lives in [the Settlement SDK](settlement-sdk.md), which this package imports rather than wraps.

## What it does

| Module | Exports |
|---|---|
| `codec.ts` | `ZenoReader`, `ZenoWriter`, `ZenoCodecError`: the canonical codec. The reader rejects trailing bytes, as the Go decoder does |
| `derive.ts` | `chainId`, `l1Caller`, `contractId`, `smtKey`, `encodeMetadata`, `decodeMetadata`. All hashing is SHA3-256, not Keccak-256 |
| `decode.ts` | `decodeReceipt`, `decodeExecutionResult`, `decodeOutboxMessage` (both kinds), `decodeBundle`, the deploy and upgrade `ReturnData` decoders, `decodeRevertData` |
| `input.ts` | `encodeInput`, `inputHash`, `inputRoot`, `merkleRoot` |
| `constants.ts` | `InputKind`, `ReceiptStatus` and the codec bounds |
| `gateway/` | `GatewayClient`, its HTTP and WebSocket transports, typed errors, `verifyProofs` and the `contracts/1` profile helpers |
| `interface/` | The interface file types, `parseInterface`, the `zeno-selector-v1` encoder, `prepareDeploy` and `prepareCall` |
| `runtime/` | `registerInterfaceEncoder`, `registerPathDerivation` and the WASM path derivation |

**The gateway client.** `GatewayClient.connect(url)` opens an HTTP client for an `http(s)://` URL and a WebSocket, which also carries subscriptions, for a `ws(s)://` URL. One client serves every domain. Its methods wrap the `zeno_*` API one to one: `chainInfo`, `getHead`, `getBatch`, `getReceipt`, `getStateAt`, `getProof`, `getInputs`, `getEvents`, `getOutbox`, `getOutboxProof`, `getBundle`, `getReceiptSidecar`, `read`, `simulate` and `subscribe`. Every state or batch read takes a tag: `pending` (built by the executor, not yet on L1), `latest` (accepted by Settlement), `finalized` or a `batchId`. Every result carries `batchId` and `verified`.

**Profiles.** `read()` is the only domain-specific call. `contracts(domainId)` returns a `ContractsProfile` for a domain that implements `contracts/1`, with its feature flags and typed `contract`, `storage`, `contractsByDeployer` and `module` views. Code written against a profile works for every runtime that implements it.

**Verification.** The client checks each proof against the root the response names, then checks that root against the Settlement batch entry read through the Settlement SDK. For a state-backed view it re-derives each path with the runtime's path derivation and rebuilds the result from the proven values. For a module it checks `SHA3-256(bytes) = codeHash`. A failed check throws `GatewayVerificationError`, naming the method and the check. The client never upgrades `verified`.

**Runtime modules.** Two pieces stay runtime-specific: the interface encoder, keyed by the interface file's `encoding`, and the path derivation, keyed by `runtimeKind`. A new runtime adds one of each and nothing else.

## What it does not do

- It holds no keys and sends nothing. `prepareDeploy` and `prepareCall` return bytes for the Settlement SDK's `planDeploy` and `callL2` builders.
- It never re-exports `znn-typescript-sdk`. Settlement builders, the read RPC, the deploy planner, the state verifier and `SettlementStorage` are imported from that package directly.
- It does not implement `SubmitBatch`, the executor or the WASM runtime.
- It contains no wallet or UI code. [The console](console.md) consumes it.

## Where to read more

In the `zeno-client-ts` repository: `README.md` for install and commands, and `AGENTS.md` for the layout, the boundary rule and the fixture workflow.

In these docs: [EXECUTOR](../phase-1/EXECUTOR.md) for the gateway API the client wraps, and [the contract SDK](contract-sdk.md) for the interface file the encoder reads.
