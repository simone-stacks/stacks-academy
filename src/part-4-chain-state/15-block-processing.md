# Chapter 15: Block Processing

## Purpose

Block processing is the core state machine of the Stacks blockchain. When a new Stacks block arrives -- whether from the network or from the local miner -- the block processor validates it, executes every transaction against the Clarity VM, updates the MARF state index, schedules miner rewards, and commits the result. This chapter covers `StacksChainState`, the central coordinator for all chain state operations, and traces the block processing pipeline from receipt to commitment.

The implementation lives primarily in `stackslib/src/chainstate/stacks/db/`, with the main struct in `db/mod.rs` and block processing logic in `db/blocks.rs`.

## Key Concepts

### Two Block Types

Epoch 2.x has two block types:

- **Anchored blocks** (`StacksBlock`): On-chain blocks produced by the sortition winner. Each anchored block is tied to a Bitcoin block via a block-commit transaction.
- **Microblocks** (`StacksMicroblock`): Off-chain blocks streamed between anchored blocks by the current tenure holder. Microblocks provide fast confirmation but can be orphaned.

In Nakamoto (Epoch 3.0+), microblocks are eliminated entirely (see Chapter 18).

### The StacksChainState Struct

This is the main entry point for all chain state operations:

```rust
// stackslib/src/chainstate/stacks/db/mod.rs:103-116
pub struct StacksChainState {
    pub mainnet: bool,
    pub chain_id: u32,
    pub clarity_state: ClarityInstance,
    pub nakamoto_staging_blocks_conn: NakamotoStagingBlocksConn,
    pub state_index: MARF<StacksBlockId>,
    pub blocks_path: String,
    pub clarity_state_index_path: String,
    pub clarity_state_index_root: String,
    pub root_path: String,
    pub unconfirmed_state: Option<UnconfirmedState>,
    pub fault_injection: StacksChainStateFaults,
    marf_opts: Option<MARFOpenOpts>,
}
```

Key fields:
- **`clarity_state`**: The Clarity VM instance that executes smart contracts
- **`state_index`**: A `MARF<StacksBlockId>` that indexes block headers and metadata
- **`nakamoto_staging_blocks_conn`**: Connection to the Nakamoto staging blocks database
- **`unconfirmed_state`**: Tracks microblock stream state for mempool processing

### StacksHeaderInfo

Every processed block produces a `StacksHeaderInfo` record:

```rust
// stackslib/src/chainstate/stacks/db/mod.rs:166-192
pub struct StacksHeaderInfo {
    pub anchored_header: StacksBlockHeaderTypes,
    pub microblock_tail: Option<StacksMicroblockHeader>,
    pub stacks_block_height: u64,
    pub index_root: TrieHash,
    pub consensus_hash: ConsensusHash,
    pub burn_header_hash: BurnchainHeaderHash,
    pub burn_header_height: u32,
    pub burn_header_timestamp: u64,
    pub anchored_block_size: u64,
    pub burn_view: Option<ConsensusHash>,
    pub total_tenure_size: u64,
}
```

The `anchored_header` field is an enum that handles both Epoch 2 and Nakamoto headers:

```rust
// stackslib/src/chainstate/stacks/db/mod.rs:148-152
pub enum StacksBlockHeaderTypes {
    Epoch2(StacksBlockHeader),
    Nakamoto(NakamotoBlockHeader),
}
```

## Architecture: The Block Processing Pipeline

Processing an Epoch 2.x anchored block follows this pipeline:

```
1. Receive block
2. Validate header (parent exists, work score, VRF proof)
3. Validate burnchain linkage (consensus hash matches sortition)
4. setup_block() -- Open Clarity transaction, process microblocks, find matured rewards
5. process_block_transactions() -- Execute each transaction
6. finish_block() -- Commit miner rewards, write header to MARF
7. append_block() -- Store block data, emit receipt
```

### Step 1-3: Header and Burnchain Validation

Before any execution, the block header is validated against the burnchain:

- The `consensus_hash` must correspond to a valid sortition
- The `parent_block` must be the previous block in this fork
- The `total_work` score must equal the parent's work plus the burn amount
- The VRF `proof` must be valid for the miner's registered VRF key
- The `parent_microblock` and `parent_microblock_sequence` must match the microblock stream being confirmed

This validation happens in `validate_anchored_block_burnchain` (blocks.rs:3099).

### Step 4: setup_block()

The `setup_block` method (blocks.rs:4990-5170) prepares the execution context:

```rust
// blocks.rs:4990 (signature)
pub fn setup_block<'a, 'b>(
    chainstate_tx: &'b mut ChainstateTx,
    clarity_instance: &'a mut ClarityInstance,
    burn_dbconn: &'b dyn BurnStateDB,
    sortition_dbconn: &'b dyn SortitionDBRef,
    conn: &Connection,
    pox_constants: &PoxConstants,
    chain_tip: &StacksHeaderInfo,
    burn_tip: &BurnchainHeaderHash,
    burn_tip_height: u32,
    parent_consensus_hash: &ConsensusHash,
    parent_header_hash: &BlockHeaderHash,
    parent_microblocks: &[StacksMicroblock],
    mainnet: bool,
    miner_id_opt: Option<usize>,
) -> Result<SetupBlockResult<'a, 'b>, Error>
```

This method:

1. **Finds matured miner rewards**: Rewards are delayed by a maturation period. `get_scheduled_block_rewards` and `get_parent_matured_miner` look up which miners can now claim their coinbase.

2. **Loads burnchain operations**: Retrieves `StackStxOp`, `TransferStxOp`, `DelegateStxOp`, and `VoteForAggregateKeyOp` from the sortition DB for processing.

3. **Opens a Clarity block**: Creates a `ClarityTx` that wraps a MARF transaction for the new block:

```rust
// blocks.rs:5058-5066
let mut clarity_tx = StacksChainState::chainstate_block_begin(
    chainstate_tx,
    clarity_instance,
    burn_dbconn,
    parent_consensus_hash,
    parent_header_hash,
    &MINER_BLOCK_CONSENSUS_HASH,
    &MINER_BLOCK_HEADER_HASH,
);
```

4. **Processes parent microblocks**: Executes the microblock transactions in the parent's microblock stream, accumulating their execution costs and fees.

5. **Handles epoch transitions**: If the block crosses an epoch boundary (e.g., from Epoch 2.05 to 2.1), applies the transition logic. Blocks that cross epoch boundaries cannot confirm microblocks.

The result is a `SetupBlockResult` containing the open Clarity transaction, microblock receipts, matured rewards, epoch information, and burn operations.

### Step 5: process_block_transactions()

After setup, `process_block_transactions` (`blocks.rs:4489`) executes the anchored block's transactions:

```rust
pub fn process_block_transactions(
    clarity_tx: &mut ClarityTx,
    block_txs: &[StacksTransaction],
    mut tx_index: u32,
) -> Result<(u128, u128, Vec<StacksTransactionReceipt>), Error>
```

The return tuple contains `(total_fees, total_burns, receipts)` -- fees and burns are `u128` values accumulated across all transactions.

Each transaction goes through:
1. **Authorization check**: Verify signatures, check nonces, debit fees
2. **Payload execution**: Run the Clarity code (for contract calls/deployments) or transfer STX
3. **Post-condition evaluation**: Check that all post-conditions are satisfied
4. **Receipt generation**: Record the execution result, events, and costs

Failed transactions still consume their fee (the fee was debited before execution). Post-condition failures cause the transaction state changes to be rolled back, but the fee is still paid.

### Step 6: finish_block()

Once all transactions are processed, `finish_block` finalizes the block state:

1. Distributes matured miner rewards to the appropriate accounts
2. Processes burnchain STX operations (stack-stx, transfer-stx, delegate-stx)
3. Records the microblock public key hash (to prevent reuse)
4. Updates the `.signers` boot contract if needed
5. Computes the final MARF root hash

### Step 7: append_block()

The `append_block` method (blocks.rs:5338) orchestrates the full pipeline and commits the result:

```rust
pub fn append_block<'a>(
    chainstate_tx: &mut ChainstateTx,
    clarity_instance: &'a mut ClarityInstance,
    burn_dbconn: &mut SortitionHandleTx,
    pox_constants: &PoxConstants,
    parent_chain_tip: &StacksHeaderInfo,
    chain_tip_consensus_hash: &ConsensusHash,
    chain_tip_burn_header_hash: &BurnchainHeaderHash,
    chain_tip_burn_header_height: u32,
    chain_tip_burn_header_timestamp: u64,
    block: &StacksBlock,
    block_size: u64,
    microblocks: &[StacksMicroblock],
    burnchain_commit_burn: u64,
    burnchain_sortition_burn: u64,
    do_not_advance: bool,
) -> Result<(StacksEpochReceipt, PreCommitClarityBlock, Option<RewardSetData>), Error>
```

Key validations in `append_block`:
- **Epoch boundary check**: If the parent crossed an epoch boundary, the child cannot confirm microblocks (blocks.rs:5374-5393)
- **Microblock pubkey uniqueness**: The block's `microblock_pubkey_hash` must not have been used before on this fork
- **Microblock continuity**: The `parent_microblock` and `parent_microblock_sequence` in the block header must match the actual microblock stream

The method returns a `StacksEpochReceipt`:

```rust
// stackslib/src/chainstate/stacks/db/mod.rs:203-222
pub struct StacksEpochReceipt {
    pub header: StacksHeaderInfo,
    pub tx_receipts: Vec<StacksTransactionReceipt>,
    pub matured_rewards: Vec<MinerReward>,
    pub matured_rewards_info: Option<MinerRewardInfo>,
    pub parent_microblocks_cost: ExecutionCost,
    pub anchored_block_cost: ExecutionCost,
    pub parent_burn_block_hash: BurnchainHeaderHash,
    pub parent_burn_block_height: u32,
    pub parent_burn_block_timestamp: u64,
    pub evaluated_epoch: StacksEpochId,
    pub epoch_transition: bool,
    pub signers_updated: bool,
    pub coinbase_height: u64,
}
```

## Block Headers

### Epoch 2 Block Header

```rust
// stackslib/src/chainstate/stacks/mod.rs:1165-1176
pub struct StacksBlockHeader {
    pub version: u8,
    pub total_work: StacksWorkScore,
    pub proof: VRFProof,
    pub parent_block: BlockHeaderHash,
    pub parent_microblock: BlockHeaderHash,
    pub parent_microblock_sequence: u16,
    pub tx_merkle_root: Sha512Trunc256Sum,
    pub state_index_root: TrieHash,
    pub microblock_pubkey_hash: Hash160,
}
```

- **`total_work`**: Cumulative work score (burn amount + block count) up to and including this block
- **`proof`**: VRF proof from the miner, proving they won the sortition
- **`parent_block`**: Hash of the parent anchored block header
- **`parent_microblock`**: Hash of the last microblock being confirmed (or empty sentinel)
- **`tx_merkle_root`**: Merkle root of the transaction IDs in this block
- **`state_index_root`**: Root hash of the MARF after processing this block (enables state proofs)
- **`microblock_pubkey_hash`**: Hash160 of the public key this miner will use to sign microblocks

The block hash is computed as SHA-512/256 of the serialized header. The `StacksBlockId` (used as the MARF key) is the SHA-512/256 of the consensus hash concatenated with the block header hash.

### Microblock Header

```rust
// stackslib/src/chainstate/stacks/mod.rs:1179-1186
pub struct StacksMicroblockHeader {
    pub version: u8,
    pub sequence: u16,
    pub prev_block: BlockHeaderHash,
    pub tx_merkle_root: Sha512Trunc256Sum,
    pub signature: MessageSignature,
}
```

Microblocks form a linked list via `prev_block`. The `sequence` monotonically increases. The `signature` is produced using the private key corresponding to the parent anchored block's `microblock_pubkey_hash`.

## Block and Microblock Structs

```rust
// stackslib/src/chainstate/stacks/mod.rs:1150-1162
pub struct StacksBlock {
    pub header: StacksBlockHeader,
    pub txs: Vec<StacksTransaction>,
}

pub struct StacksMicroblock {
    pub header: StacksMicroblockHeader,
    pub txs: Vec<StacksTransaction>,
}
```

Block size limits:
- `MAX_BLOCK_LEN`: 2 MB (mod.rs:74)
- `MAX_EPOCH_SIZE`: 2 MB total across anchored block + microblocks (mod.rs:1214)
- `MAX_MICROBLOCK_SIZE`: 64 KB per microblock (mod.rs:1218)

## MARF Integration

Every processed block creates a new trie in the `state_index` MARF. The block's `StacksBlockId` (SHA-512/256 of consensus_hash + block_hash) serves as the trie identifier. The MARF stores:

1. Block header metadata (height, parent, timestamps)
2. Account nonces and balances (via the Clarity MARF)
3. Block height mappings (height <-> block hash)

The `state_index_root` in the block header is the MARF root hash at the time the block was sealed. This root hash enables light clients to verify state proofs: given a key and its value, a Merkle proof can demonstrate membership against the published root.

## Epoch Transitions

The block processor handles epoch transitions transparently. When `setup_block` detects that the current block runs in a different epoch than its parent:

1. The `applied_epoch_transition` flag is set to `true`
2. New boot code may be deployed (e.g., pox-2, pox-3, pox-4, signers contracts)
3. Execution cost limits may change
4. New transaction types become valid
5. Microblock confirmation across epochs is forbidden

The `evaluated_epoch` in the receipt tells callers which epoch rules were applied.

## The process_blocks Pipeline

At the top level, `process_blocks` (blocks.rs:6360) is the entry point called by the coordinator:

```rust
pub fn process_blocks<T: BlockEventDispatcher>(
    &mut self,
    burnchain_dbconn: &mut SortitionHandleTx,
    max_blocks: usize,
    event_dispatcher: Option<&T>,
) -> Result<Vec<(StacksEpochReceipt, ...)>, Error>
```

This method:
1. Finds staging blocks that are ready to process (have a known parent)
2. For each block, calls `append_block`
3. Commits the MARF and SQLite transactions on success
4. Dispatches block events to observers
5. Returns receipts for all processed blocks

## How It Connects

- **MARF** (Chapter 13): The `state_index` MARF provides authenticated state storage. Each block creates a new trie.
- **Transactions** (Chapter 14): Blocks contain `Vec<StacksTransaction>` that are executed by `process_block_transactions`.
- **Clarity VM** (Chapters 5-9): Transaction execution happens inside a `ClarityTx`, which wraps the Clarity interpreter.
- **Sortition DB** (Chapter 12): Block validation requires checking consensus hashes and sortition results.
- **Stacking/PoX** (Chapter 16): The block processor invokes PoX boot contracts and processes burnchain STX operations.
- **Mining** (Chapter 17): Miners use `StacksBlockBuilder` to create blocks that will be validated by this pipeline.
- **Coordinator** (Chapter 20): The coordinator calls `process_blocks` in response to new burnchain blocks.

## Edge Cases and Gotchas

1. **Microblock orphaning**: When a block does not confirm the expected microblock stream (or confirms fewer microblocks), the unconfirmed microblocks are orphaned. Their transactions return to the mempool.

2. **Epoch boundary microblock prohibition**: A block that crosses an epoch boundary (e.g., the first block in Epoch 2.1) cannot confirm any microblocks from its parent. This prevents executing microblock transactions under the wrong epoch rules.

3. **Miner reward maturation**: Miner rewards are not paid immediately. They mature after a delay (currently 100 blocks). The `find_mature_miner_rewards` function looks back through the chain to find rewards that are now claimable.

4. **The MINER_BLOCK sentinel**: During block processing, the new block is temporarily given the consensus hash `MINER_BLOCK_CONSENSUS_HASH` and header hash `MINER_BLOCK_HEADER_HASH` (both `[1u8; 20/32]`). The real hashes are committed only when the block is finalized.

5. **Cost budget**: The total execution cost of microblock transactions plus anchored block transactions must fit within a single block's cost budget. If parent microblocks exhaust the budget, the anchored block cannot include compute-heavy transactions.

6. **`do_not_advance` flag**: When set, the block is processed but the chain tip is not advanced. This is used during reorgs to process blocks that are not yet the canonical tip.
