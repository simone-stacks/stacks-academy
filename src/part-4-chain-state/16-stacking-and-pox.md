# Chapter 16: Stacking and Proof of Transfer (PoX)

## Purpose

Proof of Transfer (PoX) is the consensus mechanism that connects Stacks to Bitcoin. Miners spend BTC to win the right to produce Stacks blocks, and that BTC is sent to STX holders who "stack" (lock) their tokens. This chapter covers the PoX boot contracts (pox-1 through pox-4), how reward sets are calculated, the `pox-locking` crate that enforces STX locks at the Rust level, and how stacking state transitions work across epochs.

The PoX boot contracts live in `stackslib/src/chainstate/stacks/boot/` (as `.clar` files). The Rust-level lock enforcement lives in the `pox-locking/` crate. Reward set computation and contract orchestration happen in `stackslib/src/chainstate/stacks/boot/mod.rs`.

## Key Concepts

### The PoX Cycle

PoX operates on **reward cycles**, which are fixed-length periods of burnchain (Bitcoin) blocks. Each reward cycle has two phases:

1. **Prepare phase**: The last few blocks of the cycle. During this phase, the reward set for the *next* cycle is finalized by reading the stacking state from the anchor block.
2. **Reward phase**: The remaining blocks. Miners send their BTC to addresses in the active reward set.

The cycle parameters are configured in `PoxConstants`:

- `reward_cycle_length`: Total blocks per cycle (e.g., 2100 on mainnet)
- `prepare_length`: Number of blocks in the prepare phase (e.g., 100)
- `anchor_threshold`: Fraction of prepare-phase blocks that must confirm the anchor block

### Stacking Flow

1. A STX holder calls `stack-stx` on the active PoX contract, specifying:
   - The amount of STX to lock
   - A Bitcoin reward address (where BTC will be sent)
   - The number of reward cycles to lock for
2. The PoX contract validates the request (minimum threshold, valid address, etc.)
3. The `pox-locking` crate applies the lock to the user's `STXBalance` in the Clarity database
4. At the start of the next reward cycle's prepare phase, the reward set is computed by reading all active stacking entries
5. During the reward phase, miners must send BTC to addresses in the reward set

### The Four PoX Contracts

Stacks has gone through four versions of the PoX contract, each introduced at a specific epoch:

| Contract | Epoch | Purpose |
|----------|-------|---------|
| `pox` (pox-1) | Genesis - 2.1 | Original PoX contract |
| `pox-2` | 2.1 - 2.2 | Fixed bugs, added `stack-extend` and `stack-increase` |
| `pox-3` | 2.2 - 2.5 | Addressed security issue in pox-2 |
| `pox-4` | 2.5+ | Added signer key registration for Nakamoto |

When a new PoX version activates, the previous version becomes "defunct" -- write operations are rejected (but read-only calls still work). This transition is enforced at the Rust level in the `pox-locking` crate.

## Boot Contract Architecture

### Contract Loading

Boot contracts are compiled into the binary as string constants:

```rust
// stackslib/src/chainstate/stacks/boot/mod.rs:51-86
const BOOT_CODE_POX_BODY: &str = std::include_str!("pox.clar");
const BOOT_CODE_POX_TESTNET_CONSTS: &str = std::include_str!("pox-testnet.clar");
const BOOT_CODE_POX_MAINNET_CONSTS: &str = std::include_str!("pox-mainnet.clar");
const POX_2_BODY: &str = std::include_str!("pox-2.clar");
const POX_3_BODY: &str = std::include_str!("pox-3.clar");
const POX_4_BODY: &str = std::include_str!("pox-4.clar");
```

The PoX contracts for pox-1, pox-2, and pox-3 have network-specific constants (reward cycle length, prepare length, etc.) that are prepended to the contract body:

```rust
// stackslib/src/chainstate/stacks/boot/mod.rs:116-128
lazy_static! {
    pub static ref BOOT_CODE_POX_MAINNET: String =
        format!("{BOOT_CODE_POX_MAINNET_CONSTS}\n{BOOT_CODE_POX_BODY}");
    pub static ref BOOT_CODE_POX_TESTNET: String =
        format!("{BOOT_CODE_POX_TESTNET_CONSTS}\n{BOOT_CODE_POX_BODY}");
    pub static ref POX_2_MAINNET_CODE: String =
        format!("{BOOT_CODE_POX_MAINNET_CONSTS}\n{POX_2_BODY}");
    // ...
    pub static ref POX_4_CODE: String = POX_4_BODY.to_string();
}
```

Note that pox-4 does not need network-specific constants prepended -- it reads configuration from the chain state directly.

### Contract Names and Discovery

```rust
// stackslib/src/chainstate/stacks/boot/mod.rs:63-70
pub const POX_1_NAME: &str = "pox";
pub const POX_2_NAME: &str = "pox-2";
pub const POX_3_NAME: &str = "pox-3";
pub const POX_4_NAME: &str = "pox-4";
pub const SIGNERS_NAME: &str = "signers";
pub const SIGNERS_VOTING_NAME: &str = "signers-voting";
pub const SIP_031_NAME: &str = "sip-031";
```

The `PoxVersions` enum provides type-safe version discrimination:

```rust
// stackslib/src/chainstate/stacks/boot/mod.rs:218-223
define_named_enum!(PoxVersions {
    Pox1("pox"),
    Pox2("pox-2"),
    Pox3("pox-3"),
    Pox4("pox-4"),
});
```

## The pox-locking Crate

The `pox-locking` crate (`pox-locking/src/`) bridges the Clarity contract layer with the Rust-level account balance system. When a PoX contract call succeeds, the crate intercepts the result and applies the corresponding lock to the user's `STXBalance`.

### The Central Dispatcher

```rust
// pox-locking/src/lib.rs:66-150
pub fn handle_contract_call_special_cases(
    global_context: &mut GlobalContext,
    sender: Option<&PrincipalData>,
    _sponsor: Option<&PrincipalData>,
    contract_id: &QualifiedContractIdentifier,
    function_name: &str,
    args: &[Value],
    result: &Value,
) -> Result<(), VmExecutionError>
```

This function is called after every contract call. It matches the `contract_id` against the known PoX contracts and dispatches to the appropriate handler. Before dispatching, it checks epoch validity -- calling a defunct PoX contract in a post-transition epoch raises `RuntimeError::DefunctPoxContract`:

```rust
// pox-locking/src/lib.rs:75-93
if *contract_id == boot_code_id(POX_1_NAME, global_context.mainnet) {
    if !pox_1::is_read_only(function_name)
        && global_context.database.get_v1_unlock_height()
            <= global_context.database.get_current_burnchain_block_height()?
    {
        return Err(VmExecutionError::Runtime(
            RuntimeError::DefunctPoxContract, None,
        ));
    }
    return pox_1::handle_contract_call(global_context, sender, function_name, result);
}
```

### Locking Mechanics (pox-4 example)

The `pox_lock_v4` function (pox-locking/src/pox_4.rs:36-65) applies an STX lock:

```rust
pub fn pox_lock_v4(
    db: &mut ClarityDatabase,
    principal: &PrincipalData,
    lock_amount: u128,
    unlock_burn_height: u64,
) -> Result<(), LockingError> {
    assert!(unlock_burn_height > 0);
    assert!(lock_amount > 0);

    let mut snapshot = db.get_stx_balance_snapshot(principal)?;

    if snapshot.has_locked_tokens()? {
        return Err(LockingError::PoxAlreadyLocked);
    }
    if !snapshot.can_transfer(lock_amount)? {
        return Err(LockingError::PoxInsufficientBalance);
    }
    snapshot.lock_tokens_v4(lock_amount, unlock_burn_height)?;
    snapshot.save()?;
    Ok(())
}
```

The `STXBalance` struct in the Clarity database tracks both liquid and locked STX for each account. The lock records the amount and the burnchain height at which the tokens automatically unlock.

Lock operations:
- **`stack-stx`** -> `pox_lock_v4()`: Initial lock
- **`stack-extend`** -> `pox_lock_extend_v4()`: Extend the unlock height (tokens stay locked longer)
- **`stack-increase`** -> `pox_lock_increase_v4()`: Lock additional STX in the same cycle
- **`delegate-stack-stx`**: Delegated stacking (pool operator stacks on behalf of the user)

### Locking Errors

```rust
// pox-locking/src/lib.rs:43-52
pub enum LockingError {
    DefunctPoxContract,
    PoxAlreadyLocked,
    PoxInsufficientBalance,
    PoxExtendNotLocked,
    PoxIncreaseOnV1,
    PoxInvalidIncrease,
    Clarity(VmExecutionError),
}
```

These errors are translated to `Error::PoxAlreadyLocked`, `Error::PoxInsufficientBalance`, etc. in the chainstate error type.

## Reward Set Computation

### The Threshold

At the start of a reward cycle's prepare phase, the system computes which stackers qualify for reward slots. The threshold is determined by `get_reward_threshold_and_participation` (boot/mod.rs:950-982):

```rust
pub fn get_reward_threshold_and_participation(
    pox_settings: &PoxConstants,
    addresses: &[RawRewardSetEntry],
    liquid_ustx: u128,
) -> (u128, u128) {
    let participation = addresses
        .iter()
        .fold(0, |agg, entry| agg + entry.amount_stacked);

    // Scale by at least 25% of liquid supply
    let scale_by = cmp::max(participation, liquid_ustx / POX_MAXIMAL_SCALING);

    let reward_slots = u128::try_from(pox_settings.reward_slots()).unwrap();
    let threshold_precise = scale_by / reward_slots;
    // Round up to nearest POX_THRESHOLD_STEPS_USTX (10,000 STX)
    let ceil_amount = match threshold_precise % POX_THRESHOLD_STEPS_USTX {
        0 => 0,
        remainder => POX_THRESHOLD_STEPS_USTX - remainder,
    };
    let threshold = threshold_precise + ceil_amount;
    (threshold, participation)
}
```

Key rules:
- The threshold is based on total participation divided by reward slots
- Minimum scaling is 25% of liquid STX supply (`POX_MAXIMAL_SCALING = 4`)
- The threshold is rounded up to the nearest 10,000 STX increment

### Building the Reward Set

`get_reward_set_entries_at_block` (boot/mod.rs:2775-2788) retrieves and sorts all stacking entries:

```rust
pub fn get_reward_set_entries_at_block(
    state: &mut StacksChainState,
    burnchain: &Burnchain,
    sortdb: &SortitionDB,
    block_id: &StacksBlockId,
    burn_block_height: u64,
) -> Result<Vec<RawRewardSetEntry>, Error> {
    state
        .get_reward_addresses(burnchain, sortdb, burn_block_height, block_id)
        .map(|mut addrs| {
            addrs.sort_by_key(|k| k.reward_address.bytes());
            addrs
        })
}
```

Each entry in the raw set:

```rust
// stackslib/src/chainstate/stacks/boot/mod.rs:203-209
pub struct RawRewardSetEntry {
    pub reward_address: PoxAddress,
    pub amount_stacked: u128,
    pub stacker: Option<PrincipalData>,
    pub signer: Option<[u8; SIGNERS_PK_LEN]>,  // 33 bytes, Nakamoto only
}
```

The `signer` field was added for Nakamoto: each stacker must register a signing key that participates in block validation.

### The RewardSet

After filtering by threshold, the final reward set is:

```rust
// stackslib/src/chainstate/stacks/boot/mod.rs:264-273
pub struct RewardSet {
    pub rewarded_addresses: Vec<PoxAddress>,
    pub start_cycle_state: PoxStartCycleInfo,
    pub signers: Option<Vec<NakamotoSignerEntry>>,  // Nakamoto only
    pub pox_ustx_threshold: Option<u128>,
}
```

In Nakamoto, the `signers` field maps stacking participation to signing weight:

```rust
// stackslib/src/chainstate/stacks/boot/mod.rs:256-262
pub struct NakamotoSignerEntry {
    pub signing_key: [u8; 33],
    pub stacked_amt: u128,
    pub weight: u32,
}
```

The `weight` determines how many of the 4000 reward slots (`SIGNERS_MAX_LIST_SIZE`) a signer controls. Weight is proportional to the amount stacked.

## Automatic Unlocking

When a stacker's lock period expires, their STX automatically unlock at the start of the next reward cycle. The block processor handles this during `setup_block` by checking the current burnchain height against each account's `unlock_burn_height`.

The `PoxStartCycleInfo` struct tracks stackers who missed reward slots:

```rust
// stackslib/src/chainstate/stacks/boot/mod.rs:226-234
pub struct PoxStartCycleInfo {
    pub missed_reward_slots: Vec<(PrincipalData, u128)>,
}
```

Stackers who miss slots (e.g., due to threshold changes) get early-unlocked to avoid indefinite lockup.

## Delegated Stacking

Users who do not meet the minimum stacking threshold can delegate to a pool operator. The flow is:

1. User calls `delegate-stx` on the PoX contract, granting delegation to a pool operator
2. The pool operator calls `delegate-stack-stx` to lock the delegated STX
3. The pool operator calls `stack-aggregation-commit` to commit the aggregated STX to a reward address

Delegated stacking introduces a `DelegateStxOp` burnchain operation, allowing delegation to be initiated from Bitcoin.

## Epoch Transitions and Contract Upgrades

Each PoX contract transition follows the same pattern:

1. The new epoch activates at a configured burnchain height
2. The new PoX contract is deployed as a boot contract
3. The old contract is marked "defunct" -- write operations return `DefunctPoxContract`
4. Existing locks may be migrated or remain valid under the new contract
5. Users must re-stack under the new contract when their current lock expires

The `pox-locking` crate enforces the defunct check before dispatching:

```rust
// pox-locking/src/lib.rs:95-108 (example: pox-2 defunct after Epoch 2.2)
} else if *contract_id == boot_code_id(POX_2_NAME, global_context.mainnet) {
    if !pox_2::is_read_only(function_name) && global_context.epoch_id >= StacksEpochId::Epoch22
    {
        return Err(VmExecutionError::Runtime(
            RuntimeError::DefunctPoxContract, None,
        ));
    }
    return pox_2::handle_contract_call(/* ... */);
}
```

## How It Connects

- **Boot Contracts** (Chapter 16, this chapter): The PoX contracts are Clarity programs deployed at genesis.
- **Sortition DB** (Chapter 12): The reward set feeds into the sortition algorithm, determining where miners send BTC.
- **Block Processing** (Chapter 15): `setup_block` processes burnchain stacking operations and auto-unlocks expired locks.
- **Bitcoin Integration** (Chapter 10): Miners include `StackStxOp`, `DelegateStxOp`, and `TransferStxOp` as Bitcoin transactions.
- **Signers** (Chapter 27): In Nakamoto, the reward set's `signers` field determines the active signer set for block validation.
- **Clarity VM** (Chapters 5-9): PoX contracts execute within the Clarity VM; the `pox-locking` crate is a special-case handler.

## Edge Cases and Gotchas

1. **Minimum stacking amount**: The minimum is dynamic, based on total liquid supply and the number of reward slots. It is always a multiple of 10,000 STX.

2. **Reward address sorting**: The reward set is sorted by reward address bytes (`sort_by_key(|k| k.reward_address.bytes())`). This deterministic ordering ensures all nodes compute the same reward set.

3. **Stack-extend timing**: A stacker can only extend their lock before it expires. Calling `stack-extend` after the unlock height has passed will fail because `has_locked_tokens()` returns false.

4. **Pool operator trust**: Delegated stacking requires trusting the pool operator to commit the aggregated STX. The protocol does not enforce that the operator actually commits -- the user's STX are locked regardless.

5. **pox-4 signer keys**: Starting with pox-4, every stacker must register a `signer-key`. This key is used in the Nakamoto signer protocol. Stacking without a valid signer key is not possible in Epoch 2.5+.

6. **Defunct contract read-only access**: Read-only functions on defunct PoX contracts still work. This allows querying historical stacking state even after a contract upgrade. Only state-changing functions are blocked.

7. **Assertion panics in locking**: The `pox_lock_v4` function uses `assert!` for `unlock_burn_height > 0` and `lock_amount > 0`. These should always be enforced by the Clarity contract, but the Rust assertions provide defense in depth.
