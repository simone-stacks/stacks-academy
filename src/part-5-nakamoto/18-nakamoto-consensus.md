# Chapter 18: Nakamoto Consensus

## Purpose

Nakamoto consensus is the fundamental redesign of how the Stacks blockchain produces and validates blocks. In the pre-Nakamoto epoch 2.x model, each Bitcoin sortition produced exactly one Stacks block, tightly coupling Stacks throughput to Bitcoin's ~10-minute block time. Nakamoto decouples this: a single sortition starts a **tenure** during which a miner can produce many fast blocks (targeting ~5 seconds each), with **signers** providing approval instead of relying solely on the PoX anchor chain. This chapter covers the core data structures, the tenure model, signer-based block approval, shadow blocks, and how Nakamoto consensus differs from epoch 2.

## Key Concepts

### The Tenure Model

The most important mental model shift from epoch 2 to Nakamoto is the concept of a **tenure**:

- In **epoch 2**, each sortition produces exactly one block. One sortition = one block = one miner.
- In **Nakamoto**, each sortition starts a **tenure** -- a sequence of potentially many blocks produced by the winning miner. The miner keeps producing blocks until a new sortition selects a different winner.

A tenure is identified by its **consensus hash** -- the consensus hash of the Bitcoin sortition that chose the miner. This consensus hash is globally unique across all Stacks chain histories and all Bitcoin forks.

From `stackslib/src/chainstate/nakamoto/tenure.rs:17-48`:

```
//! A _tenure_ is the sequence of blocks that a miner produces from a winning sortition.  A tenure
//! can last for the duration of one or more burnchain blocks, and may be extended by Stackers.  As
//! such, every tenure corresponds to exactly one cryptographic sortition with a winning miner.
//! The consensus hash of the winning miner's sortition serves as the _tenure ID_, and it is
//! guaranteed to be globally unique across all Stacks chain histories and burnchain histories.
```

### Tenure Changes

Tenures are created and modified via `TenureChange` transactions, which come in two flavors:

1. **BlockFound** -- triggered by a winning sortition. A new miner takes over block production.
2. **Extended** -- triggered by Stackers. Resets the tenure's execution budget so the current miner can continue producing blocks.

SIP-034 adds more granular extension causes that reset individual cost dimensions:

```rust
// stackslib/src/chainstate/stacks/mod.rs:705-717
pub enum TenureChangeCause {
    /// A valid winning block-commit
    BlockFound = 0,
    /// Extend all dimensions
    Extended = 1,
    // NEW in SIP-034: extend specific dimensions
    ExtendedRuntime = 2,
    ExtendedReadCount = 3,
    ExtendedReadLength = 4,
    ExtendedWriteCount = 5,
    ExtendedWriteLength = 6,
}
```

### Signers Replace PoX Anchor Validation

In epoch 2, block validity was enforced purely by the sortition mechanism and PoX. In Nakamoto, a set of **signers** (selected from the PoX reward set) must approve each block by providing a weighted threshold of ECDSA signatures. This is the cryptographic mechanism that prevents miners from producing invalid or conflicting blocks.

## Architecture

The Nakamoto consensus system is spread across several modules:

```
stackslib/src/chainstate/nakamoto/
    mod.rs          -- NakamotoBlock, NakamotoBlockHeader, NakamotoChainState
    tenure.rs       -- Tenure tracking (NakamotoTenureEvent, NakamotoTenureEventId)
    signer_set.rs   -- NakamotoSigners, reward set calculation
    shadow.rs       -- Shadow blocks (emergency recovery)
    miner.rs        -- NakamotoBlockBuilder (covered in Chapter 19)
    staging_blocks.rs -- Block staging DB for unprocessed blocks
    keys.rs         -- MARF key helpers for tenure lookups
    coordinator/    -- Nakamoto-specific coordinator logic (covered in Chapter 20)
```

## Key Data Structures

### NakamotoBlockHeader

The block header is the core consensus-critical structure. Defined at `stackslib/src/chainstate/nakamoto/mod.rs:633-668`:

```rust
pub struct NakamotoBlockHeader {
    pub version: u8,
    /// Total number of StacksBlock and NakamotoBlocks preceding this block
    pub chain_length: u64,
    /// Total amount of BTC spent producing the sortition that selected this block's miner
    pub burn_spent: u64,
    /// Consensus hash of the burnchain block that selected this tenure
    pub consensus_hash: ConsensusHash,
    /// Index block hash of the immediate parent of this block
    pub parent_block_id: StacksBlockId,
    /// Root of a SHA512/256 merkle tree over all this block's transactions
    pub tx_merkle_root: Sha512Trunc256Sum,
    /// MARF trie root hash after this block has been processed
    pub state_index_root: TrieHash,
    /// Unix timestamp of when this block was mined
    pub timestamp: u64,
    /// Recoverable ECDSA signature from the tenure's miner
    pub miner_signature: MessageSignature,
    /// Recoverable ECDSA signatures from the active signer set (ordered by reward set order)
    pub signer_signature: Vec<MessageSignature>,
    /// Bitvec indicating whether reward addresses should be punished (burned) or not
    pub pox_treatment: BitVec<4000>,
}
```

Key observations:

- **`consensus_hash`** ties this block to a specific tenure (sortition). All blocks in the same tenure share the same `consensus_hash`.
- **`parent_block_id`** is a `StacksBlockId`, which is the hash of the parent block's hash + consensus hash. This creates a globally unique chain of blocks.
- **`miner_signature`** is the miner's signature over all fields except both signatures.
- **`signer_signature`** is the set of signer signatures over all fields except `signer_signature` itself (but including `miner_signature`).
- **`pox_treatment`** is a bitvec where each bit corresponds to a reward address; a 0 bit means the address's PoX rewards are burned (punished).
- **`version`** uses the high bit (`0x80`) to indicate shadow blocks.

### NakamotoBlock

The block itself is simply a header plus transactions (`stackslib/src/chainstate/nakamoto/mod.rs:707-711`):

```rust
pub struct NakamotoBlock {
    pub header: NakamotoBlockHeader,
    pub txs: Vec<StacksTransaction>,
}
```

### TenureChangePayload

The payload carried by `TenureChange` transactions (`stackslib/src/chainstate/stacks/mod.rs:833-853`):

```rust
pub struct TenureChangePayload {
    /// Consensus hash of this tenure's sortition
    pub tenure_consensus_hash: ConsensusHash,
    /// Consensus hash of the previous tenure's sortition
    pub prev_tenure_consensus_hash: ConsensusHash,
    /// Current consensus hash on the underlying burnchain (last-seen sortition)
    pub burn_view_consensus_hash: ConsensusHash,
    /// StacksBlockId of the last block from the previous tenure
    pub previous_tenure_end: StacksBlockId,
    /// Number of blocks produced since the last sortition-linked tenure
    pub previous_tenure_blocks: u32,
    /// Cause of this tenure change
    pub cause: TenureChangeCause,
    /// ECDSA public key hash of the current tenure's miner
    pub pubkey_hash: Hash160,
}
```

The three consensus hashes serve different purposes:
- **`tenure_consensus_hash`** identifies *which* sortition created this tenure.
- **`prev_tenure_consensus_hash`** identifies the parent tenure's sortition.
- **`burn_view_consensus_hash`** is the Stackers' view of the most recent sortition when they created the tenure change.

### NakamotoTenureEvent

Each tenure change is recorded as an event in the `nakamoto_tenure_events` table (`stackslib/src/chainstate/nakamoto/tenure.rs:215-234`):

```rust
pub struct NakamotoTenureEvent {
    pub tenure_id_consensus_hash: ConsensusHash,
    pub prev_tenure_id_consensus_hash: ConsensusHash,
    pub burn_view_consensus_hash: ConsensusHash,
    pub cause: TenureChangeCause,
    pub block_hash: BlockHeaderHash,
    pub block_id: StacksBlockId,
    pub coinbase_height: u64,
    pub num_blocks_confirmed: u32,
}
```

Note that `coinbase_height` counts only sortition-induced tenures (not extensions), providing the tenure-based equivalent of block height.

## Code Walkthrough: Signature Verification

The signer verification logic is critical to Nakamoto consensus. It lives in `NakamotoBlockHeader::verify_signer_signatures` at `stackslib/src/chainstate/nakamoto/mod.rs:846-926`.

**Step 1: Load the signer set**

```rust
let Some(signers) = &reward_set.signers else {
    return Err(ChainstateError::InvalidStacksBlock(
        "No signers in the reward set".to_string(),
    ));
};
```

**Step 2: Handle shadow blocks (special case)**

Shadow blocks are treated as if every signer signed them:

```rust
if self.is_shadow_block() {
    return Ok(self.get_shadow_signer_weight(reward_set)?);
}
```

**Step 3: Build a lookup map and iterate signatures**

```rust
let mut signers_by_pk: HashMap<_, _> = signers
    .iter()
    .enumerate()
    .map(|(i, signer)| (&signer.signing_key, (signer, i)))
    .collect();

for signature in self.signer_signature.iter() {
    let public_key = Secp256k1PublicKey::recover_to_pubkey(message.bits(), signature)?;
    // ... look up signer by recovered public key
    // ... accumulate weight, enforce ordering
}
```

**Step 4: Enforce ordering and threshold**

Signatures must appear in reward-set order (no duplicates, no out-of-order). The signer is removed from the map after matching, preventing duplicate signatures. The threshold is computed as:

```rust
// stackslib/src/core/mod.rs:184
pub const NAKAMOTO_SIGNER_BLOCK_APPROVAL_THRESHOLD: u64 = 7;
```

This constant represents 7/10 = 70%. The actual threshold calculation at `mod.rs:930-943` computes `ceil(total_weight * 7 / 10)`.

## Code Walkthrough: Two-Phase Signing

The block header has two separate signature hashes:

**Miner signature hash** (`mod.rs:766-778`) -- covers everything *except* both signatures:
```
version | chain_length | burn_spent | consensus_hash | parent_block_id |
tx_merkle_root | state_index_root | timestamp | pox_treatment
```

**Signer signature hash** (`mod.rs:783-797`) -- covers everything *except* `signer_signature`, but *includes* `miner_signature`:
```
version | chain_length | burn_spent | consensus_hash | parent_block_id |
tx_merkle_root | state_index_root | timestamp | miner_signature | pox_treatment
```

This two-phase approach means:
1. The miner first signs the block content (without any signatures).
2. Signers then sign a hash that includes the miner's signature, binding their approval to this specific miner's block.

The `block_hash()` method (`mod.rs:807-814`) returns the same hash as the signer signature hash, ensuring the block's identity includes the miner's commitment.

## Code Walkthrough: Block Acceptance

When a new Nakamoto block arrives, it goes through `NakamotoChainState::accept_block` (`mod.rs:2434`):

1. Check if we already have this block (skip if so).
2. Validate the block against burnchain state (consensus hash, burn totals).
3. Verify the tenure-change transaction (if present) is consistent.
4. Verify the VRF proof (if this is a tenure-start block).
5. **Verify signer signatures** against the active reward set.
6. Store the block in the staging blocks DB for later processing.

Processing happens later in `process_next_nakamoto_block` (`mod.rs:1885`), which picks the next ready block from the staging DB and applies it to the chainstate (executing transactions, updating MARF state).

## Shadow Blocks

Shadow blocks (`stackslib/src/chainstate/nakamoto/shadow.rs`) are an emergency recovery mechanism defined by the comment at the top of the file:

```
/// In the event of an emergency chain halt, a SIP will be written to declare that a chain halt has
/// happened, and what transactions and blocks (if any) need to be mined at which burnchain block
/// heights to recover the chain.
```

Shadow blocks are:
- **Not mined or relayed** -- they are synthesized as part of an emergency node upgrade.
- **Inserted directly into the staging DB** as a schema update.
- **Identified by the high bit** of the version field: `version & 0x80 != 0`.
- **Given full signer weight** without actual signatures (since they're consensus-defined).
- **Coinbase recipients must be the burn address** -- shadow blocks do not award STX.

Use cases include:
- Creating a PoX anchor block when the prepare phase has no block-commits.
- Adding extra `stack-stx` transactions to ensure a healthy signer set.

## Tenure Extensions

When a sortition produces no winner (or the winner goes offline), the current miner's tenure continues. Stackers can issue a `TenureChange` with `cause: Extended` to reset the execution budget, allowing the miner to keep producing blocks beyond the normal per-tenure cost limits.

SIP-034 introduced granular extensions (`ExtendedRuntime`, `ExtendedReadCount`, etc.) that reset only a single dimension of the execution cost, giving Stackers finer-grained control over how much additional work a tenure can perform.

The `MinerTenureInfoCause` enum in `miner.rs:106-115` maps these causes:

```rust
pub enum MinerTenureInfoCause {
    NoTenureChange,
    BlockFound,
    Extended,           // resets all dimensions
    ExtendedRuntime,    // SIP-034: reset only runtime
    ExtendedReadCount,
    ExtendedReadLength,
    ExtendedWriteCount,
    ExtendedWriteLength,
}
```

## How Consensus Differs from Epoch 2

| Aspect | Epoch 2 | Nakamoto |
|--------|---------|----------|
| Blocks per sortition | Exactly 1 | Many (limited by cost budget) |
| Block time | ~10 min (tied to Bitcoin) | ~5 seconds |
| Block validation | Sortition + PoX rules | Sortition + signer threshold (70%) |
| Finality | Probabilistic (follows Bitcoin) | Bitcoin finality + signer approval |
| Fork resolution | Longest chain with most burns | No Stacks forks (single canonical tip per Bitcoin fork) |
| Miner identity | Per-block | Per-tenure |
| Coinbase maturity | Per-block height | Per-coinbase height (tenure count) |
| Cost budget | Per-block | Per-tenure (resettable by extension) |
| Mempool GC | By Stacks height | By receive time |
| PoX anchor block | Selected in prepare phase | First block of last tenure in reward phase |

The elimination of Stacks-level forks is perhaps the most significant change. In epoch 2, miners could produce competing blocks at the same height, requiring fork resolution. In Nakamoto, the signer set ensures that at most one block can exist at any given chain height (with >30% honest signers). This is enforced by the 70% signing threshold -- no two conflicting blocks can both gather enough signatures.

## How It Connects

- **Chapter 16 (Stacking and PoX)**: The signer set is derived from the PoX reward set, connecting consensus directly to stacking.
- **Chapter 19 (Nakamoto Mining)**: The `NakamotoBlockBuilder` uses the structures defined here to construct blocks within a tenure.
- **Chapter 20 (Coordinator)**: The coordinator orchestrates the processing of Nakamoto blocks and sortitions.
- **Chapter 12 (Sortition DB)**: Sortitions still select miners, but now they start tenures rather than producing individual blocks.
- **Chapter 27 (Signer Architecture)**: Defines the `libsigner` message types and WSTS threshold signing scheme that produce the `signer_signature` field in `NakamotoBlockHeader`.
- **Chapter 28 (Signer Implementation)**: The `stacks-signer` binary that validates block proposals and accumulates threshold signatures.

## Edge Cases and Gotchas

1. **Tenure can span multiple Bitcoin blocks**: If successive sortitions have no winner, the current tenure continues. The `burn_view_consensus_hash` in `TenureChangePayload` tracks the Stackers' view of the latest sortition.

2. **First Nakamoto tenure is special**: The first Nakamoto block's parent is the last epoch 2 block. The builder enforces that building atop an epoch 2 block requires both a tenure-change and a coinbase (`miner.rs:279-290`).

3. **Signature ordering is enforced**: Signer signatures must appear in reward-set index order. Out-of-order signatures cause the block to be rejected. This prevents signature malleability attacks.

4. **`pox_treatment` is a bitvec, not a list**: Each bit maps to a reward address. A 0 bit means "burn this address's rewards." The maximum size is 4000 entries.

5. **Block hash = signer signature hash**: The `block_hash()` method returns the signer signature hash (which includes the miner signature). This means the block's identity depends on the miner's commitment, preventing block-stealing.

6. **Shadow blocks bypass signature verification**: They receive full weight automatically. This is safe because they're consensus-defined (everyone has the same shadow blocks from the same node upgrade).

7. **Coinbase height vs chain length**: `chain_length` counts every block (both epoch 2 and Nakamoto). `coinbase_height` counts only sortition-induced tenures. Miner rewards mature based on coinbase height, not chain length.
