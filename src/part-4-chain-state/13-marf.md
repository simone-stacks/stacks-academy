# Chapter 13: The MARF (Merklized Adaptive Radix Forest)

## Purpose

The MARF -- Merklized Adaptive Radix Forest -- is the authenticated state index that underpins the entire Stacks blockchain. Every piece of chain state (account balances, smart contract data, block headers) lives inside a MARF. The MARF provides three critical capabilities that a naive key-value store cannot:

1. **Merkle proofs**: Any value can be proven to exist (or not exist) at a given block height by presenting a compact proof against the root hash.
2. **Historical lookups**: You can read the value of any key as it was at any previous block, without replaying transactions.
3. **Efficient rollbacks**: Fork handling is built into the data structure -- switching to a different chain tip is an O(1) pointer change, not a replay.

The name "Forest" (not "Tree") is important: the MARF is a collection of tries, one per block. Each new block's trie shares structure with its parent via back-pointers, making the entire history accessible through a single unified index.

## Key Concepts

### Adaptive Radix Trie

The MARF uses an adaptive radix trie -- the same general idea as the ART data structure from database research. A radix trie indexes keys by their byte-level content: each level of the tree corresponds to one byte of the key. An "adaptive" radix trie varies the node fanout based on how many children actually exist:

- **Node4**: Up to 4 children (compact for sparse regions)
- **Node16**: Up to 16 children
- **Node48**: Up to 48 children (uses an index array for O(1) lookup)
- **Node256**: Up to 256 children (one slot per possible byte value)
- **Leaf**: Stores the actual value

Nodes are promoted (e.g., Node4 -> Node16) when they fill up. This keeps memory usage proportional to the actual data, not the theoretical key space.

### Path Compression

Each node carries a `path` field: a compressed segment of the key that all children share. For example, if every key under a node starts with the bytes `[0xab, 0xcd]`, those bytes are stored in the node's `path` rather than consuming two levels of the trie. This drastically reduces trie depth for real-world workloads.

### The Forest: Back-Pointers and Copy-on-Write

When a new block is processed, the MARF creates a new trie that shares most of its structure with the parent trie via **back-pointers**. Only the nodes that change are copied (copy-on-write semantics). A back-pointer in a `TriePtr` is indicated by the high bit (0x80) of the node ID:

```rust
// stackslib/src/chainstate/stacks/index/node.rs:67-79
pub fn is_backptr(id: u8) -> bool {
    id & 0x80 != 0
}

pub fn set_backptr(id: u8) -> u8 {
    id | 0x80
}

pub fn clear_backptr(id: u8) -> u8 {
    id & 0x7f
}
```

When walking the trie and encountering a back-pointer, the MARF follows it to the referenced block's trie to continue the lookup. This is how historical reads work: the current trie delegates to older tries for data that has not changed.

## Architecture

The MARF implementation lives in `stackslib/src/chainstate/stacks/index/` and is organized into these modules:

| Module | Responsibility |
|--------|---------------|
| `mod.rs` | Core types: `MARFValue`, `TrieLeaf`, `MarfTrieId`, `TrieMerkleProof`, error types |
| `node.rs` | Trie node types (`TrieNode4/16/48/256`), `TriePtr`, `TrieCursor`, `TrieNodeType` enum |
| `trie.rs` | Single-trie operations: read, insert, walk, hash computation |
| `marf.rs` | Forest-level operations: `MARF` struct, `MarfTransaction`, cross-trie lookups |
| `proofs.rs` | Merkle proof generation and verification |
| `storage.rs` | `TrieFileStorage`, SQLite-backed persistence, block-to-trie mapping |
| `trie_sql.rs` | SQL schema and queries for trie persistence |
| `bits.rs` | Low-level byte encoding/decoding for nodes and paths |
| `cache.rs` | In-memory caching layer (`TrieCache`) |
| `file.rs` | External blob storage for trie data |
| `profile.rs` | Benchmarking instrumentation |

## Key Data Structures

### MARFValue

The leaf payload -- a 40-byte array storing the hash of the actual value plus 8 reserved bytes:

```rust
// stackslib/src/chainstate/stacks/index/mod.rs:153
pub struct MARFValue(pub [u8; 40]);
```

The first 32 bytes hold a `TrieHash` (SHA-512/256 of the value string). The remaining 8 bytes are reserved. The MARF stores hashes, not raw values -- the actual data lives in a side-store (typically the Clarity MARF's SQLite database).

### TrieLeaf

A leaf node pairing a path suffix with its value:

```rust
// stackslib/src/chainstate/stacks/index/mod.rs:83-87
pub struct TrieLeaf {
    pub path: Vec<u8>,   // remaining path bytes (lazily expanded)
    pub data: MARFValue, // the stored value hash
}
```

### TriePtr -- Child Pointer

Every internal node holds pointers to children via `TriePtr`:

```rust
// stackslib/src/chainstate/stacks/index/node.rs:187-193
pub struct TriePtr {
    pub id: u8,         // Node type ID; high bit indicates back-pointer
    pub chr: u8,        // The byte value this child is indexed under
    pub ptr: u32,       // Offset within the storage blob
    pub back_block: u32, // Block ID if this is a back-pointer (0 otherwise)
}
```

The `id` field doubles as a node type discriminator and a back-pointer flag. When `id & 0x80 != 0`, the child lives in a different block's trie, identified by `back_block`.

### TrieNodeType

The unified enum over all node variants:

```rust
// stackslib/src/chainstate/stacks/index/node.rs:1227-1233
pub enum TrieNodeType {
    Node4(TrieNode4),
    Node16(TrieNode16),
    Node48(Box<TrieNode48>),
    Node256(Box<TrieNode256>),
    Leaf(TrieLeaf),
}
```

Node48 and Node256 are boxed because they are large (Node256 contains a 256-element array of `TriePtr`).

### TrieCursor

Tracks state while walking through the trie:

```rust
// stackslib/src/chainstate/stacks/index/node.rs:323-331
pub struct TrieCursor<T: MarfTrieId> {
    pub path: TrieHash,              // the full key path being walked
    pub index: usize,                // position in the path
    pub node_path_index: usize,      // position in current node's compressed path
    pub nodes: Vec<TrieNodeType>,    // nodes visited
    pub node_ptrs: Vec<TriePtr>,     // pointers taken
    pub block_hashes: Vec<T>,        // which tries were visited
    pub last_error: Option<CursorError>,
}
```

The cursor records every node and pointer visited during a walk. This history is used when inserting new nodes (to know where to split paths) and when computing Merkle proofs (to gather sibling hashes).

### MARF Struct

The top-level forest:

```rust
// stackslib/src/chainstate/stacks/index/marf.rs:40-43
pub struct MARF<T: MarfTrieId> {
    storage: TrieFileStorage<T>,
    open_chain_tip: Option<WriteChainTip<T>>,
}
```

`TrieFileStorage` manages the SQLite database and optional external blob file. `open_chain_tip` tracks the current write session (block being built).

### TrieFileStorage

The persistence layer:

```rust
// stackslib/src/chainstate/stacks/index/storage.rs:1256-1270
pub struct TrieFileStorage<T: MarfTrieId> {
    pub db_path: String,
    db: Connection,               // SQLite connection
    blobs: Option<TrieFile>,      // Optional external blob storage
    data: TrieStorageTransientData<T>,
    cache: TrieCache<T>,          // In-memory node/hash cache
    bench: TrieBenchmark,
    hash_calculation_mode: TrieHashCalculationMode,
}
```

## Code Walkthrough: How a Key Lookup Works

### Step 1: Hash the Key

All MARF keys are strings. Before lookup, the key is hashed to produce a 32-byte `TrieHash` path:

```rust
// stacks-common, TrieHash::from_key
let path = TrieHash::from_key(key);  // SHA-512/256 of the key string
```

This hash becomes the path through the trie. Hashing ensures uniform distribution and fixed-length paths.

### Step 2: Open the Target Block

The MARF positions its storage to the requested block's trie:

```rust
// stackslib/src/chainstate/stacks/index/marf.rs:141-143
fn get(&mut self, block_hash: &T, key: &str) -> Result<Option<MARFValue>, Error> {
    self.with_conn(|c| MARF::get_by_key(c, block_hash, key))
}
```

### Step 3: Walk the Trie with TrieCursor

The `TrieCursor::walk` method (node.rs:406-492) consumes the path byte by byte:

1. **Consume the node's compressed path**: Compare the node's `path` bytes against the lookup path. If they diverge, the key does not exist.
2. **Follow the child pointer**: Use the next path byte to find the child via `node.walk(chr)`.
3. **Handle back-pointers**: If the child pointer has bit 0x80 set, follow it to the referenced block's trie (`MARF::walk_backptr`).
4. **Repeat** until a Leaf is reached or the path is exhausted.

### Step 4: Return the Value

If the walk reaches a `TrieLeaf` whose path matches, the `MARFValue` is returned. The caller can then look up the actual data in the side-store using this hash.

## MARF Proofs

The MARF supports Merkle inclusion proofs via `TrieMerkleProof` (defined in `proofs.rs`). A proof consists of a sequence of `TrieMerkleProofType` entries -- one per node visited on the path from root to leaf:

```rust
// stackslib/src/chainstate/stacks/index/mod.rs:44
pub struct TrieMerkleProof<T: MarfTrieId>(pub Vec<TrieMerkleProofType<T>>);

// stackslib/src/chainstate/stacks/index/mod.rs:56-63
pub enum TrieMerkleProofType<T> {
    Node4((u8, ProofTrieNode<T>, [TrieHash; 3])),
    Node16((u8, ProofTrieNode<T>, [TrieHash; 15])),
    Node48((u8, ProofTrieNode<T>, [TrieHash; 47])),
    Node256((u8, ProofTrieNode<T>, [TrieHash; 255])),
    Leaf((u8, TrieLeaf)),
    Shunt((i64, Vec<TrieHash>)),
}
```

Each node entry includes:
- The byte value (`chr`) of the child that was followed
- The node itself (with block hashes for back-pointers, via `ProofTrieNode`)
- The hashes of all **sibling** children (so the verifier can reconstruct the parent hash)

The `Shunt` variant handles back-pointers: when the proof crosses from one block's trie into another, the shunt records which block boundary was crossed and the intermediate hashes needed to verify continuity.

To verify a proof, a client walks the proof entries from leaf to root, recomputing each node's hash from the provided siblings and checking that the final hash matches the block's `state_index_root`.

## Block Height Mappings

The MARF stores internal metadata alongside application data. Three special keys track block height relationships:

```rust
// stackslib/src/chainstate/stacks/index/marf.rs:35-37
pub const BLOCK_HASH_TO_HEIGHT_MAPPING_KEY: &str = "__MARF_BLOCK_HASH_TO_HEIGHT";
pub const BLOCK_HEIGHT_TO_HASH_MAPPING_KEY: &str = "__MARF_BLOCK_HEIGHT_TO_HASH";
pub const OWN_BLOCK_HEIGHT_KEY: &str = "__MARF_BLOCK_HEIGHT_SELF";
```

When a new block extends the MARF (`MarfTransaction::begin`), the `set_block_heights` method (marf.rs:463-523) inserts mappings for both the new block and its parent, enabling lookups like "what block was at height N?" and "what height is block X at?".

## Rollbacks and Fork Handling

Rollbacks are a natural consequence of the MARF's copy-on-write design. Each block's trie is an independent entity linked to its parent via back-pointers. Switching forks means:

1. **Drop the current trie**: `MarfTransaction::drop_current()` discards the in-progress trie and resets storage to the sentinel block.
2. **Open a different chain tip**: The MARF can `open_block` to any previously committed block, instantly making that block's state visible.

No data is destroyed when dropping an uncommitted trie. Committed tries are permanent and immutable.

The `check_ancestor_block_hash` method (marf.rs:198-232) verifies that two blocks are on the same fork by checking height mappings, preventing cross-fork state confusion.

## Hash Calculation Modes

The MARF supports two hash calculation modes:

```rust
// stackslib/src/chainstate/stacks/index/storage.rs
pub enum TrieHashCalculationMode {
    Immediate,  // Hash nodes as they are written
    Deferred,   // Defer hashing until seal() is called
}
```

**Deferred mode** (the default) skips hash computation during writes and computes all hashes in a single pass when `seal()` is called. This is significantly faster for batch operations like block processing, where many keys are written before the final root hash is needed.

## MARF Key Patterns for Headers Index

The headers MARF (`chainstate/vm/index.sqlite`) stores chain state metadata alongside application data. In Nakamoto, the following key patterns are used to track tenures and blocks:

### Tenure Keys

| Key Pattern | Description |
|-------------|-------------|
| `nakamoto::tenures::ongoing_tenure_id` | Current ongoing tenure (consensus_hash + block_id) |
| `nakamoto::tenures::ongoing_tenure_coinbase_height::<height>` | Block ID at a given coinbase height |
| `nakamoto::tenures::block_found_tenure_id::<ch>` | Block-found block ID for a tenure |
| `nakamoto::tenures::highest_block_in_tenure::<ch>` | Highest block ID produced in a tenure |
| `nakamoto::tenures::finished_tenure_consensus_hash::<ch>` | Final block ID of a completed tenure |
| `nakamoto::tenures::parent_tenure_consensus_hash::<ch>` | Parent tenure's consensus hash |

### Header Keys

| Key Pattern | Description |
|-------------|-------------|
| `nakamoto::headers::coinbase_height::<ch>` | Coinbase height (hex u64) at a given consensus hash |
| `nakamoto::headers::tenure_start_block_id::<ch>` | Start block ID for a tenure |

Where `<ch>` is a consensus hash (40 hex characters) and `<height>` is a u64.

### Internal Keys

| Key | Description |
|-----|-------------|
| `__MARF_BLOCK_HASH_TO_HEIGHT` | Maps a block hash to its height |
| `__MARF_BLOCK_HEIGHT_TO_HASH` | Maps a block height to its hash |
| `__MARF_BLOCK_HEIGHT_SELF` | This block's own height |

These keys can be queried directly using `stacks-inspect header-indexed-get` (with `--state-dir` pointing to the chainstate directory) or `stacks-inspect marf-get` (with the path to `index.sqlite`). See Appendix D for command details.

## How It Connects

- **Clarity VM** (Chapters 5-9): The Clarity database (`MarfedKV`) uses a MARF as its backing store. Every `define-data-var`, `define-map`, and account balance is a MARF entry.
- **Block Processing** (Chapter 15): `StacksChainState.state_index` is a `MARF<StacksBlockId>` that indexes block headers and metadata.
- **Sortition DB** (Chapter 12): The sortition DB uses a separate MARF (`MARF<SortitionId>`) to track burnchain state.
- **State Proofs**: The `state_index_root` in every `StacksBlockHeader` is the root hash of the MARF after processing that block. Light clients verify state proofs against this root.

## Edge Cases and Gotchas

1. **Sentinel block**: The very first trie has no parent. The MARF uses a sentinel value (`[0xff; 32]`) as the "parent" of the genesis trie. Code that walks back-pointers must handle this sentinel.

2. **Unconfirmed state**: The MARF supports an "unconfirmed" mode for mempool processing, where a temporary trie can be created and destroyed without affecting committed state. The unconfirmed tip has a synthetic block hash to avoid collisions.

3. **Node256 as root**: The root node of every trie is a `TrieNode256`. When reading the root, the code first tries to read it as a back-pointer (`set_backptr(Node256)`), falling back to a direct Node256 if that fails (`Trie::read_root_maybe_hash` in trie.rs:73-101).

4. **40-byte values**: `MARFValue` is 40 bytes, not 32. The first 32 bytes are the value hash; the last 8 are reserved. Code converting between `MARFValue` and block hashes (`From<MARFValue> for StacksBlockId`) will panic if non-zero data appears in the reserved bytes.

5. **SQLite and blobs**: Trie node data can be stored either in the SQLite database or in an external flat file (`TrieFile`), controlled by `MARFOpenOpts.external_blobs`. The external blob option reduces SQLite WAL size at the cost of more complex storage management.
