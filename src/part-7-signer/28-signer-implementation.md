# Chapter 28: Signer Implementation (stacks-signer)

## Purpose

The `stacks-signer` crate is the actual signer binary that operators run alongside their Stacks nodes. It implements the block validation logic, the state machine for deciding whether to approve or reject blocks, a local SQLite database (`SignerDb`) for tracking block proposals and signer state, and the outer run loop that manages signer lifecycle across reward cycles. This chapter traces how a block proposal becomes an approved or rejected block.

## Key Concepts

### The Signer Decision Pipeline

When a miner proposes a block, the signer goes through these stages:

1. **Receive** the `BlockProposal` via StackerDB
2. **Validate** by forwarding the block to the Stacks node's `/v2/block_proposal` endpoint
3. **Evaluate** the node's validation response against local policy (tenure rules, state machine)
4. **Vote** by signing the block hash (accept) or creating a rejection message
5. **Broadcast** the vote to StackerDB for other signers and the miner to observe
6. **Track** the global consensus (did enough signers accept/reject?)

### Block States

A block proposal moves through a state machine tracked in `SignerDb`:

```
Unprocessed -> PreCommitted -> LocallyAccepted -> GloballyAccepted
                            -> LocallyRejected -> GloballyRejected
```

### Two Running Signers

The outer run loop maintains up to two signer instances: one for the current reward cycle and one for the next (during prepare phase). They are indexed by `reward_cycle % 2`.

## Architecture Overview

```
stacks-signer binary
  |
  +-- GlobalConfig (TOML config file)
  |
  +-- SpawnedSigner (crate::SpawnedSigner<Signer, SignerMessage>)
  |     |
  |     +-- SignerEventReceiver (HTTP server, receives events from node)
  |     +-- RunLoop (implements SignerRunLoop trait)
  |           |
  |           +-- stacks_signers: HashMap<u64, ConfiguredSigner>
  |           |     +-- Signer (v0) -- actual signing logic
  |           |           +-- SignerDb (SQLite)
  |           |           +-- StackerDB client
  |           |           +-- LocalStateMachine
  |           |           +-- GlobalStateEvaluator
  |           +-- StacksClient (HTTP client to node RPCs)
  |           +-- SortitionsView (cached sortition data)
```

## Key Data Structures

### Signer (v0)

Defined in `stacks-signer/src/v0/signer.rs:89`, this is the core signer for the current protocol version:

```rust
pub struct Signer {
    private_key: StacksPrivateKey,
    pub stacks_address: StacksAddress,
    pub stackerdb: StackerDB<MessageSlotID>,
    pub mainnet: bool,
    pub mode: SignerMode,
    pub signer_slot_ids: Vec<SignerSlotID>,
    pub signer_addresses: Vec<StacksAddress>,
    pub reward_cycle: u64,
    pub signer_weights: HashMap<StacksAddress, u32>,
    pub signer_db: SignerDb,
    pub proposal_config: ProposalEvalConfig,
    pub block_proposal_validation_timeout: Duration,
    pub submitted_block_proposal: Option<(Sha512Trunc256Sum, Instant)>,
    pub block_proposal_max_age_secs: u64,
    pub local_state_machine: LocalStateMachine,
    recently_processed: RecentlyProcessedBlocks<100>,
    pub global_state_evaluator: GlobalStateEvaluator,
    pub validate_with_replay_tx: bool,
    pub tx_replay_scope: ReplayScopeOpt,
    pub reset_replay_set_after_fork_blocks: u64,
    pub capitulate_miner_view_timeout: Duration,
    pub last_capitulate_miner_view: SystemTime,
}
```

Key fields:
- `mode` -- Either `DryRun` (observer only) or `Normal { signer_id }` (active participant)
- `signer_weights` -- Maps signer addresses to their stacking weight, used for threshold calculations
- `local_state_machine` -- Tracks what this signer believes the current chain state is
- `global_state_evaluator` -- Evaluates global signer consensus from `StateMachineUpdate` messages
- `submitted_block_proposal` -- The block currently being validated by the node, with submission timestamp for timeout tracking

### BlockInfo

Defined in `stacks-signer/src/signerdb.rs:178`, this tracks everything known about a block proposal:

```rust
pub struct BlockInfo {
    pub block: NakamotoBlock,
    pub burn_block_height: u64,
    pub reward_cycle: u64,
    pub vote: Option<NakamotoBlockVote>,
    pub valid: Option<bool>,
    pub signed_over: bool,
    pub proposed_time: u64,
    pub signed_self: Option<u64>,
    pub signed_group: Option<u64>,
    pub state: BlockState,
    pub validation_time_ms: Option<u64>,
    pub ext: ExtraBlockInfo,
    pub reject_reason: Option<RejectReason>,
}
```

The `state` field uses the `BlockState` enum (`signerdb.rs:116`):

```rust
BlockState {
    Unprocessed = 0,
    LocallyAccepted = 1,
    LocallyRejected = 2,
    GloballyAccepted = 3,
    GloballyRejected = 4,
    PreCommitted = 5,
}
```

State transitions are validated by `check_state()` -- for example, you cannot go from `GloballyAccepted` to `GloballyRejected`, and `PreCommitted` can only be entered from `Unprocessed`.

### SignerDb

Defined in `stacks-signer/src/signerdb.rs:350`:

```rust
pub struct SignerDb {
    db: Connection,
}
```

This is a SQLite database that persists signer state across restarts. Its schema includes:

- **`blocks`** -- Every block proposal the signer has seen, with consensus hash, height, state, and full `BlockInfo` as JSON
- **`burn_blocks`** -- Burn blocks observed by this signer
- **`signer_states`** -- Encrypted signer state per reward cycle
- **`block_signatures`** -- Individual signatures collected for blocks
- **`block_rejection_signer_addrs`** -- Which signers rejected which blocks
- **`db_config`** -- Schema version tracking

The database goes through multiple schema migrations (the code contains `CREATE_BLOCKS_TABLE_1` through progressive versions, and migration logic like `MIGRATE_BLOCKS_TABLE_2_BLOCKS_TABLE_3`). Indexes are carefully crafted for the most common query patterns.

### RunLoop

Defined in `stacks-signer/src/runloop.rs:183`:

```rust
pub struct RunLoop<Signer, T> {
    pub config: GlobalConfig,
    pub stacks_client: StacksClient,
    pub stacks_signers: HashMap<u64, ConfiguredSigner<Signer, T>>,
    pub state: State,
    pub current_reward_cycle_info: Option<RewardCycleInfo>,
    pub sortition_state: Option<SortitionsView>,
}
```

The `stacks_signers` map uses `reward_cycle % 2` as the key, holding either a `RegisteredSigner` or `NotRegistered` variant.

### ConfiguredSigner

The `ConfiguredSigner` enum (`runloop.rs:133`) distinguishes between active and inactive signers:

```rust
pub enum ConfiguredSigner<Signer, T> {
    RegisteredSigner(Signer),
    NotRegistered { cycle: u64, _phantom_state: PhantomData<T> },
}
```

## Code Walkthrough: Signer Startup

The entry point is `stacks-signer/src/main.rs:217`. The `main()` function:

1. Parses CLI arguments with `clap` (the `Cli` struct)
2. Sets up tracing/logging
3. For the `Run` command, loads `GlobalConfig` from a TOML file
4. Creates a `SpawnedSigner` (`v0::SpawnedSigner::new(config)`)
5. Calls `join()` to block until the signer exits

The `SpawnedSigner` (defined in `stacks-signer/src/v0/mod.rs:31` as `type SpawnedSigner = crate::SpawnedSigner<Signer, SignerMessage>`) is constructed with:
- A `RunLoop` instance (the outer event-processing loop)
- A `SignerEventReceiver` (the HTTP server)
- A result sender channel

Calling `spawn()` on it starts the HTTP server and run loop in separate threads.

## Code Walkthrough: Event Processing

The `RunLoop::run_one_pass()` method (`runloop.rs:514`) is called on each event:

1. **Status checks**: If the event is `StatusCheck`, gather state info from all running signers and send it back through the result channel.

2. **Initialization**: On the first pass (`State::Uninitialized`), call `initialize_runloop()` which:
   - Fetches the current reward cycle info from the node via `get_current_reward_cycle_info()`
   - Calls `refresh_signer_config()` for the current cycle
   - If in the prepare phase, also refreshes config for the next cycle

3. **Burn block updates**: On `NewBurnBlock` events, call `refresh_runloop()` which:
   - Updates the reward cycle info if the cycle changed
   - Refreshes signer configs for cycles that aren't yet configured
   - Cleans up stale signers from past cycles (only if they have no unprocessed blocks)

4. **Dispatch to signers**: For each registered signer, call `signer.process_event()` with the event, the shared `StacksClient`, and the `SortitionsView`.

### Signer Configuration

`get_signer_config()` (`runloop.rs:236`) builds a `SignerConfig` by:

1. Fetching the reward set from the node
2. Parsing it into `SignerEntries`
3. Checking that the StackerDB has been updated for this cycle
4. Looking up the signer's slot ID and signer ID in the reward set
5. If `dry_run` is configured, verifying the signer is NOT registered (to prevent conflicts)

## Code Walkthrough: Block Validation Decision

The v0 `Signer` (in `stacks-signer/src/v0/signer.rs`) processes block proposals through several phases:

### Phase 1: Receiving the Proposal

When a `MinerMessages` event arrives containing a `BlockProposal`:
- The proposal is checked against `block_proposal_max_age_secs` -- stale proposals are dropped
- A `BlockInfo` is created from the proposal and stored in `SignerDb`
- The block is submitted to the node for validation via the `/v2/block_proposal` HTTP endpoint

### Phase 2: Validation Response

When a `BlockValidationResponse` event arrives:
- If `BlockValidateOk`: the block passed the node's consensus checks
- If `BlockValidateReject`: the node found the block invalid, with a `ValidateRejectCode`

### Phase 3: Sortition-Based Evaluation

The signer cross-references the proposal against its `SortitionsView` to verify:
- The miner actually won the sortition for this tenure
- The tenure change payload (if any) is consistent with the expected consensus hash
- The block builds on the expected parent

### Phase 4: State Machine Coordination

The `LocalStateMachine` tracks the signer's view of what the current chain tip should be. The `GlobalStateEvaluator` aggregates `StateMachineUpdate` messages from other signers to reach consensus on the chain view. If the signer's local view diverges from the majority for longer than `capitulate_miner_view_timeout`, it will adopt the majority view.

### Phase 5: Signing or Rejecting

If the block is valid and passes all checks:
1. The signer creates a `NakamotoBlockVote` with the signer signature hash
2. Signs it with its private key
3. Wraps it in a `BlockResponse::Accepted(BlockAccepted { ... })`
4. Publishes to StackerDB slot `MessageSlotID::BlockResponse`

If rejected:
1. Creates a `BlockRejection` with the `RejectReason`
2. Wraps it in `BlockResponse::Rejected(BlockRejection { ... })`
3. Publishes to the same StackerDB slot

### Phase 6: Global Consensus Tracking

As `SignerMessages` events arrive with other signers' `BlockResponse` messages:
- Accepted signatures are stored in the `block_signatures` table
- Rejected signer addresses are stored in `block_rejection_signer_addrs`
- When enough weighted signatures accumulate, the block transitions to `GloballyAccepted`
- When enough weighted rejections accumulate, it transitions to `GloballyRejected`

## The Pre-Commit Protocol

Before sending a full signature, a signer may send a `BlockPreCommit` message (just the signer signature hash). This indicates intent to sign without committing the actual signature, allowing the network to gauge whether a block will reach threshold before expending the full signing cost. Pre-committed blocks enter `BlockState::PreCommitted`.

## Dry-Run Mode

When `dry_run: true` is set in the config, the signer:
- Does NOT publish any messages to StackerDB
- Still receives and processes all events
- Validates blocks and tracks state
- Fails to start if the configured address is actually registered in the reward set (safety check)

This mode is useful for monitoring the signer network without participating.

## How It Connects

- **Chapter 27 (Signer Architecture)**: `libsigner` provides the event receiver, run loop trait, and session that this implementation builds on.
- **Chapter 25 (StackerDB)**: The transport mechanism for all signer-to-signer and miner-to-signer communication.
- **Chapter 18 (Nakamoto Consensus)**: The signer threshold is what makes Nakamoto blocks final. The weights in `signer_weights` directly correspond to the stacking weights.
- **Chapter 19 (Nakamoto Mining)**: The miner sends proposals and waits for signatures from this signer implementation.
- **Chapter 16 (Stacking and PoX)**: The reward set that determines who the signers are comes from the PoX contract.

## Edge Cases and Gotchas

1. **Block state transitions are strict**: `BlockInfo::move_to()` enforces a valid state machine. You cannot go backwards (e.g., `GloballyAccepted` to `LocallyRejected`), and `PreCommitted` can only come from `Unprocessed`. Attempts to make invalid transitions return an error string.

2. **Schema migrations**: The `SignerDb` has gone through many schema versions. Each migration is applied sequentially on database open. The `blocks` table has been restructured multiple times to add new columns and indexes.

3. **Stale signer cleanup**: When a reward cycle ends, the `cleanup_stale_signers()` method removes old signers -- but only if they have no unprocessed blocks. This prevents losing votes for blocks that straddle cycle boundaries.

4. **Validation timeout**: If the node takes longer than `block_proposal_validation_timeout` to respond to a validation request, the signer treats the block as invalid. The `submitted_block_proposal` field tracks the submission time.

5. **Recently processed blocks cache**: The `RecentlyProcessedBlocks<100>` ring buffer prevents re-processing blocks the node has already confirmed. It uses a fixed-size array with a rotating write head.

6. **Two signer instances share one database**: Both the current and next cycle signers use the same `SignerDb` file. The `reward_cycle` column in the `blocks` table disambiguates data between cycles.
