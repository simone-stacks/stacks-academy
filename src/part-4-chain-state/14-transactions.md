# Chapter 14: Transactions

## Purpose

Transactions are the fundamental unit of state change on the Stacks blockchain. Every STX transfer, smart contract deployment, contract call, coinbase reward, and tenure change is encoded as a `StacksTransaction`. This chapter covers the transaction data model: how transactions are structured, what payload types exist, how authorization works, what post-conditions enforce, and how transactions are serialized and signed.

All transaction types are defined in `stackslib/src/chainstate/stacks/mod.rs` with serialization logic in `transaction.rs` and authorization logic in `auth.rs`.

## Key Concepts

A Stacks transaction has a layered structure:

1. **Envelope**: Version, chain ID, anchor mode
2. **Authorization**: Who is paying and who is sending (standard or sponsored)
3. **Post-conditions**: Safety assertions about asset transfers
4. **Payload**: The actual operation (transfer, contract call, etc.)

This separation means the same payload types work across mainnet/testnet, can be sponsored by third parties, and carry safety guarantees regardless of what the smart contract does.

## Architecture

A transaction flows through several stages: construction and signing (client-side), submission to the mempool (via HTTP or P2P), validation (nonce checks, fee checks, post-condition evaluation), and finally execution within a Clarity block context. The `StacksTransaction` struct at the center is designed as a layered envelope: outer metadata (version, chain ID, anchor mode), authorization (signatures and spending conditions), post-conditions (safety assertions), and the inner payload (the actual operation). Serialization uses the `StacksMessageCodec` trait for deterministic consensus encoding.

## The StacksTransaction Struct

```rust
// stackslib/src/chainstate/stacks/mod.rs:1128-1137
pub struct StacksTransaction {
    pub version: TransactionVersion,
    pub chain_id: u32,
    pub auth: TransactionAuth,
    pub anchor_mode: TransactionAnchorMode,
    pub post_condition_mode: TransactionPostConditionMode,
    pub post_conditions: Vec<TransactionPostCondition>,
    pub payload: TransactionPayload,
}
```

### Field Breakdown

**`version`** distinguishes mainnet (`0x00`) from testnet (`0x80`):

```rust
// stackslib/src/chainstate/stacks/mod.rs:1121-1126
pub enum TransactionVersion {
    Mainnet = 0x00,
    Testnet = 0x80,
}
```

**`chain_id`** is a u32 that prevents replay across different Stacks networks (mainnet uses `0x00000001`).

**`anchor_mode`** controls where the transaction can appear:

```rust
// stackslib/src/chainstate/stacks/mod.rs:416-420
pub enum TransactionAnchorMode {
    OnChainOnly = 1,  // must be in a StacksBlock
    OffChainOnly = 2, // must be in a StacksMicroBlock
    Any = 3,          // either
}
```

## Transaction Payloads

The `TransactionPayload` enum defines all operations:

```rust
// stackslib/src/chainstate/stacks/mod.rs:904-913
pub enum TransactionPayload {
    TokenTransfer(PrincipalData, u64, TokenTransferMemo),
    ContractCall(TransactionContractCall),
    SmartContract(TransactionSmartContract, Option<ClarityVersion>),
    PoisonMicroblock(StacksMicroblockHeader, StacksMicroblockHeader),
    Coinbase(CoinbasePayload, Option<PrincipalData>, Option<VRFProof>),
    TenureChange(TenureChangePayload),
}
```

Each variant is identified by a `TransactionPayloadID` byte on the wire:

```rust
// stackslib/src/chainstate/stacks/mod.rs:948-960
define_u8_enum!(TransactionPayloadID {
    TokenTransfer = 0,
    SmartContract = 1,
    ContractCall = 2,
    PoisonMicroblock = 3,
    Coinbase = 4,
    CoinbaseToAltRecipient = 5,
    VersionedSmartContract = 6,
    TenureChange = 7,
    NakamotoCoinbase = 8
});
```

### TokenTransfer

The simplest payload -- send STX from sender to recipient:

```
TokenTransfer(PrincipalData, u64, TokenTransferMemo)
```

- `PrincipalData`: The recipient (standard address or contract principal)
- `u64`: Amount in microSTX
- `TokenTransferMemo`: A 34-byte memo field (same length as Stacks v1 for compatibility)

### ContractCall

Invoke a public function on a deployed smart contract:

```rust
// stackslib/src/chainstate/stacks/mod.rs:662-668
pub struct TransactionContractCall {
    pub address: StacksAddress,
    pub contract_name: ContractName,
    pub function_name: ClarityName,
    pub function_args: Vec<Value>,
}
```

The `function_args` are Clarity `Value` objects serialized directly into the transaction. On deserialization, function args are read using a `BoundReader` limited to `MAX_TRANSACTION_LEN` (2 MB) to prevent memory exhaustion attacks.

The function name is validated to be a legal Clarity variable name during deserialization (transaction.rs:52-58).

### SmartContract

Deploy a new Clarity smart contract:

```rust
// stackslib/src/chainstate/stacks/mod.rs:678-682
pub struct TransactionSmartContract {
    pub name: ContractName,
    pub code_body: StacksString,
}
```

The optional `ClarityVersion` in `SmartContract(TransactionSmartContract, Option<ClarityVersion>)` was added for versioned smart contracts (Clarity 2, 3, 4). When `None`, the contract defaults to Clarity 1. When `Some`, the wire format uses `VersionedSmartContract` (payload ID 6) instead of `SmartContract` (payload ID 1).

### Coinbase

Block reward transaction -- must be the first transaction in every anchored block:

```
Coinbase(CoinbasePayload, Option<PrincipalData>, Option<VRFProof>)
```

- `CoinbasePayload`: 32 bytes of miner-chosen data
- `Option<PrincipalData>`: Alternate recipient for the coinbase reward (payload ID 5, `CoinbaseToAltRecipient`)
- `Option<VRFProof>`: Required in Nakamoto blocks (payload ID 8, `NakamotoCoinbase`), proves the miner won sortition

### PoisonMicroblock

Evidence that a microblock producer equivocated (produced two microblocks with the same sequence number but different content):

```
PoisonMicroblock(StacksMicroblockHeader, StacksMicroblockHeader)
```

Both headers must have the same `sequence` and `prev_block` but different `tx_merkle_root`. Submitting a valid poison microblock transaction punishes the equivocating producer.

### TenureChange (Nakamoto)

Signals a change or extension of mining tenure:

```rust
// stackslib/src/chainstate/stacks/mod.rs:832-853
pub struct TenureChangePayload {
    pub tenure_consensus_hash: ConsensusHash,
    pub prev_tenure_consensus_hash: ConsensusHash,
    pub burn_view_consensus_hash: ConsensusHash,
    pub previous_tenure_end: StacksBlockId,
    pub previous_tenure_blocks: u32,
    pub cause: TenureChangeCause,
    pub pubkey_hash: Hash160,
}
```

The `TenureChangeCause` enum distinguishes between a new tenure from sortition (`BlockFound`) and various tenure extensions:

```rust
// stackslib/src/chainstate/stacks/mod.rs:703-717
pub enum TenureChangeCause {
    BlockFound = 0,         // New sortition winner
    Extended = 1,           // Extend all budget dimensions
    ExtendedRuntime = 2,    // Extend runtime only (SIP-034)
    ExtendedReadCount = 3,
    ExtendedReadLength = 4,
    ExtendedWriteCount = 5,
    ExtendedWriteLength = 6,
}
```

The `cause` field deliberately does not implement `PartialEq` to force callers to use explicit methods like `is_new_tenure()` and `is_extended()`, preventing accidental incomplete equality checks when new variants are added.

## Transaction Authorization

### Auth Types

Every transaction has an `auth` field of type `TransactionAuth`:

```rust
// stackslib/src/chainstate/stacks/mod.rs:655-659
pub enum TransactionAuth {
    Standard(TransactionSpendingCondition),
    Sponsored(TransactionSpendingCondition, TransactionSpendingCondition),
}
```

- **Standard**: The sender pays the fee and authorizes the transaction.
- **Sponsored**: The first spending condition is the sender (origin), the second is the sponsor who pays the fee. This enables gasless transactions for end users.

On the wire, auth type is identified by:

```rust
// stackslib/src/chainstate/stacks/mod.rs:423-428
pub enum TransactionAuthFlags {
    AuthStandard = 0x04,
    AuthSponsored = 0x05,
}
```

### Spending Conditions

A spending condition encodes everything needed to verify that a transaction is authorized:

```rust
// stackslib/src/chainstate/stacks/mod.rs:647-652
pub enum TransactionSpendingCondition {
    Singlesig(SinglesigSpendingCondition),
    Multisig(MultisigSpendingCondition),
    OrderIndependentMultisig(OrderIndependentMultisigSpendingCondition),
}
```

**Singlesig**:

```rust
// stackslib/src/chainstate/stacks/mod.rs:627-635
pub struct SinglesigSpendingCondition {
    pub hash_mode: SinglesigHashMode,   // P2PKH or P2WPKH
    pub signer: Hash160,                // Hash of the public key
    pub nonce: u64,                     // Sequence number for replay protection
    pub tx_fee: u64,                    // Fee in microSTX
    pub key_encoding: TransactionPublicKeyEncoding,
    pub signature: MessageSignature,
}
```

**Multisig**:

```rust
// stackslib/src/chainstate/stacks/mod.rs:617-625
pub struct MultisigSpendingCondition {
    pub hash_mode: MultisigHashMode,    // P2SH or P2WSH
    pub signer: Hash160,                // Hash of the redeem script
    pub nonce: u64,
    pub tx_fee: u64,
    pub fields: Vec<TransactionAuthField>,  // Mix of signatures and public keys
    pub signatures_required: u16,
}
```

The `fields` vector contains a mix of `TransactionAuthField::Signature` and `TransactionAuthField::PublicKey` entries. The order matters for standard multisig (signatures must be in the same order as keys in the redeem script). For `OrderIndependentMultisig` (added in Epoch 3.0), signatures can appear in any order.

### Hash Modes

Hash modes correspond to Bitcoin address types:

| Mode | Value | Type |
|------|-------|------|
| `P2PKH` | 0x00 | Pay-to-public-key-hash (singlesig) |
| `P2SH` | 0x01 | Pay-to-script-hash (multisig) |
| `P2WPKH` | 0x02 | Pay-to-witness-public-key-hash (singlesig segwit) |
| `P2WSH` | 0x03 | Pay-to-witness-script-hash (multisig segwit) |
| `P2SH` (order-independent) | 0x05 | Order-independent multisig P2SH |
| `P2WSH` (order-independent) | 0x07 | Order-independent multisig P2WSH |

### Signing Process

Transaction signing uses `StacksTransactionSigner`:

```rust
// stackslib/src/chainstate/stacks/mod.rs:1139-1146
pub struct StacksTransactionSigner {
    pub tx: StacksTransaction,
    pub sighash: Txid,
    origin_done: bool,
    check_oversign: bool,
    check_overlap: bool,
}
```

The signing flow:
1. Serialize the transaction with empty signatures to get the initial sighash
2. For each signer, compute a new sighash that includes the previous signature (chaining)
3. Sign the current sighash with the private key
4. Append the signature to the auth fields

The `Txid` is computed as SHA-512/256 of the fully serialized transaction (mod.rs:383-388):

```rust
pub fn from_stacks_tx(txdata: &[u8]) -> Txid {
    let h = Sha512Trunc256Sum::from_data(txdata);
    let mut bytes = [0u8; 32];
    bytes.copy_from_slice(h.as_bytes());
    Txid(bytes)
}
```

## Post-Conditions

Post-conditions are the key safety mechanism that protects users from unexpected asset transfers. They are checked **after** transaction execution and cause the transaction to abort if violated.

### Post-Condition Mode

```rust
// stackslib/src/chainstate/stacks/mod.rs:1113-1118
pub enum TransactionPostConditionMode {
    Allow = 0x01, // allow any asset changes not explicitly constrained
    Deny = 0x02,  // deny any asset changes not explicitly constrained
}
```

In `Deny` mode, any asset transfer not covered by a post-condition causes the transaction to fail. This is the recommended mode for wallet software.

### Post-Condition Types

```rust
// stackslib/src/chainstate/stacks/mod.rs:1095-1110
pub enum TransactionPostCondition {
    STX(PostConditionPrincipal, FungibleConditionCode, u64),
    Fungible(PostConditionPrincipal, AssetInfo, FungibleConditionCode, u64),
    Nonfungible(PostConditionPrincipal, AssetInfo, Value, NonfungibleConditionCode),
}
```

**STX post-condition**: "Principal X must send exactly/at-most/at-least Y microSTX"

**Fungible post-condition**: Same as STX but for custom fungible tokens, identified by `AssetInfo`:

```rust
// stackslib/src/chainstate/stacks/mod.rs:963-968
pub struct AssetInfo {
    pub contract_address: StacksAddress,
    pub contract_name: ContractName,
    pub asset_name: ClarityName,
}
```

**Nonfungible post-condition**: "Principal X must/must-not send NFT Y"

### Condition Codes

Fungible conditions support five comparison operators:

```rust
// stackslib/src/chainstate/stacks/mod.rs:991-998
pub enum FungibleConditionCode {
    SentEq = 0x01,  // ==
    SentGt = 0x02,  // >
    SentGe = 0x03,  // >=
    SentLt = 0x04,  // <
    SentLe = 0x05,  // <=
}
```

NFT conditions are binary:

```rust
// stackslib/src/chainstate/stacks/mod.rs:1024-1028
pub enum NonfungibleConditionCode {
    Sent = 0x10,
    NotSent = 0x11,
}
```

### Post-Condition Principals

Post-conditions can constrain the transaction origin or any specific principal:

```rust
// stackslib/src/chainstate/stacks/mod.rs:1062-1067
pub enum PostConditionPrincipal {
    Origin,
    Standard(StacksAddress),
    Contract(StacksAddress, ContractName),
}
```

## Serialization

All transaction components implement `StacksMessageCodec` for deterministic binary serialization. The wire format is:

```
[version: 1 byte]
[chain_id: 4 bytes]
[auth_type: 1 byte]
[spending_condition(s): variable]
[anchor_mode: 1 byte]
[post_condition_mode: 1 byte]
[post_conditions: length-prefixed list]
[payload_type: 1 byte]
[payload: variable]
```

The `TransactionPayload` serialization (transaction.rs:188-260) dispatches on variant, writing the payload type ID byte followed by variant-specific data. For example, `TokenTransfer` writes the recipient principal, amount (u64), and 34-byte memo. `ContractCall` writes the contract address, name, function name, and argument list.

Deserialization enforces size limits via `BoundReader` to prevent memory exhaustion from malicious payloads. The maximum transaction size is 2 MB (`MAX_TRANSACTION_LEN`), matching the maximum block size.

## How It Connects

- **Block Processing** (Chapter 15): Blocks contain vectors of `StacksTransaction`. The block processor validates each transaction's auth, executes its payload, and checks post-conditions.
- **Clarity VM** (Chapters 5-9): `ContractCall` and `SmartContract` payloads are executed by the Clarity interpreter.
- **Mempool** (Chapter 26): Transactions are validated and stored in the mempool before being selected for block inclusion.
- **Mining** (Chapter 17): The miner's `StacksBlockBuilder` selects transactions from the mempool and applies them via `try_mine_tx_with_len`.
- **Nakamoto** (Chapter 18): `TenureChange` and `NakamotoCoinbase` payloads are new in the Nakamoto epoch.

## Edge Cases and Gotchas

1. **Nonce management**: Each spending condition carries a `nonce` that must exactly match the account's current nonce. A gap causes rejection; a duplicate causes replay rejection. Sponsored transactions have independent nonces for origin and sponsor.

2. **Fee bidding**: The `tx_fee` in the spending condition is the fee the sender/sponsor is willing to pay. The actual fee charged equals this value -- there is no gas refund mechanism in Stacks.

3. **Coinbase position**: The coinbase transaction must be the first transaction in an anchored block. In Nakamoto, the first block of a tenure must start with a `TenureChange(BlockFound)` transaction followed by a `NakamotoCoinbase`.

4. **Version-gated payloads**: `OrderIndependentMultisig` spending conditions are only valid starting in Epoch 3.0. `VersionedSmartContract` (payload ID 6) is only valid from Epoch 2.1. `TenureChange` (ID 7) and `NakamotoCoinbase` (ID 8) are only valid in Epoch 3.0+.

5. **Memo immutability**: The `TokenTransferMemo` is 34 bytes and zero-padded. It is included in the transaction hash but is not interpreted by the protocol -- it exists purely for application-level metadata.

6. **Sponsored transaction auth fields**: In a sponsored transaction, the origin signs first, then the sponsor signs over the origin's signature. The sighash chain ensures neither party can tamper with the other's authorization after signing.
