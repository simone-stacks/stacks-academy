# Chapter 0: The SIP Landscape

## Purpose

Every major subsystem in `stacks-core` exists because a Stacks Improvement Proposal (SIP) specified it. SIPs are the formal design documents that define the protocol's consensus rules, token standards, network protocols, and governance processes. Before diving into code, this chapter maps each SIP to the codebase chapters that implement it. Use it as a lookup: when a chapter references "per SIP-007" or "introduced in SIP-021," this is where you find out what that means and why the design looks the way it does.

## What Is a SIP?

SIP-000 defines the governance process itself. A SIP passes through a lifecycle: **Draft** -> **Accepted** -> **Recommended** -> **Activation-In-Progress** -> **Ratified**. SIPs are categorized by type (Consensus, Standard, Meta, Informational) and by consideration (Technical, Economic, Governance). Consensus-type SIPs require implementation by all nodes; Standard-type SIPs may affect only specific subsystems.

Three committees govern the process: a **Steering Committee** that votes on ratification, **Consideration Advisory Boards** that provide expert feedback, and **SIP Editors** that vet incoming proposals for quality. The SIP repository lives at `https://github.com/stacksgov/sips`.

There are currently 30 SIPs (SIP-000 through SIP-035, with some numbers unassigned). The sections below group them by function.

## Core Protocol: Consensus and Blocks

These SIPs define how the blockchain fundamentally works -- leader election, block structure, and state commitments.

### SIP-001: Burn Election (Sortition)

SIP-001 specifies **proof-of-burn** leader election. Miners register a VRF key on Bitcoin, then submit a block-commit transaction that burns BTC and commits to a chain tip. A **verifiable random function** (VRF) produces an unbiased seed, and a weighted distribution selects the winner proportional to BTC burned. The SIP also defines the block-streaming model: miners produce an **anchored block** (committed on-chain) plus a stream of **microblocks** between tenures, with a 60/40 fee split to incentivize building on the latest microblock.

**Code:** Chapter 11 (Burn Operations) implements the wire formats (`LeaderBlockCommit`, `LeaderKeyRegister`). Chapter 12 (Sortition DB) implements the VRF-weighted selection. Chapter 17 (Mining in Epoch 2.x) implements the microblock streaming model. The `BURN_COMMITMENT_WINDOW` modulus field and the parent-block/parent-txoff encoding in `leader_block_commit.rs` directly mirror SIP-001's wire format specification.

### SIP-004: Cryptographic Commitment to Materialized Views (MARF)

SIP-004 specifies the **Merklized Adaptive Radix Forest** -- the authenticated data structure that commits to all chain state. Each block's trie is linked to previous blocks via **back-pointers**, forming a forest rather than a single tree. The SIP defines the trie node types (`node4`, `node16`, `node48`, `node256`, and `leaf`), the copy-on-write insertion algorithm, the **Merklized skip-list** for cross-block proofs, and the O(1) read/write complexity with O(log B) proof generation.

Key design decisions: (1) back-pointer children use the **block header hash** instead of the child node hash for performance (avoids disk seeks); (2) the skip-list hashes blocks at distances 1, 2, 4, 8, 16... from the current block; (3) a `MARF_BLOCK_HEIGHT_TO_HASH` mapping bootstraps height queries through the MARF itself.

**Code:** Chapter 13 (MARF) covers the implementation in `stackslib/src/chainstate/stacks/index/`. The trie node types, `TrieCursor::walk`, back-pointer resolution, and `TrieMerkleProof` all implement SIP-004's specification.

### SIP-005: Blocks, Transactions, and Accounts

SIP-005 is the largest core SIP. It defines:

- **Account model**: Standard accounts (owned by private keys) and contract accounts (created by deploying contracts). Accounts have a nonce (Lamport clock), an address (versioned hash), and an asset map.
- **Transaction format**: Version byte, chain ID, authorization (standard or sponsored), anchor mode, post-condition mode, post-conditions list, and payload. Five payload types: token-transfer (0x00), smart-contract (0x01), contract-call (0x02), poison-microblock (0x03), coinbase (0x04).
- **Authorization signing**: The chained sighash algorithm where each signature feeds into the next signature's digest, enabling both single-sig and multisig spending conditions. Four hash modes (P2PKH, P2SH, P2WPKH-P2SH, P2WSH-P2SH) for backwards compatibility with Stacks v1.
- **Post-conditions**: Assertions about asset transfers that abort the transaction if violated. Fungible conditions (equal, greater, less) and non-fungible conditions (sent, not-sent).
- **Block structure**: Anchored blocks (header + coinbase + transactions) and microblocks (header + transactions). The block header includes VRF proof, cumulative work score, transaction Merkle root, and state Merkle root.
- **Clarity value serialization**: The binary encoding for all Clarity types (type IDs 0x00-0x0e).

**Code:** Chapter 14 (Transactions) implements `StacksTransaction`, `TransactionAuth`, `TransactionPayload`, and the signing/verification algorithms. Chapter 15 (Block Processing) implements block validation. Chapter 3 (Clarity Type System) implements value serialization. The hash modes in `mod.rs:627-652` directly correspond to SIP-005's four spending condition types.

## Clarity Language

### SIP-002: The Clarity Smart Contract Language

SIP-002 defines Clarity itself: a LISP-based, non-Turing-complete, decidable smart contract language. Key constraints: no recursion, no `lambda`, no mutable variables, loops only via `map`/`filter`/`fold`, and human-readable source deployed directly on-chain (no compiler). Public functions (`define-public`) must return Response types (`ok`/`err`). The SIP establishes `tx-sender` (immutable across contract calls) and `contract-caller` (changes per call), the `as-contract` function for contract-owned operations, and the data map primitive.

The SIP also introduces **traits** for dynamic dispatch: contracts can define trait signatures and call functions on contract arguments typed as trait references, enabling composability patterns like DEX routing.

**Code:** Chapters 5-9 (the entire Part 2) implement Clarity. Chapter 5 (Parsing) handles the LISP syntax. Chapter 6 (Analysis) implements the type system and trait validation. Chapter 7 (Execution) runs the interpreter. Chapter 8 (Database) manages contract state.

### SIP-006: Clarity Execution Cost Assessment

SIP-006 establishes the **5-dimensional cost model** for Clarity execution: runtime, data-read-count, data-read-length, data-write-count, and data-write-length. Each native function has cost functions along these dimensions. Block execution is bounded by per-block budgets in each dimension. Costs can be updated by on-chain governance voting through the `costs-3` boot contract.

**Code:** Chapter 9 (Clarity Costs) covers `LimitedCostTracker`, cost functions per epoch, and the block budget enforcement in `ExecutionCost`.

### SIP-008: Clarity Parsing and Analysis Cost Assessment

Extends SIP-006 to cover the cost of *parsing and analyzing* contracts at deploy time, not just executing them. This ensures that deploying an expensive-to-analyze contract costs proportional to its complexity.

### SIP-020: Bitwise Operations

Adds six bitwise operations to Clarity: `bit-xor`, `bit-and`, `bit-or`, `bit-not`, `bit-shift-left`, `bit-shift-right`. Commits to 2's complement representation for signed integers and modulo-128 shift distances. These were first available in Clarity 2 (Epoch 2.1+).

### SIP-033: Clarity 4

Introduces Clarity version 4 in Epoch 3.3 with five features:
- `contract-hash?` -- query the SHA-256 hash of any deployed contract's source code
- `restrict-assets?` and `as-contract?` -- default deny-all asset access from `as-contract` calls, with explicit per-asset allowances
- `to-ascii?` -- convert a Clarity value to its ASCII string representation
- `stacks-block-time` -- access the current block's timestamp
- `secp256r1-verify` -- verify NIST P-256 signatures (enables WebAuthn/passkey integration)

**Code:** Chapter 7 (Clarity Execution) covers native function dispatch per epoch. The `secp256r1-verify` behavior is further clarified by SIP-035 (it uses double-SHA256, not single).

## Proof of Transfer and Stacking

### SIP-007: Stacking Consensus (PoX)

SIP-007 replaces proof-of-burn with **proof-of-transfer** (PoX). Instead of burning BTC, miners transfer it to STX holders who have **stacked** (locked) their tokens. Key specifications:

- **Reward cycles**: 2100 Bitcoin blocks (~2 weeks), consisting of a prepare phase (100 blocks) and reward phase (2000 blocks).
- **Anchor blocks**: The PoX anchor block for a cycle is chosen by F*w threshold confirmation (the block at the PoX prepare phase start that has 80%+ of subsequent miners building on it).
- **Threshold**: A minimum STX lock amount calculated from total liquid supply divided by the number of reward slots.
- **Delegate stacking**: Allows pooled stacking where multiple STX holders delegate to a pool operator.

The SIP also defines the PoX **sunset phase** -- a period after which PoX reverts to PoB. This was later overridden by SIP-015 which removed the sunset.

**Code:** Chapter 16 (Stacking and PoX) covers the boot contracts (`pox-4`), reward set calculation, and the `pox-locking` crate. The `PoxConstants` struct (`burnchains/mod.rs`) encodes the cycle lengths.

### SIP-012: New Cost Limits via Network Upgrade

Specifies the first network upgrade mechanism using burn-block-height-triggered hard forks to introduce new execution cost limits. Established the pattern that all subsequent epoch transitions follow.

**Code:** Chapter 21 (Epoch Transitions) covers how epoch activations are height-triggered.

### SIP-015: Stacks 2.1 Network Upgrade

A major upgrade activating Epoch 2.1 with:
- **PoX-2** smart contract replacing PoX-1 (removes the sunset phase, adds `stack-extend` and `stack-increase`)
- **Clarity 2** with new native functions (`stx-account`, `principal-destruct?`, `principal-construct?`, `get-burn-block-info?`, `slice?`, `string-to-int?`, `int-to-string?`, bitwise ops)
- **Segwit PoX addresses**: Support for `p2wpkh` and `p2wsh` Bitcoin address types
- **Auto-unlock**: If a stacker's PoX participation fails (e.g., anchor block not confirmed), their STX auto-unlock

**Code:** Chapter 16 (Stacking and PoX) covers the PoX contract evolution. Chapter 21 (Epoch Transitions) covers the 2.05→2.1 activation.

### SIP-029: Preserving Economic Incentives (Halving Alignment)

SIP-029 modifies the STX emission schedule to align Stacks halvings with Bitcoin halvings starting in 2028. The first Stacks halving (at Bitcoin block 876,434) would have cut block rewards from 1000 to 500 STX right as Nakamoto and sBTC were launching -- destabilizing miner and signer incentives during a critical transition. SIP-029 defers the first halving to synchronize with the next Bitcoin halving, reduces the tail emission from 125 STX to 62.5 STX per block, and results in a 0.77% lower 2050 target STX supply. Activated in Epoch 3.1.

**Code:** Chapter 21 (Epoch Transitions) covers the Epoch 3.1 activation and coinbase reward schedule changes.

### SIP-031: Five-Year Stacks Growth Emissions

Specifies the creation of a **Stacks Endowment** funded by a one-time 100M STX mint plus 300M STX emitted over 5 years via new block rewards, raising average annual inflation from ~3.5% to ~5.75%. Governance by a 9-member Treasury Committee with staggered 4-year terms. Activated in Epoch 3.2 with a boot contract implementing `update-recipient`, `get-recipient`, and `claim` functions.

**Code:** Chapter 21 (Epoch Transitions) covers the Epoch 3.2 activation.

## Emergency Fixes

Three SIPs represent emergency hard forks to patch critical bugs found in production:

### SIP-022: Emergency PoX Fix (Stacking Increase Bug)

Fixed a critical bug in `stack-increase` that allowed overstating locked STX amounts. Required two hard forks: Epoch 2.2 (disabled PoX entirely as a safety measure) and Epoch 2.3 (re-enabled PoX via the PoX-3 contract with the bug fixed).

### SIP-023: Emergency Trait Invocation Fix

Fixed a regression in Clarity trait type canonicalization introduced in Epoch 2.1 that broke pre-2.1 contracts using trait-based dynamic dispatch. The fix preserved the buggy behavior during Epoch 2.2 to avoid chain reorgs, then applied the correction in Epoch 2.3.

### SIP-024: Emergency Data Validation Fix

Addressed a denial-of-service vulnerability from tuple type mismatches where nested tuples could carry extra fields that bypassed type checking. Introduced a value sanitization routine at contract call boundaries and database reads. Activated in Epoch 2.4.

**Code:** Chapter 21 (Epoch Transitions) covers all three fixes as part of the Epoch 2.2→2.4 rapid succession of hard forks.

## Nakamoto Era

### SIP-021: Nakamoto (Fast and Reliable Blocks)

The most significant protocol change since launch. SIP-021 decouples Stacks block production from Bitcoin block times by introducing:

- **Tenures**: A miner's tenure spans between two Bitcoin blocks. Within a tenure, the miner produces multiple fast Stacks blocks (seconds, not minutes).
- **Signer sets**: Stackers (or their delegates) form a signer set that validates and co-signs blocks using weighted Schnorr threshold signatures (WSTS).
- **70% threshold**: A block requires signatures from signers representing at least 70% of stacked STX.
- **Bitcoin finality**: Once a block's tenure is committed on Bitcoin, the block (and all blocks within that tenure) cannot be reverted without a Bitcoin reorg.
- **TenureChange transactions**: Signal the start/end of a tenure and carry the new VRF proof.

**Code:** Part 5 (Nakamoto) implements this entirely. Chapter 18 (Nakamoto Consensus) covers the block format and finality rules. Chapter 19 (Nakamoto Mining) covers the miner state machine. Chapter 20 (Coordinator) covers tenure processing. Part 7 (Signer) covers the signing protocol.

### SIP-025: Weighted Schnorr Threshold Signatures (WSTS)

Specifies the signing scheme for Nakamoto in two iterations:
- **Iteration 1** (shipped): Concatenated ECDSA signatures -- each signer appends a 65-byte signature to the block header. Simple but O(n) space.
- **Iteration 2** (future): Full WSTS with BFT Paxos for aggregated Schnorr signatures -- O(1) signature size regardless of signer count.

**Code:** Chapter 27 (Signer Architecture) covers `libsigner`'s signing primitives. Chapter 28 (Signer Implementation) covers the `stacks-signer` binary's block validation and signing state machine.

### SIP-034: Dimension-Specific Tenure Extend Variants

A rider to SIP-021 adding granular tenure extension options. Instead of extending the entire tenure budget, miners can request reset of individual cost dimensions (read-count, read-length, runtime, write-count, write-length) via extended `TenureChange` cause codes (0x02-0x06). This handles cases where one dimension is exhausted while others have capacity.

**Code:** Chapter 18 (Nakamoto Consensus) covers `TenureChangeCause` and its 7 variants.

## Networking and Standards

### SIP-003: Stacks P2P Network

Defines the peer-to-peer protocol: message framing (preamble + relayers + payload), big-endian scalars, 4-byte length-prefixed vectors, typed containers. Specifies 18 message types including `Handshake`, `GetNeighbors`, `GetBlocksInv`, `BlocksAvailable`, `Transaction`, `Nack`, `Ping`/`Pong`, and `NatPunchRequest`/`Reply`. The protocol uses a K-regular random graph topology with **MHRWDA** (Metropolis-Hastings Random Walk with Delayed Acceptance) for neighbor selection, prioritizing AS (autonomous system) diversity. Data plane uses HTTP(S); control plane uses the custom wire protocol.

**Code:** Chapter 22 (P2P Protocol) implements the wire protocol and neighbor walk. Chapter 23 (Block Sync) implements inventory synchronization. Chapter 26 (Relay and Mempool) implements message forwarding.

### SIP-009 and SIP-010: Token Standards

**SIP-009** defines the standard NFT trait: `get-last-token-id`, `get-token-uri`, `get-owner`, and `transfer`. Built on Clarity's native `define-non-fungible-token` primitive.

**SIP-010** defines the standard FT trait: `transfer`, `get-name`, `get-symbol`, `get-decimals`, `get-balance`, `get-total-supply`, and `get-token-uri`. Built on Clarity's native `define-fungible-token` primitive.

Both use Clarity traits for composability -- any contract implementing the trait can be used interchangeably by DEXes, wallets, and other contracts.

### SIP-013: Semi-Fungible Token Standard

Extends the token standard patterns with a trait for semi-fungible tokens -- tokens that have both a type ID and a per-type balance (like ERC-1155 in Ethereum).

### SIP-016 and SIP-019: Token Metadata

**SIP-016** defines a JSON schema for token metadata (name, description, image, attributes) referenced by the `get-token-uri` functions in SIP-009/010. **SIP-019** defines a mechanism for emitting on-chain events to notify indexers when token metadata changes.

### SIP-018: Signed Structured Data

Specifies how to sign arbitrary Clarity values off-chain and verify them on-chain. Uses a domain-specific hashing scheme (prefixed with `SIP018` bytes) to prevent cross-chain/cross-application replay. Enables meta-transactions, gasless operations, and off-chain authorization patterns.

### SIP-028: sBTC (Wrapped Bitcoin on Stacks)

SIP-028 specifies **sBTC**, a SIP-010 fungible token backed 1:1 by BTC held in a threshold-signature-controlled UTXO on Bitcoin. Users lock BTC on the Bitcoin chain and mint equivalent sBTC on Stacks; they can redeem sBTC for BTC at any time. The sBTC signer set (which overlaps with the Nakamoto block signers) collectively manages the peg wallet. This SIP specifically addresses signer selection criteria and governance -- technical implementation details are deferred to follow-up SIPs. sBTC relies on Nakamoto's fork resistance (SIP-021) to guarantee the 1:1 peg cannot be broken by a Stacks chain reorg.

**Code:** The sBTC signer integration uses the same `libsigner` infrastructure covered in Chapter 27 (Signer Architecture) and the StackerDB communication layer in Chapter 25 (StackerDB). The `.sbtc` boot contract follows patterns from Chapter 16 (Stacking and PoX).

### SIP-027: Non-Sequential Multisig

Adds new hash modes (0x05 and 0x07) for multisig transactions that allow signing in **any order**, unlike the original format where signers must sign sequentially in a fixed order. Each signer signs the transaction independently, and signatures are collected and assembled into the spending condition.

**Code:** Chapter 14 (Transactions) covers `OrderIndependentMultisigSpendingCondition`.

## SIP-to-Chapter Quick Reference

| SIP | Title | Primary Chapters |
|-----|-------|-----------------|
| 000 | SIP Process | (governance, not in code) |
| 001 | Burn Election / Sortition | 11, 12, 17 |
| 002 | Clarity Language | 5, 6, 7, 8 |
| 003 | P2P Network | 22, 23, 26 |
| 004 | MARF | 13 |
| 005 | Blocks, Transactions, Accounts | 3, 14, 15 |
| 006 | Execution Cost Assessment | 9 |
| 007 | Stacking / PoX | 16 |
| 008 | Parsing Cost Assessment | 9 |
| 009 | NFT Standard | 6 (trait validation) |
| 010 | FT Standard | 6 (trait validation) |
| 012 | Cost Limits Upgrade | 21 |
| 015 | Stacks 2.1 Upgrade | 16, 21 |
| 018 | Signed Structured Data | 7 (VM internals) |
| 020 | Bitwise Operations | 7 |
| 021 | Nakamoto | 18, 19, 20, 27, 28 |
| 022 | Emergency PoX Fix | 16, 21 |
| 023 | Emergency Trait Fix | 6, 21 |
| 024 | Emergency Data Fix | 6, 21 |
| 025 | WSTS | 27, 28 |
| 027 | Non-Sequential Multisig | 14 |
| 028 | sBTC Peg | 25, 27, 16 |
| 029 | Halving Alignment | 21 |
| 031 | Growth Emissions | 21 |
| 033 | Clarity 4 | 7 |
| 034 | Tenure Extend Variants | 18, 19 |
| 035 | secp256r1-verify Clarification | 7 |

## How It Connects

- **Chapter 1 (Architecture Overview)**: Provides the crate-level map that this chapter's SIP-to-code references build on.
- **Chapter 21 (Epoch Transitions)**: The mechanism by which SIPs actually activate in the running network.
- **Appendix A (Glossary)**: Defines terms introduced by SIPs (sortition, tenure, PoX, MARF, etc.).

## Edge Cases and Gotchas

1. **SIP numbers are not contiguous.** Numbers 11, 14, 17, 26, 28-30, 32 are unassigned or withdrawn. Do not assume a gap means a missing SIP.

2. **PoX has four versions.** PoX-1 (SIP-007, Epoch 2.0), PoX-2 (SIP-015, Epoch 2.1), PoX-3 (SIP-022, Epoch 2.3), and PoX-4 (Epoch 2.5+). Each is a separate Clarity boot contract. Only PoX-4 is active in current epochs.

3. **Clarity has four versions.** Clarity 1 (Epoch 2.0), Clarity 2 (Epoch 2.1, SIP-015), Clarity 3 (Epoch 3.0, SIP-021), Clarity 4 (Epoch 3.3, SIP-033). Each adds new functions and may change semantics. Contracts are pinned to their deployment Clarity version.

4. **SIP-021 (Nakamoto) is the biggest protocol change.** It touches nearly every subsystem: block format, consensus, mining, signing, P2P sync, and the coordinator. If you see a `nakamoto/` subdirectory in the code, that is SIP-021's influence.

5. **Emergency SIPs (022-024) happened in rapid succession.** Epochs 2.2, 2.3, and 2.4 activated within weeks of each other. The code contains `Epoch22`, `Epoch23`, `Epoch24` variants that each adjust very specific behaviors. The emergency nature means the changes are surgical and narrowly scoped.

6. **SIP-001's microblock model is obsolete.** Nakamoto (SIP-021) replaced microblocks with fast Nakamoto blocks. Microblock code remains in the codebase for pre-Nakamoto epoch support but is not used in Epoch 3.0+.
