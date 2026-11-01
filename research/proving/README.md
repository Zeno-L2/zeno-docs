# Proving

Phase 2 validity-proof research. `../../phase-2/SPEC.md` §5 defines each item: the question, the method and the data to collect. One file per item holds its findings and ends in measured data or a decision record.

| Item | Question | File | Status |
|---|---|---|---|
| R1 | Which zkVM? | `R1-zkvm-selection.md` | Findings collected; decision open |
| R2 | What does proving WASM cost? | `R2-wasm-proving-cost.md` | Not started |
| R3 | Keep SHA3-256 for the SMT? | `R3-smt-hash.md` | Not started |
| R4 | Can an embedded contract verify the SNARK? | `R4-onchain-snark-verify.md` | Not started |
| R5 | Who pays for proving? | `R5-proving-economics.md` | Not started |
| R6 | How does Ethereum learn Zenon finality? | `R6-zenon-finality-on-ethereum.md` | Not started |
| R7 | Can the guest verify Ethereum? | `R7-guest-verifies-ethereum.md` | Not started |
| R8 | Can the verifying key replace `stfSpecHash` and the version checks? | `R8-verifying-key-versioning.md` | Not started |
| R9 | Can the input hash chain replace per-input assignment records? | `R9-input-chain-records.md` | Not started |

R6 and R7 serve an Ethereum bridge and are not on the mainnet path; they are tracked here because they share the zkVM.
