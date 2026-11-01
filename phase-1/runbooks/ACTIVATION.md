# Phase 1 activation runbook

**Status: draft.** Every devnet step was run on the local devnet of zeno-devnet. The runs are recorded in "Devnet run record" at the end.

Testnet activation is a later ticket. This runbook is its input, and it is not part of Ticket E2-27.

> **Phase 1 activates a single point on a general trust surface: attestation by one administrator-registered pillar, with no external inputs and no bond. It is not trustless execution, and it runs on testnet only.** This is normative context for the entire specification and **MUST** be communicated to users and integrators without qualification or obfuscation.

## Conventions

The runbook is one ordered list of steps with ids `<stage>.<n>`, for example `C.2`, in stages A to H. The list order is the testnet order. On the devnet the steps run in list order, except that a `run by` step happens inside the earlier step it names.

Each step has five fields:

- **Call or command**: what is sent or run.
- **Expected**: the result when the step works.
- **Check**: the read that proves it.
- **Rollback or pause**: the step id that undoes it, or `none`.
- **Devnet**: one of the four markers below.

The Devnet markers:

- `run`: the step's own command or call ran on the devnet as written.
- `run by <step id>`: on the devnet, the command of the named earlier step performs this step, so it has no command of its own there. The Check still runs.
- `devnet only`: a public network skips the step (wipe, the scripted flows, the vault-testhook deploy, stopping the executor).
- `not run on devnet`: a public-network step, written out but not run.

A rollback step carries its own marker. It is `run` only when a flow or the console run exercises it, and otherwise `not run on devnet`.

The devnet commands run from the zeno-devnet checkout. `URL` below is the node's HTTP RPC: `http://127.0.0.1:35997` on the devnet, the testnet node's URL on testnet.

The devnet publishes every port on `127.0.0.1` only (go-zenon `docker-compose.yml:14,63-64,108-109,126`, zeno-devnet `docker-compose.yml:61,96`), so only the Docker host reaches it. `DEVNET_BIND_ADDRESS=0.0.0.0` binds go-zenon's ports to all interfaces instead (go-zenon `docker/devnet/README.md:139-147`). The devnet keys are public, so a port open on a shared network lets any host on it publish to the devnet.

## A. Release and sporks

### A.1 Release the sporks

- **Call or command**: the spork release flow of go-zenon `common/types/spork.go:9-26`: placeholder id, governance `CreateSpork`, the real id in `spork.go`, a coordinated binary release, `ActivateSpork`, enforcement. `StateRootSpork` enforces before `OffchainExecutionSpork` (SPEC §5.1).
- **Expected**: both sporks are active and enforced, in that order.
- **Check**: `embedded.spork.getAll` with params `[0, 20]` lists each spork with `activated` and `enforcementHeight`. Compare the two enforcement heights.
- **Rollback or pause**: A.4.
- **Devnet**: `not run on devnet`

### A.2 Confirm the network administrator address

- **Call or command**: run `go version -m <znnd>` on the released binary, then read `types.NetworkAdmin` in the source at the release commit (go-zenon `common/types/address.go:52`). The source constant cannot show the binary's build tags. A `devnet` build (`common/types/address_devnet.go:9`) replaces it with devnet index 1, whose key derives from the public devnet mnemonic, and cuts `MinAdministratorDelay` to 12 and `WithdrawalDelayMin` to 30 (`vm/constants/embedded_devnet.go:14-16`). The address has no on-chain setter, so SPEC §25.2 requires this check before activation.
- **Expected**: the address is the intended Phase 1 administrator, and the binary is not a `devnet` build.
- **Check**: the `go version -m` output shows no `-tags` build setting that contains `devnet`, and `vcs.revision` equals the release commit. The source constant at that commit is the intended address. Both are recorded in the run record. After B.6, `getDomainInfo` for domain 1 reads back the registration: B.6 is sent from this address and succeeds only if the binary's `NetworkAdmin` equals it (go-zenon `vm/embedded/implementation/settlement.go:234`). No deposit is accepted before that read.
- **Rollback or pause**: none. Cut a new binary release if the address is wrong.
- **Devnet**: `not run on devnet`

### A.3 Devnet genesis sporks

- **Call or command**: none. The devnet genesis activates `az`, `htlc`, `dynamic-plasma`, `merkle-state-root` and `offchain-execution` at height 0 and `libp2p` at height 10, and leaves `bridge-liq` inactive (go-zenon `docker/devnet/genesis.json:106-157`). `merkle-state-root` and `offchain-execution` both enforce at height 0 (go-zenon `docker/devnet/README.md:73`), so the devnet does not exercise the SPEC §5.1 ordering.
- **Expected**: the devnet nodes accept Settlement blocks from the first momentum.
- **Check**: B.2 finishes and `scripts/devnet.sh status` prints only `ok` lines.
- **Rollback or pause**: none.
- **Devnet**: `run`

### A.4 Roll back or pause the sporks

- **Call or command**: before `ActivateSpork`, do not activate. After enforcement, pause both scopes globally, then set domain 1 `PAUSED` (B.11) to stop new inputs.
- **Expected**: the global pause halts batch submission and withdrawal relay, release, fee settlement and `CollectReward` (go-zenon `vm/embedded/implementation/settlement.go:805,943,1529-1534,1794,1889`). It does not stop deposits or calls: Settlement keeps admitting them into escrow until the domain status is `PAUSED` (`getInputDomain`, `settlement.go:1119-1127`; SPEC §23). Until B.11, escrowed funds can be neither batched nor withdrawn.
- **Check**: `embedded.settlement.getPauseState` with params `[]` shows `global` with `submitPaused: true` and `relayPaused: true`. After B.11, `getDomainInfo` shows `status` `1`, which is `PAUSED` (go-zenon `vm/embedded/definition/settlement.go:406-408`). Before the human approves each Pause send, the wallet popup's signer equals the A.2 address. The console reports a send signed by any other account as "Published, refused", and it reports a Pause as sent only once `getPauseState` shows it (zeno-console `src/core/admin-service.ts:94-106,160-174`). The pause holds only once the read shows it.
- **Rollback or pause**: none.
- **Devnet**: `not run on devnet`

## B. Bring-up

### B.1 Wipe the devnet

- **Call or command**: `scripts/devnet.sh wipe`
- **Expected**: the containers, volumes and console state are gone.
- **Check**: `docker ps -a --filter label=com.docker.compose.project=zeno-devnet` lists nothing.
- **Rollback or pause**: none.
- **Devnet**: `devnet only`

### B.2 Bring the devnet up

- **Call or command**: `scripts/devnet.sh up`. It prints the commit of go-zenon, zeno-executor, zeno-console, znn-typescript-sdk and zeno-client-ts, builds the images, starts the nodes and the explorer, starts the console from the zeno-console repo root, sends B.3 to B.10, starts the executor and the gateway, and runs `status`. Its `preflight` refuses a zeno-console install older than its package files, and `up` restarts a running console whose zeno-console commit or installed packages changed.
- **Expected**: the command exits 0 after about 20 minutes on a cold machine. Every published port binds to `127.0.0.1`.
- **Check**: `scripts/devnet.sh status` prints only `ok` lines, `ok configuration` and `ok conservation` included. `docker ps --filter label=com.docker.compose.project=zeno-devnet` shows every published port on `127.0.0.1`.
- **Rollback or pause**: B.1, then B.2 again.
- **Devnet**: `run`

### B.3 Fuse QSR for the administrator

- **Call or command**: `plasma.fuse(admin, 10,000 QSR)` from the administrator (`src/cli.ts:58-62`). No delay.
- **Expected**: the administrator's account has plasma.
- **Check**: `status` prints `ok configuration`, which reports a missing `fuse:admin` otherwise.
- **Rollback or pause**: none.
- **Devnet**: `run by B.2`

### B.4 Fuse QSR for the executor

- **Call or command**: `plasma.fuse(executor, 10,000 QSR)` from the administrator (`src/cli.ts:58-62`). No delay.
- **Expected**: the executor's account has plasma.
- **Check**: `status` prints `ok configuration`.
- **Rollback or pause**: none.
- **Devnet**: `run by B.2`

### B.5 Fuse QSR for the user

- **Call or command**: `plasma.fuse(user, 10,000 QSR)` from the user (`src/cli.ts:58-62`). No delay. Testnet amounts are set in the testnet ticket.
- **Expected**: the user's account has plasma.
- **Check**: `status` prints `ok configuration`.
- **Rollback or pause**: none.
- **Devnet**: `run by B.2`

### B.6 Register domain 1

- **Call or command**: `RegisterDomain` for domain 1 with the fields of `src/cli.ts:67-83`, which are the same on both networks except `stfSpecHash` (section C notes). The first registration has no delay.
- **Expected**: domain 1 is `ACTIVE` with the delays of C.1 and C.2.
- **Check**: `embedded.settlement.getDomainInfo` as in C.1. On testnet the send comes from the A.2 address, so the read also confirms the administrator gate.
- **Rollback or pause**: B.11.
- **Devnet**: `run by B.2`

### B.7 Register the executor

- **Call or command**: `RegisterExecutor("dev1", <executor address>)` from the administrator (`src/cli.ts:90`). It takes effect after `AdministratorDelay`, which is 12 momentums on the devnet build and 17,280 on a non-devnet build.
- **Expected**: the executor set lists the executor as `ACTIVE`.
- **Check**: `status` prints `ok configuration`.
- **Rollback or pause**: first set `PAUSE_SUBMIT` and `PAUSE_RELAY` for domain 1. Both apply at once (`settlement.go:619-655`, no time challenge) and stop the wrong executor's batches and withdrawals. Then register a replacement executor. A replacement takes effect after `MinAdministratorDelay` (17,280 momentums on a non-devnet build, twice the 8,640 withdrawal delay), so without the pause a wrongly registered or compromised executor's withdrawals finalize and release first. `getPauseState` shows both scopes `true` for domain 1.
- **Devnet**: `run by B.2`

### B.8 Approve the DA mode

- **Call or command**: `SetApprovedDAModes(1, [BEST_EFFORT])` from the administrator (`src/cli.ts:91`). It takes effect after `SoftDelay`, which is 6 momentums on the devnet build and 8,640 on a non-devnet build.
- **Expected**: domain 1 approves `BEST_EFFORT` only.
- **Check**: C.6.
- **Rollback or pause**: C.9.
- **Devnet**: `run by B.2`

### B.9 Set the ZNN asset limit

- **Call or command**: `SetAssetConfig(1, <ZNN asset id>, MaxBatchWithdrawal)` from the administrator (`src/cli.ts:92`). It takes effect after `SoftDelay`.
- **Expected**: ZNN is custodyable on domain 1 with the C.3 limit.
- **Check**: C.3.
- **Rollback or pause**: C.10.
- **Devnet**: `run by B.2`

### B.10 Set the byte prices

- **Call or command**: `SetPeripheryConfig(znnPerByteOfBytecode, znnPerByteOfState)` from the administrator (`src/cli.ts:93`). It takes effect after `SoftDelay`.
- **Expected**: both prices are the C.4 and C.5 values.
- **Check**: C.4 and C.5.
- **Rollback or pause**: C.9.
- **Devnet**: `run by B.2`

### B.11 Roll back or pause the bring-up

- **Call or command**: set domain 1 `PAUSED` through `RegisterDomain` status. It applies at once, outside any time challenge (`settlement.go:310-312`).
- **Expected**: domain 1 accepts no new inputs. Batches and withdrawals continue while the domain is `PAUSED`: a `PAUSED` domain still accepts `SubmitBatch` and registers withdrawals (go-zenon `settlement_test.go:1506,2539`). `PAUSE_SUBMIT` and `PAUSE_RELAY` (C.10) are the only controls that stop them.
- **Check**: `getDomainInfo` shows `status` `1`, which is `PAUSED`.
- **Rollback or pause**: none.
- **Devnet**: `not run on devnet`

On the devnet the rollback of the whole stage is B.1, then B.2.

## C. Parameters

Phase 1 never activates on mainnet (SPEC §1), so the "pre-mainnet" values are the values the testnet activation uses. The devnet uses the testnet value unless a flow would then not finish in practical time. The testnet keeps today's placeholder ceilings, because SPEC §25.2 and go-zenon `vm/constants/settlement.go:38-40` finalize them before mainnet, which Phase 1 never reaches. Every value below is inside its bound on both builds.

Line numbers in the Core bounds column are go-zenon `vm/constants/settlement.go` unless named otherwise. The Check column reads `embedded.settlement` methods over JSON-RPC, so the same check works on testnet with another `URL`:

```
curl -s -H 'Content-Type: application/json' -d '{"jsonrpc":"2.0","id":1,"method":"<method>","params":<params>}' URL
```

| Parameter | Setter | Core bounds | Testnet value | Devnet value | Reason | Check |
|---|---|---|---|---|---|---|
| Withdrawal delay | `RegisterDomain` initial delays (`SetDomainConfig` later) | `WithdrawalDelayMin` 8,640 (30 on the devnet build, `embedded_devnet.go:16`) to `WithdrawalDelayMax` 120,960 (placeholder), `:44,46,86-88` | 8,640 momentums (1 day) | 30 (devnet `WithdrawalDelayMin`) | The Core floor `WithdrawalDelayMin` on a non-devnet build. In Phase 1 it is the administrator's window to see a bad batch and set `PAUSE_RELAY` before its withdrawals release (SPEC §1, §21). At 8,640 every flow's finality wait takes a day, so the devnet must differ. | `getDomainInfo` (`rpc/api/embedded/settlement.go:201`), params `[1]`, field `withdrawalDelay` |
| Runtime-upgrade delay | `RegisterDomain` initial delays (`SetDomainConfig` later) | At least the domain's active withdrawal delay, at most `WithdrawalDelayMax` 120,960 (`IsRuntimeUpgradeDelayValid`, `:113-115`) | 17,280 momentums (2 days) | 17,280 | At least the withdrawal delay (Core floor), and equal to `MinAdministratorDelay`, the delay of the other custody-consequential change (`RegisterExecutor`, SPEC §23). No flow waits on it, so the devnet uses it as-is. | `getDomainInfo`, params `[1]`, field `runtimeUpgradeDelay` |
| `MaxBatchWithdrawal` (domain 1, ZNN) | `SetAssetConfig` | `MaxBatchWithdrawalFloor` 1 to `MaxBatchWithdrawalCeiling` 100,000,000,000,000 (1,000,000 ZNN, placeholder), `:54,56,92-97` | 100,000,000,000 (1,000 ZNN) | same | Bounds what one dishonest batch can release (SPEC §1). Core refuses any later decrease (SPEC §23), so it starts at the smallest value above honest per-batch use and only rises. | `getAssetInfo` (`:410`), params `[1, "0000000000000000000000000000000000000000000014e66318c6318c6318c6"]`, field `maxBatchWithdrawal` |
| `znnPerByteOfBytecode` | `SetPeripheryConfig` | `ZnnPerByteOfBytecodeFloor` 1 to `ZnnPerByteOfBytecodeCeiling` 100,000,000 (placeholder), `:76,78,126-131` | 1,000 base units per byte | same | Testnet ZNN has no market value, so the price only deters spam: a 256 KiB deploy costs 2.62 ZNN. The measured-cost derivation of SPEC §32 is a mainnet requirement. | `getPeripheryConfig` (`:347`), params `[]`, field `znnPerByteOfBytecode` |
| `znnPerByteOfState` | `SetPeripheryConfig` | `ZnnPerByteOfStateFloor` 1 to `ZnnPerByteOfStateCeiling` 100,000,000 (placeholder), `:79,81,135-140` | 1,000 base units per byte | same | Same reason. One 96-byte balance leaf costs 0.00096 ZNN. | `getPeripheryConfig`, params `[]`, field `znnPerByteOfState` |
| Approved DA modes (domain 1) | `SetApprovedDAModes` | none: a set of DA mode values, not a bounded number | `[BEST_EFFORT]` | same | The Phase-1 DA mode the executor serves. | `getDomainInfo`, params `[1]`, field `approvedDAModes` |
| `MaxInputGas` | none: build-time constant (go-zenon `:67`, zeno-executor `plugins/wasm/limits.go:19`) | below 2^63 − 1 (SPEC §25.1, `:62-63`); it is itself the ceiling `IsInputGasInBounds` applies to a `plasma_limit` (`:120-122`) | 1,000,000,000 (placeholder) | same | Ticket E2-29 sizes it on reference hardware. The laptop numbers put a worst-case batch (`64 × MaxInputGas`) at about 43 minutes (E2-29 "The problem"). | no RPC read: the source constants at the commits in the run record (go-zenon `vm/constants/settlement.go:67`, zeno-executor `plugins/wasm/limits.go:19`) |
| Network administrator address | none: build-time `types.NetworkAdmin` | none | the address in the released binary (go-zenon `common/types/address.go:52`), confirmed in A.2 | devnet index 1, `z1qq6eg8n43g032hanpsfp02qcdmv7zfj3y2lt5d` (`common/types/address_devnet.go:9`) | SPEC §25.2 requires confirming the released binary's address before activation, since it has no on-chain setter. | no RPC read: on testnet the `go version -m` result and the source constant of A.2 (go-zenon `address.go:52`), on the devnet `address_devnet.go:9` |

Also recorded here:

- `MaxBatchMint` (SPEC §25.2, reserved): not set, because ESCROW is the only custody mode and nothing reads it (`:69-73`).
- `MinAdministratorDelay` / `MinSoftDelay`: 17,280 / 8,640 on a non-devnet build; 12 / 6 on the devnet build (`vm/constants/embedded_devnet.go:14-15`), so a two-step admin call takes about 2 minutes.
- `FeeEpochMomentums` / `UpdateMinNumMomentums`: 8,640 / 300 on a non-devnet build; 60 / 30 on the devnet build, so `flow:fees` settles an epoch in about 15 minutes.
- Domain 1 registration fields: `src/cli.ts:67-83`, identical on both networks except `stfSpecHash`. The testnet `stfSpecHash` is the SHA3-256 hash of zeno-docs `phase-1/SPEC.md`, the STF spec document, at the zeno-docs release commit: `git show <release commit>:phase-1/SPEC.md | openssl dgst -sha3-256`. The release computes it with that rule and records the commit and the value in the testnet run record. The devnet registers the placeholder `00…01` (zeno-devnet `src/devnet.ts:21`), which is devnet only and never registered on testnet.
- Fused QSR: 10,000 QSR each for admin, executor and user on the devnet (`src/devnet.ts:28`); testnet amounts are not run on devnet.

### C.1 Withdrawal delay

- **Call or command**: the `withdrawalDelay` of B.6.
- **Expected**: 30 on the devnet, 8,640 on testnet.
- **Check**: the table's `getDomainInfo` read, field `withdrawalDelay`.
- **Rollback or pause**: C.9.
- **Devnet**: `run by B.6`

### C.2 Runtime-upgrade delay

- **Call or command**: the `runtimeUpgradeDelay` of B.6.
- **Expected**: 17,280 on both networks.
- **Check**: the table's `getDomainInfo` read, field `runtimeUpgradeDelay`. `status` also prints `ok configuration`, which reports a conflict otherwise.
- **Rollback or pause**: C.9.
- **Devnet**: `run by B.6`

### C.3 `MaxBatchWithdrawal`

- **Call or command**: the `SetAssetConfig` of B.9.
- **Expected**: 100,000,000,000 on both networks.
- **Check**: the table's `getAssetInfo` read, field `maxBatchWithdrawal`.
- **Rollback or pause**: C.10.
- **Devnet**: `run by B.9`

### C.4 `znnPerByteOfBytecode`

- **Call or command**: the `SetPeripheryConfig` of B.10.
- **Expected**: 1,000 on both networks.
- **Check**: the table's `getPeripheryConfig` read, field `znnPerByteOfBytecode`.
- **Rollback or pause**: C.9.
- **Devnet**: `run by B.10`

### C.5 `znnPerByteOfState`

- **Call or command**: the `SetPeripheryConfig` of B.10.
- **Expected**: 1,000 on both networks.
- **Check**: the table's `getPeripheryConfig` read, field `znnPerByteOfState`.
- **Rollback or pause**: C.9.
- **Devnet**: `run by B.10`

### C.6 Approved DA modes

- **Call or command**: the `SetApprovedDAModes` of B.8.
- **Expected**: `[BEST_EFFORT]` on both networks. The read returns the number, so `[0]` means `[BEST_EFFORT]`.
- **Check**: the table's `getDomainInfo` read, field `approvedDAModes`.
- **Rollback or pause**: C.9.
- **Devnet**: `run by B.8`

### C.7 `MaxInputGas`

- **Call or command**: none. It is a build-time constant.
- **Expected**: 1,000,000,000 in both repos.
- **Check**: read go-zenon `vm/constants/settlement.go:67` and zeno-executor `plugins/wasm/limits.go:19` at the commits in the run record.
- **Rollback or pause**: none. Cut a new release.
- **Devnet**: `run`

### C.8 Network administrator address

- **Call or command**: none. It is a build-time constant (A.2).
- **Expected**: the devnet build carries index 1, `z1qq6eg8n43g032hanpsfp02qcdmv7zfj3y2lt5d`. The testnet build carries the address confirmed in A.2 and is not a `devnet` build.
- **Check**: on the devnet, read go-zenon `common/types/address_devnet.go:9` at the commit in the run record. On testnet, repeat the A.2 `go version -m` check on the binary and read `address.go:52`.
- **Rollback or pause**: none. Cut a new release.
- **Devnet**: `run`

### C.9 Roll back a parameter

- **Call or command**: re-send the setter with the earlier value and wait for its delay.
- **Expected**: the earlier value is back.
- **Check**: the table's read for that parameter.
- **Rollback or pause**: none.
- **Devnet**: `not run on devnet`

### C.10 Roll back `MaxBatchWithdrawal`

- **Call or command**: Core refuses any decrease, so pause with `PAUSE_RELAY` for domain 1.
- **Expected**: no withdrawal relays or releases.
- **Check**: `getPauseState` shows `relayPaused: true` for domain 1. Before the human approves the send, the wallet popup's signer equals the A.2 address. The console reports a send signed by any other account as "Published, refused", and it reports the Pause as sent only once `getPauseState` shows it. The pause holds only once the read shows it.
- **Rollback or pause**: none.
- **Devnet**: `not run on devnet`

## D. Executor and gateway

### D.1 Executor start and catch-up

- **Call or command**: `scripts/devnet.sh:145-155` starts the executor and the gateway and waits for `zeno_getHead` to return `OK`.
- **Expected**: `gateway OK`.
- **Check**: `scripts/devnet.sh status` prints `ok container executor`, `ok container gateway` and `ok gateway`.
- **Rollback or pause**: D.3.
- **Devnet**: `run by B.2`

### D.2 Stand up the public gateway

- **Call or command**: Ticket E2-22, after H.1 is published.
- **Expected**: the public gateway serves domain 1 with status `OK`. It opens only after the H.1 disclosure is public, because E and F take deposits.
- **Check**: `zeno_getHead` returns `OK` from the public endpoint.
- **Rollback or pause**: D.3.
- **Devnet**: `not run on devnet`

### D.3 Replace the executor or recover from a crash

- **Call or command**: the executor-replacement and crash-recovery flows of F.9 (`npm run flow:executor-replacement`, `npm run flow:crash-recovery`).
- **Expected**: both flows end with `PASS`.
- **Check**: the flow output, and `scripts/devnet.sh status` afterwards.
- **Rollback or pause**: none.
- **Devnet**: `run`

## E. Vault deployment

Preparation, done once by the human before E.1:

1. In nom-webwallet `feature/wallet-connect` at the commit in the run record: `npm install`, `npm run build:extension`, load `dist-extension/` unpacked in Chrome. No `.env` is needed, because the console uses the injected provider, not WalletConnect.
2. Devnet only: in the wallet, import the devnet mnemonic (`src/devnet.ts:33-34`) with any password, add accounts until index 1 and 8 show, and set chain id 69, network id 69, node `ws://localhost:35998`. The devnet mnemonic and every key in zeno-devnet are public. They must never hold testnet funds. On testnet the operator uses the operator's own accounts and the testnet chain id and node.
3. Devnet only for the values: in the console at `http://localhost:5173`, the same network in the network indicator; gateway `http://127.0.0.1:35999` and domain 1 in Settings; the index 1 address as "Administrator address". On testnet the "Administrator address" is the A.2 address.
4. Keep Chrome out of macOS native fullscreen (nom-webwallet `README.md:78`), and expect an unlock prompt after 30 idle minutes.
5. Keep `vault.wasm`, `vault.interface.json`, `vault-testhook.wasm` and `vault-testhook.interface.json` from `zeno-contract-sdk-rs/examples/vault/artifacts/` to hand.

No scripted flow runs during E.1 to F.8, since the wallet and the flows both send from index 8. After the steps named in the run record the tester runs `scripts/devnet.sh status` and records the `conservation` line. That line asserts counter consistency for domain 1 ZNN and the fee counters only. It is not evidence of Settlement's escrow balance.

The `/tx` page shows the stages Published, Received, Executed, Submitted and Finalized (zeno-console `src/components/SendStages.vue:10-17`). This runbook names them by those labels, with the gateway tag in brackets: Executed (`pending`), Submitted (`latest`), Finalized (`finalized`).

The ZNN asset id is `0000000000000000000000000000000000000000000014e66318c6318c6318c6` (zeno-docs `artifacts/serialization-conformance-v1.json:98`). The index 9 address is `z1qqcam4ycu0ta8333hx38r5j2z3ry9jjfxkc7t5` (go-zenon `docker/devnet/README.md:234`).

The record notes per send:

- Input sends (E.2's start, each chunk and finalize, E.3, F.1, F.2's deploy sends and BuyStorage, F.3, F.4): the human opens each send's `/tx` page and records the stage reached, the receipt badge and whether any field shows "(unchecked)". Finalized with a Verified badge and no "(unchecked)" mark is the evidence that the receipt passed the console's check at `finalized`. The tester reads `zeno_getReceipt` for each send and records the stage reached, `pending`, `latest` or `finalized`, with `verified` and the status.
- L1-only sends (F.6 Relay and Release, F.8 Pause and Resume) end at Received on the `/tx` page, which shows no Executed step for them (zeno-console `src/components/SendStages.vue:21`). The record notes Received plus one Settlement read by the tester, named in the step.
- F.6's Relay is root-checked: the Withdrawals view rebuilds the outbox root from the inclusion proof and matches it to Settlement's batch before it sends (zeno-console `src/core/withdrawal-read-service.ts:51-80`).

The `/tx` page checks each receipt with the zeno-client-ts receipt verifier (zeno-client-ts `src/gateway/verify.ts:406`, zeno-console `src/core/tx-tracker-service.ts:45`). At every stage it binds the receipt to the node's copy of the send block. From Submitted on it also checks the receipt against the batch Settlement accepted, and its return data, events and outbox items against the receipt's roots. So the receipt and proof criterion covers console-run inputs as well as the scripted flows.

### E.1 Connect the wallet

- **Call or command**: the human connects the wallet on index 8 in the console.
- **Expected**: the console shows the index 8 address and the devnet network.
- **Check**: `scripts/devnet.sh status` prints only `ok` lines.
- **Rollback or pause**: none.
- **Devnet**: `run`

### E.2 Deploy the vault

- **Call or command**: the human opens `/deploy` and deploys `vault.wasm` as immutable (Start, each chunk, Finalize).
- **Expected**: each send reaches Executed (`pending`), Submitted (`latest`) and Finalized (`finalized`), and its `/tx` page reaches Finalized with the receipt badge Verified. The success link opens `/contracts/1/<id>`, and the console's code hash equals `vault.hashes.json` `code_hash`. The Finalize receipt's `codeHash` equals `vault.hashes.json` `instrumented_code_hash`.
- **Check**: the human opens the `/tx` page of Start, each chunk and Finalize from `/domains/1/inputs` and records Finalized with a Verified badge for each. The tester reads `zeno_getReceipt` for each send (status SUCCESS, `verified: true`) and `status` `conservation`.
- **Rollback or pause**: E.4.
- **Devnet**: `run`

### E.3 Buy storage

- **Call or command**: the human opens the Manage tab of the vault and buys 96 bytes of storage.
- **Expected**: the send reaches Executed (`pending`), Submitted (`latest`) and Finalized (`finalized`), and the `/tx` page reaches Finalized with the receipt badge Verified.
- **Check**: the human waits for Received before F.1, because the purchase is sent before the deposit, and returns to the `/tx` page for Finalized. The tester reads `zeno_getReceipt` at `latest`, then at `finalized`, and `status` `conservation`.
- **Rollback or pause**: none.
- **Devnet**: `run`

### E.4 Pause and replace a defective vault

- **Call or command**: the vault has no pause or owner (zeno-contract-sdk-rs `examples/vault/src/lib.rs`), so the only controls are domain-wide: `PAUSE_SUBMIT`, `PAUSE_RELAY` or domain status `PAUSED` (A.4, B.11). They stop every contract in domain 1, not one vault, and `PAUSE_RELAY` also stops withdrawals from the defective vault. Then deploy the fixed vault as a new contract. The vault is immutable.
- **Expected**: users move to the new vault. An immutable vault leaves users only its own methods. If the defect blocks them, users cannot exit the defective vault, and the new contract starts with its own state and balances.
- **Check**: the new vault's `/contracts/1/<id>` page.
- **Rollback or pause**: none.
- **Devnet**: `not run on devnet`

## F. Smoke flows

### F.1 Deposit

- **Call or command**: the human opens the Manage tab of the E.2 vault and deposits 1 ZNN.
- **Expected**: the send reaches Executed (`pending`), Submitted (`latest`) and Finalized (`finalized`), and the `/tx` page reaches Finalized with the receipt badge Verified.
- **Check**: the tester reads `zeno_getReceipt` at `latest`, then at `finalized`, `getAssetInfo(1, ZNN)` `totalDeposited`, and `status` `conservation`.
- **Rollback or pause**: none.
- **Devnet**: `run`

### F.2 Deploy the test vault

- **Call or command**: the human deploys `vault-testhook.wasm` and buys it 96 bytes of storage. It is the only artifact with a payable method (`claim_partial`).
- **Expected**: each send reaches Executed (`pending`), Submitted (`latest`) and Finalized (`finalized`), and its `/tx` page reaches Finalized with the receipt badge Verified. The Finalize receipt's `codeHash` equals `vault-testhook.hashes.json` `instrumented_code_hash`.
- **Check**: the tester reads `zeno_getReceipt` for each send and `status` `conservation`.
- **Rollback or pause**: none. Never deploy `vault-testhook` on a public network.
- **Devnet**: `devnet only`

### F.3 Payable call with refund

- **Call or command**: the human opens the Call tab of the test vault and calls `claim_partial(50000000)` with a 2 ZNN deposit, so 1.5 ZNN is refunded to the outbox.
- **Expected**: the send reaches Executed (`pending`), Submitted (`latest`) and Finalized (`finalized`), and the `/tx` page reaches Finalized with the receipt badge Verified.
- **Check**: the tester reads `zeno_getReceipt` at `latest`, then the batch's `isFinalized` and the receipt at `finalized`.
- **Rollback or pause**: none.
- **Devnet**: `run`

### F.4 Withdrawal call

- **Call or command**: the human opens the Call tab of the E.2 vault and calls `withdraw(<ZNN asset id>, 50000000, <index 9 address>)`.
- **Expected**: the send reaches Executed (`pending`), Submitted (`latest`) and Finalized (`finalized`), and the `/tx` page reaches Finalized with the receipt badge Verified.
- **Check**: the tester reads `zeno_getReceipt` at `latest`, then the batch's `isFinalized` and the receipt at `finalized`.
- **Rollback or pause**: none.
- **Devnet**: `run`

### F.5 Stop the executor

- **Call or command**: once the receipts of F.3 and F.4 both read `latest`, the tester runs `docker stop $(docker ps -q --filter label=com.docker.compose.project=zeno-devnet --filter label=com.docker.compose.service=executor)`, so the executor cannot relay first. The executor relays and releases on its own (zeno-executor `internal/supervisor/relay.go:44`), so the console's Relay and Release are reachable only with it stopped.
- **Expected**: the executor container is stopped.
- **Check**: `docker stop` exits 0 before either batch's `finalizeAt`, and `getReleaseQueue(1, 0, 10)` is empty. From F.5 to F.7 `status` prints `FAIL container executor`, which is expected.
- **Rollback or pause**: F.7.
- **Devnet**: `devnet only`

### F.6 Relay and release

- **Call or command**: once the batch or batches of F.3 and F.4 show Finalized (30 momentums after Submitted), the human connects index 8, opens `/withdrawals`, turns "Mine only" off, and relays the F.3 refund (150,000,000 to index 8) and the F.4 withdrawal (50,000,000 to index 9). The view lists every withdrawal of the selected domain, and any account may relay any withdrawal. The human then releases once. One Release pays every entry in the queue.
- **Expected**: Relay is root-checked in the console. After the first Relay, `getReleaseQueue(1, 0, 10)` lists the F.3 refund. After the second, it lists both outbox ids. After Release, `getReleaseQueue` is empty, `totalReleased` rose by both amounts and `pendingWithdrawalReserve` is 0 (`rpc/api/embedded/settlement.go:441,410`).
- **Check**: the tester runs `getReleaseQueue(1, 0, 10)` after each Relay, then `getReleaseQueue` and `getAssetInfo` after Release. The `/tx` page of each Relay and Release ends at Received, so the record notes Received plus the read.
- **Rollback or pause**: none.
- **Devnet**: `run`

### F.7 Restart the executor

- **Call or command**: the tester runs `scripts/devnet.sh config`, which starts the executor and waits for the gateway.
- **Expected**: `gateway OK`.
- **Check**: `scripts/devnet.sh status` prints only `ok` lines.
- **Rollback or pause**: none.
- **Devnet**: `devnet only`

### F.8 Pause and resume

- **Call or command**: the human switches the wallet to index 1, opens the Admin view, confirms that the approval popup shows the index 1 address, and approves a `PAUSE_SUBMIT` Pause for domain 1. After the pause reads back, the human resumes it the same way. The approval popup shows and signs with the account that was active when the request arrived (nom-webwallet `src/sw/router.ts:182-199`, `src/core/composables/useDappApprovals.ts:111-157`). Settlement refuses a Pause from any other account with no descendant block, and the console then reports "Published, refused".
- **Expected**: the `/tx` page of each send ends at Received, with no Executed step, because the sends never reach the domain. The Admin view reports each send only once `getPauseState` shows the change. After Pause, `getPauseState` lists domain 1 with `submitPaused: true` and `relayPaused: false`. After Resume, `submitPaused: false` (`rpc/api/embedded/settlement.go:319-345`). `getPauseState` lists domain 1 only after the first pause.
- **Check**: the tester runs `getPauseState` after each send. The wallet popup's signer equals the A.2 address before approval, and the pause holds only once `getPauseState` shows it.
- **Rollback or pause**: F.10.
- **Devnet**: `run`

### F.9 Scripted flows

- **Call or command**: `npm run flows`, the ten flows in order.
- **Expected**: every flow ends with `PASS`. A failed flow is re-run alone once.
- **Check**: the flow output, and `scripts/devnet.sh status` after the last flow.
- **Rollback or pause**: F.10.
- **Devnet**: `devnet only`

### F.10 Resume a paused domain

- **Call or command**: resume `PAUSE_SUBMIT` in the Admin view. `flow:pause` also clears a pause it left.
- **Expected**: `getPauseState` shows domain 1 `submitPaused: false`.
- **Check**: `getPauseState`.
- **Rollback or pause**: none.
- **Devnet**: `run`

## G. Acceptance checks

### G.1 zeno-executor CI steps

- **Call or command**: at the zeno-executor commit `up` prints, `{ go version && go build ./... && go vet ./... && go test ./... && ./scripts/check-trie-imports.sh; }`, the CI job's steps (`.github/workflows/ci.yml:20-23`). The §31 suites run in `./internal/codec` (serialization), `./plugins/wasm` (execution) and `./internal/state` (SMT).
- **Expected**: exit 0.
- **Check**: the log shows every package `ok`. It is the evidence that the §31 suites pass at that commit.
- **Rollback or pause**: none.
- **Devnet**: `run`

### G.2 zeno-executor CI run

- **Call or command**: `gh run list --repo <origin of zeno-executor> --workflow ci --commit <sha> --json conclusion,url`
- **Expected**: CI runs only on pushes to `main` and on pull requests (`ci.yml:3-6`), so an empty list is the expected result for a `dev` commit. zeno-executor has no remote until the human pushes it after the push-time steps of Ticket E2-28, which make CI able to build it.
- **Check**: a run with conclusion `success` for the pushed commit closes the "in the executor's CI" part of the §31 criterion. An empty list leaves it open, and the run record's Open criteria list names it.
- **Rollback or pause**: none.
- **Devnet**: `run`

### G.3 go-zenon tests

- **Call or command**: at the go-zenon commit `up` prints, `go test ./common/trie/... ./vm/embedded/... ./vm/constants/...`.
- **Expected**: exit 0.
- **Check**: every package `ok`.
- **Rollback or pause**: none.
- **Devnet**: `run`

### G.4 go-zenon devnet-build tests

- **Call or command**: `go test -tags devnet ./vm/constants/...`
- **Expected**: exit 0.
- **Check**: every package `ok`.
- **Rollback or pause**: none.
- **Devnet**: `run`

## H. Trust-model disclosure

### H.1 Publish the disclosure

- **Call or command**: publish the SPEC §1 statement that Phase 1 is unbonded single-executor attestation on testnet, not trustless execution, with the SPEC §26 DA and trust limits.
- **Expected**: the statement is public before D.2 opens the gateway, so before any user deposits.
- **Check**: the published page quotes the statement.
- **Rollback or pause**: none.
- **Devnet**: `not run on devnet`

## Devnet run record

The latest pass ran on 2026-10-29, after the Ticket E2-28 fixes. It re-ran B.1, B.2 and E.1 to F.8, and F.9 ran in the zeno-devnet test run named in its row. Every other step run on the devnet keeps its 2026-10-25 result, recorded under "2026-10-25 run".

### 2026-10-29 pass

Date: 2026-10-29

| Repo | Branch | Commit |
|---|---|---|
| go-zenon | feature/settlement | 938fca3 |
| zeno-executor | dev | 4d4df30 |
| zeno-console | main | 9e06bca (the pass ran this commit; main is now 1cb0a8b, which adds the N-1 fix) |
| znn-typescript-sdk | feature/settlement | e3b7518 |
| zeno-client-ts | main | 35610d2 |
| zeno-devnet | main | c9401de |
| nom-webwallet | feature/wallet-connect | 731436d, built with `npm run build:extension` |
| zeno-contract-sdk-rs | main | f616b1c |

| Vault | zeno-contract-sdk-rs commit | `code_hash` | `instrumented_code_hash` |
|---|---|---|---|
| `vault.wasm` | f616b1c | `f7de3e6f84434ce1ade0c45d91c0720c36d2892db3eef80c1e742717753569ff` | `29f032c42a1c22e66389f27ac64f0ab90da09e7fa86b1c7a16a09b8e56d0405f` |
| `vault-testhook.wasm` | f616b1c | `41c9278ed7f69351fc64154e4984ea736b8e8d5a6946cf2f439a5e9103005b4e` | `f254bb5dd0bcd78f4ab4da192e024786d31bf3388d37b74e35ac4292648be358` |

No release tag exists for the vault artifact yet, so the commit and the hashes identify it. The push-time steps of Ticket E2-28 tag it `v0.1.0`.

| Step | Result | Duration | Findings |
|---|---|---|---|
| B.1 | pass | not timed | |
| B.2 | pass: every published port binds `127.0.0.1` (35991, 35997, 35998, 35999, 36000) | about 10 min | |
| B.3 to B.10 | pass (`ok configuration`) | in B.2 | |
| D.1 | pass | in B.2 | |
| D.3 | pass (executor-replacement and crash-recovery flows ended with PASS, see F.9) | 37 min | |
| E.1 | pass after clearing the console's `localStorage` | not timed | N-1 |
| E.2 | pass | not timed | |
| E.3 | pass | 5m42s from `pending` to `finalized` | |
| F.1 | pass | not timed | |
| F.2 | pass | not timed | |
| F.3 | pass | not timed | |
| F.4 | pass | not timed | |
| F.5 | pass: the executor stopped at 11:01:33, before either `finalizeAt`, and `getReleaseQueue` was empty | not timed | |
| F.6 | pass: the human relayed the F.4 withdrawal from index 7 with "Mine only" off, and released from index 9 | not timed | |
| F.7 | pass | not timed | |
| F.8 | pass on the first attempt: the popup showed index 1 for Pause and Resume, with no window closed after the account switch | not timed | |
| F.9 | pass in the zeno-devnet test run that finished on 2026-10-29, against the zeno-executor tree of 4d4df30: all ten flows ended with PASS (deploy-quota 11m28s, deposit 9m59s, call-refund 9m57s, withdrawal 11m9s, force-inclusion 6m30s, worker-crash 12m18s, crash-recovery 17m30s, executor-replacement 19m1s, pause 11m16s, fees 22m19s). `flow:deposit` passed again after that change's last fix | 131 min | |
| F.10 | pass (F.8 Resume) | in F.8 | |

Input receipts, read by the tester with `zeno_getReceipt` at `finalized`. Each has status SUCCESS and `verified: true`:

| Step | Send | Hash | Batch |
|---|---|---|---|
| E.2 | Start | `b7e001e8…` | 0 |
| E.2 | Chunk | `3ad3b71b…` | 1 |
| E.2 | Finalize | `b6070c8e…` | 2 |
| E.3 | BuyStorage | `564cd262…` | 3 |
| F.1 | Deposit | `e52b3135…` | 4 |
| F.2 | Start | `58731fd6…` | 5 |
| F.2 | Chunk | `d3ac7408…` | 5 |
| F.2 | Finalize | `2f147607…` | 6 |
| F.2 | BuyStorage | `5e01cfc3…` | 7 |
| F.3 | `claim_partial` | `00cced4f…` | 8 |
| F.4 | `withdraw` | `43e983b6…` | 9 |

The E.2 Finalize `codeHash` is `29f032c4…405f` and the F.2 Finalize `codeHash` is `f254bb5d…e358`. Each equals its artifact's `instrumented_code_hash`. The vault is contract `4dbe92ec…` and the test vault is contract `a2e74c9a…`.

L1-only sends, each read back from Settlement:

- F.6: the Relay of the F.3 refund from index 8 (Settlement height 39), the Relay of the F.4 withdrawal from index 7, then Release from index 9 (height 44). After Release, `getReleaseQueue` is empty, `totalReleased` is 200,000,000 and `pendingWithdrawalReserve` is 0.
- F.8: Pause from index 1 (height 46) left domain 1 at `submitPaused: true`, `relayPaused: false`. Resume from index 1 (height 47) set `submitPaused: false`.

Conservation evidence: `status` printed `ok conservation` after E.2, E.3, F.1, F.2, F.7 and F.8. It was not taken after F.3, F.4, F.5 or the F.6 sends. For those steps the `getAssetInfo` reads are the evidence. Before the F.6 Relays, `totalDeposited`, `pendingWithdrawalReserve` and `totalReleased` read 300,000,000, 200,000,000 and 0. After Release they read 300,000,000, 0 and 200,000,000, so the asset inequality held at each read.

Final `scripts/devnet.sh status` after F.8: only `ok` lines, `ok configuration` and `ok conservation` included.

Console checklist, from zeno-console `docs/smoke-checklist.md`:

| Item | Steps | Result |
|---|---|---|
| Deploy | E.2, F.2 | pass |
| Call | F.3, F.4 | pass |
| Manage | E.3, F.1 | pass |
| Withdrawals | F.5, F.6, F.7 | pass |
| Admin | F.8 | pass |

The pass records no result for the Redeploy, Upgrade, Domain 2, Events and Sidebar items.

New finding:

- N-1 (zeno-console, defect, fixed in 1cb0a8b): a deploy journal is keyed `domainId:deployer:codeHash` with no chain identity, and a finished journal stayed saved. After the wipe, deploying `vault.wasm` from index 8 resumed the previous chain's finished journal: the page showed Start, Chunk and Finalize as published and sent nothing. The same happened on any network when one address deployed the same code twice. The fix starts a new journal whenever the stored one finished (zeno-console `src/core/composables/useDeploy.ts:111,229`), and the Deploy page can discard a stale pending journal.

Open criteria, each quoting an E2-27 acceptance criterion the pass leaves open:

1. "All §31 conformance suites green in the executor's CI and the node's trie/embedded tests": zeno-executor has no remote yet, so no CI run exists (F-22, accepted). The push-time steps of Ticket E2-28 make CI able to build it, and G.2 closes the criterion after the push.
2. "`totalReleased + pendingWithdrawalReserve ≤ totalDeposited` per (domain, asset) at every step" and the fee conservation criterion: the `conservation` line was not taken after F.3, F.4, F.5 or the F.6 sends (see "Conservation evidence"). No fee-counter reading exists for those states.

The receipt and proof criterion is met: every input receipt above verified at `finalized`, F.2 included, and the console's `/tx` page checks each receipt with the zeno-client-ts verifier, as the scripted flows do.

### 2026-10-25 run

The Ticket E2-27 run. Its findings, and how Ticket E2-28 closed each one, are in `phase-1/tickets/epic-2-executor/E2-27-findings.md`. The steps below were not re-run on 2026-10-29.

| Repo | Branch | Commit |
|---|---|---|
| zeno-devnet | pipeline/e2-27-activation-runbook | 2eb05a2 |
| go-zenon | pipeline/e2-27-devnet-repo | 2679182 |
| go-zenon version (built with `-tags devnet`) | | v0.0.9-alphanet-266-g2679182 |
| zeno-executor | dev | 828e0fc |
| znn-typescript-sdk | feature/settlement | e3b7518 |
| zeno-client-ts | main | 7697e35 |

| Step | Result | Duration | Findings |
|---|---|---|---|
| A.3 | pass | 1 s | |
| C.1 | pass | <1 s | |
| C.2 | pass | <1 s | |
| C.3 | pass | <1 s | |
| C.4 | pass | <1 s | |
| C.5 | pass | <1 s | |
| C.6 | pass | <1 s | F-20 |
| C.7 | pass | <1 s | |
| C.8 | pass | <1 s | |
| G.1 | pass | 142 s | |
| G.2 | pass (expected empty read) | 3 s | F-22 |
| G.3 | pass | 24 s | |
| G.4 | pass | 1 s | |
