# Enum and Set support in TiKV Coprocessor

## Motivation

Currently, TiKV and TiDB see an enum as a string. In this RFC,
we want to discuss adding real enum support in TiKV coprocessor.

## Representation of Enum and Set

### Chunk Format of Enum and Set

Enum column stores a finite set of string values. To represent one enum column,
we first need to introduce a chunk format for the enum column.

Inside the enum chunk, we first store all possible values of this column. After
that, we use a usize vector to store actual elements of an enum column. For
example, we have the following column from MySQL reference manual:

```text
size ENUM('x-small', 'small', 'medium', 'large', 'x-large')
col SET('a', 'b', 'c', 'd')
```

First, we store all possible values sequentially in one byte vector. We use an
offset array to indicate the beginning of each element.

```text
Byte vector: x-smallsmallmediumlargex-large
Offset array: 0, 7, 12, 18, 23, 30
```

This also applies to set.

Then, we have a bitmap and usize array to store each element. We take “small”,
“medium”, NULL as an example.

```text
Bitmap: 110 (=6)
Array: 2, 3, 0
```

And for set, we store `BitVec` inside array. We take “('a,d'), ('a'), ('')” as
an example.

```text
Bitmap: 111 (=7)
Array: 11B, 01B, 00B
```

This design leads to an enum chunk vector and a set chunk vector, which
efficiently stores an enum column.

```rust
pub ChunkedVecEnum {
    var_offset: Vec<usize>,
    enum_data: Vec<u8>,
    bitmap: BitVec,
    values: Vec<usize>
}
```

```rust
pub ChunkedVecSet {
    var_offset: Vec<usize>,
    set_data: Vec<u8>,
    bitmap: BitVec,
    values: Vec<BitVec>
}
```

## Support Enum and Set in Vectorized Functions

To add support for enums in vectorized functions, we need to change the `rpn_fn`
macro, and define corresponding types to represent one enum value.

Enum can only appear as parameters of vectorized functions. No function could
return an enum value. This constraint would greatly simplify our design.

An enum value must be binded with an enum chunk vector. Hence, to store enum
values inside the coprocessor framework, we must define the following structures.

To represent only one enum value, we could use `Enum` structure. Note that this
structure should only be used for unit tests. Enums should only be stored and
accesed in the format of enum chunk vectors.

```rust
pub struct Enum {
    var_offset: Vec<usize>,
    enum_data: Vec<u8>,
    value: usize
}
```

```rust
pub struct Set {
    var_offset: Vec<usize>,
    set_data: Vec<u8>,
    value: BitVec
}
```

To represent reference to an enum value, we could use `EnumRef` structure.
Typically, `var_offset` and `enum_values` refers to the same fields in
`ChunkedVecEnum`.

```rust
pub struct EnumRef <'a> {
    var_offset: &'a[usize],
    enum_data: &'a[u8],
    index: usize
}
```

```rust
pub struct SetRef <'a> {
    var_offset: &'a[usize],
    enum_data: &'a[u8],
    value: BitVec
}
```

After that, we could refactor the coprocesser framework to support using
enums during computation.

```rust
#[derive(Debug, PartialEq, Clone)]
pub enum VectorValue {
    Int(ChunkedVecSized<Int>),
    Real(ChunkedVecSized<Real>),
    Decimal(ChunkedVecSized<Decimal>),
    Bytes(ChunkedVecBytes),
    DateTime(ChunkedVecSized<DateTime>),
    Duration(ChunkedVecSized<Duration>),
    Json(ChunkedVecJson),
    Enum(ChunkedVecEnum),
    Set(ChunkedVecSet)
}
```

```rust
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum ScalarValueRef<'a> {
    Int(Option<&'a Int>),
    // ... other fixed-size types ...
    Bytes(Option<BytesRef<'a>>),
    Json(Option<JsonRef<'a>>),
    Enum(Option<EnumRef<'a>>),
    Set(Option<SetRef<'a>>)
}
```

```rust
#[derive(Clone, Debug, PartialEq)]
pub enum ScalarValue {
    Int(Option<super::Int>),
    Real(Option<super::Real>),
    Decimal(Option<super::Decimal>),
    Bytes(Option<super::Bytes>),
    DateTime(Option<super::DateTime>),
    Duration(Option<super::Duration>),
    Json(Option<super::Json>),
    Enum(Option<super::Enum>),
    Set(Option<super::Set>)
}
```

Like `Bytes` and `Json`, an enum vectorized function accepts `EnumRef` as parameter.

```rust
#[rpn_fn]
pub fn cast_enum_to_int(data: EnumRef) -> Result<Option<Int>>;
```

## Add Vectorized Functions for Enum and Set

### Cast Functions

Enum could be casted to `Bytes` and `Int`. In these functions, enums and sets
are only used as inputs. Therefore, in TiKV coprocessor, we will need to
implement `cast_enum_to_bytes` and `cast_enum_to_int`, etc.

### Control Functions

For `IF` and `CASE` functions, the output vector and input vectors should have
the same ranges of values. We will need to modify the coprocessor `rpn_fn` macro
to support this kind of functions.

### Aggregators

For other SQL functions, such as `MAX`, `MIN`, and so on, we could implement them
as aggregators. This can be done by modifying current implemented aggregators.

## Integration with TiDB (future work)

Currently, TiDB treat enum and set as (name, value) pair. To enable full support
for enum functions, TiDB also need to be refactored. This may include:

* Change EvalType and FieldType in tipb
* Add new signatures in tipb
* Cast enum to string and enum to int in SQL plan
* Implement enum and set Chunk vector on TiDB side
* decode new chunk format in LazyColumn

This task should be done on TiDB side. In this RFC, we doesn’t consider this
part. In this RFC, we only ensure that TiKV coprocessor would work correctly
with the future enum/set chunk vector.
