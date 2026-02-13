# Appendix A: Glossary of Key Terms

This glossary defines the core concepts, protocols, and data structures used throughout the Stacks codebase. Each entry references the relevant source files so you can trace the concept to its implementation.

---

## Anchor Block

In pre-Nakamoto epochs (before 3.0), the anchor block is the Stacks block that is committed to by a leader block commit on Bitcoin. It "anchors" the Stacks chain to a specific Bitcoin block. The anchor block is produced by the sortition winner and contains a set of transactions that become confirmed once the block is accepted. In Nakamoto, the concept is replaced by tenure-change blocks, but the term still appears in configuration (`commit_anchor_block_within`) and legacy code paths.

**Code:** `stackslib/src/chainstate/stacks/db/blocks.rs`, `stackslib/src/config/mod.rs:1304`

---

## ATC (Assumed Total Commitment)

Assumed Total Commitment is a mechanism used during sortition to prevent a miner from winning with a trivially small burn commitment when total participation is low. The sortition algorithm uses ATC to set a floor on the effective total burn, ensuring that the probability distribution of winning is not heavily skewed by a single small commitment in a low-participation round. This helps maintain fairness in the leader election process.

**Code:** `stackslib/src/burnchains/mod.rs`

---

## Boot Contract

Boot contracts are Clarity smart contracts that are deployed at genesis (block height 0) and form the protocol's on-chain infrastructure. They implement core protocol functions like PoX stacking (`pox-4`), BNS name registration (`bns`), and cost voting (`costs-3`). Boot contracts live at the special address `SP000000000000000000002Q6VF78` (mainnet) or `ST000000000000000000002AMW42H` (testnet). The `.miners` and `.signers-*-*` StackerDB contracts coordinate Nakamoto mining and signing.

**Code:** `stackslib/src/chainstate/stacks/boot/mod.rs`, constant `MINERS_NAME` at `stackslib/src/chainstate/stacks/boot/mod.rs`

---

## Burnchain

The underlying blockchain (Bitcoin) that Stacks uses for its consensus mechanism. Stacks miners commit BTC on the burnchain to participate in leader election (sortition). The burnchain provides finality guarantees, block ordering, and a source of randomness (via VRF). All Stacks state transitions are ultimately anchored to burnchain blocks.

**Code:** `stackslib/src/burnchains/mod.rs` (the `Burnchain` struct), `stacks-node/src/burnchains/` (controllers)

---

## C32

C32Check is the address encoding format used by Stacks, analogous to Bitcoin's Base58Check. It encodes a version byte and a 20-byte hash into a human-readable string using the Crockford Base32 alphabet (characters `0123456789ABCDEFGHJKMNPQRSTVWXYZ`). The encoding includes a 4-byte checksum for error detection. Stacks mainnet addresses start with `SP`, while testnet addresses start with `ST`.

**Code:** `stacks-common/src/address/c32.rs:370` (`c32_address` function), `stacks-common/src/address/c32.rs:362` (`c32_address_decode`)

---

## Clarity

Clarity is the smart contract language for Stacks. It is a decidable, non-Turing-complete language designed to be predictable and safe. Key properties include: no re-entrancy, no unbounded loops, static analysis of execution cost before runtime, and human-readable on-chain source code. Clarity has two major versions: Clarity 1 (Epoch 2.0+) and Clarity 2 (Epoch 2.1+), with Clarity 3 introduced in Epoch 3.0.

**Code:** `clarity/src/vm/` (the VM implementation), `clarity/src/vm/analysis/` (static analysis), `clarity/src/vm/costs/` (cost tracking)

---

## Consensus Hash

A 20-byte hash that uniquely identifies a specific sortition (leader election) on the burnchain. It is derived from the burnchain block hash, the previous consensus hash, and the proof-of-burn data for that sortition. The consensus hash chains together to form a tamper-evident history of all sortitions. It is used extensively as a key in the SortitionDB and to link Stacks blocks to their winning sortitions.

**Code:** `stacks-common/src/types/chainstate.rs` (the `ConsensusHash` type), `stackslib/src/chainstate/burn/db/sortdb.rs`

---

## DKG (Distributed Key Generation)

Distributed Key Generation is the protocol by which Stacks signers collectively generate a shared threshold public key without any single party learning the full private key. In Nakamoto, signers run DKG at the start of each reward cycle to produce a new aggregate key. This key is used for WSTS threshold signing of Stacks blocks. DKG failure means signers cannot produce valid signatures for that cycle.

**Code:** `stacks-signer/src/v0/signer.rs` (DKG orchestration), `libsigner/` (signing library interfaces)

---

## Epoch

An epoch is a distinct phase of the Stacks protocol defined by a range of burnchain block heights. Each epoch activates a specific set of consensus rules, execution cost limits, and protocol capabilities. The codebase defines epochs from 1.0 through 3.3, configured via `StacksEpochId`. Epoch transitions are irreversible and coordinated by burnchain height. For example, Epoch 3.0 activates Nakamoto consensus rules.

**Code:** `stacks-common/src/types/mod.rs` (`StacksEpochId` enum, `StacksEpoch<L>` generic struct at line 1085), `stackslib/src/core/mod.rs:42` (`StacksEpoch` type alias = `GenericStacksEpoch<ExecutionCost>`), `stackslib/src/config/mod.rs:1697-1714` (epoch config constants like `EPOCH_CONFIG_3_0_0`)

---

## EventDispatcher

The EventDispatcher is the node's webhook notification system. It sends JSON payloads via HTTP POST to configured observer endpoints whenever significant state changes occur (new blocks, mempool transactions, burn blocks, StackerDB chunks, etc.). Observers subscribe to specific event types using `EventKeyType` keys. The dispatcher uses a SQLite-backed queue for delivery reliability.

**Code:** `stacks-node/src/event_dispatcher.rs:182` (`EventDispatcher` struct), see Appendix C for full details.

---

## MARF (Merklized Adaptive Radix Forest)

The MARF is Stacks' authenticated key-value store, used for all chain state (account balances, contract data, nonces). It is a persistent trie structure where each historical state root is addressable, enabling efficient state proofs and rollbacks. The "forest" aspect means each block's state is a new trie layered on top of previous ones, sharing unchanged nodes. The MARF backs both the Clarity data store and the chainstate headers database.

**Code:** `stackslib/src/chainstate/stacks/index/marf.rs:40` (`MARF` struct), `stackslib/src/chainstate/stacks/index/` (trie nodes, storage, proofs)

---

## Microblock

Microblocks were a streaming mechanism in pre-Nakamoto Stacks (Epochs 2.0-2.4) that allowed the current tenure holder to produce small, incremental transaction batches between anchor blocks. This improved transaction throughput and reduced confirmation latency. Microblocks were confirmed by being included in the next anchor block. They have been removed in Nakamoto (Epoch 3.0+), replaced by the fast block production model where the tenure holder produces full Nakamoto blocks.

**Code:** `stackslib/src/chainstate/stacks/mod.rs` (`StacksMicroblock` struct)

---

## Nakamoto Block

A Nakamoto block is the block format introduced in Epoch 3.0. Unlike pre-Nakamoto blocks, Nakamoto blocks require threshold signatures from the active signer set (via WSTS) before they are considered valid. Multiple Nakamoto blocks can be produced within a single tenure (between two Bitcoin blocks). The block header includes a `signer_signature` field containing the aggregate Schnorr signature.

**Code:** `stackslib/src/chainstate/nakamoto/mod.rs:708` (`NakamotoBlock`), `stackslib/src/chainstate/nakamoto/mod.rs:633` (`NakamotoBlockHeader`)

---

## PoB (Proof of Burn)

Proof of Burn was the original consensus mechanism for Stacks (Epoch 1.0), where miners burned BTC (sent it to an unspendable address) to compete in sortition. The probability of winning was proportional to the amount burned relative to total burns. PoB was replaced by PoX (Proof of Transfer) in Epoch 2.0, which redirects the BTC to stackers instead of burning it.

**Code:** `stackslib/src/chainstate/burn/operations/` (burn operations)

---

## PoX (Proof of Transfer)

Proof of Transfer is the consensus mechanism that replaced PoB starting in Epoch 2.0. Instead of burning BTC, miners transfer it to STX holders who have "stacked" (locked) their tokens. Stackers receive BTC rewards proportional to their locked STX. PoX has evolved through multiple versions: PoX-1 (Epoch 2.0), PoX-2 (Epoch 2.1), PoX-3 (Epoch 2.4), and PoX-4 (Epoch 2.5+/Nakamoto). Each version is implemented as a boot contract.

**Code:** `stackslib/src/chainstate/stacks/boot/pox_4.rs` (current PoX), `stackslib/src/burnchains/mod.rs` (`PoxConstants` struct)

---

## Post-Condition

Post-conditions are a safety feature of Stacks transactions that protect users from unexpected asset transfers. A post-condition specifies an assertion about asset movements (STX, fungible tokens, or NFTs) that must hold true after a transaction executes. If any post-condition fails, the entire transaction is aborted (or only the call is aborted, depending on the mode). Post-conditions are checked after Clarity execution but before the transaction is committed.

**Code:** `stackslib/src/chainstate/stacks/mod.rs` (`TransactionPostCondition` enum), `clarity/src/vm/events.rs`

---

## Reward Cycle

A reward cycle is a fixed-length period measured in burnchain blocks during which the set of stackers eligible for BTC rewards remains constant. On mainnet, a reward cycle is 2100 Bitcoin blocks (approximately 2 weeks). Each cycle consists of a reward phase and a prepare phase. During the prepare phase, the next cycle's stacker set is determined. The `PoxConstants` struct defines `reward_cycle_length` and `prepare_length`.

**Code:** `stackslib/src/burnchains/mod.rs` (`PoxConstants` fields: `reward_cycle_length`, `prepare_length`)

---

## Shadow Block

A shadow block is a synthetic block created by the node when it encounters a missing Stacks block during chain processing. If a sortition has a committed block that cannot be retrieved from the network, the node may create a shadow block as a placeholder to maintain chain continuity. Shadow blocks contain no transactions and exist primarily as a recovery mechanism.

**Code:** `stackslib/src/chainstate/stacks/db/blocks.rs`

---

## Signer

In Nakamoto, signers are STX holders (or their delegates) who participate in the block validation process. After a miner produces a block, the active signer set validates it and produces a threshold signature using WSTS. A block requires signatures from signers representing at least 70% of the stacked STX for the current reward cycle. Signers communicate through StackerDB and the P2P network.

**Code:** `stacks-signer/src/` (the signer binary), `stacks-node/src/nakamoto_node/signer_coordinator.rs`

---

## Sortition

Sortition is the leader election process that selects which miner gets to produce the next Stacks block (or tenure, in Nakamoto). Miners submit block-commit transactions on Bitcoin, burning/transferring BTC. A VRF (Verifiable Random Function) is used to select the winner, with probability weighted by the amount committed. Each Bitcoin block triggers exactly one sortition, though it may be "empty" if no valid commits exist.

**Code:** `stackslib/src/chainstate/burn/db/sortdb.rs:754` (`SortitionDB`), `stackslib/src/chainstate/burn/operations/leader_block_commit.rs`

---

## Stacker

A stacker is an STX token holder who has locked their tokens through the PoX protocol to participate in consensus and earn BTC rewards. Stackers commit their STX for one or more reward cycles and specify a BTC address to receive mining transfers. In Nakamoto, stackers also serve as signers (or delegate to signers) who validate and co-sign blocks.

**Code:** `stackslib/src/chainstate/stacks/boot/pox_4.rs` (stacking contract)

---

## StackerDB

StackerDB is an off-chain replicated data store built on top of the Stacks P2P network. Each StackerDB instance is identified by a smart contract that defines the DB's access control policy (who can write to which slots). StackerDB is used for miner-signer coordination in Nakamoto: miners publish block proposals to the `.miners` StackerDB, and signers publish their signatures to `.signers-*-*` StackerDB contracts.

**Code:** `stackslib/src/net/stackerdb/mod.rs:225` (`StackerDBs`), `libstackerdb/src/libstackerdb.rs`, `stacks-node/src/nakamoto_node/stackerdb_listener.rs`

---

## STX

STX is the native token of the Stacks blockchain. It is used to pay transaction fees, participate in PoX stacking (earning BTC rewards), and as the unit of account within the Clarity VM. Balances are tracked in microSTX (1 STX = 1,000,000 microSTX) internally. The genesis distribution is defined in `stx-genesis/`.

**Code:** `stx-genesis/` (genesis balances), `clarity/src/vm/events.rs` (`STXEventType`)

---

## Tenure

A tenure is the period during which a single miner has the right to produce Stacks blocks. In pre-Nakamoto epochs, a tenure corresponds to exactly one anchor block (plus microblocks). In Nakamoto, a tenure spans the time between two successive Bitcoin blocks (sortitions), and the winning miner can produce multiple fast Nakamoto blocks within that tenure. A tenure begins with a tenure-change transaction and may be extended via a tenure-extend transaction.

**Code:** `stacks-node/src/tenure.rs`, `stackslib/src/chainstate/nakamoto/tenure.rs`

---

## Tenure-Extend

A tenure extension allows the current tenure holder to continue producing blocks beyond the normal tenure boundary. This can happen when: (1) the next sortition winner fails to produce blocks, (2) the next sortition is empty (no valid block commits), or (3) a time-based threshold is reached. The miner includes a `TenureExtend` transaction in a block to signal the extension, which must be approved by signers.

**Code:** `stackslib/src/chainstate/nakamoto/tenure.rs`, `stackslib/src/config/mod.rs:2958-2993` (tenure extend config: `tenure_extend_poll_timeout`, `tenure_extend_wait_timeout`, `tenure_timeout`)

---

## VRF (Verifiable Random Function)

A VRF is a cryptographic function that produces a pseudorandom output with a proof that the output was correctly computed. In Stacks, miners register a VRF public key on the burnchain and use the corresponding private key to produce VRF proofs in their block commits. The VRF output determines the sortition winner, providing unbiasable randomness anchored to the burnchain.

**Code:** `stacks-common/src/util/vrf.rs:214` (`VRFProof` struct), `stacks-common/src/util/vrf.rs:236` (`VRF_PROOF_ENCODED_SIZE`)

---

## WSTS (Weighted Schnorr Threshold Signing)

WSTS is the threshold signature scheme used in Nakamoto for block signing. Signers collectively hold shares of a signing key (produced via DKG), and any subset whose combined weight exceeds the threshold (70% of stacked STX) can produce a valid aggregate Schnorr signature. This enables decentralized block finalization without requiring all signers to be online.

**Code:** `stacks-signer/src/v0/signer.rs`, `libsigner/src/` (WSTS integration), `stacks-node/src/nakamoto_node/signer_coordinator.rs`

---

## Additional Terms

### Block Commit
A Bitcoin transaction submitted by a Stacks miner that commits to a specific Stacks block (or tenure, in Nakamoto). The block commit includes the miner's VRF proof, a hash of the proposed block, and BTC transferred to stackers' reward addresses.

**Code:** `stackslib/src/chainstate/burn/operations/leader_block_commit.rs`

### Chain ID
A 32-bit integer identifying the Stacks network. Mainnet uses `0x00000001` (`CHAIN_ID_MAINNET`), testnet uses `0x80000000` (`CHAIN_ID_TESTNET`). Used in transaction signing and P2P communication to prevent cross-network replay.

**Code:** `stackslib/src/core/mod.rs` (`CHAIN_ID_MAINNET`, `CHAIN_ID_TESTNET`)

### Coinbase Height
In Nakamoto, the coinbase height is the number of tenures (sortitions with block production) rather than the number of individual blocks. This decouples the block reward schedule from the block production rate.

### Magic Bytes
A 2-byte identifier in network messages that identifies the Stacks network variant. Mainnet uses `"X2"`, testnet uses `"T2"`. Configured via `burnchain.magic_bytes` in the node's TOML config.

**Code:** `stackslib/src/burnchains/mod.rs` (`MagicBytes`, `BLOCKSTACK_MAGIC_MAINNET`)

### Peer Version
A 32-bit value encoding the node's software version, used during P2P handshakes to ensure compatibility. Mainnet and testnet have separate version tracks (`PEER_VERSION_MAINNET`, `PEER_VERSION_TESTNET`).

**Code:** `stackslib/src/core/mod.rs` (`PEER_VERSION_MAINNET`, `PEER_VERSION_TESTNET`)
