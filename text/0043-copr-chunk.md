# Using chunk format in coprocessor framework

* RFC PR: https://github.com/tikv/rfcs/pull/43
* Tracking Issue: https://github.com/tikv/tikv/issues/7724

## Summary

This RFC describes using [Chunk format][reading-chunk] during operator evaluation
in the coprocessor framework. The Chunk format is a column-based data format,
which has much more compact memory layout than the Rust vector `Vec<Option<T>>`.

## Motivation

There are multiple benifits to use chunk format during operator evaluation:

1. Many SQL builtin functions handle NULL data in the same way. If any of the
   input arguments is NULL, the result is NULL. In TiDB Chunk format, a bitmap
   vector is used to store if a cell is null or not. By merging bitmap vectors
   before loop, we can achieve better cache locality.
2. In Chunk format, variable-sized type (e.g. `Bytes`, `Json`) are stored
   adjacently in memory. We can reduce overhead of memory allocation.
3. SIMD instructions require continuous data in memory.
4. If we use the same memory format as TiDB, we can reduce encoding and decoding
   overhead when passing data to TiDB.

## Detailed Design

### A unified interface for references

In coprocessor framework, different data types have different types for their
reference. The relationship can be seen as follows.

| Data Type | Type of its Reference |
|-|-|
| `Int`, `Decimal`, ... | `&Int`, `&Decimal`, ... |
| `Json` | `JsonRef` |
| `Bytes` | `BytesRef` (or `&[u8]`) |

`BytesRef` is a newly-introduced type. It is defined as
`type BytesRef<'a> = &'a [u8]`.

To describe the relationship between these types in Rust, we propose using three
traits.

* `Evaluable`: Implemented only for fixed-size types.
* `EvaluableRef`: Implemented on reference types.
* `EvaluableRet`: Implemented for all owned types.
* If `T` is Evaluable, `&T` is EvaluableRef

| Trait            | Implemented by                                             |
|----------------|------------------------------------------------------------|
| `Evaluable`    | `Int`, `Decimal`, `Duration`, `DateTime`, `Real`           |
| `EvaluableRef` | `&Int`, ..., `&Real`, `BytesRef`, `JsonRef` |
| `EvaluableRet` | `Int`, ..., `Real`, `Bytes`, `Json` |

Separating traits for owned type and reference types leads to a unified
interface for both types, while not breaking support for generics too much. As
we don't plan to add lifetime support for the coprocessor framework, now
developers may only use generic vectorized functions for fixed-size types.

### ChunkedVec: Chunk format in TiKV

We propose using Chunk format vector `ChunkedVec`, a type-safe vector using
chunk format, during operator evaluation. Currently, [Chunk format][tikv-chunk]
has already been implemented in TiKV, which is a simple rewrite of
[TiDB chunk format][tidb-chunk] without type safety.

We have 3 separate implementations of Chunk format vectors for different data
types. `ChunkedVecSized<T>` stores elements of fixed-size types in Chunk format.
`ChunkedVecBytes` and `ChunkedVecJson` store `Bytes` and `Json` respectively.
Inside each `ChunkedVec`, we have a `BitVec`, which acts as a bitmap and
compactly stores boolean values in `u8`.

All Chunk format vectors implement two traits. `ChunkedVec` trait is used when
constructing a chunk vector. `ChunkRef` trait is built for read-only access to
chunk vectors.

```rust
pub trait ChunkedVec<T> {
    fn chunked_with_capacity(capacity: usize) -> Self;
    fn chunked_push(&mut self, value: Option<T>);
}
```

`ChunkedVec` provides a simple interface for constructing chunk vectors. This
trait is mainly used in `rpn_fn` macro. If a struct implements `ChunkedVec<T>`,
we can push objects of type `T` to this struct.

```rust
pub trait ChunkRef<'a, T: EvaluableRef<'a>>: Copy + Clone + std::fmt::Debug + Send + Sync {
    fn get_option_ref(self, idx: usize) -> Option<T>;

    fn phantom_data(self) -> Option<T>;
}
```

`ChunkRef` provides an interface for getting reference to elements from chunk
vectors. This trait is used all over coprocessor framework. If a chunk vector
implements `ChunkRef<&T>`, we can get `Option<&T>` from this vector, using
`get_option_ref`. `phantom_data` is designated to get an null element of this
chunk vector. It should always return `None`.

The relationship between chunked vector structs and traits is shown in the
following table.

| Struct | ChunkedVec trait | ChunkRef trait |
|-|-|-|
| `ChunkedVecSized<T>` | `ChunkedVec<T>` | `ChunkRef<&T>` |
| `ChunkedVecBytes` | `ChunkedVec<Bytes>` | `ChunkRef<BytesRef>` |
| `ChunkedVecJson` | `ChunkedVec<Json>` | `ChunkRef<JsonRef>` |

This design allows the coprocessor framework to use chunk format during operator
evaluation. It provides a unified interface for constructing and accessing chunk
vectors.

### Rpn Expression

Bringing chunk vectors to coprocessor framework requires refactors in some RPN
expression structs.

`VectorValue` should now store the new chunk vectors. Currently, it is defined
as follows.

```rust
#[derive(Debug, PartialEq, Clone)]
pub enum VectorValue {
    Int(Vec<Option<Int>>),
    Real(Vec<Option<Real>>),
    Decimal(Vec<Option<Decimal>>),
    // TODO: We need to improve its performance, i.e. store strings in adjacent memory places
    Bytes(Vec<Option<Bytes>>),
    DateTime(Vec<Option<DateTime>>),
    Duration(Vec<Option<Duration>>),
    Json(Vec<Option<Json>>),
}
```

We will modify the `VectorValue` to:

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
}
```

In `ScalarRef`, we will lift lifetime into the `Option` struct. In this way,
references from chunk vectors can be placed inside `ScalarValueRef`.

```rust
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum ScalarValueRef<'a> {
    Int(&'a Option<Int>),
    // other fixed-size types
    Bytes(&'a Option<Bytes>),
    Json(&'a Option<Json>),
}
```

It will become:

```rust
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum ScalarValueRef<'a> {
    Int(Option<&'a Int>),
    // other fixed-size types
    Bytes(Option<BytesRef<'a>>),
    Json(Option<JsonRef<'a>>),
}
```

Also, we have to modify all rpn_function signatures.

```rust
// From
#[rpn_fn]
pub fn logical_and(lhs: &Option<Int>, rhs: &Option<Int>) -> Result<Option<Int>>;

// To
#[rpn_fn(nullable)]
pub fn logical_and(lhs: Option<&Int>, rhs: Option<&Int>) -> Result<Option<Int>>;

// From
#[rpn_fn]
pub fn length(arg: &Option<Bytes>) -> Result<Option<Int>> {
    Ok(arg.map(|bytes| bytes.len() as Int))
}

// To
#[rpn_fn(nullable)]
pub fn length(arg: Option<BytesRef>) -> Result<Option<Int>> {
    Ok(arg.map(|bytes| bytes.len() as Int))
}
```

### New features

Many SQL builtin functions handle NULL data in the same way. They return NULL if
any of the arguments is NULL. In this way, all rpn_functions are now by default
non-nullable. Developers should add `nullable` attribute if they want to handle
null values by themselves.

```rust
// A nullable example
#[rpn_fn(nullable)]
pub fn logical_and(lhs: Option<&Int>, rhs: Option<&Int>) -> Result<Option<Int>>;

// A non-nullable example
#[rpn_fn]
pub fn logical_and(lhs: &Int, rhs: &Int) -> Result<Option<Int>>;
```

Now bitmap vectors inside chunk vectors store whether a cell is null or not. For
those non-nullable functions, bitmaps are merged before calling vectorized
functions by default. This is done by introducing a `BitAndIterator`. It can be
used like this:

```rust
for (idx, i) in BitAndIterator::new(&[&vec1, &vec2, &vec3, &vec5], output_rows).enumerate() {
    // ...
}
```

For `ChunkedVecBytes` and `ChunkedVecJson`, as now we store `Bytes` and `Json`
data adjacently in memory, extra overhead is introduced when writing return
value from vectorized functions to chunk vectors. In this way, we introduce some
helper structs. Currently, this only applies on `Bytes`.

```rust
pub struct BytesGuard {
    chunked_vec: ChunkedVecBytes,
}

pub struct BytesWriter {
    chunked_vec: ChunkedVecBytes,
}

impl BytesWriter {
    pub fn begin(self) -> PartialBytesWriter;

    pub fn write(mut self, data: Option<Bytes>) -> BytesGuard;

    pub fn write_ref(mut self, data: Option<BytesRef>) -> BytesGuard;
}

pub struct PartialBytesWriter {
    chunked_vec: ChunkedVecBytes,
}

impl<'a> PartialBytesWriter {
    pub fn partial_write(&mut self, data: BytesRef);

    pub fn finish(mut self) -> BytesGuard;
}
```

The definition of `Writer` and `PartialWriter` ensures some properties:

1. `Guard` is zero-overhead, and the only way to obtain a `Guard` is calling
   `write` or `finish`.
2. `write` can be called at most once on one `Writer`.
3. Only one of `write` and `begin` can be called at most once on one `PartialWriter`.

And developers should specify writer attribute in rpn functions.

```rust
// using partial writer
#[rpn_fn(writer)]
pub fn repeat(input: BytesRef, cnt: &Int, writer: BytesWriter) -> Result<BytesGuard> {
    let mut writer = writer.begin();
    for _i in 0..*cnt {
        writer.partial_write(input);
    }
    Ok(writer.finish())
}

// using direct write
#[rpn_fn(writer)]
fn json_type(arg: JsonRef, writer: BytesWriter) -> Result<BytesGuard> {
    Ok(writer.write(Some(Bytes::from(arg.json_type()))))
}
```

[reading-chunk]: https://pingcap.com/blog-cn/tidb-source-code-reading-10/
[tikv-chunk]: https://github.com/tikv/tikv/blob/cc424374af5c50246d4f1e0cb282f63540beb1b9/components/tidb_query/src/codec/chunk
[tidb-chunk]: https://github.com/pingcap/tidb/blob/9627132db25e5cb560ba6ed89c2e39c32edb267a/util/chunk/chunk.go
