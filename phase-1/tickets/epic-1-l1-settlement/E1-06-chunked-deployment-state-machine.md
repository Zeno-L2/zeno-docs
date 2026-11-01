# Ticket E1-06 — Chunked deployment state machine (on-chain bookkeeping)

**Epic:** 1 — L1 Settlement · **Phase:** 1B · **Order:** E1-06

## Goal
Implement the on-chain half of WASM deployment: the chunked, hash-committed
`DeployContractStart/Chunk/Finalize` state machine keyed by `(domainId,
deployer, code_hash)`, with per-chunk hash verification and expiry. End state: a deployer
can commit a large module across chunks and finalize it deterministically.
**Settlement verifies chunk hashes at receive and record bookkeeping only — it
never assembles, validates, or instruments code** (assembly verification and
the §11.3 pipeline are the executor's STF, Ticket E2-04); Settlement MUST NOT
execute WASM.

## SPEC refs
SPEC §11 (deployment), §11.5 (expiry), §12 (identity/upgrade — `code_hash` vs
`CodeHash` split), §5.3 (deploy trio), §1 (no WASM on L1).

## Depends on
Tickets E1-03 (domain), E1-01 (deploy-record storage), E1-05 (input-admission helper).

## Key work
- `DeployContractStart(domainId, code_hash, metadata_hash, total_size,
  chunk_count, metadata)`: reject at receive unless
  `SHA3-256(metadata) == metadata_hash` and
  `len(metadata) ≤ MaxContractMetadataSize (256)` (§11.1). Settlement does not
  decode `metadata`; the runtime validates it at finalize (§27.12). Add
  `MaxContractMetadataSize` to `vm/constants/settlement.go`.
  `total_size ≤ MaxCodeSize (256 KiB)`,
  `chunk_count = ceil(total_size / MaxDeployChunkData) ≤ MaxChunkCount (18)`;
  run the expiry sweep (below), then create a header-only pending record
  (`MetadataHash`, `TotalSize`, `ChunkCount`, `StartHeight`, `ReceivedCount = 0` —
  no pre-allocated chunk slots) and append `(record key, StartHeight)` to the start
  queue; reject a duplicate non-expired record.
- **Rejection is at receive, not a receipt (§4.6).** Every check in this ticket —
  duplicate record, absent or expired record, wrong deployer, out-of-range or
  duplicate `chunk_index`, a `chunk_hash` that does not match, a finalize with a
  chunk missing — makes Settlement reject the call at its receive. A rejected
  call is not an input: it consumes no index, produces no receipt and
  contributes no `inputRoot` leaf. The `VALIDATION_FAILED` receipts below belong
  to the executor's STF, which sees only calls Settlement accepted.
- `DeployContractChunk(domainId, code_hash, chunk_index, chunk_hash,
  chunk_data)`: the record `(domainId, caller, code_hash)` must exist (non-final, non-expired);
  `chunk_index ∈ [0, chunk_count)` not already present; sizing rules (all but
  last exactly `MaxDeployChunkData (15 KiB)` — sized so chunk + ABI envelope fits
  `MaxDataLength` —, last = remainder; `chunk_count==1` ⇒ single chunk size ==
  `total_size`); `chunk_hash == sha3(chunk_data)`. Store
  `chunk_hash` under its own key `(domainId, deployer, codeHash, startHeight,
  chunkIndex)` and increment `ReceivedCount`; a repeated `chunkIndex` finds its
  entry present and is rejected. Only the hash is committed (bytes go to DA).
- `DeployContractFinalize(domainId, code_hash)`: on-chain checks only —
  the record `(domainId, caller, code_hash)` exists, all chunks present, attached ZNN equals
  exactly `total_size × znnPerByteOfBytecode` (SPEC §10.7; add the fee to
  `totalFeesCollected` and the assigned-index record, never to custody, Ticket E1-04); on acceptance the
  record is consumed (finalized). The **executor's STF** performs assembly,
  `sha3(assembled) == code_hash` + `len == total_size` verification (mismatch ⇒
  `VALIDATION_FAILED` receipt, no contract), the §11.3 validation +
  instrumentation, and `CodeHash`/`ContractID` derivation (this contract does
  none of it — §11.3 on-chain/off-chain split).
- Mutually exclusive consumers (§12): a completed pending record is consumed by
  **either** `DeployContractFinalize` (new contract) **or** `UpgradeContract`
  (re-codes an existing contract, Ticket E1-09) — whichever is accepted first; a
  consumed record is finalized for §11 purposes. Wire the record-consumption
  hook Ticket E1-09 calls.
- Expiry: a record unfinalized within `DeploymentExpiryWindow (2880)` heights is
  treated as nonexistent and frees its key; plasma not refunded. Applies to
  upgrade-destined records identically.
- **Consumption deletes everything (§11.3).** Finalize (and Ticket E1-09's upgrade)
  require `ReceivedCount == ChunkCount`, then delete the header and all its
  chunk-hash entries (at most `MaxChunkCount`).
- **Expiry sweep (SPEC v1.16.5, §11.5).** A FIFO start queue holds
  `(record key, StartHeight)` per accepted start. Each `DeployContractStart`, before
  writing its own record, examines at most `MaxDeploySweepPerStart = 2` head
  entries, stopping at the first unexpired one. For each expired entry: delete its
  chunk-hash entries under `(…, StartHeight, ·)`, delete the record only if it
  exists with the same `StartHeight`, and pop the entry. Every node must delete
  identically (storage feeds `ChangesHash`). The sweep never changes an accept or
  reject outcome, so the executor needs no change.
- **Envelope and paused domains (SPEC §4.1).** The three deploy methods are typed
  ABI methods with `domainId uint32` first, like `Deposit` (Ticket E1-05), and each
  rejects a domain whose `status` is `PAUSED` at the receive.
- **Record key (SPEC v1.16.4, §11).** Key every pending record by `(domainId,
  deployer, code_hash)`, the deployer being the `DeployContractStart` sender. Chunk,
  finalize and the upgrade consumer look up with the caller's address, so a call
  from any other address finds no record. This removes code-hash squatting: a party
  who starts a record for someone else's `code_hash` only creates its own. Two
  deployers may stage the same `code_hash` at once.
- **Settlement send plasma (SPEC v1.16.4, §10.1).** In `GetBasePlasmaForAccountBlock`
  (`vm/plasma.go`), for `block.ToAddress == types.SettlementContract` return
  `max(method.GetPlasma(...), AccountBlockBasePlasma + len(block.Data) ×
  ABByteDataPlasma)`. Apply the same rule on every path that computes a send's
  required plasma (block verification and the RPC that reports required plasma), so
  they agree. A full chunk costs about 1.07M plasma and needs fusion; integration
  tests give the deployer fused plasma.
- **Decisions (SPEC v1.16.3).**
  - Heights are receive heights (`context.GetFrontierMomentum().Height`, §4.6). The
    start height is recorded at the `DeployContractStart` receive; a record is
    expired at receive height `h` when `h ≥ startHeight + DeploymentExpiryWindow`
    (§11.5).
  - Stateless checks run at send validation and again at the receive, as `Deposit`
    and `RegisterDomain` do.
  - `total_size = 0` is rejected at `DeployContractStart` (§11.1).
  - While `znnPerByteOfBytecode` is unset, `DeployContractFinalize` is rejected
    (`ErrDataNonExistent`); a zero fee is never accepted (§10.7).
  - Errors: `ErrDataNonExistent` for an absent or expired record,
    `ErrPermissionDenied` for the wrong deployer, `ErrInvalidTokenOrAmount` for a
    wrong fee, `ErrForbiddenParam` for every other rejection.

## Code references
- Ticket E1-01 deploy-record storage [NEW]; Ticket E2-03 canonical input encodings for
  the deploy kinds (§27.5) [NEW].
- `vm/constants/` deployment constants (Ticket E1-01) [NEW].

## Acceptance criteria
- A finalize with a missing, non-ZNN or wrong-amount bytecode fee is rejected at
  receive; an accepted fee is kept even when the executor's pipeline yields
  `VALIDATION_FAILED`.
- A burst of starts that all expire is swept by later starts, two per start, until
  no expired entry remains; storage then holds no header or chunk entry for them.
- A record that was finalized and later reaches its expiry height leaves nothing
  behind once swept.
- A record restarted after expiry (same key, new `StartHeight`) survives the sweep
  of its predecessor's queue entry, and never sees the predecessor's chunk hashes.
- Every existing accept and reject outcome is unchanged by the sweep.
- Multi-chunk deploy with all chunks present finalizes (record consumed); a
  finalize with a missing chunk or wrong `caller` is rejected; a chunk whose
  `chunk_hash ≠ sha3(chunk_data)` is rejected at receive.
- CV-DEPLOY-1 single-chunk (`chunk_count==1`) sizing accepted.
- Duplicate-active record rejected; wrong-deployer chunk rejected; out-of-range /
  duplicate `chunk_index` rejected.
- A record left unfinalized past `DeploymentExpiryWindow` is freed and the
  `code_hash` is reusable.
