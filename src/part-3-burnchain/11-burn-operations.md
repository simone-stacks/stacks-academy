# Chapter 11: Burn Operations

## Purpose

Stacks operations are encoded as Bitcoin transactions. A miner who wants to participate in sortition creates a Bitcoin transaction with a specially formatted `OP_RETURN` output. The Stacks node scans every Bitcoin block, identifies these transactions by their magic bytes and opcode, and parses them into structured operation types. This chapter covers every operation type the system recognizes, how each is encoded on the wire, and how operations are validated before they enter the sortition database.

## Key Concepts

**Opcodes.** Each Stacks operation is identified by a single ASCII byte immediately following the two-byte magic prefix in the `OP_RETURN` payload. The opcodes are defined at `stackslib/src/chainstate/burn/mod.rs:58-67`:

```rust
#[repr(u8)]
pub enum Opcodes {
    LeaderBlockCommit = b'[',       // 0x5B
    LeaderKeyRegister = b'^',       // 0x5E
    StackStx          = b'x',       // 0x78
    PreStx            = b'p',       // 0x70
    TransferStx       = b'$',       // 0x24
    DelegateStx       = b'#',       // 0x23
    VoteForAggregateKey = b'v',     // 0x76
}
```

**Two-phase stacking operations.** The `StackStx`, `TransferStx`, and `DelegateStx` operations use a two-phase protocol: first a `PreStxOp` is submitted, then the actual stacking operation references it. The `PreStxOp` locks an output to a specific Stacks address, allowing the subsequent operation to prove the sender's identity through Bitcoin's UTXO model.

**Common fields.** Every operation shares four metadata fields derived from its position in the burnchain:

| Field | Type | Description |
|-------|------|-------------|
| `txid` | `Txid` | Bitcoin transaction ID |
| `vtxindex` | `u32` | Position of this tx within the Bitcoin block |
| `block_height` | `u64` | Bitcoin block height |
| `burn_header_hash` | `BurnchainHeaderHash` | Hash of the Bitcoin block header |

## Architecture

Burn operations follow a pipeline: the Bitcoin indexer scans each Bitcoin block for transactions with the Stacks magic bytes prefix in an `OP_RETURN` output, the parser decodes the opcode byte and deserializes the operation-specific payload, and then the validator checks the operation against the current sortition DB state (e.g., verifying a block commit references a valid VRF key). Valid operations are stored in the `BurnchainDB` and fed into the sortition algorithm. The implementation lives in `stackslib/src/chainstate/burn/operations/`, with one file per operation type.

## The BlockstackOperationType Enum

All parsed operations are wrapped in a single enum (`stackslib/src/chainstate/burn/operations/mod.rs:346-354`):

```rust
pub enum BlockstackOperationType {
    LeaderKeyRegister(LeaderKeyRegisterOp),
    LeaderBlockCommit(LeaderBlockCommitOp),
    PreStx(PreStxOp),
    StackStx(StackStxOp),
    TransferStx(TransferStxOp),
    DelegateStx(DelegateStxOp),
    VoteForAggregateKey(VoteForAggregateKeyOp),
}
```

This enum provides uniform access to common fields through methods like `txid()`, `vtxindex()`, `block_height()`, `burn_header_hash()`, and `opcode()`.

## Operation Types

### LeaderKeyRegisterOp (opcode `^`)

**Purpose:** A miner registers a VRF public key that will be used to prove their sortition eligibility. This must happen before the miner can submit block commits.

**Wire format** (`stackslib/src/chainstate/burn/operations/leader_key_register.rs:90-99`):

```
0      2  3              23                       55                      75        80
|------|--|---------------|-----------------------|-----------------------|---------|
 magic  op consensus hash   proving public key      block-signing hash160    memo
           (ignored)
```

**Struct** (`operations/mod.rs:263-274`):

```rust
pub struct LeaderKeyRegisterOp {
    pub consensus_hash: ConsensusHash, // consensus hash at time of issuance
    pub public_key: VRFPublicKey,      // EdDSA VRF public key (32 bytes)
    pub memo: Vec<u8>,                 // extra bytes in the OP_RETURN
    // + common fields
}
```

**Key details:**
- The `consensus_hash` field is parsed but actually ignored during validation -- it exists for historical reasons.
- The `public_key` is a 32-byte Ed25519 VRF public key used for generating VRF proofs in block commits.
- In Nakamoto, the first 20 bytes of the `memo` field encode the Hash160 of the miner's block-signing key, retrieved via `interpret_nakamoto_signing_key()` (`leader_key_register.rs:69-71`).
- Parsing requires at least 52 bytes of payload (20 bytes consensus hash + 32 bytes VRF key).

### LeaderBlockCommitOp (opcode `[`)

**Purpose:** The core mining operation. A miner commits BTC to compete for the right to produce the next Stacks block. The amount of BTC committed determines the miner's probability of winning the sortition.

**Wire format** (`stackslib/src/chainstate/burn/operations/leader_block_commit.rs:194-243`):

```
0      2  3            35               67     71     73    77   79     80
|------|--|-------------|---------------|------|------|-----|-----|-----|
 magic  op  block hash     new seed     parent parent key   key   burn_parent_modulus
                                        block  txoff  block txoff  + memo (packed)
```

**Struct** (`operations/mod.rs:216-257`):

```rust
pub struct LeaderBlockCommitOp {
    pub block_header_hash: BlockHeaderHash, // hash of proposed Stacks block
    pub new_seed: VRFSeed,                  // VRF proof output for this block
    pub parent_block_ptr: u32,              // burn block height of parent
    pub parent_vtxindex: u16,               // tx index of parent's commit
    pub key_block_ptr: u32,                 // burn block of key registration
    pub key_vtxindex: u16,                  // tx index of key registration
    pub memo: Vec<u8>,                      // extra byte (memo + modulus)
    pub burn_fee: u64,                      // total BTC committed (satoshis)
    pub input: (Txid, u32),                 // input UTXO for commitment smoothing
    pub burn_parent_modulus: u8,            // (block_height - 1) % 5
    pub apparent_sender: BurnchainSigner,   // unauthenticated sender info
    pub commit_outs: Vec<PoxAddress>,       // PoX reward addresses
    pub treatment: Vec<Treatment>,          // reward/punish per PoX address
    pub sunset_burn: u64,                   // extra burn for PoX sunset
    // + common fields
}
```

**Key details:**

- **Outputs per commit.** A block commit must have exactly `OUTPUTS_PER_COMMIT` (2) outputs after the `OP_RETURN`, each paying to a PoX reward address (`leader_block_commit.rs:78`). During proof-of-burn phases, these outputs are burn addresses.

- **Burn parent modulus.** The low 3 bits of byte 76 encode `(intended_burn_height - 1) % 5`. This prevents a miner from submitting a commit for one sortition and having it land in a different one (`leader_block_commit.rs:79`).

- **UTXO chaining.** The `input` field references the UTXO spent by this commit. Miners are expected to chain their commits through a sequence of UTXOs across successive blocks. This is critical for the mining commitment smoothing window (see `BurnSamplePoint::make_min_median_distribution` in `distribution.rs`).

- **Minimum payload.** Parsing requires at least 77 bytes of data payload.

- **The `Treatment` enum.** After validation, each PoX address in `commit_outs` gets a `Treatment::Reward` or `Treatment::Punish` designation. Punished addresses receive no rewards. This is set during `check()`, not during parsing.

### PreStxOp (opcode `p`)

**Purpose:** Authorizes a subsequent `StackStx`, `TransferStx`, or `DelegateStx` operation by locking a Bitcoin output to a Stacks address.

**Struct** (`operations/mod.rs:202-213`):

```rust
pub struct PreStxOp {
    pub output: StacksAddress,    // the Stacks address that will perform the operation
    // + common fields
}
```

**Parsing** (`stack_stx.rs:72-99`): The `output` address is extracted from the first Bitcoin transaction output (after OP_RETURN). The parser validates that the opcode matches, there is at least one input and one output, and the first output can be decoded into a Stacks address.

### StackStxOp (opcode `x`)

**Purpose:** Lock STX tokens for stacking (PoX consensus), specifying a Bitcoin reward address.

**Struct** (`operations/mod.rs:181-200`):

```rust
pub struct StackStxOp {
    pub sender: StacksAddress,
    pub reward_addr: PoxAddress,                    // PoX Bitcoin reward address
    pub stacked_ustx: u128,                         // amount of uSTX to lock
    pub num_cycles: u8,                             // number of reward cycles
    pub signer_key: Option<StacksPublicKeyBuffer>,  // signer key (PoX-4+)
    pub max_amount: Option<u128>,                   // max amount (PoX-4+)
    pub auth_id: Option<u32>,                       // auth ID (PoX-4+)
    // + common fields
}
```

**Wire format parsed data** (`stack_stx.rs:34-40`):

```rust
struct ParsedData {
    stacked_ustx: u128,           // 16 bytes, big-endian
    num_cycles: u8,               // 1 byte
    signer_key: Option<StacksPublicKeyBuffer>,  // 33 bytes (optional, PoX-4)
    max_amount: Option<u128>,     // 16 bytes (optional, PoX-4)
    auth_id: Option<u32>,         // 4 bytes (optional, PoX-4)
}
```

The `sender` is determined from the referenced `PreStxOp` -- it is the address that was locked in the pre-stx transaction. The `reward_addr` comes from the second Bitcoin output.

### TransferStxOp (opcode `$`)

**Purpose:** Transfer STX tokens on the burnchain (without going through a Stacks transaction).

**Struct** (`operations/mod.rs:167-179`):

```rust
pub struct TransferStxOp {
    pub sender: StacksAddress,
    pub recipient: StacksAddress,
    pub transfered_ustx: u128,
    pub memo: Vec<u8>,
    // + common fields
}
```

### DelegateStxOp (opcode `#`)

**Purpose:** Delegate STX tokens to another Stacks address for pooled stacking.

**Struct** (`operations/mod.rs:276-293`):

```rust
pub struct DelegateStxOp {
    pub sender: StacksAddress,
    pub delegate_to: StacksAddress,
    pub reward_addr: Option<(u32, PoxAddress)>,  // optional: output index + address
    pub delegated_ustx: u128,
    pub until_burn_height: Option<u64>,
    // + common fields
}
```

### VoteForAggregateKeyOp (opcode `v`)

**Purpose:** A signer votes for the aggregate public key to be used in the next reward cycle. This is part of the Nakamoto signer coordination protocol.

**Wire format** (`vote_for_aggregate_key.rs:55-64`):

```
0     2    3           5              38     42           50
|-----|----|-----------|--------------|------|------------|
magic  op  signer_index aggregate_key  round  reward_cycle
```

**Struct** (`operations/mod.rs:295-309`):

```rust
pub struct VoteForAggregateKeyOp {
    pub sender: StacksAddress,
    pub aggregate_key: StacksPublicKeyBuffer,   // 33-byte compressed public key
    pub round: u32,                             // voting round
    pub reward_cycle: u64,                      // target reward cycle
    pub signer_index: u16,                      // signer's position in the set
    pub signer_key: StacksPublicKeyBuffer,      // signer's own public key
    // + common fields
}
```

**Key details:**
- Payload must be exactly 47 bytes.
- The `signer_key` is extracted from the Bitcoin transaction input, not from the OP_RETURN payload itself.
- The `aggregate_key` is the 33-byte compressed secp256k1 public key the signer is voting for.

## How Operations Are Parsed from Bitcoin Transactions

The parsing pipeline works as follows:

1. **Block parser** (`blocks.rs`) scans each Bitcoin transaction in a block and calls `maybe_burnchain_tx` to filter candidates.

2. **Data extraction** -- `parse_data` extracts the opcode and payload bytes from the `OP_RETURN` output, stripping magic bytes.

3. **Opcode dispatch** -- Based on the opcode byte, the appropriate `from_tx` method is called on the corresponding operation struct.

4. **Operation-specific parsing** -- Each operation's `parse_data` method interprets the payload bytes according to its wire format. For example, `LeaderBlockCommitOp::parse_data` (`leader_block_commit.rs:194-243`) reads 32 bytes for the block hash, 32 bytes for the VRF seed, 4+2 bytes for parent pointers, 4+2 bytes for key pointers, and 1 byte for the modulus+memo.

5. **Input/output parsing** -- The parser also extracts Bitcoin inputs (for sender identification) and outputs (for PoX reward addresses or recipient addresses).

6. **Validation** -- After parsing, each operation's `check` method validates it against the current chain state via the SortitionDB. For example, `LeaderBlockCommitOp::check` verifies that the referenced key registration exists, that the parent block commit exists, and that the output addresses match the expected PoX reward set.

## Code Walkthrough: Block Commit Parsing

The `LeaderBlockCommitOp::from_tx` method (`leader_block_commit.rs:245`) orchestrates the parsing (a separate `parse_from_tx` method at line 266 handles the lower-level deserialization):

1. Calls `parse_data` to extract the wire-format fields.
2. Extracts `burn_fee` from the Bitcoin transaction's data output amount.
3. Reads `commit_outs` from the Bitcoin transaction outputs (positions 1 through `OUTPUTS_PER_COMMIT`).
4. Extracts the `apparent_sender` from the first transaction input.
5. Extracts the `input` UTXO reference from the first input.

The critical validation in `check()` includes:
- The key registration referenced by `(key_block_ptr, key_vtxindex)` must exist in the SortitionDB.
- The parent commit referenced by `(parent_block_ptr, parent_vtxindex)` must exist (unless this is the first commit).
- The `burn_parent_modulus` must match `(block_height - 1) % BURN_BLOCK_MINED_AT_MODULUS`.
- The commit outputs must match the expected PoX reward set or be burn addresses.

## Mining Commitment Smoothing

The burn distribution is not simply based on a single block commit's BTC spend. Instead, `BurnSamplePoint::make_min_median_distribution` (`stackslib/src/chainstate/burn/distribution.rs:167-172`) considers a window of recent commits:

1. **UTXO chain linking** -- Block commits are linked across the window by tracing UTXO inputs. A commit at the current height is traced back through its spent UTXO to find the prior commit by the same miner.

2. **Median calculation** -- For each miner's chain of commits, the burn amounts are sorted and the median is computed over the window.

3. **Min(median, most_recent)** -- The effective burn for sortition probability is `min(median_burn, most_recent_burn)`. This prevents a miner from inflating their probability with a single large commit while spending minimally in prior blocks.

4. **Frequency tracking** -- The number of blocks in which a miner committed (their "frequency" within the window) is tracked. Miners who skip blocks in the window have reduced sortition chances.

5. **Range mapping** -- Effective burns are mapped to non-overlapping ranges within `[0, 2^256)`, proportional to each miner's share of total effective burn.

## How It Connects

- **Chapter 10 (Bitcoin Integration)**: The `BitcoinTransaction` objects from the block parser are the input to operation parsing.
- **Chapter 12 (SortitionDB)**: Parsed and validated operations are stored in the sortition database. Block commits drive the sortition algorithm.
- **Chapter 16 (Stacking and PoX)**: The `StackStxOp`, `DelegateStxOp`, and commit outputs interact with the PoX reward set calculation.
- **Chapter 17 (Mining)**: Miners construct and submit `LeaderKeyRegisterOp` and `LeaderBlockCommitOp` transactions.
- **Chapter 27 (Signer)**: Signers submit `VoteForAggregateKeyOp` to coordinate aggregate key selection.

## Edge Cases and Gotchas

**Pre-stx requirement for stacking ops.** A `StackStxOp`, `TransferStxOp`, or `DelegateStxOp` is only valid if a corresponding `PreStxOp` exists in a prior Bitcoin block. The `PreStxOp` establishes the sender's identity through Bitcoin's UTXO model, since there is no other way to authenticate a Stacks address from a Bitcoin transaction.

**PoX-4 optional fields.** The `StackStxOp` gained three optional fields (`signer_key`, `max_amount`, `auth_id`) starting with PoX-4. The wire format length determines whether these fields are present.

**Apparent sender is unauthenticated.** The `apparent_sender` field on `LeaderBlockCommitOp` is extracted from the Bitcoin transaction inputs but is explicitly documented as unauthenticated. It should only be used for logging, not for consensus decisions.

**UTXO output index convention.** The `expected_chained_utxo` method on `LeaderBlockCommitOp` returns the expected output index that a subsequent commit must spend. During PoX (non-burn) phases, this is output index 2 (after OP_RETURN and the PoX reward outputs). During proof-of-burn phases, this is output index 1.

**Byte 76 packing.** In block commits, byte 76 of the data payload is split between the `burn_parent_modulus` (low 3 bits) and a `memo` field (bits 3-7). The modulus is computed as `value % BURN_BLOCK_MINED_AT_MODULUS` where `BURN_BLOCK_MINED_AT_MODULUS` is 5.

**Missed block commits.** When a miner's block commit lands in a Bitcoin block different from what they intended (due to timing), it becomes a `MissedBlockCommit`. These are still tracked for UTXO chain linking in the commitment smoothing window -- they count as a burn of 1 satoshi rather than the actual fee.
