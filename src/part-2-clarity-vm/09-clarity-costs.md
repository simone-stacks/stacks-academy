# Chapter 9: Clarity Costs -- Metering Execution

## Purpose

Every operation in Clarity costs something. The cost system ensures that transaction processing has predictable resource consumption, prevents denial-of-service attacks via expensive contracts, and allows the network to price block space. This chapter covers the `CostTracker` trait, the `LimitedCostTracker` implementation, the `ExecutionCost` struct with its five-dimensional model, cost functions and their evolution across epochs, and the on-chain cost-voting mechanism that allows the community to update cost parameters.

## Key Concepts

**Five-dimensional cost model.** Clarity does not have a single "gas" number. Instead, each operation produces a cost vector with five dimensions: `runtime`, `read_count`, `read_length`, `write_count`, `write_length`. A transaction's total cost in each dimension must stay below the block limit for that dimension. This prevents a transaction from being cheap in CPU time but expensive in I/O, or vice versa.

**Cost functions are Clarity contracts.** In production, cost functions are defined in boot contracts (`costs`, `costs-2`, `costs-3`, `costs-4`). Each cost function takes an input size and returns an `ExecutionCost` tuple. The community can vote to replace cost functions via the `cost-voting` boot contract.

**Compile-time fallback.** For performance, the default cost functions are hard-coded in Rust (the `costs_1.rs`, `costs_2.rs`, `costs_3.rs`, `costs_4.rs` modules). These are used as the "fast path" when no custom cost overrides have been voted in. Only when a community-voted replacement exists does the system actually execute a Clarity cost function.

**Free tracking for tests.** The `LimitedCostTracker::Free` variant disables all cost checking. The `()` implementation of `CostTracker` returns `ExecutionCost::ZERO` for everything. Both are used extensively in tests.

## Architecture

```
Clarity eval loop
    |
    v
runtime_cost(ClarityCostFunction, tracker, input_size)
    |
    v
CostTracker::compute_cost(function, input)  --> ExecutionCost
    |
    v
CostTracker::add_cost(cost)  --> checks against limit, returns CostErrors if exceeded
    |
    v
(also: add_memory / drop_memory for memory tracking)
```

## Key Data Structures

### ExecutionCost (`clarity/src/vm/costs/mod.rs` -- re-exported from `clarity_types`)

```rust
pub struct ExecutionCost {
    pub write_length: u64,
    pub write_count: u64,
    pub read_length: u64,
    pub read_count: u64,
    pub runtime: u64,
}
```

Each dimension is tracked independently:

- **runtime** -- CPU cycles / computational complexity. The dominant dimension for arithmetic-heavy contracts.
- **read_count** -- Number of storage reads. Counts each `get_data` call.
- **read_length** -- Total bytes read from storage. Scales with the size of data map values.
- **write_count** -- Number of storage writes. Counts each `put_all_data` item.
- **write_length** -- Total bytes written to storage. Scales with the size of values being stored.

`ExecutionCost` provides a constant `ZERO` for free operations and implements `CostOverflowingMath` for checked addition across all dimensions.

### CostTracker Trait (`clarity/src/vm/costs/mod.rs:128`)

```rust
pub trait CostTracker {
    fn compute_cost(
        &mut self,
        cost_function: ClarityCostFunction,
        input: &[u64],
    ) -> Result<ExecutionCost, CostErrors>;

    fn add_cost(&mut self, cost: ExecutionCost) -> Result<(), CostErrors>;
    fn add_memory(&mut self, memory: u64) -> Result<(), CostErrors>;
    fn drop_memory(&mut self, memory: u64) -> Result<(), CostErrors>;
    fn reset_memory(&mut self);

    fn short_circuit_contract_call(
        &mut self,
        contract: &QualifiedContractIdentifier,
        function: &ClarityName,
        input: &[u64],
    ) -> Result<bool, CostErrors>;
}
```

The trait has two roles:
1. **Cost computation** -- `compute_cost` takes a cost function identifier and input sizes, returning the resulting `ExecutionCost`.
2. **Cost enforcement** -- `add_cost` adds the cost to the running total and returns `CostErrors::CostBalanceExceeded` if any dimension exceeds the limit.

The `short_circuit_contract_call` method enables **cost short-circuiting**: for known boot contract calls (like PoX functions), the system can charge a pre-computed cost without actually executing the function. This is an optimization for expensive but well-understood operations.

### The () Implementation (`clarity/src/vm/costs/mod.rs:150`)

The unit type implements `CostTracker` as a no-op -- all costs return `ExecutionCost::ZERO`, all operations succeed. Used during AST parsing and in tests where cost tracking is not needed.

### LimitedCostTracker (`clarity/src/vm/costs/mod.rs:348`)

```rust
pub enum LimitedCostTracker {
    Limited(TrackerData),
    Free,
}
```

The production cost tracker. `Free` is used in tests and for boot contract initialization. `Limited` contains `TrackerData`:

```rust
pub struct TrackerData {
    cost_function_references: HashMap<&'static ClarityCostFunction, ClarityCostFunctionEvaluator>,
    cost_contracts: HashMap<QualifiedContractIdentifier, ContractContext>,
    contract_call_circuits: HashMap<(QualifiedContractIdentifier, ClarityName), ClarityCostFunctionReference>,
    total: ExecutionCost,
    limit: ExecutionCost,
    memory: u64,
    memory_limit: u64,
    pub epoch: StacksEpochId,
    mainnet: bool,
    chain_id: u32,
}
```

Key fields:
- `cost_function_references` -- maps each `ClarityCostFunction` to its evaluator (either a default Rust implementation or a Clarity contract)
- `contract_call_circuits` -- maps specific contract-call targets to pre-computed cost functions (the short-circuit mechanism)
- `total` -- the accumulated cost so far
- `limit` -- the maximum cost allowed (set from the block's execution cost limit)
- `memory` / `memory_limit` -- separate memory tracking (default limit: `CLARITY_MEMORY_LIMIT` = 100 MB)

## ClarityCostFunction Enum (`clarity/src/vm/costs/cost_functions.rs`)

Every metered operation has a named cost function. The enum has over 100 variants:

```rust
define_named_enum!(ClarityCostFunction {
    AnalysisTypeAnnotate("cost_analysis_type_annotate"),
    AnalysisTypeCheck("cost_analysis_type_check"),
    AstParse("cost_ast_parse"),
    AstCycleDetection("cost_ast_cycle_detection"),
    LookupVariableDepth("cost_lookup_variable_depth"),
    LookupVariableSize("cost_lookup_variable_size"),
    LookupFunction("cost_lookup_function"),
    Add("cost_add"),
    Sub("cost_sub"),
    Mul("cost_mul"),
    Div("cost_div"),
    Hash160("cost_hash160"),
    Sha256("cost_sha256"),
    FetchEntry("cost_fetch_entry"),
    SetEntry("cost_set_entry"),
    FetchVar("cost_fetch_var"),
    SetVar("cost_set_var"),
    ContractCall("cost_contract_call"),
    StxTransfer("cost_stx_transfer"),
    FtMint("cost_ft_mint"),
    NftMint("cost_nft_mint"),
    // ... many more
});
```

These are grouped into categories:
- **Analysis costs** (`AnalysisTypeAnnotate`, `AnalysisTypeCheck`, `AnalysisVisit`, etc.) -- charged during static analysis
- **AST costs** (`AstParse`, `AstCycleDetection`) -- charged during parsing
- **Lookup costs** (`LookupVariableDepth`, `LookupVariableSize`, `LookupFunction`) -- charged at every variable/function reference
- **Arithmetic costs** (`Add`, `Sub`, `Mul`, `Div`, `Mod`, `Pow`, etc.)
- **Crypto costs** (`Hash160`, `Sha256`, `Sha512`, `Keccak256`, `Secp256k1recover`)
- **Storage costs** (`FetchEntry`, `SetEntry`, `FetchVar`, `SetVar`, `CreateMap`, `CreateVar`)
- **Asset costs** (`StxTransfer`, `FtMint`, `FtTransfer`, `NftMint`, `NftTransfer`)

Each cost function takes input sizes (typically the byte size of the operation's input) and returns an `ExecutionCost`.

## Cost Function Versions

Cost functions have evolved across Stacks epochs:

### DefaultVersion (`clarity/src/vm/costs/mod.rs:190`)

```rust
pub enum DefaultVersion {
    Costs1,
    Costs2,
    Costs2Testnet,
    Costs3,
    Costs4,
}
```

- **Costs1** (`costs_1.rs`) -- The original cost functions from Stacks 2.0
- **Costs2** (`costs_2.rs`) -- Updated for Epoch 2.05 mainnet
- **Costs2Testnet** (`costs_2_testnet.rs`) -- Testnet variant with different parameters
- **Costs3** (`costs_3.rs`) -- Updated for Epoch 2.1+
- **Costs4** (`costs_4.rs`) -- Updated for Epoch 3.x (Nakamoto)

Each version is a struct that implements cost evaluation functions. The `DefaultVersion::evaluate` method (`costs/mod.rs:200`) dispatches to the correct version:

```rust
pub fn evaluate(
    &self,
    cost_function_ref: &ClarityCostFunctionReference,
    f: &ClarityCostFunction,
    input: &[u64],
) -> Result<ExecutionCost, CostErrors> {
    let n = input.first()?;
    match self {
        DefaultVersion::Costs1 => f.eval::<Costs1>(*n),
        DefaultVersion::Costs2 => f.eval::<Costs2>(*n),
        DefaultVersion::Costs2Testnet => f.eval::<Costs2Testnet>(*n),
        DefaultVersion::Costs3 => f.eval::<Costs3>(*n),
        DefaultVersion::Costs4 => f.eval::<Costs4>(*n),
    }
}
```

## The `runtime_cost` Helper

The most common way to charge costs in Clarity code:

```rust
pub fn runtime_cost<T: TryInto<u64>, C: CostTracker>(
    cost_function: ClarityCostFunction,
    tracker: &mut C,
    input: T,
) -> Result<(), CostErrors> {
    let size: u64 = input.try_into().map_err(|_| CostErrors::CostOverflow)?;
    let cost = tracker.compute_cost(cost_function, &[size])?;
    tracker.add_cost(cost)
}
```

This is used throughout the codebase. For example, in the eval loop:

```rust
// Before looking up a variable:
runtime_cost(ClarityCostFunction::LookupVariableDepth, env, context.depth())?;

// Before evaluating a hash:
runtime_cost(ClarityCostFunction::Hash160, env, input_value.size()?)?;

// Before a storage read:
runtime_cost(ClarityCostFunction::FetchEntry, env, key_size)?;
```

## Memory Tracking

Separate from the five-dimensional cost model, Clarity tracks memory usage:

```rust
pub const CLARITY_MEMORY_LIMIT: u64 = 100 * 1000 * 1000; // 100 MB
```

Memory is tracked through `add_memory` and `drop_memory`. The main consumers:
- Evaluated function arguments (tracked in `apply` in `vm/mod.rs:257`)
- Values constructed during `let` bindings
- Large value constructions (buffers, lists)

Memory is "freed" when values go out of scope (e.g., when `apply` finishes evaluating arguments, it calls `drop_memory` for the total argument memory).

## The Cost-Voting Mechanism

Stacks includes an on-chain governance mechanism for updating cost functions. The `cost-voting` boot contract allows STX holders to:

1. **Propose** a new cost function for a specific `ClarityCostFunction`
2. **Vote** on proposals by locking STX
3. **Confirm** proposals that reach sufficient votes

### Loading Cost State (`clarity/src/vm/costs/mod.rs:496`)

`load_cost_functions` reads the current cost state:

```rust
fn load_cost_functions(
    mainnet: bool,
    clarity_db: &mut ClarityDatabase,
    apply_updates: bool,
) -> // ...
```

If `apply_updates` is true, the function checks for newly confirmed proposals in the `cost-voting` contract and applies them. The resulting `CostStateSummary` is cached.

### CostStateSummary (`clarity/src/vm/costs/mod.rs:277`)

```rust
pub struct CostStateSummary {
    pub contract_call_circuits: HashMap<
        (QualifiedContractIdentifier, ClarityName),
        ClarityCostFunctionReference
    >,
    pub cost_function_references: HashMap<
        ClarityCostFunction,
        ClarityCostFunctionReference
    >,
}
```

- `cost_function_references` -- maps cost functions to their replacements (if any were voted in)
- `contract_call_circuits` -- maps specific contract calls to pre-computed cost functions

The summary is serialized as JSON and stored in the `cost-voting` contract's metadata space.

### ClarityCostFunctionEvaluator (`clarity/src/vm/costs/mod.rs:258`)

```rust
pub enum ClarityCostFunctionEvaluator {
    Default(ClarityCostFunctionReference, ClarityCostFunction, DefaultVersion),
    Clarity(ClarityCostFunctionReference),
}
```

- **Default** -- Use the hard-coded Rust implementation. This is the common case.
- **Clarity** -- Execute an actual Clarity contract to compute the cost. This happens when a community-voted replacement is in effect.

When a `Clarity` evaluator is used, the system loads the replacement contract's `ContractContext` (cached in `TrackerData.cost_contracts`) and executes it with the input size. The contract must return a tuple with `runtime`, `read_count`, `read_length`, `write_count`, `write_length` fields. The `ArithmeticOnlyChecker` (Chapter 6) ensures that only safe operations are used in cost contracts.

## How Costs Flow Through Execution

1. **Transaction begins**: A `LimitedCostTracker` is created with the block's cost limits.
2. **Parsing**: `build_ast` charges `AstParse` (proportional to source code length) and `AstCycleDetection` (proportional to dependency graph edges).
3. **Analysis**: The type checker charges per-expression costs (`AnalysisVisit`, `AnalysisTypeAnnotate`, `AnalysisTypeCheck`, `AnalysisBindName`, etc.).
4. **Execution**: Every `eval` and `apply` call charges appropriate costs. Variable lookups charge `LookupVariableDepth` + `LookupVariableSize`. Function lookups charge `LookupFunction`. Each native function charges its specific cost.
5. **At any point**: If `add_cost` causes any dimension to exceed the limit, a `CostErrors::CostBalanceExceeded` error propagates up, aborting execution.
6. **Transaction ends**: The final `total` cost is extracted from the tracker and used for block limit accounting.

## How It Connects

- **Upstream**: Called by the parsing pipeline (Chapter 5), analysis pipeline (Chapter 6), and execution engine (Chapter 7) at every metered operation.
- **Downstream**: The block limit (an `ExecutionCost` value) comes from the epoch configuration. The remaining budget after processing a transaction determines how many more transactions fit in the block.
- **Governance**: The `cost-voting` boot contract allows the community to update cost functions. The `ArithmeticOnlyChecker` (Chapter 6) validates that proposed cost contracts are safe.

## Edge Cases and Gotchas

1. **Cost is charged even on failure.** If a transaction exceeds its cost budget, the transaction fails but the cost consumed up to that point is still charged to the block. This prevents "free" failed transactions from filling blocks.

2. **Five dimensions means five possible failure modes.** A transaction can fail by exceeding any single dimension. In practice, `runtime` is the most common bottleneck for compute-heavy contracts, while `read_length` and `write_length` are bottlenecks for data-heavy contracts.

3. **Cost functions can change mid-chain.** When the community votes in a new cost function, it takes effect at the next block boundary. This means the same Clarity operation can have different costs on different blocks. The `epoch` field in `TrackerData` ensures the correct cost schedule is used.

4. **Memory tracking is separate from cost.** A transaction can exhaust its memory budget (`CLARITY_MEMORY_LIMIT` = 100 MB) even if it still has cost budget remaining. Memory tracking prevents a single transaction from consuming excessive RAM during execution.

5. **The `()` CostTracker returns ZERO for everything.** This means that in test code using `build_ast(&id, source, &mut (), ...)`, no cost is charged and no cost limits are enforced. This is intentional but means tests do not exercise cost-related failure modes unless they explicitly use `LimitedCostTracker`.

6. **Short-circuiting avoids double-counting.** When `short_circuit_contract_call` returns `true`, the actual contract call is skipped and only the pre-computed cost is charged. This is used for boot contracts where the cost is well-known and execution is expensive. Without this optimization, calling PoX functions would be prohibitively expensive because both the cost of evaluating the cost function AND the cost of the actual function would be charged.

7. **Costs4 is the current default for Nakamoto.** As of Epoch 3.x, `Costs4` is the default cost schedule. It includes cost functions for new Clarity 3/4 features like `get-stacks-block-info?`, `get-tenure-info?`, and `as-contract?`.
