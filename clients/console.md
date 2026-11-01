# The console

**Repository:** `Zeno-L2/zeno-console`. One Vue 3 web app built on `nom-ui`, `znn-typescript-sdk` and `zeno-client`.

The console is the web app for Zeno. Anyone reads domains, batches, contracts and receipts without a wallet. With the Zenon web wallet connected, a user deploys, upgrades, calls and manages WASM contracts, relays and releases withdrawals, and the network administrator reaches the administrator methods.

**Normative source:** [SPEC](../phase-1/SPEC.md).

## Why it exists

Zeno needs no wallet of its own. Every L2 action is an L1 send to Settlement, so signing an account block is the only capability a user supplies, and the Zenon web wallet already does that. The console builds each send as an unsigned `AccountBlockTemplate` with the Settlement SDK and asks the wallet to sign and publish it through the injected `window.zenon` provider. It holds no keys.

Reading needs no wallet. The console reads L1 state from a znnd node through [the Settlement SDK](settlement-sdk.md) and L2 state from a gateway through [the client SDK](client-sdk.md), and checks every L2 value against the roots Settlement accepted.

## What it does

| Page | Content |
|---|---|
| Dashboard | L1 height, domain count, inputs received, paused domains, and a card per domain |
| Network | Pause state, domains, prices, pending time challenges, executors and fees, custody |
| Domain | Overview, Batches, Inputs, Events, Outbox and Contracts for one domain |
| Contracts | A verified overview, Call, State, Events, Activity, Manage (with a wallet) and the interface file |
| Deploy | Load a module, configure, review the plan and fees, send each chunk through the wallet, resume after a reload, upgrade |
| Withdrawals | A domain's outbox withdrawals, relay with a proof check against the batch's `outboxRoot`, and release |
| Admin | Pause and resume, and every time-challenged administrator change, for the configured administrator |
| Send tracker | One send from Published through Received, Executed and Submitted to Finalized |
| Playground | One raw `zeno_*` request to the gateway, unverified |

**Verification badges.** Every L2 value carries one of three badges. **Verified**: read at `latest` or `finalized`, with every proof checked against the Settlement batch root. **Pending**: read at `pending`, built by the executor and not yet accepted on L1. **Inconsistent**: the domain status is `INCONSISTENT` or an SDK check failed; the console shows a banner and the last verified data. In Phase 1 a Verified value rests on a batch Settlement accepted under `ATTESTATION`; in Phase 2 it rests on the validity proof Settlement verified at `SubmitBatch`.

**Typed calls and state.** With a contract's interface file, the Call tab renders a form per method, the State tab lists the declared storage entries, and events and return data decode to names and fields. A `readonly` method runs through `simulate` with no send and no fee.

**Test before deploy.** The Deploy page runs a module against zeno-executor's local runner, `zeno-exec serve`, on the user's own machine. Nothing is sent or signed.

**Runtime independence.** The console never branches on `runtimeKind`. Contract features are gated on the `contracts/1` profile and its feature flags, and decoding follows the interface file's `encoding`. A new runtime reaches the console as a dependency bump.

## What it does not do

- It holds no keys and stores no secrets. The wallet adapter loads only when the user clicks Connect. Local storage holds settings, interface files, deploy journals and pending administrator proposals.
- It never signs. Every write goes through `window.zenon`, one send at a time.
- It never upgrades a value to Verified. The badge comes from the gateway result and the client SDK's checks.
- It does not submit batches or run contracts. The executor relays its own withdrawals; the console's Relay and Release cover a stalled executor.

## What it needs

A znnd node with the Settlement spork active, a gateway that lists the console's origin in `allowedOrigins`, and, for writes, the Zenon web wallet on the same network. Only a gateway beside the executor serves `pending`; elsewhere the console falls back to `latest`.

## Where to read more

In the `zeno-console` repository: `README.md` for the pages and how to run it, `AGENTS.md` for the layout and rules, and `docs/faq.md` for reading data, fees, contracts, wallet setup and local devnet values.

In these docs: [EXECUTOR](../phase-1/EXECUTOR.md) for the gateway the console reads from, and [the contract SDK](contract-sdk.md) for the reference vault and counter it deploys.
