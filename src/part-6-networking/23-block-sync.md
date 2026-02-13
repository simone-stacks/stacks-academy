# Chapter 23: Block Synchronization

## Purpose

A Stacks node that has just started or fallen behind needs to discover which blocks exist on the network and download the ones it is missing. This is the job of the block synchronization subsystem -- two cooperating state machines that first learn *what* data peers have (inventory sync), then *fetch* the missing pieces (block download).

The system must handle two fundamentally different block models: **Epoch 2.x** (one Stacks block per sortition, plus microblock streams) and **Nakamoto** (multiple blocks per tenure, organized into tenures rather than individual sortitions). Both inventory and download machinery exist in parallel versions for each epoch.

## Key Concepts

**Inventory sync comes first.** Before downloading any blocks, a node must know which blocks each peer has. It does this by exchanging compact bit vectors that describe block availability per reward cycle.

**Block download follows inventory.** Once the node knows who has what, it schedules downloads -- prioritizing missing data and spreading requests across peers to avoid overloading any single one.

**The two epochs differ fundamentally.** Epoch 2.x tracks individual blocks and microblock streams per sortition. Nakamoto tracks *tenures* -- contiguous ranges of blocks signed by the same miner between two sortitions. This changes the granularity of both inventories and downloads.

## Architecture

The block sync subsystem spans two directories:

- `stackslib/src/net/inv/` -- Inventory synchronization (what peers have)
- `stackslib/src/net/download/` -- Block downloading (fetching what we lack)

Each directory contains an `epoch2x` module and a `nakamoto` module.

The overall flow, driven by `PeerNetwork::dispatch_network()` via the `PeerNetworkWorkState` state machine:

```
GetPublicIP -> BlockInvSync -> BlockDownload -> AntiEntropy -> Prune
                  ^                  |
                  |                  |
           inv/epoch2x.rs    download/epoch2x.rs
           inv/nakamoto.rs   download/nakamoto/
```

## Epoch 2.x Inventory Sync

### InvState

The epoch 2.x inventory state machine is `InvState` (`stackslib/src/net/inv/epoch2x.rs:934`):

```rust
pub struct InvState {
    pub block_stats: HashMap<NeighborKey, NeighborBlockStats>,
    request_timeout: u64,
    first_block_height: u64,
    pub last_change_at: u64,
    sync_interval: u64,
    pub hint_learned_data: bool,
    pub hint_learned_data_height: u64,
    hint_do_rescan: bool,
    last_rescanned_at: u64,
    num_inv_syncs: u64,
}
```

The `block_stats` map stores per-neighbor `NeighborBlockStats`, which contains the `PeerBlocksInv` for that peer.

### PeerBlocksInv

Defined in `stackslib/src/net/inv/epoch2x.rs:46`:

```rust
pub struct PeerBlocksInv {
    pub block_inv: Vec<u8>,          // bitmap of anchored blocks
    pub microblocks_inv: Vec<u8>,    // bitmap of microblock streams
    pub pox_inv: Vec<u8>,            // bitmap of PoX anchor block knowledge
    pub num_sortitions: u64,
    pub num_reward_cycles: u64,
    pub last_updated_at: u64,
    pub first_block_height: u64,
}
```

Each bit in `block_inv` corresponds to a sortition. Bit `i` being set means the peer has the block for sortition `first_block_height + i`. The `microblocks_inv` has the same layout for microblock streams.

### The Protocol

Inventory exchange uses two rounds of request/response:

1. **PoX Inventory** (`GetPoxInv` / `PoxInv`): First, the node asks each peer which PoX reward cycles it knows about. The response is a bitvector where bit `i` means the peer knows the anchor block status for reward cycle `i`. This tells the node whether it even makes sense to ask for block inventories in that cycle.

2. **Block Inventory** (`GetBlocksInv` / `BlocksInv`): For each reward cycle where PoX inventories align, the node sends a `GetBlocksInv` with the cycle's starting consensus hash and number of sortitions. The peer replies with `BlocksInvData`:

```rust
pub struct BlocksInvData {
    pub bitlen: u16,
    pub block_bitvec: Vec<u8>,
    pub microblocks_bitvec: Vec<u8>,
}
```

The `INV_SYNC_INTERVAL` is 150 seconds in production (`stackslib/src/net/inv/epoch2x.rs:39`), and the system syncs `INV_REWARD_CYCLES = 2` reward cycles at a time.

## Nakamoto Inventory Sync

### NakamotoInvStateMachine

The Nakamoto inventory state machine (`stackslib/src/net/inv/nakamoto.rs:710`) takes a fundamentally different approach:

```rust
pub struct NakamotoInvStateMachine<NC: NeighborComms> {
    pub(crate) comms: NC,
    pub(crate) inventories: HashMap<NeighborAddress, NakamotoTenureInv>,
    reward_cycle_consensus_hashes: BTreeMap<u64, ConsensusHash>,
    last_sort_tip: Option<BlockSnapshot>,
    burst_deadline_ms: u128,
    last_burst_ms: u128,
}
```

Instead of tracking individual blocks, it tracks *tenures*.

### NakamotoTenureInv

Per-peer tenure inventory (`stackslib/src/net/inv/nakamoto.rs:460`):

```rust
pub struct NakamotoTenureInv {
    pub tenures_inv: BTreeMap<u64, BitVec<2100>>,  // reward cycle -> tenure bitmap
    pub last_updated_at: u64,
    pub first_block_height: u64,
    pub reward_cycle_len: u64,
    pub neighbor_address: NeighborAddress,
    pub cur_reward_cycle: u64,
    pub online: bool,
    pub start_sync_time: u64,
}
```

The `tenures_inv` maps each reward cycle to a `BitVec<2100>` (the maximum reward cycle length on mainnet is 2100 sortitions). Bit `j` is set if the peer has processed all blocks for the tenure that started at burnchain block `cycle_start + j`.

### The Protocol

Nakamoto uses `GetNakamotoInv` / `NakamotoInv` messages:

```rust
pub struct GetNakamotoInvData {
    pub consensus_hash: ConsensusHash,  // start of reward cycle
}

pub struct NakamotoInvData {
    pub tenures: BitVec<2100>,  // one bit per sortition in the cycle
}
```

A bit is set when (1) there was a sortition at that position AND (2) the peer has processed all blocks for that tenure. This is strictly more informative than epoch 2.x inventories because it tells you about tenure completeness, not just individual block presence.

### InvGenerator and Caching

Generating inventory responses requires querying both the sortition DB and chainstate for tenure information. Since this happens frequently, the `InvGenerator` (`stackslib/src/net/inv/nakamoto.rs:107`) aggressively caches:

```rust
pub struct InvGenerator {
    processed_tenures: HashMap<StacksBlockId, HashMap<ConsensusHash, Option<InvTenureInfo>>>,
    sortitions: HashMap<ConsensusHash, InvSortitionInfo>,
    tip_ancestor_search_depth: u64,
    cache_misses: u128,
}
```

Tenure information is immutable once written, so caching is safe. The `tip_ancestor_search_depth` (default 10) limits how far back the generator searches for ancestor tips when the chain tip changes, keeping overhead bounded.

## Epoch 2.x Block Download

### BlockDownloader

The epoch 2.x downloader (`stackslib/src/net/download/epoch2x.rs:171`):

```rust
pub struct BlockDownloader {
    state: BlockDownloaderState,
    pox_id: PoxId,
    block_sortition_height: u64,
    microblock_sortition_height: u64,
    next_block_sortition_height: u64,
    next_microblock_sortition_height: u64,
    num_blocks_downloaded: u64,
    num_microblocks_downloaded: u64,
    empty_block_download_passes: u64,
    empty_microblock_download_passes: u64,
    pub finished_scan_at: u64,
    last_inv_update_at: u64,
    max_inflight_requests: u64,
    blocks_to_try: HashMap<u64, VecDeque<BlockRequestKey>>,
    microblocks_to_try: HashMap<u64, VecDeque<BlockRequestKey>>,
    // ...
}
```

The downloader scans the inventory state to find blocks we are missing, then issues HTTP requests to peers who claim to have them. It tracks:

- **What to download next** via sortition height cursors (`block_sortition_height`, `microblock_sortition_height`)
- **Download candidates** in `blocks_to_try` -- for each sortition height, a queue of peers to try
- **In-flight limits** via `max_inflight_requests` to avoid overwhelming peers

The download state machine cycles through: scanning the inventory for missing blocks, issuing requests, collecting responses, and advancing the cursor.

## Nakamoto Block Download

The Nakamoto downloader is significantly more complex because tenures can contain many blocks. It lives in `stackslib/src/net/download/nakamoto/`.

### NakamotoDownloadStateMachine

The top-level state machine (`download_state_machine.rs:63`):

```rust
pub struct NakamotoDownloadStateMachine {
    nakamoto_start_height: u64,
    pub(crate) reward_cycle: u64,
    pub(crate) wanted_tenures: Vec<WantedTenure>,
    pub(crate) prev_wanted_tenures: Option<Vec<WantedTenure>>,
    last_sort_tip: Option<BlockSnapshot>,
    state: NakamotoDownloadState,
    tenure_block_ids: HashMap<NeighborAddress, AvailableTenures>,
    pub(crate) available_tenures: HashMap<ConsensusHash, Vec<NeighborAddress>>,
    pub(crate) tenure_download_schedule: VecDeque<ConsensusHash>,
    tenure_downloads: NakamotoTenureDownloaderSet,
    neighbor_rpc: NeighborRPC,
    nakamoto_tip: StacksBlockId,
    // ...
}
```

It operates in two modes (`NakamotoDownloadState`):

```rust
pub enum NakamotoDownloadState {
    Confirmed,     // IBD: download confirmed tenures using inventory + sortition data
    Unconfirmed,   // steady-state: download the latest 2 unconfirmed tenures
}
```

### WantedTenure

`WantedTenure` (`tenure.rs:26`) represents a tenure the node wants to download:

```rust
pub struct WantedTenure {
    pub tenure_id_consensus_hash: ConsensusHash,
    pub winning_block_id: StacksBlockId,
    pub burn_height: u64,
    pub processed: bool,
}
```

The `winning_block_id` comes from the block-commit in the sortition DB. The `processed` flag prevents re-downloading completed tenures.

### NakamotoTenureDownloader

Each individual tenure download is handled by a `NakamotoTenureDownloader` (`tenure_downloader.rs:85`):

```rust
pub struct NakamotoTenureDownloader {
    pub tenure_id_consensus_hash: ConsensusHash,
    pub tenure_start_block_id: StacksBlockId,
    pub tenure_end_block_id: StacksBlockId,
    pub naddr: NeighborAddress,
    pub start_signer_keys: RewardSet,
    pub end_signer_keys: RewardSet,
    pub idle: bool,
    pub state: NakamotoTenureDownloadState,
    // ...
}
```

It needs both the start and end block IDs for a tenure, plus the signer reward sets for both boundaries (to validate block signatures). The download state machine proceeds:

```rust
pub enum NakamotoTenureDownloadState {
    GetTenureStartBlock(StacksBlockId, u128),  // fetch the first block
    GetTenureEndBlock(StacksBlockId, u128),    // fetch the last block (boundary)
    GetTenureBlocks(StacksBlockId, u128),      // fetch blocks in between
    Done,
}
```

Tenure blocks are fetched from highest to lowest (newest first). The tenure-end block belongs to the *next* tenure, so it serves as a boundary marker -- the downloader fetches blocks backward from the tenure-end block until it reaches the tenure-start block.

### NakamotoTenureDownloaderSet

The `NakamotoTenureDownloaderSet` (`tenure_downloader_set.rs`) manages multiple concurrent tenure downloads. It maps consensus hashes to their individual downloaders and handles scheduling, deduplication, and completion tracking.

### Unconfirmed Tenure Download

The `NakamotoUnconfirmedTenureDownloader` (`tenure_downloader_unconfirmed.rs`) handles the special case of the most recent tenures that have not yet been confirmed by a subsequent sortition. In steady state, the node is always downloading the current and previous unconfirmed tenures from peers who announce having higher tips.

## How the Pieces Fit Together

The `PeerNetworkWorkState` drives the overall sync flow:

```rust
pub enum PeerNetworkWorkState {
    GetPublicIP,     // discover our public IP
    BlockInvSync,    // exchange inventories with peers
    BlockDownload,   // fetch missing blocks
    AntiEntropy,     // push blocks to peers who lack them
    Prune,           // disconnect excess peers
}
```

In `BlockInvSync`, the appropriate inventory state machine runs (epoch 2.x or Nakamoto depending on the current epoch). Once inventories are updated, the state advances to `BlockDownload`, where the downloader uses the freshly-updated inventories to schedule fetches.

`AntiEntropy` is the inverse of download: the node proactively pushes blocks and microblocks to peers whose inventories indicate they are missing data. This helps blocks propagate even when peers are not actively requesting them.

## Code Walkthrough: Nakamoto Inventory Sync

The Nakamoto inventory sync proceeds as follows:

1. **Identify peers to query**: The state machine selects connected peers that advertise Nakamoto support.

2. **Send GetNakamotoInv**: For each peer, send a request for the tenure inventory of the current reward cycle. The request contains the consensus hash at the cycle start.

3. **Process NakamotoInv responses**: Each response is a `BitVec<2100>` indicating which tenures the peer has. The state machine stores this in `inventories[neighbor_address].tenures_inv[reward_cycle]`.

4. **Advance reward cycle**: Once all peers respond (or time out), move to the next reward cycle and repeat.

5. **Burst mode**: After the sortition tip changes, the state machine enters a "burst" mode where it queries peers more aggressively to quickly learn about new tenures.

## Code Walkthrough: Nakamoto Tenure Download

1. **Build wanted tenures**: For each sortition in the current reward cycle, check if there was a winning block-commit. If so, create a `WantedTenure` for it.

2. **Match against inventories**: Cross-reference wanted tenures with the inventory data. For each wanted tenure, find which peers claim to have it. Store in `available_tenures`.

3. **Compute tenure boundaries**: Using block-commit data from the sortition DB, determine each tenure's start block ID and end block ID (which is the start block of the next tenure).

4. **Schedule downloads**: Build a `tenure_download_schedule` queue, prioritizing tenures we need.

5. **Create downloaders**: For each scheduled tenure, create a `NakamotoTenureDownloader` assigned to a specific peer.

6. **Execute downloads**: Each downloader fetches the tenure-start block, the tenure-end block, then fills in all blocks between (highest to lowest). Downloaded blocks are validated against the signer reward set.

7. **Return results**: Completed tenure downloads are collected into `NetworkResult::nakamoto_blocks` for the Relayer to process.

## How It Connects

- **Chapter 22 (P2P Protocol)** provides the message transport for `GetBlocksInv`, `BlocksInv`, `GetNakamotoInv`, `NakamotoInv`, and `NakamotoBlocks`.
- **Chapter 24 (RPC API)** provides the HTTP endpoints used by the Nakamoto downloader to fetch tenure blocks from peers.
- **Chapter 26 (Relay and Mempool)** consumes the `NetworkResult` to process downloaded blocks into chainstate.
- **Chapter 18 (Nakamoto Consensus)** defines the tenure model that the Nakamoto inventory and download systems are built around.

## Edge Cases and Gotchas

1. **Dual state machines during transition**: During the epoch 2.x to Nakamoto transition, both `inv_state` (epoch 2.x) and `inv_state_nakamoto` run simultaneously. The `PeerNetwork` has separate `work_state` and `nakamoto_work_state` fields for this.

2. **Tenure boundary ambiguity**: A tenure's end block ID is the start block of the *next* tenure. If the next tenure has not yet been committed, the boundary is unknown, which is why the unconfirmed tenure downloader exists as a separate mechanism.

3. **Reward set validation**: Nakamoto block downloads must validate signer signatures. The downloader carries `start_signer_keys` and `end_signer_keys` to verify blocks at both boundaries. If the reward set is not available (e.g., we have not yet processed the PoX anchor block), the download cannot proceed.

4. **Inventory staleness**: The `NakamotoInvStateMachine` tracks `last_sort_tip` to detect when inventories need to be refreshed. If the sortition tip advances, all inventories may be stale for the latest reward cycle.

5. **Backward block fetching**: Nakamoto tenure blocks are fetched from highest to lowest because each block's parent hash provides the chain backward. The downloader stores blocks and reverses them before returning.

6. **CHECK_UNCONFIRMED_TENURES_MS = 1000**: Unconfirmed tenure checks happen at most once per second (`download_state_machine.rs:41`), preventing excessive polling during steady-state operation.
