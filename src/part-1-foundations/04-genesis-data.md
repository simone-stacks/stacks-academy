# Chapter 4: Genesis Data (stx-genesis)

## Purpose

Every blockchain begins with a genesis block -- the initial state from which all subsequent blocks build. For Stacks, this genesis state is substantial: it includes token balances from the 2017 Blockstack token offering, token lockup schedules (vesting), BNS (Blockchain Naming System) namespaces and names migrated from the Blockstack v1 chain, and their associated zone files.

The `stx-genesis` crate packages all of this data into the compiled binary so that any Stacks node can deterministically reconstruct the genesis chain state without external data sources.

## Key Concepts

Genesis data is **embedded at compile time** using Rust's `include_bytes!` macro, so no external files are needed at runtime. The data includes initial STX balances, token lockup (vesting) schedules, BNS namespace definitions, and BNS name registrations. A SHA-256 integrity check at build time ensures the data has not been tampered with. The `GenesisData` struct provides lazy, iterator-based access to each section via on-demand decompression.

## Architecture

The genesis system has three layers:

1. **Raw data files** (`stx-genesis/chainstate.txt`, `chainstate-test.txt`): Plain-text files containing all genesis data in a sectioned CSV format.
2. **Build script** (`stx-genesis/build.rs`): Compresses the raw data into deflate archives at compile time, verifies SHA-256 integrity, and writes the archives to the build output directory.
3. **Runtime library** (`stx-genesis/src/lib.rs`): Provides iterator-based access to the compressed data via the `GenesisData` struct.

The key insight is that **genesis data is embedded at compile time** using Rust's `include_bytes!` macro. This means the compressed data becomes part of the `stx-genesis` library binary itself -- no external files are needed at runtime.

## Raw Data Format

The raw data lives in `stx-genesis/chainstate.txt` (production) and `stx-genesis/chainstate-test.txt` (test). These files use a PEM-like section format:

```
-----BEGIN NAMESPACES-----
namespace_id,address,buckets,base,coeff,nonalpha_discount,no_vowel_discount,lifetime
blockstack,SP355AMV5R2A5C7VQCF4YPPAS05W93HTB68574W7S,7;6;5;4;3;2;1;1;1;1;1;1;1;1;1;1,4,250,4,4,0
...
-----END NAMESPACES-----
-----BEGIN NAMES-----
name,address,zonefile_hash
0.id,SP1HPCXTGV31W5659M3WTBEFP5AN55HV4B1Q9T31F,c5f5b4197c94d4561f6332160f624d728b0da8bf
...
-----END NAMES-----
-----BEGIN STX BALANCES-----
address,balance
11128v8ZF3YZhhchh72LbKVtjiD1t1JP7S,100000000
...
-----END STX BALANCES-----
-----BEGIN STX VESTING-----
address,amount,block_height
SM37EFPD9ZVR3YRJE7673MJ3W0T350JM1HVZVCDC3,180555557,1
...
-----END STX VESTING-----
```

There are four sections:

| Section | Content |
|---|---|
| **NAMESPACES** | BNS namespace definitions (pricing, lifetime, discount rules) |
| **NAMES** | BNS name registrations (owner address, zonefile hash) |
| **STX BALANCES** | Initial token balances (address, amount in microSTX) |
| **STX VESTING** | Token lockup schedules (address, amount, unlock block height) |

Zone files are stored separately in `stx-genesis/name_zonefiles.txt` as hash-content pairs.

### Test vs Production Data

The test data set (`chainstate-test.txt`) is dramatically smaller -- it contains only a handful of entries in each section. This allows integration tests to run quickly without processing thousands of genesis entries. The production data contains all real-world allocations from the Blockstack token distribution.

## Build Process

The `stx-genesis/build.rs` script runs at compile time and performs three critical steps:

### Step 1: Integrity Verification

```rust
// stx-genesis/build.rs:141-166
fn verify_genesis_integrity(test_data: bool) -> std::io::Result<()> {
    let genesis_data_sha = sha256_digest(open_chainstate_file(test_data));
    let expected_genesis_sha = fs::read_to_string(expected_genesis_sha_file).unwrap();
    if !genesis_data_sha.eq_ignore_ascii_case(&expected_genesis_sha) {
        panic!(
            "FATAL ERROR: chainstate.txt hash mismatch, expected {}, got {}",
            expected_genesis_sha, genesis_data_sha
        );
    }
    // ... writes hash to OUT_DIR for runtime access
}
```

The expected SHA-256 hash is stored in `chainstate.txt.sha256`. If the raw data has been modified (even one byte), the build fails with a panic. This prevents accidental or malicious modification of genesis state.

### Step 2: Section Extraction and Compression

Each section is extracted from the raw file and compressed independently:

```rust
// stx-genesis/build.rs:61-95
fn write_chainstate_archive(
    test_data: bool,
    output_file_name: &str,
    section_name: &str,
) -> std::io::Result<()> {
    // ... opens chainstate file, creates deflate encoder
    for line in reader
        .lines()
        .map(|line| line.unwrap())
        .skip_while(|line| !line.eq(&section_header))
        .skip(2)      // skip the section header and column header
        .take_while(|line| !line.eq(&section_footer))
    {
        encoder.write_all(&[line.as_bytes(), b"\n"].concat())?;
    }
    // ... finalizes deflate stream
}
```

This produces four compressed archives:
- `account_balances.gz` (from "STX BALANCES" section)
- `account_lockups.gz` (from "STX VESTING" section)
- `namespaces.gz` (from "NAMESPACES" section)
- `names.gz` (from "NAMES" section)

And their test equivalents with `-test` suffix.

### Step 3: Embedding in Binary

The compressed archives are written to Cargo's `OUT_DIR` and included in the binary via `include_bytes!` at compile time.

## Key Data Structures

### GenesisData

The main entry point for consuming genesis data:

```rust
// stx-genesis/src/lib.rs:47-56
pub struct GenesisData {
    use_test_chainstate_data: bool,
}

impl GenesisData {
    pub fn new(use_test_chainstate_data: bool) -> GenesisData {
        GenesisData { use_test_chainstate_data }
    }
}
```

The boolean flag selects between production and test data. Each reader method returns a boxed iterator:

```rust
// stx-genesis/src/lib.rs:57-63
pub fn read_balances(&self) -> Box<dyn Iterator<Item = GenesisAccountBalance>> {
    read_balances(if self.use_test_chainstate_data {
        include_bytes!(concat!(env!("OUT_DIR"), "/account_balances-test.gz"))
    } else {
        include_bytes!(concat!(env!("OUT_DIR"), "/account_balances.gz"))
    })
}
```

### GenesisAccountBalance

```rust
// stx-genesis/src/lib.rs:6-11
pub struct GenesisAccountBalance {
    pub address: String,    // STX or BTC address
    pub amount: u64,        // Balance in microSTX
}
```

Addresses can be either Stacks format (`SP...`) or Bitcoin format (base58). When consumed during chain initialization, Bitcoin addresses are converted to their corresponding Stacks addresses.

### GenesisAccountLockup

```rust
// stx-genesis/src/lib.rs:13-20
pub struct GenesisAccountLockup {
    pub address: String,
    pub amount: u64,         // Locked amount in microSTX
    pub block_height: u64,   // Block height when tokens unlock
}
```

Lockups implement the Stacks token vesting schedule. Tokens are locked at genesis and unlock at specified block heights. The `block_height` field is relative to the genesis block (i.e., how many blocks after genesis the tokens become available).

### GenesisNamespace

```rust
// stx-genesis/src/lib.rs:22-31
pub struct GenesisNamespace {
    pub namespace_id: String,
    pub importer: String,       // Address that controls namespace imports
    pub buckets: String,        // Pricing tiers (semicolon-separated)
    pub base: i64,
    pub coeff: i64,
    pub nonalpha_discount: i64,
    pub no_vowel_discount: i64,
    pub lifetime: i64,          // Name lifetime in blocks (0 = permanent)
}
```

BNS namespaces define the pricing and registration rules for names within them. For example, the `.id` namespace has specific pricing tiers based on name length (the `buckets` field encodes price multipliers for 1-letter names, 2-letter names, etc.).

### GenesisName

```rust
// stx-genesis/src/lib.rs:33-37
pub struct GenesisName {
    pub fully_qualified_name: String,  // e.g., "example.id"
    pub owner: String,
    pub zonefile_hash: String,          // 40-character hex (Hash160)
}
```

### GenesisZonefile

```rust
// stx-genesis/src/lib.rs:39-42
pub struct GenesisZonefile {
    pub zonefile_hash: String,
    pub zonefile_content: String,
}
```

Zone files are DNS-like records that map names to resources (URLs, public keys, etc.). They are stored separately from names because they can be large and many names share zone file formats.

## Decompression and Parsing

The internal decompression uses the `libflate` crate's deflate decoder, wrapped in a buffered reader:

```rust
// stx-genesis/src/lib.rs:125-134
fn iter_deflated_csv(deflate_bytes: &'static [u8]) -> Box<dyn Iterator<Item = Vec<String>>> {
    let cursor = io::Cursor::new(deflate_bytes);
    let deflate_decoder = deflate::Decoder::new(cursor);
    let buff_reader = BufReader::new(deflate_decoder);
    let line_iter = buff_reader
        .lines()
        .map(|line| line.unwrap())
        .map(|line| line.split(',').map(String::from).collect());
    Box::new(line_iter)
}
```

Each line is split by comma and the resulting columns are mapped to the appropriate struct. This is lazy -- data is decompressed and parsed on-demand as the iterator is consumed.

Zone files use a different format (line pairs instead of CSV):

```rust
// stx-genesis/src/lib.rs:94-107
struct LinePairReader {
    val: Lines<BufReader<Decoder<Cursor<&'static [u8]>>>>,
}

impl Iterator for LinePairReader {
    type Item = [String; 2];
    fn next(&mut self) -> Option<Self::Item> {
        if let (Some(l1), Some(l2)) = (self.val.next(), self.val.next()) {
            Some([l1.unwrap(), l2.unwrap()])
        } else {
            None
        }
    }
}
```

## How Genesis Data Is Consumed

The actual chain initialization happens in `stackslib/src/chainstate/stacks/db/mod.rs`. The `ChainStateBootData` struct (`mod.rs:961`) collects all genesis data sources:

```rust
// stackslib/src/chainstate/stacks/db/mod.rs:961
pub struct ChainStateBootData {
    pub first_burnchain_block_hash: BurnchainHeaderHash,
    pub first_burnchain_block_height: u32,
    pub first_burnchain_block_timestamp: u32,
    pub initial_balances: Vec<(PrincipalData, u64)>,
    pub pox_constants: PoxConstants,
    pub post_flight_callback: Option<Box<dyn FnOnce(&mut ClarityTx)>>,
    pub get_bulk_initial_lockups:
        Option<Box<dyn FnOnce() -> Box<dyn Iterator<Item = ChainstateAccountLockup>>>>,
    pub get_bulk_initial_balances:
        Option<Box<dyn FnOnce() -> Box<dyn Iterator<Item = ChainstateAccountBalance>>>>,
    pub get_bulk_initial_namespaces:
        Option<Box<dyn FnOnce() -> Box<dyn Iterator<Item = ChainstateBNSNamespace>>>>,
    pub get_bulk_initial_names:
        Option<Box<dyn FnOnce() -> Box<dyn Iterator<Item = ChainstateBNSName>>>>,
}
```

The struct uses closure fields (`get_bulk_initial_*`) that return boxed iterators over genesis data. Here is a typical usage from the test code:

```rust
let mut boot_data = ChainStateBootData {
    initial_balances: vec![],
    first_burnchain_block_hash: BurnchainHeaderHash::zero(),
    first_burnchain_block_height: 0,
    first_burnchain_block_timestamp: 0,
    pox_constants: PoxConstants::testnet_default(),
    post_flight_callback: None,
    get_bulk_initial_lockups: Some(Box::new(|| {
        Box::new(GenesisData::new(true).read_lockups().map(|item| {
            ChainstateAccountLockup {
                address: item.address,
                amount: item.amount,
                block_height: item.block_height,
            }
        }))
    })),
    // ... balances, namespaces, and names similarly
};
```

The boot data closures return iterators that are consumed during genesis block processing. Each balance becomes a MARF entry, each namespace is registered in the BNS contract state, and each lockup is stored in the STX lockup table.

## The Genesis Hash

The SHA-256 hash of the genesis data is accessible at runtime:

```rust
// stx-genesis/src/lib.rs:44-45
pub static GENESIS_CHAINSTATE_HASH: &str =
    include_str!(concat!(env!("OUT_DIR"), "/chainstate.txt.sha256"));
```

This hash is used to verify that all nodes are working from the same genesis state. If two nodes compute different genesis hashes, they are on incompatible chains.

## How It Connects

- **Chapter 1** (Architecture) shows where `stx-genesis` fits in the crate dependency graph.
- **Chapter 2** (Common Types) provides the `StacksAddress` and `Hash160` types used to represent genesis addresses.
- **Chapter 13** (MARF) is where genesis balances are actually stored as initial trie entries.
- **Chapter 15** (Block Processing) covers the genesis block construction that consumes this data.
- **Chapter 16** (Stacking and PoX) uses the lockup data for initial vesting schedules.

## Edge Cases and Gotchas

1. **BTC addresses in genesis data**: The raw data contains both STX addresses (C32 format, `SP...`) and BTC addresses (base58 format). The chain initialization code must convert BTC addresses to their STX equivalents. This is why `GenesisAccountBalance.address` is a `String` rather than a typed `StacksAddress`.

2. **Burn address entries**: The genesis data includes entries for the burn address (`1111111111111111111114oLvT2`), which receives tokens that were burned during the Blockstack token sale. These appear as normal balance entries.

3. **Test data is much smaller**: The test chainstate file (`chainstate-test.txt`) contains only a few dozen entries. Do not rely on it for realistic load testing. The production file contains thousands of balances and hundreds of BNS names.

4. **Integrity is checked at compile time, not runtime**: If the SHA-256 hash of `chainstate.txt` doesn't match `chainstate.txt.sha256`, the build panics. There is no runtime fallback. This means a corrupted genesis file prevents compilation entirely.

5. **Lockup block heights are relative**: The `block_height` field in `GenesisAccountLockup` is the number of blocks *after* the genesis block. It is not an absolute Stacks or Bitcoin block height. The lockup processing code adds this offset to the genesis block height.

6. **Zone file escaping**: Zone file content may contain newlines, which are escaped as `\n` in the raw data. The parser converts these back: `pair[1].replace("\\n", "\n")` at `lib.rs:120`.

7. **No runtime dependency on stacks-common**: The `stx-genesis` crate only depends on `libflate` (for deflate decompression). It does not depend on `stacks-common`. This means it cannot use any Stacks-specific types -- addresses are plain strings, and amounts are plain `u64`. The conversion to typed structures happens in the consuming code in `stackslib`.
