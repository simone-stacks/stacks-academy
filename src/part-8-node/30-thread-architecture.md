# Chapter 30: Thread Architecture

## Purpose

A running Stacks node is a concurrent system with multiple threads communicating through channels and shared state. This chapter maps out the threads, their responsibilities, how they communicate, and the `Globals` structure that serves as the central shared-state hub. Understanding this architecture is essential for debugging deadlocks, tracing data flow, and reasoning about the node's behavior under load.

## Key Concepts

### Thread Roster

A Nakamoto-mode Stacks node runs these major threads:

| Thread | Name Pattern | Responsibility |
|--------|-------------|----------------|
| **Main** | (process main) | Runs the outer run loop, processes sortitions, coordinates startup/shutdown |
| **Relayer** | `relayer:{port}` | Processes network results, manages mining, submits block commits |
| **P2P** | `p2p:({p2p_port},{rpc_port})` | Handles peer connections, serves RPC, relays blocks and transactions |
| **Coordinator** | (spawned in `spawn_chains_coordinator`) | Processes burn blocks, manages sortition DB, chain state |
| **Miner** | (spawned by relayer) | Mines a single Nakamoto block (short-lived, one per mining attempt) |
| **PoX Watchdog** | (part of sync control) | Monitors IBD progress, communicates sync status |
| **Monitoring** | (optional) | Serves Prometheus metrics |
| **Event Dispatcher** | (within relayer/coordinator) | Pushes events to registered observers |

### Communication Patterns

Threads communicate through three mechanisms:
1. **Channels** (`SyncSender`/`Receiver`) -- For directed messages between specific threads
2. **Shared state** (`Arc<Mutex<T>>`, `Arc<AtomicBool>`) -- For state that multiple threads read/write
3. **Coordinator channels** (`CoordinatorChannels`) -- Specialized channel for the coordinator thread

## The Globals Structure

The `Globals` struct (`stacks-node/src/globals.rs:45`) is the central inter-thread communication hub:

```rust
pub struct Globals<T> {
    last_sortition: Arc<Mutex<Option<BlockSnapshot>>>,
    miner_status: Arc<Mutex<MinerStatus>>,
    pub(crate) coord_comms: CoordinatorChannels,
    unconfirmed_txs: Arc<Mutex<UnconfirmedTxMap>>,
    pub relay_send: SyncSender<T>,
    pub counters: Counters,
    pub sync_comms: PoxSyncWatchdogComms,
    pub should_keep_running: Arc<AtomicBool>,
    pub leader_key_registration_state: Arc<Mutex<LeaderKeyRegistrationState>>,
    last_miner_config: Arc<Mutex<Option<MinerConfig>>>,
    last_burnchain_config: Arc<Mutex<Option<BurnchainConfig>>>,
    last_miner_spend_amount: Arc<Mutex<Option<u64>>>,
    start_mining_height: Arc<Mutex<u64>>,
    estimated_winning_probs: Arc<Mutex<HashMap<u64, f64>>>,
    previous_best_tips: Arc<Mutex<BTreeMap<u64, TipCandidate>>>,
    initiative: Arc<Mutex<Option<String>>>,
}
```

The type parameter `T` is the relayer directive type -- for Neon nodes it's `RelayerDirective` from `globals.rs`, for Nakamoto nodes it's `nakamoto_node::relayer::RelayerDirective`. The type alias `NeonGlobals = Globals<RelayerDirective>` is defined at line 23.

### Key Fields Explained

**`last_sortition`**: The most recently processed burn block snapshot. Written by the main thread after processing a sortition, read by other threads to know the current burnchain tip.

**`miner_status`**: Controls whether the miner is allowed to run. Uses `MinerStatus::add_blocked()` / `remove_blocked()` for reference-counted blocking. The relayer blocks the miner during certain operations and unblocks it when ready.

**`relay_send`**: A bounded synchronous channel (`SyncSender<T>`) for sending directives to the relayer thread. The relayer receives from the other end. The buffer size is `RELAYER_MAX_BUFFER = 1`, meaning at most one directive can be buffered.

**`unconfirmed_txs`**: Shared between the relayer and p2p threads. The relayer writes unconfirmed transactions from the chainstate, and the p2p thread reads them to serve RPC requests without doing its own disk I/O.

**`should_keep_running`**: The global shutdown flag. Any thread can set this to `false` to trigger system-wide shutdown. All threads periodically check this flag.

**`leader_key_registration_state`**: Tracks the VRF key registration lifecycle:

```rust
pub enum LeaderKeyRegistrationState {
    Inactive,                              // No key registered
    Pending(u64, Txid),                    // Registration tx sent, waiting for confirmation
    Active(RegisteredKey),                 // Key confirmed and ready for mining
}
```

Written by the relayer (when sending a key registration tx) and the main thread (when the registration is confirmed in a burn block).

**`initiative`**: A flag that can be raised by any thread to wake up the main loop. The main loop calls `take_initiative()` to check and clear it. Used to signal events that require immediate attention without waiting for the next poll cycle.

**`estimated_winning_probs`** and **`previous_best_tips`**: Caches used by the mining system to track win probability estimates and previously selected chain tips across burn block heights.

## Relayer Thread

The relayer thread (`stacks-node/src/nakamoto_node/relayer.rs`) is the most complex thread in the system. It manages:

1. **Network result processing**: Handling blocks, transactions, and other data from the p2p thread
2. **Mining coordination**: Starting miner threads when this node wins a sortition
3. **Block commit submission**: Building and submitting `LeaderBlockCommitOp` transactions to Bitcoin
4. **VRF key registration**: Submitting key registration transactions

### Nakamoto RelayerDirective

The Nakamoto relayer processes these directives (`relayer.rs:93`):

```rust
pub enum RelayerDirective {
    HandleNetResult(NetworkResult),
    ProcessedBurnBlock(ConsensusHash, BurnchainHeaderHash, BlockHeaderHash),
    IssueBlockCommit(ConsensusHash, BlockHeaderHash),
    RegisterKey(BlockSnapshot),
    Exit,
}
```

- **`HandleNetResult`**: Sent by the p2p thread when new network data arrives. The relayer processes downloaded blocks, relays transactions, and updates the unconfirmed state.
- **`ProcessedBurnBlock`**: Sent by the main thread after processing a new sortition. The relayer checks if this node won and starts mining if so.
- **`IssueBlockCommit`**: Triggers a new block commit to Bitcoin, either after a new burn block or after a Nakamoto tenure's first block is processed.
- **`RegisterKey`**: Starts the VRF key registration process.
- **`Exit`**: Clean shutdown.

### Last Commit Tracking

The `LastCommit` struct (`relayer.rs:122`) tracks the most recently submitted block commit:

```rust
pub struct LastCommit {
    block_commit: LeaderBlockCommitOp,
    burn_tip: BlockSnapshot,
    stacks_tip: StacksBlockId,
    tenure_consensus_hash: ConsensusHash,
    start_block_hash: BlockHeaderHash,
    epoch_id: StacksEpochId,
    txid: Option<Txid>,
}
```

This is used for RBF (Replace-By-Fee) -- if the node needs to update its block commit before the previous one is confirmed, it can bump the fee and resubmit.

### Burn Block Commit Timer

The `BurnBlockCommitTimer` enum (`relayer.rs:143`) implements a timeout for issuing block commits:

```rust
enum BurnBlockCommitTimer {
    NotSet,
    Set { start_time: Instant, burn_tip: ConsensusHash },
}
```

When a new burn block arrives but no tenure change has happened yet, the timer starts. If it expires (the timeout is configurable), the relayer issues a block commit anyway. The timer resets when the burn tip changes.

## P2P Thread

The p2p thread (`stacks-node/src/nakamoto_node/peer.rs`) runs the `PeerNetwork` and HTTP RPC server:

1. Listens for incoming peer connections
2. Manages the peer graph (discovery, handshake, ping)
3. Downloads blocks from peers (block sync)
4. Serves the HTTP RPC API (block queries, mempool, StackerDB)
5. Relays blocks and transactions to peers
6. Forwards `NetworkResult` to the relayer thread

The p2p thread communicates with the relayer through the `relay_send` channel in `Globals`. When it has new network data (blocks downloaded, transactions received), it wraps them in `RelayerDirective::HandleNetResult` and sends them.

## Miner Thread

In Nakamoto mode, mining happens in short-lived threads spawned by the relayer (`stacks-node/src/nakamoto_node/miner.rs`). Each `BlockMinerThread`:

1. Is spawned when the relayer determines this node should mine
2. Assembles a single Nakamoto block from the mempool
3. Proposes it to signers via StackerDB
4. Waits for signer signatures
5. If approved, pushes the signed block to the network
6. Exits

The miner thread is blocked/unblocked via the `MinerStatus` in `Globals`. When the relayer needs exclusive access to chainstate (e.g., to process a new burn block), it blocks the miner by calling `globals.block_miner()`.

## StacksNode Spawn

The `StacksNode::spawn()` method (`stacks-node/src/nakamoto_node.rs:176`) creates the relayer and p2p threads:

```rust
pub fn spawn(
    runloop: &RunLoop,
    globals: Globals,
    relay_recv: Receiver<RelayerDirective>,
    data_from_neon: Option<Neon2NakaData>,
) -> StacksNode {
    // ... setup keychain, peer network, relayer ...

    let relayer_thread_handle = thread::Builder::new()
        .name(format!("relayer:{}", local_peer.port))
        .stack_size(BLOCK_PROCESSOR_STACK_SIZE)  // 32 MB
        .spawn(move || { relayer_thread.main(relay_recv); })
        .expect("FATAL: failed to start relayer thread");

    let p2p_thread_handle = thread::Builder::new()
        .stack_size(BLOCK_PROCESSOR_STACK_SIZE)
        .name(format!("p2p:({p2p_port},{rpc_port})"))
        .spawn(move || { p2p_thread.main(p2p_event_dispatcher); })
        .expect("FATAL: failed to start p2p thread");

    StacksNode {
        atlas_config,
        globals,
        is_miner,
        p2p_thread_handle,
        relayer_thread_handle,
    }
}
```

Both threads get 32 MB stacks (`BLOCK_PROCESSOR_STACK_SIZE`). The `StacksNode` struct holds the join handles for clean shutdown.

## Counters

The `Counters` struct (`run_loop/neon.rs:107`) provides atomic counters for monitoring and testing:

```rust
pub struct Counters {
    pub blocks_processed: RunLoopCounter,
    pub microblocks_processed: RunLoopCounter,
    pub missed_tenures: RunLoopCounter,
    pub cancelled_commits: RunLoopCounter,
    pub sortitions_processed: RunLoopCounter,
    pub naka_submitted_vrfs: RunLoopCounter,
    pub neon_submitted_commits: RunLoopCounter,
    pub naka_submitted_commits: RunLoopCounter,
    pub naka_mined_blocks: RunLoopCounter,
    pub naka_rejected_blocks: RunLoopCounter,
    pub naka_proposed_blocks: RunLoopCounter,
    pub naka_mined_tenures: RunLoopCounter,
    pub naka_signer_pushed_blocks: RunLoopCounter,
    // ... more counters
}
```

In test builds, `inc()` and `set()` atomically update the counters. In release builds, these are no-ops -- the counters exist but are never updated, minimizing overhead. The `Counters` struct is cloned cheaply (it's all `Arc` internally) and shared across threads.

## Signal Handling and Shutdown

The shutdown sequence:

1. An external signal (SIGINT/SIGTERM) or internal error sets `should_keep_running` to `false`
2. The main loop detects this and sends `RelayerDirective::Exit` to the relayer
3. The relayer exits its main loop
4. The coordinator channels are dropped, causing the coordinator to exit
5. The main loop joins all thread handles
6. The process exits

The `Globals::signal_stop()` method (line 223) performs step 1:

```rust
pub fn signal_stop(&self) {
    self.should_keep_running.store(false, Ordering::SeqCst);
}
```

All `Ordering::SeqCst` operations ensure that the stop signal is immediately visible across all threads.

## Data Flow: Block Proposal to Confirmation

Tracing a Nakamoto block from proposal to confirmation through the thread architecture:

1. **P2P thread** receives a new StackerDB chunk from the miner (block proposal)
2. **P2P thread** wraps it in `HandleNetResult` and sends to relayer via `relay_send`
3. **Relayer** processes the network result, updating StackerDB state
4. **Event dispatcher** fires the `stackerdb_chunks` event to the signer (via HTTP POST)
5. **Signer** (separate process) validates and signs, posting response back to StackerDB
6. **P2P thread** picks up the StackerDB update with the signer response
7. **P2P thread** sends another `HandleNetResult` to the relayer
8. **Relayer** detects the signed block and processes it
9. **Coordinator** is notified to process the new block into chainstate

## How It Connects

- **Chapter 29 (Node Startup)**: Describes how these threads are created during the boot sequence.
- **Chapter 31 (Bitcoin Controller)**: The `BitcoinRegtestController` used by the relayer for block commits.
- **Chapter 22 (P2P Protocol)**: The `PeerNetwork` run by the p2p thread.
- **Chapter 20 (Coordinator)**: The coordinator thread's block processing logic.
- **Chapter 19 (Nakamoto Mining)**: The miner thread spawned by the relayer.

## Edge Cases and Gotchas

1. **Relayer buffer of 1**: The `RELAYER_MAX_BUFFER = 1` means the `relay_send` channel can only buffer a single directive. If the relayer is busy processing, senders will block. This is intentional backpressure to prevent the relayer from falling behind.

2. **Miner blocking is reference-counted**: `MinerStatus` uses `add_blocked()` / `remove_blocked()` rather than a simple boolean. Multiple subsystems can independently block the miner, and it only unblocks when all blockers are removed.

3. **Unconfirmed tx sharing**: The `unconfirmed_txs` mutex is the mechanism by which the relayer shares transaction state with the p2p thread. The relayer calls `send_unconfirmed_txs()` and the p2p thread calls `recv_unconfirmed_txs()`. This avoids the p2p thread needing to do disk I/O to serve unconfirmed state.

4. **Initiative flag**: The `initiative` mutex (`globals.rs:79`) provides a way for any thread to wake up the main loop without sending a channel message. It stores a `String` indicating who raised it, which is useful for debugging why the main loop woke up.

5. **32 MB thread stacks**: Both the relayer and p2p threads use `BLOCK_PROCESSOR_STACK_SIZE = 32 * 1024 * 1024` bytes. This is much larger than the default Rust stack size (typically 8 MB) and is necessary because block processing can involve deep recursive calls through the MARF and Clarity VM.

6. **Globals is generic over T**: The `Globals<T>` struct is parameterized by the relayer directive type. The Neon node uses `RelayerDirective` from `globals.rs` while the Nakamoto node uses `nakamoto_node::relayer::RelayerDirective`. This allows both node types to share the same `Globals` infrastructure with different message types.
