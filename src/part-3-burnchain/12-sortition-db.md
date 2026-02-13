# Chapter 12: Sortition DB

## Purpose

The SortitionDB is the persistent store for everything related to cryptographic sortition -- the process by which Stacks selects a block producer (or miner) for each Bitcoin block. Every Bitcoin block triggers a sortition: the node evaluates all block commits in that block, computes a weighted random selection based on how much BTC each miner burned, and records the result as a `BlockSnapshot`. The SortitionDB stores these snapshots alongside the raw operations (key registrations, block commits, stacking operations), epoch definitions, and the full fork history of the burnchain.

This database is backed by a MARF (Merklized Adaptive Radix Forest) layered over SQLite. The MARF provides efficient fork-aware key-value lookups, allowing the node to query any point in the burnchain's fork history without replaying state. This chapter covers the database schema, the `BlockSnapshot` structure, how consensus hashes are computed, how the sortition algorithm selects winners, and the null miner mechanism introduced in Nakamoto.

## Key Concepts

**BlockSnapshot.** Each Bitcoin block processed by the Stacks node produces exactly one `BlockSnapshot` (`stackslib/src/chainstate/burn/mod.rs:70-113`). The snapshot records whether a sortition occurred, who won, the cumulative burn total, the consensus hash, and memoized state about the canonical Stacks chain tip. Snapshots are the fundamental unit of sortition history.

**SortitionHash.** A rolling hash that accumulates entropy from Bitcoin block headers. Defined at `burn/mod.rs:52-55`, it is computed by mixing each new Bitcoin block header hash into the previous sortition hash via SHA-256. The sortition hash is then combined with the last winning miner's VRF seed to produce the random index that selects the winner.

**ConsensusHash.** A 20-byte identifier that uniquely identifies a particular sortition fork at a particular height. It is computed as RIPEMD-160(SHA-256(fork_version ++ burn_header_hash ++ ops_hash ++ total_burn ++ pox_id ++ geometric_series_of_prev_consensus_hashes)). The geometric series means each consensus hash transitively commits to an exponentially growing portion of history, enabling efficient fork detection.

**Null miner.** Introduced in Epoch 3.0 (Nakamoto), the null miner is a virtual participant that can "win" sortition if the total BTC committed in a block is significantly lower than the recent median. This mechanism prevents miners from temporarily reducing their spend to manipulate sortition probabilities. When the null miner wins, no block is produced for that sortition.

**MARF indexing.** The SortitionDB uses a MARF to index snapshots across burnchain forks. Each `SortitionId` is a key in the MARF trie, and the MARF's fork-aware structure means the node can answer queries like "what was the consensus hash at height N in fork F?" without maintaining separate copies of state for each fork.

## Architecture

The sortition subsystem spans several files:

| File | Responsibility |
|------|---------------|
| `chainstate/burn/db/sortdb.rs` | Database struct, schema, queries, MARF integration |
| `chainstate/burn/sortition.rs` | Sortition algorithm: `make_snapshot`, `select_winning_block`, null miner |
| `chainstate/burn/distribution.rs` | `BurnSamplePoint`, UTXO chain linking, min-median distribution |
| `chainstate/burn/mod.rs` | `BlockSnapshot`, `SortitionHash`, `ConsensusHash`, `OpsHash` |
| `chainstate/burn/atc.rs` | Assumed Total Commit (ATC) rational arithmetic and lookup table |

The flow for processing a new Bitcoin block is:

1. The coordinator feeds parsed burn operations into the SortitionDB.
2. Operations are stored in their respective tables (leader_keys, block_commits, stack_stx, etc.).
3. `BlockSnapshot::make_snapshot()` is called, which computes the burn distribution, runs the sortition, evaluates the null miner (in Epoch 3.0+), and produces a new `BlockSnapshot`.
4. The snapshot is inserted into the `snapshots` table and indexed by the MARF.

## Key Data Structures

### SortitionDB

Defined at `sortdb.rs:754-775`:

```rust
pub struct SortitionDB {
    pub readwrite: bool,
    pub dryrun: bool,
    pub marf: MARF<SortitionId>,
    pub first_block_height: u64,
    pub first_burn_header_hash: BurnchainHeaderHash,
    pub pox_constants: PoxConstants,
    pub path: String,
}
```

| Field | Description |
|-------|-------------|
| `readwrite` | Controls whether write operations (transactions, migrations) are permitted |
| `dryrun` | When true, writes are permitted but not committed. Used by `stacks-inspect` for simulation |
| `marf` | The MARF handle that indexes sortition state across burnchain forks |
| `first_block_height` | The Bitcoin block height where Stacks monitoring begins |
| `first_burn_header_hash` | Hash of the first monitored Bitcoin block |
| `pox_constants` | PoX parameters: reward cycle length, prepare phase length, activation heights |
| `path` | Filesystem path to the database |

Transaction and handle types wrap the MARF connection with context:

```rust
pub struct SortitionDBTxContext {
    pub first_block_height: u64,
    pub pox_constants: PoxConstants,
    pub dryrun: bool,
}

pub struct SortitionHandleContext {
    pub first_block_height: u64,
    pub pox_constants: PoxConstants,
    pub chain_tip: SortitionId,
    pub dryrun: bool,
}
```

The `SortitionHandleContext` carries a `chain_tip` -- the specific `SortitionId` that anchors all queries to a particular burnchain fork. This is how the MARF resolves fork-ambiguous lookups.

### BlockSnapshot

Defined at `burn/mod.rs:70-113`, this is the per-burn-block state record with over 20 fields:

```rust
pub struct BlockSnapshot {
    pub block_height: u64,
    pub burn_header_timestamp: u64,
    pub burn_header_hash: BurnchainHeaderHash,
    pub parent_burn_header_hash: BurnchainHeaderHash,
    pub consensus_hash: ConsensusHash,
    pub ops_hash: OpsHash,
    pub total_burn: u64,
    pub sortition: bool,
    pub sortition_hash: SortitionHash,
    pub winning_block_txid: Txid,
    pub winning_stacks_block_hash: BlockHeaderHash,
    pub index_root: TrieHash,
    pub num_sortitions: u64,
    pub stacks_block_accepted: bool,
    pub stacks_block_height: u64,
    pub arrival_index: u64,
    pub canonical_stacks_tip_height: u64,
    pub canonical_stacks_tip_hash: BlockHeaderHash,
    pub canonical_stacks_tip_consensus_hash: ConsensusHash,
    pub sortition_id: SortitionId,
    pub parent_sortition_id: SortitionId,
    pub pox_valid: bool,
    pub accumulated_coinbase_ustx: u128,
    pub miner_pk_hash: Option<Hash160>,
}
```

Key fields worth highlighting:

- **`sortition`**: Boolean indicating whether a miner was selected. False when there are no block commits, when burns overflow, or when the null miner wins.
- **`sortition_hash`**: The rolling hash of Bitcoin block headers. Computed as `SHA256(parent_sortition_hash ++ burn_header_hash)` at each block.
- **`winning_block_txid` / `winning_stacks_block_hash`**: Identify the winning block commit and the Stacks block it proposed. All zeros when `sortition == false`.
- **`total_burn`**: Cumulative BTC burned across all sortitions in this fork. If this overflows, future sortitions are denied.
- **`num_sortitions`**: Count of successful sortitions (where `sortition == true`) up to this point.
- **`accumulated_coinbase_ustx`**: When a sortition has no winner, the coinbase reward accumulates and is awarded to the next winner.
- **`pox_valid`**: False if the PoX anchor block for this reward cycle was not confirmed, invalidating PoX reward payouts.
- **`canonical_stacks_tip_*`**: Memoized fields tracking the highest known Stacks block in this burnchain fork, updated as blocks are processed.

### BurnSamplePoint

Defined at `distribution.rs:29-43`, this represents one miner's weighted entry in the sortition lottery:

```rust
pub struct BurnSamplePoint {
    pub burns: u128,          // min(median_burn, most_recent_burn)
    pub median_burn: u128,    // median burn over the UTXO chain
    pub frequency: u8,        // how many blocks this miner committed in the window
    pub range_start: Uint256, // distribution range start in [0, 2^256)
    pub range_end: Uint256,   // distribution range end in [0, 2^256)
    pub candidate: LeaderBlockCommitOp,
}
```

The `burns` field is the effective burn used for probability calculation. The `range_start` and `range_end` define a non-overlapping interval within `[0, 2^256)` proportional to this miner's share of total effective burn.

## Database Schema

The SortitionDB uses a multi-version schema. The core tables from the initial schema (`sortdb.rs:490-619`) are:

**`snapshots`** -- One row per processed Bitcoin block. Stores all `BlockSnapshot` fields. Primary key is `sortition_id`. Has a unique constraint on `consensus_hash` and `index_root`.

**`leader_keys`** -- VRF key registrations. Keyed by `(txid, sortition_id)`. Each row links a miner's VRF public key to the sortition in which it was registered.

**`block_commits`** -- Block commit operations. Keyed by `(txid, sortition_id)`. Stores the full commit data including `block_header_hash`, `new_seed`, parent pointers, key pointers, `burn_fee`, `commit_outs`, and `apparent_sender`.

**`stack_stx`** / **`transfer_stx`** -- Stacking and transfer operations. Keyed by `(txid, burn_header_hash)` -- not by `sortition_id`, because these operations are valid regardless of which sortition fork they appear in.

**`missed_commits`** -- Block commits that landed in a different Bitcoin block than intended. Keyed by `(txid, intended_sortition_id)`. These are still tracked for UTXO chain linking.

**`snapshot_transition_ops`** -- Records which operations were accepted and which keys were consumed at each sortition.

Later schema versions add:

- **`epochs`** (schema v2, `sortdb.rs:621-629`) -- Defines Stacks epoch boundaries with start/end heights and block limits.
- **`block_commit_parents`** (schema v3, `sortdb.rs:631-640`) -- Maps each block commit to its parent's sortition ID for efficient ancestry lookups.
- **`delegate_stx`** (schema v4, `sortdb.rs:642-663`) -- Delegation operations for pooled stacking.
- **`preprocessed_reward_sets`** (schema v8, `sortdb.rs:683-688`) -- Eagerly computed reward sets cached before the next reward cycle starts.
- **`stacks_chain_tips`** (schema v8, `sortdb.rs:692-697`) -- Canonical chain tip at each sortition. Nakamoto relies on this exclusively.
- **`vote_for_aggregate_key`** (schema v8, `sortdb.rs:703-717`) -- Signer votes for the aggregate public key.

The current schema version is v10. Migrations v9 and v10 add additional indexes and columns to support Nakamoto-specific queries.

The database has 22 indexes (`sortdb.rs:725-748`) optimizing common query patterns like looking up snapshots by burn header hash, block commits by sortition ID, and stacking operations by burn header hash.

## Code Walkthrough: The Sortition Algorithm

### Step 1: Build the Burn Distribution

Before sortition runs, `BurnSamplePoint::make_min_median_distribution` (`distribution.rs:167-172`) builds the weighted distribution from the commitment window:

1. **Link UTXO chains.** Starting from the current block's commits, trace each commit's spent UTXO backward through the window. A commit at relative height 0 (most recent) traces its input to find the prior commit by the same miner at height -1, then -1 to -2, etc. Missed commits count as a burn of 1 satoshi (`distribution.rs:80-84`).

2. **Compute median burn.** For each miner's chain of burns across the window, sort the values and take the median (`distribution.rs:279-287`).

3. **Compute effective burn.** The effective burn is `min(median_burn, most_recent_burn)` (`distribution.rs:289`). This prevents a miner from inflating their probability with one large commit while spending minimally in prior blocks.

4. **Track frequency.** Count how many slots in the window the miner actually committed in (`distribution.rs:291-298`). Miners who skip blocks have reduced chances.

5. **Map to ranges.** Each miner's effective burn is mapped to a non-overlapping range within `[0, 2^256)`, proportional to their share of total effective burn.

### Step 2: Select the Winner

`BlockSnapshot::select_winning_block` (`sortition.rs:192-220`) performs the actual selection:

1. **Get the last VRF seed.** Retrieve the VRF seed from the most recent sortition winner in this fork (`sortition.rs:155-179`). If no prior winner exists, use the initial VRF seed.

2. **Sample the distribution.** Compute `HASH(sortition_hash ++ last_VRF_seed)` via `sortition_hash.mix_VRF_seed(vrf_seed)` and convert to a `Uint256` (`sortition.rs:136`). Find which miner's range contains this value.

3. **Return the winner.** The winning `LeaderBlockCommitOp` and its index in the distribution are returned.

The key insight is that the sortition hash accumulates entropy from Bitcoin's proof-of-work (through block header hashes), while the VRF seed provides miner-specific randomness. Together, `HASH(sortition_hash ++ VRF_seed)` produces a value that no single miner can predict or manipulate without controlling Bitcoin's block production.

### Step 3: Null Miner Evaluation (Epoch 3.0+)

After selecting a winner, `make_snapshot_in_epoch` (`sortition.rs:534-697`) applies two additional checks in Nakamoto:

**Miner frequency check** (`sortition.rs:288-311`): If the winning miner did not commit in at least `k` of the last `n` blocks in the window (where `k` is `epoch_id.mining_commitment_frequency()`), the winner is rejected.

**Null miner check** (`sortition.rs:313-487`): The Assumed Total Commit (ATC) measures whether total burns in this block are consistent with the recent median:

```
ATC = min(1, total_block_spend / median_windowed_total_block_spend)
```

If ATC < 1.0 (miners spent less than expected), the null miner can win with probability:

```
NullP(atc) = (1 - atc) + atc * adv(atc)
```

Where `adv(atc)` is a logistic advantage function looked up from a precomputed table (`atc.rs`). The logistic function ensures that even small drops in mining commitment give the null miner a disproportionate advantage, discouraging manipulation.

The null miner sortition (`null_miner_wins`, `sortition.rs:427-487`) creates a two-element burn distribution: one range for the null miner (size proportional to `NullP`) and one for the actual winner (the complement). The same `sample_burn_distribution` function is used, but with this synthetic distribution.

### Step 4: Build the Snapshot

After winner selection and null miner evaluation, a `BlockSnapshot` is constructed. Key computations:

**Consensus hash** (`burn/mod.rs:280-331`):
```rust
fn from_ops(burn_header_hash, opshash, total_burn, prev_consensus_hashes, pox_id) -> ConsensusHash {
    let mut hasher = Sha256::new();
    hasher.update(SYSTEM_FORK_SET_VERSION);
    hasher.update(burn_header_hash);
    hasher.update(opshash);
    hasher.update(&total_burn.to_be_bytes());
    write!(hasher, "{}", pox_id);
    for ch in prev_consensus_hashes {
        hasher.update(ch);
    }
    RIPEMD160(hasher.finalize())
}
```

**Geometric series of previous consensus hashes** (`burn/mod.rs:335-368`): The `get_prev_consensus_hashes` method collects consensus hashes at heights `h-1, h-3, h-7, h-15, ...` (i.e., `h - (2^i - 1)` for `i = 0, 1, 2, ...`). This means each consensus hash transitively commits to all prior history with exponentially decreasing granularity. Verifying that two nodes share the same consensus hash at height N proves they agree on the entire fork history up to N.

**OpsHash** (`burn/mod.rs:218-231`): Simply `SHA256(txid_0 ++ txid_1 ++ ... ++ txid_n)` over all transaction IDs in the block.

**Accumulated coinbase** (`sortition.rs:560-575`): When no sortition occurs, the coinbase reward for that block accumulates. The next successful sortition winner receives the accumulated total plus their own coinbase.

## How It Connects

- **Chapter 10 (Bitcoin Integration)**: The SPV client and block parser produce `BurnchainBlockHeader` and `BitcoinTransaction` objects that feed into the SortitionDB.
- **Chapter 11 (Burn Operations)**: Parsed operations (`LeaderBlockCommitOp`, `LeaderKeyRegisterOp`, etc.) are stored in the SortitionDB and drive the sortition algorithm.
- **Chapter 13 (MARF)**: The SortitionDB is built on top of the MARF, which provides fork-aware indexing via `MARF<SortitionId>`.
- **Chapter 15 (Block Processing)**: After a sortition selects a winning block commit, the Stacks chain state processor attempts to fetch and validate the corresponding Stacks block.
- **Chapter 16 (Stacking and PoX)**: The `pox_constants` in the SortitionDB parameterize reward cycles. The `preprocessed_reward_sets` table caches computed reward sets.
- **Chapter 18 (Nakamoto Consensus)**: Nakamoto introduces the null miner, miner frequency checks, and relies on the `stacks_chain_tips` table for canonical tip tracking.
- **Chapter 20 (Coordinator)**: The coordinator drives the sortition loop, calling `make_snapshot` for each new Bitcoin block.

## Edge Cases and Gotchas

**Total burn overflow.** If `total_burn` (cumulative BTC burned across all sortitions) overflows a `u64`, future sortitions are permanently denied (`sortition.rs:631-638`). The chain remains queryable but cannot make progress. This is an intentional safety mechanism rather than an error.

**Stack/transfer keying.** The `stack_stx` and `transfer_stx` tables use `(txid, burn_header_hash)` as their primary key, not `(txid, sortition_id)`. This is because these operations are valid regardless of which sortition fork they appear in -- the same Bitcoin transaction produces the same stacking effect in all forks (`sortdb.rs:588-591`).

**Missed commits count as 1 satoshi.** When a miner's block commit lands in the wrong Bitcoin block (a missed commit), it is counted as a burn of 1 satoshi in the UTXO chain for commitment smoothing (`distribution.rs:80-84`). This preserves the UTXO chain continuity while giving the miner minimal credit.

**The initial snapshot is special.** `BlockSnapshot::initial()` (`sortition.rs:73-107`) creates the sentinel first snapshot with `sortition: true` but all-zero winning hashes. Its `sortition_id` is derived from the first burn header hash via `SortitionId::stubbed()`, not from a proper MARF computation.

**Dryrun mode.** Setting `dryrun: true` allows write operations to proceed without committing. This exists for the `stacks-inspect` tool, which can replay sortitions with different parameters (e.g., testing alternative anti-MEV strategies) without corrupting the real database (`sortdb.rs:758-762`).

**Schema migrations drop epoch data.** Several schema upgrades (v5, v6, v7, v8) delete all rows from the `epochs` table (`sortdb.rs:667-678`). The epoch definitions are re-inserted from the code after migration, meaning epoch boundaries are always defined by the node's software version, not by database state.

**Consensus hash lifetime.** The constant `CONSENSUS_HASH_LIFETIME` is set to 24 (`burn/mod.rs:43`), but it is not directly used in the geometric series computation. The geometric series naturally covers much more history (up to `2^64 - 1` blocks back). The lifetime constant is used elsewhere for cache expiration.

**SortitionId vs BurnchainHeaderHash.** These are distinct: `BurnchainHeaderHash` is the Bitcoin block hash, while `SortitionId` incorporates the PoX fork identity. Two nodes processing the same Bitcoin block but with different PoX histories will produce different `SortitionId` values but the same `BurnchainHeaderHash`. Queries that need fork-awareness must use `SortitionId`.
