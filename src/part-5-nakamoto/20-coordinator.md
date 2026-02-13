# Chapter 20: ChainsCoordinator

## Purpose

The `ChainsCoordinator` is the central orchestrator that drives both burnchain (Bitcoin) and Stacks chain processing. It sits in a loop, responding to events -- new Bitcoin blocks and new Stacks blocks -- and sequences them so that sortitions are evaluated, reward cycles are computed, and Stacks blocks are appended to the chain in the correct order. In Nakamoto, the coordinator has dual code paths: one for epoch 2.x blocks and one for Nakamoto blocks, with special handling for the transition between them.

## Key Concepts

### Event-Driven Loop

The coordinator runs in its own thread and responds to three events:

1. **NEW_BURN_BLOCK** -- A new Bitcoin block has been downloaded and indexed.
2. **NEW_STACKS_BLOCK** -- A new Stacks block has been received and staged.
3. **STOP** -- Graceful shutdown signal.

These events arrive via the `CoordinatorReceivers` communication channel. The coordinator processes one event at a time, blocking mining while doing so (via `signal_mining_blocked` / `signal_mining_ready`).

### Three Operating Modes

The main loop in `ChainsCoordinator::run` (`stackslib/src/chainstate/coordinator/mod.rs:534-556`) dispatches to different handlers depending on the current reward cycle:

```rust
loop {
    let bits = comms.wait_on();
    if inst.in_subsequent_nakamoto_reward_cycle() {
        // Pure Nakamoto: only handle Nakamoto events
        if !inst.handle_comms_nakamoto(bits, miner_status.clone()) { return; }
    } else if inst.in_first_nakamoto_reward_cycle() {
        // Transition: handle both Nakamoto and epoch 2 events
        if !inst.handle_comms_nakamoto(bits, miner_status.clone()) { return; }
        if !inst.handle_comms_epoch2(bits, miner_status.clone()) { return; }
    } else {
        // Pre-Nakamoto: only handle epoch 2 events
        if !inst.handle_comms_epoch2(bits, miner_status.clone()) { return; }
    }
}
```

During the first Nakamoto reward cycle, both handlers run to ensure epoch 2 blocks are still processed while Nakamoto blocks start arriving.

### Reward Cycle Awareness

The coordinator determines which reward cycle it's in using:

- `get_first_nakamoto_reward_cycle()` -- The first reward cycle at or after epoch 3.0's start height.
- `get_current_reward_cycle()` -- The reward cycle of the canonical sortition tip.
- `in_first_nakamoto_reward_cycle()` -- Returns true when `current == first`.
- `in_subsequent_nakamoto_reward_cycle()` -- Returns true when `current > first`.

These are defined at `stackslib/src/chainstate/nakamoto/coordinator/mod.rs:617-663`.

## Architecture

```
stackslib/src/chainstate/coordinator/
    mod.rs       -- ChainsCoordinator struct, epoch 2 handlers, RewardSetProvider
    comm.rs      -- CoordinatorCommunication, events, channels

stackslib/src/chainstate/nakamoto/coordinator/
    mod.rs       -- Nakamoto-specific handlers, reward cycle info
```

The coordinator owns all the database handles:

```rust
// stackslib/src/chainstate/coordinator/mod.rs:200-228
pub struct ChainsCoordinator<'a, T, N, R, CE, FE, B> {
    pub canonical_sortition_tip: Option<SortitionId>,
    pub burnchain_blocks_db: BurnchainDB,
    pub chain_state_db: StacksChainState,
    pub sortition_db: SortitionDB,
    pub burnchain: Burnchain,
    pub atlas_db: Option<AtlasDB>,
    pub dispatcher: Option<&'a T>,
    pub cost_estimator: Option<&'a mut CE>,
    pub fee_estimator: Option<&'a mut FE>,
    pub reward_set_provider: R,
    pub notifier: N,
    pub atlas_config: AtlasConfig,
    pub config: ChainsCoordinatorConfig,
    burnchain_indexer: B,
    pub refresh_stacker_db: Arc<AtomicBool>,
    pub in_nakamoto_epoch: bool,
}
```

Generic parameters:
- `T: BlockEventDispatcher` -- announces processed blocks to event listeners.
- `N: CoordinatorNotices` -- notifies other threads about processed blocks/sortitions.
- `R: RewardSetProvider` -- computes PoX reward sets.
- `CE: CostEstimator` -- updates cost estimation after each block.
- `FE: FeeEstimator` -- updates fee estimation after each block.
- `B: BurnchainHeaderReader` -- reads burnchain headers.

## Key Data Structures

### RewardCycleInfo

Describes the PoX anchor block status for a reward cycle (`coordinator/mod.rs:105-144`):

```rust
pub struct RewardCycleInfo {
    pub reward_cycle: u64,
    pub anchor_status: PoxAnchorBlockStatus,
}

pub enum PoxAnchorBlockStatus {
    SelectedAndKnown(BlockHeaderHash, Txid, RewardSet),
    SelectedAndUnknown(BlockHeaderHash, Txid),
    NotSelected,
}
```

In Nakamoto, the reward set must always be known (`SelectedAndKnown`); if it's unknown, sortition processing halts until the PoX anchor block is available.

### NewBurnchainBlockStatus

The result of processing a burnchain block (`coordinator/mod.rs:82-103`):

```rust
pub enum NewBurnchainBlockStatus {
    Ready,                          // can proceed with Stacks blocks
    WaitForPox2x(BlockHeaderHash),  // missing 2.x PoX anchor block
    WaitForPoxNakamoto,             // missing Nakamoto anchor block
}
```

### OnChainRewardSetProvider

The production implementation of `RewardSetProvider` (`coordinator/mod.rs:291-365`):

```rust
pub struct OnChainRewardSetProvider<'a, T: BlockEventDispatcher>(pub Option<&'a T>);
```

It has two code paths for computing reward sets:
- **`get_reward_set_epoch2`** reads directly from the PoX contract.
- **`get_reward_set_nakamoto`** reads from the `.signers` contract, which stores pre-computed reward sets.

The key routing logic at `coordinator/mod.rs:299-354`:

```rust
fn get_reward_set(&self, ...) -> Result<RewardSet, Error> {
    // Determine if this reward cycle will be active in Nakamoto
    let is_nakamoto_reward_set = /* check if cycle >= first_nakamoto_cycle */;

    // Always compute via epoch2 method (reads from .pox-4)
    let reward_set = self.get_reward_set_epoch2(...)?;

    // If the reward set will be used in Nakamoto, verify signers exist
    if is_nakamoto_reward_set
        && (reward_set.signers.is_none() || reward_set.signers == Some(vec![]))
    {
        return Err(Error::PoXAnchorBlockRequired);
    }

    Ok(reward_set)
}
```

## Code Walkthrough: Processing a New Burnchain Block

### Entry Point: handle_new_burnchain_block

This is the unified entry point (`coordinator/mod.rs:1058-1104`):

```rust
pub fn handle_new_burnchain_block(&mut self) -> Result<NewBurnchainBlockStatus, Error> {
    let canonical_burnchain_tip = self.burnchain_blocks_db.get_canonical_chain_tip()?;
    let target_epoch = epochs.epoch_at_height(canonical_burnchain_tip.block_height);

    if target_epoch.epoch_id < StacksEpochId::Epoch30 {
        // Pre-Nakamoto: use epoch 2 handler
        return self.handle_new_epoch2_burnchain_block(&mut HashSet::new());
    }

    // Catch up sortition DB if needed
    let canonical_snapshot = /* ... */;
    if canonical_snapshot_epoch < Epoch30 {
        self.handle_new_epoch2_burnchain_block(&mut HashSet::new())?;
    }

    // Process Nakamoto sortitions
    self.handle_new_nakamoto_burnchain_block()
}
```

### Nakamoto Burnchain Block Processing

`handle_new_nakamoto_burnchain_block` at `nakamoto/coordinator/mod.rs:1032`:

1. **Find unprocessed sortitions**: Walk backwards from the canonical burnchain tip to find all burn blocks without a sortition.

2. **For each unprocessed block**:
   - If this is a reward cycle start, compute the reward cycle info. In Nakamoto, this requires the PoX anchor block to be available. If it isn't, processing halts (`return Ok(false)`).
   - Evaluate the sortition via `SortitionDB::evaluate_sortition`.
   - Mark the burn block as processed in the staging DB.
   - Update the canonical sortition tip.

3. **Reward set must be known**: Unlike epoch 2, Nakamoto cannot proceed with an unknown anchor block. The key comment at `nakamoto/coordinator/mod.rs:1094-1101` explains why:

```
// NOTE(safety): the reason it's safe to use the local best stacks tip here is
// because as long as at least 30% of the signers are honest, there's no way there
// can be two or more distinct reward sets calculated for a reward cycle.
```

## Code Walkthrough: Processing New Nakamoto Stacks Blocks

`handle_new_nakamoto_stacks_block` at `nakamoto/coordinator/mod.rs:773`:

The method processes blocks in a loop, one at a time:

```rust
loop {
    let mut processed_block_receipt = match NakamotoChainState::process_next_nakamoto_block(
        &mut self.chain_state_db,
        &mut self.sortition_db,
        &canonical_sortition_tip,
        self.dispatcher,
        self.config.txindex,
    ) { ... };

    let Some(block_receipt) = processed_block_receipt.take() else {
        break;  // out of blocks
    };

    // Notify about processed block
    self.notifier.notify_stacks_block_processed();

    // Update cost & fee estimators
    // Process Atlas attachment events
    // Check if we're in the prepare phase for reward set processing
}
```

After processing each block, the coordinator checks whether the block is in the prepare phase. If so, it determines whether the next reward cycle's reward set is available, potentially unblocking sortition processing.

### The Epoch 2 to Nakamoto Transition

The `handle_comms_nakamoto` method at `nakamoto/coordinator/mod.rs:669-751` handles the transition:

```rust
if !self.in_nakamoto_epoch {
    // Check if canonical tip is now a Nakamoto block
    if let Ok(Some(canonical_header)) = NakamotoChainState::get_canonical_block_header(...) {
        if canonical_header.is_nakamoto_block() {
            self.in_nakamoto_epoch = true;
        } else {
            // Still epoch 2 -- process epoch 2 blocks
            self.handle_new_stacks_block()?;
        }
    }
}
// Then process Nakamoto blocks
self.handle_new_nakamoto_stacks_block()?;
```

This ensures a smooth transition: during the first Nakamoto reward cycle, epoch 2 blocks are still processed alongside Nakamoto blocks.

## Code Walkthrough: Reward Set Calculation in Nakamoto

In Nakamoto, reward sets are calculated differently from epoch 2:

### Epoch 2 Path

The PoX anchor block is selected during the prepare phase, and the reward set is computed from the PoX contract at that anchor block. The reward set can be unknown if the anchor block hasn't been downloaded yet.

### Nakamoto Path

The `.signers` contract stores pre-computed reward sets. The reward set is calculated during the **first block of the prepare phase** via `NakamotoSigners::check_and_handle_prepare_phase_start` (`signer_set.rs:349`):

1. Check if we're in the prepare phase (`pox_constants.is_in_prepare_phase`).
2. Check if the `.signers` contract needs updating.
3. Read reward slots from `pox-4.get-reward-set-pox-address`.
4. Compute the signer set with weights.
5. Write the result to `.signers` via `stackerdb-set-signer-slots` and `set-signers`.
6. Store the reward set in the `nakamoto_reward_sets` table.

The `OnChainRewardSetProvider::read_reward_set_nakamoto` (`nakamoto/coordinator/mod.rs:83-142`) reads this stored data:

1. Look up the `cycle-set-height` from `.signers` to find which coinbase height calculated the reward set.
2. Find the block at that coinbase height.
3. Load the reward set from `nakamoto_reward_sets` for that block.
4. Verify the reward set has signers (fatal error if not).

### Assumed Total Commitment (ATC)

ATC is the mechanism that determines miner block-commit validity in Nakamoto. The constant `MINING_COMMITMENT_FREQUENCY_NAKAMOTO` at `stacks-common/src/types/mod.rs:102` requires miners to commit in at least 3 out of every 6 sortitions:

```rust
pub const MINING_COMMITMENT_WINDOW: u8 = 6;
pub const MINING_COMMITMENT_FREQUENCY_NAKAMOTO: u8 = 3;
```

A miner that doesn't commit frequently enough is excluded from sortition. This prevents flash-mining attacks where a miner appears only for a single profitable sortition.

## Code Walkthrough: The Epoch 2 Stacks Block Handler

For completeness, `handle_new_stacks_block` (`coordinator/mod.rs:960-967`) is much simpler:

```rust
pub fn handle_new_stacks_block(&mut self) -> Result<Option<BlockHeaderHash>, Error> {
    if let Some(pox_anchor) = self.process_ready_blocks()? {
        self.process_new_pox_anchor(pox_anchor, &mut HashSet::new())
    } else {
        Ok(None)
    }
}
```

It processes ready blocks and, if a new PoX anchor block is found, reprocesses the reward cycle.

## The BlockEventDispatcher Trait

The coordinator announces events via `BlockEventDispatcher` (`coordinator/mod.rs:146-182`):

```rust
pub trait BlockEventDispatcher {
    fn announce_block(
        &self,
        block: &StacksBlockEventData,
        metadata: &StacksHeaderInfo,
        receipts: &[StacksTransactionReceipt],
        parent: &StacksBlockId,
        winner_txid: &Txid,
        matured_rewards: &[MinerReward],
        // ... many more parameters
    );

    fn announce_burn_block(
        &self,
        burn_block: &BurnchainHeaderHash,
        burn_block_height: u64,
        rewards: Vec<(PoxAddress, u64)>,
        burns: u64,
        // ...
    );
}
```

This is how external systems (like the stacks-node event observer, the API indexer, etc.) learn about new blocks and burnchain events.

## How It Connects

- **Chapter 10 (Bitcoin Integration)**: The burnchain block processing reads Bitcoin blocks from `BurnchainDB`.
- **Chapter 12 (Sortition DB)**: `evaluate_sortition` is the core SortitionDB operation the coordinator triggers.
- **Chapter 18 (Nakamoto Consensus)**: The coordinator validates and processes Nakamoto blocks using the consensus rules defined there.
- **Chapter 16 (Stacking and PoX)**: Reward set computation ties directly into PoX cycles.
- **Chapter 21 (Epoch Transitions)**: The coordinator's epoch-aware dispatching is what triggers epoch transitions.

## Edge Cases and Gotchas

1. **Blocking on the anchor block**: In Nakamoto, if the PoX anchor block for the next reward cycle is not yet available, the coordinator refuses to process more burnchain blocks. This is because every Nakamoto reward cycle *must* have a known reward set (unlike epoch 2 where it could default to burn).

2. **The `in_nakamoto_epoch` flag**: The coordinator uses a sticky boolean to track whether it has seen a Nakamoto block as the canonical tip. Once set, it stops trying to process epoch 2 blocks for `NEW_STACKS_BLOCK` events. This is a one-way transition.

3. **Mining is blocked during processing**: The coordinator calls `signal_mining_blocked` before processing and `signal_mining_ready` after. This prevents the miner from building on stale state. Mining can only proceed when the coordinator is idle.

4. **Invalid blocks don't stop the loop**: If `process_next_nakamoto_block` returns `ChainstateError::InvalidStacksBlock`, the coordinator logs a warning and continues to the next block. This is important for resilience against malicious block proposals.

5. **The `refresh_stacker_db` flag**: When a processed block updates the signer set (`block_receipt.signers_updated`), the coordinator sets this `AtomicBool` to true. The P2P thread checks this flag to know when to reload StackerDB configuration.

6. **First Nakamoto cycle runs both handlers**: During `in_first_nakamoto_reward_cycle()`, both `handle_comms_nakamoto` and `handle_comms_epoch2` run for every event. This ensures epoch 2 blocks that arrive during the transition are still processed.

7. **Cost and fee estimators are updated per-block**: After each processed block, the coordinator notifies the cost estimator and fee estimator with the block's receipts and the epoch's block limit. This allows gas estimation to adapt to current network conditions.
