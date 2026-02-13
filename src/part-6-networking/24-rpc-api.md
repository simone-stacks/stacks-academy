# Chapter 24: RPC API

## Purpose

The Stacks RPC API provides an HTTP interface for external clients -- wallets, explorers, miners, signers, and other nodes -- to query chain state and submit transactions. It is the primary way applications interact with a running Stacks node.

The API is not a standalone web server. It runs inside the same event loop as the P2P protocol, sharing the `PeerNetwork`'s `mio`-based poll system. This co-hosting means the HTTP server has zero-copy access to the same chain state that the P2P layer uses, avoiding the overhead of inter-process communication.

## Key Concepts

**Trait-based request handlers.** Every API endpoint is a struct that implements the `RPCRequestHandler` trait. This trait requires methods to parse the request, execute logic against chain state, and produce a response. Adding a new endpoint means creating a new handler struct and registering it.

**Regex-based routing.** Routes are matched by HTTP verb + regex pattern. The `StacksHttp` struct holds a vector of `(verb, regex, permissive_regex, handler)` tuples. For each incoming request, it iterates through handlers to find a match.

**StacksNodeState provides chain access.** Handlers do not receive raw database connections. Instead, they receive a `&mut StacksNodeState`, which wraps `PeerNetwork`, `SortitionDB`, `StacksChainState`, `MemPoolDB`, and `RPCHandlerArgs` into a single facade. The `with_node_state()` method provides scoped access.

## Architecture

```
External Client
    |
    | HTTP request
    v
mio event loop (PeerNetwork::run)
    |
    v
HttpPeer (server.rs) -- accepts connections, creates ConversationHttp
    |
    v
ConversationHttp (rpc.rs) -- per-connection session, read/write buffering
    |
    v
StacksHttp (httpcore.rs) -- protocol family, routes to handlers
    |
    v
RPCRequestHandler impl (api/*.rs) -- endpoint logic
    |
    v
StacksNodeState -- scoped access to PeerNetwork, SortitionDB, ChainState, MemPool
```

## Key Data Structures

### ConversationHttp

Defined in `stackslib/src/net/rpc.rs:39`:

```rust
pub struct ConversationHttp {
    connection: ConnectionHttp,
    conn_id: usize,
    timeout: u64,
    peer_host: PeerHost,
    outbound_url: Option<UrlString>,
    peer_addr: SocketAddr,
    keep_alive: bool,
    total_request_count: u64,
    total_reply_count: u64,
    last_request_timestamp: u64,
    last_response_timestamp: u64,
    connection_time: u64,
    reply_streams: VecDeque<(ReplyHandleHttp, HttpResponseContents, bool)>,
    pending_request: Option<ReplyHandleHttp>,
    pending_response: Option<StacksHttpResponse>,
    socket_send_buffer_size: u32,
}
```

This is the HTTP equivalent of `ConversationP2P`. Each HTTP connection gets its own `ConversationHttp` that manages buffered I/O, request parsing, and response streaming. The `reply_streams` queue allows pipelining multiple responses (though in practice, most clients send one request at a time).

### StacksHttp

Defined in `stackslib/src/net/httpcore.rs:984`:

```rust
pub struct StacksHttp {
    peer_addr: SocketAddr,
    body_start: Option<usize>,
    num_preamble_bytes: usize,
    last_four_preamble_bytes: [u8; 4],
    reply: Option<StacksHttpReplyData>,
    chunk_size: usize,
    request_handler_index: Option<usize>,
    request_handlers: Vec<(String, Regex, Regex, Box<dyn RPCRequestHandler>)>,
    pub maximum_call_argument_size: u32,
    pub read_only_call_limit: ExecutionCost,
    pub auth_token: Option<String>,
    allow_arbitrary_response: bool,
}
```

`StacksHttp` implements the `ProtocolFamily` trait, making it the HTTP equivalent of `StacksP2P`. The `request_handlers` vector is the routing table -- each entry is a `(verb, strict_regex, permissive_regex, handler)` tuple. The two-regex approach enables distinguishing between 404 (no matching route) and 400 (matched route, invalid parameters):

- The **strict regex** matches only well-formed parameters
- The **permissive regex** loosens parameter patterns to `[a-zA-Z0-9._~:@!$&'()*+,;=-]+`
- If strict fails but permissive matches, the server returns 400 instead of 404
- If neither matches but a different verb's permissive regex matches, the server returns 405 (Method Not Allowed)

### RPCRequestHandler Trait

Defined in `stackslib/src/net/httpcore.rs:413`:

```rust
pub trait RPCRequestHandler: HttpRequest + HttpResponse + RPCRequestHandlerClone {
    fn restart(&mut self);
    fn try_handle_request(
        &mut self,
        request_preamble: HttpRequestPreamble,
        request_body: HttpRequestContents,
        state: &mut StacksNodeState,
    ) -> Result<(HttpResponsePreamble, HttpResponseContents), NetError>;
}
```

Every handler must implement three traits:

- `HttpRequest`: Provides `verb()`, `path_regex()`, `metrics_identifier()`, and `try_parse_request()` -- these define the route and parse incoming data.
- `HttpResponse`: Provides `try_parse_response()` -- used when the node itself acts as an HTTP client (e.g., fetching blocks from peers).
- `RPCRequestHandler`: Provides `try_handle_request()` -- the actual endpoint logic, with access to `StacksNodeState`.

### StacksNodeState

Defined in `stackslib/src/net/mod.rs:629`:

```rust
pub struct StacksNodeState<'a> {
    inner_network: Option<&'a mut PeerNetwork>,
    inner_sortdb: Option<&'a SortitionDB>,
    inner_chainstate: Option<&'a mut StacksChainState>,
    inner_mempool: Option<&'a mut MemPoolDB>,
    inner_rpc_args: Option<&'a RPCHandlerArgs<'a>>,
    relay_message: Option<StacksMessageType>,
    ibd: bool,
    txindex: bool,
}
```

All fields are `Option` because they are temporarily `take()`-en out during handler execution and restored afterward. The `with_node_state()` method provides a closure-based API:

```rust
node.with_node_state(|network, sortdb, chainstate, mempool, rpc_args| {
    // handler logic here
})
```

## Endpoint Catalog

The full list of registered endpoints is in `stackslib/src/net/api/mod.rs:77-158`. Key categories:

### Node Information
| Endpoint | File | Description |
|----------|------|-------------|
| `GET /v2/info` | `getinfo.rs` | Node version, chain tips, peer info |
| `GET /v2/pox` | `getpoxinfo.rs` | Current PoX state and parameters |
| `GET /v2/neighbors` | `getneighbors.rs` | Known P2P neighbors |
| `GET /v2/health` | `gethealth.rs` | Simple liveness check |

### Block Data
| Endpoint | File | Description |
|----------|------|-------------|
| `GET /v2/blocks/{block_hash}` | `getblock.rs` | Epoch 2.x block by hash |
| `GET /v3/blocks/{block_id}` | `getblock_v3.rs` | Nakamoto block by ID |
| `GET /v3/blocks/height/{height}` | `getblockbyheight.rs` | Nakamoto block by height |
| `GET /v3/tenures/{consensus_hash}` | `gettenure.rs` | Nakamoto tenure blocks |
| `GET /v3/tenures/info` | `gettenureinfo.rs` | Current tenure information |

### Smart Contract Reads
| Endpoint | File | Description |
|----------|------|-------------|
| `POST /v2/contracts/call-read/{addr}/{name}/{function}` | `callreadonly.rs` | Call read-only function |
| `GET /v2/contracts/source/{addr}/{name}` | `getcontractsrc.rs` | Get contract source |
| `GET /v2/contracts/interface/{addr}/{name}` | `getcontractabi.rs` | Get contract ABI |
| `GET /v2/data_var/{addr}/{name}/{var}` | `getdatavar.rs` | Read data variable |
| `POST /v2/map_entry/{addr}/{name}/{map}` | `getmapentry.rs` | Read map entry |

### Transactions
| Endpoint | File | Description |
|----------|------|-------------|
| `POST /v2/transactions` | `posttransaction.rs` | Submit transaction |
| `GET /v2/transactions/{txid}` | `gettransaction.rs` | Get confirmed transaction |
| `POST /v2/fees/transaction` | `postfeerate.rs` | Estimate transaction fees |
| `GET /v2/accounts/{addr}` | `getaccount.rs` | Account balance and nonce |

### StackerDB
| Endpoint | File | Description |
|----------|------|-------------|
| `GET /v2/stackerdb/{addr}/{name}/chunks` | `getstackerdbchunk.rs` | Get StackerDB chunk |
| `GET /v2/stackerdb/{addr}/{name}/metadata` | `getstackerdbmetadata.rs` | Get slot metadata |
| `POST /v2/stackerdb/{addr}/{name}/chunks` | `poststackerdbchunk.rs` | Write StackerDB chunk |

### Mining
| Endpoint | File | Description |
|----------|------|-------------|
| `POST /v2/block_proposal` | `postblock_proposal.rs` | Propose block (auth required) |
| `POST /v3/blocks` | `postblock_v3.rs` | Submit Nakamoto block (auth required) |

## Code Walkthrough: How a Request Is Processed

Let us trace a `GET /v2/info` request end-to-end.

### 1. Socket Accept

The `mio` event loop detects a new connection on the HTTP listener. `HttpPeer` (in `server.rs`) accepts the TCP connection and creates a new `ConversationHttp`.

### 2. Request Parsing

When data arrives on the socket, `ConversationHttp` reads bytes into its buffer. `StacksHttp::read_preamble()` searches for the `\r\n\r\n` delimiter marking the end of HTTP headers. Once found, it parses the HTTP request line and headers into an `HttpRequestPreamble`.

### 3. Route Matching

`StacksHttp` iterates through `request_handlers` looking for a matching verb and regex:

```rust
// In httpcore.rs - conceptual flow
for (verb, strict_regex, permissive_regex, handler) in &self.request_handlers {
    if verb == request.verb() && strict_regex.is_match(request.path()) {
        // Found handler, parse the request
        let contents = handler.try_parse_request(&preamble, &captures, query, &body)?;
        return handler;
    }
}
```

For `GET /v2/info`, the `RPCPeerInfoRequestHandler` matches because its regex is `^/v2/info$` (defined in `stackslib/src/net/api/getinfo.rs:158`).

### 4. Handler Execution

The matched handler's `try_handle_request()` is called with the parsed request and a `StacksNodeState`:

```rust
// getinfo.rs:188
fn try_handle_request(
    &mut self,
    preamble: HttpRequestPreamble,
    _contents: HttpRequestContents,
    node: &mut StacksNodeState,
) -> Result<(HttpResponsePreamble, HttpResponseContents), NetError> {
    let rpc_peer_info = node.with_node_state(|network, _sortdb, chainstate, _mempool, rpc_args| {
        Ok(RPCPeerInfoData::from_network(
            network, chainstate, rpc_args.exit_at_block_height,
            &rpc_args.genesis_chainstate_hash, coinbase_height, ibd,
        ))
    });
    let preamble = HttpResponsePreamble::ok_json(&preamble);
    let body = HttpResponseContents::try_from_json(&rpc_peer_info)?;
    Ok((preamble, body))
}
```

### 5. Response

The handler returns `(HttpResponsePreamble, HttpResponseContents)`. The `ConversationHttp` serializes the response headers and body, then writes them to the socket using chunked transfer encoding.

## Code Walkthrough: The RPCPeerInfoData Response

The `GET /v2/info` response (`RPCPeerInfoData`, `stackslib/src/net/api/getinfo.rs:55`) is the richest snapshot of node state:

```rust
pub struct RPCPeerInfoData {
    pub peer_version: u32,
    pub pox_consensus: ConsensusHash,
    pub burn_block_height: u64,
    pub stable_pox_consensus: ConsensusHash,
    pub stable_burn_block_height: u64,
    pub server_version: String,
    pub network_id: u32,
    pub parent_network_id: u32,
    pub stacks_tip_height: u64,
    pub stacks_tip: BlockHeaderHash,
    pub stacks_tip_consensus_hash: ConsensusHash,
    pub genesis_chainstate_hash: Sha256Sum,
    pub unanchored_tip: Option<StacksBlockId>,
    pub tenure_height: u64,
    pub exit_at_block_height: Option<u64>,
    pub is_fully_synced: bool,
    pub node_public_key: Option<StacksPublicKeyBuffer>,
    pub node_public_key_hash: Option<Hash160>,
    pub last_pox_anchor: Option<RPCLastPoxAnchorData>,
    pub stackerdbs: Option<Vec<String>>,
}
```

All these fields come directly from `PeerNetwork` -- `chain_view`, `stacks_tip`, `local_peer`, etc. The `from_network()` constructor pulls them together (getinfo.rs:88-148).

## Fee Estimation

The `POST /v2/fees/transaction` endpoint (`postfeerate.rs`) is notable for its complexity. It accepts a serialized transaction payload and returns three fee estimates (low, middle, high):

```rust
pub struct RPCFeeEstimateResponse {
    pub estimated_cost: ExecutionCost,
    pub estimated_cost_scalar: u64,
    pub estimations: Vec<RPCFeeEstimate>,
    pub cost_scalar_change_by_byte: f64,
}
```

The estimation pipeline:
1. Deserialize the transaction payload
2. Use the `CostEstimator` to estimate execution cost
3. Use the `CostMetric` to reduce the multi-dimensional cost to a scalar
4. Use the `FeeEstimator` to convert the scalar to fee rates at three confidence levels
5. Multiply fee rates by the estimated transaction size

The response includes `cost_scalar_change_by_byte`, which tells clients how much the cost scalar increases per additional byte of transaction data -- useful for wallets that need to estimate fees before the final transaction is constructed.

## Authentication

Some endpoints require an authentication token, configured via `StacksHttp::auth_token`. The block proposal endpoint (`postblock_proposal.rs`) and the Nakamoto block submission endpoint (`postblock_v3.rs`) check for this token in the request headers. This prevents unauthorized parties from influencing block production.

## Tip Query Parameter

Many read endpoints accept a `tip=` query parameter to specify which chain state to query:

```rust
pub enum TipRequest {
    UseLatestAnchoredTip,         // tip= absent or empty
    UseLatestUnconfirmedTip,      // tip=latest
    SpecificTip(StacksBlockId),   // tip=<block_id_hex>
}
```

This is defined in `stackslib/src/net/httpcore.rs:88` and allows clients to read historical state or the latest unconfirmed state.

## How It Connects

- **Chapter 22 (P2P Protocol)**: The HTTP server runs alongside the P2P server in the same `PeerNetwork::run()` event loop, sharing the `mio` poll system.
- **Chapter 23 (Block Sync)**: The Nakamoto downloader uses HTTP RPC calls to fetch tenure blocks from peers (via `GET /v3/tenures/{consensus_hash}`).
- **Chapter 25 (StackerDB)**: StackerDB chunks are read/written through HTTP endpoints as well as P2P messages.
- **Chapter 26 (Relay and Mempool)**: The `POST /v2/transactions` endpoint feeds into the mempool, and uploaded blocks/microblocks go through the relay layer.

## Edge Cases and Gotchas

1. **Request handler index caching**: When `StacksHttp` finds a matching handler for a request, it caches the index in `request_handler_index` so the response parser can find the same handler without re-matching. This avoids O(n) regex matching on both the request and response paths.

2. **Read-only call limits**: The `callreadonly` endpoint enforces both a `read_only_call_limit` (execution cost budget) and `maximum_call_argument_size` (32KB by default). If a read-only call exceeds the cost budget, it is aborted.

3. **STACKS_HEADER_HEIGHT**: Every response includes an `X-Canonical-Stacks-Tip-Height` header (defined in `httpcore.rs:60`). Clients can use this to detect whether the node is synced without making a separate `/v2/info` call.

4. **HTTP_REQUEST_ID_RESERVED = 0**: Responses from non-Stacks HTTP servers (CDNs, Gaia hubs) lack the `X-Request-Id` header, so they are assumed to have request ID 0. This is a compatibility shim for fetching block data from generic HTTP endpoints.

5. **Chunked transfer encoding**: Large responses (like full blocks) are sent using HTTP chunked transfer encoding with a configurable `STREAM_CHUNK_SIZE` of 4096 bytes (`rpc.rs:37`). This prevents the node from having to buffer entire blocks in memory before sending.

6. **Connection limits**: The HTTP server respects `max_http_clients` from `ConnectionOptions`. Connections beyond this limit are refused. Each connection has an `idle_timeout` after which it is closed.
