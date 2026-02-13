# Chapter 6: Clarity Analysis -- Static Checking Before Execution

## Purpose

After parsing produces a `ContractAST`, the analysis pipeline determines whether the contract is safe to execute. This is Clarity's static analysis phase: it type-checks every expression, verifies read-only correctness, validates trait implementations, and checks whether the contract qualifies as a cost function. Analysis happens at deploy time -- a contract that fails analysis is rejected and never stored on-chain. This is fundamentally different from languages like Solidity where type errors are caught at compile time but the deployed bytecode can still fail in unexpected ways.

## Key Concepts

**Analysis is a prerequisite for execution.** Every contract deployed on Stacks must pass analysis. The analysis result (`ContractAnalysis`) is persisted alongside the contract and consulted when other contracts call into this one.

**Epoch-aware type checkers.** Two complete type checker implementations exist: `TypeChecker2_05` (for Epoch 2.0 and 2.05) and `TypeChecker2_1` (for Epoch 2.1+). The 2.1 checker added support for new Clarity 2 features like callable types. The correct checker is selected at runtime based on the epoch.

**Ordered passes.** The analysis passes must execute in a specific order:
1. `ReadOnlyChecker` -- must run first because it determines which functions are read-only
2. `TypeChecker` -- depends on read-only annotations
3. `TraitChecker` -- depends on type information from the type checker
4. `ArithmeticOnlyChecker` -- a post-pass that checks cost contract eligibility

## Architecture

The analysis entry point is `run_analysis` in `clarity/src/vm/analysis/mod.rs:122`:

```rust
pub fn run_analysis(
    contract_identifier: &QualifiedContractIdentifier,
    expressions: &[SymbolicExpression],
    analysis_db: &mut AnalysisDatabase,
    save_contract: bool,
    cost_tracker: LimitedCostTracker,
    epoch: StacksEpochId,
    version: ClarityVersion,
    build_type_map: bool,
) -> Result<ContractAnalysis, Box<(StaticCheckError, LimitedCostTracker)>>
```

The function creates a `ContractAnalysis` struct, then runs each pass inside an `analysis_db.execute()` transaction. If any pass fails, the database transaction rolls back. If all passes succeed and `save_contract` is true, the analysis result is persisted.

## Key Data Structures

### ContractAnalysis (`clarity/src/vm/analysis/types.rs:46`)

The central output of analysis, persisted on-chain:

```rust
pub struct ContractAnalysis {
    pub contract_identifier: QualifiedContractIdentifier,
    pub private_function_types: BTreeMap<ClarityName, FunctionType>,
    pub variable_types: BTreeMap<ClarityName, TypeSignature>,
    pub public_function_types: BTreeMap<ClarityName, FunctionType>,
    pub read_only_function_types: BTreeMap<ClarityName, FunctionType>,
    pub map_types: BTreeMap<ClarityName, (TypeSignature, TypeSignature)>,
    pub persisted_variable_types: BTreeMap<ClarityName, TypeSignature>,
    pub fungible_tokens: BTreeSet<ClarityName>,
    pub non_fungible_tokens: BTreeMap<ClarityName, TypeSignature>,
    pub defined_traits: BTreeMap<ClarityName, BTreeMap<ClarityName, FunctionSignature>>,
    pub implemented_traits: BTreeSet<TraitIdentifier>,
    pub contract_interface: Option<ContractInterface>,
    pub is_cost_contract_eligible: bool,
    pub epoch: StacksEpochId,
    pub clarity_version: ClarityVersion,
    // Not serialized:
    pub expressions: Vec<SymbolicExpression>,
    pub type_map: Option<TypeMap>,
    pub cost_track: Option<LimitedCostTracker>,
}
```

Key fields:
- `public_function_types` / `read_only_function_types` -- maps function names to their full type signatures. Used by other contracts at analysis time to validate `contract-call?` arguments.
- `map_types` -- maps data map names to their (key-type, value-type) pairs.
- `defined_traits` -- maps trait names to their function signature maps. Used by the trait checker to validate `impl-trait`.
- `type_map` -- a transient (non-serialized) mapping from expression IDs to their inferred types. Only built when `build_type_map` is true (used by developer tools).
- `cost_track` -- the `LimitedCostTracker` is threaded through analysis and returned to the caller, even on error, so cost accounting is preserved.

### AnalysisDatabase (`clarity/src/vm/analysis/analysis_db.rs:32`)

```rust
pub struct AnalysisDatabase<'a> {
    store: RollbackWrapper<'a>,
}
```

A wrapper around `RollbackWrapper` that provides analysis-specific operations: loading/saving `ContractAnalysis`, looking up function types from previously deployed contracts, and querying trait definitions. The `execute` method (`analysis_db.rs:46`) wraps operations in a begin/commit/rollback transaction:

```rust
pub fn execute<F, T, E>(&mut self, f: F) -> Result<T, E>
where
    F: FnOnce(&mut Self) -> Result<T, E>,
{
    self.begin();
    let result = f(self).or_else(|e| {
        self.roll_back()?;
        Err(e)
    })?;
    self.commit()?;
    Ok(result)
}
```

## Code Walkthrough: The Analysis Passes

### Pass 1: ReadOnlyChecker (`clarity/src/vm/analysis/read_only_checker/mod.rs`)

The `ReadOnlyChecker` validates that functions declared as `define-read-only` do not modify chain state. It also determines whether private functions are effectively read-only.

```rust
pub struct ReadOnlyChecker<'a, 'b> {
    db: &'a mut AnalysisDatabase<'b>,
    defined_functions: HashMap<ClarityName, bool>,
    epoch: StacksEpochId,
    clarity_version: ClarityVersion,
}
```

The `defined_functions` map tracks whether each function in the contract is read-only (`true`) or not (`false`). The checker works by iterating over all top-level expressions (`run` at `read_only_checker/mod.rs:87`):

1. For `define-read-only` functions, it recursively checks every expression in the body to ensure no write operations occur (no `var-set`, `map-set`, `map-insert`, `map-delete`, `stx-transfer?`, `ft-transfer?`, `nft-transfer?`, `ft-mint?`, `nft-mint?`, `ft-burn?`, `nft-burn?`).

2. For `define-private` and `define-public` functions, it still analyzes them to determine if they are *effectively* read-only, recording the result in `defined_functions`. This information is used when a read-only function calls a private function -- the private function must also be read-only.

3. For `contract-call?` to external contracts, the checker looks up the target function's type in the `AnalysisDatabase`. If the target is a read-only function, it is safe to call from a read-only context.

**Error produced**: `StaticCheckErrorKind::WriteAttemptedInReadOnly` -- emitted when a write operation is found inside a read-only function body.

### Pass 2: TypeChecker

The type checker is the most complex analysis pass. Two versions exist:

- **v2_05** (`clarity/src/vm/analysis/type_checker/v2_05/mod.rs`) -- Used for Epoch 2.0 and 2.05
- **v2_1** (`clarity/src/vm/analysis/type_checker/v2_1/mod.rs`) -- Used for Epoch 2.1+

Selection happens in `run_analysis` (`analysis/mod.rs:141`):

```rust
match epoch {
    StacksEpochId::Epoch20 | StacksEpochId::Epoch2_05 => {
        TypeChecker2_05::run_pass(&epoch, &mut contract_analysis, db, build_type_map)
    }
    StacksEpochId::Epoch21 | /* ... */ StacksEpochId::Epoch33 => {
        TypeChecker2_1::run_pass(&epoch, &mut contract_analysis, db, build_type_map)
    }
    // ...
}
```

The type checker traverses every expression and infers its type. For each expression, it:

1. **Resolves atoms** -- Looks up variables and constants in the current type context.
2. **Checks function applications** -- When it encounters a list like `(+ 1 2)`, it looks up the type signature of `+`, checks that the arguments conform, and returns the result type.
3. **Processes definitions** -- For `define-constant`, `define-data-var`, `define-map`, `define-fungible-token`, `define-non-fungible-token`, it records the types in `ContractAnalysis`.
4. **Handles special forms** -- `if`, `let`, `match`, `begin`, etc. have custom type-checking logic.
5. **Cross-contract calls** -- For `contract-call?`, the checker loads the target contract's `ContractAnalysis` from the database and validates argument types.

The type checker uses context objects (in `clarity/src/vm/analysis/type_checker/contexts.rs`, shared by both v2_05 and v2_1) to track:
- `TypingContext` -- the current variable-to-type bindings
- `ContractContext` -- the contract-level definitions seen so far

Native functions have their type checking rules in the `natives/` subdirectory, organized by category:
- `natives/options.rs` -- `some`, `none`, `unwrap!`, `match`, etc.
- `natives/sequences.rs` -- `map`, `fold`, `filter`, `len`, `append`, etc.
- `natives/maps.rs` -- `map-get?`, `map-set`, `map-insert`, `map-delete`
- `natives/assets.rs` -- `ft-transfer?`, `nft-mint?`, `stx-transfer?`, etc.
- `natives/conversions.rs` -- `to-int`, `to-uint`, `buff-to-int-le`, etc.

### Pass 3: TraitChecker (`clarity/src/vm/analysis/trait_checker/mod.rs`)

The trait checker validates `impl-trait` declarations. For each trait a contract claims to implement, the checker:

1. Loads the trait definition from the `AnalysisDatabase` (it may be defined in another contract).
2. Calls `ContractAnalysis::check_trait_compliance` (`analysis/types.rs:228`), which iterates over every function in the trait definition and verifies:
   - The contract has a public or read-only function with the matching name
   - The function's argument types are compatible
   - The function's return type is compatible

Type compatibility uses `admits_type`, which handles subtyping rules (e.g., `(response uint uint)` admits `(response uint uint)`).

### Pass 4: ArithmeticOnlyChecker (`clarity/src/vm/analysis/arithmetic_checker/mod.rs`)

This pass checks whether a contract is eligible to be used as a **cost function**. Cost functions in Stacks are themselves Clarity contracts, but they are restricted to a safe subset: only arithmetic, comparison, and control flow operations. No state reads, no cross-contract calls, no asset operations.

The result is stored in `contract_analysis.is_cost_contract_eligible`. This flag is used by the cost-voting mechanism (Chapter 9) to validate proposed cost function replacements.

## The AnalysisDatabase in Detail

The `AnalysisDatabase` provides several key lookup functions used during analysis:

- `load_contract(contract_id, epoch)` -- Loads a previously stored `ContractAnalysis`, canonicalizing types for the current epoch.
- `get_public_function_type(contract_id, function_name, epoch)` -- Returns the type signature of a public function.
- `get_read_only_function_type(contract_id, function_name, epoch)` -- Returns the type signature of a read-only function.
- `get_defined_trait(contract_id, trait_name, epoch)` -- Returns a trait's function signature map.
- `get_implemented_traits(contract_id)` -- Returns all traits a contract implements.
- `insert_contract(contract_id, analysis)` -- Persists a new contract's analysis. Fails with `ContractAlreadyExists` if the contract has already been analyzed.

All type lookups go through `load_contract_non_canonical` followed by `.canonicalize(epoch)`. Type canonicalization is necessary because types changed representation between epochs (e.g., `TraitReferenceType` was replaced by `CallableType(Trait(...))` in Epoch 2.1).

## Type Canonicalization

`ContractAnalysis::canonicalize_types` (`analysis/types.rs:198`) walks all stored types and converts them to the canonical form for the target epoch. This is called every time a contract's analysis is loaded for cross-contract reference. The motivation is backward compatibility: contracts deployed in Epoch 2.0 stored types in the old format, but contracts in Epoch 2.1+ need to reason about those types using the new format.

## How It Connects

- **Upstream**: Receives `SymbolicExpression` from the parsing pipeline (Chapter 5). The expressions in `ContractAST.expressions` become `ContractAnalysis.expressions`.
- **Downstream**: The `ContractAnalysis` is stored on-chain and used during execution (Chapter 7) for cross-contract call validation. The `type_map` is used by developer tools for hover-type information.
- **Cost tracking**: Analysis charges costs through the `LimitedCostTracker` embedded in `ContractAnalysis.cost_track`. Major cost functions include `AnalysisTypeCheck`, `AnalysisTypeAnnotate`, `AnalysisVisit`, `AnalysisBindName`, etc. (Chapter 9).

## Edge Cases and Gotchas

1. **Analysis results are epoch-stamped.** A contract's `ContractAnalysis` records the `epoch` and `clarity_version` it was analyzed under. When a newer contract references it, types are canonicalized to the newer epoch's format. This means the same contract can appear to have different types depending on which epoch is loading it.

2. **The cost tracker survives errors.** When analysis fails, the error return is `Box<(StaticCheckError, LimitedCostTracker)>` -- the cost tracker is always returned so the transaction processor can account for the cost consumed before failure.

3. **`build_type_map` is optional.** In production (`build_type_map = true` in `run_analysis`), the type map is always built because it is needed for the contract interface. But the infrastructure supports skipping it for performance in contexts where only validation matters.

4. **Read-only is a static property.** Once the `ReadOnlyChecker` labels a function as read-only, that label is permanent. A function cannot be "sometimes read-only" -- if any code path could write, the entire function is not read-only.

5. **Cross-contract analysis loads the whole contract.** When type-checking a `contract-call?`, the system loads the entire `ContractAnalysis` of the target contract just to extract one function's type. The code has `TODO` comments noting this could be optimized by storing function types as separate metadata entries.

6. **Trait compliance is structural, not nominal.** A contract satisfies a trait if it has public/read-only functions with matching names and compatible signatures. It does not need to import the trait definition -- `impl-trait` just triggers the compliance check. The functions can have broader argument types (contravariance) and narrower return types (covariance).
