# R1 — Which zkVM?

**Status:** Findings collected; decision open
**Spec:** `../../phase-2/SPEC.md` §5 R1 (question, method, data to collect)
**Answers:** the zkVM choice every other item builds on; Q-5 (who runs provers); the S3 condition (prover on commodity hardware or an open prover network)

Research date: 2026-10-23. Every external claim carries its URL and the version or date it applies to. Vendor-reported numbers are marked (vendor-reported). Anything not verified directly is marked (inferred).

## Problem restatement

Phase 2 needs one zkVM to prove each batch's state transition before Settlement accepts it (`phase-2/SPEC.md` §1, §4.3). The zkVM must run one Rust crate (`zeno-stf`) as a guest, including a WASM interpreter (wasmi) that executes integer-only WASM Core 1.0 contracts (DT-3, §4.2). Its final proof must be a succinct proof that a Go embedded contract in go-zenon can verify deterministically and cheaply, and that fits with its public outputs in one account-block (`MaxDataLength` = 16 KiB) (DT-5, R4). Proving must be available to anyone, either on commodity hardware or through an open prover network, for the permissionless fallback S3 to hold (§4.5). The same zkVM also serves R6 (ed25519 pillar signatures) and R7 (recursive verification of an Ethereum light-client proof). This document collects the evidence to make that choice; it does not make it.

## Current state

No zkVM code exists in any of the four repos. The facts below are what a zkVM must fit.

1. **Settlement accepts batches without a proof.** `decodeBatchCommitment` requires a trailing u32 `proofData` length of exactly 0 (`go-zenon/vm/embedded/implementation/settlement.go:1444-1447`), and `ProofData` is set to empty (`settlement.go:1422`). `SubmitBatch` is priced at `EmbeddedSimple` (`settlement.go:1616-1618`), which is 2.5 × 21 000 = 52 500 plasma (`vm/constants/plasma.go:31`). The proof slot already exists in the wire format; its verification does not.
2. **Size budget.** Account-block data is capped at `MaxDataLength` = 16 384 bytes (`vm/constants/plasma.go:42`, enforced at `vm/plasma.go:142`). The commitment is capped at `MaxOnChainCommitmentSize` = 15 KiB (`vm/constants/settlement.go:7`). The fixed commitment is 303 bytes plus up to `MaxAssetsPerBatch` = 16 asset-flow entries of 96 bytes (`settlement.go:1204`, `vm/constants/settlement.go:16`), so a worst-case commitment is 1 839 bytes before the proof. That leaves about 13.5 KiB for proof bytes and any R9 additions (computed from the cited constants).
3. **Hash function everywhere is SHA3-256 (FIPS-202), not Keccak-256.** `crypto.Hash` is `sha3.New256()` (`go-zenon/common/crypto/hash.go:9-10`); `Keccak256` exists separately (`hash.go:25-26`). The SMT uses it for leaves and interior nodes (`go-zenon/common/trie/hash.go:47-59`), and the artifact checklist says "Not Keccak-256" (`zeno-docs/artifacts/smt-implementation-checklist.md:6`, `artifacts/smt-v1-test-vectors.json:3`). The executor STF also uses `crypto.Hash` for the outbox and receipt Merkle trees, key derivation and code hashes (`zeno-executor/internal/codec/merkle.go:131-135`, `internal/codec/receipt.go:181`, `plugins/wasm/internal/keys/keys.go:16`, `plugins/wasm/keys.go:53`). So SHA3-256 is the dominant hash in the guest, not just in the SMT.
4. **Signatures are ed25519 over a SHA3-256 hash.** Account-blocks and momentums carry an `ed25519.PublicKey` and a signature over the 32-byte block hash (`go-zenon/chain/nom/account_block.go:118-119`, `chain/nom/momentum.go:55-56`). Verification is `ed25519.Verify(pubkey, hash, sig)` (`go-zenon/wallet/crypto.go:59-63`, called at `verifier/momentum.go:443` and `verifier/account_block.go:445`). Each momentum carries exactly one signature, from its producing pillar; go-zenon has no multi-signature finality object (no match for "finali" under `consensus/`). Pillars therefore sign momentums with plain ed25519 (feeds R6).
5. **No pairing or SNARK library in go-zenon.** `go.mod` lists no gnark, BN254 or BLS12-381 dependency (`go-zenon/go.mod:5-33`); a search for `bn256|bn254|gnark|bls12` finds nothing. go-zenon targets Go 1.23 (`go.mod:3`). R4 must add a verifier dependency.
6. **Phase 1 WASM runtime is wazero in Go.** `zeno-executor/go.mod:8` pins `github.com/tetratelabs/wazero v1.8.2`. The Rust guest plan (wasmi) has no code yet (`phase-2/SPEC.md` §4.2).
7. **WASM profile is narrow.** Core 1.0 MVP only, no floats, SIMD, threads, sign-extension or `trunc_sat` ops (`phase-1/SPEC.md:515-532`). Memory is capped at `MaxMemory` = 4 MiB (64 pages) and stack at 512 frames (`phase-1/SPEC.md:1330-1331`). Code is capped at 256 KiB pre-instrumentation (`phase-1/SPEC.md:1326`). Only three host imports exist: `storage_len`, `storage_read`, `abort` (`phase-1/SPEC.md:768`).
8. **Worst-case batch is large.** `MaxBatchInputs` = 64 (`vm/constants/settlement.go:10`) and `MaxInputGas` = 1 000 000 000 (`vm/constants/settlement.go:63`, marked "pre-mainnet; finalize before mainnet"). The spec notes that lowering `MaxInputGas` later is a breaking change to the validity-proof program (`phase-1/SPEC.md:1373`).
9. **Contract SDK toolchain.** Contracts build with Rust 1.99.1 for target `wasm32v1-none`, edition 2024 (`zeno-contract-sdk-rs/rust-toolchain.toml:2-3`, `Cargo.toml:7`). This is the contract side, not the guest side; the guest toolchain is whatever the zkVM pins.

## Constraints

1. **Final proof must be a pairing SNARK or similar, not a raw STARK.** Proof plus commitment must fit `MaxOnChainCommitmentSize` = 15 KiB inside a 16 KiB account-block (`vm/constants/settlement.go:7`, `vm/constants/plasma.go:42`). Raw STARK proofs are far larger: RISC Zero's succinct receipt is ~200 kB ([RISC Zero recursion docs](https://dev.risczero.com/api/recursion)) and OpenVM 2.0's STARK is under 300 kB ([Axiom blog, 2026-07-10](https://axiom.xyz/blog/openvm-2-production)). Only a wrapped proof fits, for example SP1's Groth16 (~260 B) or PLONK (~868 B) ([SP1 proof types](https://docs.succinct.xyz/docs/sp1/generating-proofs/proof-types)).
2. **Verification runs in every go-zenon node, in Go, deterministically.** Settlement is an embedded contract (`phase-2/SPEC.md` DT-5, §4.1). The verifier should be pure Go: the only cgo in go-zenon is the `cmd/libznn` library wrapper (`cmd/libznn/main_libznn.go`), not the node path (a search for `import "C"` finds nothing else). It must also be deterministic and priced in plasma. `SubmitBatch` is currently a flat 52 500 plasma (`settlement.go:1616-1618`). R4 owns pricing.
3. **Go toolchain version.** go-zenon is on Go 1.23 (`go-zenon/go.mod:3`). SP1's own Go verifier module requires Go 1.24.0 and gnark v0.14.0 through a `replace` to a fork (`github.com/p4u/gnark`) ([sp1 gnark-ffi go.mod, dev branch, fetched 2026-10-23](https://raw.githubusercontent.com/succinctlabs/sp1/dev/crates/recursion/gnark-ffi/go/go.mod)). Whether upstream gnark verifies SP1 proofs without the fork is unverified. Any gnark-based verifier may force a go-zenon Go upgrade (inferred).
4. **SHA3-256 must be fast in the guest.** Every STF commitment hashes with SHA3-256 (Current state item 3). A zkVM whose Keccak precompile exposes only a fixed Keccak-256 digest, rather than the raw Keccak-f[1600] permutation, cannot accelerate SHA3-256 without a separate code path. The hash is frozen once `StateRootSpork` activates (`common/trie/hash.go:8-10`), so changing it is a Q-3 decision, not a zkVM detail.
5. **Guest must run wasmi over integer-only WASM MVP.** Contracts are Core 1.0 with no floats (`phase-1/SPEC.md:515-532`). wasmi itself needs `alloc` and compiles to `no_std` (inferred from the wasmi crate's documented `no_std` support; not checked against a zkVM target here). A zkVM guest must support `no_std` + `alloc` Rust or a full `std` target.
6. **Worst-case trace size.** `MaxBatchInputs × MaxInputGas` = 64 × 10⁹ gas units per batch (`vm/constants/settlement.go:10,63`). The cycle cost per gas unit is R2's question; the zkVM must handle continuations (sharding a long execution) for any non-trivial batch (inferred).
7. **Permissionless proving (S3).** S3 holds only if anyone can produce a proof without special hardware or through an open prover network (`phase-2/SPEC.md` §4.5, Q-5).
8. **Guest portability.** The risk table asks to "keep the guest portable across zkVMs" (`phase-2/SPEC.md` §8). Precompile access through patched RustCrypto crates (same API natively and in the guest) supports this; zkVM-specific syscalls do not.
9. **Licence.** go-zenon is GPL-3.0 (`go-zenon/LICENSE:1-2`). MIT and Apache-2.0 verifier code can be linked into a GPL-3.0 work; the reverse does not hold, so a verifier under a licence incompatible with GPL-3.0 (for example a source-available or non-commercial licence) cannot ship in go-zenon. The guest and prover run off-chain and are not bound by go-zenon's licence (inferred; this is a reading of the licences, not legal advice).

## Options

Four candidates are serious enough to compare: SP1 Hypercube, RISC Zero, OpenVM, and zkWasm (the non-RISC-V route). The others surveyed are recorded at the end of this section with the reason they are not presented as options. Every candidate below is MIT and/or Apache-2.0, so licence does not separate them.

### Option A: SP1 Hypercube (Succinct), SP1 v6

**Status and maintenance.** SP1 v6 is the Hypercube line, "a new multilinear proof system" ([SP1 intro, fetched 2026-10-23](https://docs.succinct.xyz/docs/sp1/introduction)). Hypercube went live on mainnet on 2026-02-19 ([Succinct blog](https://blog.succinct.xyz/sp1-hypercube-is-now-live-on-mainnet/)). Release cadence is about two weeks: v6.3.0 (20 Jun) through v6.10.0 (23 Oct 2026) ([releases](https://github.com/succinctlabs/sp1/releases)). The guest ISA is RV64IM at SP1 6.3.1 ([Ethproofs zkVMs](https://ethproofs.org/zkvms)). Licence: MIT and Apache-2.0 ([SP1 intro](https://docs.succinct.xyz/docs/sp1/introduction)).

**Precompiles.** Syscalls in `crates/zkvm/lib/src/lib.rs` (dev branch, fetched 2026-10-23, [source](https://raw.githubusercontent.com/succinctlabs/sp1/dev/crates/zkvm/lib/src/lib.rs)):

- SHA-256: `syscall_sha256_extend`, `syscall_sha256_compress`. No SHA-512 syscall exists.
- Keccak: `syscall_keccak_permute(state: *mut [u64; 25])`, the raw Keccak-f[1600] permutation.
- ed25519: `syscall_ed_add`, `syscall_ed_decompress`, reached through the patched `curve25519-dalek` 4.1.3 ([precompiles page, SP1 v6](https://docs.succinct.xyz/docs/sp1/optimizing-programs/precompiles)).
- BLS12-381 and BN254: point add, double, decompress, plus Fp and Fp2 add, sub, mul. Also secp256k1, secp256r1, uint256 mulmod, Poseidon2.

**SHA3-256 (thread 1): accelerated.** SP1's patched RustCrypto `sha3` 0.11.0 routes every `Sha3HasherCore` permutation through `crate::succinct::keccak_permute` when `target_os = "zkvm"` and `target_vendor = "succinct"`. The gate depends only on the target, not on the padding constant, so `Sha3_256` uses the precompile ([sha3 block_api.rs, tag patch-sha3-0.11.0-sp1-6.0.0](https://raw.githubusercontent.com/sp1-patches/RustCrypto-hashes/patch-sha3-0.11.0-sp1-6.0.0/sha3/src/block_api.rs)). The same tag's `lib.rs` names `syscall_keccak_permute` in a comment. Cycle cost per SHA3-256 call is not published (R3).

**ed25519 (thread 2).** Curve operations are accelerated; SHA-512 inside ed25519 verification runs in software, because no SHA-512 syscall exists (inferred from the syscall list). No published cycle count per verification was found.

**SNARK wrapping (thread 3).**

- Groth16 over BN254: ~260 bytes, ~270k EVM gas. PLONK over BN254: ~868 bytes, ~300k gas, about 1m30s longer than compressed ([proof types](https://docs.succinct.xyz/docs/sp1/generating-proofs/proof-types), vendor-reported).
- Proof bytes on the wire: 4-byte Groth16 vkey-hash prefix, then 32-byte `exit_code`, `vk_root`, `proof_nonce`, then the raw gnark proof ([sp1-verifier groth16/mod.rs, dev](https://raw.githubusercontent.com/succinctlabs/sp1/dev/crates/verifier/src/groth16/mod.rs)).
- Five public inputs: `sp1_vkey_hash`, `committed_values_digest`, `exit_code`, `vk_root`, `proof_nonce`. The verifier checks `vk_root` against a pinned constant (same source).
- `committed_values_digest` = SHA-256 of the committed public-value bytes with the top three bits of the first byte cleared (`result[0] &= 0x1F`). The verifier also accepts a Blake3 variant ([sp1-verifier utils.rs](https://raw.githubusercontent.com/succinctlabs/sp1/dev/crates/verifier/src/utils.rs)). Settlement can rebuild the digest with Go's standard `crypto/sha256`.
- **Go verifier exists.** `crates/recursion/gnark-ffi/go/sp1/verify.go` has `VerifyGroth16` and `VerifyPlonk`. They call gnark's `groth16.Verify` or `plonk.Verify` on BN254 with the five inputs ([verify.go, dev](https://raw.githubusercontent.com/succinctlabs/sp1/dev/crates/recursion/gnark-ffi/go/sp1/verify.go)). Its module is Go 1.24.0, gnark v0.14.0, with `replace github.com/consensys/gnark => github.com/p4u/gnark` ([go.mod](https://raw.githubusercontent.com/succinctlabs/sp1/dev/crates/recursion/gnark-ffi/go/go.mod)). The Groth16 wrap itself is produced by gnark, so the proof and key formats are native gnark (inferred from the gnark-ffi layout).
- A Rust `no_std` verifier (`sp1-verifier`) covers Groth16, PLONK and compressed proofs ([README](https://raw.githubusercontent.com/succinctlabs/sp1/dev/crates/verifier/README.md)).

**Trusted setup.** PLONK uses the Aztec Ignition universal setup (rated green by L2BEAT). The Hypercube v6.1.0 Groth16 phase 2 had 18 participants from 8 organisations "without public calls to participate", rated red by L2BEAT ([L2BEAT SP1 Hypercube](https://l2beat.com/zk-catalog/sp1hypercube), fetched 2026-10-23). L2BEAT reports 98 bits of proven security from the EF soundcalc (same source).

**Hardware (thread 4).** Local proving: 16+ CPU cores; 16 GB+ for core and compressed; 16 GB+ for Groth16 (wrap needs ~14 GB); 64 GB+ for PLONK (wrap needs ~60 GB). GPU: CUDA compute capability ≥ 8.0, 24 GB+ VRAM, Linux x86_64 only ([hardware requirements](https://docs.succinct.xyz/docs/sp1/getting-started/hardware-requirements), fetched 2026-10-23). CPU-only proving works; the docs "highly recommend" the network for non-trivial programs (same source). Vendor-reported: 99.7% of 954 Ethereum L1 blocks proved under 12 s on 16 RTX 5090s ([blog, 2025-11-18](https://blog.succinct.xyz/real-time-proving-16-gpus/)); the post does not say whether the wrap is included.

**Prover network (thread 4).** The Succinct Prover Network runs a reverse auction. Requesters pay in PROVE: a dynamic base fee plus an auction-set PROVE-per-PGU price, settled on Ethereum ([auction](https://docs.succinct.xyz/docs/protocol/spn/auction)). Provers must stake PROVE; the docs call proving "open and permissionless" but aimed at operators "with extensive expertise" ([provers intro](https://docs.succinct.xyz/docs/provers/introduction)). No standing price is published; Succinct also sells "reserved capacity" ([network intro](https://docs.succinct.xyz/docs/sp1/prover-network/intro)).

**WASM in the guest (thread 5).** Two public data points:

- vApps paper, Table 5, SP1 (version unstated): wasmi costs 20× native on Fibonacci-1k (347 000 vs 16 947 cycles) and 31× to 32× on Fibonacci-10k and 100k ([arXiv 2504.14809v5, 2025-04-30](https://arxiv.org/html/2504.14809)).
- wasmi-sp1-bench, Wasmi 2.0.0 on SP1 6.8.1, `fib(10000)`: 1.79M to 3.28M cycles depending on wasmi dispatch features. Local CPU core proof took 200 s to 490 s on a 15 GiB machine with a reduced shard size ([results.md](https://raw.githubusercontent.com/imikushin/wasmi-sp1-bench/main/results.md), third-party, undated).

**Recursion and aggregation (thread 6).** A guest verifies SP1 proofs with `verify_sp1_proof(vkey, public_values_digest)`; inner proofs must be compressed. The docs say aggregation adds "some small overhead" and that a single proof is cheaper than many aggregated ([proof aggregation](https://docs.succinct.xyz/docs/sp1/writing-programs/proof-aggregation)). SP1 Helios is an Ethereum light client with a Zellic audit report in the repo ([sp1-helios](https://github.com/succinctlabs/sp1-helios)) and an OpenZeppelin audit dated 2025-05-12 ([OpenZeppelin](https://www.openzeppelin.com/news/sp1-helios-audit)).

**Soundness record (thread 7).** Six GitHub advisories ([advisories](https://github.com/succinctlabs/sp1/security/advisories), fetched 2026-10-23):

- High, 2026-10-11: GHSA-p7v2-jjv3-8p8f, V6 native Rust verifier, mismatched tensor lengths.
- High, 2026-04-11: GHSA-63x8-x938-vx33 (CVE-2026-40323), V6 recursion circuit row-count binding gap; affects 6.0.0 to 6.0.2, fixed in 6.1.0 ([Mend CVE summary](https://mend.io/vulnerability-database/CVE-2026-40323/), secondary).
- High, 2025-06-03: Plonky3 and Rust verifier checks. High, 2025-01-15: missing verifier checks and Fiat-Shamir observations. High, 2024-10-28: BabyBear range checks. Moderate, 2024-11-08: cumulative sum observation.

Audits listed in the repo: Cantina, Code4rena, Zellic (incl. `hypercube-zellic.pdf`), Kalos, rkm0959, Veridise, and an SP1 v4 report ([audits](https://github.com/succinctlabs/sp1/tree/main/audits)). All 62 RISC-V opcodes are formally verified against the Sail spec with Nethermind and the EF (vendor-reported, [blog 2026-02-19](https://blog.succinct.xyz/sp1-hypercube-is-now-live-on-mainnet/)). The January 2025 disclosure drew public criticism over its timeline ([Succinct update](https://blog.succinct.xyz/sp1-security-update-1-27-25/); [critique](https://nockchain.org/writings/succinct-vs-fede)).

**Production.** L2BEAT lists the Groth16 verifier as used by X Layer, ApeChain, Fluent and Appchain, and the PLONK verifier by about 15 chains including Base, Mantle, Celo and Taiko ([L2BEAT](https://l2beat.com/zk-catalog/sp1hypercube)).

**Tradeoffs.**

- For: SHA3-256, ed25519 curve ops and BLS12-381 are accelerated through patched crates, so the guest code stays portable. An existing Go verifier uses gnark-format proofs. It has the fastest release cadence, the widest production use, and an open network.
- Against: the Go verifier needs Go 1.24 and a forked gnark. The Groth16 setup is rated red. Two High advisories in V6 within six months mean frequent mandatory upgrades. A fix to the recursion or wrap circuit changes the pinned `vk_root` or the Groth16 key, so each one is a verifying-key rotation in Settlement under the administrator delay (§4.6) (inferred). The network requires PROVE on Ethereum, which Zeno executors do not otherwise hold. PLONK avoids the red setup but its wrap needs ~60 GB of RAM.

### Option B: RISC Zero (R0VM), zkVM 3.0.x

**Status and maintenance.** The latest stable is v3.0.6 (17 Jul 2026), a toolchain fix. v3.0.5 (3 Feb) backported several security fixes. A v5.0.0-rc.1 pre-release (15 Jan) lists only a build fix ([releases](https://github.com/risc0/risc0/releases)). The last commit on `main` is 2026-07-20, with roughly monthly commits since March ([commits API](https://api.github.com/repos/risc0/risc0/commits?per_page=10)). Ethproofs lists R0VM 3.0.3 under "inactive" zkVMs ([Ethproofs](https://ethproofs.org/zkvms)). The ISA is RV32IM (same source). Licence: Apache-2.0 ([repo API](https://api.github.com/repos/risc0/risc0)).

**Precompiles** (docs version 3.0, [precompiles](https://dev.risczero.com/api/zkvm/precompiles)): `sha2` (0.9.9 to 0.10.8), `tiny-keccak` 2.0.2 (needs the unstable flag), `curve25519-dalek` 4.1.x (for `ed25519-dalek`), `k256`, `p256`, `rsa`, `substrate-bn` (BN254), `bls12_381` 0.8.0, `blst`, `crypto-bigint`, `c-kzg`. No RustCrypto `sha3` patch is listed.

**SHA3-256 (thread 1): accelerated through `tiny-keccak` only.** The patched `KeccakF::execute` calls `risc0_keccak_update(&mut [u64; 25])` on `target_os = "zkvm"` ([keccakf.rs, tag tiny-keccak/v2.0.2-risczero.0](https://raw.githubusercontent.com/risc0/tiny-keccak/tiny-keccak/v2.0.2-risczero.0/src/keccakf.rs)). The `keccakf` module is shared by the `keccak` and `sha3` features ([lib.rs, same tag](https://raw.githubusercontent.com/risc0/tiny-keccak/tiny-keccak/v2.0.2-risczero.0/src/lib.rs)), so `tiny_keccak::Sha3::v256` is accelerated (inferred from the shared permutation). The STF would have to hash with `tiny-keccak` rather than RustCrypto `sha3` to get this (inferred).

**ed25519 (thread 2).** Through patched `curve25519-dalek` (same precompiles page). Cycle cost not found.

**SNARK wrapping (thread 3).** Groth16 over BN254 only. L2BEAT says the wrap uses Snarkjs, and that the `control_root` public input lets the recursion circuits change without a new setup ([L2BEAT RISC Zero](https://l2beat.com/zk-catalog/risc0)). Five public inputs: `control_root` split into two field elements, the claim digest split into two, and `bn254_control_id`. The seal decodes to A, C (G1) and B (G2) ([risc0-groth16 verifier.rs, main](https://raw.githubusercontent.com/risc0/risc0/main/risc0/groth16/src/verifier.rs)). Seal size is 256 bytes (inferred from uncompressed BN254 points; not confirmed in source). Settlement would recompute the claim digest from the image ID and the SHA-256 journal digest (inferred). **No Go verifier** from RISC Zero was found. A generic gnark Groth16 verifier would need the snarkjs key converted (inferred; R4 to confirm). Succinct (STARK) receipts are ~200 kB ([recursion](https://dev.risczero.com/api/recursion)).

**Trusted setup.** Phase 1 reuses the Polygon zkEVM ceremony (55 contributions), phase 2 had 238 participants; L2BEAT rates it yellow ([L2BEAT](https://l2beat.com/zk-catalog/risc0)).

**Hardware (thread 4).** Local CPU and GPU proving are supported ([proving options, 3.0](https://dev.risczero.com/api/generating-proofs/proving-options)), with optional CUDA ([risc0-groth16 docs](https://docs.rs/risc0-groth16/)). Docs for 1.2 to 2.3 say the Groth16 step (stark2snark) runs only on x86 and needs Docker ([proving options 2.3](https://dev.risczero.com/api/2.3/generating-proofs/proving-options)). The 3.0 page is silent on that limit. No published RAM minimum was found.

**Prover network (thread 4).** Boundless: a reverse Dutch auction where requesters deposit funds to cover the maximum price, priced in ETH in the example. Provers lock ZKC collateral (example 5 ZKC) that is slashed on a missed deadline ([proof lifecycle](https://docs.boundless.network/developers/proof-lifecycle)). Collateral is typically about 10× the request's maximum fee ([ZKC intro](https://docs.boundless.network/zkc/introduction)). Provers fulfil batches with one aggregated Groth16 root and give each request a Merkle inclusion seal. The request `selector` can ask for a plain Groth16 instead (same lifecycle page). Zeno would need the plain Groth16 selector, because Settlement cannot read Boundless's Ethereum-side set verifier (inferred). Boundless documents only RISC Zero programs (same page).

**WASM in the guest (thread 5).** RISC Zero ships an official `examples/wasm` guest on `wasmi = "0.31"` with `std` ([guest Cargo.toml](https://raw.githubusercontent.com/risc0/risc0/main/examples/wasm/methods/guest/Cargo.toml)). No cycle numbers were found.

**Recursion and aggregation (thread 6).** Receipts compose: a `resolve` program removes an assumption by verifying a STARK, and repeated `join` merges segments ([recursion](https://dev.risczero.com/api/recursion)). An official `composition` example exists ([examples](https://github.com/risc0/risc0/tree/main/examples)). R0VM Helios had a short zkSecurity review on 2025-04-02 that recommends "a more comprehensive audit" ([zkSecurity](https://reports.zksecurity.xyz/reports/risc0-helios/)).

**Soundness record (thread 7).** Six advisories ([advisories](https://github.com/risc0/risc0/security/advisories)):

- Critical, 2025-10-01: guest arbitrary code execution via `sys_read` (fixed in 2.3.2 and 3.0.3 per L2BEAT).
- Critical, 2025-06-18: missing rv32im constraint (fixed in 2.1.0).
- Moderate, 2025-08-04: underconstrained division (fixed in 2.2.0). High, 2024-09-25: multi-step instruction modes. Low, 2024: host syscall abuse; zero-knowledge notes.

Main also carries "Add missing m3 constraints around alignment" (2026-03-27) and "Verify proper cycle order for BigInt operations" (2026-03-26) with no matching advisory ([commits](https://api.github.com/repos/risc0/risc0/commits?per_page=10)). Whether these were soundness bugs is unverified. L2BEAT lists no audits for the zkVM itself ([L2BEAT](https://l2beat.com/zk-catalog/risc0)).

**Production.** Ronin, MegaETH, BOB, Taiko Alethia and Soon use RISC Zero verifiers ([L2BEAT](https://l2beat.com/zk-catalog/risc0)).

**Tradeoffs.**

- For: the Groth16 setup is the strongest of the candidates (238 phase-2 participants), and `control_root` decouples zkVM upgrades from the setup. Boundless is a live open market and accepts ETH-denominated payment. Precompile coverage matches the R1 list.
- Against: development activity has dropped, with no stable minor release since 3.0 and Ethproofs listing it inactive. SHA3-256 is accelerated only through `tiny-keccak` behind an unstable flag. No Go verifier exists; R4 would write or adapt one. There are two Critical advisories in 2025. The ISA is 32-bit, so wasmi's i64 operations cost more instructions (inferred).

### Option C: OpenVM (Axiom), 2.x

**Status and maintenance.** OpenVM 2.0 went to production on 2026-07-10. It secures Scroll, and Lighter EVM is adopting it ([Axiom blog](https://axiom.xyz/blog/openvm-2-production)). Ethproofs lists 2.1.0-preview on `rv64im_zicclsm` ([Ethproofs](https://ethproofs.org/zkvms)). Licence: MIT and Apache-2.0, including the GPU prover (Axiom blog).

**Precompiles.** Extensions on `main`: `algebra`, `bigint`, `deferral`, `ecc`, `keccak256`, `pairing`, `rv32-adapters`, `rv32im`, `sha2` ([extensions](https://github.com/openvm-org/openvm/tree/main/extensions)). ECC covers short Weierstrass curves only, such as secp256k1 and secp256r1. Pairing covers BN254 and BLS12-381 ([Axiom docs](https://docs.axiom.xyz/user-guides/openvm/intro-to-openvm)). No Edwards or ed25519 module exists in `ecc/guest/src` ([source](https://github.com/openvm-org/openvm/tree/main/extensions/ecc/guest/src)). ed25519 field arithmetic could use the generic modular-arithmetic extension (inferred). SHA-512 is reported in the `sha2` extension by a secondary source ([DeepWiki](https://deepwiki.com/openvm-org/openvm/4.3-hash-extensions:-keccak256-and-sha-256)).

**SHA3-256 (thread 1): primitives exist.** The `keccak256` guest exposes `native_keccakf` (Keccak-f[1600] on a 200-byte state) and `native_xorin` (absorb up to 136 bytes) ([guest lib.rs, main](https://raw.githubusercontent.com/openvm-org/openvm/main/extensions/keccak256/guest/src/lib.rs)). Padding is left to the caller, so SHA3-256 can be built on them. Whether a patched RustCrypto `sha3` exists is unknown.

**SNARK wrapping (thread 3).** Halo2 PLONK with KZG over BN254, used to wrap the final STARK; Solidity verifier generated by the SDK ([L2BEAT OpenVM](https://l2beat.com/zk-catalog/openvmprover); [zkSecurity audit](https://reports.zksecurity.xyz/reports/openvm/)). STARK proofs are under 300 kB; the Halo2 wrap takes 8.1 s on one RTX 5090 (vendor-reported, Axiom blog). Wrapped proof size and gas were not found. **No Go verifier** was found. Verifying Halo2 KZG in Go would need a new verifier implementation, since gnark verifies its own Groth16 and PLONK formats (inferred). Trusted setup: the first 71 Perpetual Powers of Tau contributions, rated yellow (L2BEAT).

**Hardware and network (thread 4).** Vendor-reported real-time Ethereum proving on 8 GPUs (p99 9.8 s) and 16 GPUs (p99 6.3 s) (Axiom blog). No CPU-only requirements and no open prover network were found.

**WASM, recursion (threads 5, 6).** No public wasmi-in-OpenVM evidence was found. OpenVM 2.0 adds "deferrals", which let a guest verify proofs from arbitrary ZK circuits, plus STARK recursion (Axiom blog).

**Soundness (thread 7).** On 2025-05-26 a Plonky3 final-polynomial degree-check bug had an OpenVM analogue, fixed by a Scroll emergency upgrade (L2BEAT). OpenVM 2.0 has a zkSecurity audit and formal verification of RV32IM (Nethermind, Lean), Keccak and SHA-2 (Axiom blog; [Axiom security](https://docs.axiom.xyz/user-guides/openvm/security)).

**Tradeoffs.**

- For: Keccak-f and SHA-2 are formally verified, its Keccak primitive fits SHA3-256 directly, and it is in production for a large rollup.
- Against: no ed25519 acceleration. The Halo2 KZG final proof has no Go verifier, so R4 grows from "integrate gnark" to "write a Halo2 verifier in Go". No open prover network exists, so S3 rests on self-hosted GPUs.

### Option D: zkWasm (Delphinus Lab), non-RISC-V

**What it is.** zkWasm writes "the whole WASM virtual machine in ZKSNARK circuits" rather than running an interpreter on RISC-V ([repo](https://github.com/DelphinusLab/zkWasm)). It is built on Delphinus's Halo2 fork (`halo2-gpu-specific`, tag `stable-logup-shuffle-v1.0.1`), a continuation batcher, and host circuits ([workspace Cargo.toml](https://raw.githubusercontent.com/DelphinusLab/zkWasm/main/Cargo.toml)). The Halo2 commitment scheme (KZG over BN254) is inferred from the PSE lineage, not confirmed. Licence: Apache-2.0; last push 2026-02-10; 35 open issues ([repo API](https://api.github.com/repos/DelphinusLab/zkWasm)). Its operations spec is marked "WIP" (repo README).

**Precompiles (host circuits).** Test scripts exist for keccak, poseidon, merkle, bn254 and jubjub ([host circuits](https://github.com/DelphinusLab/zkWasm-host-circuits)). SHA-256, BLS12-381 and ed25519 were not found. Whether the keccak circuit is Keccak-256 or exposes the permutation is unknown.

**Fit with Zeno.** Zeno's contracts are integer-only WASM MVP (`phase-1/SPEC.md:515-532`), the profile zkWasm targets (inferred). The STF itself (codec, SMT, driver, gas metering, structural limits) is Rust, not WASM. Under zkWasm there are two shapes (inferred):

1. The whole STF, including the interpreter's job, compiles to WASM and runs as the proved program. DT-3 survives (one crate, built natively and for WASM), but the SMT's SHA3-256 runs as WASM code unless a host circuit covers it.
2. Only contract execution is proved in zkWasm, and the rest of the STF is proved elsewhere. That needs two proofs per batch, which conflicts with "one proof per batch" (§1) and DT-5.

**Proving, verification, maturity.** The final proof targets an EVM verifier through the batcher ([repo](https://github.com/DelphinusLab/zkWasm)). No Go verifier was found. CertiK reports a Coq formal verification of zkWasm's circuits ([HackerNoon](https://hackernoon.com/certik-completes-formal-verification-of-zkwasm-mathematically-proving-the-zkvm-sound), date not shown). No open prover network was found beyond a zkVerify verification integration ([zkVerify, 2025-09-24](https://zkverify.io/blog/zkverify-and-delphinus-labs-powering-trustless-wasm-execution-with-efficient-zk-proof)).

**Tradeoffs.**

- For: no interpreter overhead, which the vApps paper puts at 20× to 32× native for wasmi on SP1 ([vApps](https://arxiv.org/html/2504.14809)). A 2024 third-party post reports zkWasm cycle counts "relatively closer to native Rust" than wasmi in a zkVM (search snippet only; [Kautuk on X](https://x.com/Kautukkundan/status/1867589273446232368), not read in full).
- Against: SHA3-256, ed25519 and BLS12-381 host circuits were not found. No Go verifier exists, and it would be Halo2. Activity is low. It is WASM-only, so it does not serve `runtime-evm` or `runtime-btc` (§4.2), nor R6 and R7 on one stack.

### Others surveyed, not presented as options

| zkVM | Why not an option now | Source |
|---|---|---|
| Jolt (a16z) | README: "in alpha and is not suitable for production"; "has not been audited". Latest tag v0.3.0-alpha (2025-10-01). No on-chain wrapping mentioned | [repo](https://github.com/a16z/jolt), [releases](https://api.github.com/repos/a16z/jolt/releases?per_page=5) |
| ZisK (Polygon) | README: "currently undergoing security and correctness audits", no production release; Ethproofs lists 1.3.1-alpha. Has `keccakf`, `sha256f`, `arith_eq`, `arith_eq_384` precompile dirs and a PLONK final proof with a Solidity verifier | [repo](https://github.com/0xPolygonHermez/zisk), [precompiles](https://github.com/0xPolygonHermez/zisk/tree/main/precompiles), [v1.3.0-alpha notes](https://newreleases.io/project/github/0xPolygonHermez/zisk/release/v1.3.0-alpha), [Ethproofs](https://ethproofs.org/zkvms) |
| Pico (Brevis) | v3.0.0 released 2026-10-19 as a breaking release. Its Sherlock audit covers 2.x only. The in-repo verifier "comes from a test setup". Final proof is Groth16 on BN254 | [repo](https://github.com/brevis-network/pico), [releases](https://api.github.com/repos/brevis-network/pico/releases?per_page=5) |
| ZKsync Airbender | "RISC-V prover system for ZKsync OS"; delegation circuits for blake2s, poseidon2, reduced keccak; no general SNARK wrap documented | [repo](https://github.com/matter-labs/zksync-airbender) |
| Ceno (Scroll) | Listed inactive at v0.1 | [Ethproofs](https://ethproofs.org/zkvms) |
| Nexus | Not researched in this pass | — |

### Comparison table

Sources for each cell are in the option sections above. "?" means not found or not verified.

| Criterion | SP1 Hypercube (v6.10) | RISC Zero (3.0.6) | OpenVM (2.0/2.1) | zkWasm (Delphinus) |
|---|---|---|---|---|
| Guest language / ISA | Rust, `std`; RV64IM | Rust, `std`; RV32IM | Rust; RV64 (2.1 preview) | WASM bytecode, no interpreter |
| Keccak precompile | Raw Keccak-f[1600] syscall | Keccak-f via patched `tiny-keccak` (unstable flag) | `keccakf` + `xorin` primitives | Keccak host circuit (variant ?) |
| SHA3-256 accelerated | Yes, patched RustCrypto `sha3` 0.11.0 | Yes via `tiny-keccak` `Sha3` (inferred); no RustCrypto `sha3` patch | Possible from primitives; ready crate ? | ? |
| SHA-256 | Yes | Yes | Yes (SHA-2 family) | ? |
| ed25519 | Edwards add + decompress syscalls; SHA-512 in software | Patched `curve25519-dalek` | No Edwards extension | ? |
| BLS12-381 | Yes | Yes (`bls12_381`, `blst`) | Yes (pairing) | ? |
| Final SNARK | Groth16 or PLONK, BN254, gnark | Groth16, BN254, snarkjs | Halo2 KZG, BN254 | Halo2 (KZG inferred) |
| Final proof size | ~260 B + 100 B prefix (Groth16); ~868 B (PLONK) | 256 B seal (inferred) | ? | ? |
| Public inputs | 5 BN254 scalars | 5 BN254 scalars | ? | ? |
| Go verifier | Yes: SP1 `gnark-ffi/go` (Go 1.24, forked gnark v0.14) | None found; gnark generic Groth16 possible (inferred) | None found | None found |
| EVM verify gas (proxy for cost) | ~270k (Groth16), ~300k (PLONK), vendor | ? | ? | ? |
| Trusted setup | PLONK: Aztec Ignition (green). Groth16: 18-party phase 2 (red) | Groth16: 238-party phase 2 (yellow) | Perpetual Powers of Tau, 71 contributions (yellow) | ? |
| Local hardware | CPU 16+ cores, 16 GB (Groth16) / 64 GB (PLONK); GPU optional, Linux x86_64 | CPU or GPU; Groth16 wrap x86 + Docker (docs ≤ 2.3) | GPUs for real-time; CPU minimum ? | ? |
| Open prover network | Succinct Prover Network: PROVE stake, reverse auction, PROVE payment | Boundless: ZKC collateral, reverse Dutch auction, ETH-priced | None found | None found |
| WASM-in-guest evidence | wasmi 20× to 32× native (vApps); 1.8M to 3.3M cycles for `fib(10000)` on 6.8.1 | Official `examples/wasm` (wasmi 0.31), no numbers | None found | Native |
| Recursion / aggregation | `verify_sp1_proof` on compressed proofs | Composition (`resolve`, `join`) | Deferrals, STARK recursion | Continuation batcher |
| Ethereum light client | SP1 Helios (Zellic, OpenZeppelin audits) | R0VM Helios (short zkSecurity review) | ? | ? |
| Advisories | 6 (5 High), latest 2026-10-11 | 6 (2 Critical), latest 2025-10-01 | 1 known (2025-05-26) | ? |
| Audits | Cantina, Code4rena, Zellic, Kalos, rkm0959, Veridise; RISC-V opcodes formally verified | None listed by L2BEAT | zkSecurity; Lean RV32IM, Keccak, SHA-2 | CertiK Coq formal verification |
| Production users | ~15 chains on PLONK, 4 on Groth16, some overlap (L2BEAT) | Ronin, MegaETH, BOB, Taiko, Soon | Scroll; Lighter (adopting) | ? |
| Release cadence | ~2 weeks | Patch only since 3.0; v5 RC Jan 2026; Ethproofs "inactive" | 2.0 Jul 2026, 2.1 preview | Last push 2026-02-10 |
| Licence | MIT / Apache-2.0 | Apache-2.0 | MIT / Apache-2.0 | Apache-2.0 (MIT also listed) |

## Unknowns

### Only the human can answer

1. **Is a token-gated prover network acceptable for S3?** `blocking`. Both open networks need a token on Ethereum: requesters pay PROVE on the Succinct network, and Boundless provers post ZKC collateral. The spec says only "an open prover network" (§4.5). Whether an Ethereum token dependency meets S3, or S3 must rest on local CPU proving alone, decides how much weight the network criterion carries.
2. **What counts as "special hardware" in S3?** `blocking`. SP1 proves on a 16-core, 16 GB CPU for Groth16 (64 GB for PLONK) but recommends GPUs or the network for anything non-trivial. A threshold (for example "one 16-core, 64 GB machine proves a worst-case batch within `T₂`") is needed before R2's numbers can pass or fail a candidate.
3. **Trusted-setup policy.** `blocking`. SP1's Groth16 setup is rated red (18 parties, no public call); its PLONK path avoids it at ~868 B and a 60 GB wrap. RISC Zero's Groth16 is yellow. Whether Zeno accepts a circuit-specific setup at all, and at what participation level, decides Groth16 versus PLONK and so R4's prototype target.
4. **Go toolchain upgrade for go-zenon.** `minor`. SP1's Go verifier needs Go 1.24 and a forked gnark. Whether go-zenon can move from Go 1.23 (`go.mod:3`), and whether a fork in the consensus path is acceptable, is a go-zenon maintainer decision.
5. **Upgrade-cadence tolerance.** `minor`. SP1 shipped two High advisories in V6 in six months. Each forced upgrade is a key rotation under the administrator delay (§4.6). Whether Zeno pins one version and patches only for advisories, or tracks releases, affects R8.

### Not determinable from public sources in this pass

6. **Cycle cost of SHA3-256 per call on each zkVM.** `minor`. No candidate publishes it. R3 measures it.
7. **ed25519 verification cycle cost.** `minor`. Not published for SP1 or RISC Zero. R6 measures it; it does not block the Zeno mainnet path. go-zenon verifies every account-block signature at L1 (`verifier/account_block.go:445`), and the guest consumes inputs through the input hash chain (§4.4 item 1), so the STF guest needs no ed25519 (inferred).

## Mapping to the spec's open items

- **Q-3 (SMT hash).** SHA3-256 is not a blocker for SP1 (patched RustCrypto `sha3` uses the permutation precompile) or OpenVM (`keccakf` primitive). It is accelerated on RISC Zero only through `tiny-keccak` behind an unstable flag. SHA3-256 is used across the whole STF, not only the SMT (Current state item 3), so R3 should measure the total SHA3 share per batch.
- **Q-5 (who runs provers).** SP1 has the Succinct Prover Network (PROVE-staked provers, PROVE payment). RISC Zero has Boundless (ZKC collateral, ETH-priced example). OpenVM and zkWasm have no open network found. Executors can self-prove on SP1 and RISC Zero with CPUs (SP1: 16 cores, 16 GB for Groth16).
- **Q-9 (aggregation).** SP1 and RISC Zero both verify their own proofs in a guest (`verify_sp1_proof`; composition). OpenVM 2.0 has deferrals. SP1's docs say a single proof is cheaper than aggregating several, so aggregation helps Settlement cost, not proving cost.
- **S3 condition.** Met in principle by SP1 and RISC Zero through CPU proving or an open network, pending Unknowns 1 and 2. Not met by OpenVM or zkWasm today on public evidence.
- **R2 inputs.** Use wasmi 2.x with `indirect_dispatch` and `portable_dispatch` on SP1 as the measured cheapest configuration ([wasmi-sp1-bench](https://raw.githubusercontent.com/imikushin/wasmi-sp1-bench/main/results.md)). RISC Zero's official example pins wasmi 0.31. The vApps factor (20× to 32× native) is the only published baseline. The worst case is 64 × 10⁹ gas per batch (`vm/constants/settlement.go:10,63`).
- **R3 inputs.** SP1: `sha3` 0.11.0 patch tag `patch-sha3-0.11.0-sp1-6.0.0`. RISC Zero: `tiny-keccak` tag `tiny-keccak/v2.0.2-risczero.0` with the unstable flag. Native side: `golang.org/x/crypto/sha3` `New256` (`common/crypto/hash.go:10`).
- **R4 inputs.**
  1. SP1 Groth16: wire layout is prefix(4) + exit_code(32) + vk_root(32) + proof_nonce(32) + gnark proof. Five public inputs. Digest is SHA-256 masked with `0x1F`. Go reference: `gnark-ffi/go/sp1/verify.go`.
  2. SP1 PLONK: ~868 B, same five inputs, Aztec Ignition setup.
  3. RISC Zero Groth16: 5 inputs (control root ×2, claim digest ×2, BN254 control ID), snarkjs circuit, no Go code.
  4. Budget: about 13.5 KiB free in the commitment for proof and R9 additions (Current state item 2). Every wrapped proof above fits.
  5. A zkVM version change changes the wrap key, so Settlement may need to hold one key per zkVM version alongside the per-domain program key (DT-6) (inferred).
- **R7 inputs.** SP1 Helios (Zellic and OpenZeppelin audits) and R0VM Helios (short zkSecurity review) exist. Each verifies inside its own zkVM's recursion: compressed SP1 proofs or RISC Zero receipts. Cross-zkVM recursion was not found, so R7 effectively follows the R1 choice (inferred).

## Threads left unpulled

1. Reading the audit PDFs themselves (SP1 `hypercube-zellic.pdf`, OpenVM zkSecurity, SP1 Helios Zellic) for findings and scope. Only their existence was recorded.
2. Diffing `p4u/gnark` against upstream gnark v0.14.0 to see why SP1 needs the fork.
3. Per-precompile cycle costs (Keccak-f, SHA-256 compress, Edwards add) from SP1 or RISC Zero cost tables or the Ethproofs data; none were extracted.
4. ZisK internals beyond the precompile directory names: its `keccakf` interface, PLONK proof size, Go verifiability, and prover network plans.
5. Nexus zkVM, Ziren (ZKM, MIPS) and zkDTVM (AntChain) were not researched. Nexus was named in the brief.
6. Third-party prover markets beyond Succinct and Boundless (for example Brevis's network for Pico, or zkVerify as a verification layer) were not surveyed.
7. The GitHub advisory pages were read only as titles and dates; affected and patched versions were confirmed only for GHSA-63x8-x938-vx33 and the three RISC Zero entries cited by L2BEAT.
8. **Does upstream gnark verify SP1 proofs, or only the `p4u/gnark` fork?** `minor`. Matters for R4's dependency choice. The fork's diff was not inspected.
9. **RISC Zero Groth16 in Go.** `minor`. Whether a snarkjs-format RISC Zero key and seal verify under gnark's `groth16.Verify` with a format conversion was not tested. R4 owns it.
10. **OpenVM and zkWasm final proof size and verify cost.** `minor`. Not found. Matters only if either stays in contention, because a Halo2 verifier in Go would be new code.
11. **Network pricing.** `minor`. Neither network publishes a standing price. R5 can sample fulfilled requests on each explorer.
12. **RISC Zero roadmap.** `minor`. The v5.0.0-rc.1 notes are empty and main has seen only maintenance commits since March 2026. Whether R0VM is still actively developed could not be confirmed.
