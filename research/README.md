# Research

Research the specifications are drawn from. Nothing here is normative; [phase-1/SPEC.md](../phase-1/SPEC.md) governs on any conflict, and [phase-2/SPEC.md](../phase-2/SPEC.md) states the Phase 2 direction. One document per topic.

| Topic | Document | Holds |
|---|---|---|
| Proving | [proving/](proving/README.md) | The Phase 2 validity-proof research items R1–R9: which zkVM, proving cost, the SMT hash, on-chain SNARK verification, economics, Ethereum finality, key versioning, input-chain records. One file per item |
| Bridges | [bridges.md](bridges.md) | How a bridge or messaging channel would be built as a Zeno domain: configuration, the components a bridge author writes, lifecycles, worked examples, security. Nothing is built |
| Other chains and runtimes | [multichain.md](multichain.md) | Bringing a new runtime as an `EXECUTION` domain, and integrating a smart-contract chain as a bridge domain |
| State root | [state-root.md](state-root.md) | go-zenon's momentum state root: what a proof proves, how it is built into consensus, the SMT core the executor shares, the build at activation |
