# Chapter 2: Common Types (stacks-common)

## Purpose

The `stacks-common` crate is the foundation of the entire Stacks codebase. Every other crate depends on it. It provides the cryptographic primitives (hashes, keys, signatures), address encoding (C32), consensus serialization (`StacksMessageCodec`), epoch definitions, and utility functions that appear throughout the system.

This chapter walks through every major type family so you can recognize them when reading any part of the code.

## Key Concepts

The crate provides four families of types that appear throughout the Stacks codebase: **cryptographic hashes** (SHA-256, SHA-512/256, Hash160, RIPEMD-160), **asymmetric keys** (secp256k1 ECDSA, VRF ed25519), **address encoding** (C32Check and Base58Check), and **consensus serialization** (`StacksMessageCodec`). Understanding these primitives is a prerequisite for reading any other crate.

## Crate Structure

The crate entry point is `stacks-common/src/libcommon.rs`, which declares these top-level modules:

```rust
// stacks-common/src/libcommon.rs:22-40
pub mod util;        // Hashes, keys, VRF, logging, hex encoding
pub mod codec;       // StacksMessageCodec trait and serialization helpers
pub mod types;       // StacksEpochId, StacksAddress, chainstate hash types
pub mod address;     // C32 and Base58 address encoding
pub mod bitvec;      // Bit vector utilities (used in PoX)
pub mod consts;      // Network constants (chain IDs, peer versions, etc.)
pub mod deps_common; // Vendored dependencies (Bitcoin script, bech32, httparse)
```

## Hash Types

All hash types live in `stacks-common/src/util/hash.rs`. They share a common pattern: a newtype wrapper around a fixed-size byte array, with macros providing serialization, display, and comparison.

### Hash160 (RIPEMD-160 of SHA-256)

```rust
// stacks-common/src/util/hash.rs:83-93
pub struct Hash160(pub [u8; 20]);
```

This is the Bitcoin-style address hash: `RIPEMD160(SHA256(data))`. It is used for:
- Stacks address derivation (the 20-byte payload inside every `StacksAddress`)
- Public key hashing (`Hash160::from_node_public_key`)

The construction chain is explicit:

```rust
// stacks-common/src/util/hash.rs:182-186
pub fn from_data(data: &[u8]) -> Hash160 {
    let sha2_result = Sha256::digest(data);
    let ripe_160_result = Ripemd160::digest(sha2_result.as_slice());
    Hash160(ripe_160_result.into())
}
```

### Sha256Sum, Sha512Sum, Sha512Trunc256Sum

```rust
// stacks-common/src/util/hash.rs:108-147
pub struct Sha256Sum(pub [u8; 32]);
pub struct Sha512Sum(pub [u8; 64]);
pub struct Sha512Trunc256Sum(pub [u8; 32]);
```

- **Sha256Sum**: Standard SHA-256. Used for data integrity checks and in Bitcoin-compatible operations.
- **Sha512Sum**: Full SHA-512. Used in VRF proof computation.
- **Sha512Trunc256Sum**: SHA-512/256 (the first 256 bits of SHA-512). This is the workhorse hash for Stacks-specific data structures -- it is used for MARF trie node hashing and `SortitionId` computation.

### DoubleSha256

```rust
// stacks-common/src/util/hash.rs:150-159
pub struct DoubleSha256(pub [u8; 32]);
```

`SHA256(SHA256(data))` -- the Bitcoin double-hash. Used in transaction Merkle trees and Bitcoin block header verification.

### TrieHash

```rust
// stacks-common/src/types/chainstate.rs:37
pub struct TrieHash(pub [u8; 32]);
```

A `SHA-512/256` hash used specifically for MARF trie nodes. The `from_empty_data()` method returns a hardcoded constant (the hash of the empty string) to avoid repeated computation:

```rust
// stacks-common/src/types/chainstate.rs:52-58
pub fn from_empty_data() -> TrieHash {
    TrieHash([
        0xc6, 0x72, 0xb8, 0xd1, 0xef, 0x56, 0xed, 0x28,
        0xab, 0x87, 0xc3, 0x62, 0x2c, 0x51, 0x14, 0x06,
        // ... (SHA-512/256 of "")
    ])
}
```

### MerkleHashFunc Trait

All hash types that can be used in Merkle trees implement this trait:

```rust
// stacks-common/src/util/hash.rs:34-42
pub trait MerkleHashFunc {
    fn empty() -> Self where Self: Sized;
    fn from_tagged_data(tag: u8, data: &[u8]) -> Self where Self: Sized;
    fn bits(&self) -> &[u8];
}
```

The `from_tagged_data` method prepends a tag byte before hashing. This is critical for Merkle tree security: leaf nodes use tag `0x00` and internal nodes use tag `0x01`, preventing second-preimage attacks. The `MerkleTree<H>` generic struct at `hash.rs:358` uses these tags.

### Hex Encoding Utilities

```rust
// stacks-common/src/util/hash.rs:579
pub fn hex_bytes(s: &str) -> Result<Vec<u8>, HexError> { ... }

// stacks-common/src/util/hash.rs:649
pub fn to_hex(s: &[u8]) -> String { ... }
```

These appear everywhere in the codebase for converting between byte arrays and hex strings. The `to_hex` function uses a precomputed lookup table (`HEX_CHARS` at line 623) for performance.

## Cryptographic Keys

### secp256k1 Keys

Stacks uses the same elliptic curve as Bitcoin (secp256k1) for transaction signing. The key types live in `stacks-common/src/util/secp256k1/`:

```rust
// stacks-common/src/util/secp256k1/native.rs:38-46
pub struct Secp256k1PublicKey {
    key: LibSecp256k1PublicKey,
    compressed: bool,
}

// stacks-common/src/util/secp256k1/native.rs:49-57
pub struct Secp256k1PrivateKey {
    key: LibSecp256k1PrivateKey,
    compress_public: bool,
}
```

The `compressed` flag determines serialization format: compressed (33 bytes, prefix `0x02`/`0x03`) or uncompressed (65 bytes, prefix `0x04`). Most of the codebase uses compressed keys.

A per-thread secp256k1 context is maintained for performance:

```rust
// stacks-common/src/util/secp256k1/native.rs:35
thread_local!(static _secp256k1: Secp256k1<secp256k1::All> = Secp256k1::new());
```

### MessageSignature

```rust
// stacks-common/src/util/secp256k1/mod.rs:15
pub struct MessageSignature(pub [u8; 65]);
```

A 65-byte recoverable ECDSA signature. The first byte is the recovery ID (0-3), followed by 64 bytes of the compact signature (r, s). The recovery ID allows extracting the signer's public key without having it in advance -- this is how Stacks verifies transaction signatures:

```rust
// stacks-common/src/util/secp256k1/native.rs:77-83
pub fn from_secp256k1_recoverable(sig: &LibSecp256k1RecoverableSignature) -> MessageSignature {
    let (recid, bytes) = sig.serialize_compact();
    let mut ret_bytes = [0u8; 65];
    ret_bytes[0] = recid.to_i32() as u8;
    ret_bytes[1..=64].copy_from_slice(&bytes[..64]);
    MessageSignature(ret_bytes)
}
```

### PublicKey and PrivateKey Traits

These traits abstract over key types and are used by the `Address` system:

```rust
// stacks-common/src/types/mod.rs:70-84
pub trait PublicKey: Clone + fmt::Debug + serde::Serialize + serde::de::DeserializeOwned {
    fn to_bytes(&self) -> Vec<u8>;
    fn verify(&self, data_hash: &[u8], sig: &MessageSignature) -> Result<bool, &'static str>;
}

pub trait PrivateKey: Clone + fmt::Debug + serde::Serialize + serde::de::DeserializeOwned {
    fn to_bytes(&self) -> Vec<u8>;
    fn sign(&self, data_hash: &[u8]) -> Result<MessageSignature, &'static str>;
}
```

`Secp256k1PublicKey` implements `PublicKey` with a signature verification that recovers the public key from the signature and compares it to `self`, also enforcing low-S normalization for malleability protection (`native.rs:197-238`).

### VRF Keys

Verifiable Random Functions are used in sortition (miner selection). VRF keys use Ed25519 (Curve25519), not secp256k1:

```rust
// stacks-common/src/util/vrf.rs:35-38
pub struct VRFPublicKey(pub ed25519_dalek::VerifyingKey);
pub struct VRFPrivateKey(pub ed25519_dalek::SigningKey);
```

A miner registers a VRF public key on Bitcoin, then uses the corresponding private key to produce a VRF proof in their block commit. The proof is deterministically verifiable but unpredictable before creation -- this is what makes sortition fair.

### SchnorrSignature

```rust
// stacks-common/src/util/secp256k1/mod.rs:21-26
pub struct SchnorrSignature(pub [u8; 65]);
```

Used in Nakamoto for aggregate block signatures produced by the signer set via the WSTS (Weighted Schnorr Threshold Signatures) protocol.

## Addresses

### C32 Encoding

Stacks addresses use **C32Check** encoding, defined in `stacks-common/src/address/c32.rs`. C32 is a base-32 encoding using the Crockford alphabet, which excludes visually ambiguous characters. The character set is:

```rust
// stacks-common/src/address/c32.rs:21
const C32_CHARACTERS: &[u8; 32] = b"0123456789ABCDEFGHJKMNPQRSTVWXYZ";
```

Note the missing characters: `I`, `L`, `O`, `U`. The decoder maps `O`->`0`, `L`->`1`, `I`->`1` for user convenience (`c32.rs:49-178`).

A C32Check address has the format: `S` + version_char + c32_encode(data + checksum)

```rust
// stacks-common/src/address/c32.rs:370-373
pub fn c32_address(version: u8, data: &[u8]) -> Result<String, Error> {
    let c32_string = c32_check_encode(version, data)?;
    Ok(format!("S{c32_string}"))
}
```

The checksum is a 4-byte double-SHA256 of `[version] ++ data`. The leading `S` prefix distinguishes Stacks addresses from other base-32-encoded strings.

### Address Versions

```rust
// stacks-common/src/address/mod.rs:31-34
pub const C32_ADDRESS_VERSION_MAINNET_SINGLESIG: u8 = 22; // P
pub const C32_ADDRESS_VERSION_MAINNET_MULTISIG: u8 = 20;  // M
pub const C32_ADDRESS_VERSION_TESTNET_SINGLESIG: u8 = 26;  // T
pub const C32_ADDRESS_VERSION_TESTNET_MULTISIG: u8 = 21;   // N
```

The version byte encodes both the network (mainnet/testnet) and the signature scheme (single/multi). It maps to a C32 character that becomes the second character of the address string. So mainnet addresses start with `SP` and testnet addresses start with `ST`.

### AddressHashMode

```rust
// stacks-common/src/address/mod.rs:97-103
pub enum AddressHashMode {
    SerializeP2PKH = 0x00,   // hash160(public-key)
    SerializeP2SH = 0x01,    // hash160(multisig-redeem-script)
    SerializeP2WPKH = 0x02,  // hash160(segwit-program-00(p2pkh))
    SerializeP2WSH = 0x03,   // hash160(segwit-program-00(public-keys))
}
```

These modes mirror Bitcoin's address types. Stacks encodes addresses the same way Bitcoin does internally, which allows Bitcoin addresses to be directly converted to Stacks addresses. The conversion functions in `address/mod.rs:151-216` construct the appropriate scripts and hash them.

### StacksAddress

```rust
// stacks-common/src/types/chainstate.rs:289-293
pub struct StacksAddress {
    version: u8,
    bytes: Hash160,
}
```

The version field is constrained to 5 bits (0-31) by the constructor:

```rust
// stacks-common/src/types/chainstate.rs:296-305
pub fn new(version: u8, hash: Hash160) -> Result<StacksAddress, AddressError> {
    if version >= 32 {
        return Err(AddressError::InvalidVersion(version));
    }
    Ok(StacksAddress { version, bytes: hash })
}
```

The `Display` implementation encodes the address as a C32Check string. The `Address::from_string` implementation decodes it back.

## Consensus Serialization (StacksMessageCodec)

All data that crosses the wire or goes into consensus hashes must be serialized deterministically. The `StacksMessageCodec` trait defines this:

```rust
// stacks-common/src/codec/mod.rs:67-87
pub trait StacksMessageCodec {
    fn consensus_serialize<W: Write>(&self, fd: &mut W) -> Result<(), Error>
    where Self: Sized;

    fn consensus_deserialize<R: Read>(fd: &mut R) -> Result<Self, Error>
    where Self: Sized;

    fn serialize_to_vec(&self) -> Vec<u8>
    where Self: Sized {
        let mut bytes = vec![];
        self.consensus_serialize(&mut bytes)
            .expect("BUG: serialization to buffer failed.");
        bytes
    }
}
```

This trait is implemented for:
- All integer primitives (u8, u16, u32, u64, i64) -- serialized as **big-endian** bytes
- Fixed-size byte arrays ([u8; 20], [u8; 32])
- `Vec<T>` where `T: StacksMessageCodec` -- prefixed with a `u32` length
- All hash types, signature types, and address types (via macros)

The helper functions `write_next` and `read_next` are the standard way to compose serialization:

```rust
// stacks-common/src/codec/mod.rs:122-128
pub fn write_next<T: StacksMessageCodec, W: Write>(fd: &mut W, item: &T) -> Result<(), Error> {
    item.consensus_serialize(fd)
}

pub fn read_next<T: StacksMessageCodec, R: Read>(fd: &mut R) -> Result<T, Error> {
    let item: T = T::consensus_deserialize(fd)?;
    Ok(item)
}
```

For bounded deserialization (preventing memory exhaustion from malicious inputs), use:

```rust
pub fn read_next_at_most<R, T>(fd: &mut R, max_items: u32) -> Result<Vec<T>, Error>
```

The maximum message size is 16 MB plus overhead:

```rust
// stacks-common/src/codec/mod.rs:203-205
pub const MAX_PAYLOAD_LEN: u32 = 1 + 16 * 1024 * 1024;
pub const MAX_MESSAGE_LEN: u32 = MAX_PAYLOAD_LEN
    + (PREAMBLE_ENCODED_SIZE + MAX_RELAYERS_LEN * RELAY_DATA_ENCODED_SIZE);
```

## Chainstate Identifier Types

Several opaque 32-byte identifiers are used to reference chain state entities:

```rust
// stacks-common/src/types/chainstate.rs
pub struct BurnchainHeaderHash(pub [u8; 32]);  // Bitcoin block hash
pub struct BlockHeaderHash(pub [u8; 32]);      // Stacks block header hash
pub struct SortitionId(pub [u8; 32]);          // Unique sortition identifier
pub struct VRFSeed(pub [u8; 32]);              // VRF output seed
```

**SortitionId** is derived from a burnchain header hash combined with the PoX ID (a bitvector tracking which reward cycles had anchor blocks):

```rust
// stacks-common/src/types/chainstate.rs:162-178
pub fn new(bhh: &BurnchainHeaderHash, pox: &PoxId) -> SortitionId {
    if pox == &PoxId::stubbed() {
        SortitionId(bhh.0)
    } else {
        let mut hasher = Sha512_256::new();
        hasher.update(bhh);
        write!(hasher, "{pox}").expect("...");
        let h = Sha512Trunc256Sum::from_hasher(hasher);
        SortitionId(h.0)
    }
}
```

## The StacksPublicKeyBuffer

A 33-byte buffer for holding compressed secp256k1 public keys. This is the wire format for public keys in messages:

```rust
// stacks-common/src/types/mod.rs:49-54
pub struct StacksPublicKeyBuffer(pub [u8; 33]);
impl_array_newtype!(StacksPublicKeyBuffer, u8, 33);
impl_byte_array_message_codec!(StacksPublicKeyBuffer, 33);
```

It converts to/from `Secp256k1PublicKey` via `from_public_key()` and `to_public_key()`.

## Constants

The `consts` module (`libcommon.rs:42-106`) defines network-wide constants:

| Constant | Value | Meaning |
|---|---|---|
| `CHAIN_ID_MAINNET` | `0x00000001` | Chain identifier for mainnet |
| `CHAIN_ID_TESTNET` | `0x80000000` | Chain identifier for testnet |
| `MINER_REWARD_MATURITY` | 100 (prod) / 2 (test) | Blocks before coinbase rewards can be spent |
| `SIGNER_SLOTS_PER_USER` | 13 | StackerDB slots per signing key |
| `MICROSTACKS_PER_STACKS` | 1,000,000 | 1 STX = 1,000,000 uSTX |
| `TOKEN_TRANSFER_MEMO_LENGTH` | 34 | Max memo bytes in a token transfer |

## How It Connects

- **Chapter 1** (Architecture) shows how `stacks-common` sits at the bottom of the dependency graph.
- **Chapter 3** (Clarity Type System) builds on `Hash160`, `StacksAddress`, and `StacksMessageCodec` for principal types.
- **Chapter 10** (Bitcoin Integration) uses `BurnchainHeaderHash`, `DoubleSha256`, and `AddressHashMode`.
- **Chapter 13** (MARF) uses `TrieHash` and `Sha512Trunc256Sum` extensively.
- **Chapter 14** (Transactions) serializes everything via `StacksMessageCodec`.
- **Chapter 22** (P2P Protocol) builds the message framing on top of `StacksMessageCodec`.

## Edge Cases and Gotchas

1. **Big-endian integers**: All integers in `StacksMessageCodec` are big-endian. This matches Bitcoin's on-wire convention but differs from Rust's native representation on x86.

2. **Version constraint**: `StacksAddress::new()` rejects versions >= 32. This is because the version must fit in a single C32 character (5 bits). The `new_unsafe` constructor bypasses this check and must never be used in production.

3. **Compressed key assumption**: Most of the codebase assumes compressed public keys (33 bytes). Uncompressed keys (65 bytes) are supported for Bitcoin legacy compatibility but are rare in Stacks-native code.

4. **VRF uses Ed25519, not secp256k1**: The VRF system uses a completely different curve than transaction signing. Do not confuse `VRFPublicKey` (Ed25519) with `Secp256k1PublicKey`.

5. **Macro-generated implementations**: Most hash and signature types get their `Clone`, `PartialEq`, `Eq`, `Hash`, `Display`, `Debug`, `StacksMessageCodec`, and serde implementations from macros (`impl_array_newtype!`, `impl_byte_array_newtype!`, `impl_array_hexstring_fmt!`, `impl_byte_array_message_codec!`). If you are looking for a trait implementation and cannot find it, check the macro invocations.

6. **C32 normalization**: The C32 decoder normalizes `O`->`0`, `L`->`1`, `I`->`1` and is case-insensitive. This means multiple string representations decode to the same address. Always compare decoded bytes, not strings.
