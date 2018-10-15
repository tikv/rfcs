# Summary

Reduce one data copy in the process of sending messages in grpcio.
This is not a new feature but an optimization.

# Motivation

In pingcap/grpc-rs (which is named [`grpcio`](https://docs.rs/crate/grpcio/) on crates.io), we can call gRPC's core API to send data.
In our old implementation, the whole process is:

+ User code produces the data to be sent (particularly, the protobuf library's deserialization) -- a list of `&[u8]` in a call-back
+ The data get copied into a `Vec<u8>`. For the protobuf library, this is done internally (**copy 0**)
  ```rust
  pub fn bin_ser(t: &Vec<u8>, buf: &mut Vec<u8>) {
    buf.extend_from_slice(t)
  }
  ```
+ We create a `grpc_slice` (a ref-counted single string) by copying the `Vec<u8>` mentioned above (**copy 1**)
+ We create a `grpc_byte_buffer` (a ref-counted list of strings) from only one `grpc_slice`
+ Send data

In total, we copy the data twice. The first copy is unnecesasry.

# Detailed design

## Basic concepts

First, due to Rust and C have different naming conventions:

+ `grpc_slice` on C side is renamed to `GrpcSlice` on Rust side
+ `grpc_byte_buffer` on C side is renamed to `GrpcByteBuffer` on Rust side

So don't be confused by the naming difference.

## Changes

After this refactoring,

+ User code produces the data to be sent (particularly, the protobuf library's deserialization) -- a list of `&[u8]`
+ We copy each produced `&[u8]` and collect them into a `Vec<GrpcSlice>` (**copy 0**)
    ```rust
    fn write(&mut self, buf: &[u8]) -> io::Result<usize> {
        let in_len: size_t = buf.len();
        self.size += in_len;
        unsafe {
            self.data.push(grpc_sys::grpc_slice_from_copied_buffer(
                buf.as_ptr() as _,
                in_len,
            ));
        }
        Ok(in_len)
    }
    ```
+ We create a `grpc_byte_buffer` from the collected `Vec<GrpcSlice>`
    ```rust
    pub unsafe fn as_ptr(&self) -> *mut GrpcByteBuffer {
        grpc_sys::grpc_raw_byte_buffer_create(self.data.as_ptr(), self.data.len())
    }
    ```
+ Send data

# Drawbacks

This is purely an optimization, no drawbacks.
The reason why still one copy:

+ Rust manages its own allocated memory's lifetime, and we're not able to disable this
+ C-side uses a ref-count mechanism to manage the lifetime of objects
+ We either need to extend the Rust objects' lifetime or do a copy (see alternatives section)

# Alternatives

## Reduce re-allocations

Since we're copying data into a `Vec<u8>`, there must be a lot of re-allocations when we continuously add data into it.
I see protobuf can compute the deserialized data size, so I added a `reserve` before copying data.

### Result

This does not work as good as the accepted solution and we cannot assume that other users can compute the data size too, so it's given up.

## Dirty hack

This idea prevents the second copy which originally happens on C-side.
Since we cannot disbale Rust's deallocation, we can avoid using Rust objects at the very beginning.

I created a C++ vector. We add data to it, and take the raw data pointer away.
Now we have a pointer that points to a piece of memory that we only have to worry about when to deallocate it.
This can be easily done with the ref-count mechanism.

### Result

Superseded by the use of `GrpcByteBuffer`.

## Zero copy

Actuallty I've tried to extend Rust objects' lifetime.

The basic idea is, extend the data's lifetime by moving it into our context object which is destroyed automatically when one send is complete.
Then we mark this data (which is stored in a `grpc_slice`) as static on C-side, so the ref-count system won't do the deallocation.

### Result

A feature provided by gRPC called buffer-hint which requires the data to have a much longer lifetime has prevented me from accepting this idea.
It's too hard to decide how to manage the lifetime in such case (since the time that the data is no longer needed on C-side is too hard to estimate, we'll have to do lots of thread-sync and `if`s).

# Unresolved questions

- There's still one copy.
