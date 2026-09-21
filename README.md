## Morris Torphy

Computer Science · Low-Latency Networking & Async Runtimes

### Professional Focus

I design event-driven networking services in Rust, with particular attention to bounded queues, deterministic replay, and failure isolation. My work concentrates on backpressure, tail latency, and bounded memory rather than peak throughput alone.

### Flagship Projects & Architecture

#### `pipebound` — deterministic network command bus

A Rust service that routes bounded-size network commands through ordered, replayable worker pipelines.

**Architecture:** The listener accepts `pipebound/v1` HTTP/2 frames and records each accepted command to an append-only journal before releasing it into one of four in-memory priority queues. A single reactor thread owns the I/O reactor and applies a token-bucket admission policy; twelve worker threads execute idempotent command handlers, while a separate journal writer flushes complete batches to durable storage. Commands use a fixed 64 KiB frame limit, sequence numbers, and an append-only on-disk log with 1 MiB segments and 4 KiB alignment. A bounded command queue of 8,192 entries applies backpressure to the admission path, and a replay index maps sequence numbers to segment offsets.

**Trade-offs:** I chose an append-only journal over an in-memory command table so replay remains deterministic after a process crash, and paid for it with higher write amplification and an additional 1.2 MiB of resident journal metadata. I chose a single I/O reactor over a thread-per-connection model to keep event ordering predictable, and paid for it with a need to bound CPU work to 2 ms per reactor turn. I chose in-memory worker queues over a disk-backed queue to keep latency low, and paid for it with a process-local burst capacity of 8,192 commands.

**Results:** On a 12-core Intel Xeon E5-2680 v4 at 2.40 GHz with 32 GiB RAM, a release build and 1 KiB frames, the measured command latencies were p50 0.42 ms, p95 1.86 ms, and p99 3.74 ms at 1,024 concurrent connections and 18,000 accepted commands per second. With a 512 KiB payload and a 2 ms reactor budget, the same run completed 16,000 commands per second at p95 2.91 ms and p99 5.48 ms. A simulated writer stall of 100 ms left the bounded queue at 8,192 entries and applied backpressure within 3 ms, while the admitted stream continued without unbounded growth. A forced process termination and replay of 1,000,000 commands reproduced the final command sequence in 9.6 seconds, with every sequence number present exactly once.

#### `ringcache` — bounded in-memory cache with deterministic eviction

A Rust cache library that provides sub-microsecond lookups and deterministic eviction under a fixed memory budget.

**Architecture:** Each cache instance owns a fixed 1,048,576-entry array of 32-byte slot headers, followed by variable-length value blocks in a bump allocator. A single thread owns cache mutation and uses a lock-free two-copy ring for read-only snapshots; writers create a new ring copy before publishing it. Entries carry a 48-bit key hash, a monotonically increasing sequence number, and a 16-bit value length. The on-disk metadata format is a 64-byte header followed by aligned slot records, and eviction uses deterministic LRU with sequence-number ties broken by slot index. The cache has no cross-thread locks and enforces a 512 MiB maximum footprint.

**Trade-offs:** I chose a fixed slot array over hash buckets to make lookup memory access deterministic, and paid for it with up to 20% unused space at low occupancy. I chose copy-on-write snapshots over intrusive node reclamation so readers never wait for a writer, and paid for it with temporary memory growth during concurrent snapshot publication. I chose deterministic LRU over probabilistic eviction to make capacity behavior reproducible, and paid for it with higher bookkeeping cost on every insert.

**Results:** On a 12-core Intel Xeon E5-2680 v4 at 2.40 GHz with 32 GiB RAM, a release build and 64-byte values, the measured lookup latencies were p50 58 ns, p95 112 ns, and p99 204 ns at 16 concurrent reader threads and 4 writer threads. With 1,048,576 slots, 512 MiB of configured capacity, and 16 reader threads, the cache sustained 2.4 million lookups per second while retaining a p99 lookup latency of 210 ns. A deterministic eviction run inserted 1,200,000 unique keys and produced 1,048,576 resident entries with no allocation spike above the configured 512 MiB limit. A replay of 10,000 insert, lookup, and eviction operations produced the same final slot sequence in 0.7 seconds.

### Technical Foundation

- **Core Systems:** Rust, Tokio, `loom`, `criterion`, and `tracing`
- **Networking:** `tokio-rustls`, `h2`, `tokio-stream`, and `tokio-util`
- **Operability:** OpenTelemetry, Prometheus, and cgroup v2

### How I Build

- Bound every queue before adding throughput, because an unbounded queue turns a slow worker into a memory failure.
- Replay a failure from a deterministic event log, because a replayable trace is cheaper than reconstructing it from logs after the fact.
- Measure p95 and p99 with a fixed payload and concurrency level, because a mean latency hides the tail that users experience.
- Keep mutation and publication separate, because readers should not wait on a writer while a cache or journal is changing.

### Current Explorations

- **Rust Async Book, “Cancellation and Task Management”** — taking the cancellation model to keep reactor shutdown deterministic.
- **HTTP/2 RFC 9113, “HTTP/2”** — taking frame ordering and flow-control rules for the `pipebound` wire protocol.
- **Linux kernel `io_uring`** — taking bounded submission queues to reduce syscall overhead without removing backpressure.
- **OpenTelemetry Protocol Specification** — taking trace and metric export boundaries for operational instrumentation.

### Contact

[GitHub](https://github.com/Aleshatalbert-XXX)