# Other chains and runtimes

**Status:** Research — informative.
**Design source:** [phase-2/SPEC.md](../phase-2/SPEC.md) for the runtime model; [bridges.md](bridges.md) for the bridge components this document assumes.

Settlement anchors domains and never runs a VM. That allows two kinds of extension without a Settlement code change: a new execution runtime joins as a domain, and a foreign chain joins as a domain whose STF verifies that chain. This document sets out what each kind contributes and what each inherits.

## 1. Two ways to generalise

A domain is a runtime, its STF, its input vocabulary and its verifying key. Settlement treats every domain the same way: it accepts inputs, keeps a per-domain input hash chain, verifies one proof per batch at `SubmitBatch`, and holds custody per `(domain, asset)`.

| | New runtime | Foreign chain |
|---|---|---|
| Domain class | `EXECUTION` | `BRIDGE` (moves value, with custody) or `MESSAGING` (moves data only) |
| What the STF does | Runs contracts: WASM, EVM, Move | Verifies the foreign chain: headers, finality, inclusion |
| Inputs | `CallL2`, `Deposit`, deploy chunks, `BuyStorage` | Relayed headers, deposit proofs, redeems, release confirmations |
| Outbox | Withdrawals and cross-domain messages | `kind=1` peg-out authorisations consumed on the foreign chain |
| Example | ZVM | Ethereum, Solana |

Both reuse the Rust crate `zeno-stf` (codec, SMT, driver), the zkVM guest and prover, the executor shell, which runs every active domain in its own worker process, the gateway, and Settlement unchanged. In Phase 1 only the WASM `EXECUTION` domain is active, under `ATTESTATION`; in Phase 2 every domain runs under `ZK_VALIDITY` with its own verifying key.

## 2. Bringing a runtime

A runtime is a crate under `zeno-stf` that implements the `Runtime` trait. It contributes four things.

- **The STF.** Given a state root and a batch of decoded inputs, produce the new root, receipts, events and outbox messages. For an existing VM this is an interpreter wired to the shared state trait: `runtime-wasm` today, `runtime-evm` as a revm wrapper.
- **Input decoding.** Settlement inputs are shared by every runtime. The runtime decodes only its own payload, for example EVM calldata inside `CallL2`.
- **Genesis.** The state the domain starts from, committed as its first `preStateRoot`.
- **Gas.** A gas table and structural limits, so a batch stays inside `MaxBatchInputs × MaxInputGas` and the prover's budget.

Everything else is inherited: input order and availability from L1, the executor state SMT and its proofs, the witness, batch submission, custody and conservation, withdrawals through the outbox, and fees. The runtime never talks to Settlement; the driver in `core` does.

The runtime follows the `core` determinism rules: no floats, no wall-clock time, no randomness, no hash-map iteration on the consensus path, and no I/O except through the state trait. The same crate builds as the native worker and as the zkVM guest, so any non-determinism appears as a proof that does not verify.

Conformance vectors are the gate. A runtime ships vectors that pin its input decoding, state transitions and commitments. The worker and the guest both pass them before the domain's verifying key is registered. The Phase 1 WASM vectors are the template.

**EVM as the worked case.** ZVM, an EVM rollup on NoM, is the candidate second domain. Zeno wraps its runtime as `runtime-evm` and registers a second domain with its own verifying key. ZVM's front-end applications keep working, because the EVM semantics they target do not change. Settlement does not learn about the EVM: it sees `CallL2` inputs, a proof and an outbox root. Interaction between the two domains is asynchronous through the outbox.

## 3. Integrating a smart-contract chain

A bridge has two verification directions, and a chain with smart contracts makes the second one easy.

| Direction | Secures | Runs in |
|---|---|---|
| Zenon verifies the foreign chain | Minting: a deposit happened and is final | A `ChainVerifier` in the domain's STF, proven in the guest |
| The foreign chain verifies Zenon | Peg-out: Zenon authorised the release | A verifier contract on the foreign chain |

For the first direction the STF runs a light client of the foreign chain. For Ethereum that means sync-committee and Casper FFG finality tracking, with Merkle-Patricia inclusion proofs against `receiptsRoot`; research item [R7](proving/R7-guest-verifies-ethereum.md) evaluates verifying an existing ZK Ethereum light client recursively in the guest. Solana has no light sync committee, so a ZK proof of the bank state is the likely route. Settlement never sees a foreign header; it sees a proof.

For the second direction the foreign contract verifies a Zenon consensus proof: pillar signatures over a momentum header, plus a Settlement storage proof against the momentum state root that shows the `kind=1` authorisation is final. Research item [R6](proving/R6-zenon-finality-on-ethereum.md) designs this proof and measures its EVM verify cost. A Solana program can verify the same SNARK within its compute budget.

Because the foreign chain verifies Zenon itself, peg-out is lock-and-release: the collateral locked at deposit is the peg-out liquidity, with no operator, bond or challenge window. The per-chain difficulty moves to the light client: medium for Ethereum, hard for Solana. Bitcoin is the mirror image, with a cheap SPV light client and a hard peg-out. The bridge ledger, mint and burn through Settlement, replay protection and the `kind=1` authorisation object are inherited; [bridges.md](bridges.md) describes them.

## 4. What stays per chain

| Per chain | New runtime | Foreign chain |
|---|---|---|
| STF crate | Interpreter, gas table, limits | `ChainVerifier`: headers, finality, inclusion |
| Input payloads | Calldata and deploy format | Header, deposit, redeem and release variants |
| Genesis | Initial state root | Checkpointed foreign header or committee root |
| Relay | None | A permissionless relayer that posts foreign data as Settlement inputs |
| Foreign side | None | A lock vault and a Zenon-verifier contract |
| Asset identity | None | One ZTS per bridged asset, minted and burned only by Settlement |
| Verifying key | One, in the verifier registry | One, in the verifier registry |
| Open questions | Gas calibration against proving cost (R2) | Proof portability across curves; light-client data cost; finality-window alignment |

Everything not in this table is shared.
