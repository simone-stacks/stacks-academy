# Chapter 3: Clarity Type System

## Purpose

Clarity is the smart contract language on Stacks. Unlike Solidity or Move, Clarity has a decidable type system -- every program can be fully type-checked before execution without running it. This chapter covers the core types that represent Clarity values at the Rust level: the `Value` enum, `TypeSignature`, principal types, name types, and the serialization format.

These types live primarily in the `clarity-types` crate (`clarity-types/src/types/`) and are re-exported through the `clarity` crate (`clarity/src/vm/types/`).

## Key Concepts

Clarity's type system is **decidable** -- every program can be fully type-checked without execution. The system is built around the `Value` enum (runtime representation of every Clarity value), `TypeSignature` (compile-time type information), and `PrincipalData` (account identity). All values have a well-defined binary serialization format used for consensus encoding and storage in the MARF.

## Architecture

The type system is split across two crates for dependency reasons:

- **clarity-types** (`clarity-types/`): Core value types, type signatures, serialization. Depends only on `stacks-common`.
- **clarity** (`clarity/`): The full VM -- parser, analyzer, interpreter. Re-exports everything from `clarity-types` and adds VM-specific extensions.

This split allows crates that only need to work with Clarity values (like `libsigner`) to depend on `clarity-types` without pulling in the entire VM.

## The Value Enum

Every Clarity value in the runtime is represented by the `Value` enum, defined in `clarity-types/src/types/mod.rs:314`:

```rust
pub enum Value {
    Int(i128),
    UInt(u128),
    Bool(bool),
    Sequence(SequenceData),
    Principal(PrincipalData),
    Tuple(TupleData),
    Optional(OptionalData),
    Response(ResponseData),
    CallableContract(CallableData),
}
```

### Numeric Types

- **Int(i128)**: A signed 128-bit integer. Clarity's `int` type. Range: -2^127 to 2^127-1.
- **UInt(u128)**: An unsigned 128-bit integer. Clarity's `uint` type. Range: 0 to 2^128-1.

Using 128-bit integers is a deliberate design choice: STX amounts in microSTX can exceed 64-bit range when multiplied or accumulated.

### Bool

`Bool(bool)` maps directly to Clarity's `bool` type (`true` / `false`).

### Sequence Types

`Sequence(SequenceData)` wraps all ordered collection types:

```rust
// clarity-types/src/types/mod.rs:329-334
pub enum SequenceData {
    Buffer(BuffData),
    List(ListData),
    String(CharType),
}
```

Where `CharType` (`mod.rs:665`) distinguishes string encoding:

```rust
pub enum CharType {
    UTF8(UTF8Data),
    ASCII(ASCIIData),
}
```

The individual data types:

```rust
// clarity-types/src/types/mod.rs:68-78
pub struct BuffData {
    pub data: Vec<u8>,
}

pub struct ListData {
    pub data: Vec<Value>,
    pub type_signature: ListTypeData,  // tracks the element type and max length
}
```

- **BuffData**: Raw bytes. Clarity `(buff N)` type. Maximum size: 1 MB (`MAX_VALUE_SIZE`).
- **ListData**: A homogeneous list of Values. Carries its `ListTypeData` for runtime type checking.
- **ASCIIData / UTF8Data**: String types. ASCII strings are byte-per-character; UTF8 strings store `Vec<Vec<u8>>` (one inner vec per Unicode scalar value).

### Principal Types

Principals are the identity type in Clarity -- they represent accounts and contracts:

```rust
// clarity-types/src/types/mod.rs:213-217
pub enum PrincipalData {
    Standard(StandardPrincipalData),
    Contract(QualifiedContractIdentifier),
}
```

#### StandardPrincipalData

```rust
// clarity-types/src/types/mod.rs:80-81
pub struct StandardPrincipalData(u8, pub [u8; 20]);
```

A standard principal is a version byte (0-31) and a 20-byte Hash160. This is identical to a `StacksAddress` in structure. The version byte encodes the network and signature scheme:

- Version 22 (`P`): Mainnet single-sig
- Version 26 (`T`): Testnet single-sig
- Version 20 (`M`): Mainnet multi-sig
- Version 21 (`N`): Testnet multi-sig

The constructor enforces the version constraint:

```rust
// clarity-types/src/types/mod.rs:93-98
pub fn new(version: u8, bytes: [u8; 20]) -> Result<Self, ClarityTypeError> {
    if version >= 32 {
        return Err(ClarityTypeError::InvalidPrincipalVersion(version));
    }
    Ok(Self(version, bytes))
}
```

Display uses C32Check encoding: `StandardPrincipalData(22, [0xa4, 0x6f, ...])` displays as `SP2J6ZY48GV1EZ5V2V5RB9MP66SW86PYKKNRV9EJ7`.

#### QualifiedContractIdentifier

```rust
// clarity-types/src/types/mod.rs:167-170
pub struct QualifiedContractIdentifier {
    pub issuer: StandardPrincipalData,
    pub name: ContractName,
}
```

A contract principal is a deployer address plus a contract name. It displays as `SP2J6ZY48GV1EZ5V2V5RB9MP66SW86PYKKNRV9EJ7.my-contract`.

The `parse` method splits on the first `.`:

```rust
// clarity-types/src/types/mod.rs:196-204
pub fn parse(literal: &str) -> Result<QualifiedContractIdentifier, ClarityTypeError> {
    let split: Vec<_> = literal.splitn(2, '.').collect();
    if split.len() != 2 {
        return Err(ClarityTypeError::QualifiedContractMissingDot);
    }
    let sender = PrincipalData::parse_standard_principal(split[0])?;
    let name = split[1].to_string().try_into()?;
    Ok(QualifiedContractIdentifier::new(sender, name))
}
```

Boot contracts (deployed by the null address `S00000000000000000000000`) return `true` for `is_boot()`.

### Compound Types

#### OptionalData

```rust
// clarity-types/src/types/mod.rs:231-234
pub struct OptionalData {
    pub data: Option<Box<Value>>,
}
```

Represents Clarity's `(optional T)`. `None` is `OptionalData { data: None }` and `(some v)` is `OptionalData { data: Some(Box::new(v)) }`.

The global constant `NONE` is pre-defined for efficiency.

#### ResponseData

```rust
// clarity-types/src/types/mod.rs:236-240
pub struct ResponseData {
    pub committed: bool,
    pub data: Box<Value>,
}
```

Represents Clarity's `(response T E)`. When `committed` is `true`, the value is `(ok data)`. When `false`, it is `(err data)`. This is Clarity's primary error handling mechanism -- public function return values must be Response types, and `committed: false` triggers transaction rollback.

#### TupleData

```rust
// clarity-types/src/types/mod.rs:62-66
pub struct TupleData {
    pub type_signature: TupleTypeSignature,
    pub data_map: BTreeMap<ClarityName, Value>,
}
```

Tuples are ordered maps from `ClarityName` keys to `Value` fields. Using `BTreeMap` ensures deterministic iteration order (alphabetical by key), which is critical for consensus-deterministic serialization.

#### CallableData

```rust
// clarity-types/src/types/mod.rs:242-246
pub struct CallableData {
    pub contract_identifier: QualifiedContractIdentifier,
    pub trait_identifier: Option<TraitIdentifier>,
}
```

Represents a reference to a callable contract, optionally constrained to a specific trait. Used when passing contracts as arguments to functions that expect trait-typed parameters.

## TypeSignature

The `TypeSignature` enum describes the type of a Clarity value without carrying actual data. It is the compile-time counterpart to the runtime `Value`:

```rust
// clarity-types/src/types/signatures.rs:272-297
pub enum TypeSignature {
    NoType,
    IntType,
    UIntType,
    BoolType,
    SequenceType(SequenceSubtype),
    PrincipalType,
    TupleType(TupleTypeSignature),
    OptionalType(Box<TypeSignature>),
    ResponseType(Box<(TypeSignature, TypeSignature)>),
    CallableType(CallableSubtype),
    ListUnionType(HashSet<CallableSubtype>),
    TraitReferenceType(TraitIdentifier),
}
```

Where `SequenceSubtype` further breaks down:

```rust
// clarity-types/src/types/signatures.rs:299-304
pub enum SequenceSubtype {
    BufferType(BufferLength),
    ListType(ListTypeData),
    StringType(StringSubtype),
}
```

Key design decisions:

- **ResponseType** carries *both* the ok-type and err-type: `Box<(TypeSignature, TypeSignature)>`.
- **ListUnionType** tracks a set of callable subtypes for deferred type resolution (when a list contains multiple contract literals that might need to be coerced to a trait type).
- **NoType** is a sentinel for variables that haven't been assigned a type yet during analysis.
- **BufferLength** wraps a `u32` constrained to `<= MAX_VALUE_SIZE` (1 MB).

### TupleTypeSignature

```rust
// clarity-types/src/types/signatures.rs:68-72
pub struct TupleTypeSignature {
    type_map: Arc<BTreeMap<ClarityName, TypeSignature>>,
}
```

Uses `Arc` for cheap cloning -- tuple types are shared across many analysis contexts.

### Size Constraints

```rust
// clarity-types/src/types/mod.rs:43-59
pub const MAX_VALUE_SIZE: u32 = 1024 * 1024;           // 1 MB
pub const BOUND_VALUE_SERIALIZATION_BYTES: u32 = MAX_VALUE_SIZE * 2;
pub const MAX_TYPE_DEPTH: u8 = 32;                      // Maximum nesting depth
pub const WRAPPER_VALUE_SIZE: u32 = 1;                   // Size of optional/response wrapper
```

Every `TypeSignature` can compute its `size()` in bytes. The type checker verifies that no value can exceed `MAX_VALUE_SIZE` and that nesting doesn't exceed `MAX_TYPE_DEPTH`. This bounds-checking is what makes Clarity decidable.

## Name Types

### ClarityName

A validated string for Clarity identifiers (function names, variable names, map keys):

```rust
// clarity-types/src/representations.rs:64-70
guarded_string!(
    ClarityName,
    CLARITY_NAME_REGEX,    // ^[a-zA-Z]([a-zA-Z0-9]|[-_!?+<>=/*])*$|^[-+=/*]$|^[<>]=?$
    MAX_STRING_LEN,        // 128 bytes
    ClarityTypeError,
    ClarityTypeError::InvalidClarityName
);
```

The `guarded_string!` macro creates a newtype around `String` that validates against the regex on construction. Valid examples: `transfer`, `is-valid?`, `+`, `>=`. Max length: 128 characters.

### ContractName

```rust
// clarity-types/src/representations.rs:72-78
guarded_string!(
    ContractName,
    CONTRACT_NAME_REGEX,   // ^[a-zA-Z]([a-zA-Z0-9]|[-_]){0,127}$|^__transient$
    MAX_STRING_LEN,
    ClarityTypeError,
    ClarityTypeError::InvalidContractName
);
```

Contract names are more restrictive than ClarityNames -- they only allow alphanumeric characters, hyphens, and underscores. The `__transient` special name is used for ephemeral contract contexts during analysis.

Both `ClarityName` and `ContractName` implement `StacksMessageCodec` for consensus serialization (length-prefixed with a `u8`).

## Trait Identifiers

```rust
// clarity-types/src/types/mod.rs:248-252
pub struct TraitIdentifier {
    pub name: ClarityName,
    pub contract_identifier: QualifiedContractIdentifier,
}
```

A trait is identified by the contract that defines it and the trait's name within that contract. For example, `.my-contract.sip-010-trait` is the `sip-010-trait` trait defined in `my-contract`.

## AssetIdentifier

```rust
// clarity-types/src/types/signatures.rs:33-36
pub struct AssetIdentifier {
    pub contract_identifier: QualifiedContractIdentifier,
    pub asset_name: ClarityName,
}
```

Identifies a specific token type (fungible or non-fungible) defined in a contract. The special `AssetIdentifier::STX()` represents native STX tokens (with a null principal issuer).

## Serialization Format

Clarity values are serialized for storage in the MARF and for transmission on the wire. The format uses a **type prefix byte** followed by type-specific data:

```rust
// clarity-types/src/types/serialization.rs:131-147
define_u8_enum!(TypePrefix {
    Int = 0,
    UInt = 1,
    Buffer = 2,
    BoolTrue = 3,
    BoolFalse = 4,
    PrincipalStandard = 5,
    PrincipalContract = 6,
    ResponseOk = 7,
    ResponseErr = 8,
    OptionalNone = 9,
    OptionalSome = 10,
    List = 11,
    Tuple = 12,
    StringASCII = 13,
    StringUTF8 = 14
});
```

Serialization layout for each variant:

| Type | Prefix | Payload |
|---|---|---|
| Int | 0x00 | 16 bytes, big-endian i128 |
| UInt | 0x01 | 16 bytes, big-endian u128 |
| Buffer | 0x02 | 4-byte length (u32) + raw bytes |
| Bool (true) | 0x03 | (none) |
| Bool (false) | 0x04 | (none) |
| Standard Principal | 0x05 | 1 version byte + 20 hash bytes |
| Contract Principal | 0x06 | standard principal + 1-byte name length + name bytes |
| Response Ok | 0x07 | serialized inner Value |
| Response Err | 0x08 | serialized inner Value |
| Optional None | 0x09 | (none) |
| Optional Some | 0x0a | serialized inner Value |
| List | 0x0b | 4-byte length (u32) + serialized elements |
| Tuple | 0x0c | 4-byte field count + (clarity_name + value) pairs |
| String ASCII | 0x0d | 4-byte length + ASCII bytes |
| String UTF8 | 0x0e | 4-byte byte-length + UTF8 bytes |

The deserialization code enforces `BOUND_VALUE_SERIALIZATION_BYTES` (2 MB) as a read limit and `MAX_TYPE_DEPTH` (32) as a nesting limit. The epoch `DESERIALIZATION_TYPE_CHECK_EPOCH` is pinned to `Epoch21` so that values stored before epoch 2.4 can still be read.

## Value Construction Helpers

The `Value` type provides many factory methods (in `clarity-types/src/types/mod.rs`):

```rust
Value::Int(42)
Value::UInt(1_000_000)
Value::Bool(true)
Value::none()                                  // OptionalData { data: None }
Value::some(inner)                             // wraps in OptionalData
Value::okay(inner)                             // ResponseData { committed: true, data }
Value::error(inner)                            // ResponseData { committed: false, data }
Value::buff_from(bytes)                        // BuffData from Vec<u8>
Value::string_ascii_from_bytes(bytes)          // ASCIIData from bytes
Value::list_from(values)                       // Validates homogeneous types
Value::Principal(PrincipalData::Standard(..))
```

The `list_from` constructor is notable because it infers the list element type from the first element and validates that all subsequent elements match.

## How It Connects

- **Chapter 2** (Common Types) provides the underlying `Hash160`, `StacksAddress`, and `StacksMessageCodec` used by principal types.
- **Chapter 5** (Clarity Parsing) shows how source text becomes `SymbolicExpression` nodes that carry `Value` literals.
- **Chapter 6** (Clarity Analysis) uses `TypeSignature` for static type checking.
- **Chapter 7** (Clarity Execution) evaluates expressions and produces `Value` results.
- **Chapter 8** (Clarity Database) stores and retrieves `Value` instances using the serialization format described above.
- **Chapter 14** (Transactions) wraps Clarity function arguments and return values as `Value`.

## Edge Cases and Gotchas

1. **128-bit integer serialization**: Clarity uses i128/u128 natively, but consensus serialization is big-endian. Ensure byte order is correct when manually constructing values.

2. **MAX_VALUE_SIZE is 1 MB**: No single Clarity value can exceed 1 MB when serialized. This includes nested structures -- a list of tuples of buffers must fit within 1 MB total. The type checker validates this statically.

3. **BTreeMap ordering in tuples**: Tuple fields are stored in a `BTreeMap<ClarityName, Value>`, which sorts keys lexicographically. This means serialization order is deterministic but may differ from the order fields were specified in source code.

4. **Response vs Optional**: Both wrap an inner value, but they serve different purposes. Responses are used for public function returns and control transaction commit/rollback. Optionals are used for nullable values. They are not interchangeable.

5. **Value sanitization**: Starting in Epoch 2.4, values loaded from storage are "sanitized" -- their type signatures are checked against expected types and adjusted if necessary. This was introduced to handle values that were stored before stricter validation was added. The `value_sanitizing()` method on `StacksEpochId` gates this behavior.

6. **Type depth limit**: The `MAX_TYPE_DEPTH` of 32 prevents stack overflow during type checking and serialization of deeply nested types. Pre-epoch-2.4, the deserialization depth limit was only 16 (`UNSANITIZED_DEPTH_CHECK`).

7. **The CallableContract variant**: `Value::CallableContract` is used internally during VM execution to represent contract references in trait-typed contexts. It should not appear in stored data or on the wire.
