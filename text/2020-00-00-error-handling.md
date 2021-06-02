# Error handling

## Summary

Modernise, unify, and simplify error handling in TiKV. I propose using a
customised or future version of thiserror in all our modules. Migrating TiKV's
error types will make an excellent 'quest issue' for new contributors.

## Motivation

TiKV takes error handling seriously and has a thorough system of errors.
Unfortunately, like software tends to do, this system has become large and
untidy as it has grown. TiKV has more than 30 different error types using two
different error libraries (as well as `std::Error`).

TiKV has some very large error types which are inefficient to pass by value. So
in some places (e.g.,
[src/storage](https://github.com/tikv/tikv/blob/master/src/storage/errors.rs))
we use a pattern of having `Error` and `ErrorInner` types and boxing the inner
error. Unfortunately this leads to very unergonomic matching, e.g., in
[src/storage](https://github.com/tikv/tikv/blob/dec3d008acb56ad0fb34eeeecca62aaa9091b994/src/storage/errors.rs#L174-L180).

Due to having many error types, error types are often nested together to form
chains. However, we rarely use error backtraces (most of our errors don't even
support them) so this just adds code complexity and duplication (different error
types with identical variants). In particular, when handling errors it leads to
deep matching to find all instances of an error.

## Detailed design

Goals and constraints:

* Use statically typed errors (c.f.,
  [anyhow](https://github.com/dtolnay/anyhow), etc.) to eliminate many matching
  errors (this constraint also precludes a single error type).
* Use a single, modern, well-maintained library which is common in the wider
  Rust ecosystem and supports `std::Error`.
* Support boxing errors more ergonomically.
* Reduce deep nesting of errors.
* Reduce error boilerplate.
* Reduce custom error handling code (e.g., our `box_try` macro).
* Support incremental migration to the new error format.

I propose using [thiserror](https://github.com/dtolnay/thiserror). We will
modify thiserror to support *boxed* and *unwrapping* `From` implementations.
Each error type will be migrated to a single enum (no inner enum). Large nested
errors will be individually boxed (note that a boxed enum is the same size as an
enum where every variant is boxed).

### Examples

Current definition (from
[src/storage/txn/mod.rs](https://github.com/tikv/tikv/blob/dec3d008acb56ad0fb34eeeecca62aaa9091b994/src/storage/txn/mod.rs#L42-L94)
):

```rust
quick_error! {
    #[derive(Debug)]
    pub enum ErrorInner {
        Engine(err: crate::storage::kv::Error) {
            from()
            cause(err)
            description(err.description())
        }
        Mvcc(err: crate::storage::mvcc::Error) {
            from()
            cause(err)
            description(err.description())
        }
        ProtoBuf(err: protobuf::error::ProtobufError) {
            from()
            cause(err)
            description(err.description())
        }
        Io(err: IoError) {
            from()
            cause(err)
            description(err.description())
        }
        InvalidTxnTso {start_ts: TimeStamp, commit_ts: TimeStamp} {
            description("Invalid transaction tso")
            display("Invalid transaction tso with start_ts:{},commit_ts:{}",
                        start_ts,
                        commit_ts)
        }
        Other(err: Box<dyn error::Error + Sync + Send>) {
            from()
            cause(err.as_ref())
            description(err.description())
            display("{:?}", err)
        }
    }
}
```

Definition with proposed changes:

```rust
#[derive(Debug, thiserror::Error)]
#[error(unwrap)]
pub enum Error {
    #[error(transparent)]
    Engine(#[from] crate::storage::kv::Error),

    #[error(transparent)]
    Mvcc(#[from] crate::storage::mvcc::Error),

    #[error(transparent)]
    ProtoBuf(#[from] Box<protobuf::error::ProtobufError>),

    #[error(transparent)]
    Io(#[from] IoError),

    #[error("Invalid transaction tso with start_ts: \
             {start_ts}, commit_ts: {commit_ts}")]
    InvalidTxnTso {
        start_ts: TimeStamp,
        commit_ts: TimeStamp,
    },

    #[error(transparent)]
    Other(#[from] AnyError),
}
```

Note that I've elided some variants for concision.

Current pattern (from
[src/storage/errors.rs](https://github.com/tikv/tikv/blob/dec3d008acb56ad0fb34eeeecca62aaa9091b994/src/storage/errors.rs#L174-L180)
):

```rust
    Err(Error(box ErrorInner::Engine(EngineError(box EngineErrorInner::Request(e)))))
    | Err(Error(box ErrorInner::Txn(TxnError(box TxnErrorInner::Engine(EngineError(
        box EngineErrorInner::Request(e),
    ))))))
    | Err(Error(box ErrorInner::Txn(TxnError(box TxnErrorInner::Mvcc(MvccError(
        box MvccErrorInner::Engine(EngineError(box EngineErrorInner::Request(e))),
    )))))) => ...,
```

Pattern with proposed changes:

```rust
    Err(Error::Engine(EngineError::Request(e)))
    | Err(Error::Txn(TxnError::Engine(EngineError::Request(e))))
    | Err(Error::Txn(TxnError::Mvcc(MvccError::Engine(
        EngineError::Request(e),
    )))) => ...,
```

Once the error migration is complete, we are guaranteed that the second and
third clauses are impossible, so the code can be further simplified to:

```rust
    Err(Error::Engine(EngineError::Request(e))) => ...,
    _ => unreachable!(),
```

I have performed the complete migration for the error types in components/backup
and src/storage/txn. The changes can be seen in
[this branch](https://github.com/tikv/tikv/compare/master...nrc:errors-3?expand=1).

### Changes to thiserror

In my opinion, [thiserror](https://github.com/dtolnay/thiserror) is currently
the best library for error handling in high-quality code like TiKV. However, for
TiKV we can do a bit better than the current version by having better support
for boxed errors and multiple error types.

When a field in an enum variant or struct is marked with `#[from]`, thiserror
generates an implementation of `From` to convert from an object of the field's
type to the error type. I propose that this attribute is extended to support an
optional `boxed` modifer: `#[from(boxed)]`. `#[from(boxed)]` is only valid on a
field with type `Box<...>`. The `From` impl generated is from the unboxed type
(i.e., `T` in `Box<T>`) to the error type, rather than the boxed type
(`Box<T>`).

`#[from(boxed)]` means that bare `?` can be used to convert errors without
requiring `.map_err(Box::new)`.

My second proposal is for `#[error(unwrap)]` on error types and
`#[from(unwrap)]` on fields (note that `#[from(boxed, unwrap)]` would be valid).
Where an error marked with `#[error(unwrap)]` is converted to an error type via
a `From` impl generated from `#[from(unwrap)]`, then where possible nested
errors will be unwrapped rather than nested.

For example:

```rust
#[derive(Error)]
struct ErrA;

#[derive(Error)]
#[error(unwrap)]
enum ErrB {
    Var1(#[from] ErrA),
    Var2(String),
}

#[derive(Error)]
enum ErrC {
    Var3(#[from] ErrA),
    Var4(#[from-unwrap] ErrB),
}

fn main() {
    let x: ErrC = ErrB::Var1(ErrA).into();
    let y: ErrC = ErrB::Var2(format!(...)).into();
}
```

`x` will be `ErrC::Var3(ErrA)` (note that the `ErrB` has been unwrapped, and
`ErrA` is stored directly in `ErrC`). `y` will be
`ErrC::Var4(ErrB::Var2("..."))` (note that the `String` is nested inside an
`ErrB`).

A prototype implementation can be seen in [this branch](https://github.com/dtolnay/thiserror/compare/master...nrc:boxed?expand=1).

### Implementation process

I have implemented a prototype of the [changes to
thiserror](https://github.com/dtolnay/thiserror/compare/master...nrc:boxed?expand=1)
and changes to [two error types in
TiKV](https://github.com/tikv/tikv/compare/master...nrc:errors-3?expand=1). If
this RFC is accepted, I will make RFCs to thiserror to see if there is interest
in the changes or if we must maintain our own fork. I will then improve the
prototype implementation to thiserror and land those changes either upstream or
in a fork.

Migrating TiKV (and some of its upstream crates, where appropriate) to the new
error system can be done incrementally, one error type at a time. I propose that
we use this as an opportunity for new contributors by creating a 'quest issue'
with a checklist of error types to migrate, instructions, and links to errors
already migrated to use as examples. Such issues have proved successful ways to
increase contribution and mentor new contributors both in TiKV/TiDB and Rust.

## Drawbacks

Implementing this RFC is a lot of work and brings benefit only in terms of
engineering/technical debt, there is no business-facing benefit (performance,
features, etc.). This should be mitigated by making most of the work into tasks
for new contributors.

This change will cause a fair amount of churn in the TiKV repository. Due to the
way errors are used, changes are spread around the code, not limited to files
with error definitions.

This proposal will probably mean us using a forked version of thiserror (at
least for some time). It is a relatively stable crate, so I don't think it will
be too much effort, and we fork a lot of crates. I would like to try and
upstream our changes once we have enough experience to justify them.

## Alternatives

An obvious alternative is to continue to use one of the error libraries we
already use. Then we should migrate all errors to that library and try to find
patterns to use in all code.

We could also choose to migrate to a more modern error library (of which there
are many in the Rust ecosystem, here is a [good
survey](https://blog.yoshuawuyts.com/error-handling-survey/)). I believe that
thiserror is currently the best choice for TiKV since it satisfies the most
goals and constraints (detailed above).

We might choose to use an unpatched version of thiserror. That would have
benefits in terms of maintenance, but would not address many of our ergonomic
issues.

We could also implement our own library. This would increase the work required
and the ongoing maintenance costs, however, it might lead to a better
programming experience. I have investigated several approaches here, the most
promising of which was [box-error](https://github.com/nrc/box-error). None ended
up working well or better than the solution proposed in this RFC.

An alternative that sounds good in theory is to use a single error type for all
our code. This would certainly improve some ergonomics. However, we would still
have to deal with different errors from upstream libraries. Additionally, we
would lose some benefits from static checking (most functions expect only a
subset of errors).

## Unresolved questions

I have proposed some syntax above. We might be able to do better; I have not
thought through all the possibilities. In particular, the `#[error(unwrap)]`
attribute is not strictly necessary if we don't care about strict backwards
compatibility. We could use a Cargo feature, or just use the unwrapping
behaviour by default. If/when we upstream our changes, I expect we would need
some kind of opt-in.

We should test performance of this solution to ensure we don't cause a
regression. Although I expect this proposal to be as fast or faster then the
current situation, I have not tested that.

As we migrate our error handling as proposed, it might be worth combining or
splitting some error types. I have not investigated if or where this would make
sense.
