# Ticket E1-03 — Domain registry + `RegisterDomain` lifecycle

**Epic:** 1 — L1 Settlement · **Phase:** 1B · **Order:** E1-03

## Goal
Implement admin-only domain registration, the lifecycle constraints that make the
schema multi-domain-ready while pinning Phase 1 to the degenerate WASM case, and the
per-domain Periphery the registry makes possible. End state: exactly one domain can
be registered (WASM / `L1_NATIVE` / `SINGLE`); reserved values are rejected; the
withdrawal delay, runtime-upgrade delay and approved `DAMode` set are per domain,
and `MaxBatchWithdrawal` is per (domain, asset), each behind its own setter.

## SPEC refs
SPEC §6 (domain model, domain ids and reconfiguration, initial delays, `valueCaps`
window prefix), §6.1 (input-source descriptor), §6.2 (executor set — network-wide,
not per domain), §6.3–§6.5 (profile ladders, legal combinations — reserved), §5.2
(Periphery scopes), §5.3 (`RegisterDomain`, `SetPeripheryConfig`,
`SetDomainConfig`, `SetAssetConfig`, `SetApprovedDAModes`), §5.4 (storage), §18.5
(custodyable assets), §20 (approved `DAMode` set), §23 (time challenges keyed by
scope, hard bounds, no-lowering rule), §27.9.

## Depends on
Tickets E1-01 (storage/constants), E1-02 (admin gate, the network-wide Periphery record
this ticket narrows).

## Key work
- **`RegisterDomain(domainId, domainClass, runtimeKind, stfSpecHash, inputSource,
  execProfile, foreignProfile, profileConfig, chainBinding, minExecutors,
  proposerPolicy, valueCaps, finalityModel, status, withdrawalDelay,
  runtimeUpgradeDelay)`** — admin-only.
  - **Ids (§6).** Assigned sequentially from `1`; `0` names no domain. `domainId`
    equal to the next unassigned id registers; a lower id reconfigures that domain;
    a higher id is rejected.
  - **Reconfiguration (§6).** Reject any change to `domainClass`, `runtimeKind`,
    `inputSource`, `execProfile`, `foreignProfile`, `profileConfig`, `chainBinding`
    or `proposerPolicy`, and reject a reconfiguration identical to the stored
    record and delays, so a repeated registration call fails even while the Phase-1
    clamp pins every fixed field. A change to
    `valueCaps` or `minExecutors` is time-challenged at `AdministratorDelay` under
    the scoped key `RegisterDomain/<domainId>` (§23). `stfSpecHash` changes only
    under the runtime-upgrade delay. `withdrawalDelay` and `runtimeUpgradeDelay` are
    set at registration only; a reconfiguration carrying different delays is
    rejected (`SetDomainConfig` changes them).
  - **Initial delays (§6).** At registration validate `withdrawalDelay`
    (`IsWithdrawalDelayInBounds`), `runtimeUpgradeDelay` against it
    (`IsRuntimeUpgradeDelayValid`) and a non-empty `valueCaps` window against it
    (`IsValueCapWindowValid`); no time challenge.
- **Phase 1 profile clamp (§6.1, §6.5):** accept only `domainClass = EXECUTION`,
  `runtimeKind = WASM`, `inputSource = L1_NATIVE`, `execProfile = ATTESTATION`,
  `foreignProfile = NONE`, `chainBinding` empty, `profileConfig` empty,
  `proposerPolicy = SINGLE`, `minExecutors = 1`, and `finalityModel = DELAY`.
  `inputSource = 2` is retired and rejected like any unlisted value. Custody mode
  has no `RegisterDomain` input; the first `AssetRegistry` writer enforces
  `ESCROW`-only.
- **`valueCaps` (§6).** Empty means no caps and no window check. Otherwise the first
  8 bytes are the window in momentum heights (big-endian `u64`), validated with
  `IsValueCapWindowValid(window, domainWithdrawalDelay)`; the rest stays opaque.
  Phase 1 enforces no cap.
- **Remove `executors` from `domainRecord`.** The executor set is network-wide
  (§6.2, Ticket E1-04); nothing reads the field.
- **Periphery scopes (§5.2, §5.3).**
  - Narrow `SetPeripheryConfig` to the network-wide record. Ticket E1-04 adds the fee
    prices to it; until then the record has no fields of its own and the method
    has nothing to set.
  - `SetDomainConfig(domainId, withdrawalDelay, runtimeUpgradeDelay)` — bounds and
    the relative floor as at registration, plus a non-empty `valueCaps` window
    re-checked against the new delay; `SoftDelay` under key
    `SetDomainConfig/<domainId>`.
  - `SetAssetConfig(domainId, assetId, maxBatchWithdrawal)` —
    `IsMaxBatchWithdrawalInBounds`; reject a value below the one in force for as
    long as the domain is registered, whatever its `status` (§23, §27.9);
    `SoftDelay` under `SetAssetConfig/<domainId>/<assetId>`. A configured value is
    what makes the asset custodyable for the domain (§18.5, Ticket E1-05).
  - `SetApprovedDAModes(domainId, modes)` — replace semantics; reject an empty set
    and any unlisted `DAMode`; `SoftDelay` under `SetApprovedDAModes/<domainId>`;
    stored under its own key prefix. Ticket E1-07 checks a batch's `DAMode` against
    its domain's set.
  - Time challenges use the shared helper with the scoped key string (§23), so a
    change for one domain or asset never restarts another's.
- **Reserved `AssetRegistry` schema (§5.4).** Give the per-(domain, asset) entry the
  full field set — `custodyMode`, the `(foreignChainId, foreignAssetRef,
  decimalsPolicy)` binding as opaque bytes, and the four counters — before it gains
  its first writer, so activating a `MINTED` profile is a spork rather than a
  storage migration.
- Per-domain keying invariant: all later accounting keyed by `domainId` /
  `(domainId, asset)` — this ticket establishes the registry the rest reads.

## Code references
- Ticket E1-01 `DomainRecord` storage [NEW].
- Admin verification pattern: `vm/embedded/implementation/bridge.go`
  (`sendBlock.Address == Administrator`) [EXISTS] (consumed via Ticket E1-02).

## Acceptance criteria
- Registering `EXECUTION`/WASM/`L1_NATIVE`/`ATTESTATION`/`NONE`/`SINGLE` with
  `domainId = 1` succeeds, and every field round-trips; the domain's withdrawal
  delay and runtime-upgrade delay are readable.
- Registration fails for: a reserved or retired `inputSource`, reserved
  `proposerPolicy`, `domainClass ∈ {BRIDGE, MESSAGING}`, `execProfile ≠ ATTESTATION`,
  `foreignProfile ≠ NONE`, non-empty `chainBinding` or `profileConfig`, a
  `finalityModel` other than `DELAY`, an out-of-bounds `withdrawalDelay`, a
  `runtimeUpgradeDelay` below it, a `valueCaps` window below it, a `domainId` of
  `0`, and a `domainId` above the next unassigned id.
- Repeating the registration call with the same `domainId` is rejected, whether it
  is an exact repeat or changes a fixed field or a delay; one changing `valueCaps` or `minExecutors` takes effect only
  after `AdministratorDelay`, observable while pending.
- `SetDomainConfig`, `SetAssetConfig` and `SetApprovedDAModes` take effect only
  after `SoftDelay`; a pending change for domain or asset A survives a change
  submitted for B.
- Lowering a (domain, asset)'s `MaxBatchWithdrawal` is rejected while the domain is
  `ACTIVE` and while it is `PAUSED`.
- `SetApprovedDAModes` with an empty set or an unlisted value is rejected.
- `SetPeripheryConfig` no longer carries the withdrawal delay, `MaxBatchWithdrawal`
  or the runtime-upgrade delay.
