# Chapter 19: Nakamoto Mining

## Purpose

Nakamoto mining is fundamentally different from epoch 2 mining. In epoch 2, a miner won a sortition, produced one block, and was done. In Nakamoto, a miner wins a sortition, starts a **tenure**, and produces a continuous stream of blocks until the next sortition selects a new miner. This chapter covers the `NakamotoBlockBuilder`, the tenure lifecycle, how blocks are proposed to signers, and the mechanics of transaction selection within a tenure.

## Key Concepts

### Tenure Lifecycle

A miner's tenure follows this lifecycle:

1. **Win a sortition** -- The miner's block-commit is selected by the VRF-weighted cryptographic sortition.
2. **Receive a TenureChange** -- Signers produce a `TenureChange` transaction with `cause: BlockFound`, acknowledging the new miner.
3. **Produce the first block** -- Contains the `TenureChange` transaction and a `Coinbase` transaction. This is the tenure-start block.
4. **Produce subsequent blocks** -- Each block contains regular transactions selected from the mempool. These blocks share the same `consensus_hash` but increment `chain_length`.
5. **Tenure ends** -- When the next sortition's `TenureChange` arrives, or the tenure's execution budget is exhausted.

A tenure can also be **extended** if signers issue a `TenureChange` with `cause: Extended`, which resets the execution budget.

### Block vs Tenure Budget

In Nakamoto, the execution cost limit applies to the **tenure**, not individual blocks. A miner can produce many blocks, but their total execution cost across all blocks in the tenure must stay within the tenure budget. Individual blocks can also have a soft limit (a percentage of the remaining budget) to spread capacity across multiple blocks.

## Architecture

Mining code lives primarily in two files:

```
stackslib/src/chainstate/nakamoto/miner.rs  -- NakamotoBlockBuilder, MinerTenureInfo
stackslib/src/chainstate/stacks/miner.rs    -- BlockBuilder trait, select_and_apply_transactions
```

The builder follows a pipeline:
```
NakamotoBlockBuilder::new()
    -> load_tenure_info()     -- opens DB connections, loads reward set
    -> tenure_begin()         -- sets up ClarityTx, processes epoch transitions
    -> mine transactions      -- select_and_apply_transactions()
    -> mine_nakamoto_block()  -- finalize header, sign
    -> tenure_finish()        -- commit MARF trie
```

## Key Data Structures

### NakamotoBlockBuilder

The main builder struct (`stackslib/src/chainstate/nakamoto/miner.rs:77-101`):

```rust
pub struct NakamotoBlockBuilder {
    /// Parent block header (None for genesis)
    parent_header: Option<StacksHeaderInfo>,
    /// Signed coinbase tx, if starting a new tenure
    coinbase_tx: Option<StacksTransaction>,
    /// Tenure change tx, if starting or extending a tenure
    tenure_tx: Option<StacksTransaction>,
    /// Total burn this block represents
    total_burn: u64,
    /// Matured miner rewards to process, if any
    pub(crate) matured_miner_rewards_opt: Option<MaturedMinerRewards>,
    /// Bytes of space consumed so far
    pub bytes_so_far: u64,
    /// Transactions selected for this block
    txs: Vec<StacksTransaction>,
    /// Header being filled in
    pub header: NakamotoBlockHeader,
    /// Optional soft limit for this block's budget usage
    soft_limit: Option<ExecutionCost>,
    /// Max percentage of budget for contract calls before falling back to transfers
    contract_limit_percentage: Option<u8>,
    /// Maximum size of the whole tenure in bytes
    pub max_tenure_bytes: u64,
}
```

### NakamotoTenureInfo

Passed to `build_nakamoto_block` to describe the tenure context (`miner.rs:49-55`):

```rust
pub struct NakamotoTenureInfo {
    /// Coinbase tx, if this is a new tenure
    pub coinbase_tx: Option<StacksTransaction>,
    /// Tenure change transaction from Stackers
    pub tenure_change_tx: Option<StacksTransaction>,
}
```

The `cause()` method determines the type of tenure event:

```rust
impl NakamotoTenureInfo {
    pub fn cause(&self) -> MinerTenureInfoCause {
        let Some(tenure_change_tx) = self.tenure_change_tx.as_ref() else {
            return MinerTenureInfoCause::NoTenureChange;
        };
        // ... extract cause from the TenureChangePayload
    }
}
```

### MinerTenureInfo

The internal state bag used during block construction, created by `load_tenure_info` (`miner.rs:189-206`):

```rust
pub struct MinerTenureInfo<'a> {
    pub chainstate_tx: ChainstateTx<'a>,
    pub clarity_instance: &'a mut ClarityInstance,
    pub burn_tip: BurnchainHeaderHash,
    pub burn_tip_height: u32,
    pub mainnet: bool,
    pub parent_consensus_hash: ConsensusHash,
    pub parent_header_hash: BlockHeaderHash,
    pub parent_stacks_block_height: u64,
    pub parent_burn_block_height: u32,
    pub coinbase_height: u64,
    pub cause: MinerTenureInfoCause,
    pub active_reward_set: RewardSet,
    pub tenure_block_commit_opt: Option<LeaderBlockCommitOp>,
    pub ephemeral: bool,
}
```

This struct owns database connections ensuring they live long enough for block processing to complete.

### BlockMetadata

Returned from `build_nakamoto_block` (`miner.rs:209-221`):

```rust
pub struct BlockMetadata {
    /// The block that was built
    pub block: NakamotoBlock,
    /// Execution cost consumed so far by the current tenure
    pub tenure_consumed: ExecutionCost,
    /// Cost budget for the current tenure
    pub tenure_budget: ExecutionCost,
    /// Size of blocks in the current tenure in bytes
    pub tenure_size: u64,
    /// Events emitted by transactions in this block
    pub tx_events: Vec<TransactionEvent>,
}
```

### MinerTenureInfoCause

Classifies what kind of tenure event is occurring (`miner.rs:106-115`):

```rust
pub enum MinerTenureInfoCause {
    NoTenureChange,       // continuation block within same tenure
    BlockFound,           // new tenure from sortition
    Extended,             // Stackers extended all dimensions
    ExtendedRuntime,      // SIP-034: extend runtime only
    ExtendedReadCount,    // SIP-034: extend read count only
    ExtendedReadLength,   // SIP-034: extend read length only
    ExtendedWriteCount,   // SIP-034: extend write count only
    ExtendedWriteLength,  // SIP-034: extend write length only
}
```

Key methods:
- `is_new_tenure()` -- true only for `BlockFound`.
- `is_tenure_extension_any()` -- true for all `Extended*` variants.
- `is_sip034_tenure_extension()` -- true for the single-dimension SIP-034 variants.

## Code Walkthrough: Building a Block

The main entry point is `NakamotoBlockBuilder::build_nakamoto_block` at `miner.rs:639`:

### Step 1: Create the Builder

```rust
// miner.rs:669-680
let mut builder = NakamotoBlockBuilder::new(
    parent_stacks_header,
    tenure_id_consensus_hash,
    total_burn,
    tenure_info.tenure_change_tx(),
    tenure_info.coinbase_tx(),
    signer_bitvec_len,
    None,                          // soft_limit set later
    settings.mempool_settings.contract_cost_limit_percentage,
    None,                          // timestamp
    settings.max_tenure_bytes,
)?;
```

The constructor (`miner.rs:262-317`) validates the context:
- If building atop an epoch 2 block, both a tenure change and coinbase are required.
- The header is initialized with `NakamotoBlockHeader::from_parent_empty()`, setting `chain_length = parent + 1`, `consensus_hash`, and `timestamp = max(parent_timestamp, now)`.

### Step 2: Load Tenure Info

```rust
// miner.rs:684-685
let mut miner_tenure_info =
    builder.load_tenure_info(&mut chainstate, burn_dbconn, tenure_info.cause())?;
```

This opens database transactions, loads the reward set that elected this miner, and determines the coinbase height. The coinbase height increments only for `BlockFound` (new tenure), not for extensions or continuation blocks (`miner.rs:466-473`):

```rust
let is_new_tenure = cause.is_new_tenure();
let coinbase_height = if is_new_tenure {
    parent_coinbase_height.checked_add(1).expect("Blockchain overflow")
} else {
    parent_coinbase_height
};
```

### Step 3: Begin the Tenure

```rust
// miner.rs:687
let mut tenure_tx = builder.tenure_begin(burn_dbconn, &mut miner_tenure_info)?;
```

`tenure_begin` (`miner.rs:501-557`) calls `NakamotoChainState::setup_block`, which:
1. Processes any epoch transition (if this block crosses an epoch boundary).
2. Handles Stacks-on-Bitcoin operations (stack-stx, transfer-stx, etc.).
3. Processes PoX auto-unlocks.
4. Calculates matured miner rewards.
5. Optionally computes the signer set (if in prepare phase).
6. Returns an open `ClarityTx` for mining.

### Step 4: Compute Soft Limit

```rust
// miner.rs:693-718
if let Some(percentage) = settings.mempool_settings.tenure_cost_limit_per_block_percentage {
    // Calculate: soft_limit = cost_so_far + (remaining * percentage / 100)
    let mut remaining_limit = tenure_budget.clone();
    remaining_limit.sub(&cost_so_far);
    remaining_limit.divide(100);
    remaining_limit.multiply(percentage.into());
    remaining_limit.add(&cost_so_far);
    soft_limit = Some(remaining_limit);
}
```

This prevents a single block from consuming the entire tenure budget. For example, with `tenure_cost_limit_per_block_percentage = 25`, each block can use at most 25% of the remaining budget.

### Step 5: Select and Apply Transactions

```rust
// miner.rs:731-747
let (blocked, tx_events) = StacksBlockBuilder::select_and_apply_transactions(
    &mut tenure_tx,
    &mut builder,
    mempool,
    parent_stacks_header.stacks_block_height,
    &initial_txs,        // tenure change + coinbase (if any)
    settings,
    event_observer,
    replay_transactions,
)?;
```

This reuses the epoch 2 transaction selection logic from `BlockBuilder`. The `initial_txs` always go first (tenure change, then coinbase), followed by mempool transactions ordered by fee rate. If `blocked` is true, mining was aborted (e.g., by a shutdown signal).

If no transactions were selected, mining fails with `Error::NoTransactionsToMine`.

### Step 6: Finalize the Block

```rust
// miner.rs:763-765
let block = builder.mine_nakamoto_block(&mut tenure_tx, burn_chain_height);
let tenure_size = builder.bytes_so_far;
let tenure_consumed = builder.tenure_finish(tenure_tx)?;
```

`mine_nakamoto_block` calls `finalize_block` (`miner.rs:579-597`), which:
1. Computes the Merkle tree root of all transaction IDs.
2. Seals the Clarity transaction to get the `state_index_root`.
3. Assembles the final `NakamotoBlock`.

`tenure_finish` (`miner.rs:561-574`) commits the MARF trie under a sentinel block hash (`MINER_BLOCK_CONSENSUS_HASH` / `MINER_BLOCK_HEADER_HASH`), returning the total execution cost consumed.

## Code Walkthrough: Tenure-Start Block Structure

A tenure-start block has a specific transaction ordering:

1. **TenureChange transaction** (mandatory) -- `cause: BlockFound`
2. **Coinbase transaction** (mandatory) -- includes VRF proof
3. **Regular transactions** -- from mempool

A tenure-continuation block (not the first in a tenure) has:

1. **Regular transactions** -- from mempool only

A tenure-extension block has:

1. **TenureChange transaction** (mandatory) -- `cause: Extended*`
2. **Regular transactions** -- from mempool

## How Blocks Get Proposed to Signers

After the miner builds an unsigned block:

1. The miner signs the block header with their private key (`NakamotoBlockHeader::sign_miner`).
2. The signed block is proposed to the signer set via StackerDB (the `.miners` contract).
3. Each signer validates the block independently:
   - Checks that the block builds on the canonical tip.
   - Verifies the miner's signature.
   - Validates the tenure change (if present).
   - Executes all transactions to verify the state root.
   - Checks the timestamp is reasonable (greater than parent, at most 15 seconds in the future).
4. If the signer approves, it creates a signature over the signer signature hash and sends it back.
5. Once the miner collects signatures representing at least 70% of the total signing weight, it assembles the `signer_signature` vector and broadcasts the complete block.

The `.miners` StackerDB has space for two miners (the current and previous tenure winners), tracked by `MinersDBInformation` (`mod.rs:555-583`):

```rust
pub struct MinersDBInformation {
    signer_0_sortition: ConsensusHash,
    signer_1_sortition: ConsensusHash,
    latest_winner: u16,
}
```

## Ephemeral Blocks

The builder supports **ephemeral** blocks via `load_ephemeral_tenure_info` (`miner.rs:336-343`). Ephemeral blocks are used for cost estimation and block simulation without modifying chainstate. They call `NakamotoChainState::setup_ephemeral_block` instead of `setup_block`, which uses concurrent (non-exclusive) database transactions.

## Shadow Block Mining

Shadow blocks (`stackslib/src/chainstate/nakamoto/shadow.rs`) have their own build path. They bypass normal mining constraints:

- No block-commit is required (shadow blocks aren't produced by sortition winners).
- No VRF proof verification (the "miner" is the SIP process, not a real miner).
- No miner signature verification.
- Coinbase recipients must be the burn address (no STX rewards).
- Full signer weight is granted without actual signatures.

Shadow blocks use `inner_load_tenure_info` with `shadow_block = true`, which skips block-commit lookup (`miner.rs:370-388`).

## How It Connects

- **Chapter 18 (Nakamoto Consensus)**: Defines the `NakamotoBlockHeader` and `TenureChangePayload` structures that the builder fills in.
- **Chapter 20 (Coordinator)**: Processes the blocks produced by miners, calling `process_next_nakamoto_block`.
- **Chapter 14 (Transactions)**: The `select_and_apply_transactions` function applies transaction validation rules.
- **Chapter 13 (MARF)**: `tenure_finish` commits the state trie, and the `state_index_root` in the header is the MARF root.
- **Chapter 25 (StackerDB)**: Block proposals are published to the `.miners` StackerDB for signer validation, and signer responses are read back from `.signers-*-*` StackerDB contracts.
- **Chapter 27 (Signer Architecture)**: The signer message types (`BlockProposal`, `BlockResponse`) that the miner exchanges with signers via StackerDB.

## Edge Cases and Gotchas

1. **Building atop epoch 2 requires special handling**: The first Nakamoto block's parent is an epoch 2 block. The builder enforces that both a tenure change and coinbase must be present (`miner.rs:279-290`).

2. **Empty blocks are rejected**: If `select_and_apply_transactions` produces no transactions, `build_nakamoto_block` returns `Error::NoTransactionsToMine` (`miner.rs:757-760`). This prevents miners from flooding the network with empty blocks.

3. **Soft limit prevents monopolization**: Without a soft limit, the first block in a tenure could consume the entire execution budget, leaving no room for subsequent blocks. The `tenure_cost_limit_per_block_percentage` setting prevents this.

4. **Coinbase height only increments on `BlockFound`**: Extensions and continuation blocks reuse the parent's coinbase height. This is critical for miner reward maturity calculations.

5. **Timestamp monotonicity**: The block timestamp must be `max(parent_timestamp, current_time)`, enforced in `from_parent_empty` (`mod.rs:965`). Signers additionally require the timestamp to be at most 15 seconds in the future.

6. **`max_tenure_bytes` limits total tenure size**: Beyond the execution cost budget, there is a byte-size limit on the entire tenure. This is tracked in `bytes_so_far` and the `total_tenure_size` column in the block headers table.

7. **Reward set loading uses the block's consensus hash**: The builder resolves the reward set from the sortition that elected it, not the canonical tip. This prevents a miner from using a different reward set than the one that authorized their tenure (`miner.rs:430-438`).
