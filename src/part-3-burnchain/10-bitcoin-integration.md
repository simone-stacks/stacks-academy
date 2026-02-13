# Chapter 10: Bitcoin Integration

## Purpose

Stacks does not run its own proof-of-work consensus. Instead, it anchors to Bitcoin: every Stacks sortition corresponds to a Bitcoin block, every miner commitment is a Bitcoin transaction, and every PoX reward is a Bitcoin payout. The `burnchains/bitcoin` module is the bridge that makes this possible. It implements an SPV (Simplified Payment Verification) client that downloads and validates Bitcoin block headers, a peer-to-peer network layer that speaks the Bitcoin wire protocol, an indexer that orchestrates header sync and block fetching, and a parser that extracts Stacks-relevant operations from raw Bitcoin transactions.

Understanding this module is essential because it defines the trust boundary between Stacks and Bitcoin. If the SPV client accepts a fraudulent header chain, or if the parser misidentifies a transaction, the entire sortition history is compromised.

## Key Concepts

**Burnchain abstraction.** Although the code is structured to support multiple burnchains, Bitcoin is the only one implemented. The `BurnchainTransaction` and `BurnchainBlock` enums in `stackslib/src/burnchains/mod.rs` wrap their Bitcoin-specific variants:

```rust
// stackslib/src/burnchains/mod.rs:173-177
pub enum BurnchainTransaction {
    Bitcoin(BitcoinTransaction),
    // TODO: fill in more types as we support them
}
```

**Magic bytes.** Stacks operations are identified inside Bitcoin transactions by a two-byte magic prefix in the `OP_RETURN` data. On mainnet, this is `id` (bytes `[105, 100]`), defined at `stackslib/src/burnchains/mod.rs:73`:

```rust
pub const BLOCKSTACK_MAGIC_MAINNET: MagicBytes = MagicBytes([105, 100]); // 'id'
```

**Epoch-gated parsing.** The parser's behavior changes across Stacks epochs. Before Epoch 2.1, only legacy Bitcoin address types (p2pkh, p2sh) were accepted in transaction inputs and outputs. Starting with Epoch 2.1, segwit (p2wpkh, p2wsh, p2tr) outputs and raw (unparsed) inputs are accepted.

## Architecture

The Bitcoin integration layer has five main components, each in its own file under `stackslib/src/burnchains/bitcoin/`:

1. **SPV Client** (`spv.rs`) -- Downloads and validates Bitcoin block headers using an SQLite database. Verifies proof-of-work, difficulty adjustments, and chain continuity.

2. **Indexer** (`indexer.rs`) -- Orchestrates the sync process. Manages the TCP connection to a Bitcoin peer, coordinates SPV header sync, triggers block downloads, and exposes the `BurnchainIndexer` trait implementation.

3. **Block Parser** (`blocks.rs`) -- Extracts Stacks operations from raw Bitcoin blocks. Identifies `OP_RETURN` outputs with the correct magic bytes, parses opcodes and payloads, and constructs `BitcoinTransaction` objects.

4. **Script Parser** (`bits.rs`) -- Low-level Bitcoin script parsing. Extracts public keys from scriptSigs and witness data, handles p2pkh, p2sh, p2wpkh-over-p2sh, and p2wsh-over-p2sh input types.

5. **Address Module** (`address.rs`) -- Bitcoin address types: legacy (p2pkh, p2sh) and segwit (p2wpkh, p2wsh, p2tr). Handles encoding/decoding between script pubkeys and address representations.

The flow is: **Indexer** drives the **SPV Client** to sync headers, then uses the **Block Downloader** to fetch full blocks, which are handed to the **Block Parser** to extract operations. The parser calls into `bits.rs` for script analysis and `address.rs` for address resolution.

## Key Data Structures

### BurnchainParameters

Defined at `stackslib/src/burnchains/mod.rs:76-86`, this captures the static configuration for a burnchain network:

```rust
pub struct BurnchainParameters {
    chain_name: String,              // "bitcoin"
    network_name: String,            // "mainnet", "testnet", "regtest"
    network_id: u32,                 // wire protocol magic number
    stable_confirmations: u32,       // 7 for mainnet/testnet, 1 for regtest
    consensus_hash_lifetime: u32,    // 24 blocks
    pub first_block_height: u64,     // first Bitcoin block Stacks monitors
    pub first_block_hash: BurnchainHeaderHash,
    pub first_block_timestamp: u32,
    pub initial_reward_start_block: u64,
}
```

The `stable_confirmations` field is critical: it defines how many Bitcoin confirmations are required before the Stacks node considers a block "stable." This is 7 on mainnet, meaning sortitions lag 7 blocks behind the Bitcoin tip.

### Burnchain

The runtime configuration struct at `stackslib/src/burnchains/mod.rs:258-272`:

```rust
pub struct Burnchain {
    pub peer_version: u32,
    pub network_id: u32,
    pub chain_name: String,
    pub network_name: String,
    pub working_dir: String,
    pub consensus_hash_lifetime: u32,
    pub stable_confirmations: u32,
    pub first_block_height: u64,
    pub first_block_hash: BurnchainHeaderHash,
    pub first_block_timestamp: u32,
    pub pox_constants: PoxConstants,
    pub initial_reward_start_block: u64,
}
```

This struct is used throughout the codebase to parameterize burnchain-aware logic. The embedded `PoxConstants` struct controls reward cycle length, prepare phase length, PoX activation heights, and unlock heights for each PoX version.

### SpvClient

Defined at `stackslib/src/burnchains/bitcoin/spv.rs:110-120`:

```rust
pub struct SpvClient {
    pub headers_path: String,           // path to SQLite DB
    pub start_block_height: u64,        // start of current sync range
    pub end_block_height: Option<u64>,  // end of current sync range
    pub cur_block_height: u64,          // current sync progress
    pub network_id: BitcoinNetworkType, // mainnet/testnet/regtest
    readwrite: bool,                    // read-only or read-write mode
    reverse_order: bool,                // fetch headers newest-first
    headers_db: DBConn,                 // SQLite connection
    check_txcount: bool,               // validate tx_count == 0 in headers
}
```

The SPV client stores headers in SQLite with the schema (`spv.rs:94-108`):

```sql
CREATE TABLE headers(
    version INTEGER NOT NULL,
    prev_blockhash TEXT NOT NULL,
    merkle_root TEXT NOT NULL,
    time INTEGER NOT NULL,
    bits INTEGER NOT NULL,
    nonce INTEGER NOT NULL,
    height INTEGER PRIMARY KEY NOT NULL,
    hash TEXT NOT NULL
);
```

A separate `chain_work` table stores cumulative work scores per 2016-block difficulty interval, enabling efficient validation that new headers represent more work than existing ones.

### BitcoinTransaction

The parsed representation of a Bitcoin transaction relevant to Stacks, at `stackslib/src/burnchains/bitcoin/mod.rs:216-226`:

```rust
pub struct BitcoinTransaction {
    pub txid: Txid,
    pub vtxindex: u32,       // position within the block
    pub opcode: u8,          // Stacks operation code (e.g., '[' for block commit)
    pub data: Vec<u8>,       // OP_RETURN payload after magic bytes and opcode
    pub data_amt: u64,       // satoshis sent to OP_RETURN output
    pub inputs: Vec<BitcoinTxInput>,
    pub outputs: Vec<BitcoinTxOutput>,
}
```

### BitcoinTxInput

Two variants exist, reflecting the epoch-gated parsing (`mod.rs:198-202`):

```rust
pub enum BitcoinTxInput {
    Structured(BitcoinTxInputStructured),  // pre-2.1: parsed keys
    Raw(BitcoinTxInputRaw),               // 2.1+: raw scriptSig/witness
}
```

The `Structured` variant extracts actual public keys from the script, which was necessary for sender identification in early epochs. The `Raw` variant simply stores the raw bytes, since Epoch 2.1 identifies senders differently.

### BitcoinAddress

Three address families are supported (`address.rs:34-61`):

```rust
pub enum BitcoinAddress {
    Legacy(LegacyBitcoinAddress),  // p2pkh, p2sh
    Segwit(SegwitBitcoinAddress),  // p2wpkh, p2wsh, p2tr
}

pub enum SegwitBitcoinAddress {
    P2WPKH(BitcoinNetworkType, [u8; 20]),
    P2WSH(BitcoinNetworkType, [u8; 32]),
    P2TR(BitcoinNetworkType, [u8; 32]),
}
```

## Code Walkthrough

### SPV Header Validation

When the SPV client receives headers, it validates them in `handle_headers` (`spv.rs:813-884`):

1. **Record chain work before insertion** via `update_chain_work()`.
2. **Insert headers** using either `insert_block_headers_after` (ascending) or `insert_block_headers_before` (descending), depending on `self.reverse_order`.
3. **Validate proof-of-work** by calling `validate_header_work`, which iterates over each 2016-block difficulty interval and checks:
   - Each header's timestamp exceeds the median of the prior 11 blocks (`spv.rs:588-605`).
   - Each header's `bits` field matches the expected difficulty target.
   - Each header hash meets the required difficulty.
4. **Verify chain work did not decrease** by comparing `update_chain_work()` before and after.

The difficulty target calculation in `get_target` (`spv.rs:1073-1152`) handles network-specific rules: regtest uses a fixed maximum target, testnet allows minimum-difficulty blocks when timestamps are too far apart, and mainnet follows standard Bitcoin retargeting.

### Parsing Stacks Operations from Bitcoin Transactions

The `BitcoinBlockParser::parse_data` method (`blocks.rs:258-299`) extracts Stacks payloads:

```rust
fn parse_data(&self, data_output: &Script) -> Option<(u8, Vec<u8>)> {
    // Must be OP_RETURN
    // Must have exactly 2 script instructions: OP_RETURN <data>
    // Data must start with magic bytes
    // After magic bytes: opcode byte + payload
    let opcode = *data.get(MAGIC_BYTES_LENGTH)?;
    let data = data.get(MAGIC_BYTES_LENGTH + 1..)?.to_vec();
    Some((opcode, data))
}
```

The `maybe_burnchain_tx` method (`blocks.rs:304-345`) checks whether a Bitcoin transaction is a candidate Stacks operation:
- Output 0 must be a parseable `OP_RETURN` with valid magic bytes.
- All remaining outputs must be recognized address types (legacy pre-2.1, any decodable type post-2.1).

Transaction parsing then proceeds through `parse_tx` which constructs a `BitcoinTransaction` with the opcode, payload, parsed inputs, and parsed outputs.

### Input Script Parsing

The `bits.rs` module handles multiple Bitcoin script formats. For pre-2.1 (structured parsing), the priority order is:

1. **p2pkh** (`from_bitcoin_p2pkh_script_sig`, `bits.rs:41-76`): Two pushdata instructions -- signature + public key.
2. **p2sh multisig** (`from_bitcoin_p2sh_multisig_script_sig`, `bits.rs:252-300`): `OP_0 <sig1> ... <sigm> <redeem_script>` where the redeem script contains `OP_m <pk1> ... <pkn> OP_n OP_CHECKMULTISIG`.
3. **p2wpkh-over-p2sh** (`from_bitcoin_p2wpkh_p2sh_script_sig`, `bits.rs:303-350`): ScriptSig is a 22-byte witness program hash, witness contains `<sig> <pubkey>`.
4. **p2wsh-over-p2sh multisig** (`from_bitcoin_p2wsh_p2sh_multisig_script_sig`, `bits.rs:353-415`): ScriptSig is a 34-byte witness program hash, witness contains signatures plus a multisig redeem script.

For post-2.1 (raw parsing), `from_bitcoin_txin_raw` (`bits.rs:487-493`) simply stores the raw scriptSig and witness bytes.

### BitcoinIndexer Connection Flow

The `BitcoinIndexer` (`indexer.rs:144-148`) manages the TCP connection:

```rust
pub struct BitcoinIndexer {
    pub config: BitcoinIndexerConfig,
    pub runtime: BitcoinIndexerRuntime,
    pub should_keep_running: Option<Arc<AtomicBool>>,
}
```

The `BitcoinIndexerConfig` (`indexer.rs:103-129`) contains peer connection details:
- `peer_host` / `peer_port`: Bitcoin node address for the P2P protocol
- `rpc_port` / `rpc_ssl`: Bitcoin RPC interface for fetching blocks
- `spv_headers_path`: path to the SPV SQLite database
- `magic_bytes`: the two-byte Stacks operation prefix
- `epochs`: optional custom epoch definitions

The indexer implements the `BurnchainIndexer` trait, providing the `sync_headers`, `read_headers`, and `downloader`/`parser` factory methods that the higher-level burnchain sync loop uses.

## How It Connects

- **Chapter 11 (Burn Operations)**: The `BitcoinTransaction` objects produced here are consumed by operation parsers like `LeaderBlockCommitOp::from_tx` and `LeaderKeyRegisterOp::from_tx`.
- **Chapter 12 (SortitionDB)**: The `BurnchainBlockHeader` values derived from SPV-validated headers are stored in the sortition database and used to compute consensus hashes.
- **Chapter 20 (Coordinator)**: The coordinator drives the burnchain sync loop, calling into the indexer to fetch new blocks and feeding them to the sortition processor.
- **Chapter 16 (PoX)**: The `PoxConstants` embedded in `Burnchain` parameterize reward cycle computation throughout the system.

## Edge Cases and Gotchas

**Endianness reversal.** Bitcoin txids are stored in reversed byte order compared to the rest of the Stacks codebase. The `to_txid` helper in `bits.rs:525-532` reverses the bytes when converting from the Bitcoin library's representation.

**SPV database migrations.** The SPV database has gone through 3 schema versions. Version 3 (`spv.rs:86-108`) added a `hash` column to the headers table, requiring a full re-download of headers on upgrade. The migration drops all headers and chain work data.

**Witness size attack vector.** The test `test_witness_size` in `spv.rs:1745-1779` documents a known concern: a Bitcoin transaction with 1.5 million empty witness items occupies far more memory when deserialized than its serialized form, because each empty `Vec<u8>` costs 24 bytes on the heap. This is relevant for resource limits when processing Bitcoin blocks.

**Difficulty validation on regtest.** Regtest mode skips all difficulty validation and uses a fixed maximum target (`spv.rs:1089-1099`), making it unsuitable for testing PoW validation logic.

**Check ordering in `maybe_burnchain_tx`.** The parser rejects a transaction if *any* non-OP_RETURN output has an unrecognized address type, even if only the OP_RETURN payload matters for Stacks. This means a valid Stacks operation embedded in a transaction with an exotic Bitcoin output will be silently dropped.

**Reverse-order header sync.** The SPV client supports fetching headers in descending order (`reverse_order` flag). This is used when the node needs to validate headers from a specific height backward, such as when recovering from a reorg. In reverse mode, continuity is validated against child headers rather than parent headers.
