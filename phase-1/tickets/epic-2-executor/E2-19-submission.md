# Ticket E2-19 — Submission: signer, `SubmitBatch`, devnet harness

**Epic:** 2 — Executor · **Phase:** 1C · **Order:** E2-19

## Goal
Submit sealed batches to Settlement. End state: the supervisor builds, signs and
publishes `SubmitBatch` account blocks from the executor address, confirms
acceptance by reading Settlement state, and a batch is accepted on-chain by a
devnet node with both sporks enforced.

## SPEC refs
SPEC §10.1 (Settlement send plasma), §19 (`SubmitBatch` checks), §22.1 (executor
address); EXECUTOR §1.5 (key isolation), §5.5, §15 (failure table), §17.

## Depends on
E2-16, E2-17, E2-18 (batch-root ordering tests: this ticket submits those roots).

## Key work
- **Key and signer.** `submissionKeySource` (FILE or ENV) yields a 32-byte ed25519
  seed, hex-encoded. The supervisor derives the key pair with go-zenon `wallet`,
  checks it matches `executorAddress`, and zeroes the seed after use. KMS stays
  unimplemented. Workers never see the key (EXECUTOR §1.5).
- **Account-block builder.** Fill height and previous hash from the executor
  address's frontier account block, `MomentumAcknowledged`, plasma
  (`max(P_method, 21000 + 68 × len(data))`), hash and signature. One account chain
  serves every domain, so sends are serialised. Read the account frontier from the
  node before every send, and never reuse an account height. Insufficient plasma is
  an operator-visible error that retries, not a stall or a halt.
- **Own activity.** New required config field `pillarName`. Before sending, check
  the pillar is active and its executor-set member names `executorAddress`. If not,
  log an operator-visible warning and do not submit; never halt.
- **`SubmitBatch`.** Pre-submit guard: read the domain cursor; submit only when the
  batch's `batchId == nextBatchId`, its first input is ordinal `consumedInputCount`,
  and its `preStateRoot` is the previous accepted `postStateRoot` (zero hash for the
  first batch). After the receive confirms, re-read the batch entry: present and
  equal means accepted; absent means rejected at the receive, so retry after
  re-checking the submit pause (`PAUSE_SUBMIT`, global or per domain, §23) and own
  activity. A `PAUSED` domain status does not block submission: its accepted inputs
  are still batched (§6). A chain batch at `batchId` that
  differs from the local record is a consistency fault (HALT; re-sync).
- **Submission log.** Append-only, under `<storagePath>/supervisor/`, one entry per
  send (domain, method, key such as `batchId`, account height, block hash, outcome),
  replayed against chain state at boot.
- **Devnet harness.** A Docker devnet from go-zenon's devnet tooling with
  `StateRootSpork` and `OffchainExecutionSpork` active from genesis, a registered
  domain and executor, and a configured asset; used by this ticket and by E2-21.

## Code references
- zeno-executor `internal/supervisor/` (`key.go`, `serve.go`, `config.go`),
  `docker/` [EXISTS].
- go-zenon `wallet/keypair.go`, `rpc/api/ledger.go` (`PublishRawTransaction`),
  `vm/plasma.go` (`GetBasePlasmaForAccountBlock`), `rpc/api/embedded/plasma.go`,
  `vm/embedded/implementation/settlement.go` (`acceptBatch`) [EXISTS].

## Acceptance criteria
- A contiguous batch is accepted on the devnet; the confirmed batch entry equals the
  local commitment.
- Signing round-trips with `wallet.VerifySignature`; a seed not matching
  `executorAddress` fails boot.
- A submit pause or an inactive pillar blocks submission without halting; a domain
  whose status is `PAUSED` still has its batches submitted.
- A supervisor restart with an unconfirmed send outstanding neither reuses its
  account height nor submits twice.
- `go test ./...` passes; the key-isolation tests from E2-13 still pass.
