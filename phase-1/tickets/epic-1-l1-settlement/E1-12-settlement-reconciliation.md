# Ticket E1-12 — Settlement reconciliation: fraud-proof stubs, `valueCaps`, dead code, comments

**Epic:** 1 — L1 Settlement · **Phase:** 1C · **Order:** E1-12

## Goal
Remove what the two-phase plan makes obsolete, remove dead code, and make go-zenon's new code
comment-clean. End state:
- Settlement carries no fraud-referee or cooldown records and no `valueCaps`;
- the code added by `feature/merkle-state-root` and `feature/settlement` has no unused declarations;
- no Go file on either branch cites a document section, a ticket or a design id.

## Depends on
None. Runs in parallel with E2-23 parts A, C, D and E; E2-23 part B waits for this ticket.

## Key work

### Fraud-proof stubs (`feature/settlement`)
- Remove from `vm/embedded/definition/settlement.go`:
  - the `fraudRefereeRegistryEntry` ABI variable and its variable-name constant;
  - `FraudRefereeRegistryEntry`, its key builder, `Save` and getter;
  - the `fraudRefereeRegistryPrefix` (`0x10`) declaration. Byte `0x10` stays unassigned and is never
    reused.
- Remove `ExecutorStatusCooldown`. Keep `ExecutorStatusLapsed` (Phase 2 liveness) and the validity-key
  registry.
- Update the tests that list them: `definition/settlement_test.go:377` and
  `implementation/settlement_test.go:494`.

### Remove `valueCaps` (`feature/settlement`)
`valueCaps` bounds the same risk as `MaxBatchWithdrawal`. Settlement stores it and validates its window,
but enforces no cap from it.
- `RegisterDomain` and `SetDomainConfig`: drop the `valueCaps` parameter and its window check
  (`valueCapWindow`, `implementation/settlement.go:135-138, 216-217, 417`).
- `DomainRecord`: drop the field from the ABI variable, the struct and the storage encoding.
- The time-challenge rule: a registered domain's change waits `AdministratorDelay` when `minExecutors`
  changes (`implementation/settlement.go:317`).
- `vm/constants/settlement.go`: drop `IsValueCapWindowValid` and its bounds.
- RPC: drop `valueCaps` from the domain record (`rpc/api/embedded/settlement.go:62, 544`) and the SDK
  fixture.
- Update the ABI dump, `vm/embedded/tests/settlement_test.go`, `settlement_rpc_test.go`, and the
  definition and constants tests.

### Dead code (`feature/settlement`, `feature/merkle-state-root`)
- Scope: the code these two branches add to upstream go-zenon. Upstream code is out of scope.
- Run `deadcode ./...` (golang.org/x/tools) and `staticcheck -checks U1000 ./...`, and filter the
  output to the scope.
- Also check by hand: constants, struct fields and storage values that are written but never read,
  configuration values that nothing consumes, and reserved enum values the two-phase plan no longer
  needs.
- zeno-executor builds against go-zenon (`replace => ../go-zenon`), so an export that looks dead inside
  go-zenon may be live. Before removing an exported declaration, grep zeno-executor for it.
- Remove what is unused. A value read only by a test is unused. Record anything kept, with the reason,
  in the run directory.

### Comment cleanup (`feature/settlement`)
- `vm/embedded/implementation/settlement.go`: one `§` reference.
- `vm/embedded/definition/settlement.go:449`: "(Ticket 15)".
- `vm/constants/settlement.go:51`: "a later parameterization ticket".

### Comment cleanup (`feature/merkle-state-root`)
- 78 `§` references across `common/trie`, `chain/state_tree.go`, `chain/interface.go`,
  `chain/nom/momentum_test.go`, `node/config.go` and `zenon/config.go`. All come from commit `33d7d1e`
  and cite the merkle-state-root documents.
- Fix them on `feature/merkle-state-root`. 

### Rewrite rule
State the rule the code enforces and why, in words. Delete the comment when the code already says
it. Conformance vector ids and `common/trie/testdata` stay.

## Code references
- `vm/embedded/definition/settlement.go`, `vm/embedded/implementation/settlement.go`,
  `vm/constants/settlement.go` [EXISTS].
- `common/trie/*.go`, `chain/state_tree.go`, `chain/interface.go`, `node/config.go`,
  `zenon/config.go` [EXISTS].

## Acceptance criteria
- On both branches, `grep -rnE '§|\b(SPEC|EXECUTOR)(\.md)?\b|Ticket [0-9]|parameterization ticket' --include=*.go .`
  returns nothing outside `p2p/`.
- No identifier named `FraudReferee*`, `fraudReferee*`, `ExecutorStatusCooldown`, `ValueCaps`,
  `valueCap*` or `IsValueCapWindowValid` remains.
- `deadcode` and `staticcheck -checks U1000` report nothing in scope.
- The comment-cleanup stage is comment-only: each changed file's source printed with comments
  stripped is identical before and after.
- `go build ./...`, `go vet ./...` and `go test ./vm/... ./common/... ./chain/...` pass, including
  the ABI dump tests.
