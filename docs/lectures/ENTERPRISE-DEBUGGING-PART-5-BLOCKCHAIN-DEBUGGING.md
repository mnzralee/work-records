# Research Track 5: Smart Contract Debugging & On-Chain Forensics

**Perspective:** May 2026
**Scope:** State-of-the-art tooling and practitioner workflows for EVM and Hyperledger Fabric chaincode debugging, forensics, and incident response.

---

## 1. EVM Debugging — The Production Toolchain

### 1.1 Tenderly (the dominant transaction debugger)

Tenderly is the de-facto debugger most serious EVM teams reach for first. Its debugger renders a fully decoded call trace: every internal call, every event, and the SLOAD/SSTORE storage opcodes with the slot accessed and before/after values. The Gas Profiler tab visualises gas as a flame chart broken down per function and per opcode, distinguishing total gas from "actual gas" (net cost after refunds). The 2025 redesign added an option to surface storage access and event logs directly inside the execution trace via toggle checkboxes, and to jump from any frame in the trace into the Gas Profiler in-place. The platform's killer feature is the Simulator: load any historical transaction, override block number, timestamp, msg.sender, balances, or arbitrary storage slots, then re-execute against a forked state — used everywhere from "what would have happened if the multisig fired earlier" to mainnet exploit replays.

- Tools: Tenderly Debugger, Gas Profiler, Simulator, Virtual TestNets, Alerts.
- https://docs.tenderly.co/debugger
- https://docs.tenderly.co/simulations/state-overrides

### 1.2 Foundry (`forge debug`, `cast run`)

Foundry, written in Rust by Paradigm, hit v1.0 in February 2025 and is now the default toolchain for most new EVM projects and audit firms. The interactive `forge debug` TUI steps through opcodes with stack, memory, storage, and source-map view side-by-side. `cast run <txhash>` re-executes any historical transaction locally and prints a coloured call trace; combined with `--decode-internal`, internal function calls, state changes, and decoded events become readable in the terminal. Failure traces from `forge test` automatically show the exact revert path with decoded args. Audit firms (Trail of Bits, Spearbit, Cyfrin) ship their own Foundry test scaffolds for clients.

- Tools: `forge debug`, `forge test -vvvv`, `cast run --debug`, `cast trace`.
- https://book.getfoundry.sh/reference/forge/forge-debug
- https://www.paradigm.xyz/2025/02/announcing-foundry-v1-0

### 1.3 Hardhat — `console.log` and `hardhat-tracer`

Hardhat remains widely used despite Foundry's rise, primarily for the in-Solidity `console.log` ergonomics. Importing `hardhat/console.sol` lets contracts emit log lines from `view` and `pure` functions during Hardhat Network execution — the closest thing EVM developers have to printf debugging. The community plugin `hardhat-tracer` adds `--trace`/`--traceError` flags that print internal calls, events, and storage operations during test runs. `@nomicfoundation/hardhat-network-helpers` provides `time.increase`, `mine`, `setStorageAt`, and `impersonateAccount` cheats for time-warp and state manipulation in test fixtures.

- Tools: `hardhat/console.sol`, `hardhat-tracer`, `hardhat-network-helpers`.
- https://hardhat.org/docs/reference/console-log
- https://github.com/zemse/hardhat-tracer

### 1.4 Remix Debugger

Remix is no longer the primary IDE for production teams but remains the go-to for one-off contract investigation, especially for non-Foundry users. Its debugger steps through opcodes one at a time with synchronised panels for stack, memory, storage, return data, locals, and source. The "Use generated sources" toggle reveals Solidity's auto-generated Yul routines, eliminating gaps where the debugger previously skipped over compiler-emitted code. Auditors still use Remix when triaging unfamiliar contracts pasted from a block explorer.

- Tools: Remix IDE, Remix Debugger.
- https://remix-ide.readthedocs.io/en/latest/debugger.html

### 1.5 Phalcon (BlockSec)

Phalcon Explorer is BlockSec's transaction-analysis tool, popular among on-chain analysts and security researchers. Its Debugger links Solidity source with the trace; the Call Trace Panel renders external calls and events in a tree, and source code, logs, parameters, and return values are kept in sync — even for transactions with 4,000+ internal calls (DEX router or aggregator territory). Coverage spans 26+ EVM chains. The Funds-Flow visualiser is the feature most cited in DeFi exploit threads on X.

- Tools: Phalcon Explorer, Phalcon Debugger, Funds-Flow.
- https://blocksec.com/explorer
- https://docs.blocksec.com/phalcon/explorer/debugger

### 1.6 Etherscan / Blockscout — the baseline

Verified-source viewing on Etherscan, Blockscout, and chain-specific forks is the lowest-common-denominator forensic tool. The decoded "State Changes" tab and the geth `debug_traceTransaction` integration ("Geth Debug Trace" button) let any user pull an opcode-level trace without running a node. Everyone still starts here.

- https://etherscan.io
- https://docs.blockscout.com

---

## 2. EIP-2535 Diamond Pattern — Debugging Specifics

The Diamond pattern's core debugging problem: a single transaction's call trace jumps across multiple facet contract addresses via `delegatecall`, and off-chain tooling frequently fails to decode them because the function selector resolves on the diamond, but the bytecode lives elsewhere. Stack traces look as if "function X exists on diamond" but Etherscan's verified source for the diamond contains only `fallback()`. Audit reports (ChainScore, Veritas) repeatedly flag this as the primary failure mode.

The mitigation is rigorous use of `DiamondLoupeFacet` — the four loupe functions (`facets`, `facetFunctionSelectors`, `facetAddresses`, `facetAddress`) are mandatory by spec and let any indexer reconstruct the live routing table. The community-built **louper.dev** is the reference inspector: it reads loupe functions directly and shows facets, selectors, and bytecode hashes for over 4,000 diamonds across 49 chains. Selector collisions are the second pitfall — two facets exposing the same `bytes4` would silently overwrite each other in the diamond mapping. Mature teams catch this at deploy time using Slither's `unprotected-upgrade` and `function-id-collision` detectors, Surya for selector lists, and custom CI scripts that diff `facetFunctionSelectors` output before/after a `diamondCut`. Nick Mudge's `diamond-3-hardhat` reference implementation includes deploy-time selector-collision assertions copied widely.

- Tools: louper.dev, DiamondLoupeFacet, mudgen/diamond-3-hardhat, Slither selector checks, Surya.
- https://louper.dev
- https://eip2535diamonds.substack.com/p/useful-data-for-eip-2535-diamonds
- https://github.com/mudgen/diamond-3-hardhat

---

## 3. On-Chain Forensics & Post-Incident Analysis

The standard incident-response forensic flow at a DeFi protocol now looks like: (a) Etherscan/Blockscout for the raw tx and verified source, (b) Tenderly Simulator to fork mainnet at the exploit block and step through with state overrides, (c) Phalcon Funds-Flow to trace stolen funds across hops and bridges, (d) `debug_traceTransaction` via geth/erigon for ground-truth opcode traces when the visualisers disagree. The geth `debug` namespace exposes `prestateTracer` (returns accounts the tx touched, plus a "diff" mode showing pre/post state), `callTracer`, and `4byteTracer` — all run by re-executing the tx locally against an archive node, returning structured JSON. Chainstack and QuickNode expose these RPCs as managed services because running an archive node costs $300+/month in disk alone.

Mature teams keep an "incident-response runbook" document with pre-allocated war-room channels, multisig signer phone tree, and pause-bot wiring. The recurring tools in DeFi post-mortems from 2021-2025 are the same: **Tenderly + Phalcon + Etherscan + an archive-node RPC**. Forensics firms (BlockSec, ChainAnalysis, Elliptic, MerkleScience, Hacken, Chainalysis, ChainLight) overlay address-clustering and bridge-tracking on top.

- Tools: `debug_traceTransaction`, prestateTracer, callTracer, Phalcon Funds-Flow, Tenderly Simulator.
- https://geth.ethereum.org/docs/developers/evm-tracing
- https://chainstack.com/deep-dive-into-ethereum-trace-apis/

---

## 4. Static Analysis & Pre-Deployment Verification

### 4.1 Slither (Trail of Bits)

Slither is the SAST baseline for every audit firm and CI pipeline. It detects 90+ vulnerability classes (reentrancy, uninitialised storage, shadowed state variables, incorrect ERC standards, function-id collisions, unprotected upgrades) without execution, and ships a Python API for custom detectors. Slither runs in seconds on contracts that take Mythril hours.

- https://github.com/crytic/slither

### 4.2 Mythril (ConsenSys)

Mythril is symbolic execution — it explores reachable execution paths to find reentrancy, integer overflow, unchecked external calls, and unreachable conditions. Slower than Slither but catches a different class of bug. Still active in 2026 but largely superseded by Halmos for projects already using Foundry.

- https://github.com/Consensys/mythril

### 4.3 Certora Prover

Certora is the industrial-grade formal-verification platform used by Aave, Compound, Lido, MakerDAO, and Balancer. Auditors write rules in Certora's Specification Language (CVL) — e.g., "totalSupply equals sum of balances" or "no user can withdraw more than they deposited" — and the Prover either confirms the property holds for all reachable states or returns a concrete counterexample. Certora rules are now a hiring requirement at top DeFi protocols.

- https://www.certora.com
- https://docs.certora.com

### 4.4 Halmos (a16z crypto)

Halmos is a symbolic-execution test runner that reads existing Foundry test files and treats them as symbolic specifications: any `assertEq` or `assert` becomes a property, and Halmos searches for inputs that violate it. The killer feature is zero-overhead adoption — if you have Foundry tests, you have Halmos coverage.

- https://github.com/a16z/halmos

### 4.5 Foundry `forge invariant`

Stateful fuzzing: define an `invariant_X()` function that must hold across every random call sequence. Forge generates random call sequences (with configurable `runs` and `depth`) and after each call evaluates every invariant. The pattern is now standard for AMM math, vault accounting, and ERC-4626 conformance.

- https://book.getfoundry.sh/forge/invariant-testing
- https://www.cyfrin.io/blog/smart-contract-fuzz-testing-using-foundry

---

## 5. Hyperledger Fabric Chaincode Debugging

### 5.1 Verbose invocation and log levels

`peer chaincode invoke` accepts no `--debug` flag directly; instead, log levels are controlled via the `FABRIC_LOGGING_SPEC` environment variable on the peer. The canonical incident-debug spec is `info:dockercontroller,endorser,chaincode,chaincode.platform=debug,gossip=warning` — INFO globally, DEBUG on the chaincode-relevant loggers, WARNING on gossip to suppress noise. `core.yaml` accepts the same string under `logging.spec`.

- https://hyperledger-fabric.readthedocs.io/en/latest/logging-control.html

### 5.2 Dev mode and CCAAS

Dev mode (`peer chaincode start`) lets the developer launch chaincode manually outside the peer's lifecycle, attach a debugger (Delve for Go, JDWP for Java, node `--inspect` for TS), and step through transactions. Fabric 2.5 ships chaincode-as-a-service (CCAAS) as a built-in builder. CCAAS containers run independently of the peer's Docker socket — debug ports stay inside the cluster network, no host port-forwarding is needed, and `CORE_CHAINCODE_EXECUTETIMEOUT` must be raised to ~300s to prevent the default 30s timeout from firing while a developer is paused on a breakpoint.

- https://hyperledger-fabric.readthedocs.io/en/release-2.5/cc_basic.html
- https://github.com/hyperledger/fabric-samples/blob/main/test-network/CHAINCODE_AS_A_SERVICE_TUTORIAL.md

### 5.3 Test harnesses — MockStub and fabric-test

`MockStub` (the wearetheledger fork is the most-used) lets unit tests invoke chaincode functions in-process with a mock ledger, bypassing the peer entirely. Tests run in milliseconds and cover happy-path and revert-path logic with full code coverage. `fabric-test` is the official integration-test repo with reference networks. For ledger-state assertions, the `fabric-shim` `GetState`/`PutState` cycle is fully mocked.

- https://github.com/wearetheledger/fabric-mock-stub

### 5.4 Hyperledger Caliper

Caliper is the official performance-benchmarking framework. It drives a configurable workload (TPS, payload size, concurrency) at a Fabric, Besu, or Ethereum network and outputs an HTML report with throughput, latency (min/max/avg/p99), and resource utilisation per peer. The Fabric 2.5 baseline benchmarks (Hyperledger Foundation, Feb 2023) used Caliper.

- https://hyperledger.github.io/caliper/
- https://www.lfdecentralizedtrust.org/blog/2023/02/16/benchmarking-hyperledger-fabric-2-5-performance

### 5.5 Fabric Operations Console

The Operations Console (originally IBM Blockchain Platform, now an open-source Hyperledger Labs project) provides web-UI dashboards for peer/orderer health, channel membership, chaincode lifecycle, and ledger height. It is the de-facto observability dashboard in production Fabric deployments.

- https://github.com/hyperledger-labs/fabric-operations-console

---

## 6. Cross-Chain Observability & Indexing

Reading contract state directly from RPC does not scale: a "user transaction history" page that scans logs back to genesis is unusable past a few thousand blocks. The standard solution is event indexing into a queryable store.

**The Graph** is the dominant indexer. A subgraph is a TypeScript mapping plus a GraphQL schema; the indexer streams logs, runs the mappings, and serves GraphQL. The hosted service was deprecated in 2026; projects now run on the decentralised Graph Network (paying GRT) or on managed alternatives. **Goldsky** is the leading managed alternative — fully Graph-compatible, sub-second indexing latency via multiple cross-checked node pools, and adds Mirror/Turbo for streaming directly into Postgres or webhooks. Other 2026 contenders: **Envio**, **SubQuery**, **Ormi**.

The architectural insight: subgraphs are read-models of contract state, deliberately denormalised for the queries the frontend actually runs. They are the on-chain analogue of a CQRS projector.

- https://thegraph.com
- https://goldsky.com
- https://docs.envio.dev/blog/blog/best-blockchain-indexers-2026

---

## 7. Production Monitoring of Smart Contracts

**OpenZeppelin Defender Monitor** (called "Sentinels" before the 2024 rename) watches transactions and events against configurable filters and triggers Autotasks — JavaScript hooks that can call multisig methods, send Slack notifications, or pause contracts. Defender's Forta integration lets a Forta detection bot's alert trigger an Autotask that calls `pause()` on a vulnerable contract. **Important May-2026 caveat**: OpenZeppelin announced Defender's hosted shutdown for 1 July 2026; new sign-ups closed June 2025. Open-source Relayer and Monitor remain. Teams are migrating to Tenderly Alerts, Forta, or self-hosted equivalents.

**Forta Network** is the decentralised anomaly-detection layer. Detection bots written in TypeScript or Python scan every transaction for anomalous patterns (large outflows, governance attacks, oracle deviation, flash-loan signatures); subscribers receive webhook alerts. Forta bots prevented or mitigated several mid-size exploits in 2023-2024 by triggering automatic pauses.

**Tenderly Alerts** sets thresholds on gas usage, balance changes, function-call frequency, or arbitrary event emissions, and routes to Slack, PagerDuty, Discord, email, or webhooks. Custom alerting via direct event subscription (web3 `eth_subscribe` to SQS or Kafka) is also common at protocols with bespoke needs.

- https://docs.openzeppelin.com/defender/module/monitor
- https://docs.forta.network
- https://docs.tenderly.co/alerts

---

## 8. Mainnet Incident-Response Stories

### 8.1 Curve Finance, July 2023

Vyper compiler versions 0.2.15 / 0.2.16 / 0.3.0 had a broken reentrancy lock. Pools using those compiler versions (alETH/ETH, msETH/ETH, pETH/ETH) were drained for ~$70M starting at 13:10 UTC on 30 July 2023. The Vyper team posted on X within hours; Curve added a "Remove Liquidity" emergency UI. Several MEV searchers (notably c0ffeebabe.eth) front-ran the attacker and returned funds. Tools used in the public post-mortems: ChainLight, BlockSec Phalcon, Hacken, MerkleScience for funds-flow tracing; Etherscan for verified source; Twitter/X as the real-time ops channel.

- https://medium.com/chainlight/curve-finance-analysis-and-post-mortem-ba55f2b26909
- https://hackmd.io/@LlamaRisk/BJzSKHNjn

### 8.2 Euler Finance, March 2023

A flash-loan attack against the `donateToReserves` function — missing health-check after the donation allowed the attacker to make their own account insolvent and seize their own collateral via liquidation. ~$197M drained. Coinbase and BlockSec published deep-dive forensics; the attacker's behaviour (sloppy obfuscation, on-chain messaging) and on-chain negotiation eventually returned ~90% of funds. The exploit transaction is now a textbook case in audit training.

- https://www.coinbase.com/blog/euler-compromise-investigation-part-1-the-exploit
- https://blocksec.com/blog/euler-finance-incident-the-largest-hack-of-2023

### 8.3 Cream Finance, Oct 2021

Third Cream exploit in a year — flash-loan-driven oracle manipulation on yUSD pricePerShare via 4Pool. ~$130M drained. Yearn salvaged $9.42M from a "donation" the attacker accidentally sent to a yUSD vault and returned it to Cream. Markets paused immediately; v1 markets isolated, Iron Bank protected.

- https://medium.com/cream-finance/post-mortem-exploit-oct-27-507b12bb6f8e

### 8.4 bZx, Feb 2020

The original DeFi flash-loan exploit. Two attacks within a week using Uniswap as a manipulable oracle. Established the now-universal pattern: TWAPs, Chainlink oracles, and "never trust spot price from a single AMM." Hacken's documented flash-loan-attack timeline traces the lineage from bZx through every subsequent oracle exploit.

- https://hacken.io/discover/flash-loan-attacks/

**Recurring toolset across all post-mortems:** Etherscan, Tenderly Simulator, Phalcon Explorer, `debug_traceTransaction`, X for real-time coordination, multisig (Safe) for emergency pauses.

---

## 9. Off-Chain → On-Chain Bridge Debugging

The "submitter sent a tx but the read model didn't update" failure mode is universal in CQRS-on-chain architectures. Mature teams correlate three IDs: the off-chain command/outbox row ID, the on-chain transaction hash, and the projected read-model row ID. The standard pattern:

1. Outbox row carries a deterministic correlation ID (UUID) included in the on-chain transaction's calldata or as an indexed event topic.
2. The on-chain tx emits an event containing that correlation ID.
3. The indexer (subgraph, projector, Goldsky pipeline) writes the read-model row keyed by the same correlation ID.

When debugging "row Z is missing", the operator searches each of the three systems by correlation ID and identifies which stage is silent. Subgraph health is monitored by `_meta { block { number } }` — comparing the indexer's head block to the chain head reveals indexer lag. Goldsky and The Graph both expose subgraph-health webhooks (`indexing_errors`, `current_block_lag`).

The Graph's "deterministic indexing" guarantee — same inputs produce the same outputs — is what makes read-model verification possible: you can rebuild the projection from an archive node and diff against the production indexer to find divergence. Practitioners on X (notably DeFi infra leads at Maker, Aave, and Lido) routinely cite this pattern as the difference between "the indexer is stuck" being a 30-second triage versus a multi-hour outage.

- https://thegraph.com/docs/en/subgraphs/quick-start/
- https://docs.goldsky.com/subgraphs/introduction

---

## Recurring Themes (May 2026)

1. **Foundry has won the local-development battle.** Tenderly has won transaction debugging and simulation. The combination dominates EVM workflows.
2. **Formal verification has gone from research to production.** Certora is now mainstream at top-tier DeFi; Halmos lowered the barrier for everyone else.
3. **Diamond-pattern debugging remains hard.** The pattern is widely deployed (4,000+ diamonds), but tooling outside louper.dev and DiamondLoupe introspection is thin. Teams write custom selector-diff scripts as part of every cut.
4. **Fabric debugging is mature but lonely.** MockStub, Caliper, dev mode, CCAAS, and Operations Console cover the lifecycle, but the ecosystem is an order of magnitude smaller than EVM's. Solutions are mostly internal IBM / LFD blog posts and Slack archives, not widely-publicised tooling.
5. **Indexers are read-models.** The Graph + Goldsky pattern is the on-chain CQRS projector, and the same correlation-ID discipline that backend engineers use for distributed tracing translates directly to on-chain → off-chain debugging.
6. **OpenZeppelin Defender's shutdown reshapes monitoring.** Tenderly Alerts, Forta, and self-hosted Relayer/Monitor forks absorb the workload through 2026.

---

*End of report.*
