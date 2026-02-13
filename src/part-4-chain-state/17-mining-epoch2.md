# Chapter 17: Mining in Epoch 2.x

## Purpose

Mining in Stacks Epoch 2.x is the process by which miners assemble blocks, select transactions from the mempool, and produce both anchored blocks and microblock streams. This chapter covers `StacksBlockBuilder`, the central struct for block assembly, the transaction selection algorithm, microblock production, VRF-based miner selection, and the complete mining lifecycle.

The implementation lives primarily in `stackslib/src/chainstate/stacks/miner.rs`.

## Key Concepts

### The Mining Lifecycle

A miner's lifecycle in Epoch 2.x follows these steps:

1. **Register a VRF key**: The miner submits a `LeaderKeyRegisterOp` on Bitcoin, committing their VRF public key.
2. **Win a sortition**: The miner submits a `LeaderBlockCommitOp` on Bitcoin, burning BTC. The sortition algorithm selects a winner based on BTC spent, using the previous sortition's VRF seed.
3. **Produce an anchored block**: The winner assembles a `StacksBlock` by selecting transactions from the mempool, applying them against the chain state, and computing the block header.
4. **Stream microblocks**: Between anchored blocks, the miner produces `StacksMicroblock` instances, providing faster transaction confirmation.
5. **Repeat**: When the next Bitcoin block arrives, a new sortition occurs and a new miner takes over.

### VRF and Miner Selection

The Verifiable Random Function (VRF) is used for fair miner selection. Each miner's block-commit includes a VRF proof over the previous sortition's seed. The sortition algorithm:

1. Takes the set of all block-commits in a Bitcoin block
2. Computes a weighted lottery based on BTC burned
3. Selects the winner using the VRF output as the random seed
4. The winner's VRF proof goes into the `StacksBlockHeader.proof` field

The VRF proof serves dual purposes:
- It proves the miner was selected (verifiable by anyone)
- It provides the random seed for the *next* sortition

## The StacksBlockBuilder

The builder is the workhorse of block assembly:

```rust
// stackslib/src/chainstate/stacks/mod.rs:1193-1211
pub struct StacksBlockBuilder {
    pub chain_tip: StacksHeaderInfo,
    pub header: StacksBlockHeader,
    pub txs: Vec<StacksTransaction>,
    pub micro_txs: Vec<StacksTransaction>,
    pub total_anchored_fees: u64,
    pub total_confirmed_streamed_fees: u64,
    pub total_streamed_fees: u64,
    anchored_done: bool,
    bytes_so_far: u64,
    prev_microblock_header: StacksMicroblockHeader,
    miner_privkey: StacksPrivateKey,
    miner_payouts: Option<(MinerReward, Vec<MinerReward>, MinerReward, MinerRewardInfo)>,
    parent_consensus_hash: ConsensusHash,
    parent_header_hash: BlockHeaderHash,
    parent_microblock_hash: Option<BlockHeaderHash>,
    miner_id: usize,
}
```

Key fields:
- **`chain_tip`**: The parent block we are building on top of
- **`header`**: The block header being constructed (updated as transactions are added)
- **`txs`**: Accumulated transactions for the anchored block
- **`micro_txs`**: Transactions for the current microblock being assembled
- **`anchored_done`**: Flag indicating the anchored block is sealed; subsequent transactions go to microblocks
- **`bytes_so_far`**: Running total of serialized transaction bytes
- **`miner_privkey`**: Private key for signing microblocks

## Block Assembly Pipeline

### Step 1: Create the Builder

`make_block_builder` (miner.rs:2096-2147) initializes the builder from the parent chain tip:

```rust
pub fn make_block_builder(
    burnchain: &Burnchain,
    mainnet: bool,
    stacks_parent_header: &StacksHeaderInfo,
    proof: &VRFProof,
    total_burn: u64,
    pubkey_hash: &Hash160,
) -> Result<StacksBlockBuilder, Error>
```

For the genesis block (parent is `FIRST_BURNCHAIN_CONSENSUS_HASH`), the builder uses special bootstrap parameters. For all subsequent blocks, the builder computes the new work score:

```rust
// miner.rs:2129-2135
let new_work = StacksWorkScore {
    burn: total_burn,
    work: stacks_parent_header
        .stacks_block_height
        .checked_add(1)
        .expect("FATAL: block height overflow"),
};
```

### Step 2: Pre-Epoch Begin

`pre_epoch_begin` (miner.rs:1856) loads the parent microblock stream and prepares the execution environment:

```rust
pub fn pre_epoch_begin<'a>(
    &mut self,
    chainstate: &'a mut StacksChainState,
    burn_dbconn: &'a SortitionHandleConn,
    confirm_microblocks: bool,
) -> Result<MinerEpochInfo<'a>, Error>
```

This returns a `MinerEpochInfo` containing:

```rust
// miner.rs:304-313
pub struct MinerEpochInfo<'a> {
    pub chainstate_tx: ChainstateTx<'a>,
    pub clarity_instance: &'a mut ClarityInstance,
    pub burn_tip: BurnchainHeaderHash,
    pub burn_tip_height: u32,
    pub parent_microblocks: Vec<StacksMicroblock>,
    pub mainnet: bool,
}
```

### Step 3: Epoch Begin

`epoch_begin` opens the Clarity block transaction and processes parent microblocks:

```rust
pub fn epoch_begin<'a, 'b>(
    &mut self,
    burn_dbconn: &'a SortitionHandleConn,
    info: &'a mut MinerEpochInfo<'b>,
) -> Result<(ClarityTx<'a, 'b>, ExecutionCost), Error>
```

This calls `StacksChainState::setup_block` (see Chapter 15), which:
- Finds matured miner rewards
- Loads burnchain operations
- Opens a Clarity transaction
- Processes parent microblocks

The returned `ClarityTx` is the execution context for the new block's transactions.

### Step 4: Select and Apply Transactions

The `select_and_apply_transactions` method (miner.rs:2196-2275) drives transaction selection:

```rust
pub fn select_and_apply_transactions<B: BlockBuilder>(
    epoch_tx: &mut ClarityTx,
    builder: &mut B,
    mempool: &mut MemPoolDB,
    tip_height: u64,
    initial_txs: &[StacksTransaction],
    settings: BlockBuilderSettings,
    event_observer: Option<&dyn MemPoolEventDispatcher>,
    replay_transactions: &[StacksTransaction],
) -> Result<(bool, Vec<TransactionEvent>), Error>
```

The flow:
1. **Apply initial transactions**: The coinbase (and in Nakamoto, the tenure change) is applied first
2. **Walk the mempool**: `select_and_apply_transactions_from_mempool` iterates through the mempool in fee-rate order, applying each transaction
3. **Handle replay transactions**: If replay transactions are provided (for reprocessing), use those instead of the mempool

### Step 5: Transaction Evaluation via try_mine_tx_with_len

Each candidate transaction passes through `try_mine_tx_with_len` (miner.rs:2417-2540):

```rust
fn try_mine_tx_with_len(
    &mut self,
    clarity_tx: &mut ClarityTx,
    tx: &StacksTransaction,
    tx_len: u64,
    limit_behavior: &BlockLimitFunction,
    _max_execution_time: Option<std::time::Duration>,
    _total_receipt_size: &mut u64,
) -> TransactionResult
```

This method:
1. **Checks size budget**: If `bytes_so_far + tx_len >= MAX_EPOCH_SIZE`, skip the transaction
2. **Applies limit behavior**: Once the block hits a cost threshold, only boot-code contract calls and token transfers are allowed (no user contract calls or deployments)
3. **Validates anchor mode**: Transactions with `OffChainOnly` anchor mode are rejected for anchored blocks
4. **Checks for problematic transactions**: Known DDoS vectors are preemptively filtered
5. **Executes the transaction**: Calls into the Clarity VM to run the transaction
6. **Returns the result**: `TransactionResult::Success`, `ProcessingError`, `Skipped`, or `Problematic`

### BlockLimitFunction

The block limit function tracks fill state:

```rust
// miner.rs:292-302
pub enum BlockLimitFunction {
    NO_LIMIT_HIT,        // Block is not yet full
    CONTRACT_LIMIT_HIT,  // Soft limit reached, no more contract calls
    LIMIT_REACHED,       // Hard limit, no more transactions at all
}
```

When `CONTRACT_LIMIT_HIT` is active, the miner still accepts token transfers and boot-code calls (like PoX operations) but rejects user smart contracts. This ensures that critical protocol operations can still fit even in a nearly-full block.

### Transaction Results

Every transaction evaluation produces a `TransactionResult`:

```rust
// miner.rs:417-429
pub enum TransactionResult {
    Success(TransactionSuccess),
    ProcessingError(TransactionError),
    Skipped(TransactionSkipped),
    Problematic(TransactionProblematic),
}
```

- **Success**: Transaction executed and should be included in the block
- **ProcessingError**: Transaction failed; it is dropped from the mempool
- **Skipped**: Transaction could not be included now but may succeed later (e.g., block too full)
- **Problematic**: Transaction is a known DDoS vector and should be permanently dropped

### Step 6: Mine the Anchored Block

`mine_anchored_block` (miner.rs:1733-1743) finalizes the anchored block:

```rust
pub fn mine_anchored_block(&mut self, clarity_tx: &mut ClarityTx) -> StacksBlock {
    assert!(!self.anchored_done);
    StacksChainState::finish_block(
        clarity_tx,
        self.miner_payouts.as_ref(),
        u32::try_from(self.header.total_work.work).expect("FATAL: more than 2^32 blocks"),
        &self.header.microblock_pubkey_hash,
    )
    .expect("FATAL: call to `finish_block` failed");
    self.finalize_block(clarity_tx)
}
```

`finalize_block` computes the transaction Merkle root, sets the `state_index_root` from the MARF, and assembles the final `StacksBlock`.

### Step 7: Epoch Finish

`epoch_finish` commits the Clarity transaction and returns the total execution cost:

```rust
pub fn epoch_finish(self, tx: ClarityTx) -> Result<ExecutionCost, Error>
```

## The build_anchored_block Entry Point

The top-level mining function is `build_anchored_block` (miner.rs:2282-2411):

```rust
pub fn build_anchored_block(
    chainstate_handle: &StacksChainState,
    burn_dbconn: &SortitionHandleConn,
    mempool: &mut MemPoolDB,
    parent_stacks_header: &StacksHeaderInfo,
    total_burn: u64,
    proof: &VRFProof,
    pubkey_hash: &Hash160,
    coinbase_tx: &StacksTransaction,
    settings: BlockBuilderSettings,
    event_observer: Option<&dyn MemPoolEventDispatcher>,
    burnchain: &Burnchain,
) -> Result<(StacksBlock, ExecutionCost, u64), Error>
```

This orchestrates the entire pipeline:
1. Validates that `coinbase_tx` is actually a coinbase
2. Creates a `StacksBlockBuilder` via `make_block_builder`
3. Calls `pre_epoch_begin` -> `epoch_begin` -> `select_and_apply_transactions` -> `mine_anchored_block` -> `epoch_finish`
4. Reports metrics (block size, execution cost, tx count, assembly time)
5. Returns the assembled block, total execution cost consumed, and block byte size

## BlockBuilderSettings

Mining policy is configured through:

```rust
// miner.rs:238-247
pub struct BlockBuilderSettings {
    pub max_miner_time_ms: u64,
    pub mempool_settings: MemPoolWalkSettings,
    pub miner_status: Arc<Mutex<MinerStatus>>,
    pub confirm_microblocks: bool,
    pub max_execution_time: Option<std::time::Duration>,
    pub max_tenure_bytes: u64,
}
```

- **`max_miner_time_ms`**: Wall-clock time limit for block assembly
- **`mempool_settings`**: Controls mempool iteration (fee thresholds, ordering)
- **`miner_status`**: Shared flag for signaling the miner to stop (e.g., when a new burnchain block arrives)
- **`confirm_microblocks`**: Whether to include parent microblocks in this block
- **`max_tenure_bytes`**: Maximum total bytes across all blocks in a tenure

## Microblock Production

After producing the anchored block, the miner streams microblocks until the next sortition:

```rust
// miner.rs:1746-1794
pub fn mine_next_microblock<'a>(&mut self) -> Result<StacksMicroblock, Error>
```

Microblock production:
1. Compute the Merkle root of transaction IDs in `self.micro_txs`
2. Create the microblock header (linked to the previous via `prev_block`)
3. Sign the header with `self.miner_privkey`
4. Verify the signature against the anchored block's `microblock_pubkey_hash`
5. Clear `micro_txs` for the next microblock

Microblocks form a linked list. The first microblock's `prev_block` is the anchored block's hash. Each subsequent microblock's `prev_block` is the previous microblock's hash. The `sequence` field increments monotonically.

## Miner Status and Abort Handling

The `MinerStatus` struct provides coordinated mining control:

```rust
// miner.rs:147-151
pub struct MinerStatus {
    blockers: HashSet<ThreadId>,
    spend_amount: u64,
}
```

Other threads can block the miner via `signal_mining_blocked`:

```rust
// miner.rs:194-204
pub fn signal_mining_blocked(miner_status: Arc<Mutex<MinerStatus>>) {
    match miner_status.lock() {
        Ok(mut status) => { status.add_blocked(); }
        Err(_e) => { panic!("FATAL: mutex poisoned"); }
    }
}
```

When `is_blocked()` returns true during transaction selection, the mining loop exits and returns `Error::MinerAborted`.

## AssembledAnchorBlock

The final output of mining includes metadata about the burnchain context:

```rust
// miner.rs:121-138
pub struct AssembledAnchorBlock {
    pub parent_consensus_hash: ConsensusHash,
    pub consensus_hash: ConsensusHash,
    pub burn_hash: BurnchainHeaderHash,
    pub burn_block_height: u64,
    pub orig_burn_hash: BurnchainHeaderHash,
    pub anchored_block: StacksBlock,
    pub attempt: u64,
    pub tenure_begin: u128,
}
```

The `orig_burn_hash` may differ from `burn_hash` if the burnchain tip advanced during mining. The `attempt` counter tracks how many blocks were attempted for this sortition (multiple attempts are common since mining restarts when new information arrives).

## How It Connects

- **Sortition DB** (Chapter 12): Mining is triggered when the local node wins a sortition. The VRF proof and burn amount come from the sortition.
- **Mempool** (Chapter 26): `select_and_apply_transactions` walks the mempool to find fee-paying transactions.
- **Block Processing** (Chapter 15): The assembled block is validated by the same `append_block` pipeline used for received blocks.
- **Transactions** (Chapter 14): The miner evaluates and applies `StacksTransaction` instances via the Clarity VM.
- **Stacking/PoX** (Chapter 16): Miners must send BTC to PoX reward addresses as part of their block-commit.
- **Nakamoto Mining** (Chapter 19): In Nakamoto, `StacksBlockBuilder` is replaced by `NakamotoBlockBuilder` with different tenure semantics.

## Edge Cases and Gotchas

1. **Coinbase must be first**: The `build_anchored_block` method validates that the provided `coinbase_tx` has a `TransactionPayload::Coinbase` payload. The coinbase is always the first transaction applied.

2. **Miner can be aborted mid-block**: If `MinerStatus.is_blocked()` becomes true during transaction selection, the miner aborts cleanly. The half-built block is discarded and the MARF transaction is rolled back.

3. **Version pinning**: The block header version is set to at least `STACKS_BLOCK_VERSION_AST_PRECHECK_SIZE` (1) after `pre_epoch_begin`. This version field affects how AST size limits are enforced.

4. **Microblock key reuse prevention**: Each anchored block commits to a new `microblock_pubkey_hash`. This hash must be unique on the current fork. If a miner reuses a key, their block will be rejected during validation.

5. **Empty blocks in Nakamoto tenure changes**: When `select_and_apply_transactions` detects a `TenureChange(BlockFound)` as the initial transaction, it immediately returns -- producing an empty block (except for the tenure change + coinbase). This is an intentional heuristic to start tenures quickly.

6. **MAX_EPOCH_SIZE vs MAX_BLOCK_LEN**: `MAX_EPOCH_SIZE` (2 MB) caps the total bytes across the anchored block and all its microblocks. `MAX_BLOCK_LEN` (also 2 MB) caps a single block. In practice, the anchored block is limited by `MAX_EPOCH_SIZE` since it includes the cumulative size of confirmed microblocks.
