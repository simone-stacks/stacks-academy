# Appendix D: Testing Infrastructure and Tools

The stacks-core repository contains a comprehensive test suite spanning unit tests, integration tests, and multi-node network tests. This appendix documents the test frameworks, key test utilities, and developer tools available in the repository.

---

## Test Organization

Tests are organized across multiple crates:

```
stackslib/src/
  chainstate/stacks/tests/       # Chain state unit tests
  chainstate/nakamoto/tests/     # Nakamoto-specific tests
  chainstate/tests/mod.rs        # TestChainstate, TestChainstateConfig
  net/tests/                     # P2P and networking tests
  core/tests/                    # Core protocol tests

stacks-node/src/tests/
  neon_integrations.rs           # Pre-Nakamoto integration tests
  nakamoto_integrations.rs       # Nakamoto integration tests
  signer/                        # Signer integration tests
  epoch_205.rs .. epoch_24.rs    # Epoch transition tests
  mempool.rs                     # Mempool tests
  atlas.rs                       # Atlas attachment tests
  stackerdb.rs                   # StackerDB tests

stacks-signer/src/tests/        # Signer binary tests

contrib/
  boot-contracts-unit-tests/     # Clarity boot contract tests
  boot-contracts-stateful-prop-tests/ # Property-based contract tests
  core-contract-tests/           # Core contract tests
```

---

## Running Tests

### Unit Tests

Run all unit tests in a specific crate:

```bash
# All stackslib tests
cargo test -p stackslib

# A specific test
cargo test -p stackslib -- test_name

# Tests in a specific module
cargo test -p stackslib chainstate::stacks::tests
```

### Integration Tests

Integration tests spawn actual node processes with Bitcoin regtest backends. They require more setup and run longer:

```bash
# Run a specific integration test
cargo test -p stacks-node --test neon_integrations -- test_name

# Run Nakamoto integration tests
cargo test -p stacks-node --test nakamoto_integrations -- test_name

# Run signer tests
cargo test -p stacks-node -- tests::signer
```

Integration tests typically need `bitcoind` available on the system PATH.

### Filtering and Options

```bash
# Run tests with output
cargo test -p stackslib -- --nocapture test_name

# Run ignored (slow) tests
cargo test -p stackslib -- --ignored test_name

# List available tests
cargo test -p stackslib -- --list
```

---

## TestPeer -- Network-Level Test Harness

`TestPeer` is the primary test harness for P2P, block processing, and coordination logic. It creates an isolated node with its own sortition DB, chain state, mempool, and P2P network -- all in memory/temp directories.

**Source:** `stackslib/src/net/mod.rs:2782`

```rust
pub struct TestPeer<'a> {
    pub config: TestPeerConfig,
    pub network: PeerNetwork,
    pub relayer: Relayer,
    pub mempool: Option<MemPoolDB>,
    pub chain: TestChainstate<'a>,
    pub rpc_handler_args: Option<RPCHandlerArgsType>,
}
```

### TestPeerConfig

Configure a test peer's network identity, initial state, and behavior:

```rust
// stackslib/src/net/mod.rs:2625
pub struct TestPeerConfig {
    pub chain_config: TestChainstateConfig,
    pub peer_version: u32,
    pub private_key: Secp256k1PrivateKey,
    pub initial_neighbors: Vec<Neighbor>,
    pub connection_opts: ConnectionOptions,
    pub server_port: u16,
    pub http_port: u16,
    pub stacker_dbs: Vec<QualifiedContractIdentifier>,
    pub services: u16,
    // ...
}
```

### Usage Pattern

```rust
// Create two peers that know about each other
let mut peer_1_config = TestPeerConfig::from_port(32000);
let mut peer_2_config = TestPeerConfig::from_port(32010);

peer_1_config.add_neighbor(&peer_2_config.to_neighbor());
peer_2_config.add_neighbor(&peer_1_config.to_neighbor());

let mut peer_1 = TestPeer::new(peer_1_config);
let mut peer_2 = TestPeer::new(peer_2_config);

// Step both peers forward
peer_1.step();
peer_2.step();
```

The `step()` method runs one iteration of the P2P state machine, processing inbound messages, running downloads, and handling RPCs. This gives tests fine-grained control over timing and ordering.

---

## TestChainstate -- Chain State Test Harness

`TestChainstate` provides an isolated chain state environment with its own sortition DB, coordinator, and MARF.

**Source:** `stackslib/src/chainstate/tests/mod.rs:142`

```rust
pub struct TestChainstate<'a> {
    pub config: TestChainstateConfig,
    pub sortdb: Option<SortitionDB>,
    pub miner: TestMiner,
    pub stacks_node: Option<TestStacksNode>,
    pub coord: ChainsCoordinator<'a, TestEventObserver, ...>,
    pub nakamoto_parent_tenure_opt: Option<Vec<NakamotoBlock>>,
    pub malleablized_blocks: Vec<NakamotoBlock>,
    pub test_path: String,
    pub chainstate_path: String,
}
```

### TestChainstateConfig

Controls the chain's initial state, epochs, and test accounts:

```rust
// stackslib/src/chainstate/tests/mod.rs:81
pub struct TestChainstateConfig {
    pub network_id: u32,
    pub current_block: u64,
    pub burnchain: Burnchain,
    pub test_name: String,
    pub initial_balances: Vec<(PrincipalData, u64)>,
    pub spending_account: TestMiner,
    pub setup_code: String,
    pub epochs: Option<EpochList>,
    pub test_stackers: Option<Vec<TestStacker>>,
    pub test_signers: Option<TestSigners>,
    pub aggregate_public_key: Option<Vec<u8>>,
}
```

The `setup_code` field accepts Clarity code that runs at genesis, useful for deploying test contracts. The `test_stackers` and `test_signers` fields configure simulated stacking participants for Nakamoto tests.

All test data is written to `/tmp/stacks-node-tests/units-test-consensus/{test_name}-{random}/`.

---

## TestEventObserver -- In-Process Event Capture

`TestEventObserver` is a lightweight, in-process implementation of the `BlockEventDispatcher` trait that captures block events for test assertions. Unlike the production `EventDispatcher` (which sends HTTP webhooks), this observer stores events in a `Mutex<Vec<>>`.

**Source:** `stackslib/src/net/mod.rs:2491`

```rust
pub struct TestEventObserver {
    blocks: Mutex<Vec<TestEventObserverBlock>>,
}

impl BlockEventDispatcher for TestEventObserver {
    fn announce_block(&self, block: &StacksBlockEventData, ...) {
        self.blocks.lock().unwrap().push(TestEventObserverBlock {
            block: block.clone(),
            metadata: metadata.clone(),
            receipts: receipts.to_owned(),
            // ...
        });
    }
}
```

### Usage

```rust
let observer = TestEventObserver::new();
let mut peer = TestPeer::new_with_observer(config, Some(&observer));

// Process blocks...
peer.step();

// Assert on captured events
let blocks = observer.get_blocks();
assert_eq!(blocks.len(), 1);
assert_eq!(blocks[0].receipts.len(), 2);
```

---

## test_observer -- HTTP Event Capture for Integration Tests

For integration tests that run full node processes, the `test_observer` module provides an HTTP server (using Warp) that captures events delivered by the production `EventDispatcher`.

**Source:** `stacks-node/src/tests/neon_integrations.rs:276`

```rust
pub mod test_observer {
    pub const EVENT_OBSERVER_PORT: u16 = 50303;

    pub static NEW_BLOCKS: Mutex<Vec<serde_json::Value>> = Mutex::new(Vec::new());
    pub static MINED_BLOCKS: Mutex<Vec<MinedBlockEvent>> = Mutex::new(Vec::new());
    pub static BURN_BLOCKS: Mutex<Vec<BurnBlockEvent>> = Mutex::new(Vec::new());
    pub static MEMTXS: Mutex<Vec<String>> = Mutex::new(Vec::new());
    pub static NEW_STACKERDB_CHUNKS: Mutex<Vec<StackerDBChunksEvent>> = Mutex::new(Vec::new());
    pub static PROPOSAL_RESPONSES: Mutex<Vec<BlockValidateResponse>> = Mutex::new(Vec::new());
    // ...
}
```

### Key Functions

| Function | Description |
|----------|-------------|
| `test_observer::spawn()` | Start the HTTP observer server on port 50303. |
| `test_observer::register_any(&mut conf)` | Register a catch-all `events_keys = ["*"]` observer. |
| `test_observer::register(&mut conf, &[keys])` | Register an observer for specific event types. |
| `test_observer::get_blocks()` | Retrieve captured `new_block` events. |
| `test_observer::get_mined_blocks()` | Retrieve captured `mined_block` events. |
| `test_observer::get_burn_blocks()` | Retrieve captured `new_burn_block` events. |
| `test_observer::get_stackerdb_chunks()` | Retrieve captured StackerDB events. |
| `test_observer::clear()` | Clear all captured events. |

### Usage in Integration Tests

```rust
fn test_my_feature() {
    let (mut conf, miner_account) = neon_integration_test_conf();
    test_observer::spawn();
    test_observer::register_any(&mut conf);

    let mut btcd_controller = BitcoinRegtestController::new(conf.clone(), None);
    btcd_controller.start_bitcoind();

    let mut run_loop = neon::RunLoop::new(conf.clone());
    let _run_loop_thread = thread::spawn(move || run_loop.start(None, 0));

    // Wait for blocks...
    next_block_and_wait(&mut btcd_controller, &conf);

    // Assert on events
    let blocks = test_observer::get_blocks();
    assert!(blocks.len() > 0);
}
```

---

## Signer Command-Based Test Framework

The signer integration tests use a command-based testing approach built on the `madhouse` crate, which enables composable, property-based test sequences. This framework is specifically designed for testing multi-miner, multi-signer Nakamoto scenarios.

**Source:** `stacks-node/src/tests/signer/commands/`

### SignerTestContext

The `SignerTestContext` (`commands/context.rs:16`) wraps a `MultipleMinerTest` setup with two miners and a configurable number of signers:

```rust
pub struct SignerTestContext {
    pub miners: Arc<Mutex<MultipleMinerTest>>,
    num_signers: usize,
    num_transfer_txs: u64,
}
```

It provides access to miner configurations, public keys, sortition DBs, and chain tip heights. The context configures generous timeouts (1800s) for block proposal validation to prevent flaky tests.

### SignerTestState

The `SignerTestState` (`commands/context.rs:138`) tracks the current test state:

```rust
pub struct SignerTestState {
    pub is_booted_to_nakamoto: bool,
    pub mining_stalled: bool,
    pub epoch_3_start_block_height: Option<u64>,
    pub last_stacks_block_height: Option<u64>,
}
```

Each command's `check()` method queries this state to determine if the command is valid to run, and `apply()` updates the state after execution.

### Available Commands

Each command implements the `Command<SignerTestState, SignerTestContext>` trait from `madhouse`:

| Command | File | Description |
|---------|------|-------------|
| `ChainBootToEpoch3` | `boot.rs` | Advance the chain to Epoch 3.0 (Nakamoto activation) |
| `MinerMineBitcoinBlocks` | `bitcoin_mining.rs` | Mine Bitcoin blocks on the regtest backend |
| `ChainGenerateBitcoinBlocks` | `bitcoin_mining.rs` | Generate Bitcoin blocks without specific miner context |
| `MinerSubmitNakaBlockCommit` | `block_commit.rs` | Submit a Nakamoto block commit operation |
| `ChainExpectNakaBlock` | `block_wait.rs` | Wait for and assert a Nakamoto block was produced |
| `ChainExpectNakaBlockProposal` | `block_wait.rs` | Wait for a block proposal event |
| `ChainExpectStacksTenureChange` | `block_wait.rs` | Wait for a tenure change transaction |
| `ChainVerifyMinerNakaBlockCount` | `block_verify.rs` | Verify expected block counts per miner |
| `ChainMinerCommitOp` | `commit_ops.rs` | Execute a miner commit operation |
| `ChainExpectSortitionWinner` | `sortition.rs` | Assert which miner won a sortition |
| `ChainVerifyLastSortitionWinnerReorged` | `sortition.rs` | Verify a sortition winner was reorged |
| `ChainStacksMining` | `stacks_mining.rs` | Trigger Stacks block mining |
| `MinerSendAndMineStacksTransferTx` | `transfer.rs` | Send a STX transfer and mine it |
| `ChainShutdownMiners` | `shutdown.rs` | Gracefully shut down miner processes |

### Command Pattern

Each command follows the `madhouse` command pattern with three required methods:

```rust
impl Command<SignerTestState, SignerTestContext> for ChainBootToEpoch3 {
    fn check(&self, state: &SignerTestState) -> bool {
        // Return true only if this command is valid in the current state
        !state.is_booted_to_nakamoto
    }

    fn apply(&self, state: &mut SignerTestState) {
        // Execute the command and update state
        self.ctx.miners.lock().unwrap().boot_to_epoch_3();
        state.is_booted_to_nakamoto = true;
    }

    fn label(&self) -> String {
        "BOOT_TO_EPOCH_3".to_string()
    }
}
```

The `check()` method enables proptest-style shrinking: only valid command sequences are generated. The `label()` provides human-readable test failure output.

---

## Chain State Test Modules

### `stackslib/src/chainstate/stacks/tests/`

Unit tests for the core chain state processing:

| File | Contents |
|------|----------|
| `mod.rs` | Test fixtures, helper functions, `TestMinerTracePoint`, `TestMinerTrace` |
| `block_construction.rs` | Block assembly and validation tests |
| `chain_histories.rs` | Fork handling and chain reorganization tests |
| `accounting.rs` | STX balance, fee, and reward accounting tests |
| `reward_set.rs` | PoX reward set calculation tests |

### `stacks-node/src/tests/`

Full node integration tests:

| File | Contents |
|------|----------|
| `neon_integrations.rs` | Pre-Nakamoto integration tests (block production, API, events) |
| `nakamoto_integrations.rs` | Nakamoto-specific integration tests |
| `signer/v0/mod.rs` | Signer integration tests |
| `signer/v0/tenure_extend.rs` | Tenure extension tests |
| `signer/v0/reorg.rs` | Chain reorganization with signers |
| `signer/v0/tx_replay.rs` | Transaction replay tests |
| `epoch_205.rs` .. `epoch_24.rs` | Epoch transition tests |
| `mempool.rs` | Mempool behavior tests |
| `atlas.rs` | Atlas attachment system tests |
| `stackerdb.rs` | StackerDB replication tests |
| `integrations.rs` | Basic node integration tests |

---

## BitcoinRegtestController -- Bitcoin Backend for Tests

Integration tests interact with Bitcoin through `BitcoinRegtestController`, which manages a `bitcoind` regtest instance and provides methods for mining blocks, submitting transactions, and querying UTXO state.

**Source:** `stacks-node/src/burnchains/bitcoin_regtest_controller.rs:86`

```rust
pub struct BitcoinRegtestController {
    config: Config,
    indexer: BitcoinIndexer,
    db: Option<SortitionDB>,
    burnchain_db: Option<BurnchainDB>,
    chain_tip: Option<BurnchainTip>,
    use_coordinator: Option<CoordinatorChannels>,
    ongoing_block_commit: Option<OngoingBlockCommit>,
    rpc_client: Option<BitcoinRpcClient>,
}
```

In tests, the controller is typically created and used like this:

```rust
let mut btcd_controller = BitcoinRegtestController::new(conf.clone(), None);
btcd_controller.start_bitcoind();

// Mine a Bitcoin block and wait for the Stacks node to process it
next_block_and_wait(&mut btcd_controller, &conf);
```

The controller communicates with `bitcoind` via its JSON-RPC interface. The `rpc_client` field is `Some` for miner nodes (which submit block commits) and `None` for follower nodes. The `OngoingBlockCommit` struct tracks in-flight block commit transactions including their UTXO set and fee calculations for potential RBF (Replace-By-Fee) bumping.

---

## Developer Tools

### clarity-cli

A command-line tool for working with Clarity smart contracts outside of a running node.

**Source:** `contrib/clarity-cli/src/main.rs`

Subcommands:

| Command | Description |
|---------|-------------|
| `initialize` | Create a new Clarity state database |
| `check` | Type-check a Clarity contract |
| `launch` | Deploy a contract to the state database |
| `execute` | Execute a Clarity expression in a contract's context |
| `eval` | Evaluate a standalone Clarity expression |
| `eval_at_chaintip` | Evaluate against the latest committed state |
| `eval_raw` | Evaluate without cost tracking |
| `repl` | Interactive Clarity REPL |
| `generate_address` | Generate a new Stacks address |

Example usage:

```bash
# Initialize a new clarity database
clarity-cli initialize /tmp/clarity-db

# Check a contract
clarity-cli check contract.clar

# Launch a contract
clarity-cli launch ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM.my-contract \
    contract.clar /tmp/clarity-db

# Start the REPL
clarity-cli repl /tmp/clarity-db
```

### stacks-inspect

A comprehensive diagnostic tool for inspecting chain state databases, decoding blockchain data structures, validating blocks, and managing shadow chains.

**Source:** `contrib/stacks-inspect/src/main.rs`

**Installation:**

```bash
cargo build --release -p stacks-inspect
# Binary: target/release/stacks-inspect
```

**Global options:**
- `--config FILE` -- Path to stacks-node TOML configuration file
- `--network-config NET` -- Predefined network: `helium`, `mainnet`, `mocknet`, `xenon`

Commands that take `<NETWORK>` accept: `mainnet` (chain ID `0x00000001`), `krypton` (chain ID `0x80000100`), `naka3` (chain ID `0x80000000`).

#### Decode Commands

Deserialize blockchain data structures from hex or binary files.

| Command | Description |
|---------|-------------|
| `decode-tx <HEX>` | Decode a Stacks transaction (hex or `<file> --file`, `- --file` for stdin) |
| `decode-block <PATH>` | Decode an Epoch 2.x block from binary file (`-` for stdin) |
| `decode-nakamoto-block <HEX>` | Decode a Nakamoto block (hex or `<file> --file`) |
| `decode-bitcoin-header <HEIGHT> <SPV_DB> [--testnet\|--regtest]` | Decode a Bitcoin header from SPV data |
| `decode-net-message <JSON_BYTES>` | Decode a Stacks P2P network message |
| `decode-microblocks <PATH>` | Decode a microblock stream (Epoch 2.x) |

Example -- extract and decode a Nakamoto block from the staging database:

```bash
sqlite3 mainnet/chainstate/blocks/nakamoto.sqlite \
  "SELECT hex(data) FROM nakamoto_staging_blocks LIMIT 1" \
  | stacks-inspect decode-nakamoto-block - --file
```

#### Database Query Commands

Query the MARF and Clarity databases directly.

| Command | Description |
|---------|-------------|
| `header-indexed-get <BLOCK_ID_HASH> <KEY> --state-dir <DIR>` | Query header-indexed MARF data (see Chapter 13 for key patterns) |
| `marf-get <MARF_PATH> <TIP_HASH> <CONSENSUS_HASH> <KEY>` | Direct MARF lookup in any MARF database (see Chapters 8, 13 for key patterns) |
| `deserialize-db <DB_PATH> <BYTE_PREFIX>` | Deserialize Clarity values from `data_table` by hex prefix (e.g., `"13"` for STX balances) |
| `check-deser-data <FILE>` | Verify output from `deserialize-db` round-trips correctly |
| `get-ancestors <DB_PATH> <BLOCK_HASH> <CONSENSUS_HASH>` | Trace block ancestry through the staging database |

Example -- query an account's STX balance from the Clarity MARF:

```bash
stacks-inspect marf-get mainnet/chainstate/vm/clarity/marf.sqlite \
  <tip_hash> <consensus_hash> \
  "vm-account::SP000000000000000000002Q6VF78::19"
```

Example -- query the highest block in a Nakamoto tenure from the headers MARF:

```bash
stacks-inspect header-indexed-get <block_id_hash> \
  "nakamoto::tenures::highest_block_in_tenure::<consensus_hash>" \
  --state-dir mainnet/chainstate
```

#### Shadow Block Commands

Shadow blocks are synthetic Nakamoto blocks used for chainstate recovery. They fill gaps when expected blocks are missing.

| Command | Description |
|---------|-------------|
| `make-shadow-block <DIR> <NETWORK> <CHAIN_TIP> [TX_HEX...]` | Create a shadow block at the chain tip |
| `shadow-chainstate-repair <DIR> <NETWORK>` | Auto-detect and repair chainstate gaps (returns `[]` if healthy) |
| `shadow-chainstate-patch <DIR> <NETWORK> <JSON_FILE>` | Apply shadow blocks from a JSON array |
| `add-shadow-block <DIR> <NETWORK> <SHADOW_BLOCK_HEX>` | Add a single shadow block |

The typical repair workflow: run `shadow-chainstate-repair` to detect gaps, review the output JSON, then apply with `shadow-chainstate-patch` if needed.

#### Nakamoto Commands

| Command | Description |
|---------|-------------|
| `get-nakamoto-tip <DIR> <NETWORK>` | Get the current Nakamoto chain tip (index block hash) |
| `get-account <DIR> <NETWORK> <ADDRESS> [CHAIN_TIP]` | Get account state (nonce, balance, lock info) at tip or specific block |
| `getnakamotoinv <PEER:PORT> <DATA_PORT> <CONSENSUS_HASH>` | Fetch Nakamoto inventory from a live peer via P2P |

#### Mining Simulation Commands

| Command | Description |
|---------|-------------|
| `try-mine <PATH> [--min-fee <u64>] [--max-time <ms>]` | Simulate mining an anchored block from current mempool |
| `tip-mine <DIR> <EVENT_LOG> <HEIGHT> <MAX_TXNS>` | Mine at a specific height using a JSONL event log |
| `replay-mock-mining <PATH> <MOCK_DIR>` | Replay mock-mined blocks from numbered JSON files |

#### Validation Commands

Re-execute blocks against chainstate to verify correctness. Same subcommands for both: `prefix <HASH>`, `first <N>`, `last <N>`, `range <START> <END>`, `index-range <START> <END>`, or none (all blocks).

| Command | Description |
|---------|-------------|
| `validate-block <DB_PATH> [SUBCOMMAND]` | Validate Epoch 2.x blocks |
| `validate-naka-block <DB_PATH> [SUBCOMMAND]` | Validate Nakamoto blocks |

#### PoX and Sortition Analysis

| Command | Description |
|---------|-------------|
| `evaluate-pox-anchor <SORTITION_DB> <START> [END]` | Evaluate PoX anchor selection at burn block heights (output: CSV) |
| `analyze-sortition-mev <BURNCHAIN_DB> <SORTITION_DB> <CHAINSTATE> <START> <END> [MINER BURN ...]` | Compare Epoch 2 vs Epoch 3 sortition algorithms; optionally simulate additional miner burns |

#### Utility Commands

| Command | Description |
|---------|-------------|
| `peer-pub-key <SEED_HEX>` | Generate peer public key from seed |
| `post-stackerdb <SLOT_ID> <VERSION> <PRIVKEY> <DATA>` | Create a signed StackerDB chunk |
| `contract-hash <SOURCE_FILE>` | Compute SHA-512/256 hash of Clarity contract source |
| `dump-consts` | Output blockchain constants as JSON (chain IDs, reward maturity, signer slots, etc.) |
| `docgen` | Generate Clarity API reference (all builtins) as JSON |
| `docgen-boot` | Generate boot contract documentation as JSON |
| `exec-program <FILE>` | Execute a Clarity program file |
| `replay-chainstate <OLD_CHAIN> <OLD_SORT> <OLD_BURN> <NEW_CHAIN> <NEW_BURN>` | Replay entire chainstate from old to new database |
| `get-tenure <DIR> <BLOCK_HASH>` | Get all transactions in a block and its microblocks (JSON) |
| `get-block-inventory <DIR>` | Get block inventory (2100 headers from tip) |
| `can-download-microblock <DIR>` | Check microblock download capability |

#### Chainstate Data Requirements

Commands marked with * below do **not** require local chainstate -- they work from user-provided input or connect to peers directly. All others require a local chainstate directory (from running your own node or downloading from `https://archive.hiro.so/mainnet/stacks-blockchain/`).

**Standalone (no chainstate needed):** `decode-tx`, `decode-nakamoto-block`, `decode-net-message`, `peer-pub-key`, `post-stackerdb`, `contract-hash`, `dump-consts`, `docgen`, `docgen-boot`, `exec-program`, `getnakamotoinv`.

**Requires local chainstate:** All database query, shadow block, mining, validation, PoX, and chain state commands.

### stacks-cli (blockstack-cli)

A general-purpose CLI for constructing and inspecting Stacks transactions.

**Source:** `contrib/stacks-cli/src/main.rs`

Key operations:
- Construct signed transactions (token transfers, contract calls, contract deploys)
- Decode raw transactions
- Generate addresses and key pairs
- Convert between address formats

### contrib/tools

Utility scripts in `contrib/tools/`:

| Tool | Description |
|------|-------------|
| `block-validation.sh` | Validate blocks against a running node |
| `chain-upload.sh` | Upload chain data to a node |
| `mempool-pusher.sh` | Push transactions to a node's mempool |
| `config-docs-generator/` | Generate documentation from config structs |

---

## net-test -- Multi-Node Network Testing

The `net-test/` directory contains scripts for running multi-node test networks locally.

**Source:** `net-test/README.md`

### Structure

```
net-test/
  bin/
    start.sh      # Start master, miner, or follower nodes
    faucet.sh     # Bitcoin regtest faucet
    txload.sh     # Transaction load generator
  etc/
    *.in           # Config templates
  tests/           # Test scenarios
```

### Usage

```bash
# Start a master node
./net-test/bin/start.sh master

# Start a miner node connected to the master
./net-test/bin/start.sh miner

# Start a follower
./net-test/bin/start.sh follower

# Fund test accounts
./net-test/bin/faucet.sh <address>

# Generate transaction load
./net-test/bin/txload.sh
```

Prerequisites: `stacks-node`, `bitcoind`, and `bitcoin-cli` must be in your `$PATH`.

---

## Boot Contract Tests

### Unit Tests

`contrib/boot-contracts-unit-tests/` contains focused tests for individual boot contract functions (PoX, BNS, costs).

### Stateful Property Tests

`contrib/boot-contracts-stateful-prop-tests/` contains property-based tests that explore random sequences of stacking operations, verifying invariants hold across arbitrary state transitions.

### Core Contract Tests

`contrib/core-contract-tests/` contains tests for the core protocol contracts that define PoX, name registration, and cost functions.

---

## Test Patterns and Best Practices

### Port Allocation

Integration tests must avoid port conflicts. The test infrastructure tracks used ports in a global `Mutex<HashSet<u16>>` (`stacks-node/src/tests/mod.rs:96`):

```rust
lazy_static! {
    static ref USED_PORTS: Mutex<HashSet<u16>> = Mutex::new({
        let mut set = HashSet::new();
        set.insert(EVENT_OBSERVER_PORT);
        set
    });
}
```

Use `gen_random_port()` or other allocation helpers to avoid collisions when running tests in parallel.

### Common Integration Test Helpers

Key helpers in `stacks-node/src/tests/neon_integrations.rs`:

| Function | Description |
|----------|-------------|
| `neon_integration_test_conf()` | Generate a standard test configuration |
| `next_block_and_wait()` | Mine a Bitcoin block and wait for the Stacks node to process it |
| `submit_tx()` | Submit a transaction to the node's mempool |
| `get_chain_info()` | Query the node's `/v2/info` endpoint |
| `wait_for_runloop()` | Wait for the node's run loop to start |

### Temporary Directories

All test chain state goes under `/tmp/stacks-node-tests/`. Unit tests use `/tmp/stacks-node-tests/units-test-consensus/`, while integration tests use the node's configured `working_dir` (typically `/tmp/stacks-node-{random}`). These directories are not automatically cleaned up -- run periodic cleanup if disk space is a concern.

### Nakamoto Test Setup

Setting up a Nakamoto-era test requires configuring epochs, signers, and stackers. The `TestChainstateConfig` fields make this straightforward:

```rust
let mut config = TestChainstateConfig::default();

// Configure epochs so Nakamoto activates early
config.epochs = Some(EpochList::new(&[
    // ... earlier epochs with low start heights ...
    StacksEpoch { epoch_id: StacksEpochId::Epoch30, start_height: 50, ... },
]));

// Set up test signers (WSTS threshold signing)
config.test_signers = Some(TestSigners::default());

// Set up test stackers with stacking balances
config.test_stackers = Some(vec![TestStacker {
    signer_private_key: ...,
    stacker_private_key: ...,
    amount: 1_000_000_000,
    pox_addr: ...,
    max_amount: None,
}]);
```

For integration tests, the `MultipleMinerTest` struct (used by the signer command framework) handles the full multi-miner setup including starting `bitcoind`, configuring two miner nodes, registering signers, and bootstrapping through epochs.

### TestMiner and TestMinerFactory

The `TestMiner` struct (`stackslib/src/burnchains/tests/mod.rs:112`) represents a simulated miner identity:

```rust
pub struct TestMiner {
    pub burnchain: Burnchain,
    pub privks: Vec<StacksPrivateKey>,
    pub num_sigs: u16,
    pub hash_mode: AddressHashMode,
    pub microblock_privks: Vec<StacksPrivateKey>,
    pub vrf_keys: Vec<VRFPrivateKey>,
    pub vrf_key_map: HashMap<VRFPublicKey, VRFPrivateKey>,
    pub block_commits: Vec<LeaderBlockCommitOp>,
    pub id: usize,
    pub nonce: u64,
    pub spent_at_nonce: HashMap<u64, u128>,
    pub test_with_tx_fees: bool,
    pub chain_id: u32,
}
```

`TestMinerFactory` (`stackslib/src/burnchains/tests/mod.rs:128`) creates multiple `TestMiner` instances with unique identities for fork-testing scenarios where multiple miners compete.

### Environment Variables for Tests

| Variable | Purpose |
|----------|---------|
| `STACKS_NODE_TEST` | Set to enable test-specific code paths |
| `STACKS_LOG_LEVEL` | Control log verbosity (`debug`, `info`, `warn`) |
| `STACKS_EVENT_OBSERVER` | Auto-register an event observer endpoint |
| `STACKS_WORKING_DIR` | Override the node's working directory |

### Testing Tips

- **Use `--nocapture` liberally.** Integration tests produce extensive logging that is invaluable for debugging failures: `cargo test -p stacks-node -- --nocapture test_name`.
- **Clean up `/tmp/stacks-node-tests/` periodically.** Test chain state accumulates and can consume significant disk space.
- **Port conflicts are common.** If tests fail with "address already in use", another test process may be using the port. Use `gen_random_port()` in new tests.
- **Nakamoto tests are slow.** They must boot through all epochs and establish signer sets. Budget 2-5 minutes per test run.
- **The `#[ignore]` attribute** marks slow or resource-intensive tests. Run them with `cargo test -- --ignored`.
