# Ticket E1-09 — `CallL2` / `UpgradeContract` / `BuyStorage` input surface + payable deposits

**Epic:** 1 — L1 Settlement · **Phase:** 1B · **Order:** E1-09

## Goal
Complete the WASM-domain user input surface: `CallL2` (payable),
`UpgradeContract` and `BuyStorage`, which enqueue canonical inputs into the
domain's stream and, for payable calls, route attached value into custody. End
state: all eight SPEC §27.5 input types are accepted on-chain and surfaced as canonical inputs; the
on-chain half of payable deposit (custody increase + frame inputs) is done.

## SPEC refs
SPEC §5.3 (method list), §27.5 (CallL2/Deposit payload + input identity; `BuyStorage` is `input_kind = 7`), §10.7 (developer fees), §13.8 (storage quota), §18.5
(payable `CallL2` deposit delivery), §12 (`UpgradeContract`), §10.1 (`plasma_
limit` is L2 budget, not L1 plasma).

## Depends on
Tickets E1-03 (domain), E1-04 (fee accounting), E1-05 (custody/deposit on-chain half), E1-06
(deploy records for upgrade chaining).

## Key work
- `CallL2(domainId, target_contract, plasma_limit, payload)` — **payable**: MAY
  carry attached ZTS value. On receive, if value > 0 increase
  `totalDeposited[domainId][asset]` (unconditional/final) and mark the input as
  carrying `deposit_asset`/`deposit_amount` for the off-chain frame; enqueue as a
  canonical input (`input_kind=0`). The receive rejects a non-custodyable attached asset
  (no `MaxBatchWithdrawal` configured for the (domain, asset), §18.5);
  `ValidateSendBlock` rejects `payload > MaxCallPayload`, and a `plasma_limit` above
  `constants.MaxInputGas` (`constants.IsInputGasInBounds`, §10.1) with
  `ErrForbiddenParam` — the constant and predicate already exist and this is
  their only caller. `plasma_limit` is **not** placed in the contract-visible
  frame.
- `UpgradeContract(domainId, contract_id, new_code_hash)` — enqueue
  `input_kind=5`. Replacement bytecode is staged through the chunked mechanism
  (Ticket E1-06); `UpgradeContract` is the **finalizer** of the upgrade-destined
  record (§12): on-chain, verify a complete, non-expired pending record for
  `(domainId, caller, new_code_hash)` exists with `ReceivedCount == ChunkCount`
  (SPEC v1.16.4 keys records by deployer; v1.16.5 stores chunk hashes per entry),
  then consume it — deleting the header and its chunk-hash entries —
  via Ticket E1-06's consumption hook (mutually exclusive with
  `DeployContractFinalize`). Contract-level authorization
  (`OWNER_UPGRADEABLE`, owner via `caller`), validation/instrumentation, and
  `CodeHash` recomputation are the STF's (Ticket E2-04); `IMMUTABLE` upgrade
  attempts surface as `UNAUTHORIZED` in the off-chain receipt.
- **Upgrade policy and ownership (§12, §27.12).** A contract declares its
  policy and owner in the `metadata` its `DeployContractStart` carries (Ticket
  14), so `OWNER_UPGRADEABLE` contracts and `UpgradeContract` are reachable.
- **`TransferContractOwnership(domainId, contract_id, new_owner)`** (§5.3, §12):
  a new user method, `input_kind = 6`. Settlement checks only the envelope and
  assigns an input index (§4.6); authorization against the owner leaf is the
  STF's. Encode the input tail `contract_id:bytes32 || new_owner:Address(20)`
  (§27.5).
- **Rejection is at receive, not a receipt (§4.6).** A `CallL2` or
  `UpgradeContract` Settlement rejects — non-custodyable asset, oversize payload,
  over-budget `plasma_limit`, or a missing, incomplete, expired, wrong-deployer or
  already-consumed pending record — is not an input: no index, no receipt, no
  `inputRoot` leaf.
- **Fees (§10.7).** `UpgradeContract` carries exactly
  `total_size × znnPerByteOfBytecode` from the consumed record;
  `BuyStorage(domainId, contract_id, bytes)` carries exactly
  `bytes × znnPerByteOfState` with `bytes ≥ 1`. Reject a non-ZNN or wrong-amount
  attachment at receive. Fees go to `totalFeesCollected` and the assigned-index
  record (Ticket E1-04), never to custody or `AssetFlowSummary`.
- **No production codec in go-zenon.** Settlement never builds, hashes or commits
  a canonical input in Phase 1: Core does not verify execution (SPEC §19), and the
  executor re-encodes every input from the ABI (§4.1). The encoder lives only in
  zeno-executor's `internal/codec` (E2-03). A shared encoder in go-zenon waits until
  Settlement must recompute `inputRoot`, which is Phase 2 fraud proofs.
- **Test-only encoding check.** For each of the eight kinds, unpack the ABI call
  data Settlement accepts and compare field by field with the matching
  CV-INPUTROOT-1 leaf (§27.5) pinned in `vm/embedded/implementation/settlement_test.go`:
  `input_kind`, `source_account_block_hash`, the kind-specific tail, and for kinds 0
  and 1 the deposit asset and amount Settlement takes from the attached value. The
  leaf carries no caller or global input index. This catches
  an ABI argument whose order or width differs from the canonical tail.
- **Envelope encoding (SPEC §4.1).** Every input method is a typed ABI method with
  `domainId uint32` as its first argument, matching `Deposit` (Ticket E1-05). Ticket
  E2-06's decoder reads these ABI methods and re-encodes the canonical input (§27.5).
- **Paused domains (SPEC §4.1).** Every input method this ticket adds rejects a
  domain whose `status` is `PAUSED` at the receive.
- Replay protection inherited from L1 account-block uniqueness + `chain_id`
  binding; no application nonce.

## Decisions behind the input surface

The executor side of each of these is already shipped. The reasoning is kept here
because nothing else in the tree records it.

- **Acceptance assigns the index (§4.6).** A call Settlement rejects at its
  receive is not an input: it gets no index, no receipt and no `inputRoot` leaf.
  So every input the executor sees has already passed its method's checks, and an
  executor that observes one whose on-chain precondition fails halts rather than
  committing a status. The alternative — assigning indices on envelope
  decodability — gave different `inputRoot`s, receipt sets and numbering for the
  same on-chain activity.
- **An `input_kind` outside `0..6` is unreachable from a live Settlement.** §27.5
  still defines its encoding (the `input_kind ‖ source_account_block_hash` prefix
  with no tail) and §15 gives it `UNSUPPORTED_OPERATION`, because a receipt needs an
  `inputHash` whatever the kind. With acceptance governing, both are defensive.
  Settlement must never assign an index to an unknown kind.
- **`plasma_limit` maps to L2 gas one-for-one, and is capped, never clamped.**
  Any other ratio would be an invented constant. `MaxInputGas` stays below
  `2^63 − 1`, so every accepted budget fits a signed 64-bit ledger. A budget above
  it is rejected at send validation, which is why no receipt status is needed:
  clamping, saturating or a pre-execution `OUT_OF_GAS` would each commit a
  different `gasUsed` for the same input (§10.1).
- **Deploy, upgrade and ownership inputs carry no budget.** Kinds 2–6 have no
  `plasma_limit`, so there is nothing to charge against, and their receipts carry
  `gasUsed = 0`. Do not add a `plasma_limit` to their encodings: that would change
  the committed `gasUsed` of every such receipt (§10.1).
- **The deployer is the raw 20-byte L1 address.** `ContractID =
  SHA3-256(deployer_address(20B) ‖ deployment_global_input_index(8B BE) ‖
  chain_id)`. Settlement records the deployer as the L1 send address and compares
  the finalize `caller` against it, never a §9.2 hashed caller, so the raw address
  is the only form that type-checks against that comparison (§12).
- **Settlement does not decode contract metadata; kind 6 checks only the
  envelope.** Upgrade policy and ownership are L2 state. Settlement verifies only
  `SHA3-256(metadata) == metadata_hash` and the size bound, and the runtime
  validates the structure at finalize. Duplicating either check on L1 would put
  two parsers on one consensus rule, and authorizing a transfer on L1 would need
  L2 state Settlement does not hold (§11.1, §12).

## Code references
- `vm/embedded/implementation/liquidity.go` (`LiquidityStakeMethod` — one send
  carrying value + ABI call) [EXISTS].
- `vm/constants/plasma.go` (`MaxDataLength`, 16 KiB input ceiling) [EXISTS];
  Ticket E1-01 `MaxCallPayload` [NEW].

## Acceptance criteria
- `CallL2` with and without attached value accepted; value path increases
  `totalDeposited` and tags the input frame; oversize payload / non-custodyable
  asset rejected.
- `UpgradeContract` enqueues `input_kind=5` with correct canonical encoding;
  it consumes the caller's own complete pending record `(domainId, caller,
  new_code_hash)` and is rejected when that record is missing (including when only
  another deployer holds one for the hash), incomplete, expired, or already
  consumed by `DeployContractFinalize`.
- Every method this ticket adds pays the Settlement send-plasma rule (SPEC §10.1,
  landed in Ticket E1-06): an 8 KiB `CallL2` payload costs the plain-send data price.
- Both methods run the Ticket E1-05 input-admission helper (index assigned;
  payable `CallL2` deposits land in the cumulative totals).
- `BuyStorage` enqueues `input_kind=7` with tail `contract_id || bytes`; a wrong
  fee is rejected at receive; an accepted fee is kept whatever the receipt.
- For all eight input kinds, the fields Settlement accepts match the
  CV-INPUTROOT-1 leaf field by field (test only; go-zenon
  carries no production encoder).

## Phase 1B exit gate
With Tickets E1-01–E1-09 the Settlement contract implements every §5.3 method,
custody/conservation, deployment, batch acceptance, relay/withdrawal, and admin/
pause — ready to **ship dark on testnet** (SPEC §29-1B) ahead of wiring.
