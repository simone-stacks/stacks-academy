# Chapter 1: Architecture Overview

## Purpose

The `stacks-core` repository contains the full implementation of the Stacks blockchain -- a Layer-2 that settles on Bitcoin. This chapter maps the crate structure, explains how the crates depend on each other, identifies the major runtime threads, and traces the end-to-end data flow from a Bitcoin event through to Stacks chain state.

Understanding this big picture is the prerequisite for every other chapter in this book.

## Workspace Crates

The workspace is defined in the root `Cargo.toml` (`Cargo.toml:1`). There are 13 crates in the workspace (including 3 contrib tools):

| Crate | Path | Role |
|---|---|---|
| **stacks-common** | `stacks-common/` | Foundational types: hashes, keys, addresses, C32 encoding, the `StacksMessageCodec` trait, VRF, epoch definitions |
| **clarity-types** | `clarity-types/` | Core Clarity value types (`Value`, `TypeSignature`, `PrincipalData`) extracted so they can be used without the full VM |
| **clarity** | `clarity/` | The Clarity smart contract VM: parser, type-checker (analysis), interpreter, cost tracking, database interface |
| **stx-genesis** | `stx-genesis/` | Genesis chain state: compressed CSV data for initial balances, lockups, BNS namespaces, and names |
| **pox-locking** | `pox-locking/` | Proof-of-Transfer token locking logic, used by both `clarity` and `stackslib` |
| **libstackerdb** | `libstackerdb/` | StackerDB protocol types and session logic -- the off-chain replicated data store used by signers |
| **stackslib** | `stackslib/` | The core library: chain state (MARF, blocks, transactions), burnchain integration, sortition DB, P2P networking, RPC API, mempool, mining |
| **libsigner** | `libsigner/` | Shared signer library: message types, signer-node communication, block validation primitives |
| **stacks-signer** | `stacks-signer/` | The standalone signer binary: DKG, block signing, StackerDB integration |
| **stacks-node** | `stacks-node/` | The node binary: startup, run loops, thread orchestration, Bitcoin controller, event dispatcher |
| **stacks-inspect** | `contrib/stacks-inspect/` | CLI tool for inspecting chain state databases |
| **clarity-cli** | `contrib/clarity-cli/` | CLI tool for running Clarity code outside of a node |
| **stacks-cli** | `contrib/stacks-cli/` | CLI tool for key generation and transaction crafting |

## Crate Dependency Graph

The dependency graph flows strictly upward -- lower crates never depend on higher ones:

```
stacks-node ─────────────────────────────────────┐
    │                                             │
    ├──> stackslib ──────────────┐                │
    │       │                    │                │
    │       ├──> clarity ────────┤                │
    │       │       │            │                │
    │       │       ├──> clarity-types            │
    │       │       │       │                     │
    │       │       │       └──> stacks-common    │
    │       │       │                             │
    │       │       └──> stacks-common            │
    │       │                                     │
    │       ├──> pox-locking                      │
    │       │       ├──> clarity                  │
    │       │       └──> stacks-common            │
    │       │                                     │
    │       ├──> libstackerdb                     │
    │       │       └──> stacks-common            │
    │       │                                     │
    │       ├──> stx-genesis (no local deps)      │
    │       │                                     │
    │       └──> stacks-common                    │
    │                                             │
    ├──> stacks-signer                            │
    │       ├──> libsigner                        │
    │       │       ├──> stackslib                │
    │       │       ├──> libstackerdb             │
    │       │       ├──> clarity                  │
    │       │       └──> stacks-common            │
    │       ├──> stackslib                        │
    │       ├──> clarity                          │
    │       ├──> libstackerdb                     │
    │       └──> stacks-common                    │
    │                                             │
    └──> stx-genesis                              │
```

Key observations:
- **stacks-common** is the leaf dependency -- everything depends on it.
- **clarity-types** is a thin extraction of Clarity value types, sitting between `stacks-common` and `clarity`.
- **stackslib** is the heaviest crate, pulling together chain state, networking, and burnchain integration.
- **stx-genesis** has zero local dependencies (only the external `libflate` crate).
- The signer stack (`libsigner` -> `stacks-signer`) depends on `stackslib` for block validation types.

## Major Subsystems

The codebase is organized around five major subsystems:

### 1. Clarity VM (clarity, clarity-types, pox-locking)

The deterministic smart contract execution engine. Clarity is interpreted (not compiled), deliberately non-Turing-complete, and runs identically on every node. The VM includes:

- **Parser**: Converts Clarity source text into an AST of `SymbolicExpression` nodes.
- **Analyzer**: Static type checking, trait validation, cost estimation.
- **Interpreter**: Evaluates the AST against a `ClarityDatabase`, which abstracts the underlying MARF trie.
- **Cost Tracker**: Every operation has a cost. Blocks have a cost budget. Exceeding it aborts the transaction.

### 2. Chain State (stackslib/src/chainstate/)

Manages the Stacks blockchain state:

- **MARF** (Merklized Adaptive Radix Forest): A persistent authenticated data structure that stores all chain state. Every Stacks block produces a new MARF root hash.
- **Transactions**: Parsing, validation, and execution of Stacks transactions (token transfers, contract calls, contract deploys, coinbase, etc.).
- **Block Processing**: Validates and applies blocks to chain state, updating account balances, contract state, and the MARF.
- **PoX / Stacking**: The Proof-of-Transfer consensus mechanism where STX holders lock tokens to earn BTC rewards.

### 3. Burnchain Integration (stackslib/src/burnchains/)

The interface between Stacks and Bitcoin:

- **Bitcoin Indexer**: Parses Bitcoin blocks looking for Stacks-relevant operations (burn operations encoded as `OP_RETURN` outputs).
- **Sortition DB**: Records burn operations and determines which miner wins each sortition using VRF-weighted selection.
- **Burn Operations**: Encodes/decodes the on-chain burn operations: `LeaderBlockCommit`, `LeaderKeyRegister`, `UserBurnSupport`, etc.

### 4. Networking (stackslib/src/net/)

Peer-to-peer communication between Stacks nodes:

- **P2P Protocol**: Handshake, neighbor walking, NAT traversal, message serialization using `StacksMessageCodec`.
- **Block Synchronization**: Downloading blocks, microblocks, and Nakamoto tenure blocks from peers.
- **RPC API**: HTTP interface for wallets, explorers, and other clients.
- **StackerDB**: An off-chain replicated key-value store used by signers to coordinate.
- **Relay and Mempool**: Transaction relay, block relay, and the transaction memory pool.

### 5. Node and Signer (stacks-node, stacks-signer, libsigner)

The runtime orchestration layer:

- **Run Loop**: The main event loop that coordinates burnchain polling, block processing, mining, and P2P networking.
- **Thread Architecture**: Multiple threads for P2P, RPC, relayer, miner, and Bitcoin controller.
- **Signer**: A separate process that participates in Nakamoto block signing via threshold signatures (WSTS/FROST).

## Data Flow: Bitcoin Event to Stacks State

Here is the end-to-end flow when a new Bitcoin block arrives:

```
Bitcoin Network
     │
     ▼
[1] Bitcoin Controller (stacks-node/src/burnchains/)
     │  Polls bitcoind via RPC for new blocks
     │  Parses raw Bitcoin transactions
     ▼
[2] Burnchain Indexer (stackslib/src/burnchains/bitcoin/)
     │  Identifies Stacks burn operations in OP_RETURN outputs
     │  Decodes LeaderBlockCommit, LeaderKeyRegister, etc.
     ▼
[3] Sortition DB (stackslib/src/burnchains/db.rs)
     │  Records all burn operations
     │  Runs sortition: VRF-weighted random selection of a winning miner
     │  Produces a ConsensusHash for this burn block
     ▼
[4] Coordinator (stackslib/src/chainstate/coordinator/)
     │  Determines if a new Stacks block can be processed
     │  Handles epoch transitions (2.0 -> 2.05 -> ... -> 3.0)
     ▼
[5] Block Processing (stackslib/src/chainstate/stacks/)
     │  Validates the winning miner's block
     │  Executes each transaction through the Clarity VM
     │  Updates MARF state (account balances, contract storage, etc.)
     ▼
[6] MARF (stackslib/src/chainstate/stacks/index/)
     │  Produces a new state root hash
     │  Commits the trie to persistent storage (SQLite)
     ▼
[7] Event Dispatcher (stacks-node/src/event_dispatcher.rs)
     │  Emits events to configured observers (e.g., Stacks API)
     │  Publishes new block data, transaction receipts, contract events
     ▼
[8] P2P Relay (stackslib/src/net/relay.rs)
        Announces the new block to peers
        Relays unconfirmed transactions from the mempool
```

In the **Nakamoto era** (Epoch 3.0+), the flow adds a signing step between [5] and [6]:

- The miner proposes a block to the signer set via StackerDB.
- Signers validate the block independently and respond with signature shares.
- Once 70% weighted threshold is reached, the block is finalized with an aggregate Schnorr signature.
- Only signed blocks are accepted by the network.

## Thread Architecture

A running `stacks-node` spawns several long-lived threads:

| Thread | Responsibility |
|---|---|
| **Main / Run Loop** | Polls for new burnchain blocks, triggers sortitions, coordinates block processing |
| **P2P Thread** | Manages peer connections, handles incoming/outgoing messages, neighbor discovery |
| **RPC Thread** | Serves the HTTP API (bound to a configurable port, default 20443) |
| **Relayer Thread** | Processes new blocks and transactions, updates mempool, triggers mining |
| **Miner Thread** | Assembles candidate blocks from mempool transactions, executes them through Clarity |
| **Bitcoin Controller** | Communicates with bitcoind to submit burn operations and read Bitcoin chain state |

In Nakamoto mode, the miner thread additionally coordinates with the signer set through StackerDB.

## Epoch System

Stacks uses an **epoch system** to manage protocol upgrades. Each epoch activates at a specific Bitcoin block height and can change consensus rules, Clarity VM behavior, and P2P protocol semantics.

The epochs are defined in `stacks-common/src/types/mod.rs:120`:

```rust
define_stacks_epochs! {
    Epoch10 = 0x01000,    // Pre-Stacks 2.0 (Bitcoin-only phase)
    Epoch20 = 0x02000,    // Stacks 2.0 launch
    Epoch2_05 = 0x02005,  // Bug fixes
    Epoch21 = 0x0200a,    // Stacks 2.1 (PoX improvements)
    Epoch22 = 0x0200f,    // Stacks 2.2
    Epoch23 = 0x02014,    // Stacks 2.3
    Epoch24 = 0x02019,    // Stacks 2.4
    Epoch25 = 0x0201a,    // Stacks 2.5 (signer onboarding)
    Epoch30 = 0x03000,    // Nakamoto launch
    Epoch31 = 0x03001,    // SIP-029 coinbase schedule
    Epoch32 = 0x03002,    // SIP-031 emissions
    Epoch33 = 0x03003,    // Cost improvements, parameter limits
}
```

Most subsystems check `StacksEpochId` to branch on behavior. For example, `StacksEpochId::uses_nakamoto_blocks()` returns `true` for Epoch30+, which gates the new block header format.

## On-Disk Directory Structure

A running Stacks node stores all persistent state in SQLite databases under its working directory. Understanding this layout is essential for debugging, running `stacks-inspect`, and reasoning about what lives where.

```
<WORKING_DIR>/<NETWORK>/
│
├── chainstate/
│   ├── blocks/
│   │   ├── AA/BB/<hash>             # Epoch 2.x block files (indexed by hash prefix)
│   │   └── nakamoto.sqlite          # Nakamoto staging blocks (Epoch 3.0+)
│   │
│   └── vm/
│       ├── index.sqlite             # Stacks headers MARF + __fork_storage
│       ├── index.sqlite.blobs       # External MARF blob storage (optional)
│       │
│       └── clarity/
│           └── marf.sqlite          # Clarity VM state (contract data, balances)
│
├── burnchain/
│   ├── burnchain.sqlite             # Bitcoin block headers and raw burn operations
│   │
│   └── sortition/
│       └── marf.sqlite              # SortitionDB (sortition results, block commits)
│
├── headers.sqlite                   # Bitcoin SPV headers (lightweight chain verification)
├── stacker_db.sqlite                # StackerDB (signer messages in Nakamoto)
├── peer.sqlite                      # P2P peer database (neighbor state)
└── mempool.sqlite                   # Transaction mempool
```

### Database Summary

| File | Schema Version | Key Tables | Chapter |
|------|---------------|------------|---------|
| `chainstate/vm/index.sqlite` | 13 | `marf_data`, `__fork_storage`, `block_headers`, `nakamoto_block_headers`, `staging_blocks`, `payments`, `transactions` | 13, 15 |
| `chainstate/vm/clarity/marf.sqlite` | -- | `data_table`, `metadata_table`, `marf_data` | 8 |
| `chainstate/blocks/nakamoto.sqlite` | -- | `nakamoto_staging_blocks` (block_hash, consensus_hash, height, is_tenure_start, signing_weight, processed, orphaned, data) | 18 |
| `burnchain/sortition/marf.sqlite` | 10 | `snapshots`, `block_commits`, `leader_keys`, `stack_stx`, `delegate_stx`, `epochs`, `stacks_chain_tips` | 12 |
| `burnchain/burnchain.sqlite` | -- | Bitcoin block headers, raw burn operations | 10 |
| `headers.sqlite` | 3 | `headers` (Bitcoin SPV headers), `chain_work` | 10 |
| `stacker_db.sqlite` | -- | StackerDB chunks (signer messages) | 25 |
| `peer.sqlite` | -- | Neighbor addresses, handshake state | 22 |
| `mempool.sqlite` | -- | Pending transactions, fee estimates | 26 |

Key relationships:
- **Two MARF instances**: `index.sqlite` holds the headers/chainstate MARF (`MARF<StacksBlockId>`), while `sortition/marf.sqlite` holds the sortition MARF (`MARF<SortitionId>`). They share the same code but index different state.
- **Clarity data lives in `clarity/marf.sqlite`**, separate from the headers MARF. Both use the MARF data structure but store different key-value namespaces.
- **Epoch 2.x blocks** are stored as individual files in a hash-prefix directory tree (`blocks/AA/BB/<hash>`). **Nakamoto blocks** are stored in `nakamoto.sqlite` for efficient querying by hash, height, or tenure.
- **`headers.sqlite`** is for Bitcoin SPV verification only -- it stores 80-byte Bitcoin headers, not Stacks headers.

## How It Connects

- **Chapter 2** (Common Types) covers the foundational types from `stacks-common` that appear everywhere.
- **Chapter 3** (Clarity Type System) details the `Value` enum and `TypeSignature` from `clarity-types`.
- **Chapters 5-9** dive deep into the Clarity VM subsystem.
- **Chapters 10-12** cover the burnchain integration layer.
- **Chapters 13-17** explore chain state management in detail.
- **Chapters 18-21** cover Nakamoto consensus additions.
- **Chapters 22-26** detail the networking stack.
- **Chapters 27-28** cover the signer architecture.
- **Chapters 29-31** explore the node runtime.

## Edge Cases and Gotchas

1. **Two block formats**: Pre-Nakamoto blocks (`StacksBlock`) and Nakamoto blocks (`NakamotoBlock`) have different header structures. The epoch determines which format is used.

2. **MARF complexity**: The MARF is not a simple Merkle tree. It is a persistent, copy-on-write trie that supports efficient lookups at any historical block. Every block produces a new root but shares structure with previous blocks.

3. **Consensus-critical code**: Much of `stackslib` is consensus-critical -- changing behavior means a hard fork. Functions in these paths are marked with comments, and any modification requires extreme care.

4. **Test vs. production chain state**: The `stx-genesis` crate ships two data sets (`chainstate.txt` and `chainstate-test.txt`). The test set is much smaller and used in integration tests. The `use_test_chainstate_data` flag controls which is loaded.

5. **Feature flags**: Many crates have a `testing` feature that enables additional methods, relaxed constraints, and mock constructors (e.g., `StacksAddress::new_unsafe`). These must never be used in production.

6. **The `clarity-types` extraction**: The `clarity-types` crate was extracted from `clarity` to break a circular dependency. Code that only needs Clarity values (not the VM) should depend on `clarity-types` directly.
