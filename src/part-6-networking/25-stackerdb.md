# Chapter 25: StackerDB

## Purpose

StackerDB is a best-effort replicated key-value store layered on top of the Stacks P2P network. It provides a way for off-chain applications -- most critically, the Stacks signer set -- to exchange data without incurring the cost and latency of on-chain transactions. Signers use StackerDB to coordinate block proposals, share DKG messages, and exchange votes.

Unlike on-chain data, StackerDB state is **not consensus-critical**. Nodes do not need it to validate the blockchain. It is **eventually consistent** -- in the absence of writes and network partitions, all replicas converge to the same state. It is **ephemeral** -- chunks can be evicted at the start of each reward cycle.

The StackerDB subsystem lives in `stackslib/src/net/stackerdb/` with the core data types in `libstackerdb/src/libstackerdb.rs`.

## Key Concepts

**A StackerDB is controlled by a smart contract.** Each StackerDB is identified by the `QualifiedContractIdentifier` of its controlling Clarity contract. This contract defines who can write to which slots, how many slots exist, the maximum chunk size, and other configuration parameters.

**Data is organized into slots.** A StackerDB is an array of fixed-size slots (up to 4096 slots). Each slot holds one chunk of data. Writes replace the entire chunk atomically.

**Writes are authenticated by Lamport clocks and signatures.** Each slot has a monotonically increasing version number (Lamport clock). To write, the writer must sign the (slot_id, slot_version, data_hash) tuple with the private key corresponding to the slot's assigned public key hash. Replicas accept a write only if the version is strictly higher than the current one.

**Replication uses a gossip-like protocol.** Nodes discover StackerDB replicas through the P2P handshake, exchange chunk inventories, then fetch or push chunks as needed.

## Architecture

```
Smart Contract (Clarity)       libstackerdb (crate)
  defines slot owners,           defines StackerDBChunkData,
  chunk size, write freq         SlotMetadata, signing
        |                              |
        v                              v
stackerdb/config.rs              stackerdb/db.rs
  reads config from                stores chunks
  smart contract                   in SQLite
        |                              |
        v                              v
stackerdb/mod.rs  <------>  stackerdb/sync.rs
  StackerDBConfig,              StackerDBSync state machine
  StackerDBs facade             (connect, inventory, fetch, push)
        |                              |
        v                              v
    PeerNetwork                   P2P Messages
  (orchestrates sync)          (StackerDBGetChunkInv,
                                StackerDBChunkInv,
                                StackerDBGetChunk,
                                StackerDBPushChunk)
```

## Key Data Structures

### StackerDBChunkData (libstackerdb)

Defined in `libstackerdb/src/libstackerdb.rs:86`:

```rust
pub struct StackerDBChunkData {
    pub slot_id: u32,
    pub slot_version: u32,
    pub sig: MessageSignature,
    pub data: Vec<u8>,
}
```

This is the fundamental unit of data in StackerDB. The `sig` field is a signature over the concatenation of `slot_id`, `slot_version`, and `SHA-512/256(data)`. The `STACKERDB_MAX_CHUNK_SIZE` is 16 MB (`libstackerdb.rs:37`), and the signer-specific chunk size is 2 MB (`SIGNERS_STACKERDB_CHUNK_SIZE`, libstackerdb.rs:39).

### SlotMetadata (libstackerdb)

Defined in `libstackerdb/src/libstackerdb.rs:73`:

```rust
pub struct SlotMetadata {
    pub slot_id: u32,
    pub slot_version: u32,
    pub data_hash: Sha512Trunc256Sum,
    pub signature: MessageSignature,
}
```

This is derived from a `StackerDBChunkData` and stored separately in the database. It allows verifying chunk integrity without loading the full chunk data.

### StackerDBConfig

Defined in `stackslib/src/net/stackerdb/mod.rs:171`:

```rust
pub struct StackerDBConfig {
    pub chunk_size: u64,
    pub signers: Vec<(StacksAddress, u32)>,
    pub write_freq: u64,
    pub max_writes: u32,
    pub hint_replicas: Vec<NeighborAddress>,
    pub max_neighbors: usize,
}
```

Fields:
- `chunk_size`: Maximum byte size of a single chunk
- `signers`: List of (address, slot_count) pairs. Each signer owns `slot_count` consecutive slots
- `write_freq`: Minimum seconds between writes to the same slot
- `max_writes`: Maximum writes per slot per reward cycle
- `hint_replicas`: Seed peers that replicate this DB
- `max_neighbors`: How many replicas to connect to for sync

The `num_slots()` method (`mod.rs:201`) computes the total by summing all signers' slot counts. The `signer_ranges()` method returns the slot index range for each signer.

### StackerDBs

Defined in `stackslib/src/net/stackerdb/mod.rs:225`:

```rust
pub struct StackerDBs {
    conn: DBConn,
    path: String,
}
```

This is the facade for all StackerDB storage. It wraps a SQLite database and provides methods to:
- Create or reconfigure StackerDB replicas (`create_or_reconfigure_stackerdbs`, mod.rs:278)
- Read chunks and metadata
- Write chunks via `StackerDBTx` (a transactional wrapper)

The `.miners` contract StackerDB is handled specially -- its config is generated directly by `NakamotoChainState::make_miners_stackerdb_config()` rather than by executing the smart contract.

## The Control Plane: Smart Contract Interface

Every StackerDB controlling contract must implement two functions, defined in the trait documented in `stackslib/src/net/stackerdb/config.rs:20-37`:

```clarity
;; Get the list of signers and their slot counts
(define-public (stackerdb-get-signer-slots)
    (response (list 4096 { signer: principal, num-slots: uint }) uint))

;; Get the DB configuration
(define-public (stackerdb-get-config)
    (response {
        chunk-size: uint,
        write-freq: uint,
        max-writes: uint,
        max-neighbors: uint,
        hint-replicas: (list 128 {
            addr: (list 16 uint),
            port: uint,
            public-key-hash: (buff 20)
        })
    } uint))
```

The node calls these functions once per reward cycle to configure the local replica. `REQUIRED_FUNCTIONS` in `config.rs:65` validates that a contract conforms to the expected type signatures before it is accepted as a StackerDB controller.

The maximum number of signers times slots is `STACKERDB_INV_MAX = 4096` (`mod.rs:142`), and a contract can have at most `STACKERDB_MAX_PAGE_COUNT = 2` pages of 4096 entries each (`mod.rs:146`).

## The Sync Protocol

### Discovery

StackerDB discovery piggybacks on the P2P handshake. When two peers complete a handshake and both advertise `ServiceFlags::STACKERDB` (0x04), the response is a `StackerDBHandshakeAccept` instead of a plain `HandshakeAccept`:

```rust
StackerDBHandshakeAccept(HandshakeAcceptData, StackerDBHandshakeData)
```

The `StackerDBHandshakeData` (`stackslib/src/net/mod.rs:1083`) contains:

```rust
pub struct StackerDBHandshakeData {
    pub rc_consensus_hash: ConsensusHash,
    pub smart_contracts: Vec<QualifiedContractIdentifier>,
}
```

This tells the receiving node which StackerDB contracts the peer replicates. The peer stores this information in the PeerDB so the sync state machine can find replicas later.

### StackerDBSyncState

The sync state machine (`stackslib/src/net/stackerdb/mod.rs:375`):

```rust
pub enum StackerDBSyncState {
    ConnectBegin,
    ConnectFinish,
    GetChunksInvBegin,
    GetChunksInvFinish,
    GetChunks,
    PushChunks,
    Finished,
}
```

### StackerDBSync

The sync engine (`stackslib/src/net/stackerdb/mod.rs:386`):

```rust
pub struct StackerDBSync<NC: NeighborComms> {
    state: StackerDBSyncState,
    pub rc_consensus_hash: Option<ConsensusHash>,
    pub smart_contract_id: QualifiedContractIdentifier,
    pub num_slots: usize,
    pub write_freq: u64,
    pub chunk_invs: HashMap<NeighborAddress, StackerDBChunkInvData>,
    pub chunk_fetch_priorities: Vec<(StackerDBGetChunkData, Vec<NeighborAddress>)>,
    pub chunk_push_priorities: Vec<(StackerDBPushChunkData, Vec<NeighborAddress>)>,
    pub expected_versions: Vec<u32>,
    pub downloaded_chunks: HashMap<NeighborAddress, Vec<StackerDBChunkData>>,
    pub(crate) replicas: HashSet<NeighborAddress>,
    pub(crate) connected_replicas: HashSet<NeighborAddress>,
    pub comms: NC,
    pub stackerdbs: StackerDBs,
    // ...
}
```

The `PeerNetwork` maintains one `StackerDBSync` per replicated contract in `stacker_db_syncs`.

### Sync Flow

**Step 1: Connect** (`ConnectBegin` / `ConnectFinish`):
Find replicas from the PeerDB using `find_qualified_replicas()` (`sync.rs:85`). Filter by age and network reachability. Connect to up to `MAX_DB_NEIGHBORS = 32` replicas.

**Step 2: Inventory Exchange** (`GetChunksInvBegin` / `GetChunksInvFinish`):
Send `StackerDBGetChunkInv` messages to connected replicas. Each replica responds with `StackerDBChunkInvData`:

```rust
pub struct StackerDBChunkInvData {
    pub slot_versions: Vec<u32>,        // version of each slot
    pub num_outbound_replicas: u32,     // how many replicas this peer connects to
}
```

The `slot_versions` vector is a version vector -- each entry is the Lamport clock for that slot. By comparing the remote version vector with the local one, the node determines which chunks are newer remotely and which are newer locally.

**Step 3: Fetch Chunks** (`GetChunks`):
For each slot where the remote version exceeds the local version, the node sends a `StackerDBGetChunk` request:

```rust
pub struct StackerDBGetChunkData {
    pub contract_id: QualifiedContractIdentifier,
    pub rc_consensus_hash: ConsensusHash,
    pub slot_id: u32,
    pub slot_version: u32,
}
```

Chunks are prioritized **newest-first, then rarest-first** to ensure the latest, least-replicated data propagates quickest. Up to `MAX_CHUNKS_IN_FLIGHT = 6` concurrent requests are allowed (`sync.rs:38`).

**Step 4: Push Chunks** (`PushChunks`):
For each slot where the local version exceeds a replica's version, the node pushes the chunk via `StackerDBPushChunk`:

```rust
pub struct StackerDBPushChunkData {
    pub contract_id: QualifiedContractIdentifier,
    pub rc_consensus_hash: ConsensusHash,
    pub chunk_data: StackerDBChunkData,
}
```

The push phase ensures that locally-written chunks propagate to other replicas without waiting for them to poll.

**Step 5: Store** (`Finished`):
Downloaded chunks are validated (signature verification, version check) and stored locally. The sync result is captured in `StackerDBSyncResult`:

```rust
pub struct StackerDBSyncResult {
    pub contract_id: QualifiedContractIdentifier,
    pub chunk_invs: HashMap<NeighborAddress, StackerDBChunkInvData>,
    pub chunks_to_store: Vec<StackerDBChunkData>,
    pub(crate) stale: HashSet<NeighborAddress>,
    pub num_connections: u64,
    pub num_attempted_connections: u64,
}
```

## How Signers Use StackerDB

The Stacks signer system uses StackerDB as its primary communication channel. The signer set is divided into groups, each with a corresponding StackerDB contract (e.g., `.signers-0-0`, `.signers-0-1`, etc.). Each signer is assigned slots in these contracts based on their stacking weight.

Signers write to their assigned slots to:
- **Share DKG messages** during distributed key generation
- **Broadcast block proposals** for validation
- **Exchange signatures** to accumulate the threshold signature needed to approve a block

The `.miners` contract StackerDB (`MINERS_NAME` in `mod.rs:129`) is a special StackerDB that miners use to communicate block proposals to signers. It has `MINER_SLOT_COUNT = 2` slots per miner (`mod.rs:150`).

## HTTP API Integration

StackerDB data is accessible via the HTTP API as well as the P2P protocol:

- `GET /v2/stackerdb/{addr}/{name}/chunks` -- Read a chunk
- `GET /v2/stackerdb/{addr}/{name}/metadata` -- Get slot metadata (versions)
- `POST /v2/stackerdb/{addr}/{name}/chunks` -- Write a chunk

The `poststackerdbchunk.rs` handler validates the signature, checks rate limits, and stores the chunk. It also generates a `StackerDBPushChunkData` message that the relay system will broadcast to P2P peers.

## How It Connects

- **Chapter 22 (P2P Protocol)**: StackerDB extends the handshake with `StackerDBHandshakeAccept` and adds six message types (`StackerDBGetChunkInv`, `StackerDBChunkInv`, `StackerDBGetChunk`, `StackerDBChunk`, `StackerDBPushChunk`, `StackerDBHandshakeAccept`).
- **Chapter 24 (RPC API)**: External clients (signers, miners) write chunks via HTTP POST, and read them via HTTP GET.
- **Chapter 26 (Relay and Mempool)**: The relay layer processes `NetworkResult::pushed_stackerdb_chunks` and `uploaded_stackerdb_chunks`.
- **Chapter 27 (Signer)**: Signers are the primary consumers of StackerDB, using it for DKG, block proposal validation, and signature accumulation.

## Edge Cases and Gotchas

1. **Stale view handling**: If a peer's `rc_consensus_hash` does not match the local one, the sync machine marks it as "stale" (using `NackErrorCodes::StaleView`) and tracks it separately. Stale peers are retried on subsequent reward cycle transitions.

2. **Rate limiting**: The `write_freq` config parameter enforces a minimum wall-clock interval between writes to the same slot. If a writer tries to write too frequently, the node responds with `Error::TooFrequentSlotWrites`. Similarly, `max_writes` caps total writes per slot per reward cycle.

3. **Signature verification is mandatory**: Every chunk write must be signed by the key corresponding to the slot's assigned signer. The `SlotMetadata` hash covers `(slot_id, slot_version, SHA-512/256(data))`. An invalid signature results in `Error::BadSlotSigner`.

4. **Chunk size enforcement**: If a chunk exceeds `chunk_size` bytes, it is rejected with `Error::StackerDBChunkTooBig`. The default maximum is `STACKERDB_MAX_CHUNK_SIZE = 16 MB`.

5. **Ephemeral by design**: StackerDB state is cleared at reward cycle boundaries when the controlling contract's `stackerdb-get-signer-slots` returns a different signer set. This means applications must re-replicate data if they want persistence beyond a reward cycle.

6. **No partial updates**: Writing to a slot replaces the entire chunk. If you need to update a small portion of a large chunk, you must read the whole chunk, modify it, and write the whole thing back. This is a design trade-off for simplicity.

7. **MAX_DB_NEIGHBORS = 32**: Each StackerDB sync instance connects to at most 32 replicas (`sync.rs:39`). For the signer StackerDB contracts (which all active signers replicate), this is typically sufficient. For niche contracts with few replicas, the system degrades gracefully.
