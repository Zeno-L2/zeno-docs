# Artifacts — conformance material and copy registry

This folder holds the runtime-agnostic companion documents and conformance artifacts of `../phase-1/SPEC.md`; WASM-specific ones live in `../wasm-vm/`. This README is the registry for both folders. Several files are copies: their source of record lives in a code repository, and zeno-docs carries the copy that readers and other implementations use. Each table below says where each file comes from, which repositories hold a copy, and how each copy is pinned.

Run every command below from the folder that holds the `zeno-executor`, `zeno-client-ts`, `zeno-console`, `zeno-contract-sdk-rs` and `zeno-docs` checkouts. `go-zenon` and `znn-typescript-sdk` are in `../zenon-network/`, a sibling of that folder.

## Documents

| File | Folder | What it is | Status | Used by |
|---|---|---|---|---|
| `proof-format.md` | `artifacts/` | Canonical SMT proof format V1 (SPEC §13.4, §13.7) | Normative | Every SMT proof producer and verifier: go-zenon `common/trie`, zeno-executor, the gateway, `znn-typescript-sdk` and `zeno-client-ts` |
| `smt-implementation-checklist.md` | `artifacts/` | Checklist a new SMT implementation must meet, with `smt-v1-test-vectors.json` | Normative with the vectors | New SMT implementations in Go, Rust or TypeScript |
| `wasm-gas-table-v1.md` | `wasm-vm/` | Prose companion of `wasm-gas-table-v1.json` | Normative | Readers of the gas table |
| `execution-conformance-v1-README.md` | `wasm-vm/` | Provenance, regeneration and coverage table of the execution suite (SPEC §31) | Normative | Copy of zeno-executor `plugins/wasm/testdata/execution-conformance-v1-README.md` |

## Conformance artifacts

### `wasm-vm/execution-conformance-v1.json`

Per-input execution vectors for the WASM domain: frame, receipt, gas, state diff, events, outbox (SPEC §31).

| | |
|---|---|
| Source of record | zeno-executor `plugins/wasm/testdata/execution-conformance-v1.json` |
| Regenerate | `go test ./plugins/wasm -run TestGenerateVectors -update`, then update `vectorFileDigest` in `plugins/wasm/vectors_test.go` with the digest `TestVectorFileDigest` reports |
| Read by | zeno-executor `plugins/wasm/vectors_test.go`, `internal/batch/wasm_test.go`, `internal/gateway/views_test.go`, `worker/simulate_equality_test.go` |
| Copies | `wasm-vm/`, with its README. No other repository holds a copy |
| Changed by | Any change to execution rules, the gas table, the frame or the receipt |

### `artifacts/serialization-conformance-v1.json`

Byte-level vectors for the canonical encodings: commitments, input roots, Merkle proofs, assets, outbox, execution results, deploy (SPEC §19, §20, §24, §27).

| | |
|---|---|
| Source of record | zeno-executor `internal/codec/testdata/serialization-conformance-v1.json` |
| Read by | zeno-executor `internal/codec/vectors_test.go`, `internal/codec/fuzz_test.go`, `internal/ledger/decode_test.go` |
| Copies | `artifacts/`. `zeno-client-ts` `test/fixtures/`, pinned by `SOURCE_SHA256` in `test/conformance.spec.ts`. go-zenon holds no copy: its Settlement decoder tests carry individual vectors (CV-COMMIT-1, CV-INPUTROOT-1, CV-ASSET-1/2) as hex literals |
| Not used by | `zeno-contract` |
| Changed by | Any change to a canonical encoding |

### `artifacts/smt-v1-test-vectors.json`

The SMT suite: roots, inclusion and absence proofs, incremental application (SPEC §13.7).

| | |
|---|---|
| Source of record | go-zenon `common/trie/testdata/smt-v1-test-vectors.json` |
| Read by | go-zenon `common/trie/conformance_test.go`. zeno-executor `internal/state/conformance_test.go` reads go-zenon's file from the pinned module through `go list -m` and keeps no copy |
| Copies | `artifacts/`. `znn-typescript-sdk` `test/fixtures/`, pinned by `SOURCE_SHA256` in `test/state/smt.spec.ts` |
| Changed by | Any change to SMT hashing, layout or proofs |

### `wasm-vm/wasm-gas-table-v1.json`

The opcode gas table, `metering_version = 1` (SPEC §10).

| | |
|---|---|
| Source of record | zeno-executor `plugins/wasm/internal/module/wasm-gas-table-v1.json`, embedded and read by the metering code |
| Read by | zeno-executor `plugins/wasm/internal/module/gastable.go` and `gastable_test.go` |
| Copies | zeno-executor `plugins/wasm/internal/module/testdata/wasm-gas-table-v1.json`, held equal by `TestArtifactDrift`. `wasm-vm/`, with `wasm-gas-table-v1.md` |
| Changed by | Any price or metering rule change. A change also regenerates `execution-conformance-v1.json` |

## Client fixtures

### `zeno-client-ts` `test/fixtures/gateway/gateway-go.json`

Real gateway responses, recorded over a fixed history, that zeno-client-ts replays to test `GatewayClient` and its verification.

| | |
|---|---|
| Source of record | zeno-executor `internal/gateway/sdk_fixture_test.go` `TestDumpGatewaySdkFixture`, which writes the file to `GATEWAY_SDK_FIXTURE_OUT` and is skipped without it. No repository holds a source copy |
| Regenerate | In zeno-executor, with its go-zenon replace resolvable (step 6): `GATEWAY_SDK_FIXTURE_OUT=<absolute path> go test ./internal/gateway -run TestDumpGatewaySdkFixture -count=1` |
| Read by | zeno-client-ts `test/gateway/replay.ts`, for `test/gateway/client.spec.ts` and `test/gateway/verify.spec.ts`. zeno-console `test/replay-client.ts` and `src/core/receipt.test.ts` |
| Copies | zeno-client-ts `test/fixtures/gateway/`. zeno-console `test/fixtures/` |
| Pin | None. Check each copy by regenerating the file and comparing it byte for byte (step 6 below) |
| Changed by | Any change to the gateway's responses, or to the execution rules, the gas table or the receipt the recorded history runs through |

### `zeno-client-ts` `test/fixtures/{vault,counter,counter-v2}.interface.json`

Interface files of the reference vault and the counter example, for the client's interface parser and `zeno-selector-v1` encoder.

| | |
|---|---|
| Source of record | zeno-contract-sdk-rs `examples/vault/artifacts/vault.interface.json`, `examples/counter/artifacts/counter.interface.json` and `examples/counter/artifacts/counter-v2.interface.json`, written by `cargo zeno build` |
| Read by | zeno-client-ts `test/interface-files.spec.ts`. zeno-console reads its vault copy in `src/pages/Deploy.test.ts`, `src/components/DryRunPanel.test.ts`, `src/core/composables/useDryRun.test.ts` and `src/core/composables/useDeploy.test.ts` |
| Copies | zeno-client-ts `test/fixtures/`, each pinned by its `FIXTURE_SHA256` entry in `test/interface-files.spec.ts`. zeno-console `test/fixtures/vault.interface.json`, unpinned |
| Not a source | `examples/counter/golden/*.interface.json` is the SDK's own unit-test expectation of the interface text with a zero `code_hash`; nothing copies it |
| Changed by | Any change to an example contract's methods, events, errors or storage, or to its built module (`code_hash`) |

### `zeno-console` `test/fixtures/vault.wasm`

The reference vault's built module, which zeno-console's deploy and dry-run tests load with its interface file.

| | |
|---|---|
| Source of record | zeno-contract-sdk-rs `examples/vault/artifacts/vault.wasm`, written by `cargo zeno build` |
| Read by | zeno-console `src/pages/Deploy.test.ts`, `src/components/DryRunPanel.test.ts`, `src/core/composables/useDryRun.test.ts` and `src/core/composables/useDeploy.test.ts` |
| Copies | zeno-console `test/fixtures/`, unpinned |
| Changed by | Any change to the vault example's source, the SDK or the build toolchain. A change also changes `vault.interface.json`'s `code_hash` |

## When an artifact changes

1. Regenerate it at its source of record, never by hand.
2. Update the pin in the source repository: `vectorFileDigest`, or the `TestArtifactDrift` copy for the gas table.
3. Copy it byte for byte to every copy in its table, and update each copy's pin.
4. Copy it to its zeno-docs folder, and its README or `.md` companion with it.
5. Check that every copy has the same SHA-256:

```sh
shasum -a 256 \
  zeno-executor/plugins/wasm/testdata/execution-conformance-v1.json \
  "zeno-docs/wasm-vm/execution-conformance-v1.json"
shasum -a 256 \
  zeno-executor/plugins/wasm/testdata/execution-conformance-v1-README.md \
  "zeno-docs/wasm-vm/execution-conformance-v1-README.md"
shasum -a 256 \
  zeno-executor/internal/codec/testdata/serialization-conformance-v1.json \
  zeno-client-ts/test/fixtures/serialization-conformance-v1.json \
  "zeno-docs/artifacts/serialization-conformance-v1.json"
shasum -a 256 \
  ../zenon-network/go-zenon/common/trie/testdata/smt-v1-test-vectors.json \
  ../zenon-network/znn-typescript-sdk/test/fixtures/smt-v1-test-vectors.json \
  "zeno-docs/artifacts/smt-v1-test-vectors.json"
shasum -a 256 \
  zeno-executor/plugins/wasm/internal/module/wasm-gas-table-v1.json \
  zeno-executor/plugins/wasm/internal/module/testdata/wasm-gas-table-v1.json \
  "zeno-docs/wasm-vm/wasm-gas-table-v1.json"
shasum -a 256 \
  zeno-contract-sdk-rs/examples/vault/artifacts/vault.interface.json \
  zeno-client-ts/test/fixtures/vault.interface.json \
  zeno-console/test/fixtures/vault.interface.json
shasum -a 256 \
  zeno-contract-sdk-rs/examples/counter/artifacts/counter.interface.json \
  zeno-client-ts/test/fixtures/counter.interface.json
shasum -a 256 \
  zeno-contract-sdk-rs/examples/counter/artifacts/counter-v2.interface.json \
  zeno-client-ts/test/fixtures/counter-v2.interface.json
shasum -a 256 \
  zeno-contract-sdk-rs/examples/vault/artifacts/vault.wasm \
  zeno-console/test/fixtures/vault.wasm
```

6. Check the gateway fixture, which has no committed source, by regenerating it and comparing it with each copy. The dump builds zeno-executor, whose `go.mod` replaces go-zenon with `../go-zenon`. It therefore needs a go-zenon checkout at `zeno-executor/../go-zenon`, or `GOFLAGS=-modfile=<file>` naming a copy of `go.mod` (with its `.sum`) whose replace points at a go-zenon checkout. Each `cmp` prints nothing when its copy is current:

```sh
rm -f "${TMPDIR:-/tmp}/gateway-go.json"
(cd zeno-executor && GATEWAY_SDK_FIXTURE_OUT="${TMPDIR:-/tmp}/gateway-go.json" go test ./internal/gateway -run TestDumpGatewaySdkFixture -count=1)
cmp "${TMPDIR:-/tmp}/gateway-go.json" zeno-client-ts/test/fixtures/gateway/gateway-go.json
cmp "${TMPDIR:-/tmp}/gateway-go.json" zeno-console/test/fixtures/gateway-go.json
```
