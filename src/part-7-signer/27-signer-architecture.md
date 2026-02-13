# Chapter 27: Signer Architecture (libsigner)

## Purpose

The `libsigner` crate provides the foundational library for building Stacks Nakamoto signers. In the Nakamoto consensus model, a set of signers -- selected through the Stacking mechanism -- must approve each block before it becomes final. `libsigner` defines the event types signers receive, the message types they exchange, the run loop that drives their operation, and the session abstraction for communicating with StackerDB replicas on Stacks nodes.

This crate is deliberately separated from `stacks-signer` (the binary) so that the core protocol types and abstractions can be reused across different signer implementations.

## Key Concepts

### Signer Sets and Reward Cycles

Signers operate in **reward cycles**. At any given time, two signer sets may be active: the current cycle's set (index 0 or 1, based on `reward_cycle % 2`) and the next cycle's set during the prepare phase. Each signer has a numeric ID, a Stacks public key, and a StackerDB slot for publishing messages.

### StackerDB as the Communication Backbone

Signers do not communicate peer-to-peer. Instead, they exchange messages through **StackerDB** -- a replicated, slot-based storage system embedded in the Stacks node. Each signer writes to its assigned slot in a StackerDB contract. Other signers and miners read from these slots to observe messages. The Stacks node pushes StackerDB change events to the signer via HTTP POST.

### Event-Driven Architecture

The signer operates as an event loop: it binds an HTTP server, receives events from the Stacks node (block proposals, validation responses, new burn blocks, new Stacks blocks, StackerDB chunk updates), and dispatches them to the signer's business logic through a channel.

## Architecture Overview

The system has three main layers:

1. **Event Receiver** (`SignerEventReceiver`) -- An HTTP server that listens for POSTs from the Stacks node and converts them into typed `SignerEvent` values.
2. **Run Loop** (`SignerRunLoop` trait) -- The business logic layer that processes events one at a time, with a configurable poll timeout.
3. **Session** (`StackerDBSession`) -- An HTTP client for reading from and writing to StackerDB slots on the Stacks node.

These layers run in separate threads, connected by channels:

```
Stacks Node --HTTP POST--> [SignerEventReceiver] --channel--> [SignerRunLoop]
                                                               |
                                                               +--> [StackerDBSession] --HTTP--> Stacks Node
```

## Key Data Structures

### SignerEvent

Defined in `libsigner/src/events.rs:179`, this enum represents every type of event the signer cares about:

```rust
pub enum SignerEvent<T: SignerEventTrait> {
    /// Messages from miners via .miners StackerDB contract
    MinerMessages(Vec<T>),
    /// Messages from other signers via .signers-X-Y contracts
    SignerMessages {
        signer_set: u32,
        messages: Vec<(StacksPublicKey, T)>,
        received_time: SystemTime,
    },
    /// Block validation response from the node
    BlockValidationResponse(BlockValidateResponse),
    /// Status endpoint health check
    StatusCheck,
    /// New Bitcoin burn block processed
    NewBurnBlock {
        burn_height: u64,
        burn_header_hash: BurnchainHeaderHash,
        consensus_hash: ConsensusHash,
        received_time: SystemTime,
        parent_burn_block_hash: BurnchainHeaderHash,
    },
    /// New Stacks block processed by the node
    NewBlock {
        block_id: StacksBlockId,
        consensus_hash: ConsensusHash,
        signer_sighash: Option<Sha512Trunc256Sum>,
        block_height: u64,
        transactions: Vec<StacksTransaction>,
    },
}
```

The generic parameter `T` is the signer's message codec -- for v0 signers, this is `SignerMessage` from `libsigner/src/v0/messages.rs`.

The `SignerMessages` variant pairs each decoded message with the public key recovered from the StackerDB slot's signature, allowing the receiver to identify who sent each message.

### BlockProposal

Defined in `libsigner/src/events.rs:59`:

```rust
pub struct BlockProposal {
    pub block: NakamotoBlock,
    pub burn_height: u64,
    pub reward_cycle: u64,
    pub block_proposal_data: BlockProposalData,
}
```

This is what miners send to signers through the `.miners` StackerDB contract. The `BlockProposalData` includes the miner's server version and uses a versioned serialization format for forward compatibility -- older signers can still deserialize proposals from newer miners by reading the known fields and storing unknown trailing bytes.

### SignerMessage (v0)

Defined in `libsigner/src/v0/messages.rs:175`, the v0 message enum:

```rust
pub enum SignerMessage {
    BlockProposal(BlockProposal),
    BlockResponse(BlockResponse),
    BlockPushed(NakamotoBlock),
    MockSignature(MockSignature),
    MockProposal(MockProposal),
    MockBlock(MockBlock),
    StateMachineUpdate(StateMachineUpdate),
    BlockPreCommit(Sha512Trunc256Sum),
}
```

Each variant is prefixed with a `SignerMessageTypePrefix` discriminator byte during serialization. Messages are mapped to StackerDB slots via `MessageSlotID`:

- Slot 1: `BlockResponse` -- signer's accept/reject decisions
- Slot 2: `StateMachineUpdate` -- signer state machine coordination
- Slot 3: `BlockPreCommit` -- pre-commitment before final signature

Messages from miners (`BlockProposal`, `BlockPushed`) have no `MessageSlotID` since they use the `.miners` contract, not `.signers-X-Y`.

### MessageSlotID and MinerSlotID

These enums control which StackerDB contract a message is written to. For signers:

```rust
MessageSlotID {
    BlockResponse = 1,
    StateMachineUpdate = 2,
    BlockPreCommit = 3,
}
```

For miners:

```rust
MinerSlotID {
    BlockProposal = 0,
    BlockPushed = 1,
}
```

The `stacker_db_contract()` method on `MessageSlotID` constructs the contract identifier: `signers-{signer_set}-{slot_id}` (e.g., `signers-0-1` for block responses from signer set 0).

### SignerEntries

Defined in `libsigner/src/signer_set.rs:23`, this struct parses the `NakamotoSignerEntry` reward set into efficiently-indexed lookup tables:

```rust
pub struct SignerEntries {
    pub signer_addr_to_id: HashMap<StacksAddress, u32>,
    pub signer_id_to_addr: BTreeMap<u32, StacksAddress>,
    pub signer_id_to_pk: HashMap<u32, StacksPublicKey>,
    pub signer_pk_to_id: HashMap<StacksPublicKey, u32>,
    pub signer_pks: Vec<StacksPublicKey>,
    pub signer_addresses: Vec<StacksAddress>,
    pub signer_addr_to_weight: HashMap<StacksAddress, u32>,
}
```

The `BTreeMap` for `signer_id_to_addr` preserves reward cycle order. Weights reflect stacking amounts and are used to determine signing thresholds.

## Code Walkthrough: Event Reception

The event receiver (`libsigner/src/events.rs:288`) is an HTTP server built on `tiny_http`. When the signer starts, it binds to a local address and waits for POST requests from the Stacks node:

1. **`/stackerdb_chunks`** -- Decoded as `StackerDBChunksEvent`, then converted to either `SignerEvent::MinerMessages` (if from `.miners`) or `SignerEvent::SignerMessages` (if from `.signers-X-Y`). The conversion at line 516 checks the contract name to determine which variant to produce.

2. **`/proposal_response`** -- Decoded as `BlockValidateResponse`, converted to `SignerEvent::BlockValidationResponse`.

3. **`/new_burn_block`** -- Decoded as `BurnBlockEvent`, converted to `SignerEvent::NewBurnBlock`.

4. **`/new_block`** -- Decoded as `StacksBlockEvent`, converted to `SignerEvent::NewBlock`. Transactions are deserialized from hex in the JSON payload.

5. **`/status`** -- Returns `SignerEvent::StatusCheck` and responds with "OK".

All events are acknowledged to the dispatcher immediately (even if deserialization fails) to prevent the node from retransmitting. Events are then forwarded through `out_channels` to the run loop.

## Code Walkthrough: The Run Loop

The `SignerRunLoop` trait (`libsigner/src/runloop.rs:42`) defines the core interface:

```rust
pub trait SignerRunLoop<R: Send, T: SignerEventTrait> {
    fn set_event_timeout(&mut self, timeout: Duration);
    fn get_event_timeout(&self) -> Duration;
    fn run_one_pass(&mut self, event: Option<SignerEvent<T>>, res: &Sender<R>) -> Option<R>;
}
```

The `main_loop` method (line 59) polls the event channel with `recv_timeout`. On each iteration, it passes the event (or `None` on timeout) to `run_one_pass`. If `run_one_pass` returns `Some(R)`, the loop exits and signals the event receiver to stop.

The `Signer` struct (`libsigner/src/runloop.rs:86`) orchestrates the threading:

```rust
pub struct Signer<R, SL, EV, T> {
    signer_loop: Option<SL>,       // The business logic (SignerRunLoop impl)
    event_receiver: Option<EV>,     // The HTTP event server
    result_sender: Option<Sender<R>>, // Channel for sending results out
    phantom_data: PhantomData<T>,
}
```

The `spawn` method (line 209) performs the full setup:

1. Creates a channel pair `(event_send, event_recv)` for events
2. Adds `event_send` as a consumer on the event receiver
3. Binds the HTTP server to the configured address
4. Gets two stop signalers (one for the run loop, one for cleanup)
5. Spawns the event receiver thread (128 MB stack, named `event_receiver:{port}`)
6. Spawns the run loop thread (128 MB stack, named `signer_runloop:{port}`)
7. Returns a `RunningSigner` handle

The `RunningSigner` struct provides `stop()` and `join()` methods for clean shutdown. Signal handling is provided by `set_runloop_signal_handler` (line 163), which catches SIGINT/SIGTERM for graceful shutdown and SIGBUS for immediate abort with core dump.

## Code Walkthrough: Session Management

The `StackerDBSession` (`libsigner/src/session.rs:100`) manages TCP connections to a Stacks node's StackerDB RPC endpoints:

```rust
pub struct StackerDBSession {
    pub host: String,
    pub stackerdb_contract_id: QualifiedContractIdentifier,
    sock: Option<TcpStream>,
    socket_timeout: Duration,
}
```

Key operations:

- **`list_chunks()`** -- GET to `/v2/stackerdb/{contract}/metadata`, returns `Vec<SlotMetadata>`
- **`get_chunks()`** -- GET to `/v2/stackerdb/{contract}/chunks/{slot_id}/{version}`
- **`get_latest_chunks()`** -- GET to `/v2/stackerdb/{contract}/chunks/{slot_id}`, with size validation against `SIGNERS_STACKERDB_CHUNK_SIZE`
- **`put_chunk()`** -- POST to `/v2/stackerdb/{contract}/chunks`, returns `StackerDBChunkAckData`
- **`get_latest<T>()`** -- Convenience method that fetches the latest chunk and deserializes it using `StacksMessageCodec`

Each RPC call creates a fresh TCP connection (the code notes at line 143 that persistent connections have a known issue tracked in GitHub #3922). Read and write timeouts are applied to prevent hangs.

## How It Connects

- **Chapter 25 (StackerDB)**: The message transport layer. `libsigner` reads from and writes to StackerDB instances.
- **Chapter 28 (Signer Implementation)**: The `stacks-signer` binary implements `SignerRunLoop` with the actual block validation and signing logic.
- **Chapter 19 (Nakamoto Mining)**: Miners send `BlockProposal` messages to signers and wait for `BlockResponse` messages.
- **Chapter 18 (Nakamoto Consensus)**: The signer threshold signature scheme is what makes Nakamoto blocks final.

## Edge Cases and Gotchas

1. **Two signer sets active simultaneously**: During the prepare phase, both the current and next reward cycle's signers may be running. The `reward_cycle % 2` indexing means only two can coexist at once.

2. **Event acknowledgment before processing**: The receiver always ACKs the Stacks node (line 507 in events.rs) before attempting to deserialize the event body. This prevents the node from repeatedly retransmitting events that fail to parse.

3. **StackerDB chunk size limits**: Signer messages have a different size limit (`SIGNERS_STACKERDB_CHUNK_SIZE`) than regular StackerDB chunks (`STACKERDB_MAX_CHUNK_SIZE`). The session checks the contract name to apply the correct limit.

4. **Stop signal via TCP**: The `SignerStopSignaler` sends an actual HTTP POST to `/shutdown` on the signer's own address to wake the blocking `recv()` call. The comment at line 354 ("This makes me sad...but for now...it works") acknowledges this is a workaround for `tiny_http` not supporting non-blocking shutdown.

5. **Forward-compatible serialization**: `BlockProposalData` uses a length-prefixed inner byte buffer so older signers can read past unknown fields added in newer versions. This is critical for rolling upgrades of the signer network.
