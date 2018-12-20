# Reduce One Copy in gRPC Send

## Summary

Reduce one data copy in the process of sending messages in grpcio.
This is not a new feature but an optimization.

## Motivation

In pingcap/grpc-rs (which is named [`grpcio`](https://docs.rs/crate/grpcio/)
on crates.io), we can call gRPC's core API to send data.
In our old implementation, the whole process is:

1. User code produces the data to be sent (particularly, the protobuf library's
   deserialization) -- a list of `&[u8]` in a call-back
2. The data get copied into a `Vec<u8>`. For the protobuf library, this is done
   internally (**copy 0**)

   ```rust
   pub fn bin_ser(t: &Vec<u8>, buf: &mut Vec<u8>) {
     buf.extend_from_slice(t)
   }
   ```

3. We create a `grpc_slice` (a ref-counted single string) by copying the
   `Vec<u8>` mentioned above (**copy 1**)
4. We create a `grpc_byte_buffer` (a ref-counted list of strings) from only
   one `grpc_slice`
5. Send data

In total, we copy the data twice. The first copy is unnecessary.

## Detailed design

### Basic concepts

First, due to Rust and C have different naming conventions:

+ `grpc_slice` on C side is renamed to `GrpcSlice` on Rust side
+ `grpc_byte_buffer` on C side is renamed to `GrpcByteBuffer` on Rust side

So don't be confused by the naming difference.

### Changes

After this refactoring,

1. User code produces the data to be sent (particularly, the protobuf library's
   deserialization) -- a list of `&[u8]`
2. We copy each produced `&[u8]` into `GrpcSlice`s collect them into a
   `GrpcByteBuffer` wrapper (**copy 0**)
    ```rust
    pub fn push(&mut self, slice: GrpcSlice) {
        unsafe {
            grpc_sys::grpcwrap_byte_buffer_add(self.raw as _, slice);
        }
    }
    ```
3. We take the raw C struct pointer in `GrpcByteBuffer` away and pass it
   directly into the send functions
    ```rust
    pub unsafe fn take_raw(&mut self) -> *mut grpc_sys::GrpcByteBuffer {
        let ret = self.raw;
        self.raw = grpc_sys::grpc_raw_byte_buffer_create(ptr::null_mut(), 0);
        ret
    }
    ```

## Drawbacks

This is purely an optimization, no drawbacks.
The reason why still one copy:

+ Rust manages its own allocated memory's lifetime, and we're not able
  to disable this
+ C-side uses a ref-count mechanism to manage the lifetime of objects
+ We either need to extend the Rust objects' lifetime or do a copy (see
  alternatives section)

## Unresolved questions

Actuallty it's possible to extend Rust objects' lifetime.
This can achieve real zero-copy.

### Zero copy

The basic idea is, extend the data's lifetime by moving it into our context
object which is destroyed automatically when one send is complete.
Then we mark this data (which is stored in a `grpc_slice`) as static on
C-side, so the ref-count system won't do the deallocation.

### Result

A feature provided by gRPC called buffer-hint which requires the data to have
a much longer lifetime has prevented me from accepting this idea.
It's too hard to decide how to manage the lifetime in such case (since the time
that the data is no longer needed on C-side is too hard to estimate, we'll have
to do lots of thread-sync and `if`s).
