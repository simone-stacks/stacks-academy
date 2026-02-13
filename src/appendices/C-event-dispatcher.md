# Appendix C: Event Dispatcher (Webhook System)

The Stacks node includes a built-in webhook system that notifies external services about blockchain state changes in real time. This appendix documents the architecture, event types, configuration, and consumption patterns for the EventDispatcher.

---

## Architecture Overview

The event system consists of three layers:

1. **Event Sources** -- Chain processing code that generates events (coordinator, mempool, miner, StackerDB)
2. **EventDispatcher** -- Central routing hub that matches events to subscribed observers
3. **EventObservers** -- External HTTP servers that receive JSON payloads via POST requests

```
Chain Processing        EventDispatcher          External Services
+---------------+      +----------------+       +-----------------+
| Coordinator   |----->|                |------>| API Service     |
| Mempool       |----->| Route by       |------>| Signer          |
| Miner         |----->| EventKeyType   |------>| Indexer          |
| StackerDB     |----->|                |------>| Custom Observer  |
+---------------+      +----------------+       +-----------------+
                              |
                        SQLite Queue
                        (pending_payloads)
```

**Source:** `stacks-node/src/event_dispatcher.rs:182` (`EventDispatcher` struct)

---

## The EventDispatcher Struct

```rust
// stacks-node/src/event_dispatcher.rs:182
pub struct EventDispatcher {
    registered_observers: Vec<EventObserver>,
    contract_events_observers_lookup: HashMap<(QualifiedContractIdentifier, String), HashSet<u16>>,
    assets_observers_lookup: HashMap<AssetIdentifier, HashSet<u16>>,
    burn_block_observers_lookup: HashSet<u16>,
    mempool_observers_lookup: HashSet<u16>,
    microblock_observers_lookup: HashSet<u16>,
    stx_observers_lookup: HashSet<u16>,
    any_event_observers_lookup: HashSet<u16>,
    miner_observers_lookup: HashSet<u16>,
    mined_microblocks_observers_lookup: HashSet<u16>,
    stackerdb_observers_lookup: HashSet<u16>,
    block_proposal_observers_lookup: HashSet<u16>,
    pub stackerdb_channel: Arc<Mutex<StackerDBChannel>>,
    db_path: PathBuf,
}
```

The dispatcher uses an index-based lookup design. Each `EventObserver` is stored in `registered_observers` and assigned a `u16` index. The various `*_lookup` fields map event categories to sets of observer indices. This avoids iterating all observers for every event -- only matching observers are notified.

---

## Observer Registration

Observers are registered at startup from the `[[events_observer]]` config entries. The `register_observer()` method (line 963) creates an `EventObserver` and populates the lookup tables based on the observer's `events_keys`:

```rust
// stacks-node/src/event_dispatcher.rs:984
for event_key_type in conf.events_keys.iter() {
    match event_key_type {
        EventKeyType::SmartContractEvent(event_key) => {
            self.contract_events_observers_lookup
                .entry(event_key.clone())
                .or_default()
                .insert(observer_index);
        }
        EventKeyType::BurnchainBlocks => {
            self.burn_block_observers_lookup.insert(observer_index);
        }
        EventKeyType::AnyEvent => {
            self.any_event_observers_lookup.insert(observer_index);
        }
        // ... other event types
    }
}
```

Each `EventKeyType` variant maps to a specific lookup table. The `AnyEvent` type (`"*"` in config) subscribes to all events.

---

## Event Types and Webhook Paths

When an event occurs, the dispatcher serializes it as JSON and POSTs it to each matching observer at a specific URL path. The paths are defined as constants at `stacks-node/src/event_dispatcher.rs:147-157`:

### `new_block` (PATH: `/new_block`)

Fired when a new Stacks block is processed and accepted into the chain state. This is the most commonly consumed event.

**Trigger:** `BlockEventDispatcher::announce_block()` (line 349), called by the coordinator after processing a block.

**Payload construction:** `make_new_block_processed_payload()` at `stacks-node/src/event_dispatcher/payloads.rs:322`

The full JSON payload contains every field needed to reconstruct a block's effects:

```json
{
  "block_hash": "0x...",
  "block_height": 12345,
  "block_time": 1700000000,
  "burn_block_hash": "0x...",
  "burn_block_height": 800000,
  "burn_block_time": 1700000000,
  "miner_txid": "0x...",
  "index_block_hash": "0x...",
  "parent_block_hash": "0x...",
  "parent_index_block_hash": "0x...",
  "parent_microblock": "0x...",
  "parent_microblock_sequence": 0,
  "parent_burn_block_hash": "0x...",
  "parent_burn_block_height": 799999,
  "parent_burn_block_timestamp": 1700000000,
  "anchored_cost": { "write_length": 0, "write_count": 0, "read_length": 0, "read_count": 0, "runtime": 0 },
  "confirmed_microblocks_cost": { ... },
  "pox_v1_unlock_height": 0,
  "pox_v2_unlock_height": 0,
  "pox_v3_unlock_height": 0,
  "tenure_height": 100,
  "consensus_hash": "0x...",
  "signer_bitvec": "...",
  "reward_set": { "rewarded_addresses": [...], "signers": [...], "pox_ustx_threshold": "..." },
  "cycle_number": 42,
  "matured_miner_rewards": [ ... ],
  "events": [ ... ],
  "transactions": [ ... ]
}
```

For Nakamoto blocks, three additional fields are appended (`payloads.rs:410-423`):

- `signer_signature_hash` -- The hash that signers approved
- `miner_signature` -- The miner's block signature
- `signer_signature` -- Array of all signer signatures on the block

Each transaction in the `transactions` array is a `TransactionEventPayload` (`payloads.rs:161`) containing:

```json
{
  "txid": "0x...",
  "tx_index": 0,
  "status": "success",
  "raw_result": "0x0703...",
  "raw_tx": "0x...",
  "contract_interface": null,
  "burnchain_op": null,
  "execution_cost": { "write_length": 0, "read_count": 0, ... },
  "microblock_sequence": null,
  "microblock_hash": null,
  "microblock_parent_hash": null,
  "vm_error": null
}
```

The `status` field has three possible values: `"success"` for committed transactions, `"abort_by_response"` when the contract returns `(err ...)`, and `"abort_by_post_condition"` when a post-condition check fails.

### `new_burn_block` (PATH: `/new_burn_block`)

Fired when a new Bitcoin block is processed.

**Trigger:** `BlockEventDispatcher::announce_burn_block()` (line 390).

**Payload structure:**
```json
{
  "burn_block_hash": "0x...",
  "burn_block_height": 800001,
  "reward_recipients": [
    { "recipient": "1A1zP1...", "amt": 50000 }
  ],
  "reward_slot_holders": ["1A1zP1..."],
  "burn_amount": 0,
  "consensus_hash": "0x...",
  "parent_burn_block_hash": "0x..."
}
```

### `new_mempool_tx` (PATH: `/new_mempool_tx`)

Fired when new transactions are added to the mempool.

**Trigger:** `EventDispatcher::process_new_mempool_txs()`.

**Payload:** Array of hex-encoded raw transaction bytes.

### `drop_mempool_tx` (PATH: `/drop_mempool_tx`)

Fired when transactions are removed from the mempool (replaced, expired, or pruned).

**Trigger:** `MemPoolEventDispatcher::mempool_txs_dropped()` (line 246).

**Payload structure:**
```json
{
  "dropped_txids": ["0x..."],
  "reason": "ReplaceByFee",
  "new_txid": "0x..."
}
```

### `new_microblocks` (PATH: `/new_microblocks`)

Fired when new microblocks are processed (pre-Nakamoto only).

**Trigger:** `EventDispatcher::process_new_microblocks()` (line 658).

### `mined_block` (PATH: `/mined_block`)

Fired when the local miner produces a new Stacks block (pre-Nakamoto).

**Trigger:** `MemPoolEventDispatcher::mined_block_event()` (line 257).

**Payload type:** `MinedBlockEvent` (`stacks-node/src/event_dispatcher/payloads.rs:51`)

### `mined_nakamoto_block` (PATH: `/mined_nakamoto_block`)

Fired when the local miner produces a Nakamoto block.

**Trigger:** `MemPoolEventDispatcher::mined_nakamoto_block_event()` (line 291).

**Payload type:** `MinedNakamotoBlockEvent` (`stacks-node/src/event_dispatcher/payloads.rs:71`)
```json
{
  "target_burn_height": 800001,
  "block_hash": "0x...",
  "block_id": "0x...",
  "parent_block_id": "0x...",
  "stacks_height": 12345,
  "block_size": 4096,
  "cost": { "write_length": ..., "read_count": ... },
  "miner_signature": "0x...",
  "signer_signature_hash": "0x...",
  "tx_events": [ ... ],
  "signer_bitvec": "...",
  "signer_signature": ["0x...", ...]
}
```

### `stackerdb_chunks` (PATH: `/stackerdb_chunks`)

Fired when StackerDB chunks are updated.

**Trigger:** `StackerDBEventDispatcher::new_stackerdb_chunks()` (line 339).

**Payload type:** `StackerDBChunksEvent` containing the contract ID and updated chunk data.

### `proposal_response` (PATH: `/proposal_response`)

Fired when a block proposal validation completes (used by signers).

**Trigger:** `ProposalCallbackHandler::notify_proposal_result()` (line 226).

**Payload type:** `BlockValidateResponse` -- either `BlockValidateOk` or `BlockValidateReject`.

### `attachments/new` (PATH: `/attachments/new`)

Fired when new Atlas attachments are processed.

---

## Dispatch Matrix

For block events containing transactions, the dispatcher builds a "dispatch matrix" to efficiently route individual transaction events to the correct observers. The `create_dispatch_matrix_and_event_vector()` method (`stacks-node/src/event_dispatcher.rs:475`) processes all transaction receipts and builds a vector of `HashSet<usize>` -- one set per observer -- containing the indices of events that observer is subscribed to.

The algorithm works as follows:

1. Initialize one empty `HashSet<usize>` per registered observer
2. Flatten all events from all transaction receipts into a single indexed vector
3. For each event, check its type and look up the corresponding observer set:
   - `SmartContractEvent` -- look up by `(contract_id, event_name)` in `contract_events_observers_lookup`
   - `STXTransferEvent`, `STXMintEvent`, `STXBurnEvent`, `STXLockEvent` -- look up in `stx_observers_lookup`
   - `NFTTransferEvent`, `NFTMintEvent`, `NFTBurnEvent` -- look up by `asset_identifier` in `assets_observers_lookup`
   - `FTTransferEvent`, `FTMintEvent`, `FTBurnEvent` -- look up by `asset_identifier` in `assets_observers_lookup`
4. Add the event index to matched observers' dispatch sets
5. Always add the event to `any_event` observers' sets

Each event in the flattened vector carries a tuple of `(committed: bool, txid: Txid, event: &StacksTransactionEvent)`. The `committed` flag reflects whether the transaction's post-conditions were satisfied -- this is `!receipt.post_condition_aborted`.

### Per-Observer Payload Construction

The `process_chain_tip()` method (`event_dispatcher.rs:568`) ties everything together. After building the dispatch matrix, it iterates over each observer and constructs a *per-observer* payload using only the events that observer is subscribed to:

```rust
// stacks-node/src/event_dispatcher.rs:619-643
for (observer_id, filtered_events_ids) in dispatch_matrix.iter().enumerate() {
    let filtered_events: Vec<_> = filtered_events_ids
        .iter()
        .map(|event_id| (*event_id, &events[*event_id]))
        .collect();

    let payload = make_new_block_processed_payload(
        filtered_events,   // only events this observer subscribed to
        block, metadata, receipts, parent_index_hash,
        &winner_txid, &mature_rewards, /* ... */
    );
    self.dispatch_to_observer(/* ... */);
}
```

This means each observer receives a complete block payload (with all transactions and metadata), but the `events` array is filtered to include only the specific contract/asset/STX events that observer subscribed to. An observer subscribed to `"*"` receives all events. An observer subscribed to `"stx"` receives only STX-related events.

### Mature Miner Rewards

The `matured_miner_rewards` field contains coinbase rewards that matured in this block. Each reward entry includes the recipient, miner address, coinbase amount, and fee breakdowns:

```json
{
  "recipient": "SP123...",
  "miner_address": "SP456...",
  "coinbase_amount": "500000000",
  "tx_fees_anchored": "1000",
  "tx_fees_streamed_confirmed": "500",
  "tx_fees_streamed_produced": "200",
  "from_stacks_block_hash": "0x...",
  "from_index_consensus_hash": "0x..."
}
```

All amounts are serialized as strings to avoid JSON integer overflow for large STX values.

---

## Delivery and Reliability

### HTTP Delivery

Events are delivered via HTTP POST with JSON payloads. The `dispatch_to_observer()` method (line 1121) handles delivery:

1. Serialize the payload to JSON bytes
2. Save to the SQLite pending queue
3. Send the HTTP POST request
4. On success (HTTP 200), delete from the queue
5. On failure, retry with exponential backoff (100ms initial, capped at 3x timeout)

```rust
// stacks-node/src/event_dispatcher.rs:1166-1218
let mut backoff = Duration::from_millis(100);
loop {
    match send_http_request(host, port, request, data.timeout) {
        Ok(response) if response.preamble().status_code == 200 => break,
        _ => {
            if disable_retries { return Err(...); }
            sleep(backoff);
            backoff = min(backoff * 2 + jitter, max_backoff);
        }
    }
}
```

### SQLite Pending Queue

The `EventDispatcherDbConnection` (`stacks-node/src/event_dispatcher/db.rs`) provides a SQLite-backed persistence layer for undelivered payloads:

```sql
CREATE TABLE pending_payloads (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    url TEXT NOT NULL,
    payload BLOB NOT NULL,
    timeout INTEGER NOT NULL,
    timestamp INTEGER NOT NULL
)
```

On startup, `process_pending_payloads()` (line 1051) re-attempts delivery of any payloads that were saved but not successfully delivered. This ensures that events survive node restarts.

### The `disable_retries` Option

When `disable_retries = true` in the observer config, the dispatcher makes exactly one delivery attempt. If it fails, the event is lost. This is useful for non-critical consumers but should never be used for observers that require complete event histories (like the Stacks API indexer).

---

## StackerDB Channel

The `StackerDBChannel` (`stacks-node/src/event_dispatcher/stacker_db.rs`) provides a special internal channel for routing StackerDB events to the miner's signer coordinator. Unlike the HTTP-based external observers, this channel uses in-process `Arc<Mutex<>>` communication for low-latency delivery of signer messages to the mining thread.

```rust
// stacks-node/src/event_dispatcher.rs:213
pub stackerdb_channel: Arc<Mutex<StackerDBChannel>>,
```

This is critical for Nakamoto mining, where the miner must receive signer responses (approvals/rejections) as quickly as possible.

---

## Trait Implementations

The `EventDispatcher` implements three traits that integrate it into the node's processing pipeline:

### `BlockEventDispatcher` (from stackslib)

```rust
// stacks-node/src/event_dispatcher.rs:348
impl BlockEventDispatcher for EventDispatcher {
    fn announce_block(&self, ...) { ... }
    fn announce_burn_block(&self, ...) { ... }
}
```

Called by the coordinator (`stackslib/src/chainstate/coordinator/`) after processing new blocks and burn blocks.

### `MemPoolEventDispatcher` (from stackslib)

```rust
// stacks-node/src/event_dispatcher.rs:245
impl MemPoolEventDispatcher for EventDispatcher {
    fn mempool_txs_dropped(&self, ...) { ... }
    fn mined_block_event(&self, ...) { ... }
    fn mined_microblock_event(&self, ...) { ... }
    fn mined_nakamoto_block_event(&self, ...) { ... }
    fn get_proposal_callback_receiver(&self) -> Option<Box<dyn ProposalCallbackReceiver>> { ... }
}
```

Called by the mempool and miner threads.

### `StackerDBEventDispatcher` (from stackslib)

```rust
// stacks-node/src/event_dispatcher.rs:337
impl StackerDBEventDispatcher for EventDispatcher {
    fn new_stackerdb_chunks(&self, ...) { ... }
}
```

Called by the StackerDB replication engine when new chunks arrive.

---

## Configuring Event Observers

### For the Stacks API (Full Indexer)

```toml
[[events_observer]]
endpoint = "localhost:3700"
events_keys = ["*"]
timeout_ms = 60000
```

### For a Signer Node

```toml
[[events_observer]]
endpoint = "127.0.0.1:30000"
events_keys = ["stackerdb", "block_proposal", "burn_blocks"]
```

### For Monitoring a Specific Contract

```toml
[[events_observer]]
endpoint = "localhost:8080"
events_keys = ["SP000000000000000000002Q6VF78.pox-4::handle-unlock"]
timeout_ms = 5000
```

### For STX Transfer Tracking

```toml
[[events_observer]]
endpoint = "localhost:9090"
events_keys = ["stx"]
```

### Via Environment Variable

```bash
STACKS_EVENT_OBSERVER=localhost:3700 stacks-node start --config node.toml
```

This registers a catch-all observer equivalent to `events_keys = ["*"]`.

---

## Building a Consumer

An event observer is any HTTP server that accepts POST requests with JSON bodies and returns HTTP 200. Here is a minimal example in pseudocode:

```
POST /new_block
  -> Parse JSON body
  -> Extract transactions, events, block metadata
  -> Store in your database
  -> Return 200 OK

POST /new_burn_block
  -> Parse burn block info
  -> Update burn chain tracking
  -> Return 200 OK

POST /new_mempool_tx
  -> Parse hex-encoded transactions
  -> Update pending transaction tracking
  -> Return 200 OK
```

Key considerations:

- **Return 200 quickly.** The node blocks the dispatching thread while waiting for your response. Long processing delays will delay event delivery to other observers.
- **Handle duplicates.** Events may be re-delivered after node restarts (from the pending queue). Use idempotent operations.
- **Monitor `/new_burn_block` for chain reorganizations.** When the burnchain reorgs, you may receive new blocks that replace previously delivered ones.
- **The `test_observer` module** (`stacks-node/src/tests/neon_integrations.rs:276`) provides a reference implementation using Warp that stores events in static `Mutex<Vec<>>` collections -- useful for understanding the expected payload formats.
