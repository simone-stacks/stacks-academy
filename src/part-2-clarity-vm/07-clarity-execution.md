# Chapter 7: Clarity Execution -- The Runtime Interpreter

## Purpose

After a Clarity contract passes parsing and analysis, it is ready for execution. The Clarity runtime is a tree-walking interpreter: it recursively traverses the `SymbolicExpression` AST, evaluating each node against a layered context system. This chapter covers the execution model -- the context hierarchy (`GlobalContext`, `ContractContext`, `LocalContext`, `Environment`), the `eval` loop, how native functions are dispatched, and how events are emitted during execution.

## Key Concepts

**Tree-walking interpreter.** Clarity does not compile to bytecode. Each `SymbolicExpression` is evaluated directly. This makes the execution model simpler to reason about but means performance depends on AST structure.

**Layered context hierarchy.** Execution state is split across four context types that nest from outermost (transaction-level) to innermost (expression-level):
- `GlobalContext` -- one per transaction; holds the database, cost tracker, asset maps, events
- `ContractContext` -- one per contract; holds definitions (functions, variables, metadata)
- `LocalContext` -- one per lexical scope; holds local variable bindings
- `Environment` -- a convenience struct that bundles references to all three, plus sender/caller/sponsor

**No mutation of the call stack.** Clarity has no mutable local variables -- `let` bindings are immutable. State mutation happens only through `var-set`, `map-set`, and asset operations, all of which go through the database layer (Chapter 8).

## Architecture

The core execution loop is in `clarity/src/vm/mod.rs`. The entry point for evaluating a single expression is `eval` (line 328), which dispatches based on expression type. For contract initialization, `eval_all` (line 386) evaluates every top-level expression in sequence.

```
Transaction Processing
  |
  v
OwnedEnvironment
  |
  v
GlobalContext  <-- ClarityDatabase, LimitedCostTracker, AssetMap, Events
  |
  v
ContractContext  <-- functions, variables, persisted metadata
  |
  v
Environment  <-- bundles GlobalContext + ContractContext + CallStack + sender/caller
  |
  v
eval()  <-- recursive tree-walking evaluation
  |
  v
LocalContext  <-- variable bindings within let/match/function bodies
```

## Key Data Structures

### GlobalContext (`clarity/src/vm/contexts.rs:224`)

```rust
pub struct GlobalContext<'a, 'hooks> {
    asset_maps: Vec<AssetMap>,
    pub event_batches: Vec<(EventBatch, u64)>,
    pub database: ClarityDatabase<'a>,
    read_only: Vec<bool>,
    pub cost_track: LimitedCostTracker,
    pub mainnet: bool,
    pub epoch_id: StacksEpochId,
    pub chain_id: u32,
    pub eval_hooks: Option<Vec<&'hooks mut dyn EvalHook>>,
    pub execution_time_tracker: ExecutionTimeTracker,
}
```

Key design: `asset_maps` and `read_only` are **stacks** (Vec used as a stack). Each `contract-call?` pushes a new `AssetMap` and read-only flag. When the call completes successfully, the inner asset map is merged into the outer one via `commit_other`. On failure, it is simply dropped (rolled back).

`event_batches` collects events emitted during execution. Each batch tracks a size counter (the `u64`) to enforce `MAX_EVENTS_BATCH` (50 MB).

`execution_time_tracker` enforces wall-clock time limits. At every `eval` call, `check_max_execution_time_expired` is called to prevent runaway execution.

### ContractContext (`clarity/src/vm/contexts.rs:240`)

```rust
pub struct ContractContext {
    pub contract_identifier: QualifiedContractIdentifier,
    pub variables: HashMap<ClarityName, Value>,
    pub functions: HashMap<ClarityName, DefinedFunction>,
    pub defined_traits: HashMap<ClarityName, BTreeMap<ClarityName, FunctionSignature>>,
    pub implemented_traits: HashSet<TraitIdentifier>,
    pub persisted_names: HashSet<ClarityName>,
    pub meta_data_map: HashMap<ClarityName, DataMapMetadata>,
    pub meta_data_var: HashMap<ClarityName, DataVariableMetadata>,
    pub meta_nft: HashMap<ClarityName, NonFungibleTokenMetadata>,
    pub meta_ft: HashMap<ClarityName, FungibleTokenMetadata>,
    pub data_size: u64,
    clarity_version: ClarityVersion,
}
```

This holds all contract-level state visible during execution. `variables` stores `define-constant` values. `functions` stores `define-private`, `define-public`, and `define-read-only` functions. The `meta_*` maps store metadata for persisted storage types (data maps, variables, NFTs, FTs) which the database layer needs for key construction.

### LocalContext (`clarity/src/vm/contexts.rs:259`)

```rust
pub struct LocalContext<'a> {
    pub function_context: Option<&'a LocalContext<'a>>,
    pub parent: Option<&'a LocalContext<'a>>,
    pub variables: HashMap<ClarityName, Value>,
    pub callable_contracts: HashMap<ClarityName, CallableData>,
    depth: u16,
}
```

A chain of lexical scopes. `parent` points to the enclosing scope (e.g., a `let` inside a `let`). `function_context` points to the scope at the function boundary, enabling the interpreter to look up function-level bindings. Variable lookup walks the chain from innermost to outermost.

The `depth` field is checked against `MAX_CONTEXT_DEPTH` (256) to prevent stack overflow from deeply nested `let` expressions.

### Environment (`clarity/src/vm/contexts.rs:67`)

```rust
pub struct Environment<'a, 'b, 'hooks> {
    pub global_context: &'a mut GlobalContext<'b, 'hooks>,
    pub contract_context: &'a ContractContext,
    pub call_stack: &'a mut CallStack,
    pub sender: Option<PrincipalData>,
    pub caller: Option<PrincipalData>,
    pub sponsor: Option<PrincipalData>,
}
```

The `Environment` is a convenience struct that bundles all the context pieces needed by eval functions. It is recreated whenever context changes -- notably during `contract-call?`, where a new `Environment` is created with a different `ContractContext`.

`sender` is `tx-sender` in Clarity. `caller` is `contract-caller`. `sponsor` is the optional transaction sponsor. During `as-contract`, the sender is swapped to the contract's principal.

### CallStack (`clarity/src/vm/contexts.rs:267`)

```rust
pub struct CallStack {
    stack: Vec<FunctionIdentifier>,
    set: HashSet<FunctionIdentifier>,
    apply_depth: usize,
}
```

Tracks the current call chain. The `set` provides O(1) recursion detection -- if a `FunctionIdentifier` is already in the set when a user function is called, a `CircularReference` error is raised. `apply_depth` tracks how deeply nested the current `apply` evaluation is (not the same as function call depth).

### AssetMap (`clarity/src/vm/contexts.rs:95`)

```rust
pub struct AssetMap {
    stx_map: HashMap<PrincipalData, u128>,
    burn_map: HashMap<PrincipalData, u128>,
    token_map: HashMap<PrincipalData, HashMap<AssetIdentifier, u128>>,
    asset_map: HashMap<PrincipalData, HashMap<AssetIdentifier, Vec<Value>>>,
    stacking_map: HashMap<PrincipalData, u128>,
}
```

Tracks all asset movements during a transaction. This is used for post-condition checking -- after execution, the transaction processor compares the `AssetMap` against the transaction's declared post-conditions to decide whether to commit or abort.

## Code Walkthrough: The eval Loop

### `eval` (`clarity/src/vm/mod.rs:328`)

The heart of the interpreter:

```rust
pub fn eval(
    exp: &SymbolicExpression,
    env: &mut Environment,
    context: &LocalContext,
) -> Result<Value, VmExecutionError> {
    check_max_execution_time_expired(env.global_context)?;

    // Pre-eval hooks (for Clarinet/tooling)
    if let Some(mut eval_hooks) = env.global_context.eval_hooks.take() {
        for hook in eval_hooks.iter_mut() {
            hook.will_begin_eval(env, context, exp);
        }
        env.global_context.eval_hooks = Some(eval_hooks);
    }

    let res = match exp.expr {
        AtomValue(ref value) | LiteralValue(ref value) => Ok(value.clone()),
        Atom(ref value) => lookup_variable(value, context, env),
        List(ref children) => {
            let (function_variable, rest) = children.split_first()
                .ok_or(RuntimeCheckErrorKind::NonFunctionApplication)?;
            let function_name = function_variable.match_atom()
                .ok_or(RuntimeCheckErrorKind::BadFunctionName)?;
            let f = lookup_function(function_name, env)?;
            apply(&f, rest, env, context)
        }
        TraitReference(_, _) | Field(_) => {
            Err(VmInternalError::BadSymbolicRepresentation(...).into())
        }
    };

    // Post-eval hooks
    // ...
    res
}
```

The dispatch is simple:
- **AtomValue / LiteralValue** -- Return the value directly (constants, literals).
- **Atom** -- Look up a variable by name through the context chain.
- **List** -- The first element is the function name, the rest are arguments. Look up the function and apply it.
- **TraitReference / Field** -- These should not appear in evaluated positions; they indicate a parser or analysis bug.

### `lookup_variable` (`clarity/src/vm/mod.rs:164`)

Variable resolution follows a priority chain:

1. **Reserved variables** (`variables::lookup_reserved_variable`) -- `tx-sender`, `contract-caller`, `block-height`, `stx-liquid-supply`, etc.
2. **Local context** (`context.lookup_variable(name)`) -- `let` bindings, function parameters
3. **Contract context** (`env.contract_context.lookup_variable(name)`) -- `define-constant` values
4. **Callable contracts** (`context.lookup_callable_contract(name)`) -- Clarity 2 callable contract references

Each lookup charges a cost: `LookupVariableDepth` based on context chain depth, and `LookupVariableSize` based on the value's size.

### `lookup_function` (`clarity/src/vm/mod.rs:203`)

Function resolution:

1. **Reserved (native) functions** (`functions::lookup_reserved_functions`) -- Built-in functions like `+`, `map-get?`, `stx-transfer?`
2. **User-defined functions** (`env.contract_context.lookup_function(name)`) -- Functions from `define-private`, `define-public`, `define-read-only`

If neither matches, an `UndefinedFunction` error is raised.

### `apply` (`clarity/src/vm/mod.rs:230`)

Function application has two paths:

**Special functions** (e.g., `if`, `let`, `match`, `and`, `or`, `begin`, `contract-call?`): These receive unevaluated arguments (as `&[SymbolicExpression]`) because they need to control evaluation order. For example, `if` only evaluates one branch. The special function handler is called directly:

```rust
if let CallableType::SpecialFunction(_, function) = function {
    env.call_stack.insert(&identifier, track_recursion);
    let mut resp = function(args, env, context);
    // ...
}
```

**Native and user functions**: Arguments are evaluated eagerly (left to right), then the function is called with the resulting values:

```rust
let mut evaluated_args = Vec::with_capacity(args.len());
for arg_x in args.iter() {
    let arg_value = eval(arg_x, env, context)?;
    // Track memory usage
    env.add_memory(arg_value.get_memory_use()?)?;
    evaluated_args.push(arg_value);
}
```

Memory tracking ensures that accumulating evaluated arguments does not exceed the memory limit.

For user functions (`CallableType::UserFunction`), the `DefinedFunction::apply` method creates a new `LocalContext` with parameters bound to argument values, then evaluates the function body.

**Recursion detection**: Before calling a user function, `apply` checks `env.call_stack.contains(&identifier)`. Clarity intentionally forbids recursion -- this is a deliberate design choice to make all Clarity programs terminate.

**Stack depth limit**: `MAX_CALL_STACK_DEPTH` (64) is enforced before any function call.

## Native Functions: The functions/ Directory

Native functions are organized by domain in `clarity/src/vm/functions/`:

| File | Domain | Key Functions |
|------|--------|---------------|
| `arithmetic.rs` | Math | `+`, `-`, `*`, `/`, `mod`, `pow`, `sqrti`, `log2` |
| `boolean.rs` | Logic | `and`, `or`, `not`, `if` |
| `sequences.rs` | Collections | `map`, `fold`, `filter`, `append`, `concat`, `len`, `list` |
| `tuples.rs` | Tuples | `tuple`, `get`, `merge` |
| `database.rs` | Storage | `var-get`, `var-set`, `map-get?`, `map-set`, `map-insert`, `map-delete` |
| `assets.rs` | Tokens | `stx-transfer?`, `ft-mint?`, `nft-mint?`, `ft-burn?`, etc. |
| `options.rs` | Optionals/Results | `some`, `ok`, `err`, `unwrap!`, `match`, `default-to` |
| `crypto.rs` | Hashing | `hash160`, `sha256`, `sha512`, `keccak256`, `secp256k1-recover?` |
| `conversions.rs` | Type conversion | `to-int`, `to-uint`, `buff-to-int-le`, `int-to-ascii` |
| `principals.rs` | Principal ops | `principal-of?`, `principal-construct?`, `principal-destruct?` |
| `define.rs` | Definitions | `define-constant`, `define-public`, `define-map`, etc. |
| `post_conditions.rs` | Post-conditions | `restrict-assets?`, `with-stx` |

The `NativeFunctions` enum (`clarity/src/vm/functions/mod.rs:85`) maps every built-in function name to its implementation. It is a versioned enum -- each variant specifies the minimum `ClarityVersion` it was introduced in and an optional maximum version.

Functions are registered via a `define_versioned_named_enum_with_max!` macro which generates `lookup_by_name` methods that respect version boundaries.

### Function Dispatch Categories

Native functions fall into three `CallableType` variants:

1. **`SpecialFunction`** -- Receives unevaluated `&[SymbolicExpression]`. Used for control flow (`if`, `let`, `begin`, `match`, `and`, `or`, `asserts!`, `unwrap!`, `try!`, `contract-call?`, `as-contract`, `at-block`).

2. **`NativeFunction`** -- Receives pre-evaluated `Vec<Value>`. Cost is computed as `runtime_cost(cost_function, env, args.len())`. Used for most pure functions.

3. **`NativeFunction205`** -- Like `NativeFunction` but with a custom cost input function. Introduced in Epoch 2.05 to compute more accurate costs based on actual argument values rather than just argument count.

## Events During Execution

Events are emitted through the `GlobalContext.event_batches` mechanism. The event types are defined in `clarity/src/vm/events.rs:26`:

```rust
pub enum StacksTransactionEvent {
    SmartContractEvent(SmartContractEventData),
    STXEvent(STXEventType),
    NFTEvent(NFTEventType),
    FTEvent(FTEventType),
}
```

STX events include transfers, mints, burns, and lock events (for stacking). NFT and FT events track transfers, mints, and burns.

Events are emitted by the native functions in `assets.rs`. For example, `stx-transfer?` emits an `STXTransferEvent` with sender, recipient, and amount. The `print` function emits a `SmartContractEvent` containing an arbitrary Clarity value.

Events are batched per contract call depth. When a `contract-call?` succeeds, its event batch is committed to the parent. When it fails, the batch is discarded along with the database changes and asset map.

## Contract Initialization: `eval_all`

When a contract is first deployed, `eval_all` (`clarity/src/vm/mod.rs:386`) processes every top-level expression:

```rust
pub fn eval_all(
    expressions: &[SymbolicExpression],
    contract_context: &mut ContractContext,
    global_context: &mut GlobalContext,
    sponsor: Option<PrincipalData>,
) -> Result<Option<Value>, VmExecutionError> {
    let mut last_executed = None;
    let publisher: PrincipalData = contract_context.contract_identifier.issuer.clone().into();

    for exp in expressions {
        // ...
        let try_define = functions::define::evaluate_define(exp, &mut env)?;
        match try_define {
            DefineResult::Variable(name, value) => {
                // Store in contract_context.variables
            }
            DefineResult::Function(name, value) => {
                // Store in contract_context.functions
            }
            DefineResult::Map(name, key_type, value_type) => {
                // Create in database
            }
            DefineResult::NoDefine => {
                // Evaluate as an expression
                last_executed = Some(eval(exp, &mut env, &context));
            }
            // ...
        }
    }
    Ok(last_executed)
}
```

Top-level expressions that are definitions (`define-constant`, `define-public`, `define-map`, etc.) are handled by `evaluate_define` which returns a `DefineResult`. Non-definition expressions are evaluated normally -- their return value is tracked, and the last one becomes the contract's deployment result.

## How It Connects

- **Upstream**: Receives the `SymbolicExpression` AST from parsing (Chapter 5) and the `ContractAnalysis` from analysis (Chapter 6).
- **Downstream**: All state mutations go through `ClarityDatabase` (Chapter 8). Cost tracking goes through `LimitedCostTracker` (Chapter 9).
- **Cross-contract calls**: `contract-call?` loads the target contract's `ContractContext` from the database, creates a new `Environment`, and calls `eval_all` or the specific function. The caller's `GlobalContext` is shared, so cost tracking and event collection span the entire transaction.

## Edge Cases and Gotchas

1. **No recursion, ever.** The `CallStack.set` check prevents all recursion, including mutual recursion. This is by design -- Clarity guarantees termination.

2. **`as-contract` swaps the sender but not the caller.** When code runs inside `as-contract`, `tx-sender` becomes the contract's principal, but `contract-caller` remains unchanged. This is important for permission checks.

3. **Special functions see unevaluated arguments.** This means `(if condition (expensive-fn) (cheap-fn))` only evaluates one branch. This is critical for gas efficiency and correctness.

4. **Memory tracking is per-argument.** Each evaluated argument's memory cost is tracked and added. If adding a single argument exceeds the memory limit, the remaining arguments are not evaluated and the error propagates up.

5. **Events are soft-limited, not hard-limited.** `MAX_EVENTS_BATCH` (50 MB) is checked, but the check happens at the batch level, not per-event. A single very large `print` event could exceed the limit.

6. **The EvalHook mechanism is how Clarinet works.** The `will_begin_eval` and `did_finish_eval` hooks allow external tools to observe every evaluation step. This powers Clarinet's debugger and coverage tracking. The hooks are stored behind `Option<Vec<...>>` and are `take()`-then-replace()`d at each eval call to satisfy the borrow checker.
