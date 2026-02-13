# Chapter 31: Bitcoin Controller

## Purpose

The `BitcoinRegtestController` is the node's interface to the Bitcoin blockchain. Despite its name suggesting regtest-only use, it handles all Bitcoin interactions for every network mode (mainnet, testnet, regtest). It manages UTXO selection, constructs and signs Bitcoin transactions for Stacks burn operations (block commits, key registrations, stacking operations), implements RBF (Replace-By-Fee) for fee bumping, and tracks ongoing block commits across sortition boundaries.

## Key Concepts

### Burn Operations as Bitcoin Transactions

Stacks miners anchor their work to Bitcoin by submitting **burn operations** -- Bitcoin transactions with OP_RETURN outputs containing Stacks-specific data. The most important is the `LeaderBlockCommitOp`, which commits to a Stacks block and burns STX in the process. The Bitcoin controller is responsible for building these transactions from Stacks-level operation descriptors.

### UTXO Management

The controller must find and manage UTXOs (Unspent Transaction Outputs) belonging to the miner's Bitcoin address. It queries bitcoind via JSON-RPC, selects UTXOs that cover the required amount (fees + burn amount), and constructs transactions with proper change outputs.

### Replace-By-Fee (RBF)

When a new sortition occurs before the previous block commit is confirmed, the miner may need to update its commit. RBF allows replacing an unconfirmed transaction with a higher-fee version. The controller tracks the accumulated fees spent in previous attempts to ensure each replacement meets Bitcoin's RBF rules.

## Architecture Overview

```
Relayer Thread
  |
  +-- BitcoinRegtestController
        |
        +-- BitcoinIndexer (SPV header sync)
        +-- BitcoinRpcClient (JSON-RPC to bitcoind)
        +-- SortitionDB (read-only, for burn chain state)
        +-- BurnchainDB (read-only, for raw burn chain data)
        +-- OngoingBlockCommit (tracks in-flight commits)
```

## Key Data Structures

### BitcoinRegtestController

Defined in `stacks-node/src/burnchains/bitcoin_regtest_controller.rs:86`:

```rust
pub struct BitcoinRegtestController {
    config: Config,
    indexer: BitcoinIndexer,
    db: Option<SortitionDB>,
    burnchain_db: Option<BurnchainDB>,
    chain_tip: Option<BurnchainTip>,
    use_coordinator: Option<CoordinatorChannels>,
    burnchain_config: Option<Burnchain>,
    ongoing_block_commit: Option<OngoingBlockCommit>,
    should_keep_running: Option<Arc<AtomicBool>>,
    rpc_client: Option<BitcoinRpcClient>,
}
```

Key fields:
- **`indexer`**: The `BitcoinIndexer` handles SPV header synchronization and block data retrieval from the Bitcoin P2P network.
- **`ongoing_block_commit`**: Tracks the current in-flight block commit, enabling RBF if a new commit is needed before the previous one confirms.
- **`rpc_client`**: Only `Some` for miner nodes. Follower nodes don't need to submit transactions to bitcoind.
- **`use_coordinator`**: If set, the controller notifies the coordinator when new burn blocks arrive.

### OngoingBlockCommit

Defined at line 103:

```rust
pub struct OngoingBlockCommit {
    pub payload: LeaderBlockCommitOp,
    utxos: UTXOSet,
    fees: LeaderBlockCommitFees,
    txids: Vec<Txid>,
}
```

This tracks everything about a block commit that's been submitted but not yet confirmed:
- **`payload`**: The Stacks-level block commit operation
- **`utxos`**: The Bitcoin UTXOs consumed by this transaction
- **`fees`**: The fee structure, including amounts already spent in previous RBF attempts
- **`txids`**: All transaction IDs submitted for this commit (original + RBF replacements)

### LeaderBlockCommitFees

Defined at line 111:

```rust
struct LeaderBlockCommitFees {
    sunset_fee: u64,
    fee_rate: u64,
    sortition_fee: u64,
    outputs_len: u64,
    default_tx_size: u64,
    spent_in_attempts: u64,
    is_rbf_enabled: bool,
    final_size: u64,
}
```

This struct calculates the total cost of a block commit transaction:
- **`sunset_fee`**: Legacy fee from the PoX sunset period (minimum `DUST_UTXO_LIMIT` = 5500 sats if non-zero)
- **`fee_rate`**: Satoshis per byte for the miner fee, from `config.burnchain.satoshis_per_byte`
- **`sortition_fee`**: The amount burned for sortition (distributed across commit outputs)
- **`spent_in_attempts`**: Cumulative fees from previous RBF attempts
- **`is_rbf_enabled`**: Whether this is an RBF replacement
- **`final_size`**: Actual transaction size after signing

Key methods:

```rust
fn estimated_amount_required(&self) -> u64 {
    self.estimated_miner_fee() + self.rbf_fee() + self.sunset_fee + self.sortition_fee
}

fn total_spent(&self) -> u64 {
    self.fee_rate * self.final_size
        + self.spent_in_attempts
        + self.sunset_fee
        + self.sortition_fee
}
```

The `register_replacement()` method (line 259) updates the fee tracking when an RBF replacement is submitted:

```rust
pub fn register_replacement(&mut self, tx_size: u64) {
    let new_size = cmp::max(tx_size, self.final_size);
    if self.is_rbf_enabled {
        self.spent_in_attempts += new_size;
    }
    self.final_size = new_size;
}
```

## Code Walkthrough: Controller Construction

### For Mining Nodes

`BitcoinRegtestController::with_burnchain()` (line 322) performs full initialization:

1. Creates the burnchain working directory
2. Initializes the SPV client for header verification
3. Validates that custom epochs are not set on mainnet (safety check)
4. Configures the `BitcoinIndexer` with peer host, RPC credentials, timeouts, and magic bytes
5. Creates a `BitcoinRpcClient` for JSON-RPC communication with bitcoind

### For Follower Nodes

Follower nodes set `rpc_client: None` because they never submit transactions. The controller still handles burn block synchronization and sortition processing.

### Dummy Controllers

`new_dummy()` (line 397) creates a controller for transaction submission only (no chain tracking). `new_ongoing_dummy()` creates one pre-loaded with an ongoing block commit for RBF scenarios.

## Code Walkthrough: UTXO Management

The `get_utxos()` method (line 730) retrieves spendable UTXOs for the miner:

1. **Derive the miner address**: Converts the miner's public key to a Bitcoin address appropriate for the current epoch (legacy P2PKH or segwit P2WPKH)
2. **Query bitcoind**: Calls `retrieve_utxo_set()` via RPC to list unspent outputs for that address
3. **Handle empty results**: On regtest, imports the public key into bitcoind's wallet and retries. On mainnet/testnet, assumes the operator has already run `importaddress`.
4. **Validate total**: Checks that the sum of UTXOs meets `total_required`. Returns `None` if insufficient funds.
5. **Exclude used UTXOs**: The optional `utxos_to_exclude` parameter prevents double-spending UTXOs that are already committed in unconfirmed transactions.

### UTXO Cache Staleness

The constant `UTXO_CACHE_STALENESS_LIMIT = 6` (line 79) defines how many Bitcoin blocks can pass before the UTXO cache is force-refreshed. This prevents the controller from working with outdated UTXO information that might lead to double-spend attempts.

### Dust Limit

`DUST_UTXO_LIMIT = 5500` (line 80) is the minimum output value. Outputs below this threshold are considered "dust" by Bitcoin nodes and will be rejected. All burn outputs must meet this minimum.

## Code Walkthrough: Building a Block Commit Transaction

The `build_leader_block_commit_tx()` method (line 1489) constructs the Bitcoin transaction:

1. **Calculate fees**: `LeaderBlockCommitFees::estimated_fees_from_payload()` computes the total required amount from the commit operation
2. **Select UTXOs**: Calls `prepare_tx()` which selects UTXOs and creates the initial transaction skeleton with inputs
3. **Serialize the payload**: The `LeaderBlockCommitOp` is serialized into bytes, prefixed with the burnchain magic bytes
4. **Build outputs**:
   - An OP_RETURN output containing the serialized commit data
   - One output per commit address (the PoX reward addresses that receive the sortition fee)
   - An optional sunset fee output
   - A change output returning excess funds to the miner
5. **Sign the transaction**: Each input is signed with the miner's private key
6. **Record the actual size**: `fees.register_replacement(tx_size)` tracks the real transaction size for future RBF calculations

### OP_RETURN Construction

The OP_RETURN output encodes the burn operation using Bitcoin script:

```
OP_RETURN <magic_bytes><serialized_operation>
```

The magic bytes identify the Stacks network (mainnet vs testnet) and distinguish Stacks transactions from other OP_RETURN data on Bitcoin.

## Code Walkthrough: RBF (Replace-By-Fee)

When a new sortition occurs while a block commit is still unconfirmed, the controller must update it. The RBF flow:

1. **Detect the need for RBF**: The relayer sees a new `ProcessedBurnBlock` directive while `ongoing_block_commit` is `Some`
2. **Calculate new fees**: `fees_from_previous_tx()` (line 182) creates new fee parameters:
   ```rust
   fn fees_from_previous_tx(&self, payload: &LeaderBlockCommitOp, config: &Config)
       -> LeaderBlockCommitFees
   {
       let mut fees = LeaderBlockCommitFees::estimated_fees_from_payload(payload, config);
       fees.spent_in_attempts = cmp::max(1, self.spent_in_attempts);
       fees.final_size = self.final_size;
       fees.fee_rate = self.fee_rate + get_rbf_fee_increment(config);
       fees.is_rbf_enabled = true;
       fees
   }
   ```
   The fee rate increases by `rbf_fee_increment` (from config) with each attempt.
3. **Build replacement transaction**: Same UTXO set, new payload, higher fee
4. **Submit**: The new transaction replaces the old one in the Bitcoin mempool
5. **Track**: The new txid is appended to `ongoing_block_commit.txids`

### RBF Fee Cap

The `max_rbf` config parameter caps the total fee the miner will pay across all RBF attempts. This prevents runaway fee escalation during periods of high Bitcoin congestion.

## Code Walkthrough: Submitting Operations

The `submit_operation()` method (line 2390) is the general-purpose transaction submission path:

1. **Build the transaction**: Dispatches to type-specific builders based on the operation type (`LeaderBlockCommitOp`, `LeaderKeyRegisterOp`, `PreStxOp`, `StackStxOp`, etc.)
2. **Submit via RPC**: Calls `send_raw_transaction()` on the Bitcoin RPC client
3. **Handle errors**: Different error types trigger different behaviors (retry, abort, etc.)

For block commits specifically, `send_block_commit_operation()` (line 1364) handles the special RBF logic and ongoing commit tracking.

## Code Walkthrough: Burn Block Synchronization

The controller syncs burn blocks through two mechanisms:

### For Helium/Regtest

`receive_blocks_helium()` (line 508) uses the deprecated `sync_with_indexer_deprecated()` method which does a full chain sync each time:

```rust
fn receive_blocks_helium(&mut self) -> BurnchainTip {
    let mut burnchain = self.get_burnchain();
    let (block_snapshot, state_transition) = loop {
        match burnchain.sync_with_indexer_deprecated(&mut self.indexer) {
            Ok(x) => break x,
            Err(e) => {
                error!("Unable to sync with burnchain: {e}");
                // retry logic...
            }
        }
    };
    // ...
}
```

### For Production (Neon/Nakamoto)

Production modes use the coordinator-driven approach where the `BitcoinIndexer` syncs headers and the coordinator processes new blocks in order. The controller feeds block data to the coordinator through `CoordinatorChannels`.

## How It Connects

- **Chapter 10 (Bitcoin Integration)**: The burnchain configuration that drives this controller.
- **Chapter 11 (Burn Operations)**: The operation types (`LeaderBlockCommitOp`, `LeaderKeyRegisterOp`, etc.) that the controller serializes into Bitcoin transactions.
- **Chapter 30 (Thread Architecture)**: The controller lives in the relayer thread and is called from the relayer's main loop.
- **Chapter 29 (Node Startup)**: The controller is created during node startup as part of the `NakaRunLoop::start()` sequence.
- **Chapter 19 (Nakamoto Mining)**: The miner triggers block commit submissions through the relayer, which delegates to this controller.

## Edge Cases and Gotchas

1. **Miner-only RPC client**: The `rpc_client` field is `None` for non-miner nodes. Calling `get_rpc_client()` on a follower node panics with "BUG: BitcoinRpcClient is required, but it has not been configured properly!" This is intentional -- followers should never attempt to submit transactions.

2. **Mainnet epoch override protection**: The constructor panics if custom epochs are set while running on mainnet (line 347). This prevents accidental misconfiguration that could cause the node to disagree with the rest of the network about epoch boundaries.

3. **Address import on regtest only**: The UTXO retrieval code only calls `importaddress` on regtest networks (line 770). On mainnet/testnet, operators must manually ensure their bitcoind indexes the miner's address. The comment notes this is because the operation is "very expensive, and can be longer than bitcoin block time."

4. **UTXO exclusion for double-spend prevention**: When building a new transaction while an ongoing commit exists, the previous commit's UTXOs must be excluded (or reused for RBF). The `utxos_to_exclude` parameter in `get_utxos()` handles this.

5. **Transaction size estimation vs actual**: The `default_tx_size` from config is an estimate used for fee calculation before the transaction is built. After signing, `final_size` records the actual size. The `min_tx_size()` method returns the larger of the two, ensuring fee calculations are never under-estimated.

6. **Multiple burn operation types**: Beyond block commits, the controller handles key registrations, pre-STX operations, stack-STX, delegate-STX, transfer-STX, and vote-for-aggregate-key operations. Each has its own transaction builder with specific output structures and fee calculations.

7. **SPV header verification**: The `BitcoinIndexer` includes an SPV (Simplified Payment Verification) client that verifies Bitcoin block headers. This allows the Stacks node to validate burn blocks without running a full Bitcoin node, though production deployments typically connect to a full bitcoind instance.
