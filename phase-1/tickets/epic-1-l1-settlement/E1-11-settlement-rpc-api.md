# Ticket E1-11 — Settlement RPC API: the `embedded.settlement` namespace

**Epic:** 1 — L1 Settlement · **Phase:** 1B · **Order:** E1-11

## Goal
Give wallets, dApps and operator tooling a read API for Settlement, matching the
other embedded contracts' RPC namespaces. End state: znnd serves
`embedded.settlement.*` with point getters and paged lists for every
publicly readable Settlement record, including pending time challenges, and the
TypeScript SDK can mirror it (`../../../clients/TS-SDK-SETTLEMENT.md` §7).

## SPEC refs
SPEC §5.3 (method list), §5.4 (storage), §6 (domain record), §11 (deploy records,
expiry), §18.3 (conservation counters), §19/§21 (batch entries, finality), §22.4
(fee accounting), §23 (pause records, escape-hatch record, time challenges, "durable,
publicly readable on-chain records").

## Depends on
E1-01–E1-10 (every record this ticket reads). No consensus change and no spork:
this ticket adds read-only RPC methods and touches no state-transition code.

## Key work
- **Namespace.** `rpc/api/embedded/settlement.go` with `SettlementApi`, registered in
  `rpc/apis.go` as `embedded.settlement`, `Public: true`, like `embedded.bridge`.
  Every method reads the frontier context through `api.GetFrontierContext(chain,
  types.SettlementContract)`. Before the Settlement spork activates, every method
  returns empty results rather than an error, so clients need no spork check.
- **Domains.** `GetDomainInfo(domainId)` and `GetAllDomains(pageIndex, pageSize)`.
  `DomainInfo` joins the domain record, its `DomainConfig`, approved `DAMode` set,
  domain pause state, input-sequence cursor, domain received-input counter and
  escape-hatch record. An unregistered `domainId` returns `null`.
- **Batches.** `GetBatch(domainId, batchId)` and `GetBatches(domainId, pageIndex,
  pageSize)`, newest first. Each entry carries the stored fields plus `isFinalized`,
  evaluated with `BatchEntry.IsFinalized(frontierHeight)`. The stored `state` field
  is never used for finality, since no code writes `BatchStateFinalized`.
- **Deploy records.** `GetDeployRecord(domainId, deployer, codeHash)` and
  `GetDeployRecordsByDeployer(domainId, deployer, pageIndex, pageSize)`, the second by
  prefix iteration over `0x0A ‖ domainId ‖ deployer`. Each record carries
  `expiresAtHeight = startHeight + DeploymentExpiryWindow` and `isExpired` at the
  frontier height. Expired records not yet swept are returned with `isExpired =
  true`, since storage still holds them (§11.5).
- **Network state.** `GetPauseState()` returns the global record and every domain
  record. `GetPeripheryConfig()` returns `null` before the first `SetPeripheryConfig`.
  `GetExecutorSet()`, and `GetFeeInfo()` returning `FeeCounters` and the pending
  fee-epoch queue head.
- **Custody.** `GetAssetInfo(domainId, assetId)` joins `AssetConfig`,
  `ConservationCounters` and `CumulativeDepositTotal`. `GetReleaseQueue(domainId,
  pageIndex, pageSize)` returns `ReleaseQueueMeta` and the entries from head to tail.
  `IsOutboxProcessed(outboxId)`.
- **Time challenges.** `GetTimeChallengesInfo()` iterates the time-challenge prefix
  in Settlement storage and returns every entry whose `paramsHash` is non-zero. It
  iterates rather than listing method names as `embedded.bridge` does, because
  Settlement's scoped keys (`SetDomainConfig/<domainId>`,
  `SetAssetConfig/<domainId>/<assetId>`) cannot be enumerated from method names.
  Each entry adds `dueHeight = challengeStartHeight + delay + 1`: the first frontier
  height at which the apply passes, since `TimeChallenge` refuses while
  `challengeStartHeight + delay >= frontierHeight`. The delay comes from the
  method-name prefix of `methodName`. For `RegisterDomain/<domainId>` it is
  `max(MinAdministratorDelay, RuntimeUpgradeDelay)`, an upper bound: the stored
  hash does not say which fields change, and the apply takes `MinAdministratorDelay`
  for a custody change, `RuntimeUpgradeDelay` for an `stfSpecHash` bump, and the
  larger for both. Every delay is read from the configuration at the frontier, so a
  later configuration change moves the real due height.
- **JSON shapes.** Follow the existing embedded APIs: `*big.Int` as decimal strings,
  hashes as hex, addresses as bech32, enumerations as their `u8` value. Paged lists
  return `{count, list}` and cap `pageSize` at `api.RpcMaxPageSize`.
- **Accessors.** Domain ids are dense from 1 to `GetLastDomainId`, and batch ids
  from 0 to `NextBatchId - 1`; Settlement deletes neither record. Both lists page by
  id with point `Get*` reads, and a missing record inside the range is an error, not
  a skip. Add one `definition` iterator, for deploy records by deployer. Reuse every
  existing `Get*` accessor otherwise.

## Out of scope
- Input assignment reads: assigned-index records, ordinal entries, the received-input
  counter's history. The executor reads these through `ledger.getProof`, and a
  send-hash lookup belongs to the executor read API.
- Reserved records: asset registry, message state and the validity-key registry.

## Code references
- `rpc/api/embedded/bridge.go` (`GetTimeChallengesInfo`, paged lists) [EXISTS].
- `rpc/apis.go` (namespace registration) [EXISTS].
- `vm/embedded/definition/settlement.go` (`Get*` accessors, prefix table,
  `GetExecutorSet` and `GetPendingFeesForEpoch` as iterator patterns) [EXISTS].
- `vm/embedded/definition/common.go` (`TimeChallengeKeyPrefix`,
  `timeChallengeKey`) [EXISTS].
- `vm/constants/embedded.go` (`MinSoftDelay`, `MinAdministratorDelay`) [EXISTS].

## Acceptance criteria
- Every method returns the record written by the matching Settlement method in a
  `vm/embedded/tests` spork fixture, field for field.
- `GetBatch` reports `isFinalized = false` one height before `finalizeAtHeight` and
  `true` at it.
- `GetDeployRecordsByDeployer` returns only the named deployer's records for the
  named domain, including an expired, unswept one with `isExpired = true`.
- `GetTimeChallengesInfo` lists a pending `SetAssetConfig/<domainId>/<assetId>`
  challenge with the correct `dueHeight`. An apply at `dueHeight - 1` fails with
  `ErrTimeChallengeNotDue`; an apply at `dueHeight` passes, and the entry is then
  omitted.
- Before Settlement activation every method returns empty results.
- Paging returns each entry exactly once across pages; `pageSize` above the cap is
  rejected.
- `TestDumpContractsABIMethods` unchanged; existing Settlement tests pass.
