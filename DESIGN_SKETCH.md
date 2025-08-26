# Design sketch: ring buffers

Idea: We want something similar to Linux' `io-uring`, but more general, for arbitrary inter-process-communication
(especially given that the Redox kernel is a microkernel).

## minimal API

When thinking of the `io-uring` API, it does consist of two ring buffers.
The most minimal API imaginable shoouldn't impose on programs to use such an async remote-procedure-call-like interface,
and instead just represent that case as two ring buffers (to be interacted with using the API below).
It would then be easy to later wrap that interface in one which does bundle two ring buffers
such that the end result looks kinda like `io-uring` with its submission and completion queues.

As a concept, it should be noted that the ring buffers described here are semantically an by-default asynchronouos,
pipe-like interface (but that has packets ala `SOCK_SEQPACKET`), that ends up representing a single-producer-single-consumer message queue.

API's which might benefit from syscalls or kernel integration (e.g. flag bits on memory-mapped areas and pipes):
* request a new ring buffer (represented via a file descriptor).
* signal to the other end that data is available (in order to wake up the other end)
* wait for data to be available (waiting for the signal noted above)
* request a new buffer / memory-mapped area to be associated to the ring buffer (used to transfer data in bulk)
  * The kernel has to keep track of these associations. These buffers have to be made available to both ends.

Note that the last two items in Linux are both done by `io_uring_enter`, but can also be done using `io_uring_submit` and `io_uring_wait_cqe`.

Requesting a new ring buffer means requesting at least 2 memory-mapped areas, in particular,
* an area for the ring buffer header, mapped `read+write`
* an area for the ring buffer contents, mapped as `read`-only on one end and as `read+write` (or perhaps just `write`) on the other end.

Pure library APIs:
* push an item into the queue (might fail if the buffer/queue is full (temporary), or if the entry doesn't fit into the queue (permanently))
  * the "wait" API from above could be used for the other direction (signaling if the queue has more space in it again), too.
* pop an item from the queue (fails if the queue is empty)

Necessary metadata about queues ("queue/ ring buffer header"):
```rust
struct Header {
    ptr_queue: *mut u8,
    ptr_head: u32,
    ptr_tail: u32,
    queue_len: u32,
    item_len: Option<core::num::NonZeroU32>,
}
```
(the in-memory representation will probably be different)

As it is not immediately clear that all intended usages will have fixed-length items, it makes sense to provide a way
to denote items of dynamic length by `item_len == None`, where each entry is simply prefixed by its length as `u32`
(leading to a usage model that is basically a `SOCK_SEQPACKET` `pipe(2)`, just also async).

`ptr_queue`, `ptr_mask`, `queue_len`, `item_len` are constant over the lifetime of the ring buffer
(they are influenced by the parameters passed to the ring buffer allocation function).
It would thus make sense to actually have two structures for all the metadata:

```rust
#[repr(C, packed)]
struct FixedParameters {
    ptr_queue: *mut u8,
    queue_len: u32,
    item_len: Option<core::num::NonZeroU32>,
}

#[repr(C, packed)]
struct RingHeader {
    ptr_head: u32,
    ptr_tail: u32,
}
```

Open questions:
* do we want to allow ring buffers which are larger than 4GiB?
* how should buffers be handled (in particular, can we ensure that the buffer addresses will be the same at both ends?)

## high-level API

As most use cases will just use the API above for async remote-procedure-calls (RPCs),
we then need a library API which has a concept of submission queue entries (sqes) and completion queue entries (cqes).

Open questions:
* how much effort would be saved by only exposing the high-level API?
  (meaning the underlying API might be not particularly user friendly and/or unstable)
