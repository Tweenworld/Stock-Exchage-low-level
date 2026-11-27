# `SPSCQueue` Design Notes

## Purpose

`SPSCQueue` is an ultra-low-latency, lock-free, bounded Single Producer Single Consumer (SPSC) queue.

It is designed for inter-thread message passing on critical execution paths, including:

- Market-data feed handling
- Order-routing gateways

The implementation relies on sequential layout guarantees and the Single-Writer Principle to eliminate cross-core atomic contention.

## Template Contract

```cpp
template <typename T, std::size_t N>
class SPSCQueue;
```

### Payload type: `T`

`T` must be flat and trivially copyable. The trivially-copyable requirement provides deterministic memory operations and prevents the queue from requiring:

- Heap allocations
- Virtual-call indirections
- Complex lifecycle destructors on the critical path

It also enables the compiler to use SIMD/vectorized `memcpy` operations.

### Capacity: `N`

`N` is a compile-time queue capacity and must satisfy both of these requirements:

1. `N` must be greater than one. At least two slots are required to distinguish safely between full and empty states.
2. `N` must be a power of two. This permits bitwise indexing through a compile-time mask.

The power-of-two requirement replaces hardware modulo indexing with a bitwise `AND` operation. The implementation comments identify this as an optimization from an approximately 10-40 cycle modulo operation to a sub-nanosecond, single-cycle mask operation.

## Concurrency Model

The queue has exactly one producer and one consumer:

- The producer exclusively advances `tail_` and calls `push`.
- The consumer exclusively advances `head_` and calls `pop`.

This ownership model is the basis for the relaxed atomic loads used by each side when reading its own cursor. The queue is intended for concurrent execution by the producer and consumer on separate hardware cores.

## Ring-Buffer Indexing

The queue stores payloads in an inline array:

```cpp
std::array<T, N> buffer_{};
```

The physical slot is selected with:

```cpp
tail & (N - 1)
head & (N - 1)
```

The mask is calculated once at compile time as `N - 1`. Monotonically increasing unsigned cursor values are used for logical positions. Unsigned subtraction naturally handles integer overflow when determining whether the queue is full.

## Producer Path: `push`

`push` is exclusively executed by the producer thread.

1. Load the producer-owned `tail_` cursor with `memory_order_relaxed`.
2. Load the consumer-owned `head_` cursor with `memory_order_acquire` so slots freed by the consumer are structurally visible.
3. Treat the queue as full when:

   ```cpp
   (tail - head) == N
   ```

4. If full, return `false`. The full-state branch is marked `[[unlikely]]`.
5. Write the payload directly into the inline buffer slot selected by the tail mask.
6. Publish the completed payload by storing `tail + 1` with `memory_order_release`.
7. Return `true`.

The release store ensures that the slot assignment is committed before the updated producer index is published.

## Consumer Path: `pop`

`pop` is exclusively executed by the consumer thread.

1. Load the consumer-owned `head_` cursor with `memory_order_relaxed`.
2. Load the producer-owned `tail_` cursor with `memory_order_acquire` so the written payload is synchronized.
3. Treat the queue as empty when:

   ```cpp
   head == tail
   ```

4. If empty, return `false`. The empty-state branch is marked `[[unlikely]]`.
5. Copy the payload directly from the inline buffer slot selected by the head mask into the output reference.
6. Publish the released slot by storing `head + 1` with `memory_order_release`.
7. Return `true`.

The release store informs the producer that the consumer has finished reading the slot and that it may be overwritten.

## Memory-Ordering Protocol

The queue uses ownership-aware atomic operations:

| Operation | Cursor | Ordering | Role |
| --- | --- | --- | --- |
| Producer load | `tail_` | Relaxed | Reads the cursor written only by the producer. |
| Producer load | `head_` | Acquire | Observes slots released by the consumer. |
| Producer store | `tail_` | Release | Publishes the completed payload write. |
| Consumer load | `head_` | Relaxed | Reads the cursor written only by the consumer. |
| Consumer load | `tail_` | Acquire | Observes the producer's completed payload write. |
| Consumer store | `head_` | Release | Publishes that the consumed slot is free. |

The release/acquire pairs provide the publication and reclamation boundaries for payload slots:

- Producer release of `tail_` pairs with the consumer acquire of `tail_`.
- Consumer release of `head_` pairs with the producer acquire of `head_`.

## Cache-Isolation Strategy

Each cursor is wrapped in a `Cursor` structure aligned to 128 bytes:

```cpp
struct alignas(128) Cursor {
    std::atomic<std::uint64_t> value{0};
};
```

The alignment isolates the cursor from adjacent cache lines. This is intended to prevent the CPU's MESI cache-coherency protocol and adjacent-line prefetchers from causing destructive cross-core cache invalidation when the producer and consumer run concurrently on separate hardware cores.

`head_` and `tail_` are sequentially separated control structures. Each occupies its own isolated 128-byte cache sandbox so the producer and consumer write loops can execute concurrently without hardware line contention.

## Memory Layout

The queue uses an inline, fixed-size layout:

```text
[ inline payload buffer ][ isolated head cursor ][ isolated tail cursor ]
```

The inline buffer removes pointer indirection and is intended to maximize L1/L2 data-cache prefetching. The queue does not depend on dynamic storage for its payload array.

## API Contract

### `bool push(const T& value) noexcept`

Attempts to enqueue one trivially copyable payload.

- Returns `true` when the payload is written and published.
- Returns `false` when the bounded queue has reached capacity.
- Accepts the payload by reference to avoid an upfront copy at the call boundary.

### `bool pop(T& value) noexcept`

Attempts to dequeue one payload.

- Returns `true` when an item is fetched into `value`.
- Returns `false` when the queue is empty.
- Populates the caller's reference directly, avoiding an intermediate stack copy.

### `bool empty() const noexcept`

Provides thread-safe queue-state inspection for out-of-band monitoring or control loops.

It compares acquire-loaded `head_` and `tail_` cursors. `[[nodiscard]]` is used so an ignored empty-state result is treated as an unhandled logical assertion by the compiler.

## Design Summary

`SPSCQueue` combines:

- A bounded, compile-time-sized ring buffer
- Power-of-two bitwise indexing
- Trivially copyable payloads
- Single-writer cursor ownership
- Acquire/release publication boundaries
- 128-byte cursor alignment
- Inline payload storage
- Unlikely full and empty branches

Together, these choices target deterministic, low-latency message passing for critical exchange-processing paths.
