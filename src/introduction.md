# Stacks Core Academy

A comprehensive guide to the `stacks-core` Rust codebase, covering every major subsystem from foundational types through Nakamoto consensus and the signer architecture.

## Resources

- [Stacks Whitepaper](whitepaper.md)
- [Academy Slides](slides.md) *(February 2025 — some parts may be outdated)*

## How to Read This Book

**If you are new to Stacks internals**, start with Part 1 (Foundations) in order. Chapter 0 maps every SIP to its implementing code, Chapter 1 gives you the map of the entire codebase, and Chapters 2-4 establish the vocabulary (types, names, genesis) used everywhere else.

**If you are targeting a specific subsystem**, use the crate map below to identify which part covers your area, then read the relevant chapters. Each chapter lists its dependencies on other chapters so you can backfill as needed.

**If you are looking up a specific concept**, use Appendix A (Glossary) for quick definitions and pointers to the relevant chapter.

## Crate Map

```
stacks-node                       ──> Part 8 (Node)
  ├── stackslib                   ──> Parts 3-6 (Burnchain, Chain State, Nakamoto, Networking)
  │     ├── clarity               ──> Part 2 (Clarity VM)
  │     │     ├── clarity-types   ──> Chapter 3 (Clarity Type System)
  │     │     └── stacks-common   ──> Chapter 2 (Common Types)
  │     ├── pox-locking           ──> Chapter 16 (Stacking and PoX)
  │     ├── libstackerdb          ──> Chapter 25 (StackerDB)
  │     └── stx-genesis           ──> Chapter 4 (Genesis Data)
  ├── stacks-signer               ──> Part 7 (Signer)
  │     └── libsigner             ──> Chapter 27 (Signer Architecture)
  └── stx-genesis                 ──> Chapter 4 (Genesis Data)
```

---

## Part 1: Foundations

The base layer: the SIP specification landscape, crate layout, shared types, the Clarity type system, and how the chain bootstraps.

| Chapter                                                                       | File                                             | Description                                                                                                                                                                                                                                               |
| ----------------------------------------------------------------------------- | ------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [00 -- SIP Landscape](part-1-foundations/00-sip-landscape.md)                 | `part-1-foundations/00-sip-landscape.md`         | Every Stacks Improvement Proposal mapped to the codebase: the governance process, core protocol SIPs, Clarity language evolution, PoX iterations, Nakamoto-era specifications, and a quick-reference table linking each SIP to its implementing chapters. |
| [01 -- Architecture Overview](part-1-foundations/01-architecture-overview.md) | `part-1-foundations/01-architecture-overview.md` | All 13 workspace crates, their dependency graph, major threads, and the end-to-end data flow from Bitcoin events through Stacks state.                                                                                                                    |
| [02 -- Common Types](part-1-foundations/02-common-types.md)                   | `part-1-foundations/02-common-types.md`          | The `stacks-common` crate: hash types (SHA-256, SHA-512/256, Hash160, RIPEMD-160), secp256k1 keys, VRF keys, C32 address encoding, and the `StacksMessageCodec` consensus serialization trait.                                                            |
| [03 -- Clarity Type System](part-1-foundations/03-clarity-type-system.md)     | `part-1-foundations/03-clarity-type-system.md`   | The `clarity-types` and `clarity` crates' type layer: the `Value` enum and all variants, `TypeSignature`, `PrincipalData`, `ClarityName`, `ContractName`, and the binary serialization format.                                                            |
| [04 -- Genesis Data](part-1-foundations/04-genesis-data.md)                   | `part-1-foundations/04-genesis-data.md`          | The `stx-genesis` crate: how genesis chain state is compiled into the binary, initial token allocations, BNS name migration, vesting schedules, and integrity verification.                                                                               |

## Part 2: Clarity VM

The deterministic smart contract execution engine -- from source text to executed state transitions.

| Chapter                                                              | File                                        | Description                                                                                                                                              |
| -------------------------------------------------------------------- | ------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [05 -- Clarity Parsing](part-2-clarity-vm/05-clarity-parsing.md)     | `part-2-clarity-vm/05-clarity-parsing.md`   | Lexing and parsing Clarity source code into `SymbolicExpression` AST nodes, including the S-expression grammar and literal value parsing.                |
| [06 -- Clarity Analysis](part-2-clarity-vm/06-clarity-analysis.md)   | `part-2-clarity-vm/06-clarity-analysis.md`  | The static type checker: type inference, trait validation, read-only analysis, and the `ContractAnalysis` output that summarizes a contract's interface. |
| [07 -- Clarity Execution](part-2-clarity-vm/07-clarity-execution.md) | `part-2-clarity-vm/07-clarity-execution.md` | The interpreter: how the VM evaluates expressions, manages the call stack, handles native functions, and produces transaction receipts.                  |
| [08 -- Clarity Database](part-2-clarity-vm/08-clarity-database.md)   | `part-2-clarity-vm/08-clarity-database.md`  | The `ClarityDatabase` abstraction: how contract state is stored in and retrieved from the MARF, the header DB, and the side store.                       |
| [09 -- Clarity Costs](part-2-clarity-vm/09-clarity-costs.md)         | `part-2-clarity-vm/09-clarity-costs.md`     | The cost tracking system: how every operation is metered, block cost budgets, and cost functions for each Clarity built-in.                              |

## Part 3: Burnchain

The interface between Stacks and Bitcoin -- reading burn operations and running sortitions.

| Chapter                                                                 | File                                         | Description                                                                                                                                              |
| ----------------------------------------------------------------------- | -------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [10 -- Bitcoin Integration](part-3-burnchain/10-bitcoin-integration.md) | `part-3-burnchain/10-bitcoin-integration.md` | How Stacks connects to Bitcoin: the Bitcoin indexer, block parsing, OP_RETURN decoding, and the burnchain abstraction layer.                             |
| [11 -- Burn Operations](part-3-burnchain/11-burn-operations.md)         | `part-3-burnchain/11-burn-operations.md`     | The burn operations encoded on Bitcoin: `LeaderBlockCommit`, `LeaderKeyRegister`, `StackSTX`, `TransferSTX`, and their on-chain encoding formats.        |
| [12 -- Sortition DB](part-3-burnchain/12-sortition-db.md)               | `part-3-burnchain/12-sortition-db.md`        | The sortition database: how burn operations are recorded, VRF-weighted miner selection works, consensus hashes are computed, and fork handling operates. |

## Part 4: Chain State

Managing the Stacks blockchain state -- from the authenticated data structure through block processing.

| Chapter                                                             | File                                        | Description                                                                                                                                                                 |
| ------------------------------------------------------------------- | ------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [13 -- MARF](part-4-chain-state/13-marf.md)                         | `part-4-chain-state/13-marf.md`             | The Merklized Adaptive Radix Forest: the persistent, copy-on-write trie that stores all chain state with O(log n) lookups and historical state access.                      |
| [14 -- Transactions](part-4-chain-state/14-transactions.md)         | `part-4-chain-state/14-transactions.md`     | Stacks transaction format: type IDs, authorization (standard and sponsored), payloads (token transfer, contract call, deploy, coinbase), post-conditions, and fee handling. |
| [15 -- Block Processing](part-4-chain-state/15-block-processing.md) | `part-4-chain-state/15-block-processing.md` | How blocks are validated and applied: transaction execution ordering, MARF commits, microblock handling, and the block acceptance pipeline.                                 |
| [16 -- Stacking and PoX](part-4-chain-state/16-stacking-and-pox.md) | `part-4-chain-state/16-stacking-and-pox.md` | Proof-of-Transfer: STX locking, reward cycles, PoX contract mechanics, reward address selection, and the evolution from PoX-1 through PoX-4.                                |
| [17 -- Mining in Epoch 2.x](part-4-chain-state/17-mining-epoch2.md) | `part-4-chain-state/17-mining-epoch2.md`    | Pre-Nakamoto mining: block assembly, microblock streams, miner election via sortition, and the block commit lifecycle.                                                      |

## Part 5: Nakamoto

The Nakamoto upgrade -- fast blocks, Bitcoin finality, and threshold signing.

| Chapter                                                              | File                                       | Description                                                                                                                                              |
| -------------------------------------------------------------------- | ------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [18 -- Nakamoto Consensus](part-5-nakamoto/18-nakamoto-consensus.md) | `part-5-nakamoto/18-nakamoto-consensus.md` | Nakamoto consensus rules: tenure-based block production, the new block header format, Bitcoin finality, and fork resolution without miner-driven reorgs. |
| [19 -- Nakamoto Mining](part-5-nakamoto/19-nakamoto-mining.md)       | `part-5-nakamoto/19-nakamoto-mining.md`    | How Nakamoto miners produce blocks: tenure extends, block proposals to signers, and the miner state machine.                                             |
| [20 -- Coordinator](part-5-nakamoto/20-coordinator.md)               | `part-5-nakamoto/20-coordinator.md`        | The coordinator module: orchestrating sortition processing, block acceptance, and the transition between Epoch 2.x and Nakamoto processing modes.        |
| [21 -- Epoch Transitions](part-5-nakamoto/21-epoch-transitions.md)   | `part-5-nakamoto/21-epoch-transitions.md`  | How the node handles epoch upgrades: activation heights, consensus rule changes, boot contract deployments, and the critical 2.5-to-3.0 transition.      |

## Part 6: Networking

Peer-to-peer communication, block synchronization, and external APIs.

| Chapter                                                              | File                                        | Description                                                                                                                              |
| -------------------------------------------------------------------- | ------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| [22 -- P2P Protocol](part-6-networking/22-p2p-protocol.md)           | `part-6-networking/22-p2p-protocol.md`      | The Stacks P2P network protocol: message framing, handshake, neighbor discovery, NAT traversal, and the conversation state machine.      |
| [23 -- Block Sync](part-6-networking/23-block-sync.md)               | `part-6-networking/23-block-sync.md`        | Block download and synchronization: inventory-based block fetching, Nakamoto tenure downloads, and the downloader state machine.         |
| [24 -- RPC API](part-6-networking/24-rpc-api.md)                     | `part-6-networking/24-rpc-api.md`           | The HTTP RPC interface: endpoint routing, request handling, read-only contract calls, transaction broadcasting, and chain state queries. |
| [25 -- StackerDB](part-6-networking/25-stackerdb.md)                 | `part-6-networking/25-stackerdb.md`         | The off-chain replicated data store: slot-based storage, chunk replication between peers, and its role in signer coordination.           |
| [26 -- Relay and Mempool](part-6-networking/26-relay-and-mempool.md) | `part-6-networking/26-relay-and-mempool.md` | Transaction and block relay: mempool admission, fee estimation, replace-by-fee, and the relay module's message processing pipeline.      |

## Part 7: Signer

The Nakamoto signer system -- threshold cryptography for block finalization.

| Chapter                                                                  | File                                        | Description                                                                                                                                      |
| ------------------------------------------------------------------------ | ------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| [27 -- Signer Architecture](part-7-signer/27-signer-architecture.md)     | `part-7-signer/27-signer-architecture.md`   | The `libsigner` crate: message types, signer-node communication protocol, block validation primitives, and the WSTS threshold signature scheme.  |
| [28 -- Signer Implementation](part-7-signer/28-signer-implementation.md) | `part-7-signer/28-signer-implementation.md` | The `stacks-signer` binary: the signer run loop, DKG coordination, block proposal handling, reward cycle transitions, and StackerDB integration. |

## Part 8: Node

The runtime orchestration layer -- from startup through steady-state operation.

| Chapter                                                            | File                                    | Description                                                                                                                                                 |
| ------------------------------------------------------------------ | --------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [29 -- Node Startup](part-8-node/29-node-startup.md)               | `part-8-node/29-node-startup.md`        | The `stacks-node` binary: CLI parsing, configuration loading, chain state initialization, genesis block processing, and the boot sequence.                  |
| [30 -- Thread Architecture](part-8-node/30-thread-architecture.md) | `part-8-node/30-thread-architecture.md` | The multi-threaded runtime: the run loop, P2P thread, RPC thread, relayer thread, miner thread, and inter-thread communication patterns.                    |
| [31 -- Bitcoin Controller](part-8-node/31-bitcoin-controller.md)   | `part-8-node/31-bitcoin-controller.md`  | The Bitcoin controller: communicating with bitcoind, submitting burn operations, monitoring confirmation depth, and handling Bitcoin chain reorganizations. |

## Appendices

| Appendix                                                    | File                                | Description                                                                                                                       |
| ----------------------------------------------------------- | ----------------------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| [A -- Glossary](appendices/A-glossary.md)                   | `appendices/A-glossary.md`          | Definitions of key terms used throughout the codebase: sortition, tenure, MARF, PoX, epoch, and more.                             |
| [B -- Config Reference](appendices/B-config-reference.md)   | `appendices/B-config-reference.md`  | Complete reference for `Stacks.toml` configuration: node settings, burnchain connection, miner configuration, and network tuning. |
| [C -- Event Dispatcher](appendices/C-event-dispatcher.md)   | `appendices/C-event-dispatcher.md`  | The event observer system: how the node emits block, transaction, and contract events to external consumers like the Stacks API.  |
| [D -- Testing and Tools](appendices/D-testing-and-tools.md) | `appendices/D-testing-and-tools.md` | Testing infrastructure: unit test patterns, integration test framework, the `stacks-inspect` tool, and the `clarity-cli` REPL.    |
