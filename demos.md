# Demos

**Status:** Overview — informative.
**Sources:** [zeno-devnet](https://github.com/Zeno-L2/zeno-devnet) for the local devnet and its flows; [phase-1/runbooks/ACTIVATION.md](phase-1/runbooks/ACTIVATION.md) for activation on a shared network.

Zeno's demo is the Phase 1 system end to end: a user deploys a WASM contract, the executor runs it off-chain, Settlement anchors the result on L1, and a withdrawal settles back to an L1 account. Everything below runs today on a local devnet. Phase 2 changes how a batch is trusted, not what the demo does.

## 1. Run it locally

[zeno-devnet](https://github.com/Zeno-L2/zeno-devnet) brings up the whole stack with one command: go-zenon's devnet nodes and explorer, the executor and its gateway, and the console. The bring-up also registers the WASM domain and the executor, funds the devnet accounts and sets the fee prices, so the devnet is ready to use when the command returns.

```sh
scripts/devnet.sh up       # build, start, configure; about 20 minutes cold
scripts/devnet.sh status   # one ok/FAIL line per check
scripts/devnet.sh down     # stop; the chain survives
scripts/devnet.sh wipe     # remove everything
```

It pins the go-zenon, zeno-executor, zeno-console and zeno-contract-sdk-rs commits it runs, and can run your own checkouts instead. Docker with Compose and Node are the only host requirements.

| Service | Endpoint |
|---|---|
| Node WebSocket | `ws://127.0.0.1:35998` |
| Gateway | `http://127.0.0.1:35999` |
| Console | `http://localhost:5173` |
| Explorer | `http://localhost:36000` |

To sign from the console, install the Zenon web wallet extension, point it at the devnet (chain id 69, the node above) and import the public devnet mnemonic. Reading needs no wallet.

## 2. The reference vault

The demo contract is the reference vault from [zeno-contract-sdk-rs](https://github.com/Zeno-L2/zeno-contract-sdk-rs) (`examples/vault`). It credits deposits in full, keeps a balance per (caller, asset) in its own state, and lets a caller withdraw to an L1 user address. It is production-quality example code for any contract that holds custody, and the contract the devnet flows deploy. Deploy it from the console's Deploy page, or let a flow deploy it.

Build your own contract with the contract SDK: [wasm-vm/README.md](wasm-vm/README.md).

## 3. What the demo shows

Ten scripted flows run against the devnet (`npm run flow:<name>`, or `npm run flows` for all ten). Each prints one `ok` line per check and verifies every receipt against the batch Settlement accepted.

| Flow | Shows |
|---|---|
| `deploy-quota` | Chunked deployment through L1 account blocks, `BuyStorage`, and a deposit refused one byte short of the quota |
| `deposit`, `call-refund`, `withdrawal` | The round trip: a deposit credited on L2, a payable `CallL2` that keeps part and refunds the rest, and an L2→L1 withdrawal proved against the outbox root, relayed and released |
| `force-inclusion` | Settlement accepts inputs while the executor is stopped; the executor then runs them in canonical order. The executor cannot censor |
| `worker-crash`, `crash-recovery` | The supervisor restarts a killed domain worker; a killed executor recovers an accepted, unfinalized batch |
| `executor-replacement` | A second executor syncs from L1 alone, takes over the domain and finalizes a deposit; the first takes back over and rebuilds the second's batches |
| `pause`, `fees` | The administrator pauses, and what each pause blocks; deploy and storage fees accrue, settle per epoch (30% burned, 70% credited) and are collected by the pillar |

After every step a conservation check requires `totalReleased + pendingWithdrawalReserve ≤ totalDeposited` for the domain's ZNN. The executor-replacement flow is the one to watch: two independent executors reach the same roots from L1-anchored data, which is what makes the Phase 1 trust claim checkable.

## 4. In the console

The console shows the same demo without scripts: the dashboard and the domain's batches, inputs, events and outbox; a contract's verified state, calls and activity; the Deploy page with its chunked send and fee review; Withdrawals with relay and release; and the Admin area for the configured administrator. Every L2 value carries a verification badge: Verified (proofs checked against the Settlement batch root), Pending or Inconsistent. See [clients/console.md](clients/console.md).

## 5. On a shared network

[phase-1/runbooks/ACTIVATION.md](phase-1/runbooks/ACTIVATION.md) is the ordered procedure for a shared network: release and sporks, bring-up, parameters, executor and gateway, vault deployment, smoke flows, acceptance checks and the trust-model disclosure. Phase 1 runs on devnet today, and a public testnet is in preparation; it must not be activated on mainnet.

## 6. What Phase 2 changes

The user-visible flows stay the same. What changes is the trust behind them.

- `SubmitBatch` carries a validity proof, and Settlement verifies it. A batch is final when accepted, so a withdrawal is releasable at once rather than after a delay.
- The executor-replacement flow becomes the normal case: any registered pillar executor may submit, in rotation, and anyone with a valid proof may submit after the fallback window.
- The demo runs on mainnet.

Bridge and oracle demos (a Bitcoin SPV `MESSAGING` domain, a `MINTED` BTC bridge) were planned on the Phase 1 trust model and are not scheduled. Their design notes are in [research/bridges.md](research/bridges.md).
