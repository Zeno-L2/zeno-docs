# Ticket E2-24 — Reference vault contract (example and testnet demo)

**Epic:** 2 — Executor · **Phase:** 1C · **Order:** E2-24

## Goal
Implement the **reference vault** WASM contract (SPEC §18.1a, §32): custody with
no application logic beyond credit and debit. It has two uses:
- the production-quality example that developers copy to build contracts that
  hold value, such as DEXes, escrows and bridge pools;
- the contract behind every Ticket E2-27 E2E flow and the testnet deposit and
  withdrawal demos.

End state: an `IMMUTABLE` vault module whose deposit/withdraw round-trip passes
the deposit conformance vectors. No mainnet deployment is planned.

**Placement:** an ordinary L2 contract inside the WASM domain — deployed through
the normal §11 path, executed by the executor, balances in that domain's SMT.
It is **not** Settlement code and has **no protocol privileges**. Per-domain by
construction: a future runtime/domain ships its own equivalent for its runtime.

## SPEC refs
SPEC §18.1a (contract-managed balances; the reference vault), §18.5 (payable
deposit claim/refund), §17 (withdrawal outbox), §8 (three-import ABI), §12 (`IMMUTABLE` policy), §32 (implementation task list).

## Depends on
Ticket E2-04 (runtime to build/test against; the vault is an ordinary WASM module).
Consumed by Ticket E2-27 (E2E fixture).

## Key work
- Vault module (any language compiling to Core 1.0 WASM within the §7 profile):
  - **Payable deposit** (via `Deposit` or payable `CallL2`): credit `caller` by
    the full `deposit_amount` per (asset) in vault state; return
    `claimed_deposit = deposit_amount` (no refund remainder in the normal path).
  - **Withdraw(asset, amount, recipient_l1)**: debit `caller`'s vault balance
    (fail `REVERT` on insufficient balance) and emit one `L1_WITHDRAWAL` outbox
    message (§27.4).
  - **Balance read** for clients through the gateway's `storage` view
    (EXECUTOR.md §21.4), with the key layout published in `vault.interface.json`
    in the `contract-sdk.md` D-9 format. `ContractMetadata` (§27.12) carries only
    the upgrade policy and owner, so the layout cannot live there.
- Deploy policy `IMMUTABLE`; no owner. State layout: one derived key per
  `(account, asset)` balance; values `u256`.
- Unit tests in zeno-contract-sdk-rs `examples/vault/src/tests.rs`, run by `cargo zeno test` through zeno-executor's `zeno-exec` (the Ticket E2-04 runtime): deposit credit, partial/failed
  deposit refund behaviour (a deliberately failing call path exercises the
  §18.5 auto-refund), withdraw + outbox emission, insufficient-balance revert,
  determinism across two builds (`cargo zeno build --check`).
- Publish `vault.wasm`, `vault.interface.json` and `vault.hashes.json` (`code_hash`, `CodeHash`, metadata) in zeno-contract-sdk-rs `examples/vault/artifacts/` at a release tag, as the deployment fixture for Ticket E2-27's devnet run.

## Code references
- zeno-contract-sdk-rs `examples/vault/` (`src/lib.rs`, `src/tests.rs`, `artifacts/`) on the Rust contract SDK (`../../../wasm-vm/contract-sdk.md`) [NEW]; zeno-executor `cmd/zeno-exec` [NEW].
- Ticket E2-04 WASM runtime plugin [NEW]; Ticket E2-03 outbox/state-diff encodings
  [NEW].
- `../SPEC.md` §18.1a informative note (vault semantics) [spec-defined].

## Acceptance criteria
- EXEC-DEP-1 (partial claim + refund) reproduced against a vault variant with a
  partial-claim test hook; EXEC-DEP-3 (undeployed target full refund) covered by
  the harness; the normal vault path claims in full with no refund outbox.
- Deposit → withdraw → `RelayMessage` registration → `ReleaseWithdrawals` release
  round-trips value to L1 in the Ticket E2-27 integration environment.
- Module passes §11.3 validation/instrumentation unchanged (no prohibited
  features, no custom sections — strip the name section at build), stays within
  every §25.1 structural cap, and its instrumented size is within the §25
  ceiling.
