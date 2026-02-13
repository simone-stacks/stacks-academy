# Chapter 8: Clarity Database -- The Storage Layer

## Purpose

Every Clarity data map entry, variable, token balance, and contract is persisted through the `ClarityDatabase` and its underlying storage stack. This chapter covers the storage architecture: `ClarityDatabase` as the high-level API, `ClarityBackingStore` as the trait that abstracts over different storage backends, `RollbackWrapper` as the middleware that enables atomic transactions, and how contract state is committed or rolled back during execution.

## Key Concepts

**Three-layer storage stack.** Clarity's storage is a stack of three layers:
1. `ClarityDatabase` -- the high-level API that Clarity execution code calls (e.g., `get_value`, `set_value`, `get_stx_balance`)
2. `RollbackWrapper` -- an in-memory overlay that tracks edits and supports nested begin/commit/rollback
3. `ClarityBackingStore` -- the trait that abstracts the actual persistent storage (MARF-backed on disk, or `MemoryBackingStore` for tests)

**Key-value with Merkle proofs.** All storage is ultimately key-value. Keys are strings constructed by combining a `StoreType` discriminator, the contract identifier, and the data name/key. Values are serialized strings. The backing store can optionally return Merkle proofs alongside values, which is how the MARF (Merklized Adaptive Radix Forest) provides authenticated storage.

**Nested transactions.** The `RollbackWrapper` supports arbitrary nesting depth. Each `contract-call?` nests a new transaction. If the inner call fails, its writes are rolled back without affecting the outer transaction. On success, inner writes are committed to the outer layer.

## Architecture

```
Clarity Interpreter (eval, apply, native functions)
         |
         v
   ClarityDatabase
   - get_value / set_value
   - get_stx_balance / set_stx_balance
   - create_map / get_entry / set_entry
   - insert_contract / get_contract
         |
         v
   RollbackWrapper
   - nest / commit / rollback
   - get / put (with in-memory overlay)
   - metadata operations
         |
         v
   ClarityBackingStore (trait)
   - put_all_data / get_data
   - get_data_with_proof
   - set_block_hash
   - get_block_at_height
         |
         +---> MARF (production)
         +---> MemoryBackingStore (tests)
         +---> NullBackingStore (cost estimation)
```

## Key Data Structures

### ClarityDatabase (`clarity/src/vm/database/clarity_db.rs:135`)

```rust
pub struct ClarityDatabase<'a> {
    pub store: RollbackWrapper<'a>,
    headers_db: &'a dyn HeadersDB,
    burn_state_db: &'a dyn BurnStateDB,
}
```

Three components:
- `store` -- the `RollbackWrapper` for reading/writing contract data
- `headers_db` -- read-only access to Stacks block headers (for `get-block-info?`, `get-stacks-block-info?`, etc.)
- `burn_state_db` -- read-only access to burnchain state (unlock heights, epoch info, PoX constants)

### StoreType (`clarity/src/vm/database/clarity_db.rs:54`)

Every stored value has a type discriminator that prefixes its key:

```rust
pub enum StoreType {
    DataMap = 0x00,
    Variable = 0x01,
    FungibleToken = 0x02,
    CirculatingSupply = 0x03,
    NonFungibleToken = 0x04,
    DataMapMeta = 0x05,
    VariableMeta = 0x06,
    FungibleTokenMeta = 0x07,
    NonFungibleTokenMeta = 0x08,
    Contract = 0x09,
    SimmedBlock = 0x10,
    SimmedBlockHeight = 0x11,
    Nonce = 0x12,
    STXBalance = 0x13,
    PoxSTXLockup = 0x14,
    PoxUnlockHeight = 0x15,
}
```

This prevents key collisions between different data types. A data map named "balance" and a variable named "balance" in the same contract use different `StoreType` prefixes, so their keys never conflict.

### HeadersDB and BurnStateDB Traits

`HeadersDB` (`clarity/src/vm/database/clarity_db.rs:141`) provides block header lookups:

```rust
pub trait HeadersDB {
    fn get_stacks_block_header_hash_for_block(&self, id_bhh: &StacksBlockId, epoch: &StacksEpochId) -> Option<BlockHeaderHash>;
    fn get_burn_header_hash_for_block(&self, id_bhh: &StacksBlockId) -> Option<BurnchainHeaderHash>;
    fn get_consensus_hash_for_block(&self, id_bhh: &StacksBlockId, epoch: &StacksEpochId) -> Option<ConsensusHash>;
    fn get_vrf_seed_for_block(&self, id_bhh: &StacksBlockId, epoch: &StacksEpochId) -> Option<VRFSeed>;
    fn get_burn_block_time_for_block(&self, id_bhh: &StacksBlockId, epoch: Option<&StacksEpochId>) -> Option<u64>;
    fn get_burn_block_height_for_block(&self, id_bhh: &StacksBlockId) -> Option<u32>;
    fn get_miner_address(&self, id_bhh: &StacksBlockId, epoch: &StacksEpochId) -> Option<StacksAddress>;
    // ...
}
```

These methods back the Clarity `get-block-info?`, `get-stacks-block-info?`, and `get-tenure-info?` functions. They are read-only and do not go through `RollbackWrapper`.

`BurnStateDB` (`clarity/src/vm/database/clarity_db.rs:193`) provides burnchain constants and epoch information, including PoX unlock heights and the current epoch's cost limits.

### ClarityBackingStore Trait (`clarity/src/vm/database/clarity_store.rs:53`)

The abstraction over persistent storage:

```rust
pub trait ClarityBackingStore {
    fn put_all_data(&mut self, items: Vec<(String, String)>) -> Result<(), VmExecutionError>;
    fn get_data(&mut self, key: &str) -> Result<Option<String>, VmExecutionError>;
    fn get_data_from_path(&mut self, hash: &TrieHash) -> Result<Option<String>, VmExecutionError>;
    fn get_data_with_proof(&mut self, key: &str) -> Result<Option<(String, Vec<u8>)>, VmExecutionError>;
    fn set_block_hash(&mut self, bhh: StacksBlockId) -> Result<StacksBlockId, VmExecutionError>;
    fn get_block_at_height(&mut self, height: u32) -> Option<StacksBlockId>;
    fn get_current_block_height(&mut self) -> u32;
    fn get_open_chain_tip_height(&mut self) -> u32;
    fn get_open_chain_tip(&mut self) -> StacksBlockId;
    fn make_contract_commitment(&mut self, contract_hash: Sha512Trunc256Sum) -> String;
    fn get_cc_special_cases_handler(&self) -> Option<SpecialCaseHandler>;
}
```

Key methods:
- `put_all_data` -- writes a batch of key-value pairs atomically
- `get_data` / `get_data_with_proof` -- reads a value, optionally with a Merkle proof
- `set_block_hash` -- changes the MARF context to read from a different chain tip (used by `at-block`)
- `get_block_at_height` -- maps a block height to a block ID
- `make_contract_commitment` -- creates a hash commitment for contract deployment (contract content hash + block height)

The `SpecialCaseHandler` is a function pointer for boot contract special cases. Some boot contract calls (like `.pox-4 stack-stx`) have side effects that go beyond normal Clarity execution and are handled by custom Rust code.

### RollbackWrapper (`clarity/src/vm/database/key_value_wrapper.rs:116`)

```rust
pub struct RollbackWrapper<'a> {
    store: &'a mut dyn ClarityBackingStore,
    lookup_map: HashMap<String, Vec<String>>,
    metadata_lookup_map: HashMap<(QualifiedContractIdentifier, String), Vec<String>>,
    stack: Vec<RollbackContext>,
    query_pending_data: bool,
}
```

This is where the transactional magic happens. Key components:

- `store` -- reference to the actual backing store
- `lookup_map` -- a history of edits for each key. Each key maps to a `Vec<String>` where the last element is the most recent value. This enables O(1) lookups of pending writes.
- `metadata_lookup_map` -- same as `lookup_map` but for metadata (contract analysis, etc.)
- `stack` -- a stack of `RollbackContext` values, one per nesting level

#### RollbackContext (`clarity/src/vm/database/key_value_wrapper.rs:111`)

```rust
pub struct RollbackContext {
    edits: Vec<(String, RollbackValueCheck)>,
    metadata_edits: Vec<((QualifiedContractIdentifier, String), RollbackValueCheck)>,
}
```

Tracks which keys were modified at this nesting level. On rollback, these edits are undone by popping values from `lookup_map`. On commit, the edits are transferred to the parent context.

## Code Walkthrough: Nested Transactions

### Nesting (`nest`)

When `contract-call?` begins:

1. `RollbackWrapper.nest()` pushes a new `RollbackContext` onto the stack
2. `GlobalContext` pushes a new `AssetMap` and `read_only` flag
3. `ClarityDatabase.begin()` is called

### Reading (`get`)

When code reads a value:

1. If `query_pending_data` is true, check `lookup_map` first for in-memory edits
2. If not found (or `query_pending_data` is false), query the underlying `ClarityBackingStore`
3. Return the result

### Writing (`put`)

When code writes a value:

1. Push the new value onto `lookup_map[key]`
2. Record the edit in the current `RollbackContext.edits`

### Committing

When a `contract-call?` succeeds:

1. `RollbackWrapper.commit()` pops the current `RollbackContext`
2. The popped context's edits are merged into the parent context's edits
3. The `lookup_map` entries remain (they are still valid -- they just become the parent's responsibility)
4. `GlobalContext` commits the inner `AssetMap` into the outer one

### Rolling Back

When a `contract-call?` fails:

1. `RollbackWrapper.rollback()` pops the current `RollbackContext`
2. For each key in the popped context's `edits`, the most recent value is popped from `lookup_map[key]`, effectively undoing the write
3. `GlobalContext` discards the inner `AssetMap`

### Final Commit (Bottom of Stack)

When the outermost transaction commits (the whole Clarity execution succeeded):

1. All entries in `lookup_map` are collected
2. For each key, the most recent value is taken
3. `ClarityBackingStore.put_all_data()` writes them all to the persistent store in a single batch

This design means the underlying store is never written to during execution -- all writes are buffered in memory and flushed at the end. This is crucial for atomicity.

## ClarityDatabase High-Level Operations

### Data Variables

```
create_variable(contract, name, type, value)  -- at define-data-var time
set_variable(contract, name, value, epoch)     -- at var-set time
get_variable(contract, name, type, epoch)      -- at var-get time
```

Keys are constructed as: `{StoreType::Variable}{contract_identifier}::{variable_name}`

### Data Maps

```
create_map(contract, name, key_type, value_type)
get_entry(contract, name, key, type, epoch)     -- map-get?
set_entry(contract, name, key, value, epoch)    -- map-set
insert_entry(contract, name, key, value, epoch) -- map-insert
delete_entry(contract, name, key, epoch)        -- map-delete
```

Map keys are constructed by serializing the Clarity key value and combining it with the map name and contract identifier.

### STX Balances

```
get_stx_balance_snapshot(principal)  -- returns STXBalanceSnapshot
```

STX balances are complex because they involve liquid balance, locked balance (from stacking), and unlock heights. The `STXBalanceSnapshot` struct provides methods to query and modify the balance while maintaining these invariants.

### Contracts

```
insert_contract(contract_id, contract)  -- store a deployed contract
get_contract(contract_id)               -- retrieve a contract
get_contract_src(contract_id)           -- retrieve source code
```

Contract source code storage is controlled by `STORE_CONTRACT_SRC_INTERFACE` (a compile-time constant, currently `true`).

## MemoryBackingStore (Testing)

`MemoryBackingStore` (in `clarity/src/vm/database/sqlite.rs`, behind the `rusqlite` feature) provides an in-memory SQLite database for testing. It implements `ClarityBackingStore` and stores data in a SQLite in-memory database. It also provides convenience methods:

```rust
impl MemoryBackingStore {
    pub fn as_clarity_db(&mut self) -> ClarityDatabase;
    pub fn as_analysis_db(&mut self) -> AnalysisDatabase;
}
```

This is the standard way to set up Clarity execution in tests.

## The `at-block` Mechanism

The Clarity `at-block` function allows reading state from a historical block. This is implemented through `ClarityBackingStore::set_block_hash`:

1. `at-block` calls `env.global_context.database.set_block_hash(target_block_id)`
2. The backing store changes its MARF context to read from the target block's trie
3. The expressions inside `at-block` are evaluated with reads going to historical state
4. After evaluation, `set_block_hash` is called again to restore the current tip

Writes are not allowed inside `at-block` -- it is enforced as a read-only context.

## MARF Key Patterns for Clarity State

All Clarity state is stored in `chainstate/vm/clarity/marf.sqlite`. The MARF maps string keys to string values. Keys follow a structured naming convention based on the `StoreType` discriminator. The on-disk keys visible in the `data_table` use these patterns:

### Account Keys

| Key Pattern | StoreType | Description |
|-------------|-----------|-------------|
| `vm-account::<principal>::18` | `Nonce` (0x12) | Account transaction nonce |
| `vm-account::<principal>::19` | `STXBalance` (0x13) | STX balance (liquid + locked) |
| `vm-account::<principal>::20` | `PoxSTXLockup` (0x14) | PoX locked STX amount |
| `vm-account::<principal>::21` | `PoxUnlockHeight` (0x15) | Block height at which locked STX unlocks |

Where `<principal>` is a Stacks address (e.g., `SP000000000000000000002Q6VF78`).

### Contract Data Keys

| Key Pattern | StoreType | Description |
|-------------|-----------|-------------|
| `vm::<contract>::0::<map>::<key>` | `DataMap` (0x00) | Data map entry (key is a serialized Clarity value) |
| `vm::<contract>::1::<var>` | `Variable` (0x01) | Data variable |
| `vm::<contract>::2::<ft>` | `FungibleToken` (0x02) | Fungible token balance for this contract |
| `vm::<contract>::3::<ft>` | `CirculatingSupply` (0x03) | FT circulating supply |
| `vm::<contract>::4::<nft>::<id>` | `NonFungibleToken` (0x04) | NFT owner (id is a serialized Clarity value) |

Where `<contract>` is a qualified identifier like `SP000000000000000000002Q6VF78.pox-4`.

### Metadata Keys

| Key Pattern | StoreType | Description |
|-------------|-----------|-------------|
| `vm-metadata::5::<map>` | `DataMapMeta` (0x05) | Data map type signature metadata |
| `vm-metadata::6::<var>` | `VariableMeta` (0x06) | Variable type signature metadata |
| `vm-metadata::9::contract` | `Contract` (0x09) | Contract source code and analysis |

These patterns are useful when querying Clarity state directly with `stacks-inspect marf-get` or when debugging contract data via SQLite. The `StoreType` integer suffix (e.g., `::19` for STX balance) corresponds directly to the enum discriminant shown in the Key Data Structures section above.

## How It Connects

- **Upstream**: Called by the execution layer (Chapter 7). Every `var-get`, `var-set`, `map-get?`, `map-set`, `stx-transfer?`, etc. goes through `ClarityDatabase`.
- **Downstream**: The `ClarityBackingStore` is implemented by the MARF (Chapter 13 in Part 4), which provides Merkle-authenticated persistent storage.
- **Analysis**: The `AnalysisDatabase` (Chapter 6) uses the same `RollbackWrapper` and `ClarityBackingStore` infrastructure, but only stores/retrieves `ContractAnalysis` data.

## Edge Cases and Gotchas

1. **All writes are buffered.** No data hits the persistent store until the entire transaction succeeds and the outermost `commit` is called. This means a contract can perform millions of writes, and if the transaction ultimately fails, none of them are persisted.

2. **Key construction is convention, not type-safe.** Keys are constructed by string concatenation of `StoreType`, contract ID, and data name. A bug in key construction could cause data corruption. The `StoreType` enum discriminator is the primary defense.

3. **`put_all_data` is batched but not necessarily atomic at the store level.** The MARF implementation makes this atomic through its trie commit mechanism, but the `ClarityBackingStore` trait does not formally guarantee atomicity. Test backends may not provide the same guarantees.

4. **`query_pending_data` controls read visibility.** When `true` (the default during execution), reads see in-memory writes from the current transaction. When `false`, reads go directly to the backing store. This flag exists for special cases in the cost-voting mechanism where you need to read the committed state, not the pending state.

5. **Metadata vs. data.** The `RollbackWrapper` has separate tracking for "data" (contract state) and "metadata" (contract analysis, contract source, etc.). Both support rollback, but metadata is stored through separate `insert_metadata` / `get_metadata` methods with different key construction. The metadata path uses `prepare_for_contract_metadata` which writes a contract commitment hash to the MARF.

6. **STX balance involves three components.** An account's STX balance is not a single number -- it is `(amount_unlocked, amount_locked, unlock_height)`. The `STXBalanceSnapshot` struct manages the invariant that `amount_unlocked + amount_locked` does not change during lock/unlock operations. Getting the "effective" balance requires checking whether `unlock_height` has passed.
