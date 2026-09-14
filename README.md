## Benita Hsieh

Computer Science · Storage Engines & Observability

### Professional Focus

I design storage engines and observability paths around deterministic invariants, bounded memory, and fast recovery after a single node or disk failure. My work centers on on-disk formats, concurrent access paths, and operational signals that expose latency and durability behavior before they become incidents.

### Flagship Projects & Architecture

#### LedgerKit

A single-node, append-only key-value store built for deterministic replay, crash recovery, and predictable tail latency.

**Architecture:** LedgerKit stores a copy-on-write B+ tree in a segmented log with 64 KiB segments and variable-length 4 KiB records. A single writer appends through a bounded 64-entry queue; readers hold pinned segment offsets and use atomics for epoch transitions, avoiding heap allocation in the read path. Recovery replays a CRC32C-framed WAL from the last checkpoint and rebuilds a temporary index before publishing the active tree. The on-disk format uses little-endian headers, 16-byte record frames, and a 64-byte checkpoint footer; the client protocol uses length-prefixed binary commands over one TCP connection. The writer serializes mutations to preserve a single order of record application, while readers observe only committed segment generations.

**Trade-offs:** I chose append-only writes over an in-place LSM tree because sequential I/O gives deterministic recovery and simpler compaction, and paid for extra disk usage through segment retention. I chose pinned segment references over reference-counted node objects because readers cannot retain heap ownership during replay, and paid for occasional range scans when a segment must be traversed after a missed index hint.

**Results:** On a 14-inch Apple M2 laptop with an 8 GiB memory limit and a 2 GiB working set, a 100,000-key workload reached 48,000 commits per second at p50 commit latency of 1.8 ms, p95 of 3.4 ms, and p99 of 5.1 ms using a 4 KiB value, 8 concurrent writers, and `go test -race`-equivalent deterministic replay checks. After a forced process kill, the same dataset recovered in 96 ms with p95 read latency of 2.6 ms and p99 of 4.3 ms across 8 concurrent readers. With 10,000 keys and 64 concurrent readers, a pinned-segment read path held p95 latency at 0.42 ms and p99 at 0.71 ms; a reference-counted variant measured 0.58 ms and 0.94 ms under the same conditions.

#### TraceScope

A local distributed tracing collector that records bounded spans, aggregates latency histograms, and exposes deterministic diagnostics without retaining full traces.

**Architecture:** TraceScope accepts protobuf spans over HTTP/2, validates schema and trace identifiers, and writes them to a 32 MiB mmap-backed ring buffer. A single ingestion worker batches up to 256 spans or 16 ms, then a fixed set of aggregation workers updates sharded per-service histograms with atomic counters. The compact on-disk format uses a 4 KiB header, 16-byte metadata records, and 64-byte histogram buckets; the wire protocol uses HTTP/2 headers, protobuf bodies, and an explicit end-of-stream marker. Backpressure returns HTTP 429 with a retry-after hint when the ring buffer reaches 90% occupancy, while a durable checkpoint records the last acknowledged span sequence. A separate diagnostic reader walks the compact format without mutating the active aggregation, so analysis does not compete with ingestion.

**Trade-offs:** I chose fixed histogram buckets over arbitrary span retention because bounded memory makes overload behavior predictable, and paid for reduced query precision at bucket boundaries. I chose HTTP/2 multiplexing over a custom binary protocol because existing clients and TLS termination fit the deployment model, and paid for higher per-connection state and a larger ingestion footprint.

**Results:** On a 4-core Linux VM with a 2 GiB memory limit, a 1,000 span/s workload using 512-byte payloads reached 1,150 spans per second at p50 ingestion latency of 7.2 ms, p95 of 14.8 ms, and p99 of 22.6 ms. At 2,000 spans per second, the 90% occupancy threshold triggered HTTP 429 responses and the process stayed below 1.4 GiB of resident memory. A forced collector restart replayed the last 4,096 sequence numbers in 31 ms, with p95 diagnostic read latency of 9.1 ms and p99 of 15.7 ms across 4 concurrent readers. With 16 concurrent writers and a 64 KiB payload, the ring buffer held p95 latency at 18.9 ms and p99 at 29.4 ms before applying backpressure.

### Technical Foundation

**Core Systems:** `Rust`, ` Tokio`, `mmap`, `crossbeam`, `serde`.

**Storage & Data:** `sled`, `prost`, `leveldb`, `raft`, `btree`.

**Infrastructure & Observability:** `Prometheus`, `OpenTelemetry`, `jq`, `systemtap`, `Linux perf`.

### How I Build

- I define invariants before writing code so each concurrency boundary has a testable contract.
- I bound queues and memory so overload produces measurable backpressure instead of uncontrolled growth.
- I replay deterministic workloads so latency and recovery results can be compared across changes.
- I record failure mode, measurement conditions, and trade-offs in the same document as the design.

### Current Explorations

- **Paper:** *Raft: In Search of an Understandable Consensus Protocol* by Diego Ongaro and John Ousterhout; I am taking its log-matching and leader-election invariants into the storage recovery path.
- **RFC:** RFC 4122, *A UUID Namespace*; I am taking its deterministic identifier rules into trace and segment naming.
- **Kernel subsystem:** Linux `io_uring`; I am taking its fixed-buffer and submission-completion queue model into the bounded ingestion path.

### Contact

GitHub: [Benita Hsieh](https://github.com/BenitaHsieh)