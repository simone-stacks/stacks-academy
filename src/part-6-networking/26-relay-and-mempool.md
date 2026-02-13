# Chapter 26: Relay, Mempool, and Atlas

## Purpose

Once the P2P network delivers new blocks, transactions, and StackerDB chunks to a node, something must process them into persistent state. The **Relayer** is that bridge -- it takes the `NetworkResult` from `PeerNetwork::run()`, validates and stores blocks, feeds transactions into the mempool, and broadcasts data to other peers.

The **Mempool** (`MemPoolDB`) is the node's holding area for unconfirmed transactions. It manages admission control, nonce ordering, fee estimation, and synchronization with peer mempools.

The **Atlas** network provides a content-addressed attachment store for large data blobs referenced by on-chain transactions (historically used by BNS name registrations).

Together, these three subsystems complete the data lifecycle: data arrives from the network, gets validated and stored, and then propagates outward to peers who need it.

## Key Concepts

The **Relayer** bridges the P2P network and persistent state: it receives data from `PeerNetwork::run()`, validates and stores it, and broadcasts results to peers. The **Mempool** (`MemPoolDB`) holds unconfirmed transactions with admission control, replace-by-fee, and nonce ordering. The **Atlas** network provides content-addressed storage for large data blobs referenced by on-chain transactions (historically used by BNS name registrations). Fee estimation chains three components: a `CostEstimator` (predicts execution cost), a `CostMetric` (reduces multi-dimensional cost to a scalar), and a `FeeEstimator` (maps cost scalars to fee rates at multiple confidence levels).

## Architecture

```
PeerNetwork::run()
    |
    | NetworkResult
    v
Relayer::process_network_result()   (relay.rs)
    |
    +--> process_new_epoch2_blocks()      --> SortitionDB + ChainState
    +--> process_new_epoch3_blocks()      --> SortitionDB + ChainState (Nakamoto)
    +--> process_new_transactions()       --> MemPoolDB
    +--> process_uploaded_stackerdb_chunks() --> StackerDBs
    +--> process_stacker_db_chunks()      --> StackerDBs
    +--> process_pushed_stacker_db_chunks() --> StackerDBs
    |
    v
ProcessedNetReceipts
    |
    v
NetworkHandle::broadcast()  --> push to P2P peers
```

## The Relayer

### Relayer Struct

Defined in `stackslib/src/net/relay.rs:106`:

```rust
pub struct Relayer {
    p2p: NetworkHandle,
    connection_opts: ConnectionOptions,
    stacker_dbs: StackerDBs,
    recently_sent_nakamoto_blocks: HashMap<StacksBlockId, (ConsensusHash, u128)>,
}
```

The `Relayer` runs on a separate thread from `PeerNetwork` and communicates with it through `NetworkHandle` (a bounded `SyncSender` channel). This separation means block processing does not block the event loop.

The `recently_sent_nakamoto_blocks` cache prevents redundant re-broadcasting of blocks the node has already pushed to peers.

### RelayerStats

Defined in `stackslib/src/net/relay.rs:120`:

```rust
pub struct RelayerStats {
    pub(crate) relay_stats: HashMap<NeighborAddress, RelayStats>,
    pub(crate) relay_updates: BTreeMap<u64, NeighborAddress>,
    pub(crate) recent_messages: HashMap<NeighborKey, VecDeque<(u64, Sha512Trunc256Sum)>>,
    pub(crate) recent_updates: BTreeMap<u64, NeighborKey>,
    next_priority: u64,
}
```

This tracks which peers have relayed which messages, used for:
- **Duplicate detection**: If a peer sends the same transaction hash multiple times, it is counted but not re-processed
- **Network topology inference**: By tracking relay chains, the system can identify choke points (though this feature is not yet fully utilized)

Constants: `MAX_RELAYER_STATS = 4096`, `MAX_RECENT_MESSAGES = 256`, `MAX_RECENT_MESSAGE_AGE = 600` seconds.

### process_network_result

The main entry point (`stackslib/src/net/relay.rs:2860`) processes a complete `NetworkResult`:

```rust
pub fn process_network_result(
    &mut self,
    local_peer: &LocalPeer,
    network_result: &mut NetworkResult,
    burnchain: &Burnchain,
    sortdb: &mut SortitionDB,
    chainstate: &mut StacksChainState,
    mempool: &mut MemPoolDB,
    ibd: bool,
    coord_comms: Option<&CoordinatorChannels>,
    event_observer: Option<&dyn RelayEventDispatcher>,
) -> Result<ProcessedNetReceipts, net_error>
```

It processes data in a specific order:

1. **Epoch 2.x blocks**: `process_new_epoch2_blocks()` handles both downloaded blocks (from the block downloader) and pushed blocks (from peers). Each block is validated against the sortition DB, stored in chainstate, and if successful, the coordinator is notified.

2. **Nakamoto blocks**: `process_new_epoch3_blocks()` handles Nakamoto blocks from downloads and pushes. These are staged for processing by the coordinator.

3. **Transactions**: `process_new_transactions()` takes pushed transactions (from P2P), uploaded transactions (from HTTP), and synced transactions (from mempool sync), validates them, and adds them to the mempool.

4. **StackerDB chunks**: Three separate methods handle uploaded chunks (from HTTP), downloaded chunks (from sync), and pushed chunks (from P2P). Each validates signatures and stores the data.

The result is `ProcessedNetReceipts`:

```rust
pub struct ProcessedNetReceipts {
    pub mempool_txs_added: Vec<StacksTransaction>,
    pub processed_unconfirmed_state: ProcessedUnconfirmedState,
    pub num_new_blocks: u64,
    pub num_new_confirmed_microblocks: u64,
    pub num_new_unconfirmed_microblocks: u64,
    pub num_new_nakamoto_blocks: u64,
}
```

### Block Relay: Push vs. Pull

Blocks propagate through two complementary mechanisms:

**Push (proactive)**: When a node produces or receives a new block, it broadcasts `BlocksAvailable` / `MicroblocksAvailable` / `NakamotoBlocks` messages to peers. For outbound peers with known inventories, the node checks which peers are missing the block and sends it directly. For inbound peers without known inventories, it sends the availability announcement.

**Pull (reactive)**: The block download state machines (Chapter 23) periodically scan inventories and fetch missing blocks from peers.

**Anti-entropy**: The `AntiEntropy` phase of the `PeerNetworkWorkState` cycle proactively pushes blocks to peers whose inventories indicate they are missing data. This catches cases where push messages were lost or peers joined after the initial broadcast.

The relay constants control fan-out:
```rust
pub const MAX_BROADCAST_OUTBOUND_RECEIVERS: usize = 8;
pub const MAX_BROADCAST_INBOUND_RECEIVERS: usize = 16;
```

### Transaction Relay

Transactions received via `POST /v2/transactions` or P2P `Transaction` messages are relayed by broadcasting them to connected peers. The relay uses the `relayers` vector in `StacksMessage` to prevent loops -- a node will not relay a transaction back to a peer that already appears in its relay chain.

## The Mempool

### MemPoolDB

Defined in `stackslib/src/core/mempool.rs:885`:

```rust
pub struct MemPoolDB {
    pub db: DBConn,
    path: String,
    admitter: MemPoolAdmitter,
    bloom_counter: BloomCounter<BloomNodeHasher>,
    max_tx_tags: u32,
    cost_estimator: Box<dyn CostEstimator>,
    metric: Box<dyn CostMetric>,
    pub blacklist_timeout: u64,
    pub blacklist_max_size: u64,
}
```

The mempool is a SQLite database that stores unconfirmed transactions. Key components:

- **`admitter`**: `MemPoolAdmitter` performs validation checks before a transaction enters the pool
- **`bloom_counter`**: A counting Bloom filter used for efficient mempool synchronization between peers
- **`cost_estimator`**: Estimates the execution cost of transactions (used for fee rate calculation)
- **`metric`**: Reduces multi-dimensional execution costs to a single scalar

### Transaction Admission

When a transaction arrives via `MemPoolDB::submit()` (`mempool.rs:2462`):

```rust
pub fn submit(
    &mut self,
    chainstate: &mut StacksChainState,
    sortdb: &SortitionDB,
    consensus_hash: &ConsensusHash,
    block_hash: &BlockHeaderHash,
    tx: &StacksTransaction,
    event_observer: Option<&dyn MemPoolEventDispatcher>,
    block_limit: &ExecutionCost,
    stacks_epoch_id: &StacksEpochId,
) -> Result<(), MemPoolRejection>
```

The admission pipeline:

1. **Blacklist check**: If the txid is temporarily blacklisted (e.g., it previously caused execution errors), reject immediately.

2. **Fee rate estimation**: Call `cost_estimates::estimate_fee_rate()` to compute the transaction's fee rate. This uses the `CostEstimator` to predict execution cost and the `CostMetric` to reduce it to a scalar, then divides by the estimated byte cost.

3. **Admission checks**: `MemPoolAdmitter::will_admit_tx()` validates:
   - The transaction is well-formed
   - The sender has sufficient balance for the fee
   - The nonce is valid (not already used, not too far in the future)
   - The transaction does not duplicate an existing mempool entry with a lower fee

4. **Storage**: `MemPoolDB::try_add_tx()` stores the transaction. If a transaction with the same origin/nonce already exists, the new one replaces it only if it has a higher fee (replace-by-fee).

5. **Fee rate update**: The estimated `fee_rate` is stored alongside the transaction for use in mining prioritization.

### Transaction Selection for Mining

When a miner builds a block, it walks the mempool to select transactions. The key interface is `MemPoolDB::iterate_candidates()`, which yields transactions ordered by:

1. **Nonce ordering**: Transactions from the same sender must be included in nonce order. A transaction with nonce N cannot be included before nonce N-1.
2. **Fee rate**: Among transactions with ready nonces, higher fee rates are prioritized.

The walk handles both origin nonces and sponsor nonces (for sponsored transactions), ensuring correct ordering for both.

### Mempool Synchronization

Nodes sync their mempools with peers using the `MempoolSync` state machine (`stackslib/src/net/mempool/mod.rs:52`):

```rust
pub struct MempoolSync {
    mempool_state: MempoolSyncState,
    mempool_sync_deadline: u64,
    mempool_sync_timeout: u64,
    mempool_sync_completions: u64,
    pub(crate) mempool_sync_txs: u64,
    api_endpoint: String,
}
```

The sync state machine cycles through:

```rust
pub enum MempoolSyncState {
    PickOutboundPeer,
    ResolveURL(UrlString, DNSRequest, Txid),
    SendQuery(UrlString, SocketAddr, Txid),
    RecvResponse(UrlString, SocketAddr, usize),
}
```

**How it works**:

1. **Pick a peer**: Select a random outbound peer that has a `data_url`.
2. **Resolve DNS**: Resolve the peer's data URL to a socket address.
3. **Send query**: POST to the peer's `/v2/mempool/query` endpoint with a Bloom filter of our mempool state. The Bloom filter encodes which txids we already have.
4. **Receive response**: The peer responds with transactions that are NOT in our Bloom filter (i.e., transactions we are missing).

The Bloom filter (`BloomCounter`) is a counting variant that can be decremented when transactions are mined or expire, preventing false negatives from accumulating. It is pruned when the coinbase height advances (`prune_bloom_counter()`).

Configuration parameters from `ConnectionOptions`:
- `mempool_sync_interval`: How often to sync (seconds)
- `mempool_max_tx_query`: Maximum transactions per query
- `mempool_sync_timeout`: Maximum sync duration before timeout

### Transaction Garbage Collection

The mempool automatically prunes transactions that:
- Have been mined into a confirmed block
- Have nonces that are now below the sender's confirmed nonce
- Have been in the mempool longer than the configured timeout
- Are temporarily blacklisted (transactions that caused errors during trial execution)

The blacklist mechanism prevents the same problematic transaction from being re-admitted repeatedly. Entries expire after `blacklist_timeout` seconds.

## Fee Estimation

The fee estimation pipeline involves three components defined via traits in `stackslib/src/cost_estimates/`:

### CostEstimator

Predicts the execution cost of a transaction before it runs. Takes a `TransactionPayload` and returns an `ExecutionCost` (a multi-dimensional budget: runtime, read_count, read_length, write_count, write_length).

### CostMetric

Reduces the multi-dimensional `ExecutionCost` to a single scalar value. The scalar represents "how much of the block budget this transaction consumes."

### FeeEstimator

Given historical data about transactions in recent blocks, estimates fee rates at three confidence levels (low, middle, high). Returns a `FeeRateEstimate`:

```rust
pub struct FeeRateEstimate {
    pub high: f64,
    pub middle: f64,
    pub low: f64,
}
```

The `POST /v2/fees/transaction` endpoint chains these together: estimate cost, compute scalar, look up fee rates, multiply by scalar to get absolute fees.

## The Atlas Network

Atlas is a content-addressed attachment store, living in `stackslib/src/net/atlas/`. It stores data blobs referenced by on-chain transactions (historically, BNS name registration attachments).

### AtlasConfig

Defined in `stackslib/src/net/atlas/mod.rs:92`:

```rust
pub struct AtlasConfig {
    pub contracts: HashSet<QualifiedContractIdentifier>,
    pub attachments_max_size: u32,
    pub max_uninstantiated_attachments: u32,
    pub uninstantiated_attachments_expire_after: u32,
    pub unresolved_attachment_instances_expire_after: u32,
    pub genesis_attachments: Option<Vec<Attachment>>,
}
```

The `contracts` field specifies which smart contracts can create attachment instances. Only attachments referenced by transactions in these contracts are tracked.

### Architecture

Atlas has three components:

1. **AtlasDB** (`atlas/db.rs`): SQLite storage for attachments and attachment instances. An "instance" is a reference to an attachment from a specific transaction; the "attachment" is the actual data blob.

2. **AttachmentsDownloader** (`atlas/download.rs`): A state machine that discovers which attachments are missing and fetches them from peers. It uses the HTTP API endpoints `GET /v2/attachments/{hash}` and `GET /v2/attachments/inv`.

3. **Inventory exchange**: Nodes exchange attachment inventory pages to discover which attachments peers have, similar to block inventory sync.

### Data Flow

1. A transaction containing an attachment reference is mined into a block.
2. The coordinator creates an `AttachmentInstance` and sends it to the Atlas subsystem.
3. The `AttachmentsDownloader` checks if the actual attachment data is already stored locally.
4. If not, it queries peers for the attachment inventory and downloads missing attachments.
5. Downloaded attachments are verified against the content hash from the on-chain reference and stored in `AtlasDB`.

Key constants:
- `MAX_ATTACHMENT_INV_PAGES_PER_REQUEST = 8`
- `MAX_RETRY_DELAY = 600` seconds
- `ATTACHMENTS_CHANNEL_SIZE = 5` (bounded channel between coordinator and downloader)

### Comparison with StackerDB

Atlas and StackerDB serve different purposes:
- **Atlas**: Content-addressed, blockchain-authenticated attachments. Data is permanent once referenced on-chain. Used for BNS.
- **StackerDB**: Key-value slots, smart-contract-authenticated. Data is ephemeral (per reward cycle). Used for signer coordination.

## How It Connects

- **Chapter 22 (P2P Protocol)**: The Relayer communicates with the P2P layer via `NetworkHandle`. Transaction relay uses `NetworkRequest::Broadcast`. Block relay uses `NetworkRequest::AdvertizeBlocks`.
- **Chapter 23 (Block Sync)**: Downloaded blocks flow through `NetworkResult` into `process_network_result()`.
- **Chapter 24 (RPC API)**: The `POST /v2/transactions` endpoint feeds directly into `MemPoolDB::submit()`. The `POST /v2/fees/transaction` endpoint uses the mempool's fee estimator.
- **Chapter 25 (StackerDB)**: The Relayer processes StackerDB chunks from all three sources (upload, download, push).
- **Chapter 15 (Block Processing)**: Validated blocks from the Relayer are passed to the Coordinator for processing.

## Edge Cases and Gotchas

1. **Nonce gaps block subsequent transactions**: If a sender submits transactions with nonces 0, 1, 3 (skipping 2), transaction 3 cannot be mined until transaction 2 arrives. The mempool stores it but the mining iterator will not yield it.

2. **Replace-by-fee requires strictly higher fees**: To replace a pending transaction with the same nonce, the new transaction must have a strictly higher fee. Equal fees are rejected.

3. **IBD suppresses mempool processing**: During Initial Block Download (`ibd = true`), the Relayer skips certain processing to avoid wasting time on transactions that may be outdated. Mempool sync is also disabled during IBD.

4. **Bloom filter false positives cause missed transactions**: The mempool sync Bloom filter can produce false positives (claiming we have a transaction when we don't). This means some transactions may not be synced in a given round. The periodic re-sync and the counting Bloom filter mitigate this over time.

5. **recently_sent_nakamoto_blocks prevents re-broadcast storms**: Without this cache, a node receiving a block from 10 peers would try to broadcast it 10 times. The cache, keyed by `StacksBlockId` and timestamped, ensures each block is broadcast at most once per time window.

6. **Sponsored transaction nonce ordering**: Sponsored transactions have both an origin nonce and a sponsor nonce. The mempool must track and order by both. If the sponsor's nonce is not yet valid, the transaction is deferred even if the origin's nonce is ready.

7. **BlocksAvailableMap uses BurnchainHeaderHash as key**: The relay layer indexes available blocks by their burnchain header hash (`relay.rs:54`), not by Stacks block hash. This is because the P2P protocol's inventory system tracks block availability relative to burnchain sortitions.
