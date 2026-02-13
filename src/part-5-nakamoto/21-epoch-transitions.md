# Chapter 21: Epoch Transitions

## Purpose

Stacks uses a system of **epochs** to activate new protocol features and consensus rules at specific Bitcoin block heights. An epoch transition is the mechanism by which the node detects that a new epoch has begun and applies the corresponding changes -- upgrading the Clarity VM, deploying new boot contracts, altering cost limits, and modifying consensus rules. This chapter covers how epochs are defined, how transitions are triggered during block processing, and the special constraints around the Nakamoto (epoch 3.0) transition.

## Key Concepts

### What Is an Epoch?

An epoch is a contiguous range of burnchain (Bitcoin) block heights during which a specific set of protocol rules apply. Each epoch has:

- An **epoch ID** that identifies the rule set.
- A **start height** (inclusive) on the burnchain.
- An **end height** (exclusive) on the burnchain.
- A **block limit** (execution cost budget per block/tenure).
- A **network epoch** version byte used in P2P protocol negotiation.

Epochs are defined at compile time (for mainnet) or at configuration time (for testnet). They are stored in the SortitionDB and queried during block processing.

### Epoch IDs

The complete set of epoch IDs is defined at `stacks-common/src/types/mod.rs:120-133`:

```rust
define_stacks_epochs! {
    Epoch10 = 0x01000,   // Pre-Stacks 2.0 (Bitcoin-only era)
    Epoch20 = 0x02000,   // Stacks 2.0 launch
    Epoch2_05 = 0x02005, // Bug fixes
    Epoch21 = 0x0200a,   // Stacks 2.1 (Clarity 2, pay-to-alt-recipient coinbase)
    Epoch22 = 0x0200f,   // PoX-2 disabled (defaulted to burn)
    Epoch23 = 0x02014,   // PoX-2 disabled continued
    Epoch24 = 0x02019,   // PoX-3 activation
    Epoch25 = 0x0201a,   // PoX-4 activation, signer set computation
    Epoch30 = 0x03000,   // Nakamoto activation
    Epoch31 = 0x03001,   // Post-Nakamoto improvements
    Epoch32 = 0x03002,   // Further improvements
    Epoch33 = 0x03003,   // Latest epoch (SIP-034 granular extensions)
}
```

These values are integer-encoded to support comparison ordering: epoch IDs compare numerically, so `Epoch20 < Epoch25 < Epoch30`.

The `StacksEpochId::ALL` constant provides a static slice of all epoch IDs, and `StacksEpochId::latest()` returns the most recent epoch (currently `Epoch33`).

## Key Data Structures

The epoch system is built around three types: `StacksEpochId` (an enum identifying each epoch, e.g., `Epoch20`, `Epoch30`), `StacksEpoch` (a struct holding the start/end heights, block limit, and network version for one epoch), and `EpochList` (a sorted collection of `StacksEpoch` entries that supports lookup by height or ID). These are defined in `stacks-common/src/types/mod.rs`.

## Architecture

### The StacksEpoch Structure

Defined at `stacks-common/src/types/mod.rs:1085-1091`:

```rust
pub struct StacksEpoch<L> {
    pub epoch_id: StacksEpochId,
    pub start_height: u64,
    pub end_height: u64,
    pub block_limit: L,
    pub network_epoch: u8,
}
```

The generic parameter `L` is the block limit type. In practice, this is always `ExecutionCost` for real usage, but the generic allows test flexibility.

### EpochList

Epochs are stored in an `EpochList<L>` wrapper (`mod.rs:1132-1184`), which provides indexed access:

```rust
pub struct EpochList<L: Clone>(Vec<StacksEpoch<L>>);

impl<L: Clone> EpochList<L> {
    pub fn epoch_at_height(&self, height: u64) -> Option<StacksEpoch<L>> { ... }
    pub fn epoch_id_at_height(&self, height: u64) -> Option<StacksEpochId> { ... }
    pub fn get(&self, index: StacksEpochId) -> Option<&StacksEpoch<L>> { ... }
}
```

The `find_epoch` helper (`mod.rs:1097-1104`) performs a linear scan to find which epoch a given burnchain height falls into:

```rust
pub fn find_epoch(epochs: &[StacksEpoch<L>], height: u64) -> Option<usize> {
    for (i, epoch) in epochs.iter().enumerate() {
        if epoch.start_height <= height && height < epoch.end_height {
            return Some(i);
        }
    }
    None
}
```

### Epoch Configuration

For mainnet, epochs are hardcoded. For testnet, they can be overridden via the node configuration file. The `StacksEpochExtension::get_epochs` method at `stackslib/src/core/mod.rs:893-903` resolves this:

```rust
fn get_epochs(
    bitcoin_network: BitcoinNetworkType,
    configured_epochs: Option<&EpochList>,
) -> EpochList {
    match configured_epochs {
        Some(epochs) => {
            assert!(bitcoin_network != BitcoinNetworkType::Mainnet);
            epochs.clone()
        }
        None => get_bitcoin_stacks_epochs(bitcoin_network),
    }
}
```

Mainnet epochs cannot be overridden -- the assertion enforces this.

## Code Walkthrough: The Epoch Transition Mechanism

### Detection: Comparing Parent vs Sortition Epoch

Epoch transitions are detected during block setup (when processing or mining a block). The entry point is `StacksChainState::process_epoch_transition` at `stackslib/src/chainstate/stacks/db/blocks.rs:3950-4048`.

The core detection logic:

```rust
pub fn process_epoch_transition(
    clarity_tx: &mut ClarityTx,
    burn_dbconn: &dyn BurnStateDB,
    chain_tip_burn_header_height: u32,
) -> Result<(bool, Vec<StacksTransactionReceipt>), Error> {
    let (stacks_parent_epoch, sortition_epoch) = clarity_tx
        .with_clarity_db_readonly(|db| {
            Ok((
                db.get_clarity_epoch_version()?,      // epoch of parent block
                db.get_stacks_epoch(chain_tip_burn_header_height), // epoch per sortition DB
            ))
        })?;

    let mut current_epoch = stacks_parent_epoch;
    while current_epoch != sortition_epoch.epoch_id {
        // Apply one transition at a time
        applied = true;
        // ... match on current_epoch and call initialize_epoch_*
    }

    Ok((applied, receipts))
}
```

Two epoch values are compared:
1. **`stacks_parent_epoch`** -- The epoch stored in the Clarity DB, reflecting the parent block's epoch.
2. **`sortition_epoch`** -- The epoch determined by the burnchain height of this block's sortition.

If these differ, the block is the first in a new epoch, and transitions are applied sequentially.

### Sequential Application

Transitions are applied one at a time in order. If the parent was in Epoch 2.4 and the current burnchain height is in Epoch 3.0, the code applies 2.4 -> 2.5, then 2.5 -> 3.0:

```rust
match current_epoch {
    StacksEpochId::Epoch20 => {
        receipts.push(clarity_tx.block.initialize_epoch_2_05()?);
        current_epoch = StacksEpochId::Epoch2_05;
    }
    StacksEpochId::Epoch2_05 => {
        receipts.append(&mut clarity_tx.block.initialize_epoch_2_1()?);
        current_epoch = StacksEpochId::Epoch21;
    }
    // ... each epoch transitions to the next
    StacksEpochId::Epoch25 => {
        receipts.append(&mut clarity_tx.block.initialize_epoch_3_0()?);
        current_epoch = StacksEpochId::Epoch30;
    }
    StacksEpochId::Epoch30 => {
        receipts.append(&mut clarity_tx.block.initialize_epoch_3_1()?);
        current_epoch = StacksEpochId::Epoch31;
    }
    // ... continuing through all epochs
    StacksEpochId::Epoch33 => {
        panic!("No defined transition from Epoch33 forward")
    }
}
```

A safety assertion prevents backwards transitions:

```rust
assert!(current_epoch < sortition_epoch.epoch_id,
    "The SortitionDB believes the epoch is earlier than this Stacks block's parent");
```

After each transition step (past Epoch 2.05), the code verifies consistency:

```rust
assert_eq!(clarity_tx.block.get_epoch(), current_epoch);
assert_eq!(clarity_tx.block.get_clarity_db_epoch_version(burn_dbconn)?, current_epoch);
```

### What Each Transition Does

Each `initialize_epoch_*` method in `stackslib/src/clarity_vm/clarity.rs` performs epoch-specific setup. Common patterns include:

- **Bumping the epoch version** in the Clarity DB via `db.set_clarity_epoch_version(epoch_id)`.
- **Deploying new boot contracts** (e.g., PoX-4 in Epoch 2.5, `.signers` and `.signers-voting`).
- **Re-deploying existing boot contracts** with updated code (e.g., costs-v3 in Epoch 3.0).
- **Setting the Clarity version** for the boot code.
- **Epoch initialization is cost-free** -- a `LimitedCostTracker::new_free()` is used during transitions.

For example, `initialize_epoch_3_0` at `clarity_vm/clarity.rs:1680`:

```rust
pub fn initialize_epoch_3_0(&mut self) -> Result<Vec<StacksTransactionReceipt>, ClarityError> {
    using!(self.cost_track, "cost tracker", |old_cost_tracker| {
        self.cost_track.replace(LimitedCostTracker::new_free());
        self.epoch = StacksEpochId::Epoch30;
        self.as_transaction(|tx_conn| {
            tx_conn.with_clarity_db(|db| {
                db.set_clarity_epoch_version(StacksEpochId::Epoch30)?;
                Ok(())
            }).unwrap();
            // ... deploy/update boot contracts
        })
    })
}
```

### Recording the Transition

After applying the transition, the block records the event in the `epoch_transitions` table. This INSERT appears in two locations -- once for Nakamoto blocks (`nakamoto/mod.rs:3650-3655`) and once for Epoch 2.x blocks (`stacks/db/mod.rs:2856-2861`):

```rust
if applied_epoch_transition {
    let sql = "INSERT INTO epoch_transitions (block_id) VALUES (?)";
    let args = params![index_block_hash];
    headers_tx.deref_mut().execute(sql, args)?;
}
```

This allows nodes to quickly determine which block triggered an epoch transition.

## Epoch-Gated Features

Many features are gated by epoch checks throughout the codebase. Here are key patterns:

### Clarity Version

Different epochs support different Clarity versions:
- Epoch 2.0-2.05: Clarity 1 only
- Epoch 2.1+: Clarity 1 and Clarity 2
- Epoch 3.0+: Clarity 3

### Mempool Behavior

`stacks-common/src/types/mod.rs:471-486`:

```rust
pub fn mempool_garbage_behavior(&self) -> MempoolCollectionBehavior {
    match self {
        // Epoch 1.0 through 2.5: garbage collect by Stacks height
        StacksEpochId::Epoch10 | ... | StacksEpochId::Epoch25 =>
            MempoolCollectionBehavior::ByStacksHeight,
        // Epoch 3.0+: garbage collect by receive time
        StacksEpochId::Epoch30 | ... | StacksEpochId::Epoch33 =>
            MempoolCollectionBehavior::ByReceiveTime,
    }
}
```

This changes because in Nakamoto, Stacks height increases much faster (multiple blocks per Bitcoin block), making height-based GC too aggressive.

### Analysis Memory Checks

Epoch 2.5+ enables memory limit checks during Clarity analysis (`mod.rs:491-499`).

### PoX Contract Selection

The active PoX contract depends on the burnchain height, not directly on epoch:
- PoX-1: Early epochs
- PoX-2: Epoch 2.1+ (disabled in 2.2/2.3)
- PoX-3: Epoch 2.4
- PoX-4: Epoch 2.5+

## The Nakamoto Transition (Epoch 2.5 -> 3.0)

The transition to Nakamoto is the most complex epoch transition, with strict scheduling requirements.

### Scheduling Constraints

`validate_nakamoto_transition_schedule` at `stackslib/src/core/mod.rs:2386-2467` enforces:

1. **Epoch 3.0 must start during a reward phase**, not a prepare phase:
```rust
assert!(!burnchain.is_in_prepare_phase(epoch_3_0_start));
```

2. **Epoch 3.0 must start at or after reward cycle 2**:
```rust
assert!(activation_reward_cycle >= 2);
```

3. **Epoch 2.5 must start before the prepare phase of the cycle prior to Epoch 3.0**:
```rust
assert!(epoch_2_5_start < epoch_3_0_prepare_phase_start);
```

4. **Epoch 2.5 and 3.0 must not be in the same reward cycle**:
```rust
assert_ne!(epoch_2_5_reward_cycle, epoch_3_0_reward_cycle);
```

5. **Epoch 3.0 must not start at a reward cycle boundary** (offset 0 or 1):
```rust
assert!(epoch_3_0_start % reward_cycle_length > 1);
```

6. **Epoch 2.5 end must equal Epoch 3.0 start** (no gaps):
```rust
assert_eq!(epoch_2_5_end, epoch_3_0_start);
```

### Why These Constraints?

The constraints exist because:

- **Epoch 2.5 computes signer sets**: The `.signers` contract is deployed in Epoch 2.5 and begins computing reward sets for the first Nakamoto reward cycle. This must happen at least one full reward cycle before Nakamoto activates.

- **Not during prepare phase**: The prepare phase computes the next reward cycle's reward set. Starting Nakamoto mid-prepare would create confusion about which rules apply to the reward set computation.

- **Not at cycle boundary**: Epoch 2.5 has confusing boundary logic at offsets 0 and 1 that could interfere with reward set calculations.

### The Coordinator's Role

The coordinator handles the transition by running both epoch 2 and Nakamoto code paths during `in_first_nakamoto_reward_cycle()`, as described in Chapter 20. The `in_nakamoto_epoch` flag tracks whether the canonical tip has switched to Nakamoto blocks.

## Coinbase Schedule (SIP-029)

Epochs also affect the coinbase reward schedule. The `CoinbaseInterval` struct at `stacks-common/src/types/mod.rs:144-149`:

```rust
pub struct CoinbaseInterval {
    pub coinbase: u128,
    pub effective_start_height: u64,
}
```

Mainnet intervals (from SIP-029) at `mod.rs:165-190`:

| Effective Height | Coinbase |
|-----------------|----------|
| 0               | 1000 STX |
| 278,950         | 500 STX  |
| 383,950         | 250 STX  |
| 593,950         | 125 STX  |
| 803,950         | 62.5 STX |

These heights are measured from the Stacks genesis block, providing a fixed emission schedule regardless of epoch transitions.

### SIP-031 Emissions

Epoch 3.2+ introduces additional STX emissions via SIP-031, defined by `SIP031EmissionInterval` (`mod.rs:286-291`):

```rust
pub struct SIP031EmissionInterval {
    pub amount: u128,
    pub start_height: u64,
}
```

These emissions are added per-tenure at specific burnchain heights, supplementing the base coinbase.

## How It Connects

- **Chapter 5 (Clarity VM)**: Epoch transitions deploy and update Clarity boot contracts.
- **Chapter 16 (Stacking and PoX)**: PoX contract versions are tied to epochs.
- **Chapter 18 (Nakamoto Consensus)**: Epoch 3.0 activates the Nakamoto consensus rules.
- **Chapter 20 (Coordinator)**: The coordinator's main loop branches based on epoch.
- **Chapter 29 (Node Startup)**: The `BootRunLoop` detects the Epoch 3.0 boundary and switches from the Neon run loop to the Nakamoto run loop.

## Edge Cases and Gotchas

1. **Multiple epoch transitions in one block**: If a node is far behind and a single block spans multiple epochs (e.g., jumping from 2.4 to 3.0), all intermediate transitions are applied sequentially. The `while` loop ensures no epochs are skipped.

2. **Transition is cost-free**: All epoch initialization runs with `LimitedCostTracker::new_free()`, meaning it doesn't count against the block/tenure cost budget. This ensures transitions always succeed regardless of budget constraints.

3. **The Epoch33 panic**: If the current epoch is already `Epoch33` and the sortition DB thinks it should be later, the code panics. This is a safety net -- a new epoch transition handler must be added before any post-33 epoch is defined.

4. **Testnet epoch overrides**: Testnet can use custom epoch heights (starting them earlier or later), but mainnet cannot. The `get_epochs` function asserts against overriding mainnet epochs.

5. **Epoch 2.2/2.3 burned all PoX rewards**: During these epochs, PoX was disabled and all rewards were burned. This was a response to a PoX exploit.

6. **The `network_epoch` byte**: Each epoch has a `network_epoch` version used in P2P handshakes. This allows nodes to determine peer compatibility. Nodes in different epochs may use different message formats.

7. **The `epoch_transitions` table**: This table stores the `block_id` of every block that triggered an epoch transition. It is used during node startup to quickly identify transition blocks without re-scanning the entire chain.

8. **Epoch transitions are recorded in the MARF**: The `applied_epoch_transition` flag in `SetupBlockResult` is propagated through block processing and recorded alongside the block header, making it part of the indexed chainstate.
