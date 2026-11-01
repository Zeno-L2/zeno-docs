# Ticket E2-04 — Deterministic WASM runtime plugin

**Epic:** 2 — Executor · **Phase:** 1A · **Order:** E2-04

## Goal
Build the deterministic WebAssembly Core 1.0 interpreter as the WASM `Domain` plugin. It turns
`(witnessed pre-state, input)` into `domain.Effects` byte-identically on every node, for all six
Phase 1 input kinds. All WASM-specific logic lives behind `Domain.DecodeInput` and `Domain.Apply`.
This is the highest-risk Phase 1 component (EXECUTOR P-5; SPEC §29). The conformance suite and
determinism harness that freeze its behaviour are E2-05.

## SPEC refs
SPEC §7 (runtime profile, prohibited features, memory, depth, traps), §8 (three-import host ABI),
§9 (execution context, `chain_id`, caller), §10 (gas, `metering_version = 1`), §11 (chunked
deployment, validation, instrumentation), §12 (`ContractID`/`CodeHash`, upgrade policy), §13.3
(reserved keys), §14.3 (`ContractEffects` validation), §15 (status set, receipt), §18.2/§18.5
(deployed-target detection, deposit claim and refund), §21 (size limits), §27.1/§27.1a (frame,
effects blob), §27.4 (outbox cap), §27.7 (`gas_used` in `receiptRoot`). EXECUTOR P-3 (STF purity),
§7 (plugin interface). Artifact `wasm-gas-table-v1.json`/`.md`.

## Depends on
E2-02 (witnessing `StateAccess`), E2-03 (`internal/codec`). E2-06 and E2-07 supply
`domain.InputContext` per input.

## Key work

### Domain seam
- `domain/domain.go`: `Effects` carries `Status codec.Status`, `GasUsed`, `ContractID`,
  `CodeHash`, `AbortCode int32`, `ClaimedDeposit codec.U256`, `[]codec.Event` and
  `[]codec.OutboxMessage`. The codec types are used directly; the seam defines no status type.
- `InputContext{GlobalInputIndex, MomentumHeight, MomentumTime, L1Sender, SourceData}` crosses
  the seam as a plain Go struct. `DecodeInput(ctx InputContext, payload []byte)`.
- `Apply`'s doc comment fixes the halt/status split: a non-nil `error` halts the binary, and
  every deterministic outcome, including every failure, returns as `Effects`. It also states the
  local-store read-and-write carve-out (every read hash-verified, every write memoization only).

### Plugin shell, keys, limits, code store
- `plugins/wasm/wasm.go`: `Config{NetworkName, CodeStorePath}`; `New` errors on an empty name or
  unusable store path and computes `chainID = SHA3-256("ZENON_WASM_L2:" ‖ name)` once.
  `cmd/executor` adds a required `-network` flag and stores code under `<storage-path>/wasm-code`.
- `DecodeInput` decodes via `codec.DecodeInput`. `codec.ErrInvalidKind` yields
  `Input{Unsupported: true}`; any other decode error halts. `SourceData` must be non-empty on kind
  3 and empty on every other kind. Kind 3 verifies the chunk hash and `MaxDeployChunkData`, then
  stores the chunk. Kind 2 first deletes every chunk and pending record left by an earlier attempt
  at the same `code_hash`, then writes the new pending record (E2-15 sets the record key).
- `plugins/wasm/keys.go`: four reserved two-byte keys, matched exactly, never by prefix:
  `0x0001` `CodeHash`, `0x0002` `code_hash`, `0x0003` `upgrade_policy`, `0x0004` `owner`.
  `plugins/wasm/internal/keys` holds the single `Derive(contractID, localKey) =
  SHA3-256(contract_id ‖ local_key)`.
- `plugins/wasm/limits.go`: SPEC limits (`MaxCodeSize`, `MaxChunkCount`, `MaxDeployChunkData`,
  `MaxMemoryPages = 64`, `MaxStackDepth = 512`, `MaxInstrumentedCodeSize`, `MaxReturnData`,
  `MaxExecutionResult`) plus the plugin's structural caps: `MaxLocalsPerFunction`,
  `MaxTotalLocals`, `MaxTableEntries`, `MaxTotalParams`, `MaxGlobals`, `MaxDataSegments`,
  `MaxFunctions`, `MaxTotalElementEntries`, `MaxTypeCount`.
- `plugins/wasm/codestore.go`: disk-only content-addressed store with four entry kinds (deploy
  chunk, pending record, assembled module, instrumented module). Writes are temp-file plus rename;
  each entry carries a SHA3-256 checksum, and a mismatch reads as absent.

### Gas cost model
- `internal/module/gastable.go` embeds `wasm-gas-table-v1.json` and parses it once.
  `OpcodeCost`, `BrTableCost`, `MemoryGrowPageCost`, `InitialMemoryCost`, `HostCost`, `EffectCost`
  are pure lookups.
- The basic-block leader set is data (`TerminatesSegment`): an instruction leads a segment iff it
  is first in a body or follows `end`, `else`, `loop`, `if`, `br`, `br_if`, `br_table`, `return`
  or `unreachable`. The segment that reaches a terminator pays for it. `block`, `call` and
  `call_indirect` do not end a segment.

### Binary reader and validator
- `internal/module/binary.go`, `leb128.go`: single-pass section splitter and canonical LEB128.
- `internal/module/validate.go` returns one named error (`errors.go`) per rule:
  - structure: magic/version, section order, no duplicates, no custom sections, canonical
    LEB128, no trailing bytes, at most one result, no type-index block types;
  - imports: only `env.storage_len`, `env.storage_read`, `env.abort` with exact signatures;
  - exports: `memory` and `run (i32,i32)->i64` required; no start section, no `__zenon_` names,
    no mutable-global export;
  - instructions: deny by default — an opcode absent from the gas table is rejected, with named
    reasons for floats, sign-extension, `0xFC`, typed `select`, references, SIMD, atomics;
  - bounds and caps: index spaces checked against the original module; at most one memory and one
    table; initial memory ≤ 64 pages; every structural cap in `limits.go`; active data and element
    segments in bounds with literal `i32.const` offsets, in 64-bit arithmetic.
- `internal/engine/compile.go`: after `Validate`, the pinned engine compiles the original module
  once and discards it, for Core 1.0 type checking. A rejection is `ErrEngineRejected`.
- `github.com/tetratelabs/wazero` is pinned in `go.mod`. `runtimeConfig()` is the one engine
  configuration for compile and run: interpreter, `CoreFeaturesV1`, 64-page limit, debug info and
  custom sections off, no WASI.

### Deterministic instrumentation
- `internal/module/instrument.go`: `Instrument(original, limits)` is pure, append-only and
  byte-deterministic. It appends globals `__zenon_gas` (i64) and `__zenon_depth` (i32), neither
  exported, and helpers `__zenon_charge`, `__zenon_enter`, `__zenon_exit`, `__zenon_grow`. Only
  `__zenon_gas_get`/`__zenon_gas_set` are exported.
- Each body becomes `call __zenon_enter; block <result> B end; call __zenon_exit`, plus
  `__zenon_exit` before every `return`. A branch to the outermost label lands on the exit.
- Segment charges come from the original instruction stream, so injected code is unmetered.
  `memory.grow` becomes `__zenon_grow`, which returns `-1` past 64 pages before charging.
- The memory limit is normalized to `{min, max: 64}`. Output over 1 MiB fails.
  `CodeHash = SHA3-256(instrumented)`.

### Engine, host ABI, entry point
- `internal/engine/engine.go`: `Run(hc, codeHash, instrumented, frame, gasBudget)` holds a lock,
  takes no caller context, clamps the budget to `MaxInt64`, charges `InitialMemoryCost` before
  instantiation, seeds the ledger with the remainder, writes the frame at offset 0, and calls
  `run(0, frame_len)`. A frame larger than memory is `Trapped`.
- The compiled-module cache is keyed by `CodeHash`, bounded by entries and bytes, and evicted LRU.
  The runtime is rebuilt every `defaultCompilesPerRebuild` compiles to release interned types.
- `internal/engine/host.go`: every host function charges first through the module's own ledger
  (`chargeGas`), then works. `storage_len` returns `-1` or length; `storage_read` returns `-1`,
  `-2` or `L` and charges `10 * ceil(L/32)` before writing; `abort` traps with its code. A key
  over 32 bytes or an out-of-bounds pointer traps. A `StateAccess` error halts.
- `internal/engine/outcome.go`: `classify` in strict order — abort ⇒ `Aborted`; error with ledger
  `< 0` ⇒ `OutOfGas`; other error ⇒ `Trapped`; else `Returned`.

### Calls, effects and deposit settlement (kinds 0/1)
- `plugins/wasm/call.go` reads reserved keys `0x0001` and `0x0002` unconditionally, before any
  code-store lookup. Absent `CodeHash` ⇒ `VALIDATION_FAILED` with full refund. Otherwise it loads
  the instrumented module, or re-derives it from the assembled module, verifying both hashes.
- It builds the §27.1 frame (`caller = SHA3-256("L1_CALLER:" ‖ address)`) with
  `gasBudget = plasma_limit` and calls `Engine.Run`.
- `plugins/wasm/effects.go` maps the outcome, then validates a returned blob in fixed order:
  1. `codec.DecodeContractEffects`, then `GlobalInputIndex` match ⇒ else `RUNTIME_FAULT`;
  2. every event and outbox message attributed to the executing contract ⇒ else `RUNTIME_FAULT`;
  3. reserved-key write ⇒ `UNAUTHORIZED`; duplicate derived key ⇒ `RUNTIME_FAULT`;
  4. `claimed_deposit > deposit_amount`, or 32 contract messages plus a refund ⇒ `RUNTIME_FAULT`;
  5. effect charge (refund uncharged) ⇒ `OUT_OF_GAS`; then `MaxReturnData` and
     `MaxExecutionResult` over the final message set ⇒ `RESULT_TOO_LARGE`.
- Every failure keeps only the refund. `gas_used` is metered gas, the whole budget on
  `OUT_OF_GAS`, and 0 where nothing ran. `r < 0` ⇒ `REVERT` with `AbortCode = int32(-r)`.
- `plugins/wasm/deposit.go` refunds `deposit_amount − claimed` on success and the full deposit on
  failure, as one `L1Withdrawal` to `L1Sender` at `outbox_index` = retained contract messages.

### Deployment and upgrade (kinds 2–5)
- Kinds 2/3 return `SUCCESS` with no effects. Kind ≥ 6 returns `UNSUPPORTED_OPERATION`, gas 0, no
  refund. Kinds 2–5 never charge gas and report `GasUsed = 0`.
- `plugins/wasm/deploy.go` `prepareModule`: pending record (absent or not 64 bytes ⇒ halt); record
  bounds before any chunk lookup ⇒ `VALIDATION_FAILED`; every chunk (missing ⇒ halt); assembled
  hash and length ⇒ `VALIDATION_FAILED`; then `deriveModule`.
- `deriveModule` runs validate, compile original, instrument, compile instrumented. Any failure or
  recovered panic is `VALIDATION_FAILED`. Both module tiers are memoized; a store fault halts.
- Kind 4: `ContractID = SHA3-256(L1Sender ‖ globalInputIndex(8B BE) ‖ chain_id)` from the finalize
  input. It writes `CodeHash`, `code_hash` and `upgrade_policy = IMMUTABLE`, with
  `ReturnData = code_hash ‖ CodeHash`. A finalize reads no state.
- Kind 5 reads all four reserved keys unconditionally. Not deployed ⇒ `VALIDATION_FAILED`. Any
  policy other than one byte `1` with `owner == caller` ⇒ `UNAUTHORIZED`. Otherwise it writes
  `CodeHash`/`code_hash` only, with `ReturnData = old CodeHash ‖ new CodeHash`.

## Decisions
Each is recorded with its D-number in `plugins/wasm/UPSTREAM-AMENDMENTS.md`.
- **Custom sections are rejected, not stripped.** wazero parses a `name` section at decode time
  without bounds, so stripping after the engine compile step leaves the hole open.
- **One gas ledger, host charge first.** Host and injected charges share the module's global, so
  `GasUsed` is exact. A `FunctionListener` cannot refuse a call, so it is not used.
- **Deploy-path failures in validation, compile or instrumentation are `VALIDATION_FAILED`.** They
  are deterministic under the pinned engine. A halt would let one crafted module stop executors.
- **Local-store misses halt; they are never a status.** A node missing data cannot see what other
  nodes see. Committing a status would diverge the post-state.
- **Reserved keys are read unconditionally.** The witness set is committed in `DAHash`, so it must
  not depend on cache warmth or the authorization branch taken.
- **`ContractID` uses the finalize input's `L1Sender`, not the pending record.** Only `L1Sender`
  is consensus-sourced. A stale local record could otherwise derive a divergent `ContractID`.
- **Kind 2 clears earlier attempts.** A surviving chunk would pass per-chunk checks and assemble
  into a divergent module. This assumes decode and apply interleave per input.
- **Effect charge precedes both size checks; the refund is uncharged.** `gas_used` is committed,
  so the order must be fixed. The refund is not a contract-declared effect.
- **Structural caps beyond SPEC.** LEB128 counts are cheap to encode but force allocation at
  compile or instantiation, which the gas table does not price.
- **`gasBudget` clamps to `MaxInt64`.** The ledger is i64; an unclamped seed goes negative.
- **Every Phase 1 deployment is `IMMUTABLE`.** No transport carries the policy declaration. The
  `OWNER_UPGRADEABLE` branch is tested only with seeded reserved keys.

## Code references
- `domain/domain.go`, `cmd/executor/main.go` [EXISTS]
- `plugins/wasm/{wasm,input,keys,limits,codestore,call,effects,deposit,deploy}.go` [EXISTS]
- `plugins/wasm/internal/keys/keys.go` [EXISTS]
- `plugins/wasm/internal/module/{binary,leb128,errors,validate,gastable,instrument}.go`,
  `wasm-gas-table-v1.json`, `testdata/instrument/*.wasm` [EXISTS]
- `plugins/wasm/internal/engine/{compile,engine,host,outcome}.go` [EXISTS]
- `plugins/wasm/UPSTREAM-AMENDMENTS.md` [EXISTS] — consensus-visible decisions and upstream asks.

## Acceptance criteria
- `go build ./...` and `go vet ./...` green. `internal/module` never imports wazero
  (`TestImportBoundary`); `internal/codec` and `internal/state` are consumed, not edited.
- Reserved keys match exactly; key-derivation vectors pass. Code store round-trips, and a corrupt
  checksum reads as absent.
- `DecodeInput` accepts kinds 0–5, maps other kinds to `Unsupported`, and halts on malformed
  bytes, chunk-hash mismatch, oversized chunk and `SourceData`/kind mismatch.
- Gas table: `metering_version == 1`, 104 opcodes, drift check against the vendored artifact,
  every worked example reproduced (`TestArtifactDrift`, `TestWorkedExamples`, `TestLeaderSet`).
- Validator: one fixture per rejection reason asserts its named error (`TestRejectionCases`);
  valid fixtures pass (`TestAcceptanceCases`); engine-only rejections return `ErrEngineRejected`.
- Instrumentation: byte-identical over repeated runs and matches golden hashes. Per-path charge
  equals executed original cost; depth does not leak; depth 513 traps; worst-case expansion fits
  or fails at 1 MiB (`instrument_test.go`, `instrument_exec_test.go`).
- Engine: each host return code end to end; absent read witnessed; charge refuses before work;
  one ledger across sites; out-of-gas beats a coincident trap; warm and cold cache agree.
- Effects: status map walked row by row with `gas_used` per row (`TestSettleStatusMap`); refund
  index, outbox capacity, refund-only oversize, attribution and failure shape covered.
- Deploy: single- and multi-chunk deploys write correct keys and `ReturnData`; absurd
  `chunk_count` never touches the chunk store; missing chunk or record halts; `ContractID` matches
  a fixed vector; warm and cold code stores give identical witness sets; upgrade statuses covered.
