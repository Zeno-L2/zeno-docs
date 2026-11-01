# Ticket E2-23 — Two-phase reconciliation: docs, spec-to-code audit, cleanup, momentum hash

**Epic:** 2 — Executor · **Phase:** 1C · **Order:** E2-23

## Goal
Bring the documents in line with the two-phase plan and with the delivered code, and make
zeno-executor's comments self-contained. End state:
- SPEC, EXECUTOR and the companion documents describe two phases (testnet attestation, then
  validity proofs on mainnet) and match the code in zeno-executor and go-zenon;
- zeno-executor has no dead code, and no Go file cites a document section, a document name, a ticket
  or a design id;
- every input's context, and the frame a contract reads, carries the momentum hash beside its height
  and time.

The go-zenon half of the cleanup is Ticket E1-12.

## Inputs
- `phase-1/followups/TWO-PHASE-PLAN.md` — phase change and Phase 1 impact.

Phase 2 design (`phase-2/`, `phase-1/followups/`) is out of scope. It is worked on
separately once Phase 1 is complete.

## Depends on
E2-22. Part B also waits for E1-12, so it audits Settlement after the removals.

## Key work

### A. Two-phase documents (zeno)
- **SPEC:**
  - §1 mainnet gate and §2.2 roadmap: two phases; mainnet requires `ZK_VALIDITY`.
  - §6.3: `ZK_VALIDITY` is the mainnet profile; `OPTIMISTIC` stays a reserved ordinal only.
  - §13.1 and §13.2: the executor-state SMT has a Rust implementation pinned by the SMT vectors;
    the hash decision moves before mainnet.
  - §18.3 and §6.5: the `MINTED` floor becomes `ZK_VALIDITY`.
  - §20 and §21: drop the Phase 2 DA-enforcement and challenge-window text.
  - Remove §22.5, the `Challenge` reserved method, `ExecutorReregisterCooldown`, the challenger payout,
    and the "refunds bound to depositors" item in §32.
  - Remove `valueCaps` from §6 (`DomainRecord`), §23, §25.2 and every other mention (Ticket E1-12
    removes it from Settlement). `MaxBatchWithdrawal` is the one per-batch cap.
  - Add `momentum_hash` (part E) to §9 and §27.1.
- **EXECUTOR:** §1.4 (two implementations pinned by vectors, replacing "one verifier"), §4 roles,
  remove §5.8 and §11, §13, §20 phase table. The DA layer (§5.6) is the gateway's HTTP bundle
  serving only.
- **BRIDGES:**
  - §2.1 and §2.2: remove the watcher, `Challenge` edge and DA substrate.
  - §3.2–§3.6: `ZK_VALIDITY` replaces `OPTIMISTIC` in the floor, the ladder and every recommended
    starting point.
  - §4.5: an Ethereum `ReleaseAdapter` needs a proof of Zenon consensus plus Settlement storage
    inclusion, not the batch's validity proof.
  - §8–§12: remove withheld-bundle, challenge-liveness and referee material.
- **ARCHITECTURE, DEMOS, `clients/`:** same removals, including `valueCaps` from the TS SDK and
  Settlement UI plans. DEMOS: the BTC `ChainVerifier` is Rust; the
  relay stays Go; Demo 2 needs no watcher.
- Bump SPEC and EXECUTOR versions; add a CHANGELOG entry.
- Where Phase 1 text describes Phase 2, state only that mainnet requires `ZK_VALIDITY` and point to
  `phase-2/`. Phase 2 design detail stays out of the Phase 1 documents.

### B. Spec-to-code audit
- For every normative statement in SPEC and EXECUTOR that names executor or Settlement behaviour,
  find the implementing code, or classify the statement as reserved.
- Start from the citation map part C records, then cover the rest of SPEC §4, §8–§27 and
  EXECUTOR §1–§21.
- Record each mismatch in the run directory (not committed): section, code location, and which
  side is wrong.
- Resolve a mismatch in the documents when the code reflects a decision. When the document is the
  decision, list a code fix and triage it into its own ticket. This ticket changes no code
  behaviour.

### C. Comment cleanup (zeno-executor)
Land C as its own stage, so the comment-only check applies to it alone. Before rewording, record
every citation (section, file, line, the code it annotates) in the run directory as the citation map
part B starts from.
- **Scope:** every `.go` file, tests included. Remove:
  - section references (`§`, 318 lines across 76 files);
  - document names (`SPEC`, `EXECUTOR`, `ARCHITECTURE`, `BRIDGES`);
  - ticket references ("this ticket", "later tickets", `E2-NN`);
  - design ids (`OD-N`, decision and amendment ids such as `D20`, `A16`).
- **Rewrite rule:** state the rule the code enforces and why, in words. Delete the comment when the
  code already says it. Never write "per the spec".
- Remove the `WATCHER` role from the comment at `runtime/runtime.go:48`.
- **Stays:**
  - conformance vector ids (`CV-*`, `EXEC-*`), which name shipped test data;
  - the conformance JSON artifacts and the gas table, which are mirrored byte-identically in zeno
    `artifacts/` and `wasm-vm/` and digest-pinned;
  - `README.md`, which is operator documentation.

### D. Dead code (zeno-executor)
- Run `deadcode ./...` (golang.org/x/tools) and `staticcheck -checks U1000 ./...`.
- Also check by hand: constants, struct fields and configuration values that are set but never read,
  and declarations kept only for the watcher, fraud proofs or libp2p DA.
- Remove what is unused. A value read only by a test is unused. Record anything kept, with the
  reason, in the run directory.

### E. Momentum hash in the input context
- `domain.InputContext` gains `MomentumHash` (32 bytes) beside `MomentumHeight` and `MomentumTime`.
- The contract frame (`ExecutionInputFrame`) gains `momentum_hash : bytes32` immediately after
  `momentum_time`. The payload offset moves by 32 bytes.
- **Sources:**
  - the input path fills it from the confirmed momentum that establishes the input's position
    (`internal/ledger/decode.go:193-194`);
  - `zeno_simulate` takes it from the momentum it already reads for height and time
    (`internal/gateway/simapi.go:210-211, 259-260`).
- **Everywhere the context travels:** `internal/simipc/messages.go`, `worker/simulate.go`,
  `plugins/wasm/call.go` (`buildFrame`), `plugins/wasm/deploy.go`, `internal/codec/frame.go`, and the
  test worker `internal/testbin/worker/main.go`.
- **Version:** testnet starts from a fresh genesis (the activation runbook, E2-27) and no contract is deployed anywhere, so the
  frame change keeps `runtime_profile_version = 1`.
- **Conformance:** regenerate the execution conformance artifact and the frame fixtures. Copy the
  artifact to zeno `wasm-vm/` and update its digest.
- **Docs:**
  - SPEC §9 states that the producing pillar chooses the momentum, so its hash is grindable and
    **MUST NOT** be used as unbiased randomness;
  - EXECUTOR §7 shows the field in `InputContext`.

## Code references
- zeno-executor: the 76 files `grep -rl '§' --include=*.go .` lists; `runtime/runtime.go` [EXISTS].
- zeno: `phase-1/{SPEC,EXECUTOR,ARCHITECTURE,BRIDGES,DEMOS,CHANGELOG}.md`, `phase-1/clients/`,
  [EXISTS].

## Acceptance criteria
- In zeno-executor, `grep -rnE '§|\b(SPEC|EXECUTOR|ARCHITECTURE|BRIDGES)(\.md)?\b|[Tt]icket|E[12]-[0-9]{2}|OD-[0-9]' --include=*.go .`
  returns nothing.
- Stage C is comment-only: for every file it changes, the source printed with comments stripped is
  identical before and after.
- `deadcode` and `staticcheck -checks U1000` report nothing.
- A contract reading the frame sees the momentum hash of the momentum that positions its input, and
  `zeno_simulate` returns the same receipt as the executed input for the same momentum.
- The regenerated execution artifact is byte-identical in zeno-executor and zeno `wasm-vm/`.
- `go build ./...`, `go vet ./...` and `go test ./...` pass.
- No zeno document presents `OPTIMISTIC`, a watcher, `Challenge`, a fraud referee or a challenge
  window as planned behaviour; reserved ordinals are marked reserved.
- Every audit mismatch is resolved in the documents or listed as a triaged code fix.
