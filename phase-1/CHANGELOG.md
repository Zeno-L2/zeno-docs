# Phase 1 — Changelog

Revision history for the Phase 1 document set (`SPEC.md`, `ARCHITECTURE.md`, `EXECUTOR.md`, companion artifacts). The documents themselves are forward-reading; all history lives here.

---

## SPEC v1.17.10, EXECUTOR v0.12.3 — 2026-10-30

### Open items carried to Phase 2 (§30, §32)

§30 and §32 no longer hold the open and deferred items: the pre-mainnet parameters, `OD-4`, `OD-5` and their resolution rule, the optional landing-zone-balance extension and the pre-mainnet implementation tasks move to `../phase-2/SPEC.md` §11, which owns their resolution. §30 keeps the reserved bridge-framework paragraph and the Phase 2 pointer; §32 records the companion artifacts and the reference vault as delivered and points at Phase 2 for the rest.

### Document set

The bridge guide lives at `../research/bridges.md` and the architecture companion at `ARCHITECTURE.md` in this folder; every reference in `SPEC.md` and `EXECUTOR.md` follows. No normative rule changes.

## SPEC v1.17.9 — 2026-10-21

### Undeployed target vectors (§10.6, §15, §18.2, §31)

The execution suite gains `CV-DEPOSIT-UNDEPLOYED-1` and `CV-CALL-UNDEPLOYED-1`, and its README's coverage table lists them. They pin §18.2's rule for a `Deposit` and a payable `CallL2` whose target has no code-hash leaf: `VALIDATION_FAILED` at `gas_used` 0 with a zero receipt `codeHash`, the full deposit refunded as the only outbox message, and a witness set of exactly the target's two absent code-hash leaves. §18.2 states the rule behind that set: for a `CallL2` or `Deposit` the runtime reads the target's `CodeHash` leaf and then its pre-instrumentation `code_hash` leaf, unconditionally and before deciding whether the target is deployed or looking up its code in the code store. Both use a new `undeployed_target` shape, which carries no module and no frame. §31 lists them among the cases the artifact carries and no longer lists a case it does not carry. No existing vector changes. The `wasm-vm/` copies of the suite and its README match their sources of record.

### Copy registry

`artifacts/README.md` lists the recorded gateway responses written by go-executor's `TestDumpGatewaySdkFixture`, and the contract SDK's example artifacts that client repositories copy: three interface files and the vault module. For each it names the copies in zeno-client-ts and zeno-console, their readers and their pins. Its checksum commands cover the interface files, the vault module and the execution README copy, compare each gateway copy with a regenerated file, and run from the folder holding the zeno checkouts, with go-zenon and znn-typescript-sdk under `../zenon-network/`. The gateway check states its build precondition: go-executor's `go.mod` replace of go-zenon must resolve.

---

## SPEC v1.17.8 — 2026-10-21

### ExecutionResult input_index (§27.1a)

An `ExecutionResult` decoder must reject an event or outbox message whose `input_index` differs from the result's `globalInputIndex`, as a `ContractEffects` decoder already must. go-executor's decoder already enforces it. It is a pre-activation correction under §24: no version field changes.

### ExecutionResult and MerkleProof serialization vectors (§27.0, §27.1a, §27.2, §27.3, §27.4, §31)

The serialization suite gains `CV-RESULT-1`, `CV-RESULT-2`, `CV-RESULT-OUTBOX-1`, `CV-RESULT-OUTBOX-2` and `CV-MERKLE-1`, and §31 lists them. They pin the `ExecutionResult` decoder's rejection of a trailing byte, of roots placed after `claimed_deposit`, of a `StateDiff` in descending or duplicated derived-key order, and of a foreign `input_index`; its acceptance of `MaxOutboxPerExecution + 1` outbox messages and rejection of `MaxOutboxPerExecution + 2`; and the `MerkleProof` verifier's rejection of a `side` of 2 at a right-sibling and at a left-sibling step, a proof of `MaxMerkleProofDepth + 1` well-formed steps, a trailing byte, a leaf hash given as the element, and the empty-set root. A swap of two equal-width `ExecutionResult` fields re-parses as a different valid value, so no vector can pin it. §31 records that `CV-INPUTROOT-1` already pins odd-node promotion. No existing vector changes. The suite's `notes` names its zeno-docs copy in `artifacts/`. The `artifacts/` copy and zeno-client-ts's copy match the source of record.

---

## SPEC v1.17.7 — 2026-10-21

### Import kind and signature, run export and function-type parameters (§8.1, §25.1, §27.1, §31)

Each of `storage_len`, `storage_read` and `abort` must be imported as a function with exactly its §8.1 signature: a module importing one of those names as a table, memory or global, or with any other signature, is `VALIDATION_FAILED` at deploy. `run` must name a function the module defines: a module whose `run` export names an import is `VALIDATION_FAILED` at deploy. `MaxLocalsPerFunction` also caps the parameter count of each declared function type, whether or not a function uses it. go-executor's validator already enforces all four rules. It is a pre-activation correction under §24: no version field changes. §31 carries a bullet for the deploy validation vectors.

### Deploy validation vectors (§8.1, §11.3, §25.1, §27.1, §31)

The execution suite gains `CV-STRUCT-TYPES-1`, `CV-STRUCT-TYPEPARAMS-1`, `CV-STRUCT-PARAMS-1`, `CV-STRUCT-FUNCS-1`, `CV-STRUCT-TABLE-1`, `CV-STRUCT-GLOBALS-1`, `CV-STRUCT-ELEMS-1`, `CV-STRUCT-DATA-1`, `CV-STRUCT-LOCALS-1`, `CV-STRUCT-TOTALLOCALS-1`, `CV-STRUCT-ATCAP-1`, `CV-FIT-DATA-1`, `CV-FIT-ELEM-1`, `CV-IMPORT-MODULE-1`, `CV-IMPORT-NAME-1`, `CV-IMPORT-DUP-1`, `CV-IMPORT-SIG-1`, `CV-IMPORT-KIND-1`, `CV-SECTION-START-1`, `CV-SECTION-CUSTOM-1`, `CV-SECTION-CUSTOM-2`, `CV-EXPORT-MUTGLOBAL-1`, `CV-EXPORT-START-1`, `CV-EXPORT-ZENON-1`, `CV-EXPORT-NORUN-1`, `CV-EXPORT-NOMEMORY-1`, `CV-EXPORT-RUNSIG-1`, `CV-EXPORT-RUNIMPORT-1`, `CV-INSTR-SIZE-1` and `CV-LEB-PADDED-1`, and its README's coverage table lists them. They pin each §25.1 structural cap one above its limit, eight caps and both segment-fit bounds at their limit, the §8.1 import rules, the start and custom section rejections, the export rules, a module under `MaxCodeSize` that instruments past `MaxInstrumentedCodeSize`, and §11.3's rule that every LEB128 integer uses its shortest encoding. A `validation_rejection` vector's cause can now be its instrumentation. No existing vector changes. The README states why `MaxCodeSize`, the at-cap `MaxTotalLocals`, a successful deploy of a function type with exactly `MaxLocalsPerFunction` parameters, and §8.1's fourth import have no vector of their own. The `wasm-vm/` copies of the suite and its README match their sources of record.

---

## SPEC v1.17.6 — 2026-10-21

### Memory limit (§7.4, §11.3, §31)

A module's memory limit is the smaller of its declared maximum and `MaxMemory`, or `MaxMemory` when it declares none. A `memory.grow` past the limit returns `-1`, grows nothing and is charged only its base cost. The instrumenter writes the limit as the instrumented memory's maximum, and its growth limiter checks against it. A module whose declared initial memory exceeds `MaxMemory` is `VALIDATION_FAILED` at deploy. It is a pre-activation correction under §24: no version field changes. No existing vector changes, because every existing module declares no maximum or a maximum of `MaxMemory`. §31 carries a bullet for the memory-limit vectors. `wasm-gas-table-v1.md` §4 refers to the memory limit.

### Module behaviour vectors (§7.4, §7.5, §8.1, §16, §18.5, §27.1, §27.3, §31)

The execution suite gains `CV-TRAP-DIV-1`, `CV-TRAP-OVF-1`, `CV-TRAP-MEM-1`, `CV-TRAP-TABLE-1`, `CV-TRAP-NULL-1`, `CV-TRAP-SIG-1`, `CV-NEG-1`, `CV-NEG-2`, `CV-FRAME-2`, `CV-FRAME-3`, `CV-GROW-2` to `CV-GROW-5`, `CV-READ-3` to `CV-READ-7`, `CV-EVENTS-1`, `EXEC-DEP-12` and `CV-MEM-MIN-1`, and its README's coverage table lists them. They pin six trap classes, `run` returning `r < 0`, the frame written after the data segments, a module too small for the frame, the four memory-limit cases, the deploy rejection of 65 initial pages, the `storage_read` resolution order (including a key range past the end of memory), three events with an odd promotion in `event_root`, and a deposit refund that is the only outbox message. No existing vector changes. The `wasm-vm/` copies of the suite and its README match their sources of record.

---

## Execution conformance suite, 2026-10-21

No SPEC or EXECUTOR change.

### Deploy lifecycle vectors (§11.3, §12, §15, §31)

The execution suite gains `CV-DEPLOY-3`, `CV-DEPLOY-HASH-1`, `CV-UPGRADE-UNAUTH-2`, `CV-UPGRADE-ORDER-1`, `CV-UPGRADE-INVALID-1`, `CV-UPGRADE-HASH-1`, `CV-UPGRADE-SAMEHASH-1` and `CV-UPGRADE-SAMEHASH-2`, and its README's coverage table lists them. They pin multi-chunk assembly order, the assembly hash check on finalize and on upgrade, the owner-mismatch refusal, authorization before the §11.3 pipeline, an authorized validation failure, and §12's rule that an upgrade to the current `CodeHash` is `VALIDATION_FAILED`. No existing vector changes.

The README states how a kind-4 or kind-5 vector stages its pending record, and why a length mismatch and out-of-bounds record fields have no vector. It names its copy as `zeno-docs` `wasm-vm/`. The `wasm-vm/` copies of the suite and its README match their sources of record.

---

## SPEC v1.17.5 — 2026-10-20

### Function-entry locals charge (§10.2, §25.1, §31)

Under `metering_version = 1`, a function's entry segment costs `ceil(L / locals_per_gas)`
more, `L` its declared locals, `locals_per_gas = 2`. It is a pre-activation correction
under §24: the table changes in place and no version field changes. `MaxLocalsPerFunction`
stays 8,192 as the backstop. The execution suite gains `CV-LOCALS-1` and `CV-LOCALS-2`; no
existing vector changes, because none declares locals. The `WASM Spec/` copies of the gas
table and the execution suite match their sources of record.

### Gas table prose

`cost_principles` in `wasm-gas-table-v1.json` and §2 of `wasm-gas-table-v1.md` list integer
conversions at cost 1, as their entries do. No price changes.

---

## SPEC v1.17.4 — 2026-10-18

### Reference vault (§18.1a, §32)

The vault is the **reference vault contract**: production-quality example code for
contracts that hold value, and the testnet demo contract. It has no protocol role and no
planned mainnet deployment. Ticket E2-24 is "Reference vault contract" and drops the
audit and the published `ContractID`.

### Tickets

Local-variable gas pricing is E2-25 and lands after E2-24. Conformance coverage is E2-26.
E2-27 is the end-to-end run on a local devnet, including a console run with a real
nom-webwallet build; it waits on the console (client milestones C1–C8) and writes the
activation runbook, `runbooks/ACTIVATION.md`. Testnet activation is a later ticket that
follows the runbook. E2-28 fixes the run's findings.

### Companion artifacts (§31)

`WASM Spec/` copies of `execution-conformance-v1.json` and its README,
`serialization-conformance-v1.json` and `smt-v1-test-vectors.json` match their sources of
record. The SMT suite carries `CV-PATH-1`, `CV-PATH-2`, `CV-APPLY-1` and `CV-APPLY-2` as
entries beside `SMT-001`…`SMT-014`. `WASM Spec/README.md` lists every artifact's source,
readers and copies.

### Repository names

The repositories are `zeno-docs` (this document set), `zeno-client-ts` (npm package
`zeno-client`), `zeno-contract-sdk-rs`, `zeno-console` and `go-executor`. Open tickets, the
client plans, `contract-sdk.md` and `WASM Spec/README.md` use these names.

---

## SPEC v1.17.3 / EXECUTOR v0.12.2 — 2026-10-18

### Proposer policy and executor liveness (§5.4, §6.2, §22, §25.2, §27.11, §32)

`SINGLE` is the only proposer policy. Ordinals `1` and `2` of `proposerPolicy` are
unassigned, and Core rejects them; the proposer rule for a multi-member executor set is
specified with Phase 2. The executor `status` field has one value, `ACTIVE`; ordinals
`1` and `2` are unassigned. §22.6 is unassigned: the missed-turn, lapse and grace rules,
`ExecutorLapseWindow` and `DomainActivationGrace` are removed, together with the
reserved liveness fields in the executor-set storage. EXECUTOR.md §5.5 and §13 no longer
describe a proposer schedule.

### Companion documents

`ARCHITECTURE.md`, `BRIDGES.md`, `DEMOS.md`, the client plans, `contract-sdk.md`, the
gas-table notes and the state-root guides describe two phases: attestation on testnet
and validity proofs (`ZK_VALIDITY`) on mainnet. They describe no watcher role, fraud
proof, dispute game, challenge window, `Challenge` method or libp2p bundle path, and the
gateway serves bundles over HTTP. `MINTED` assets require `ZK_VALIDITY`, and a foreign
`ReleaseAdapter` verifies a proof of Zenon consensus and of Settlement storage. The
contract input frame header is 254 bytes.

---

## Contract SDK and clients, 2026-10-17

No SPEC or EXECUTOR change.

### Rust contract SDK (`WASM Spec/contract-sdk.md`)

The SDK is the `zeno-wasm-sdk` repository with four crates: `zeno-contract`, `zeno-contract-macros`, `zeno-testing` and `cargo-zeno`. There is no Rust client and no `cargo zeno deploy`; deploy and calls live in zeno-console. Contracts link no `alloc`, and memory has a fixed layout with the stack first. Error enums use `#[contract_error]`, and storage has no `load`, `update` or read cache.

D-8 fixes the SDK failure codes in a numbered table, -1 to -9, with -9 `InvalidRecipient` for a withdrawal to an address that is not a user address and -7 reserved. Code -5 compares a claim with the claimable amount, which is zero outside `#[on_deposit]` and payable methods. `AssetId` decodes only the canonical form, so a withdrawal never carries an asset the runtime rejects.

D-9 defines the `interface_version: 1` schema: its members, the Method, Event, Error and StorageEntry shapes, and the type list. `hashed` is a per-entry property fixed at declaration, true for any variable-length key field or more than 30 key bytes, so an unhashed key never equals a hashed one. The types are `u8` to `u256`, `u128` and `i8` to `i128` included, and `i256` is not a type. `errors` lists the SDK codes a module can raise by a fixed rule.

The build validates and instruments through `zeno-exec validate` and writes `<name>.hashes.json`. Tests run through `zeno-testing` over `zeno-exec run`, with no snapshot files.

### Clients and tickets

`ZENO-SDK.md` and `ZENO-CONSOLE.md` drop constructor arguments and follow D-9's types. The console gains an optional test-before-deploy panel over `zeno-exec serve`. E2-24's vault lives in zeno-wasm-sdk `examples/vault`, and E2-25 deploys its release artifact.

---

## SPEC v1.17.2 / EXECUTOR v0.12.1 — 2026-10-17

### SMT adapter (§13.1, §13.5.1, §13.6; EXECUTOR.md §5.3, §19)

The L1 maintenance adapter is `nodestore.go` (`NodeTree`), with its empty-value fold
rule in `momentum_fold.go` (`stagedApplier`). Its write-set methods `Update`,
`AccumulateFrom` and `ComputeRoot` and its `Prove` hash a key into a path, as do
`VerifyProof` and `VerifyAbsence`, and the L2 executor must not call any of them.
The sentence that introduces that list says the functions hash a key into a path,
because three of them take a patch and hash only the keys `foldKeep` admits (§13.1).

### Alignment with the implementation (§10.3, §11.3, §12, §13.1, §14.1, §18, §27)

The path-native functions take typed arguments. `RootOfLeaves` returns `(Hash, error)`
and fails a repeated path with `ErrDuplicatePath`. `ProveByPath` takes the leaf set as
parallel `paths` and `values` slices and returns an error. `VerifyProofByPath` and
`VerifyAbsenceByPath` take `root Hash` and `path Path` (§13.1).

Settlement rejects a `SubmitBatch`, `RelayMessage` or `SetAssetConfig` that names an
`AssetID` with a non-zero byte in `[0:22]` at send validation, with no receipt status
(§18.1).

Core tracks aggregate custody counters and keeps no per-account balance ledger of
custodied assets. The per-address developer-fee reward record lies outside that rule
(§18.1a).

A batch's worst case is `MaxBatchInputs × (MaxOutboxPerExecution + 1)` withdrawals,
deposit refunds included, so it clears in 132 momentums (§18.2).

The runtime takes the value-proportional part of the `storage_read` charge after it
reads the key and before the buffer checks. An unaffordable charge ends the input with
`OUT_OF_GAS`, and the key stays observed (§10.3).

Validation requires every LEB128 integer in its shortest encoding (§11.3). The
`UpgradeContract` signature carries `domainId` (§12).

The L1 `Address` also appears as the `new_owner` field of §27.5 and the `owner` field of
§27.12 (§14.1, §27.0).

A `ContractEffects` `StateDiff` list has no required order, while an `ExecutionResult`
`StateDiff` list is sorted by derived key (§27.1a). The `AssetFlowSummary` lists an
asset when its deposit or withdrawal amount is non-zero, so an asset touched only by a
zero-amount withdrawal has no entry (§27.6).

The runtime never commits a withdrawal whose `asset_id` Settlement would reject at
relay. Settlement also rejects a zero-amount withdrawal and a withdrawal to an
embedded-contract recipient, which the runtime commits and the relayer skips (§27.1a,
§17.3).

---

## SPEC v1.17.1 — 2026-10-17

### Bridge ledger (§6.6)

The `BridgeLedger` list includes `/fraud`, the bridge operator-collateral proof,
which shows that a pool operator spent collateral without authorization.

### Alignment with the implementation (§4.6, §5, §6, §9, §19, §21, §25, §27, §28)

An `input_kind` outside the eight §27.5 kinds yields `UNSUPPORTED_OPERATION`, and a
failure of the §11.3 deploy pipeline yields `VALIDATION_FAILED`. Neither halts the
runtime. A deploy finalize naming a record or chunk the runtime cannot read still
halts it (§4.6).

`SetPeripheryConfig` is a network-wide method with no `domainId` (§5.3).
`RegisterDomain` carries `proposerPolicy` and `finalityModel` after `minExecutors`,
and a reconfiguration rejects a change to `finalityModel`. Phase 1 registers only
domains at the base profile point, with no limit on their number (§5.3, §6).

A batch's stored state is always `SUBMITTED`. The batch is `FINALIZED` at every height
from its `finalizeAtHeight`, derived and never stored (§5.4, §19, §21, §27.11). Core
rejects a non-empty `proofData` (§19). The `assetFlowSummary` deposit check compares
each credit with the deposits recorded with the batch's assigned inputs (§19).

`chain_id` is derived once from the configured network name (§9.1). Settlement's
constants live in `vm/constants/`, and the executor defines the runtime limits
(§25.1).

The SMT maintenance adapter is `nodestore.go` (`NodeTree.Update`, `Commit`, `Prove`),
and L2 never reuses it (§28).

---

## EXECUTOR v0.12.0 — 2026-10-17

### Two phases (§1 to §5, §7, §9 to §14, §16 to §21)

The binary runs the executor role. The relay role is reserved for `L1_RELAYED`
domains. There is no watcher role, no fraud-proof harness (§5.8) and no
challenge-window rule, and §11 is unassigned so later section numbers stay stable.
Phase 1 has one executor under `SINGLE` (§13), and mainnet requires `ZK_VALIDITY`
(`../phase-2/`). A second SMT implementation is conformant only if it reproduces
the SMT vectors (§1.4). The supervisor publishes no bundles and opens no listener
reachable from outside the host; the gateway serves bundles over HTTP (§5.6, §21.5).

### Storage, configuration and input context (§7, §14, §16, §17, §21.8)

The state store holds its leaves in memory, and a snapshot is its only on-disk copy;
boot loads the snapshot at the cursor's position. `meta.json` holds `domainId` and
`executorId`, written once. Worker entries carry no `role`, and the supervisor
configuration has no `daServingEndpoint`. `InputContext` and the simulation context
carry the momentum hash beside its height and time.

### Alignment with the implementation (§1.4, §4, §5, §7, §9, §14 to §19, §21)

The configuration lists one worker per domain, and the supervisor does not compare
that list with Settlement's domain registry. A worker binary calls `worker.Main`.
The go-zenon imports include `vm/constants` and `dp`. `InputContext` carries the cap
lookup, `Source.Watch` takes a `context.Context`, and `Reader` has `ChainID`,
`DescribeInput` returning `any`, and `Profiles` (§1.4, §4, §7).

The ordinal entry names its send, the Settlement key builders are exported, and the
Settlement read path is built, so the prerequisite table keeps only the relay methods
(P-9). The WASM `Apply` serves input kinds 0 to 7, the storage quota ships at version
1, and the plugin charges effect gas. The plugin receives the canonical input
encoding and an `InputContext`. The loop reads Settlement's assignment and sends
`ReleaseWithdrawals`. Every go-zenon line reference names the `feature/settlement`
source (§5.1 to §5.5, §9, §18).

A rejected submission is retried with backoff, and an operator error follows after 5
consecutive rejections. The retry counts and intervals are fixed in the worker. A
snapshot is the cursor's only durable image: boot halts when it is missing, halts on
a snapshot tagged with another `stfSpecHash`, and a rollback skips that snapshot. The
`cursor` and `batches/` entries list their real fields. The supervisor configuration
adds `maxPendingBatches` and a `submissionKeySource` object. Worker entries have no
`inputSource` or `maxBatchInputs`. The gateway configuration adds `confirmationDepth`,
`uid` and `workerBinary`, and `maxConcurrent` defaults to 2. A binary halts on a
malformed field and on a missing field that has no stated default (§14 to §17).

The conformance text separates a malformed `ContractEffects` blob, which is
`RUNTIME_FAULT`, from a malformed input encoding, which halts. `zeno_getBatch`,
`zeno_getStateAt`, `zeno_getProof`, `zeno_read` and `zeno_simulate` take a tag, and
the list methods take `toBatch`. `zeno_getInputs` filters by `sender` and
`inputKind`. The `module` view takes `sourceCodeHash`, `contract` also returns
`ownerCaller`, `contractsByDeployer` pages by `cursor`, and a simulation result names
its momentum height, time and hash and its input index (§19, §21).

---

## SPEC v1.17.0 — 2026-10-16

### Two phases

The roadmap has two phases: Phase 1 is attestation on testnet, and Phase 2 brings
validity proofs. Mainnet requires `ZK_VALIDITY` (`../phase-2/`). Fraud proofs, the
watcher role, the `Challenge` method, the fraud referee, the challenge window, the
challenger payout and `ExecutorReregisterCooldown` are gone, and §22.5 is
unassigned. `OPTIMISTIC`, `CHALLENGE_WINDOW`, `FRAUD_REFEREE` and executor status
`COOLDOWN` stay as reserved ordinals that Core rejects (§27.11). The §32 item on
binding refunds to their depositors is dropped. A `MINTED` asset requires
`ZK_VALIDITY` (§6.5, §18.3). Under every `execProfile`, a `ReleaseAdapter` verifies
a proof of Zenon consensus and of Settlement storage showing the authorization in a
`FINALIZED` batch's `outboxRoot` (§6.6).

### SMT (§13.1, §13.2)

A Rust implementation of the executor-state SMT is conformant only if it reproduces
the SMT vectors. Whether the SMT keeps SHA3-256 or moves to a ZK-friendly hash is
settled before `StateRootSpork` reaches mainnet.

### `valueCaps` removed (§5.2, §5.3, §6, §23, §25.2, §30, §32)

`DomainRecord` and `RegisterDomain` carry no `valueCaps`, and `MaxBatchWithdrawal`
is the one per-batch withdrawal cap. A reconfiguration waits `AdministratorDelay`
only when it changes `minExecutors`.

### Momentum hash (§9, §27.1)

The execution context and the input frame carry `momentum_hash` after the momentum
time (`momentum_timestamp` in §9, `momentum_time` in the frame), so the frame header
is 254 bytes and `momentum_hash` starts at byte 146. All three momentum values
belong to the momentum that contains the input's send. The producing pillar chooses
that momentum, so its hash is grindable and must not be used as unbiased randomness.

### Data availability (§3, §20, §27.11, §29, §32)

The gateway serves bundles over HTTP; SPEC names no Sentinel or libp2p publication.

### Conformance artifact

go-executor's `execution-conformance-v1.json` carries `momentum_hash` on every
vector and the new `CV-FRAME-1`, which pins the frame offsets: digest
`694cd4e0…c49e30a`.

---

## SPEC v1.16.15 — 2026-10-12

### DA bundle size (§32, OD-5)

OD-5 gains item (e): no rule bounds a DA bundle's bytes except gas. Each observed key
adds a witness and a present witness carries its value, so a batch within
`MaxBatchInputs × MaxInputGas` can reach gigabytes, which the executor, DA
publication, watchers and gateways must all hold. A per-batch byte limit cannot close
it alone, because one input can exceed it; closing it needs a per-input witness-byte
charge in a later `metering_version`. Until then the gateway reads bundles up to a
1 GiB memory ceiling, not a protocol bound.

---

## SPEC v1.16.14 — 2026-10-10

### Contract inputs shared by every runtime (§5.3, §11)

The contract rows of §5.3 — `Deposit`'s delivery, `CallL2`, the deploy methods,
`UpgradeContract`, `TransferContractOwnership` and `BuyStorage` — are the input
vocabulary of every contract runtime. Settlement does not branch on `runtimeKind` when
accepting them, and a new runtime adds a `runtimeKind` and an STF, not methods. §11
separates Settlement's runtime-independent half of deployment from the WASM
validation pipeline, and lists what another runtime's pipeline must keep.

### Revert data (§7.5, §8.1, §10.3, §10.6, §14.3, §15, §27.1, §27.1a)

Every runtime reports a refusal as `REVERT` with an `abortCode` and revert data of at
most `MaxReturnData` bytes, carried as the `ExecutionResult`'s `ReturnData` and
committed through `returnHash`. The WASM host's `abort` becomes
`abort(code, data_ptr, data_len)`, charged `1 + 10 * ceil(data_len / 32)`; a
`data_len` over `MaxReturnData` is `RESULT_TOO_LARGE`. `wasm-gas-table-v1` and
`contract-sdk.md` D-8 follow: an SDK error carries its fields as revert data.

### ExecutionResult ReturnData bound (§27.1a)

The `ExecutionResult` grammar states `ReturnData ≤ MaxReturnData`, and a decoder rejects
a longer value. `ContractEffects.ReturnData` stays unbounded in the grammar, so an
over-long value still decodes and step 7 yields `RESULT_TOO_LARGE`.

### Conformance artifact

`execution-conformance-v1.json` and its README match go-executor E2-20 (revert data):
digest `c8a5ed07…bda49c6`.

### Native forms (§9.2, §12)

`caller` and `ContractID` stay the canonical 32-byte identities in every field this
specification defines. A runtime may expose a shorter native form of either to its
contracts, derived from the canonical identity alone, with a stated collision bound.
WASM exposes none.

---

## EXECUTOR v0.11.0 — 2026-10-10

### Gateway (§21)

A new `cmd/gateway` binary serves L2 data to clients through one `zeno_*` JSON-RPC API
for every domain, with `pending`/`latest`/`finalized` tags, receipts by L1 send hash,
proofs, events, outbox proofs and subscriptions. It rebuilds state by applying bundle
`StateDiff`s, never runs a plugin, and checks each batch against Settlement's
`DAHash`, `receiptRoot` and `postStateRoot`. It holds no key and only reads an
executor's storage. It runs `LOCAL` (beside an executor, serving `pending`) or
`REMOTE` (from other gateways, `latest` and `finalized` only).

### Supporting changes

- §1.5: the supervisor serves no client API; libp2p DA publication is its only
  externally reachable listener. HTTP bundle serving moves from the supervisor's DA
  layer (§5.6) to the gateway (§21.5).
- §7: an optional `Reader` interface, linked into the gateway only, declaring the
  domain's views. Domain-specific reads go through one generic method, `zeno_read`,
  so the API's method set is the same for every domain. Views are grouped into
  versioned profiles; `contracts/1` lets a client support any contract runtime
  without knowing which one. `zeno_getInputs` serves input history by contract and
  by L1 sender.
- §16: a `receipts/` sidecar per batch, since bundles carry no receipts; `batches/`,
  `bundles/` and `receipts/` become a versioned read interface.
- §17: gateway configuration. §19: gateway conformance. §20: the gateway ships on by
  default in Phase 1.
- §21.8: `zeno_simulate` runs a `CallL2` or `Deposit` against committed or pending
  state through the domain's worker binary in simulate mode, covering `eth_call` and
  `eth_estimateGas` for every runtime. §1.5 adds the simulate mode.
- `contracts/1` gains `nativeAddress`, `resolveNativeAddress`, `nativeCaller` and the
  `nativeAddresses` flag, so clients never derive a runtime's native form.

---

## Tickets — 2026-10-10

- E1-11 adds the `embedded.settlement` RPC namespace to go-zenon.
- E2-22 is the gateway. The watcher and libp2p DA publication move to E2-23; the
  watcher ticket's browser gateway moves to E2-22.
- E2-20 adds revert data. Withdrawals moves to E2-21, and every later ticket moves
  up by one.
- The gateway (E2-22) gains simulation.
- The system vault, E2E and conformance-coverage tickets move to E2-24, E2-25 and
  E2-26. The vault publishes its storage layout as an interface file. E2-25's flows
  assert outcomes through the gateway and the Settlement RPC.

---

## SPEC v1.16.13 — 2026-10-08

### Execution suite coverage (§31)

The artifact now carries the partial-claim deposit (`EXEC-DEP-2`, `EXEC-DEP-6`), the
omitted `claimed_deposit` (`CV-DECODE-5`) and every storage-quota case
(`CV-QUOTA-1`…`CV-QUOTA-9`), so they move to the carried list. The only case still
outstanding is a `Deposit` to an undeployed contract (E2-24).

### Conformance artifact

`execution-conformance-v1.json` and its README match go-executor E2-18 seams 1a to 1c:
150 rows, digest `e06cf0ee…554c89`.

---

## SPEC v1.16.12 — 2026-10-07

### Outbox branch selection (§27.4)

"Branch fields must match `kind` exactly" is restated as what it means for a decoder:
the branch is selected from `kind` alone, and bytes that do not continue as that branch
fail the decode. Any `kind` other than 0 or 1 is `RUNTIME_FAULT`.

---

## Tickets — 2026-10-07

E2-18 seam 1c gains the decode rows the seam 1b review found missing, a write at
exactly `MaxStateValueSize`, and a pure-move refactor of the vector generator. E2-24
gains the serialization-suite cases no contract can trigger: `ExecutionResult` decoder
rejections, the `MaxOutboxPerExecution + 2` bound and the `MerkleProof` rules.

---

## SPEC v1.16.11 — 2026-10-07

### Event and outbox input index (§27.1a)

Every `Event.input_index` and `OutboxMessage.input_index` in a returned blob must equal
the blob's `globalInputIndex`; a decoder rejects any other value (`RUNTIME_FAULT`). §16
already required it for events; outbox messages had no rule, and a foreign index could
collide with another input's `outboxId`.

---

## SPEC v1.16.10 — 2026-10-07

Rules the conformance coverage research found unstated or contradictory. Each now has
one reading, matching go-executor.

### Host reads (§8.1)

`storage_read` resolves an absent key, a too-small buffer, an out-of-bounds buffer and
a successful read in a fixed order. A key outside memory or above `MaxStateKeySize`
traps in both host functions.

### Deploy validation (§11.3)

A start section, a mutable global export, an export named `_start` and an export
beginning `__zenon_` are rejected at deploy. These were stated only in the contract
SDK notes.

### Upgrades (§12)

The upgrade reads and decision order are stated: no deployed contract, then
authorization before the §11.3 pipeline, then pipeline failure. A replacement with
the current `CodeHash` is `VALIDATION_FAILED`.

### Undeployed targets (§18.2)

A `CallL2` or `Deposit` reads the target's `0x0001` and `0x0002` leaves before the
deployed check, which fixes an undeployed target's witness set.

### Claimed deposit (§18.5)

§18.5 agrees with §27.1a: `claimed_deposit` is always in the blob, and a blob that
omits it is `RUNTIME_FAULT`.

### Withdrawal assets (§27.1a)

A contract withdrawal naming a non-canonical `AssetID` is `RUNTIME_FAULT` at decode.

### Tickets

New E2-18 (conformance vectors and batch roots) lands before submission, so
submission through E2E become E2-19 to E2-23. New E2-24 covers the remaining
conformance rows: module behaviour, the deploy lifecycle and validation, and
undeployed targets. E2-19 depends on E2-18's batch-root tests. E1-10, E2-16 and E2-17
are delivered. Delivered E1-08 and E1-10 keep their original references: their
"E2-19" is now E2-20 (withdrawals).

### Conformance artifact

`execution-conformance-v1.json` and its README now match go-executor `dev` (digest
`44fd0595…6757`): identities derived from vector IDs, rows sorted by batch and index,
and cap vectors CV-CAP-5 to CV-CAP-10.

---

## Tickets — 2026-10-06

E2-17 gains a fifth run, hardening: a 32 MiB bound on node response bodies, a
`PreviousHash` link for a send momentum older than the follower's window, one IPC round
trip for an input's cap lookups, and vectors `CV-CAP-5` to `CV-CAP-7` for per-asset
independence and the order of step 5a before step 6. No SPEC change.

---

## SPEC v1.16.9 / EXECUTOR v0.10.1 / BRIDGES — 2026-10-06

Input-path decisions from the E2-17 planning.

### Spork order (§5.1)

`OffchainExecutionSpork` must not activate before `StateRootSpork`, because the executor
reads Settlement's assignment through state proofs.

### Reading the assignment (§4.6)

A momentum with no Settlement account-block adds no input and needs no read. The
executor keeps the next expected ordinal per domain, and each momentum's ordinals must
start there. A Settlement account-block in a momentum without a state root is a
consistency fault. A send may precede the momentum confirming its receive by any
distance; it is found by hash in hash-checked content.

### Step 5a and the batch aggregate (§10.6, §27.9)

Only a non-zero total in an unconfigured asset faults. A batch builder may bound a
batch's per-asset aggregate by the highest cap step 5a used among its inputs.

### EXECUTOR

§5.1 details the assignment reads and withdrawal caps. §15 adds the assignment
consistency fault (halt) and the withdrawal-cap outage (restart, not halt).

### Tickets

E2-17 is delivered in four runs (decode, assignment reader, follower model, withdrawal
caps) with the decisions above. E2-22 records the spork order.

---

## SPEC v1.16.8 / EXECUTOR v0.10.0 / BRIDGES — 2026-10-06

Settlement client decisions from the E2-16 research.

### The executor reads Settlement's assignment (§4.6, §5.4)

The executor no longer re-evaluates Settlement's acceptance predicate. Each
`(domainId, ordinal)` entry also carries the send's `source_account_block_hash`. For
each confirmed momentum the executor proves its domains' received-input counts and the
new ordinal entries, and takes each input from the send the entry names. Rejected sends
leave no entry, so the numberings coincide by construction. Reading at a momentum needs
the node to retain its state; re-syncing from genesis needs an archive node.

### Step 5a cap reference point and unconfigured assets (§10.6, §27.9)

Step 5a reads `MaxBatchWithdrawal` from Settlement state at the end of the momentum that
confirms the input's receive. An asset with no configured cap has cap `0`, so step 5a
faults a withdrawal of it instead of producing one Settlement would refuse to register.

### EXECUTOR

- §1.4: `wallet` and `common/crypto` are allowed imports; no `rpc/*` package is.
- §5.1: the input pipeline reads the assignment; the Settlement ABI is complete.
- §5.5: proof-key layout and verification, recovery anchor (end of the finalized
  prefix), seed key format and signing, own-activity check, `SubmitBatch` guard and
  confirmation, withdrawal relay and release cadence, submission log.
- §17: `settlementAddress` removed (it is `types.SettlementContract`); key format; node
  requirements.
- §18: P-1 and P-4 built; P-6 refreshed; P-8 (ordinal entry names its send) added.

### Tickets

E2-16 is split by landing order into E2-16 (L1 read path), E2-17 (input path), E2-18
(submission) and E2-19 (withdrawals). The watcher, system vault and E2E tickets become
E2-20, E2-21 and E2-22. New E1-10 (Settlement client surface) adds the send hash to the
ordinal entry, exports the executor-facing key builders, and tests a Settlement proof.

---

## Epics — Settlement tickets renumbered — 2026-10-05

No normative change. Settlement tickets are numbered `E1-NN` in go-zenon landing
order, one squashed commit each on `feature/settlement`.

| New | Replaces |
|---|---|
| E1-01 Settlement scaffold, spork, address, storage layout | 08, 09 |
| E1-02 Core/Periphery, admin, pause | 10, plus the executor-bond key and L2 gas-budget bound that had no ticket |
| E1-03 … E1-09 | 11 … 17 |

Cross-references in both epics and `epics/README.md` use the new numbers.

---

## SPEC v1.16.7 / EXECUTOR v0.9.9 / BRIDGES — 2026-10-04

Withdrawal relay and release rules from the ticket 16 planning. EXECUTOR and BRIDGES
change only their upstream pins.

### Inclusion proof encoding (§27.3, §25.1)

`MerkleProof` is a `u32` step count followed by `(side, sibling)` steps from the leaf
up, for every §27 root. A promoted odd node contributes no step. Verifiers reject more
than `MaxMerkleProofDepth = 32` steps, any `side` other than 0 or 1, and trailing
bytes, and hash the leaf from the supplied element.

### Relay rejections (§17.3)

`RelayMessage` rejects a `kind = 0` message, a zero-amount withdrawal and a withdrawal
to an embedded-contract address, recording nothing. A release to an embedded address
would fail its descendant send and roll back the whole `ReleaseWithdrawals` call.

### Refunds bounded by the batch's deposits (§27.9, §5.4)

Registered refunds per (batch, asset) may not exceed the batch's `depositCredit` for
the asset. Without the bound, an executor could label withdrawals as refunds and
bypass `MaxBatchWithdrawal`. Settlement keeps per-(domain, batch, asset) totals for
contract withdrawals and for refunds.

### Release interval and empty queue (§18.2, §25.1)

`ReleaseWithdrawals` is rate-limited per domain by `ReleaseIntervalMomentums = 1`, not
`UpdateMinNumMomentums`. A batch can carry 2048 withdrawals; at 16 per 300 momentums a
batch of dust withdrawals delayed every later exit on its domain by about 107 hours, and
at one call per momentum it clears in 128 momentums. A call on an empty queue fails with
`ErrNothingToWithdraw` and does not consume the rate limit. A domain's first release
always passes.

### Loss bound of a fraudulent batch (§18.3, §27.9, §32)

The bound is `MaxBatchWithdrawal + depositCredit` per asset, because a refund is not
tied to its depositor. Binding refunds to their deposits is a Phase 2 item in §32.

### Conformance text (§31)

`CV-OUTBOX-1` in the serialization suite is a decode rejection, not `RUNTIME_FAULT`.

### Tickets

Ticket 16 carries all four rules. E2-16 builds the executor's `MerkleProof` builder
and adds a proof vector to the serialization suite.

---

## Epics — executor tickets renumbered — 2026-10-04

No normative change. Executor tickets are numbered `E2-NN` in landing order, and
each delivered ticket is one file describing its delivered scope. The sub-epic
folders and their sub-tickets are removed. Settlement tickets keep `08`–`17`.

| New | Replaces |
|---|---|
| E2-01, E2-02, E2-03 | 01, 02, 03 |
| E2-04 Deterministic WASM runtime plugin | 04 and sub-tickets 04a–04g |
| E2-05 WASM conformance suite + determinism harness | 04h and sub-tickets 04h1–04h6, plus the receipt-encoding, result-blob and gas-budget bounds, gas-table drift check and serialization-suite work that had no ticket |
| E2-06, E2-07, E2-08 | 05, 06, 07 |
| E2-09 | 07a |
| E2-10 | 07c |
| E2-11 Unpriced-work benchmark + limits | 07b, 07d |
| E2-12 State store scaling | 07e, 07f, 07i |
| E2-13 | 07h |
| E2-14 Storage quota, `BuyStorage`, version fields | 07g, 07j |
| E2-15 | 07k |
| E2-16, E2-17, E2-18, E2-19 | 18, 19, 20, 21 |

Cross-references in the Epic 1 tickets and `WASM Spec/wasm-gas-table-v2-draft.md`
use the new numbers. `epics/README.md` replaces the single global sequence with
per-epic numbering and a cross-epic dependency table.

---

## EXECUTOR v0.9.8 — 2026-10-04

No normative change to `SPEC.md`. The WASM plugin's node-local store keys a pending deploy record by `(deployer, code_hash)` and each deploy chunk by `(deployer, code_hash, chunk_index)`, where the deployer is the input's `L1Sender`. A `DeployContractStart` clears only its own deployer's earlier attempt, and a finalize or upgrade reads only its own sender's record and chunks. An upgrade therefore needs the new code staged by the upgrading caller (`SPEC.md` §12).

### Reorg paragraph and failure table (§15)

The halt on a pending record whose deployer differs from the finalize's sender is removed, along with its failure-table row: another deployer's record is unreachable, so the condition cannot occur. The one unrepaired path is a same-deployer second start across the expiry boundary, which is not detected. The operator remedy is unchanged.

### Tickets

Ticket 07k is delivered. `BRIDGES.md` and `epics/README.md` pin EXECUTOR v0.9.8.
Ticket 18's pre-submit guard reads the domain cursor and submits only
`batchId == nextBatchId`, and its `SettlementFilter` keys deploy records by
`(deployer, code_hash)`.

---

## Epics — 2026-10-04

No normative change. The delivery-status table in `epics/README.md` now shows
tickets 07a–07i and 11–15 as delivered, and lists 07c–07f and the
`Store.Accessor` snapshot ticket. The deploy-records-keyed-by-deployer ticket is
renumbered from 07i to 07k, since 07i already names the `Store.Accessor` snapshot
ticket.

---

## SPEC v1.16.6 / EXECUTOR v0.9.7 / BRIDGES — 2026-10-03

Batch-acceptance rules from the ticket 15 planning. EXECUTOR and BRIDGES change only
their upstream pins.

### Inputs by per-domain ordinal (§4.6, §5.4)

Every accepted input gets a per-domain ordinal, and its assigned-index record is
keyed by `(domainId, ordinal)`, carrying its `globalInputIndex`, asset, amount and
fee. A batch's inputs are read by key, at most `MaxBatchInputs` lookups, with no scan
of the global index range.

### Assigned-index record lifecycle (§4.6, §25.1)

Each accepted `SubmitBatch` deletes the records of up to
`MaxRecordSweepBatchesPerSubmit = 2` of the domain's finalized batches. Records stay
until their batch is final, so a Phase-2 replacement batch can still read them.

### `SubmitBatch` checks (§5.4, §19, §24)

- The cursor is `{consumedInputCount, nextBatchId, recordsSweptThrough}`; no index
  value is reserved as an "empty" sentinel. Closes the §32 cursor-sentinel item.
- `batchId` must equal `nextBatchId` (0, then +1), so no batch can be overwritten.
- A domain's first batch must start from its genesis root; for WASM that is the
  zero hash.
- All nine version fields must equal their Phase-1 values, which are all `1`.

### Version fields are all 1 at activation (§13.3, §15, §24, §31)

Nothing is deployed before the activation spork, so a format change made before it
updates the format in place and bumps no version: every version field is `1` at
activation. This replaces `statediff_version = 2` (storage quota) and
`receipt_version = 3` (`STORAGE_QUOTA_EXCEEDED`), and resets the earlier
`receipt_version = 2`. The reserved-key and status-code lists still require a
version increment for any change after activation.

### Quota leaf read only when `delta ≠ 0` (§13.8, §31)

Step 6a reads the quota leaf only when the input's net state growth `delta` is not
`0`. A call whose writes net to zero (including a read-only call) leaves the leaf out
of its witness set. `CV-QUOTA-*` gains a vector for it.

### Tickets

Ticket 15 carries the ordinal change to `admitInput`, the cursor, the `batchId`,
genesis-root and version checks, and the record sweep. Ticket 07g keeps its version
constants at `1` and delivers new ticket 07j with it: resetting go-executor's
`ReceiptVersion` from `2` to `1`, so the conformance suites regenerate once. 07g also
copies the regenerated suites into zeno's `WASM Spec/`.

---

## SPEC v1.16.5 / EXECUTOR v0.9.6 / BRIDGES — 2026-10-02

Stops an unfinalized deployment from leaving permanent L1 state. EXECUTOR and
BRIDGES change only their upstream pins; no acceptance outcome changes, so the
executor needs no change.

### Deploy record storage and expiry sweep (§5.4, §11.1–§11.3, §11.5, §12, §25.1)

- The pending record is a fixed-size header with a received-chunk count. Start no
  longer reserves 18 chunk-hash slots for the flat Start plasma; each chunk writes
  its own entry under `(domainId, deployer, code_hash, startHeight, chunk_index)`,
  paid for by that chunk's size-priced plasma.
- Finalize and upgrade require the received count to equal `chunk_count`, and delete
  the header and every chunk entry.
- A FIFO start queue drives an expiry sweep: each `DeployContractStart` removes at
  most `MaxDeploySweepPerStart = 2` expired records and their chunk entries before
  writing its own, so a sender of new starts pays to clear old ones.
- A restarted deployment carries a new start height, so it never reads, or is
  swept with, an earlier attempt's entries.

### Tickets

Ticket 14 carries the header-only record, per-chunk entries, the start queue and the
sweep; ticket 17's upgrade checks the received count and deletes on consumption.

---

## SPEC v1.16.4 / EXECUTOR v0.9.5 / BRIDGES — 2026-10-02

Two fixes from the ticket 14 critique. EXECUTOR and BRIDGES change only their
upstream pins; the executor's code change is ticket 07i.

### Settlement send plasma (§10.1)

A send to Settlement requires `max(method plasma, 21000 + 68 × data bytes)`. An
embedded method's flat `GetPlasma` value otherwise replaces the per-byte charge, which
made a full 15 KiB deploy chunk about 20 times cheaper than a plain send of the same
bytes. Small calls keep their flat price; a full chunk or a large `SubmitBatch` now
needs fused plasma, as §10.1 already stated.

### Deploy records keyed by deployer (§5.4, §11, §12)

Pending deployment records are keyed by `(domainId, deployer, code_hash)`. Chunk,
finalize and upgrade look up the caller's own record. Anyone who learned a
`code_hash` could otherwise start a record for it and block the real deployer for
`DeploymentExpiryWindow` heights, repeatedly. Different deployers may now stage the
same `code_hash` at once.

### Tickets

Ticket 14 implements the plasma rule and the new key; ticket 17's upgrade consumes
the caller's own record; tickets 18 and 21 give the executor and deployers fused
plasma; new ticket 07i re-keys the WASM plugin's node-local deploy store.

---

## SPEC v1.16.3 / EXECUTOR v0.9.4 / BRIDGES — 2026-10-02

Defines the height every Settlement receive-time check uses, and settles the
ticket 14 deployment details. EXECUTOR and BRIDGES change only their upstream pins.

### Receive height (§4.6, §10.7, §11, §19, §27.9)

A receive's height-dependent checks and records — deployment start and expiry, the
fee price in force, a batch's `submittedAtHeight`, the `MaxBatchWithdrawal` in
force — use the receive height: the receive block's `MomentumAcknowledged` height,
which is what `context.GetFrontierMomentum()` returns inside the receive. The SPEC
previously named the momentum containing the receive, which the receive cannot
read. The executor must take the same height from the receive block, not from the
momentum containing the send.

### Deployment (§10.7, §11.1, §11.5)

- A record is expired at receive height `h` when `h ≥ startHeight +
  DeploymentExpiryWindow`, so it is live for exactly that many heights.
- `total_size` must be at least 1.
- Until a fee price is set, every fee-carrying call is rejected; a zero fee is never
  accepted.

### Tickets

Ticket 14 records the deployment decisions; ticket 15 records `submittedAtHeight`
as the receive height; ticket 18's real `SettlementFilter` takes heights from each
send's receive block.

---

## SPEC v1.16.2 / EXECUTOR v0.9.3 / BRIDGES — 2026-10-01

Closes gaps the ticket 12 and 13 planning surfaced. EXECUTOR and BRIDGES change only
their upstream pins.

### `submittedAtHeight` is Core's acceptance height (§19, §22.4)

The wire commitment's `submittedAtHeight` must be `0`, and Core rejects any other
value. Core stores the height of the momentum containing the `SubmitBatch` receive
and derives `finalizeAtHeight` from it, so an executor cannot shorten the withdrawal
delay, and the fee-epoch queue's height order is guaranteed. go-executor already
sends `0`.

### Per-domain rate limit for `ReleaseWithdrawals` (§18.2)

The withdrawal crank keeps its rate limit in its own record keyed by `domainId`; the
shared `checkAndPerformUpdate` record belongs to the fee crank `Update`.

### Executor member record (§5.4)

The Phase-2 member fields live under their own key prefix keyed by pillar name, not
on the member record.

### Input envelope and paused domains (§4.1, §6, §18.2, §23)

- Every input method is a typed ABI method with `domainId uint32` first; the ABI is
  transport only, and the canonical input encoding (§27.5) is unchanged. Closes the
  §32 envelope item. §4.1 now lists all eight input kinds.
- A domain whose `status` is `PAUSED` rejects every input method at the receive.
  Accepted inputs, batches and withdrawals continue under the two pause scopes.
- A `Deposit` carrying no value is rejected at send validation.

### Tickets

Ticket 12 adds both new `batchEntry` fields, `executorPillar` and
`finalizeAtHeight`; ticket 15 sets them, rejects a non-zero wire
`submittedAtHeight`, and passes the batch's assigned-index records to the fee hook;
ticket 16 keeps the per-domain rate-limit record and takes the end-to-end
EXEC-DEP-3 refund test; tickets 13, 14 and 17 use the typed envelope and reject
paused domains; ticket 13 rejects zero-value deposits and tests EXEC-DEP-3's
on-chain preconditions.

---

## SPEC v1.16.1 / EXECUTOR v0.9.2 / BRIDGES — 2026-10-01

Settles the fee-settlement and domain-registration questions tickets 11 and 12
raised. EXECUTOR and BRIDGES change only their upstream pins.

### Fee settlement (§5.4, §22.4)

- The fee epoch is height-based: `E = submittedAtHeight / MomentumsPerEpoch`.
- Settlement walks a queue of epochs that hold accruals, oldest first, instead of
  every epoch in turn, so quiet periods and the gap since genesis cost nothing.
- Each queued epoch records the highest finalization height of its batches;
  `Update` settles the head once the frontier reaches it.
- When `Update` cannot settle — head not due, queue empty, paused, or guard — it
  succeeds without settling and consumes its rate limit. `CollectReward` fails with
  an error in the same cases.
- A pillar not active at settlement has its pending amount burned.
- No per-epoch burn total is stored, and §5.4 now lists the four fee counters.

### Batch finality (§19, §21)

Each batch stores a `finalizeAtHeight`, fixed at acceptance from the domain's
withdrawal delay then in force, so a later `SetDomainConfig` change never moves an
accepted batch's finality.

### Domain reconfiguration (§6)

A reconfiguration identical to the stored record and delays is rejected, as is one
carrying delays different from the stored ones.

### Tickets

Ticket 12 carries the queue, the height-based epoch, the `executorPillar` field
change, the `ReleaseWithdrawals` rename of the `Update(domainId)` stub, the §27.11
status encoding and the fee-price bound placeholders. Ticket 15 stores
`finalizeAtHeight`; ticket 16 records the rename as done; ticket 11 rejects identical
and delay-changing reconfigurations.

---

## SPEC v1.16.0 / EXECUTOR v0.9.1 / ARCHITECTURE / BRIDGES — 2026-10-01

Moves Periphery to the scopes the domain registry makes possible, and settles the
`RegisterDomain` questions ticket 11 raised.

### 1. Periphery scopes (§5.2, §5.3, §5.4, §20, §23)

- `SetPeripheryConfig` keeps only the network-wide fee prices.
- New `SetDomainConfig(domainId, withdrawalDelay, runtimeUpgradeDelay)`,
  `SetAssetConfig(domainId, assetId, maxBatchWithdrawal)` and
  `SetApprovedDAModes(domainId, modes)`, each at `SoftDelay`.
- A time challenge is keyed by method and scope, so a pending change for one domain
  or asset is never restarted by a change for another.
- The approved `DAMode` set is per domain, replaced whole, and must be non-empty;
  `SubmitBatch` checks a batch against its own domain's set.
- An asset is custodyable for a domain once its `MaxBatchWithdrawal` is configured;
  the check runs at the receive, since it needs storage.
- Closes the §32 per-domain Periphery and approved-`DAMode` items.

### 2. `RegisterDomain` (§6)

- Domain ids are sequential from `1`. The caller passes the id: the next unassigned
  id registers, a lower id reconfigures, a higher id is rejected.
- A reconfiguration rejects any change to a field fixed at registration.
- Registration takes the initial withdrawal delay and runtime-upgrade delay and
  validates them, with no time challenge.
- `profileConfig` must be empty in Phase 1.
- `valueCaps` is empty (no caps) or starts with a big-endian `u64` window, validated
  against the domain's withdrawal delay; Phase 1 enforces no cap.

### 3. No-lowering rule (§23, §27.9)

A (domain, asset)'s `MaxBatchWithdrawal` cannot be lowered for as long as the domain
is registered, whatever its `status`, so pausing a domain does not lift the rule.

### Tickets

Ticket 11 carries the new setters, the id and reconfiguration rules, and removes
`executors` from `domainRecord`. Tickets 13 and 17 move the custodyable check to the
receive; ticket 15 checks `DAMode` per domain; tickets 18 and 21 read and set the
per-scope values.

---

## EXECUTOR v0.9.0 / BRIDGES — 2026-09-28

Splits the executor binary into a supervisor process and one worker process per
domain (`SPEC.md` §6.2). No Phase-1 byte changes: no byte format, wire version
field or conformance artifact moves.

### 1. Process split and restart classes (§1.5, §5.1, §14, §15)

- §1.5: the supervisor serves confirmed momenta to every worker on request
  rather than forwarding them once. A worker must not hold key material or be
  able to read it; the supervisor withholds the key, its source and its
  location from every worker, and runs each worker as its own user where the
  host allows it.
- §1.5: the supervisor tells a worker's exit apart by its status. A halted
  worker or one that rejects its configuration is never restarted. A worker
  whose source the supervisor could not serve, or whose domain directory
  another process holds, restarts with bounded backoff, uncounted. Any other
  end is a failure, restarted with bounded backoff, and a domain that fails
  repeatedly without progress is reported `HALTED`.
- §5.1: the input-pipeline description matches the §1.5 serve-on-request
  wording.
- §14: boot acquires the domain lock (§16) instead of the single-instance
  lease, which now belongs to the supervisor alone. Transient faults restart
  through the supervisor; a consistency fault exits with the worker's halted
  status and is never restarted, and a failed local write outside the batch
  commit halts too.
- §15: the crash-mid-batch, disk-full/OOM and znnd-unavailable rows name the
  supervisor's restart, and a new row covers a malformed worker message. The
  worker's znnd retry stays a fixed interval; the supervisor's restart delay
  grows instead, because it spaces the restarts of a worker whose source
  stays unavailable and such an outage never halts the domain.

### 2. Storage (§15, §16)

- §15: the reorg-rollback remedy names the domain directory
  (`<storagePath>/<domainId>/`), which holds the plugin's store under
  `wasm-code/`, instead of the old top-level `wasm-code` path.
- §16: a worker holds an exclusive lock on its directory (`worker.lock`) for
  its lifetime, so two workers never write one domain. The layout tree gains
  `wasm-code/` and `worker.lock`.

### 3. Configuration and release (§17, versions)

- §17: a binary requires every field it consumes and rejects any field it
  does not consume as unknown. The supervisor identity line carries
  `networkName` instead of `chainId`; each plugin derives `chain_id` from it
  (`SPEC.md` §9.1). A new `workers` line points at the per-domain worker
  block, which gains `binary`, the worker executable linked with that
  domain's plugin. The block states what the supervisor hands each worker
  over the worker channel, and that a worker never receives
  `submissionKeySource`.
- `EXECUTOR.md` becomes v0.9.0. `BRIDGES.md` and `epics/README.md` pin the
  new version.

---

## SPEC v1.15.1 / EXECUTOR v0.8.1 / ARCHITECTURE / BRIDGES — 2026-09-25

Adds an incremental tree to the path-native SMT API, so the executor's root and
proof cost follows the keys an input touches rather than total domain state. No
Phase-1 byte changes: no byte format, wire version field or conformance artifact
moves.

### 1. Incremental path-native tree (§13.1)

- §13.1's path-native API adds `NewCompactTree` and `CompactTree`'s `Insert`,
  `Delete`, `Root` and `ProveByPath`.
- The shared-core scope names `CompactTree` as the incremental form of the core: it
  must reproduce `compute.go`'s roots and proofs byte for byte, gated by
  `smt-v1-test-vectors.json`.
- The tree hashes no key and never folds a zero-length value into a deletion.
- ARCHITECTURE §5 names the tree alongside the path-native functions.

### 2. Executor state store (EXECUTOR §1.3, §5.3, §18, §19)

- The `common/trie` upstream row (§1.3) and §5.3 list the tree among the path-native
  primitives the store uses, and §5.3 no longer names `CompactTree` as a forbidden
  entry point. §19's conformance grep drops it.
- §5.3's and §18's `common/trie` line citations are corrected.
- `BRIDGES.md` and the epics index pin SPEC v1.15.1 and EXECUTOR v0.8.1.

---

## SPEC v1.15.0 / EXECUTOR v0.8.0 / ARCHITECTURE / BRIDGES — 2026-09-24

Replaces the bonded, per-domain executor with pillar-operated executors shared by
every domain, and pays them from developer fees instead of a bond or emissions.
Phase 1 becomes testnet-only. Four Phase-1 byte changes: a new input kind, a new
receipt status, a new reserved state key, and the stored batch entry's submitter
field.

### 1. Executors are pillars (§6.2, §19, §22, §23)

- One network executor set, held by Settlement and shared by every domain.
  `DomainRecord` loses `executors`; every member serves every active domain and
  cannot opt out, so random selection gives each pillar an equal chance.
- A member is a pillar name plus an executor address the pillar chooses, so the
  producer key stays out of executor software. `SubmitBatch` re-checks that the
  pillar is active.
- `RegisterExecutor(pillarName, executorAddress)` is admin-only and time-challenged
  in Phase 1; in Phase 2 a pillar registers itself from its producer address.
- The bond is removed: `DepositQsr`, `WithdrawQsr`, `PostBond`, `RevokeBond`, the
  bond storage and the bond parameters are gone. The `MINTED` floor keeps its
  `MaxBatchMint` cap and relies on the `OPTIMISTIC` fraud proof.
- The stored batch entry carries `executorPillar` instead of `executorIndex`, which
  closes the §32 stored-entry attribution item.
- Phase 2 (reserved): missed turns under `RANDOM_BACKUP` lapse a member after
  `ExecutorLapseWindow`; missed turns on a new domain are not recorded during
  `DomainActivationGrace`; a successful fraud proof moves the guilty pillar's
  unsettled income to the challenger and blocks re-registration for
  `ExecutorReregisterCooldown`.

### 2. Developer fees and storage quota (§10.6, §10.7, §13.3, §13.8, §15, §27.5)

- `CallL2` and `Deposit` stay free apart from plasma.
- `DeployContractFinalize` and `UpgradeContract` carry a ZNN bytecode fee of
  `total_size × znnPerByteOfBytecode`.
- New method and input kind `BuyStorage` (`input_kind = 7`) buys state-byte quota
  for any contract at `znnPerByteOfState`.
- The runtime keeps quota in reserved key `0x0005` (`statediff_version = 2`) and
  fails over-quota writes with `STORAGE_QUOTA_EXCEEDED = 8` (`receipt_version = 3`)
  at new step 6a. New constant `StateLeafOverhead = 64`.
- Fees sit outside custody and `AssetFlowSummary`, and are never refunded.
- Closes the §32 deploy-time pricing item's mechanism; the price value stays
  pre-mainnet.

### 3. Fee settlement (§5.3, §22.4, §23, §27.10)

- Fees accrue to `(epoch, pillar)` at batch acceptance and settle per epoch once all
  of that epoch's batches are `FINALIZED`: `FeeBurnShare = 30` percent burned, the
  rest credited to the pillar's `RewardWithdrawAddress`.
- `Update` becomes the no-argument fee crank that pillars already send to embedded
  contracts; the withdrawal crank is renamed `ReleaseWithdrawals(domainId)`.
- New `CollectReward` pays from Settlement's balance and never mints.
- An always-on guard bounds settlement by collection, and `PAUSE_RELAY` on any
  domain halts settlement and collection without changing any amount.

### 4. Testnet-only Phase 1 (§1, §2.2)

The Phase-1 profile point is not activated on mainnet; mainnet waits for an
execution profile that verifies batches on-chain.

### 5. Executor process isolation (EXECUTOR §1.4, §1.5, §4, §16, §17)

The binary is a supervisor plus one worker process per domain. Workers hold no key
material, and a worker failure stops only its own domain. A release ships a worker
for every adopted domain.

### Tickets

Ticket 12 is rewritten as executor registration + fee settlement, including the
rework of Ticket 09's delivered bond storage on `epic/settlement`. Tickets 11, 13,
14, 15, 16, 17, 18 and 21 carry the renames and fee checks. New tickets 07g
(storage quota) and 07h (worker processes).

### Not updated

`phase-2/SPEC.md`, `phase-2/ADDING-A-RUNTIME.md`, `BRING-YOUR-OWN-RUNTIME.md` and the
`research/` and `reference/` notes still describe bonded, per-domain executors.

---

## EXECUTOR v0.7.2 / BRIDGES — 2026-09-23

### Stale-deployer halt now precedes validation (§15)

The stale-pending-record halt compared the deployer only on the path that passed record
validation, so a stale record that would otherwise fail validation still committed
`VALIDATION_FAILED` instead of halting. The deployer comparison now runs before the
validation-failure branch, so any stale record halts regardless of whether it would pass. The
remaining residual is a second start from the same deployer, which the halt cannot distinguish
from a legitimate one. `BRIDGES.md` now pins `EXECUTOR.md` v0.7.2.

---

## EXECUTOR v0.7.1 / BRIDGES — 2026-09-23

### Stale pending deploy record (§15)

The unrepaired reorg path's stale pending record also supplies `upgrade_policy` and `owner`,
which would let the second start's sender make itself owner of the canonical deployment. The
runtime now halts a finalize whose record names a deployer other than the finalize's sender,
and §15's failure table lists the halt. The remedy stays an operator re-sync, and closing the
path still needs a plugin rollback seam. `BRIDGES.md` now pins `EXECUTOR.md` v0.7.1.

---

## SPEC v1.14.0 / EXECUTOR v0.7.0 / BRIDGES — 2026-09-23

Closes every open design except `OD-4` and `OD-5`, which now carry a measurement rule,
including the bridge designs: input completeness, verifier dispatch and proof of non-delivery. Three changes affect Phase-1 bytes: the
contract metadata transport, a seventh input kind, and witness values in the bundle.

### 1. Contract metadata and ownership (§11.1, §11.3, §12, §15, §27.5, §27.12; closes OD-3)

`DeployContractStart` gains a `metadata` argument, the §27.12 encoding
`metadata_version:u16 ‖ upgrade_policy:u8 ‖ owner:Address(20)`. Settlement checks it against
`metadata_hash` and `MaxContractMetadataSize` at the receive. The runtime validates it at
finalize and writes the policy and owner leaves, so `OWNER_UPGRADEABLE` and `UpgradeContract`
become reachable. The kind-2 input tail carries the bytes.

An upgrade replaces code only. An upgrade of a contract with no code is `VALIDATION_FAILED`.
Ownership moves only through the new `TransferContractOwnership`, `input_kind = 6`; a zero new
owner renounces ownership and makes the contract `IMMUTABLE`.

### 2. Witness values in the bundle (§20; closes OD-13)

`WitnessEntry` carries the value it proves, so a replayer can re-execute a batch from DA alone.
No `damode_version` increment accompanies it, since no production bundle exists.

### 3. `MaxBatchWithdrawal` never falls while a domain is active (§23, §27.9; closes OD-14)

Core rejects a lower value while any domain is `ACTIVE`, so every input step 5a admitted stays
batchable and every finalized withdrawal stays registrable. Pause is the lever for reducing
at-risk value.

### 4. `EXTERNAL_OBSERVED` is dropped (§4.5, §6.1, §6.5, §27.11)

Every input is an L1 account-block. A source too large to relay raw sends a `ZK_CONSENSUS` proof
or a `COMMITTEE` signature as an `L1_RELAYED` input. Value `2` of `inputSource` is retired, and
§6.5's rules renumber. This closes the framework's completeness open design.

### 5. Bridge mechanisms (§5.3, §5.5, §17.4, §18.3, §18.4a, §19, §27.6a; closes OD-1)

All three are reserved and change no Phase-1 behaviour.

- **Verifier registry (new §5.5).** Verifiers are built into the node and activated by spork.
  The Periphery registry holds write-once `VerifierEntry` records (`verifierId`, verifying
  key), added by `RegisterVerifyingKey` under `AdministratorDelay`. Repointing a domain is a
  `stfSpecHash` change, and the dispatch call and its public inputs are fixed per verifier kind.
- **`MINTED` reconciliation (new §18.4a).** A `protocol_version = 2` commitment carries a
  `MintFlowSummary` of `(asset, mintCredit, burnDebit)`, applied to the counters at
  finalization. `MaxBatchMint` bounds `mintCredit`. Under `COMMITTEE`/`ZK_CONSENSUS`, Core
  verifies foreign locks at `RelayForeignLock` and reconciles against them. A third invariant
  bounds a `MINTED` asset's L1 circulation by its live supply.
- **Proof of non-delivery (§17.4).** A Zenon destination is checked against Settlement's own
  processed set through `TimeoutMessage`. A foreign destination is proved by a non-membership
  proof against the `ReleaseAdapter`'s committed processed-set root at a block past the
  deadline.

### 6. Pricing (§30, §32)

`OD-5` loses its deploy-time item, which moves to §32 as a pre-mainnet task for the
executor-economics design. `OD-4` and `OD-5` gain a resolution rule: an operation gets a
`metering_version = 2` term if, at its bound, it costs more than twice the time the least
triggering gas buys.

### 7. Companion and executor documents

- `EXECUTOR.md` v0.7.0 drops `EXTERNAL_OBSERVED`, updates the code-store record check, and
  replaces a stale `OD-6` reference with the committed `abortCode`.
- New `BRIDGES.md`: the guide to building a bridge or messaging channel on this model. It covers
  architecture, choosing profiles and custody, the per-chain components, lifecycles, worked
  examples, security, testing and deployment, plus the three points SPEC does not yet fix for
  bridges. SPEC, `EXECUTOR.md`, `ARCHITECTURE.md` and `DEMOS.md` cite it.
- The bridge framework companion is removed. Its core model is normative in SPEC (§5–§6,
  §17–§18), and its examples, testing and deployment material is in `BRIDGES.md`.
- `AMENDMENTS.md` is removed. Each change's reasoning lives in its entry here, and the
  reasoning behind the bytes Settlement must match lives in epic 1 tickets 15 and 17.

### 8. `MINTED` always needs the `OPTIMISTIC` floor (§6.5, §18.3)

§6.5 rule 3 accepted a Core-verifiable `foreignProfile` in place of `execProfile ≥ OPTIMISTIC`,
while §18.3 required both (a) and (b). §18.3 governs: Core verification covers the mint side
only, and burns and per-account balances stay STF state that only a fraud proof can police.
Rule 3 now states the floor, and §18.3(c) states what Core verification does and does not add.

---

## SPEC v1.13.0 / EXECUTOR v0.6.0 — 2026-09-22

Consolidates the decisions epic 2 tickets 06 and 07 fixed in go-executor (the bundle's deploy-chunk
bytes, the per-input withdrawal bound, the old-`CodeHash` retention window, and `internal/recovery`'s
**[BINARY]** decisions). One bundle-layout change, one new STF
rule, one open design closed and two opened.

### 1. The bundle carries deploy-chunk bytes (§11.2, §20; closes OD-7)

`BundleInputRecord` gains a trailing `deployChunkData` field, present only on kind-3 records, so a
DA-only replayer can reassemble every deployed module. `DAHash` changes for any batch containing a
chunk input. No `damode_version` increment accompanies it: an old-layout kind-3 record would
mis-frame under the new decoder with nothing to detect it, but no such bundle has ever been
produced and the serialization suite carries none.

### 2. Per-input withdrawal bound (§10.6 step 5a, §27.9, §31)

A new ordered validation step between the outbox-capacity check and the effect charge. An input
whose contract-emitted withdrawals exceed `MaxBatchWithdrawal` for any asset, or whose withdrawal
assets plus its deposit asset exceed `MaxAssetsPerBatch`, settles `RUNTIME_FAULT` with metered
gas. Without it, one input can be unbatchable and stall the domain. It changes a committed status,
so a divergent implementation forks.

The deposit amount is excluded from the per-asset total, as the refund already is from the batch
aggregate, so the step never caps a deposit. The deposit's asset counts toward the asset
footprint. The value read is the one in force at the input's momentum. §31 gains the required
vector case. go-executor's shipped step 5a counts the deposit amount and must drop it.

### 3. Two open designs (§30; OD-13, OD-14)

- **OD-13** — a `WitnessEntry` carries a proof, not the value it proves, so a replayer cannot
  recover an input's reads from DA alone. Gating for Phase 2 DA-only replay.
- **OD-14** — the runtime reads `MaxBatchWithdrawal` at the input's momentum while Settlement
  enforces it at withdrawal registration. Lowering it in between can stall a domain or block a
  finalized batch's withdrawals. Non-gating for Phase 1; §32's per-domain scoping bullet now
  carries it.

### 4. Old `CodeHash` retention names its window (§12)

§12's "historical state" now points to `EXECUTOR.md` §16's bundle retention, since state keeps
only current leaves. No behaviour change.

### 5. Executor persistence and recovery (EXECUTOR §14–§16)

- **Bundle retention (§16):** Phase 1 retains every bundle; garbage collection waits for the
  Phase-2 challenge window and never collects before `FINALIZED` plus that window.
- **Retry schedules (§14, §15):** own-bundle `AWAIT_DA` defaults to 5 attempts 200 ms apart; the
  znnd retry defaults to 5 attempts at a fixed 1 s interval, replacing "with backoff". Both are
  overridable and always end in `HALTED`.
- **Reorg rollback (§15):** the rollback covers the cursor, state store and recovery tree, not a
  plugin's node-local store. The one unrepaired path is a pending deploy record rewritten on a
  retracted branch; the remedy is an operator re-sync.
- **Upstream pin:** `SPEC.md` v1.13.0.

---

## SPEC v1.12.0 — 2026-09-08

Closes the last five gating open designs and the five non-gating ABI details. Four are the
decisions the WASM runtime already implements, written down; two change committed behaviour and
carry a version increment.

### 1. `gas_used` per status, and the ordered post-execution validation (new §10.6; closes OD-9)

`gasUsed` is committed in every receipt, and §10.5 required only that gas accounting be
deterministic. Two implementations could agree on that and still commit different values for the
same input. New §10.6 fixes the nine ordered steps that run after `run` returns, and gives a
per-status table covering every reachable condition.

The effect charge is step 6: it follows every content check and precedes both size checks. Two
conditions now yield `RESULT_TOO_LARGE` with different gas — an over-long blob rejected at step 0
commits metered gas alone, because step 6 has not run, while a size rejection at step 7 or 8
commits metered plus the effect charge.

### 2. Structural validation caps (§25.1, §11.3; closes OD-8)

§25.1 gains nine caps on counts that are LEB128-encoded and bounded by nothing else:
`MaxLocalsPerFunction` 8,192; `MaxTotalLocals` 2^20; `MaxTotalParams` 2^16; `MaxTypeCount` 1,024;
`MaxFunctions` 8,192; `MaxGlobals` 1,024; `MaxDataSegments` 1,024; `MaxTableEntries` 10,000;
`MaxTotalElementEntries` 16,384. Each drives allocation or table writes at every instantiation —
on every call, for the life of the contract. §11.3 requires validation to reject a module
exceeding any of them.

### 3. Deploy-time segment bounds (§11.3; closes OD-10)

Core 1.0 defers the check that an active segment's `offset + length` fits its target to
instantiation. §11.3 now requires it at deploy, so a module that would fail on its first call is
rejected when it is deployed. Every module holding a `CodeHash` is instantiable, which is what
makes an instantiation failure at call time a node-local fault rather than a receipt status.

### 4. The abort code is committed (§7.5, §15, §27.7, §24; closes OD-6)

§7.5 required the abort code be "recorded" and no committed structure had a field for it. The
receipt gains `abortCode`, an `i32` — the one signed primitive in this specification. For
`abort(code)` it is that code; for a `run` returning `r < 0` it is `(-r)` truncated to `i32`.
Every non-`REVERT` receipt carries `0`.

It is committed because a `REVERT` receipt otherwise carries nothing distinguishing one refusal
from another: the abort path returns no blob, and §14.3 discards `ReturnData`, so `returnHash` is
zero. `receipt_version` becomes `2`.

### 5. The deposit refund sits outside the contract's outbox cap (§27.4; closes OD-11(d))

`MaxOutboxPerExecution` bounds what a **contract** emits. The runtime-emitted refund is appended
after that count, as §27.9 already exempts it from the `MaxBatchWithdrawal` aggregate and §14.3
already exempts it from the failure discard. A contract emitting the maximum alongside an
unclaimed remainder keeps both its messages and the depositor's refund.

An `ExecutionResult`'s outbox list is therefore bounded at `MaxOutboxPerExecution + 1`; a
`ContractEffects` blob's stays at `MaxOutboxPerExecution`. A decoder distinguishes the two.

### 6. Four ABI details (§8.1, §27.1, §11.3, §13.3; closes OD-11(a), (b), (c), (e))

- The host imports live under the module string `"env"`; any other module string, a duplicate
  pair, or a fourth import is rejected.
- A module whose declared initial memory is smaller than the input frame yields `RUNTIME_FAULT`
  with the initial-memory charge only. The runtime does not grow memory to fit the frame.
- Custom sections, including the name section, are rejected at the split, before any section is
  parsed. `CodeHash` is a function of the module's semantics alone.
- The reserved `local_key` set is exactly `0x0001` `CodeHash`, `0x0002` `code_hash`, `0x0003`
  upgrade policy, `0x0004` owner, matched exactly rather than by prefix. A fifth needs a
  `statediff_version` increment.

### 7. Conformance coverage §31 now owes

Four cases beyond the three already listed: an over-long blob at step 0, including one that is
also malformed; a contract at the outbox cap with an unclaimed remainder; a `REVERT` receipt with
a non-zero `abortCode`; and a non-`REVERT` receipt carrying zero.

`receipt_version = 2` re-dates every vector's `receipt_hex`, so these land with the regeneration.

---

## SPEC v1.11.0 — 2026-09-08

Resolves three open design items. Two carried a **MUST**-resolve gate; the third is narrowed
from an exposure to a pricing question. Two new consensus constants, `MaxResultBlob` and
`MaxInputGas`, and one behaviour the executor implements today is now forbidden rather than
undefined.

### 1. Index assignment is on acceptance, not decodability (§4.6, §11.1–§11.3, §12; closes OD-2)

§4.6 assigned a `globalInputIndex` to every account-block whose `{domainId, payload}` envelope
decoded, while §11.1 phrased its duplicate-record rule as "fail `VALIDATION_FAILED`" — receipt
language for a call that, under the decodability reading, had already become an input. The two
readings gave different `inputRoot`, receipt sets and index numbering for identical on-chain
activity.

Acceptance now governs. An index is assigned if and only if the addressed method accepts the
call: the envelope decodes for a registered domain **and** every receive-time check passes. A
rejected call — undecodable envelope, unregistered domain, duplicate, absent, expired or
already-consumed deployment record — is not an input, consumes no index, produces no receipt and
contributes no `inputRoot` leaf. §11.1, §11.2, §11.3 and §12 state their checks as receive-time
rejections rather than receipt statuses.

Two consequences are stated normatively. Every input the executor is asked to execute has
already passed its method's checks, so a runtime observing an input whose on-chain precondition
does not hold differs from the network rather than the protocol and halts rather than committing
a status. And an `input_kind` outside the six §5.3 kinds is unreachable from a live Settlement,
so `UNSUPPORTED_OPERATION` and the unrecognized-kind input encoding are defensive.

This is the reading go-executor already implements throughout its halt/status classification for
input kinds 2–5, so no executor behaviour changes. Ticket 13's input-admission helper is restated
to match.

### 2. `plasma_limit` is bounded, and clamping is forbidden (§10.1, §25.1, §25.2; closes OD-12)

`plasma_limit` was a `u64` mapped one-for-one to gas with no ceiling, so one input could
authorize unbounded executor work. No section stated a width for the gas ledger, and clamping,
rejecting and pre-execution `OUT_OF_GAS` were all consistent with the text while committing
different `gasUsed` for the same input.

`MaxInputGas` (§25.1) now bounds it, and Settlement rejects an over-budget `CallL2` or `Deposit`
at send validation with `ErrForbiddenParam` — so such a send never becomes an input at all, which
is why no receipt status is needed for it. `MaxInputGas` **MUST** sit below `2^63 - 1`, so every
conformant budget fits a signed 64-bit ledger and no clamping or saturating behaviour is defined
or permitted. A runtime observing a larger budget halts.

The value is pre-mainnet (§25.2) and is sized from the batch rather than the input:
`MaxBatchInputs × MaxInputGas` bounds a worst-case batch. It also bounds the per-input execution
trace a Phase 3 validity proof must cover.

### 3. The returned blob is bounded before decode (§14.3, §25.1; narrows OD-4)

A contract could return a blob bounded only by declared memory (4 MiB) against a committed result
bounded at 32 KiB. Zero-initialized WASM memory supplies minimum-size entries for free, so the
blob decoded to hundreds of thousands of entries and every one had its SMT key derived before any
size check could reject it, while metered gas reflected only the fixed initial-memory charge.

`MaxResultBlob` (§25.1, 64 KiB) now bounds it, checked before decode, ahead of every other check
on the blob's contents. An over-long blob is `RESULT_TOO_LARGE` whether or not it is also
malformed — the status §14.3 already assigns to both other size violations.

OD-4 narrows to pricing: the decode and per-entry derivation still carry no gas-table term, but
the work they can be made to do is now bounded by a consensus constant rather than by declared
memory. Whether a term is warranted is a measurement question against the initial-memory charge,
and adding one is a `metering_version` increment that re-dates every `gas_used` in the execution
conformance suite.

---

## SPEC v1.10.0 / EXECUTOR v0.5.0 / ARCHITECTURE — 2026-09-01

Writes down the administration model as specified across `SPEC.md`, `EXECUTOR.md` and `ARCHITECTURE.md` — §27.11's encodings are shipped, but `RegisterExecutor` and the `RegisterDomain` delay this entry documents are not yet built (`EXECUTOR.md` §5.5). **No Phase-1 execution behaviour changes**: no constant value moves and no shipped encoding changes. §27.11 writes down encodings the code already fixed rather than choosing new ones. No Go source was touched.

### 1. Administration model (§5.2, §23, §25.2, ARCHITECTURE §2.1, EXECUTOR §6)

- Settlement's administrator is the single global network administrator address
  (`types.NetworkAdmin`), compared directly on the sending address. Settlement stores
  no administrator, exposes no method that changes one, and holds no guardian set:
  `ChangeAdministrator`, guardian recovery, `SecurityInfo` and the separate
  emergency-pause authority are gone from the Settlement administration model. (This
  pass swept `SPEC.md`, `EXECUTOR.md` and `ARCHITECTURE.md` — not `BTC-BRIDGE-RESEARCH.md`, whose `:40`/`:210`
  `TimeChallenge`/`SecurityInfo` references describe go-zenon's separate bridge
  contract machinery, not Settlement, and correctly stand.)
- Only the Core hard bounds are genuinely independent of key custody. The
  time-locks are not: `PAUSE_RELAY` takes effect with zero delay and halts
  `RelayMessage` registration and `Update` release, so a compromised key can
  suspend the time-locked exit window itself for as long as the key is held
  (§23; `BRIDGES.md` §8.2). The guardian threat model no longer
  applies because there is no per-contract key to compromise or recover.
- The two time-lock windows are fixed Core constants (`MinSoftDelay`,
  `MinAdministratorDelay`), not administrator-set values with floors:
  `SetPeripheryConfig` at `SoftDelay`; the escape hatch and `RegisterExecutor` at
  `AdministratorDelay`, both custody-consequential under the Phase-1
  `ATTESTATION`/`SINGLE` profile. A `RegisterDomain` call that changes `valueCaps`
  or `minExecutors` on an already-registered domain likewise takes effect only
  after `AdministratorDelay`, since those fields bound the same custody path;
  initial registration of a not-yet-active domain carries no prior depositors
  and is exempt. Emergency pause is the sole carve-out and takes
  effect immediately, with two independently settable scopes (`PAUSE_SUBMIT`,
  `PAUSE_RELAY`) at global and per-domain granularity. The authority check is
  enforced at receive time, before any state write.
- Consolidating the network's other administrator-gated contracts onto that address
  is intended and has not shipped; bridge, liquidity and the spork contract still
  gate on their own separate addresses. Rotation is a node release plus an
  activation spork, and **MUST** select the authority by height so pre-rotation
  administrator calls still re-execute to the same result.
- go-zenon has no embedded-contract event primitive, so "**MUST** emit a public L1
  event" is restated as "**MUST** record a durable, publicly readable on-chain
  record", naming the record. Additive if a primitive later lands.

### 2. Activation, ABI grounding and recovery anchor (§5.1, EXECUTOR §1.4, §5.5, §14)

- §5.1 names the shipped registration surface: the contract address
  `z1qxemdeddedxsettlementxxxxxxxxxxqtmqrx9` (`types.SettlementContract`), added to
  `EmbeddedContracts` via `parseEmbedded`; and the spork symbol
  `types.OffchainExecutionSpork` with its placeholder hash `…0004` and its
  `ImplementedSporksMap` entry.
- §5.1 states pre-activation behaviour normatively: before the spork activates, the
  Settlement address resolves to no contract, so calls fail with
  `constants.ErrContractDoesntExist` — distinguishable from a method-level rejection.
- `EXECUTOR.md` §1.4's import list names the ABI the binary depends on:
  `ABISettlement` (`vm/embedded/definition/settlement.go`).
- `EXECUTOR.md` §5.5 records the storage-decode convention: every Settlement storage
  entity except processed-outbox membership is ABI-packed and decoded with
  `ABISettlement.UnpackVariable`; processed-outbox membership is the literal byte
  `0x01` under its key, not an ABI blob.
- `EXECUTOR.md` §14's recovery anchor no longer asserts an on-chain signed `-1`
  sentinel for "nothing consumed": the on-chain representation is a Settlement concern
  (§32), and the executor's own genesis cursor is what the fallback describes.

### 3. Method list and Periphery surface (§5.2, §5.3, EXECUTOR §5.1, §5.5, §18)

- §5.3 gains `SetPeripheryConfig` and `InitiateEscapeHatch`, taking the enumerated
  method count from 16 to 18; `Pause`'s origin is corrected to `Admin`.
- Periphery is restated as a record of numeric scalars carrying **no address
  fields**, and now includes the runtime-upgrade delay. Core rejects out-of-bounds
  values with `ErrForbiddenParam` rather than silently clamping them.
- `EXECUTOR.md`'s prerequisite table gains a **Status** column across its five
  surviving rows — P-5 and P-7 stay retired (v1.9.0) and are not reintroduced: P-1
  grades 4 of 18 methods dispatching in go-zenon, P-2 and P-3 built in go-zenon, P-4
  the ABI's 18 storage variables and 4 method entries in go-zenon, P-6 the read path
  partial in go-zenon against the documented key layout. §5.1, §5.5 and §6's
  build-status bullets are regraded to match, and the config block no longer marks
  the Settlement address as a prerequisite.

### 4. Storage responsibilities and enum encodings (§5.4, §6, §19, new §27.11)

- The batch-chain bullet names the stored entry's fields rather than deferring to
  §19's full list: `protocolVersion` — the one of §24's nine version fields the
  stored entry keeps — the commitment's remaining fields including `proofData`, the
  `u8` `executorIndex` in place of the wire commitment's `executorId`, and the
  batch's state. The assigned-index record is split from the per-(domain, asset)
  cumulative deposit total. The release queue's per-domain head/tail cursor pair is
  named as what makes release strictly FIFO. New bullets record global pause state,
  per-domain pause state and the per-domain escape-hatch record; the reserved
  verifier registries gain their keying.
- New **§27.11** fixes the numeric encoding of every enumerated field — twelve rows,
  each ladder numbered in its own section's presentation order, so a later rung
  appends rather than renumbers. §6 cross-references it, and `profileConfig` /
  `chainBinding` / `valueCaps` are stated as opaque byte strings whose inner
  encodings remain open.

### 5. Constants status (§25.1, §25.2)

- No numeric constant value changes. §25.1 gains the shipped names:
  `MaxInstrumentedCodeSize`, and the runtime-upgrade floor as the relative predicate
  `IsRuntimeUpgradeDelayValid(upgradeDelay, withdrawalDelay)` rather than an integer
  constant.
- §25.2 names the bound holders (`WithdrawalDelayMin`/`Max`,
  `MaxBatchWithdrawalFloor`/`Ceiling`, `MaxBatchMintFloor`/`Ceiling`), replaces the
  administrator row with the network administrator address (a build-time value that
  **MUST** be confirmed in the released binary before mainnet), and deletes the
  emergency-pause-authority row.
- The `valueCaps` window row is deliberately unchanged and carries no predicate name:
  its floor is stated relative to the withdrawal delay in force, which the shipped
  fixed-constant predicate does not implement. That gap is recorded in §32.

### 6. Companions and open/deferred items (§28, §30, §32, EXECUTOR §18, ARCHITECTURE §3, epics)

- §30 drops the Settlement storage key layout (shipped and collision-tested in
  `vm/embedded/definition/settlement.go`) and its duplicate of the escape-hatch item,
  and gains the `profileConfig` / `chainBinding` / `valueCaps` inner encodings and the
  storage home for the Periphery-approved `DAMode` set. §32's escape-hatch bullet is
  narrowed to the forced-withdrawal mechanics and the triggering-condition record.
- §32 gains twelve forward-looking bullets for gaps found against the shipped tree —
  the cursor sentinel, executor-bond keying, `Update` domain keying, `Update` send
  validation, the `MaxBatchMint` bounds predicate, per-domain Periphery scoping, the
  reserved `AssetRegistry` schema, the `DAMode` storage home, contract-originated
  administrator sends, administrator-address rotation and replay, the `valueCaps`
  window floor, and the stored entry's executor attribution — a stored
  `executorIndex` alone does not identify the submitting address once
  `RegisterExecutor` replaces a set member. Most name their point of no return; the
  two migration-scoped bullets name the migration they belong to. No normative
  requirement was weakened to match the code.
- §28's template list is corrected (`RegisterSentinelMethod` for the bond,
  `TimeChallenge` for time-locking administrator changes) and the spork symbol pinned.
  `EXECUTOR.md` §18's grounding citations for the registration pattern convert to
  symbols; `ARCHITECTURE.md` §3 pins the registration list to shipped symbols and
  corrects the bond/time-window paragraph (the withdrawal delay is a Periphery scalar
  Core bounds, not a time-challenge).
- Epic ticket 10 and epic ticket 21's pre-mainnet parameter line are brought in line with
  §23. Ticket 10's time-lock clause adds `RegisterExecutor` to `AdministratorDelay`.

### 7. The `executorId` / `executorIndex` field split (C1)

Two distinct fields carry the submitting executor. The wire batch commitment (§19)
keeps `executorId : Address(20)` unchanged, with
`CV-COMMIT-1` standing — no encoding moves. The stored batch entry (§19, §5.4)
instead carries `executorIndex : u8`, the position of the submitting executor in the
domain's `executors` set, which Core resolves at `SubmitBatch` from the verified
caller address against the set as it stands when the batch is accepted. Of the wire
commitment's nine version fields (§24), the stored entry keeps only `protocolVersion`;
the other eight are not stored. go-zenon's shipped ABI and Go struct still spell the
storage field `executorId`; no encoding depends on that name, and the rename is
documentation-only in this tree.

### 8. Version choice

`EXECUTOR.md` skips v0.4.0, which go-zenon's tree already uses for a different
document. The bump across all four documents is minor, not major: no
consensus-visible byte moves and no shipped encoding changes. §9.2's version-block
enumeration replaces a placeholder with the encoding `CV-COMMIT-1` already fixes, and
the `executorIndex` naming is a spec-text rename for a field whose bytes are
positional — it changes no wire byte and invalidates no vector.

---

## SPEC v1.9.0 / EXECUTOR v0.3.3 — 2026-08-22

Closes six normative gaps surfaced by the WASM plugin's conformance/determinism round (epic 2
ticket 04's `04h1`–`04h6` sub-tickets). Four of the six record a choice the shipped, vector-frozen
plugin already makes rather than restating a forced reading, and are flagged individually below.
Two further gaps are genuinely open design questions and are recorded in §30 instead. One further
edit carries no item number: §11.1's `metadata_hash` clause gains a cross-reference to `OD-3`,
adding no requirement and clarifying an existing one.

### 19. `stateDiffHash`/`returnHash` receipt preimages (§15, §27.7)

**Records a choice.** §13.2 fixes `SHA3-256` and §27.7 fixes the field width, but neither states a
preimage. §15 now fixes both: `stateDiffHash` is `SHA3-256` of the canonical `List<StateDiffEntry>`
encoding of `StateDiff` (`bytes32(0)` when empty); `returnHash` is `SHA3-256(ReturnData)` with no
length prefix (`bytes32(0)` when empty). Both are `bytes32(0)` on every non-`SUCCESS` receipt. This
is what the shipped plugin already computes, corroborated against the four literals
`execution-conformance-v1-README.md` names for it, each of which the two equally grammatical
alternatives (a count-less concatenation, a §27.3 Merkle root) fail to reproduce; `go-zenon` is still asked, as A18, to state both
preimages normatively.

### 20. Non-`SUCCESS` `ReturnData` discard (§14.3)

**Amends a rule.** §14.3's prior `MAY` left `ReturnData` retention on a failure path to
implementation discretion; item 19's `returnHash` makes that retention consensus-visible, since two
implementations that both discard `StateDiff`/`Events`/`OutboxMessages` can still commit different
`receiptRoot`s if one retains a decoded `ReturnData` blob the other discards. Tightened to `MUST`,
matching the discard rule the other three fields already carry.

### 21. Canonical input encoding for an unrecognized `input_kind` (§27.5)

**Records a choice.** §27.5 enumerated a kind-specific tail for the six Phase-1 kinds only, though
§15 and §27.7 require a receipt — with a mandatory `inputHash` and `inputRoot` leaf — unconditional
on the kind being recognized. New fallback bullet: for any other `input_kind`, the canonical
encoding is the `input_kind:u8 || source_account_block_hash:bytes32` prefix alone, with no tail.
Dropping the undecodable tail loses no binding, since `source_account_block_hash` already
identifies the exact account-block. This is what the shipped plugin already assembles for
`EXEC-UNSUP-1`; `go-zenon` is still asked, as
A19, to state the encoding normatively.

### 22. `UNSUPPORTED_OPERATION` trigger (§15)

New sentence on the Status-codes paragraph: `UNSUPPORTED_OPERATION` is the status of an input whose
`input_kind` is outside the six defined in §27.5, with `gasUsed = 0`, zero `contractId`/`codeHash`,
and no outbox message or refund. `UNSUPPORTED_OPERATION` already has a Settlement-level trigger at
§18.1 (a non-canonical `AssetID` on a kind 0/1 input); this adds the execution-layer trigger without
claiming it is the status's only one. Whether and how the two triggers interact is unresolved and
falls under `OD-2`.

### 23. `plasma_limit`:gas ratio for `CallL2`/`Deposit` (§10.1)

New bullet: the L2 gas budget for a `CallL2` or `Deposit` input **MUST** be derived from
`plasma_limit` one-for-one, with no scaling factor. Behaviour for a `plasma_limit` beyond an
implementation's metering ledger's representable range is left open, routed to new `OD-12`.

### 24. Reserved `local_key` matching rule (§13.3)

**Records a choice.** "Prefix" was ambiguous between exact-match reservation and reservation of an
entire keyspace region. Fixed as exact match: a `local_key` is reserved only against the fixed set
of protocol-reserved values, never by prefix. The concrete byte values are runtime-specific and are
routed to new `OD-11`(e) rather than fixed normatively.

### `SPEC.md` §30

New `OD-12` (`plasma_limit` has no upper bound, and no behaviour is defined beyond a gas ledger's
representable range; non-gating for Phase-1 single-executor operation, gating for mainnet
operation). `OD-11` gains sub-item (e) (the reserved `local_key` byte values), moving its count
from four to five. `OD-8`'s candidate structural-cap set is named inside its existing parenthetical;
the item stays gating.

### §31 conformance-artifact reconciliation

The superseded artifact-status banner is retired. §31 now names three suites by file: the SMT/trie
suite (`common/trie/testdata/smt-v1-test-vectors.json`, `proof-format.md`), the execution suite
(`WASM Spec/execution-conformance-v1.json`, `WASM Spec/execution-conformance-v1-README.md`), and the
serialization suite (`WASM Spec/serialization-conformance-v1.json`). `CV-DEPLOY-1` relocates to the
execution suite. The duplicate `CV-OUTBOX-1` id — real, not an error, since it is a cross-suite
collision between the codec-layer and runtime-layer rejections of the same shape — is explained
across both suites' entries. Execution coverage is delegated to the README's own coverage table
rather than restated inline; three required-but-uncovered execution cases are stated: a payable
`CallL2` with a real deposit and a partial claim plus the §18.4 batch-level identity, `claimed_deposit`
omitted ⇒ `RUNTIME_FAULT`, and a `Deposit` to an undeployed `ContractID` ⇒ `VALIDATION_FAILED`.
`SPEC.md`'s Conventions intro companion-artifact list (`SPEC.md:26`) is extended to name
`execution-conformance-v1-README.md` and `serialization-conformance-v1.json`, aligning it with
§31's own text.

### Companion sync

`EXECUTOR.md` → v0.3.3 (upstream pin → SPEC v1.9.0; document status now also grounds against
go-executor source; the STF driver's per-input interleave and witness-set construction are stated
normatively; the deterministic WASM runtime and the node-local code store move from
`[PREREQUISITE]`/informal mention to `[EXISTS]`; the `Domain` interface gains `InputContext`; the
`Effects` field set is stated explicitly; two new §15 failure rows for the code store and the
deploy/upgrade pipeline; `P-5`/`P-7` retired from the `[PREREQUISITE]` table; §19's preconditions
drop the `P-7` marker and gain the execution-conformance suite).
`ARCHITECTURE.md` unversioned, four corrections: the SMT stage's witness-set sentence corrected to
match `SPEC.md` §20; the code store described as unnumbered prose beside the STF stage; the deploy
flow block's validation parenthetical extended with §11.3/§30's structural and segment-offset
checks; the Phase-1 `IMMUTABLE`-only clause appended to the deploy/upgrade flow's owner-policy
sentence.
`WASM Spec/serialization-conformance-v1.json` added as a byte-identical snapshot of
`internal/codec/testdata/serialization-conformance-v1.json`, holding `CV-COMMIT-1/2`,
`CV-INPUTROOT-1`, `CV-ASSET-1/2`, `CV-OUTBOX-1`, `CV-BUNDLE-1`; adoption into `go-zenon` is still
owed.

### Source

Items 19–24 are implemented across `plugins/wasm/*.go` and `plugins/wasm/internal/**`
(go-executor), and frozen by the 48-vector `execution-conformance-v1.json`. A18 and A19 remain
outstanding requests on `go-zenon`.

### Known follow-ups

A18 and A19 are unratified upstream: a different upstream answer on either invalidates both the
mirror text and every `receipt_hex` in the execution suite. `OD-2`, `OD-4`, `OD-8`, and `OD-9`
remain gating; `OD-12` is gating for mainnet, with its ledger-range half gating on any second
implementation of this specification's WASM domain. Neither vendored `serialization-conformance-v1.json` nor
`execution-conformance-v1.json` has an automated equality check against its source copy
(`execution-conformance-v1-README.md:5`). The vendored `smt-v1-test-vectors.json` is not in the
same position: `SPEC.md:1346` already states it predates `CV-PATH-1/2`/`CV-APPLY-1/2`, stops at
SMT-014, and is not the suite of record — a known divergence, not an unchecked equality claim.

---

## SPEC v1.8.0 / EXECUTOR v0.3.2 — 2026-08-08

Closes thirteen normative gaps in the WASM entry-point ABI, gas accounting, the deploy/upgrade
pipeline, and the receipt shape, surfaced while implementing and reviewing the deterministic
WASM runtime plugin (epic 2 ticket 04, all eight sub-tickets — complete, tested, reviewed).
Ten are pure clarifications: every gap closed here already had exactly one implementation
choice consistent with the surrounding text, and this makes that choice explicit. Three
(Principle 3's memoization-store carve-out; the deploy/upgrade `ReturnData` byte layout;
receipt `contractId`/`codeHash` per input kind) amend a rule rather than merely restate it —
each is called out individually below, since a literal reading of the prior text would
produce different behaviour or different bytes than the shipped plugin. A further eleven items surfaced during the same
review are genuinely open design questions, not clarifications, and are recorded as new `SPEC.md`
§30 entries `OD-2` through `OD-11` instead of applied here — four of them (`OD-2`, `OD-4`, `OD-8`,
`OD-9`) are marked gating.

### 6–7. `ContractID` preimage: `Address` enumeration and explicit width (§12, §14.1, §27.0)

§27.0's `Address(20)` enumeration named `caller` preimages, withdrawal recipients, and
`executorId` but not §12's `deployer_address`, a fourth use the document itself requires. Both
occurrences of the enumeration (§14.1, §27.0) now list it, and §12's derivation line now states
the width and source explicitly: `deployer_address(20B)`, the raw L1 address that submitted
`DeployContractFinalize`.

### 8. Linear-memory initialization vs. Core 1.0 data segments (§7.3, §27.1)

§7.3 required memory "zero-initialized before contract entry" and §27.1 said "all memory above
`frame_len` is zero" — both false for any module with an active data segment, which Core 1.0
(mandated by §7.1) permits and every real toolchain emits. Both sections now state that memory
is zero except where the module's own active data segments and the input frame write it, and
§27.1 now states the frame is written **after** segment initialization and overwrites any
segment content it overlaps.

### 9. `stfSpecHash`'s "VM version" (§6)

Clarified to mean the concrete execution-engine implementation and its exact version, not the
WebAssembly specification level (already pinned separately by §7.1) — the engine's own type
checker decides which modules validate, so only the concrete-engine reading makes `stfSpecHash`
do the job §24 already assigns it.

### 10. `gasUsed` for the four deploy/upgrade input kinds (§10.1)

New bullet: `DeployContractStart`/`Chunk`/`Finalize`/`UpgradeContract` carry no `plasma_limit`
and so are not subject to L2 gas accounting under `metering_version = 1`; their receipts
**MUST** carry `gasUsed = 0`.

### 11. `key_len > MaxStateKeySize` in a host call (§27.8)

§13.3's bound had no stated consequence, and §27.8's host return-value set had no spare
encoding for it. Now explicit: the host function **MUST** trap (canonicalizing to
`RUNTIME_FAULT`, §7.5), never via a return value, and the key is never witnessed.

### 12. `RESULT_TOO_LARGE` for a `MaxReturnData` violation (§14.3)

The prior "else `RESULT_TOO_LARGE`" attached grammatically to the `MaxExecutionResult` clause
only, leaving a `MaxReturnData`-only violation (the two bounds are not co-extensive) with no
status. Now explicit: violating either bound yields `RESULT_TOO_LARGE`.

### 13. Witness-set determinism must be cache-independent (§20)

New sentence: the set of observed keys **MUST** be a deterministic function of the input and
pre-state alone, so a warm-cache node and a cold-cache node witness identically. This is the
condition under which §20's existing "exactly one valid encoding" claim for `DAHash` is actually
true.

### 14. A domain plugin may use a node-local memoization store (`EXECUTOR.md` §2, Principle 3)

**Amends a rule.** Principle 3 forbade all environment access; deploy finalization is `Apply`
and must persist up to 1 MiB of assembled/instrumented bytecode somewhere for later calls to
reuse. Principle 3 now permits a plugin to read from and write to a node-local,
content-addressed store as a pure memoization tier, under conditions (every read
hash-verified against state or input; a miss or verification failure is a halt, never a status;
every write a pure function of state and input; nothing ever observable in `Effects`, a root, or
the witness set) that keep the store's presence unable to change what `Apply` returns, only how
fast it runs. Unlike the other items in this revision, this is a genuinely new permission, not a
restatement of the only prior-consistent reading. Instrumented bytecode could instead have lived
in SMT state under reserved keys, which would have needed no amendment; the local store was
chosen, shipped and reviewed, and Principle 3 now permits it.

### 15. Deploy/upgrade pipeline failures are always `VALIDATION_FAILED`, never a halt (§11.3)

§12 already stated this for the upgrade path; §11.3's deploy-side text attached
`VALIDATION_FAILED` to the assembly hash/length check only and fell silent on validation and
instrumentation failures. Now symmetric: any pipeline-stage failure is `VALIDATION_FAILED` with
no contract created, and **MUST NOT** be treated as a node-local fault, since the pipeline is a
pure function of the assembled bytes.

### 16. Event/outbox attribution must match the executing contract (§27.1a)

New validation-list bullet: every `Event.contract_id` and `OutboxMessage.source_contract` in a
returned blob **MUST** equal the executing `contract_id`, else `RUNTIME_FAULT`. Closes a forgery
gap where a contract could attribute an event to another contract, or — since a runtime-emitted
deposit refund is marked only by `source_contract = bytes32(0)` — emit a message
indistinguishable from a refund into the committed `outboxRoot`.

### 17. `ReturnData` byte layout for deploy/upgrade code hashes (§11.3, §12)

**Amends a rule.** Both sections required `ReturnData` to carry a pair of 32-byte hashes but
never stated the concatenation order, which is consensus-visible via `returnHash`. Now explicit:
`code_hash || CodeHash` on deploy, `oldCodeHash || newCodeHash` on upgrade — the layout the
shipped plugin already uses. Recorded as a choice, since a different order would have been
equally grammatical.

### 18. Receipt `contractId`/`codeHash` per input kind (§15)

**Amends a rule.** The prior parentheticals ("or zero if no contract target" / "or zero if none
executed"), read literally, would zero both fields on every deploy/upgrade receipt — a
`DeployContractFinalize` names no target and executes no code. Replaced with an explicit
per-kind table: zero on kinds 2/3; the newly derived `ContractID`/`CodeHash` on a successful
kind-4 finalize, zero on a failed one; the input's `contract_id` on every kind-5 outcome, with
`codeHash` tracking the contract's current code identity. Recorded as a rule change, since the
literal prior text supported a different, less useful reading than the one adopted: a receipt
that says nothing about the contract it just created, while §11.3 requires the result to carry
both hashes.

### `SPEC.md` §30: eleven new open-design entries

`OD-2` (Settlement rejection vs. `globalInputIndex` assignment, gating) through `OD-11` (four
smaller WASM ABI/validation ratification items). Four are marked gating: `OD-2`, `OD-4` (WASM
returned-blob decode/derivation pricing), `OD-8` (WASM structural validation caps beyond
`MaxCodeSize`/memory/stack — a validation-divergence fork risk), and `OD-9` (`gas_used` per
status and its ordering against the effect charge). See `SPEC.md` §30 for full text.

### Companion sync

`EXECUTOR.md` → v0.3.2 (Principle 3 amended per item 14 above; upstream pin → SPEC v1.8.0).
`ARCHITECTURE.md` unchanged — none of this revision's WASM-specific detail is restated there.

### Source

Implemented across `plugins/wasm/*.go` and `plugins/wasm/internal/**` (go-executor).

### Known follow-ups

The eleven `OD-n` entries in §30 are unresolved by design — this revision's job was closing the
gaps that had a forced answer, not the ones that don't. `OD-2`, `OD-4`, `OD-8`, and `OD-9` gate
correctness (index assignment, gas pricing on a reachable amplification, validation divergence,
and committed `gas_used`, respectively) and should be prioritized accordingly.

---

## SPEC v1.7.0 / EXECUTOR v0.3.1 — 2026-07-30

Closes five wire-format gaps in the batch commitment and outbox preimage, surfaced while
implementing the executor's canonical serialization library (epic 2 ticket 03 — complete,
tested, reviewed). No behaviour changes: every gap closed here already had exactly one
implementation choice consistent with the surrounding text; this makes that choice explicit
so Settlement (epic 1 ticket 15, not yet implemented) decodes the same bytes the executor
encodes rather than each side guessing independently. Ticket 15 carries the reasoning for each
width.

### 1. Batch-commitment version block (§19)

§19 said the commitment carries "`protocol_version` and other version fields (§24)" without
enumerating which of §24's nine version fields those are. Now explicit: all nine, `u16`
each, in §24's declaration order, immediately after `protocol_version`. The commitment
commits to roots over structures whose own versions are otherwise invisible on-chain, so
the unknown-rejects rule has to reach every one of them.

### 2. `executorId` width (§14, §27.0)

§27.0's primitives list said `Address` (20 bytes) appears "only in `caller` preimages and
withdrawal recipients" — incompatible with §19's own `executorId` field, which Settlement compares against a
20-byte L1 send address. Both occurrences of the primitives rule (§14's summary, §27.0's normative
text) now list `executorId` as a third use.

### 3. `domainId` width in the `outboxId` preimage (§27.4)

`outboxId = SHA3-256(domainId || batchId(u64) || input_index(u64) || outbox_index(u32))`
gave widths for three of the four fields but not `domainId`. Now `domainId(u32)`,
consistent with every other use of `domainId` in the document set.

### 4. `firstInputSeq` / `lastInputSeq` widths (§19)

Now explicit `u64` each, matching `batchId` and `globalInputIndex` elsewhere in the format.

### 5. `submittedAtHeight` width (§19)

Now explicit `u64`, matching momentum height elsewhere in §27.1.

### Companion sync

`EXECUTOR.md` → v0.3.1 (upstream pin only; no content change — none of the five gaps were
restated in `EXECUTOR.md`'s own text). `ARCHITECTURE.md` unchanged — its `executorId`/`§19`
references are conceptual, not wire-format, so none needed amending.

### Source

The five decisions are implemented in
`internal/codec/commitment.go` / `outbox.go` (go-executor) and locked in by the
`CV-COMMIT-1` conformance vector.

---

## SPEC v1.6.0 / EXECUTOR v0.3.0 / ARCHITECTURE — 2026-07-26

Absorbs the generalized **Bridge Framework** into the normative documents as first-class schema, with Phase 1 reframed as the single **active** point of a general model and every other configuration reserved/gated. No Phase-1 behaviour changes — the WASM `EXECUTION` domain at `(ATTESTATION, NONE, ESCROW, SINGLE)` is byte-for-byte unaffected. Its core model is normative here; worked examples, comparative mappings, and the testing, verification and deployment strategy are in `BRIDGES.md`.

### 1. Trust model generalized (§1)

Two **independent** per-domain trust dimensions: an **execution profile** (`ATTESTATION → OPTIMISTIC → ZK_VALIDITY`) and a **foreign-fact profile** (`NONE → COMMITTEE → LIGHT_CLIENT → ZK_CONSENSUS`). A domain is exactly as trustless as the weaker of the two. Three permanent Rails restated (STF-verifiable ≠ Settlement-verified; a foreign fact commits what was observed, not what is true; commitment ≠ availability). Guarantee table generalized to `ESCROW` (honesty-independent) vs `MINTED` (profile-dependent). Phase-1 active point pinned normative.

### 2. Domain model generalized (§6, §6.3–§6.6)

`DomainRecord` gains `domainClass` (`EXECUTION | BRIDGE | MESSAGING`), `execProfile`, `foreignProfile`, `profileConfig`, `chainBinding`, `finalityModel` — all folded into `stfSpecHash`, all constrained to Phase-1 defaults until a spork-gated Core release. New normative subsections: the two profile ladders (§6.3, §6.4), legal-combination + custody-mode constraints (§6.5), and the bridge abstractions `ChainVerifier` / `SourceAdapter` / `BridgeLedger` / `ReleaseAdapter` + the `kind=1` peg-out authorization (§6.6). The executor's four-method plugin interface is unchanged (`EXECUTOR.md` §7).

### 3. Messaging and asset model generalized (§17, §18)

- §17.2: `kind=1` is the single authorization object for *every* withdrawal — Zenon-native (consumed by `Update`) and bridge peg-out (consumed by a foreign `ReleaseAdapter`) are the same object with different consumers. `MESSAGING` emits `kind=0` only. §17.4: reserved timeout/ack/refund messaging. §27.4: reserved foreign-recipient `kind=1` sub-variant (version-gated).
- §18.1: custody modes `ESCROW` (Phase 1) / `MINTED` (reserved). §18.3: `MINTED` conservation predicates added alongside `ESCROW`, with the **OD-1** reconciliation resolution — economic floor (`MaxBatchMint`, §25.2) + `OPTIMISTIC` execution floor, optionally Core-direct enforcement under `COMMITTEE`/`ZK_CONSENSUS`. `MINTED` aggregate solvency is only as strong as the layer that makes the foreign backing fact checkable to Core.

### 4. Storage, versioning, bonds, open items

- §5.4: reserved `AssetRegistry` (+ `MINTED` counters), `MessageState`, and Periphery verifier registries. §5.3: `RegisterDomain` row lists the new fields.
- §22: bond sizing covers `MaxBatchMint` for `MINTED` domains. §24: `proofData` contents are profile-dependent; semantics-defining domain fields folded into `stfSpecHash`; `ATTESTATION`-quorum `proofData` needs a `protocol_version` bump.
- §30: bridge framework recorded as schema-present/activation-reserved; **OD-1 (`MINTED` reconciliation)** flagged as the single gating open-design blocker before any asset bridge deploys.

### 5. ARCHITECTURE + EXECUTOR companions

`ARCHITECTURE.md` §6–§8 generalized (domain classes, two-ladder trust, bridge forward-compat) + new §4.4 peg-in/peg-out flow. `EXECUTOR.md` §7 adds the `BridgeLedger`/`ChainVerifier`/`SourceAdapter`/`ReleaseAdapter` delegation with the plugin interface unchanged; upstream pin → SPEC v1.6.0.

### Activation gating (unchanged safety posture)

Every generalized capability ships reserved: Core **MUST** reject any non-Phase-1 value until the enabling spork-gated release lands. Activation prerequisites: fraud-proof referee registry (`OPTIMISTIC`), validity/consensus verifier registries (`ZK_VALIDITY`/`ZK_CONSENSUS`), `MINTED` issuance + OD-1 resolution, and the per-chain `ChainVerifier`/`SourceAdapter`/`ReleaseAdapter` (last off-Zenon). Recommended sequencing: schema + `MESSAGING` first (no custody, no OD-1), then `OPTIMISTIC` fraud proofs, then trust-minimized `MINTED` bridges.

---

## SPEC v1.5.0 / EXECUTOR v0.2.2 — 2026-07-06

Closes the two spec-rooted plan gaps surfaced by the epic review.

### 1. Deployment verification assignment (§11.2, §11.3, §12)

§11.3 defined the Finalize checks without assigning them on-chain vs off-chain — infeasible as an on-chain obligation, since Settlement commits only chunk hashes to state and cannot recompute `sha3(assembled)`. Now explicit:

- **Settlement (on-chain):** per-chunk `chunk_hash == sha3(chunk_data)` at each chunk receive (bytes in hand — its only byte-level code verification); at `Finalize`/`UpgradeContract`, record-existence, deployer, all-chunks-present, and record consumption. No assembly, validation, instrumentation, or bytecode storage.
- **Runtime/STF (off-chain, deterministic):** assembly, `sha3(assembled) == code_hash`, `len == total_size`, Core 1.0 validation + prohibited-feature rejection, instrumentation, `CodeHash`/`ContractID` derivation. A mismatch is a `VALIDATION_FAILED` receipt (no contract / no code change) — reproducible by any watcher from L1-anchored data, fraud-provable in Phase 2.
- §12 upgrade finalizer updated to the same split (Core: bookkeeping + consumption; STF: pipeline).

### 2. Executor bond mechanics (§5.3, §5.4, §19, §22, §25.2, §28)

The bond flow was unspecified (no posting vehicle, no return path, no methods). Now defined as the **sentinel two-leg flow**: `DepositQsr` accumulates the QSR leg (reclaimable via `WithdrawQsr` while unconsumed); `PostBond(domainId)` carries exactly the ZNN leg and consumes the required QSR deposit; accepted only from the registered executor address after admin `RegisterExecutor`. The executor is **ACTIVE** (eligible to `SubmitBatch`, enforced in §19) only while registered + fully bonded. `RevokeBond` returns both legs once the address is no longer the registered executor and all its batches are `FINALIZED`. New §5.3 methods: `DepositQsr`/`WithdrawQsr`/`PostBond`/`RevokeBond`; new §5.4 storage: per-address QSR bond-deposit balances. Precedent: `DepositQsrMethod`/`WithdrawQsrMethod`/`checkAndConsumeQsr` (`vm/embedded/implementation/common.go`), exact-amount ZNN leg per `RegisterSentinelMethod` (`sentinel.go`); amounts are pre-mainnet parameters (§25.2).

### Companion sync

`EXECUTOR.md` → v0.2.2 (upstream pin, §5.5 prerequisite adds the bond methods). Epic tickets synced in the same pass (see below).

### Epic plan changes (applied with this revision)

- Tickets **10 ↔ 11 swapped** (admin layer now builds before the domain registry that consumes its gate); cross-references updated in 12–17.
- Ticket **12** rewritten to the two-leg bond flow; ticket **14** Finalize reduced to the on-chain half (assembly/validation moved to ticket 04's STF scope); ticket **15** requires the ACTIVE (bonded) executor; ticket **04** gains the assembly-hash check.
- **Vector authoring ownership**: tickets 02/03/04 now explicitly author their layer's §31 vectors (CV-PATH/CV-APPLY, CV-COMMIT/INPUTROOT/BUNDLE/ASSET/OUTBOX/DEPLOY, EXEC-DEP/EXEC-FAIL-OUTBOX) and retire the stale-artifact banners.
- New ticket **20 — system vault contract** (§18.1a/§32; E2E fixture); the E2E/activation ticket renumbered **20 → 21**.

---

## SPEC v1.4.0 / EXECUTOR v0.2.1 — 2026-07-06

Resolves the gaps found in the source-verification review of the spec suite against `feature/merkle-state-root`.

### 1. Withdrawal release model (§5.3, §17.3, §18.2, §21, §27.9, §27.10)

The release trigger was previously unspecified: `RelayMessage` both registered a reserve and was implied to release it after a second `WithdrawalDelay` from finalization, which conflicted with `outboxId` replay protection (a second relay of the same message is rejected) and silently doubled the advertised withdrawal delay.

- `RelayMessage` is now **registration-only**: proof + `FINALIZED` + replay check → record `outboxId`, increase `pendingWithdrawalReserve`, enqueue on the domain's release queue.
- A new permissionless, rate-limited **`Update`** method (added to the §5.3 method list) drains the queue in registration order, ≤ `MaxWithdrawalsPerUpdate` (16) per call, one descendant send per release, conservation re-checked per release. Precedent: `UpdateEmbeddedAcceleratorMethod` / `checkAndPerformUpdate` (`vm/embedded/implementation/accelerator.go`, `common.go`) — note this pattern is permissionless + rate-limited, not producer-gated.
- **`FINALIZED` is the only time gate**: the post-finalization `WithdrawalDelay` (`finalizationHeight + WithdrawalDelay` release-eligible height) is removed.
- `MaxWithdrawalsPerRelay` renamed **`MaxWithdrawalsPerUpdate`** (§25). `PAUSE_RELAY` now halts `RelayMessage` registration and `Update` release (§23, §27.10).
- New §5.4 storage: the per-domain release queue.

### 2. On-chain input-sequence accounting (§4.6, §5.4, §19)

Contiguity verification and `AssetFlowSummary` deposit correspondence implicitly required Settlement to know each domain's input indices and per-range deposits, but no text assigned that obligation.

- New §4.6 normative bullet: **Settlement assigns the global input index at receive time** (every decodable input envelope for a registered domain increments the counter; undecodable sends consume no index — same predicate as the executor's §4.5 derivation), records the domain's assigned indices and per-(domain, asset) cumulative deposit totals, and verifies batch bounds against its own assignment (`firstInputSeq` = earliest unconsumed, `lastInputSeq` assigned and ≤ highest assigned, range count ≤ `MaxBatchInputs`).
- Grounding: embedded receives replay committed send order (per-contract FIFO sequencer, `chain/account/sequencer.go`). Side effect: closes the receive-lag race — a batch covering not-yet-received inputs is rejected and resubmitted.
- §19 `SubmitBatch` verification list and §5.4 storage list updated to match.

### 3. Deploy chunk sizing (§11.1, §11.2, §25)

Non-last chunks were required to be exactly `MaxDAChunkSize` (16 KiB) = `MaxDataLength`, leaving no room for the method's ABI envelope in the account-block data field — such a chunk was unpostable on L1.

- New constant **`MaxDeployChunkData` = 15 KiB** governs on-chain deploy chunk payloads; `chunk_count = ceil(total_size / MaxDeployChunkData)`; **`MaxChunkCount` = 18** (ceil(256 KiB / 15 KiB)).
- `MaxDAChunkSize` (16 KiB) now governs **only** off-chain DA bundle chunk serving (§20).
- **`MaxWasmSize` renamed `MaxCodeSize`** — the pre-instrumentation code-size cap is runtime-neutral, not WASM-specific.

### 4. Upgrade finalization (§12)

There was no path from a chunked replacement upload to an executed upgrade: `DeployContractFinalize` (the only consumer of a pending record) always derives a new `ContractID`.

- `UpgradeContract(contract_id, new_code_hash)` is now the **finalizer of upgrade-destined records**: requires a complete, non-expired pending record for `(domainId, new_code_hash)` whose deployer == caller; runs the full §11.3 pipeline; replaces code (`CodeHash` changes, `ContractID` unchanged); consumes the record. `Finalize` and `Upgrade` are mutually exclusive consumers; §11.5 expiry applies identically.

### 5. Conformance-artifact status notes (§31, `WASM Spec/execution-conformance-v1-README.md`)

`execution-conformance-v1.json` (EXEC-001…012) encodes a frame/blob layout without `deposit_asset`/`deposit_amount`/`claimed_deposit` and does not match §27.1/§27.1a; the §31 vector IDs (EXEC-DEP-*, EXEC-FAIL-OUTBOX-1, CV-*) and the CV-PATH/CV-APPLY trie vectors are not yet generated. Stale-status banners added to both documents; regeneration is gated on build stage 1A (§29, EXECUTOR P-7). Until then only SMT-001…014 (`smt-v1-test-vectors.json`) are authoritative.

### Companion-document sync

- `ARCHITECTURE.md`: §2.1 core responsibilities (input-index assignment, release queue/`Update`), §3 integration map (Settlement-side numbering, `Update` precedent), §4.2 withdrawal flow, §4.3 chunk sizing + upgrade finalization.
- `EXECUTOR.md` → v0.2.1: upstream pinned to SPEC v1.4.0; §5.5 Settlement client and P-1 method list include `Update`; §9 loop sends `Update` on finalization.

### Epic sync (applied with this revision)

The `epics/` tickets were synced to v1.4.0: README version pins; ticket 09 (renamed/added constants incl. `MaxOnChainCommitmentSize` 15 KiB, new storage entities: assigned-index records, cumulative deposit totals, release queue, `lastUpdate`); ticket 11 (pause scopes); ticket 13 (shared input-admission helper, cumulative-totals deposit correspondence); ticket 14 (`MaxDeployChunkData`/`MaxChunkCount=18`/`MaxCodeSize`, Finalize/Upgrade mutual exclusion); ticket 15 (assigned-index verification, 15 KiB size); ticket 16 (rewritten: registration-only `RelayMessage` + `Update` release crank); ticket 17 (on-chain upgrade-record checks + admission helper); tickets 18/20 (client sends `Update`; E2E flow). Also fixed misattributed code citations (`AccountBlock` is `chain/nom`, not `common/types` — tickets 01, EXECUTOR §1.4; `RegisterSentinelMethod` implementation line — tickets 12/18) and the ARCHITECTURE §2.2 watcher stage range (1–4, not 1–3). `MaxOnChainCommitmentSize` was reduced to 15 KiB in §19/§25 (same chunk-envelope bug class as `MaxDeployChunkData`).

### Known follow-ups

All open items from this revision (assembly-verification assignment, bond flow, vector-authoring ownership, system vault ticket, 10↔11 build order) were resolved in v1.5.0. The conformance artifacts themselves still regenerate at build stage 1A (tickets 02/03/04).

---

## SPEC v1.3.0

Generalised the domain schema with no behavioural change to Phase 1. Added `inputSource` (§6.1) and `proposerPolicy` (§6.2) to `DomainRecord`; the single `executor` field became the `executors` set (§6.2). Renamed batch bounds and the domain cursor to a generic **input sequence** (`firstInputSeq`/`lastInputSeq`, §4.5–§4.6, §19), with `globalInputIndex` as the value for L1-sourced domains. Generalised `SubmitBatch` acceptance and §22 to the entitled proposer of the executor set. Phase 1 remained `L1_NATIVE` / `SINGLE` / size-1; no migration required.

## SPEC v1.2.0

Prior baseline.
