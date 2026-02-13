# Chapter 5: Clarity Parsing -- From Source Code to AST

## Purpose

Before a Clarity contract can be type-checked or executed, its source code must be transformed into a structured Abstract Syntax Tree (AST). The parsing pipeline in stacks-core is not a single function call -- it is a seven-pass pipeline that lexes, parses, validates depth, assigns IDs, sorts definitions, resolves trait references, and expands syntactic sugar. Understanding this pipeline is essential because every contract deployment on Stacks runs through it, and the pipeline's design directly impacts what contracts are accepted or rejected before execution ever begins.

## Key Concepts

**Two-representation model.** Clarity parsing produces two different expression types. First, `PreSymbolicExpression` is the raw output of the parser -- it includes comments, placeholders for errors, and sugared forms like `.contract-name`. After all passes complete, the sugar expander converts these into `SymbolicExpression` values, which are the final form consumed by the type checker and interpreter.

**Multi-pass architecture.** Rather than a monolithic parser, the pipeline is a sequence of independent passes, each implementing the `BuildASTPass` trait or a similar interface. This makes each pass testable in isolation and allows the system to collect as many diagnostics as possible rather than bailing on the first error.

**Epoch-aware parsing.** Stacks has shipped two parser versions. Parser v1 is used for Epoch 2.0 and earlier; parser v2 (a complete rewrite with a proper lexer) is used from Epoch 2.1 onward. The `parse_in_epoch` function in `clarity/src/vm/ast/mod.rs:60` selects the correct parser based on the current epoch.

## Architecture: The Seven-Pass Pipeline

The entire pipeline is orchestrated by `inner_build_ast` in `clarity/src/vm/ast/mod.rs:119`. Here is the sequence:

```
Source code (string)
  |
  v
[1] Lexer + Parser (v1 or v2)  -->  Vec<PreSymbolicExpression>
  |
  v
[2] StackDepthChecker           -->  Validates nesting depth (lists only)
  |
  v
[3] VaryStackDepthChecker       -->  Validates nesting depth (lists + tuples)
  |
  v
[4] ExpressionIdentifier (pre)  -->  Assigns unique IDs to PreSymbolicExpressions
  |
  v
[5] DefinitionSorter            -->  Topological sort of top-level definitions
  |
  v
[6] TraitsResolver              -->  Resolves define-trait, use-trait, impl-trait
  |
  v
[7] SugarExpander               -->  Converts PreSymbolicExpression to SymbolicExpression
  |
  v
[8] ExpressionIdentifier (post) -->  Reassigns IDs to final SymbolicExpressions
  |
  v
ContractAST (ready for analysis)
```

## Key Data Structures

### ContractAST (`clarity/src/vm/ast/types.rs:30`)

The central output of the parsing pipeline:

```rust
pub struct ContractAST {
    pub contract_identifier: QualifiedContractIdentifier,
    pub pre_expressions: Vec<PreSymbolicExpression>,
    pub expressions: Vec<SymbolicExpression>,
    pub top_level_expression_sorting: Option<Vec<usize>>,
    pub referenced_traits: HashMap<ClarityName, TraitDefinition>,
    pub implemented_traits: HashSet<TraitIdentifier>,
}
```

- `pre_expressions` -- filled by the parser (pass 1), consumed and drained by the sugar expander (pass 7)
- `expressions` -- filled by the sugar expander, this is the final AST consumed by analysis and execution
- `top_level_expression_sorting` -- an indirection table produced by `DefinitionSorter`; entries are indices into `pre_expressions` in dependency order
- `referenced_traits` and `implemented_traits` -- populated by `TraitsResolver`

### PreExpressionsDrain (`clarity/src/vm/ast/types.rs:71`)

An iterator that yields `PreSymbolicExpression` values in the order dictated by `top_level_expression_sorting`. When the sugar expander calls `contract_ast.pre_expressions_drain()`, it gets definitions in dependency-sorted order rather than source order. This is critical: a constant defined later in the file is still available to an earlier function if the sorter determined the dependency.

### Token (`clarity/src/vm/ast/parser/v2/lexer/token.rs`)

The lexer token enum includes variants for every syntactic element in Clarity:

- `Lparen`, `Rparen`, `Lbrace`, `Rbrace` -- delimiters
- `Int(String)`, `Uint(String)` -- numeric literals (stored as strings until parsing validates them)
- `AsciiString(String)`, `Utf8String(String)` -- string literals
- `Bytes(String)` -- hex buffer literals like `0xdeadbeef`
- `Principal(String)` -- principal literals like `'SP000...`
- `Ident(String)` -- identifiers (function names, variable names)
- `TraitIdent(String)` -- trait references like `<my-trait>`
- `Placeholder(String)` -- error recovery; invalid tokens are wrapped here so parsing can continue
- `Comment(String)` -- preserved for developer tooling

## Code Walkthrough: The Pipeline Step by Step

### Pass 1: Lexer and Parser

**Lexer** (`clarity/src/vm/ast/parser/v2/lexer/mod.rs:17`):

The `Lexer` struct is a character-by-character scanner with explicit tracking of line/column positions for span information:

```rust
pub struct Lexer<'a> {
    input: Chars<'a>,
    next: char,
    offset: usize,
    pub line: usize,
    pub column: usize,
    pub diagnostics: Vec<PlacedError>,
    pub success: bool,
    fail_fast: bool,
}
```

The lexer operates in two modes controlled by `fail_fast`:

- **fail_fast = true** (production): Returns `Err` on the first lexer error. Used during block processing where invalid contracts should be rejected immediately.
- **fail_fast = false** (developer tools): Collects diagnostics and continues, inserting `Token::Placeholder` for invalid tokens. Used by the Clarity LSP and other tooling.

The main entry point is `read_token()` (`clarity/src/vm/ast/parser/v2/lexer/mod.rs:706`), which uses a large `match` on the current character to dispatch to specialized reading functions like `read_identifier`, `read_unsigned`, `read_ascii_string`, `read_hex`, etc.

Key design detail: the lexer uses `EOF` (the Unicode replacement character `U+FFFD`) as a sentinel for end-of-input rather than wrapping every character in `Option`.

**Parser** (`clarity/src/vm/ast/parser/v2/mod.rs:32`):

```rust
pub struct Parser<'a> {
    lexer: Lexer<'a>,
    tokens: Vec<PlacedToken>,
    next_token: usize,
    nesting_depth: u64,
    // ...
}
```

The parser eagerly lexes all tokens into a `Vec<PlacedToken>` during construction, then walks through them to build `PreSymbolicExpression` nodes. It tracks `nesting_depth` against `MAX_NESTING_DEPTH` (which equals `AST_CALL_STACK_DEPTH_BUFFER + MAX_CALL_STACK_DEPTH + 1`) to reject excessively nested contracts at parse time.

The parser handles two structural forms:

- **Lists**: delimited by `(` and `)`, producing `PreSymbolicExpressionType::List`
- **Tuples**: delimited by `{` and `}` with `key: value` pairs, producing `PreSymbolicExpressionType::Tuple`

Sugared forms like `.contract-name` (relative contract identifiers) are parsed into `PreSymbolicExpressionType::SugaredContractIdentifier`, which will later be expanded by the sugar expander.

### Pass 2-3: Stack Depth Checkers

`StackDepthChecker` (`clarity/src/vm/ast/stack_depth_checker.rs:45`) recursively walks the AST counting only `List` nesting depth. `VaryStackDepthChecker` (`clarity/src/vm/ast/stack_depth_checker.rs:71`) does the same but also counts `Tuple` nesting. Both enforce a limit of `AST_CALL_STACK_DEPTH_BUFFER + MAX_CALL_STACK_DEPTH` (which is `5 + 64 = 69`).

The two-checker design exists because tuples increase AST depth without corresponding call stack depth at runtime, but extremely deep tuples can still cause stack overflows during processing. The buffer of 5 provides headroom.

### Pass 4: Expression Identifier (Pre-pass)

`ExpressionIdentifier::run_pre_expression_pass` (`clarity/src/vm/ast/expression_identifier/mod.rs:47`) assigns monotonically increasing IDs to every `PreSymbolicExpression` node in the AST via `inner_relabel`. These IDs are used by the definition sorter to track dependencies -- when it finds an `Atom` reference, it can look up which top-level definition that atom belongs to by its ID.

### Pass 5: Definition Sorter

`DefinitionSorter` (`clarity/src/vm/ast/definition_sorter/mod.rs:37`) performs a topological sort of top-level definitions. This is one of the most important passes because Clarity requires forward references to be resolved -- a constant defined on line 50 can be used by a function on line 10.

The algorithm:

1. **Build a name map**: Scan all top-level expressions to find definitions (`define-constant`, `define-public`, etc.) and map their names to expression indices.

2. **Build a dependency graph**: For each top-level expression, recursively scan for `Atom` references that match a defined name, adding directed edges in the graph.

3. **Cost accounting**: Before running the sort, the function charges `ClarityCostFunction::AstCycleDetection` based on the number of edges in the graph (`clarity/src/vm/ast/definition_sorter/mod.rs:83`).

4. **Topological sort + cycle detection**: A `GraphWalker` produces a sorted order. If cycles are detected, a `CircularReference` error is returned listing the involved function names.

5. The sorted indices are stored in `contract_ast.top_level_expression_sorting`.

### Pass 6: Traits Resolver

`TraitsResolver` (`clarity/src/vm/ast/traits_resolver/mod.rs:31`) scans for three definition forms:

- **`define-trait`**: Records the trait definition (name + function signatures) in `contract_ast.referenced_traits` as a `TraitDefinition::Defined`.
- **`use-trait`**: Imports a trait from another contract, recording it as `TraitDefinition::Imported`.
- **`impl-trait`**: Records that this contract implements a specific trait in `contract_ast.implemented_traits`.

The resolver also detects name collisions -- defining a trait with a name already used by an imported trait is an error.

### Pass 7: Sugar Expander

`SugarExpander` (`clarity/src/vm/ast/sugar_expander/mod.rs:29`) performs the final transformation from `PreSymbolicExpression` to `SymbolicExpression`. Key transformations:

- **`SugaredContractIdentifier`** (`.my-contract`) becomes a fully-qualified `Value::Principal` by prepending the deployer's address.
- **`SugaredFieldIdentifier`** (`.my-contract.my-trait`) becomes a `SymbolicExpression::field` with a fully-qualified `TraitIdentifier`.
- **`Tuple`** (`{ a: 1, b: 2 }`) is desugared into a function call: `(tuple (a 1) (b 2))`. The key-value pairs from the `PreSymbolicExpressionType::Tuple` are chunked into pairs and wrapped in a `list` headed by the atom `"tuple"`.
- **`TraitReference`** (`<my-trait>`) is resolved against `contract_ast.referenced_traits` to produce a `SymbolicExpression::trait_reference`.
- **`Comment`** and **`Placeholder`** nodes are dropped (unless `developer-mode` feature is enabled, in which case comments are preserved as metadata on adjacent expressions).

### Pass 8: Expression Identifier (Post-pass)

`ExpressionIdentifier::run_expression_pass` (`clarity/src/vm/ast/expression_identifier/mod.rs:54`) reassigns IDs to the final `SymbolicExpression` tree. This is necessary because the sugar expansion creates new nodes (e.g., the `tuple` function call wrapper) that need unique IDs.

## The Public Entry Points

### `build_ast` (`clarity/src/vm/ast/mod.rs:239`)

The production entry point. Calls `inner_build_ast` with `error_early = true`, meaning it returns the first error encountered. Used during block processing.

### `build_ast_with_diagnostics` (`clarity/src/vm/ast/mod.rs:101`)

The developer tools entry point. Calls `inner_build_ast` with `error_early = false`, collecting all diagnostics. Returns a `(ContractAST, Vec<Diagnostic>, bool)` tuple where the boolean indicates overall success.

### `ast_check_size` (`clarity/src/vm/ast/mod.rs:83`)

A lightweight check that only runs the parser and stack depth checkers -- no sorting, no trait resolution, no sugar expansion. Used as a heuristic to quickly reject obviously oversized contracts before paying full analysis costs.

## How It Connects

- **Upstream**: The parsing pipeline receives raw contract source code from transaction processing (when a `smart-contract` transaction is processed) or from developer tools.
- **Downstream**: The `ContractAST.expressions` field feeds directly into the analysis pipeline (Chapter 6), which performs type checking, read-only checking, and trait compliance verification.
- **Cost tracking**: The `build_ast` function accepts a generic `CostTracker` parameter. In production, this is a `LimitedCostTracker` that charges `ClarityCostFunction::AstParse` based on source code length (Chapter 9).

## Edge Cases and Gotchas

1. **Tuple desugaring creates hidden function calls.** The expression `{ a: 1 }` is not stored as a first-class tuple literal in the final AST. It becomes `(tuple (a 1))`, meaning analysis and execution see a function call to `tuple`. This unification simplifies the interpreter but can be surprising when inspecting the AST.

2. **Definition sorting allows forward references but not cycles.** You can define `(define-constant B A)` before `(define-constant A 1)` and it will work -- the sorter reorders them. But `(define-constant A B)` with `(define-constant B A)` triggers `CircularReference`.

3. **MAX_NESTING_DEPTH discrepancy.** The parser v2 enforces nesting limits during parsing at `AST_CALL_STACK_DEPTH_BUFFER + MAX_CALL_STACK_DEPTH + 1 = 70`. The `StackDepthChecker` and `VaryStackDepthChecker` enforce limits post-parse at `AST_CALL_STACK_DEPTH_BUFFER + MAX_CALL_STACK_DEPTH = 69` (one less). The vary checker may trigger at even shallower effective depths because tuples add extra depth that the parser v2 nesting counter does not count.

4. **Placeholders propagate, not crash.** In non-fail-fast mode, invalid tokens become `Token::Placeholder` and invalid expressions become `PreSymbolicExpressionType::Placeholder`. These are silently dropped by the sugar expander, which means the final AST is always structurally valid even if the parse partially failed. The diagnostics vector tells you what went wrong.

5. **Parser v1 still exists.** Although parser v2 is used for all epochs from 2.1 onward, the v1 parser code remains in the codebase (behind `#[cfg(not(feature = "devtools"))]`) for consensus compatibility with pre-2.1 blocks. Issue #3662 tracks its eventual removal.

6. **Cost charging happens first.** The very first thing `inner_build_ast` does (line 128) is charge `ClarityCostFunction::AstParse` based on `source_code.len()`. If the cost budget is already exhausted, parsing does not even begin. This is a defense against contracts that are expensive to parse being used to DOS the network.
