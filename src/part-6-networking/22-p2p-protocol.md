# Chapter 22: P2P Protocol

## Purpose

The Stacks P2P protocol is the backbone of the decentralized network. It allows nodes to discover each other, authenticate identities through cryptographic handshakes, and exchange structured messages about blocks, transactions, and inventories. Without it, there is no consensus, no block propagation, and no decentralization.

The P2P layer lives in `stackslib/src/net/` and is organized into several cooperating subsystems: the wire protocol (message serialization), the conversation layer (per-peer session management), the peer network (the orchestrator), and the neighbor walk (peer discovery).

## Key Concepts

**Every peer is identified by a `NeighborKey`**, which combines an IP address, port, network ID, and protocol version. The network maintains two types of connections: **outbound** (we initiated) and **inbound** (they connected to us). The protocol limits both to prevent resource exhaustion.

**All communication is message-based.** Every message is a `StacksMessage` consisting of a `Preamble` (header with metadata and a cryptographic signature), a relay chain (to track message provenance), and a typed payload (`StacksMessageType`).

**Authentication happens via handshakes.** Before two peers can exchange meaningful data, they must complete a handshake that establishes each other's public keys, services, and chain state. Handshakes also double as heartbeats to keep connections fresh.

## Architecture

The P2P subsystem has a layered design:

1. **Wire Protocol** (`codec.rs`, `mod.rs`): Defines how messages are serialized/deserialized using the `StacksMessageCodec` trait. Every message type has a numeric ID and a deterministic binary encoding.

2. **Connection** (`connection.rs`): Manages raw TCP socket I/O, buffering, inbox/outbox queues, and protocol framing. `ConnectionP2P` handles the low-level read/write state machine.

3. **Conversation** (`chat.rs`): `ConversationP2P` wraps a `ConnectionP2P` and implements the session layer -- handshake negotiation, heartbeats, message validation, and per-peer statistics tracking.

4. **Peer Network** (`p2p.rs`): `PeerNetwork` is the top-level orchestrator. It manages all active conversations, the neighbor walk, inventory sync, block downloads, and the event loop that drives everything forward.

5. **Neighbor Discovery** (`neighbors/`): The `NeighborWalk` state machine uses a randomized graph walk (Metropolis-Hastings) to discover new peers and maintain a diverse frontier.

## Key Data Structures

### Preamble

Defined in `stackslib/src/net/mod.rs:846`:

```rust
pub struct Preamble {
    pub peer_version: u32,                           // software version
    pub network_id: u32,                             // mainnet, testnet, etc.
    pub seq: u32,                                    // message sequence number
    pub burn_block_height: u64,                      // last-seen burn block height
    pub burn_block_hash: BurnchainHeaderHash,        // hash of the last-seen burn block
    pub burn_stable_block_height: u64,               // stable height (tip - 7)
    pub burn_stable_block_hash: BurnchainHeaderHash, // stable burn block hash
    pub additional_data: u32,                        // RESERVED
    pub signature: MessageSignature,                 // signature from sender
    pub payload_len: u32,                            // length of the payload
}
```

Every P2P message begins with this preamble. It pins the message to a specific view of the burnchain, which lets nodes detect stale peers and reject messages from incompatible forks. The `signature` field provides authentication -- the preamble is signed with the sender's session key.

### StacksMessage

Defined in `stackslib/src/net/mod.rs:1217`:

```rust
pub struct StacksMessage {
    pub preamble: Preamble,
    pub relayers: Vec<RelayData>,
    pub payload: StacksMessageType,
}
```

The `relayers` vector records which peers have forwarded this message, enabling relay-loop detection and network topology analysis.

### StacksMessageType

Defined in `stackslib/src/net/mod.rs:1144`, this enum captures every possible P2P message:

```rust
pub enum StacksMessageType {
    Handshake(HandshakeData),
    HandshakeAccept(HandshakeAcceptData),
    HandshakeReject,
    GetNeighbors,
    Neighbors(NeighborsData),
    GetBlocksInv(GetBlocksInv),
    BlocksInv(BlocksInvData),
    GetPoxInv(GetPoxInv),
    PoxInv(PoxInvData),
    BlocksAvailable(BlocksAvailableData),
    MicroblocksAvailable(BlocksAvailableData),
    Blocks(BlocksData),
    Microblocks(MicroblocksData),
    Transaction(StacksTransaction),
    Nack(NackData),
    Ping(PingData),
    Pong(PongData),
    NatPunchRequest(u32),
    NatPunchReply(NatPunchData),
    StackerDBHandshakeAccept(HandshakeAcceptData, StackerDBHandshakeData),
    StackerDBGetChunkInv(StackerDBGetChunkInvData),
    StackerDBChunkInv(StackerDBChunkInvData),
    StackerDBGetChunk(StackerDBGetChunkData),
    StackerDBChunk(StackerDBChunkData),
    StackerDBPushChunk(StackerDBPushChunkData),
    GetNakamotoInv(GetNakamotoInvData),
    NakamotoInv(NakamotoInvData),
    NakamotoBlocks(NakamotoBlocksData),
}
```

Each variant has a corresponding numeric `StacksMessageID` (0-28) used on the wire. The protocol evolved over time: `StackerDB*` messages were added for signer communication, and `GetNakamotoInv`/`NakamotoInv`/`NakamotoBlocks` for the Nakamoto epoch.

### HandshakeData

Defined in `stackslib/src/net/mod.rs:1015`:

```rust
pub struct HandshakeData {
    pub addrbytes: PeerAddress,
    pub port: u16,
    pub services: u16,                      // bit field: RELAY=0x01, RPC=0x02, STACKERDB=0x04
    pub node_public_key: StacksPublicKeyBuffer,
    pub expire_block_height: u64,           // when this key expires
    pub data_url: UrlString,                // HTTP API endpoint
}
```

The `services` bit field advertises capabilities. `ServiceFlags::RELAY` (0x01) means the peer relays messages, `ServiceFlags::RPC` (0x02) means it serves HTTP API requests, and `ServiceFlags::STACKERDB` (0x04) means it participates in StackerDB replication.

### NeighborKey and Neighbor

`NeighborKey` (`stackslib/src/net/mod.rs:1338`) identifies a peer:

```rust
pub struct NeighborKey {
    pub peer_version: u32,
    pub network_id: u32,
    pub addrbytes: PeerAddress,
    pub port: u16,
}
```

`Neighbor` (`stackslib/src/net/mod.rs:1424`) adds mutable runtime state:

```rust
pub struct Neighbor {
    pub addr: NeighborKey,
    pub public_key: Secp256k1PublicKey,
    pub expire_block: u64,
    pub last_contact_time: u64,
    pub allowed: i64,        // allow deadline (negative = forever)
    pub denied: i64,         // deny deadline (negative = forever)
    pub asn: u32,            // AS number
    pub org: u32,            // organization identifier
    pub in_degree: u32,      // peers who list us as neighbor
    pub out_degree: u32,     // neighbors this peer has
}
```

The `in_degree` and `out_degree` fields are used by the Metropolis-Hastings random walk to ensure the peer graph is sampled uniformly.

### ConversationP2P

Defined in `stackslib/src/net/chat.rs:315`:

```rust
pub struct ConversationP2P {
    pub instantiated: u64,
    pub network_id: u32,
    pub version: u32,
    pub connection: ConnectionP2P,
    pub conn_id: usize,
    pub burnchain: Burnchain,
    pub heartbeat: u32,
    pub peer_network_id: u32,
    pub peer_version: u32,
    pub peer_services: u16,
    pub peer_addrbytes: PeerAddress,
    pub peer_port: u16,
    pub handshake_addrbytes: PeerAddress,
    pub handshake_port: u16,
    pub peer_heartbeat: u32,
    pub peer_expire_block_height: u64,
    pub data_url: UrlString,
    pub burnchain_tip_height: u64,
    pub burnchain_tip_burn_header_hash: BurnchainHeaderHash,
    pub stats: NeighborStats,
    pub db_smart_contracts: Vec<QualifiedContractIdentifier>,
    pub reply_handles: VecDeque<ReplyHandleP2P>,
    // ...
}
```

This is the session object for a single peer-to-peer connection. It tracks everything known about the remote peer -- from its burnchain view to its StackerDB contracts to detailed traffic statistics.

### PeerNetwork

Defined in `stackslib/src/net/p2p.rs:457`:

```rust
pub struct PeerNetwork {
    pub peer_version: u32,
    pub epochs: EpochList,
    pub local_peer: LocalPeer,
    pub chain_view: BurnchainView,
    pub burnchain_tip: BlockSnapshot,
    pub stacks_tip: StacksTipInfo,
    pub peerdb: PeerDB,
    pub atlasdb: AtlasDB,
    pub burnchain_db: BurnchainDB,
    pub peers: PeerMap,                     // HashMap<usize, ConversationP2P>
    pub sockets: HashMap<usize, TcpStream>,
    pub events: HashMap<NeighborKey, usize>,
    pub walk: Option<NeighborWalk<...>>,
    pub inv_state: Option<InvState>,
    pub inv_state_nakamoto: Option<NakamotoInvStateMachine<...>>,
    pub block_downloader: Option<BlockDownloader>,
    pub block_downloader_nakamoto: Option<NakamotoDownloadStateMachine>,
    pub stacker_db_syncs: Option<HashMap<QualifiedContractIdentifier, StackerDBSync<...>>>,
    pub stackerdbs: StackerDBs,
    pub http: Option<HttpPeer>,
    pub work_state: PeerNetworkWorkState,
    // ...
}
```

`PeerNetwork` is the god object of the networking layer. It owns all connections, all sub-state-machines (inventory, download, StackerDB sync, neighbor walk), and both the P2P and HTTP server handles.

## Code Walkthrough: The Handshake Flow

When peer A connects to peer B, the handshake proceeds as follows:

**Step 1: A sends a `Handshake` message** containing its `HandshakeData` -- public key, services, data URL, and key expiration.

**Step 2: B receives and validates the handshake** in `ConversationP2P::handle_handshake()` (`stackslib/src/net/chat.rs:1199`):

```rust
fn handle_handshake(
    &mut self,
    network: &mut PeerNetwork,
    message: &mut StacksMessage,
    authenticated: bool,
    ibd: bool,
) -> Result<(Option<StacksMessage>, bool), net_error> {
```

The handler first calls `validate_handshake()` to check version compatibility and chain view. If invalid, it returns a `HandshakeReject`. Otherwise, it calls `update_from_handshake_data()` to update the conversation state with the peer's public key and services.

**Step 3: B persists the peer** via `Neighbor::load_and_update()` and `neighbor.save_update()` if the key has changed (chat.rs:1262-1271).

**Step 4: B decides the response type.** If both peers advertise `ServiceFlags::STACKERDB`, the response is a `StackerDBHandshakeAccept` (which includes the list of replicated StackerDB contracts). Otherwise, it's a plain `HandshakeAccept` (chat.rs:1283-1302).

**Step 5: A receives the `HandshakeAccept`** in `handle_handshake_accept()` (chat.rs:1323), which updates its local state with B's key, heartbeat interval, and StackerDB list.

After the handshake completes, both peers have authenticated each other and know what services the other offers. Handshakes are also re-sent periodically as heartbeats to keep keys fresh.

## Code Walkthrough: The Main Event Loop

The network runs via `PeerNetwork::run()` (`stackslib/src/net/p2p.rs:5443`):

```rust
pub fn run<B: BurnchainHeaderReader>(
    &mut self,
    indexer: &B,
    sortdb: &SortitionDB,
    chainstate: &mut StacksChainState,
    mempool: &mut MemPoolDB,
    dns_client_opt: Option<&mut DNSClient>,
    download_backpressure: bool,
    ibd: bool,
    poll_timeout: u64,
    handler_args: &RPCHandlerArgs,
    txindex: bool,
) -> Result<NetworkResult, net_error> {
```

Each call to `run()` is one tick of the network state machine:

1. **Poll sockets** using `mio` (epoll/kqueue) to detect which connections have data ready.
2. **Refresh local peer state** from the PeerDB in case the key has rotated.
3. **Refresh burnchain view** from the SortitionDB to stay current with the chain.
4. **Process HTTP connections** -- parse requests, dispatch to RPC handlers, send responses.
5. **Dispatch P2P network** via `dispatch_network()` -- this drives the `PeerNetworkWorkState` state machine through its phases: `GetPublicIP` -> `BlockInvSync` -> `BlockDownload` -> `AntiEntropy` -> `Prune`.

The result is a `NetworkResult` (`stackslib/src/net/mod.rs:1492`) that aggregates all blocks, transactions, and StackerDB chunks discovered during this tick. The caller (the Relayer thread) then processes these into the chainstate.

## Neighbor Discovery: The Random Walk

Peer discovery uses a Metropolis-Hastings random walk, implemented in `stackslib/src/net/neighbors/walk.rs`. The walk state machine (`NeighborWalkState`) cycles through these phases:

```rust
pub enum NeighborWalkState {
    HandshakeBegin,
    HandshakeFinish,
    GetNeighborsBegin,
    GetNeighborsFinish,
    GetHandshakesBegin,
    GetHandshakesFinish,
    GetNeighborsNeighborsBegin,
    GetNeighborsNeighborsFinish,
    PingbackHandshakesBegin,
    PingbackHandshakesFinish,
    ReplacedNeighborsPingBegin,
    ReplacedNeighborsPingFinish,
    Finished,
}
```

The algorithm:
1. Pick a random neighbor (or the previous walk target).
2. Handshake with it to confirm it is alive.
3. Ask it for its neighbors (`GetNeighbors`).
4. Handshake with each returned neighbor to verify them.
5. Ask each verified neighbor for *their* neighbors (to compute in-degree/out-degree).
6. Use the degree ratio to decide (probabilistically) whether to step to a new neighbor or stay put.
7. Try to ping back inbound peers we have not yet probed.

This ensures the peer frontier is sampled uniformly rather than being biased toward high-degree nodes.

## The Peer Database

The `PeerDB` (`stackslib/src/net/db.rs`) is a SQLite database that persists the known peer frontier. Key tables store:

- **Neighbors**: IP addresses, ports, public keys, expiration times, ASN/org info, and allow/deny lists.
- **StackerDB Replicas**: Which peers replicate which StackerDB contracts.
- **Local Peer**: This node's own identity, private key, services, and data URL.

The peer DB uses a **slot-based system** (with `NUM_SLOTS = 8` per AS) to limit how many peers are stored per autonomous system, preventing Sybil attacks from a single network.

```rust
pub struct LocalPeer {
    pub network_id: u32,
    pub parent_network_id: u32,
    nonce: [u8; 32],
    pub private_key: Secp256k1PrivateKey,
    pub private_key_expire: u64,
    pub addrbytes: PeerAddress,
    pub port: u16,
    pub services: u16,
    pub data_url: UrlString,
    pub stacker_dbs: Vec<QualifiedContractIdentifier>,
    pub public_ip_address: Option<(PeerAddress, u16)>,
}
```

## NAT Traversal

Nodes behind NATs cannot receive inbound connections directly. The protocol addresses this with two mechanisms:

1. **NatPunchRequest / NatPunchReply** (`stackslib/src/net/mod.rs:1076`): A node sends a `NatPunchRequest` to a peer, which replies with a `NatPunchReply` containing the requester's public IP address and port as seen from the outside. This lets a NAT'ed node discover its own public address.

2. **Pingback mechanism** (`NeighborPingback` in `walk.rs:41`): When an inbound peer completes a handshake, the node records it in `walk_pingbacks`. During the neighbor walk, the node attempts to connect back to these peers to verify bidirectional connectivity. If the pingback succeeds, the peer is added to the frontier.

## Nack Error Codes

When a peer cannot fulfill a request, it sends a `Nack` with a specific error code (`stackslib/src/net/mod.rs:1041-1063`):

| Code | Name | Meaning |
|------|------|---------|
| 1 | `HandshakeRequired` | Must handshake before sending other messages |
| 2 | `NoSuchBurnchainBlock` | Unknown burnchain block referenced |
| 3 | `Throttled` | Bandwidth limit exceeded |
| 4 | `InvalidPoxFork` | PoX fork not recognized as canonical |
| 5 | `InvalidMessage` | Message inappropriate for current protocol state |
| 6 | `NoSuchDB` | StackerDB not configured on this node |
| 7 | `StaleVersion` | StackerDB chunk version is older than local |
| 8 | `StaleView` | Burnchain view too outdated |
| 9 | `FutureVersion` | StackerDB chunk version is newer than local |
| 10 | `FutureView` | StackerDB state view is stale locally relative to the requested version |

## How It Connects

- **Chapter 23 (Block Sync)** depends on the P2P protocol for inventory exchange (`GetBlocksInv`/`BlocksInv`, `GetNakamotoInv`/`NakamotoInv`) and block download.
- **Chapter 24 (RPC API)** runs on a separate HTTP server that shares the same `PeerNetwork` and `mio` event loop.
- **Chapter 25 (StackerDB)** extends the handshake with `StackerDBHandshakeAccept` and adds its own message types for chunk synchronization.
- **Chapter 26 (Relay and Mempool)** consumes the `NetworkResult` produced by `PeerNetwork::run()` and processes blocks and transactions into the chainstate.

## Edge Cases and Gotchas

1. **NeighborKey equality ignores minor version**: The `PartialEq` implementation only checks the major version byte (`self.peer_version & 0xff000000`). This means peers with different patch versions are considered the same peer.

2. **Heartbeat interval is clamped**: `MAX_PEER_HEARTBEAT_INTERVAL` is 6 hours. If a peer claims a longer heartbeat, the node forces the maximum (chat.rs:1331-1337).

3. **PeerNetwork is not thread-safe**: The event loop runs on a single thread. Other threads (like the Relayer) communicate via `NetworkHandle`, which uses a bounded `SyncSender` channel. If the channel is full, requests are dropped with `OutboxOverflow`.

4. **Relay loop detection**: The `relayers` vector in `StacksMessage` prevents infinite relay loops. Before forwarding, a node checks whether its own address is already in the relay chain.

5. **ARRAY_MAX_LEN is u32::MAX**: Protocol arrays are bounded by `u32::MAX`, but practical limits (like `MAX_NEIGHBORS_DATA_LEN = 128` for neighbor lists and `BLOCKS_PUSHED_MAX = 32` for block pushes) are much smaller.

6. **Dual state machines**: `PeerNetwork` maintains separate `work_state` and `nakamoto_work_state` fields, both of type `PeerNetworkWorkState`. During the Nakamoto transition, both epoch 2.x and Nakamoto state machines run in parallel.
