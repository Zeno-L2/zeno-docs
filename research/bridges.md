# Bridges and messaging channels on Zeno

**Status:** Research — informative. Nothing here is built.

These are design notes on how a bridge would be built as a Zeno domain. They were written against the Phase 1 attestation model; under Phase 2 the execution side of any bridge is covered by the batch validity proof, so only the foreign-fact side, how the STF learns a foreign chain's truth, remains a bridge-specific trust question.

**Schema:** the [Phase 1 SPEC](../phase-1/SPEC.md) §6, §17 and §18 define the reserved fields used here. The [Phase 2 SPEC](../phase-2/SPEC.md) §10 puts the ETH bridge and the EVM runtime out of scope and keeps [R6](proving/R6-zenon-finality-on-ethereum.md) and [R7](proving/R7-guest-verifies-ethereum.md) as research items.

## 1. What a bridge is here

A bridge is an ordinary domain: one executor plugin, one domain record in Settlement and one isolated ledger of balances, plus per-chain code at two seams. There is no second settlement layer and no per-chain withdrawal contract on Zenon. The invariant holds: consensus orders every input, including every foreign fact, as an L1 account-block; the bridge STF verifies foreign facts and keeps balances; Settlement holds custody and enforces conservation per `(domain, asset)`.

Three domain classes exist. `EXECUTION` runs general computation over `ESCROW` assets. `BRIDGE` moves value across a foreign-chain boundary with `ESCROW` or `MINTED` custody per asset. `MESSAGING` moves data only and emits `kind=0` messages alone. One template serves many instances, sharing the code but never the money: each bridge has its own state, liquidity and foreign contracts, and domain isolation keeps a fault in one from reaching another's custody.

## 2. Architecture

### 2.1 Layers and data flow

Four layers take part: Zenon L1, which orders inputs and runs no STF; Settlement, which holds custody, the domain registry, commitments, replay sets and the verifier registry; the executor, which runs one worker per domain plus a relay role for an `L1_RELAYED` domain; and the foreign chain, which runs a `ReleaseAdapter`.

The relay role's `SourceAdapter` posts headers and inclusion proofs to L1 as account-blocks, which take their place in the canonical order beside user inputs such as `Redeem`. The executor applies each confirmed input: the `ChainVerifier` checks the foreign fact and the `BridgeLedger` updates balances. The executor calls `SubmitBatch`, Settlement verifies the proof, and a relayer carries any finalized `kind=1` authorization to the `ReleaseAdapter`.

### 2.2 The exit path is shared

Getting value out is the same three steps whatever the destination: the STF debits or burns on Zenon and emits a `kind=1` outbox message; the batch finalizes; a relayer carries the authorization to its consumer, `ReleaseWithdrawals` for a Zenon withdrawal or the foreign `ReleaseAdapter` for a peg-out. The peg-out authorization is the same outbox object as a withdrawal, with a foreign recipient, which is why one set of Settlement rules serves every chain.

### 2.3 The determinism boundary

L2 state is a pure function of L1-anchored data. The relay role reads the foreign chain only to propose inputs; the STF never reads one, so a syncer replays a bridge from genesis with no foreign network access. In Phase 2 the guest runs the same STF, so the `ChainVerifier` is part of the proven program and reads no clock, network or environment.

## 3. Choosing a configuration

A domain sits at one point in five dimensions: `inputSource`, `execProfile`, `foreignProfile`, `custodyMode` per asset and `proposerPolicy`. Phase 1 activates one point, `(L1_NATIVE, ATTESTATION, NONE, ESCROW, SINGLE)`. In Phase 2 every domain is `ZK_VALIDITY`, so the choice reduces to the foreign-fact rung and the custody mode.

The foreign-fact ladder runs `NONE → COMMITTEE → LIGHT_CLIENT → ZK_CONSENSUS`. Under `LIGHT_CLIENT` the STF verifies relayed headers and inclusion proofs itself. Under `ZK_CONSENSUS` a succinct proof of foreign consensus does the same work, and Core can verify it at `RelayForeignLock`. `COMMITTEE` stays in the schema, but these notes do not build on it: a quorum of attesters is a trusted set the validity proof cannot cover. A domain's trust is the weaker of its two rungs; in Phase 2 that is always the foreign-fact rung.

Custody is chosen per asset. A foreign asset coming in (BTC to zBTC) is `MINTED`: the real asset stays locked abroad and Settlement is the sole issuer of a representation ZTS. A Zenon asset going out is `ESCROW`. `MINTED` requires `ZK_VALIDITY` and a `MaxBatchMint` cap, reconciles through `MintFlowSummary`, and holds `mintedSupply ≤ provenForeignLocks` and `authorizedForeignRelease ≤ burnedSupply`. Burns and balances are STF state Core never sees, so only the proof stops an unbacked mint.

| Bridge | Class | `inputSource` | `foreignProfile` | Custody | Why |
|---|---|---|---|---|---|
| Bitcoin | `BRIDGE` | `L1_RELAYED` | `LIGHT_CLIENT` | zBTC `MINTED` | 80-byte headers relay cheaply; SPV runs inside the proven STF |
| Messaging channel | `MESSAGING` | `L1_RELAYED` | `LIGHT_CLIENT` | none | no custody, so no `MINTED` floor; the cheapest first use of relay machinery |
| High assurance | `BRIDGE` | `L1_RELAYED` | `ZK_CONSENSUS` | `MINTED` | both ladders at the top; Core verifies each lock |
| Ethereum | `BRIDGE` | `L1_RELAYED` | `LIGHT_CLIENT` or `ZK_CONSENSUS` | zETH `MINTED` | out of Phase 2 scope; depends on R6 and R7 |

Every row runs `ZK_VALIDITY`. Climbing the foreign-fact ladder later is a `stfSpecHash` change that leaves the ledger untouched.

## 4. Components a bridge author writes

A bridge contributes a domain plugin, a `ChainVerifier`, a `SourceAdapter`, a `ReleaseAdapter` and one release of Settlement wiring. The runtime, Settlement Core and the shared `BridgeLedger` are reused unchanged.

### 4.1 The domain plugin

The same four-method plugin as the WASM domain. `Genesis()` returns the light-client anchor from `chainBinding.genesisAnchor`; `DecodeInput()` decodes payloads into `Header`, `Deposit`, `Redeem`, `Settle` and `Fraud` inputs; `Apply()` delegates to `BridgeLedger.Apply` with the chain's `ChainVerifier` injected; `Source.Watch()` is the `SourceAdapter`. In Phase 2 the plugin is a runtime in the `zeno-stf` crate and runs inside the guest.

### 4.2 `ChainVerifier`, the read seam

The only chain-specific verification code, a pure function of its arguments and the witnessed state. `VerifyHeaderChain` advances the verified light-client view and rejects any datum that does not extend it. `VerifyInclusion` proves a lock, burn or log sits under a verified header and returns a `ForeignEvent` with the event id, the Zenon recipient and the amount. `VerifyFinality` decides whether the event is final enough to act on. `VerifyConsensusProof` does consensus and inclusion in one step for `ZK_CONSENSUS`.

### 4.3 `SourceAdapter`, the relay seam

`Watch` emits what to post to L1. Relaying is permissionless and carries no trust: the STF re-validates every datum, so a dishonest relayer can only withhold, and anyone can relay in its place.

### 4.4 `BridgeLedger`, the shared state machine

One chain-neutral library runs inside every bridge's `Apply`; only the injected verifier and the asset differ. A `Header` input advances the light client. A `Deposit` proves a final foreign lock, checks the event id against the credited set and credits the recipient toward `mintCredit`. A `Redeem` debits the balance toward `burnDebit` and emits the `kind=1` foreign-recipient authorization. `Settle` and `Fraud` serve chains whose custody uses bonded operators: `Settle` reimburses an operator who provably fronted a release; `Fraud` slashes one who spent the pool without authorization. The batch builder turns credits and debits into the batch's `MintFlowSummary` entry.

### 4.5 `ReleaseAdapter`, the write seam

The foreign-side consumer of a finalized authorization. The SPEC fixes the object and the obligations, never the code:

```
PegOutAuthorization := { domainId, outboxId, asset, amount, recipient(foreign), burnRef }
```

A conformant adapter releases at most once per `outboxId`, keeping its own processed set because `processedOutbox` does not reach the foreign chain. It verifies a proof of Zenon consensus and Settlement storage showing the authorization in a `FINALIZED` batch's `outboxRoot`, releases exactly `amount` of `asset` to `recipient`, and refuses delivery past a deadline so timeouts are provable. A withheld adapter delays a release but cannot over-release. How the adapter learns Zenon finality cheaply is R6.

### 4.6 Settlement wiring

A bridge needs an input handler in `znnd` for its input vocabulary, shipped by a spork-gated release; verifier entries through `RegisterVerifyingKey`; an asset binding per `MINTED` asset; `MaxBatchWithdrawal` and `MaxBatchMint` caps; and a worker for its plugin in the executor release, since every executor runs every adopted domain. Executors post no bond; the proof is the protection against a dishonest executor.

## 5. Lifecycles

### 5.1 Peg-in

1. A user locks the asset on the foreign chain.
2. A relayer posts the lock to L1: headers plus an inclusion proof under `LIGHT_CLIENT`, or `RelayForeignLock` under `ZK_CONSENSUS`, where Core verifies the proof and raises `provenForeignLocks`.
3. The STF verifies the lock, checks the event id against its credited set and credits the recipient.
4. The batch's `MintFlowSummary` carries the credit as `mintCredit`; `SubmitBatch` checks it against `MaxBatchMint`.
5. At finality Core applies `mintedSupply += mintCredit`. Under `LIGHT_CLIENT`, `provenForeignLocks` rises by the same amount, secured by the validity proof.

Withdrawing the representation to a Zenon address is an ordinary `kind=1` withdrawal; `ReleaseWithdrawals` mints the ZTS.

### 5.2 Peg-out

1. The user sends a `Redeem` into the bridge domain. A holder of the L1 ZTS first deposits it, which burns it at the receive.
2. The STF debits the representation and emits the foreign-recipient authorization; the summary carries `burnDebit`.
3. The batch finalizes and Core applies `burnedSupply += burnDebit`. In Phase 1 finality follows the withdrawal delay; in Phase 2 acceptance at `SubmitBatch` is finality.
4. A relayer calls `RelayMessage` with an inclusion proof against the finalized `outboxRoot`. Settlement records the `outboxId` and raises `authorizedForeignRelease`, which may not exceed `burnedSupply`.
5. A relayer carries the authorization and its Zenon proof to the `ReleaseAdapter`, which releases once per `outboxId`.

### 5.3 Messages and acknowledgements

A `kind=0` message is delivery, not value. It moves `EMITTED → COMMITTED → FINALIZED → RELAYED → DELIVERED → EXECUTED → ACKED`, or from `FINALIZED` to `TIMED_OUT → REFUNDED`. Delivery is at-most-once per `outboxId`. There is no total order across batches; a channel that needs ordering carries sequence numbers in its payload. An acknowledgement is itself a `kind=0` message with its own `outboxId`, and a lost acknowledgement never reverts an executed message.

### 5.4 Timeouts

A timeout is proved against the destination's processed set, the same set that makes delivery at-most-once. For a Zenon destination, `TimeoutMessage` checks that the message is finalized, past its deadline and unprocessed, marks it timed out and hands it back to the source STF, which refunds. For a foreign destination, a relayer posts a non-membership proof against the `ReleaseAdapter`'s processed-set root at a finalized foreign block past the deadline. The windows must satisfy `sourceFinality + destinationFinality ≤ timeoutDeadline` so a refund never races a valid delivery.

### 5.5 Replay keys

| Surface | Key | Enforced by |
|---|---|---|
| L1 input | source account-block hash, bound to `chain_id` | L1 account-block uniqueness |
| Message relay on Zenon | `outboxId = SHA3-256(domainId ‖ batchId ‖ inputIndex ‖ outboxIndex)` | `processedOutbox` |
| Foreign release | `outboxId` | the `ReleaseAdapter`'s own processed set |
| Peg-in credit | foreign event id (outpoint, log id) | the STF's credited set; Core's set under `RelayForeignLock` |
| Batch | per-domain input cursor | Settlement contiguity |

No key covers two surfaces.

## 6. Standing up a bridge

1. Pick the class, the foreign-fact rung and the custody mode; fix `chainBinding` (`chainVerifierId`, `genesisAnchor`, `releaseAdapterRef`, `finalityParams`) and any timeout windows.
2. Write the `ChainVerifier` and test it against real foreign headers, including reorgs; write the plugin around `BridgeLedger` and the `SourceAdapter`.
3. Write, audit and deploy the `ReleaseAdapter` with upgrade governance that cannot turn it into a custodian.
4. Ship the input handler and verifiers in a `znnd` release; register verifier entries and asset bindings; call `RegisterDomain` with conservative caps.
5. Run an executor and relayers. Raise caps and climb the foreign-fact ladder as `stfSpecHash` changes once the verifier and adapter have earned confidence.

## 7. Worked examples

### 7.1 Bitcoin with an SPV `ChainVerifier` and `MINTED` zBTC

Domain `d_btc = (BRIDGE, L1_RELAYED, ZK_VALIDITY, LIGHT_CLIENT)`.

A Bitcoin bridge reduces to two verification problems, and only one is hard. Zenon verifying Bitcoin is easy: the `ChainVerifier` is an SPV client inside the proven STF, checking PoW headers, a Merkle branch and `K` confirmations. Bitcoin verifying Zenon is hard: Bitcoin script cannot run a Zenon light client, so it cannot natively release BTC only when Zenon says so. That one fact makes peg-out custody the chain-specific problem. Settlement's product is provable authorization, not custody; custody lives in Bitcoin script, and no pooled key exists whose holder could abscond.

Two custody options exist for the `ReleaseAdapter`. The buildable option is a pre-signed BitVM transaction graph with bonded operators: BTC is locked under a Taproot output whose only spendable futures are the graph and a user refund after timeout, operators discard their keys, and security is one honest operator plus bonds. The endgame is a covenant that verifies the Zenon proof in-script; it needs a Bitcoin soft fork and deletes the operators. The Settlement side is identical in both, and the optimistic element of BitVM lives in Bitcoin script, not in Zeno.

1. A user sends 0.5 BTC to the peg-in output. A relayer posts the PoW headers and a Merkle proof.
2. The STF verifies the lock after `K` confirmations and credits 0.5 zBTC.
3. On `Redeem`, the STF debits 0.5 zBTC and emits the foreign authorization.
4. A bonded operator fronts 0.5 BTC to the user, then claims reimbursement from the pool by presenting the Zenon proof in the BitVM graph; a `Settle` input records it on Zenon.
5. An operator who spends pool collateral without a matching authorization is proven by a `Fraud` input and slashed.

The Bitcoin confirmation depth, Zenon finality and the BitVM challenge window must be chosen together so no peg-out finalizes over a reorganisable event. Open points: how narrow a statement the Bitcoin side needs proven, bond denomination, operator rotation, and pre-declared pool rebalancing.

### 7.2 A `MESSAGING` channel with acknowledgement

Domain `d_msg = (MESSAGING, L1_RELAYED, ZK_VALIDITY, LIGHT_CLIENT)`, bound to chain `C`. A contract emits a `kind=0` message to a recipient on `C` with deadline height `H` and `ackPolicy = ON_EXECUTION`.

1. The batch commits the message in `outboxRoot` and finalizes.
2. A relayer presents the inclusion proof on `C`. The endpoint checks that the `outboxId` is unprocessed and that `C` is below `H`, then executes it.
3. `C` emits an acknowledgement with its own id. A relayer posts it back as an `L1_RELAYED` input and the source `ChainVerifier` verifies it.

No custody counter moves. Had `C` reached `H` first, a relayer would post the non-membership proof instead and the source STF would refund.

## 8. Security

Each defense bounds one failure mode; none upgrades a profile's trust. An `ATTESTATION` bridge is still executor attestation and belongs on test networks only.

### 8.1 Threats and defenses

| Threat | Defense | Residual |
|---|---|---|
| Replay of an input, message, peg-in, release or batch | one key per surface, enforced independently | an adapter without its own processed set is non-conformant |
| Withheld bundle | inputs are L1 account-blocks; only the witness bundle is off-chain | under `ZK_VALIDITY` state is proven and replayable from L1 inputs |
| Invalid or mismatched proof | every proof checked against its registered context: `outboxRoot`, `domainId`, `batchId`, the registry entry, the `ChainVerifier` | proof-system soundness is assumed |
| Foreign reorg or wrong finality | inputs only from confirmed momenta; `finalityParams` chosen so no release follows a reorganisable event | too shallow risks reorgs, too deep costs latency |
| Cross-domain fault | custody and replay keyed by `domainId`; one worker per domain | none at the protocol level |
| STF or adapter bug | conservation bounds loss per `(domain, asset)`; a divergent root has no valid proof | an adapter bug is bounded by that adapter's governance |
| Version skew | `stfSpecHash` pins the STF and profile bindings; changes wait `MinRuntimeUpgradeDelay` | Zenon cannot force a foreign adapter to upgrade |

### 8.2 Governance key compromise

The administrator controls Periphery only; Core is node binary. Every administrator action except emergency pause waits `SoftDelay` or `AdministratorDelay`, and Core bounds such as the `MaxBatchWithdrawal` floor and `MinRuntimeUpgradeDelay ≥ WithdrawalDelay` hold against a compromised key. The escape hatch, the only administrator action that moves custodied funds, is time-locked and public.

The residual is liveness. A compromised key can set `PAUSE_RELAY` with no delay and hold it, freezing releases that are already finalized. Pause moves no value and alters no root or counter, but to a holder an indefinite freeze looks like loss until administrator rotation activates. The structural fix is a multisig or governance address as administrator. On the foreign side the `ReleaseAdapter`'s upgrade governance is the trust root: time-locked and audited, ideally immutable or proof-gated, so a compromise cannot reintroduce a custodian.

### 8.3 Operations that must fail

1. A non-contiguous, replayed or oversized batch; a `preStateRoot` that does not chain; an unrecognized version field.
2. A proof that fails its registered verifier.
3. A `RelayMessage` with a mismatched `domainId` or `batchId`, an unfinalized batch, or a processed `outboxId`.
4. Any operation that would break a conservation invariant, including a mint beyond `MaxBatchMint` or a release beyond `burnedSupply`.
5. A `RegisterDomain` with a reserved value or an illegal combination; a Periphery value outside its Core bound; any executor consistency fault, which halts.
