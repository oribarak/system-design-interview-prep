# Unique ID Generator (Snowflake) -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
We need to generate globally unique, roughly time-ordered, 64-bit IDs at massive scale without any coordination between nodes. These IDs serve as database primary keys, event ordering tokens, and distributed tracing identifiers. The hard part is guaranteeing uniqueness across thousands of nodes with zero network calls during generation while handling clock skew.

## 2. Requirements
**Functional:** Generate 64-bit globally unique integers. IDs must be roughly time-ordered. No coordination between nodes during generation. Support 100K+ IDs per second per node. Encode timestamp, datacenter, and worker identity in the ID.

**Non-functional:** Absolute uniqueness guarantee -- zero collisions ever. Sub-microsecond generation latency. 99.999% availability. No single point of failure. Resilient to NTP clock adjustments.

## 3. Core Concept: The 64-Bit Layout
A Snowflake ID packs four fields into 64 bits: 1 unused sign bit, 41 bits for millisecond timestamp (69.7 years from a custom epoch), 10 bits for datacenter/worker identity (1,024 nodes), and 12 bits for a sequence number (4,096 IDs per millisecond per node). Within the same millisecond on the same node, the sequence increments. In a new millisecond, it resets to zero. Since each node has a unique worker_id, two nodes can never produce the same ID even in the same millisecond. No network calls needed -- just bit manipulation.

## 4. High-Level Architecture
```
[Application Process]
      |
[Snowflake Generator Library] -- in-process, no network calls
      |  reads worker_id assigned at startup
      |
[ZooKeeper / etcd] -- assigns unique worker_id per process (startup only)
```
- **Snowflake Generator Library**: Embedded in each application process. Generates IDs via local bit manipulation in under 1 microsecond.
- **Worker ID Registry (ZooKeeper)**: Assigns unique worker_id to each process at startup using ephemeral nodes. Only network interaction.
- **Clock Monitor**: Detects backward clock jumps and pauses generation until the clock catches up.
- **ID Parser**: Utility to decompose IDs back into timestamp, datacenter, worker, and sequence.

## 5. How ID Generation Works
1. Application calls generateId() -- a local in-process function.
2. Generator reads current timestamp in milliseconds.
3. If same millisecond as last ID: increment the 12-bit sequence counter.
4. If sequence overflows (more than 4,096 in one ms): busy-wait for next millisecond.
5. If new millisecond: reset sequence to zero.
6. If clock went backward: wait for small jumps (under 5ms), error for large jumps.
7. Combine: (timestamp - epoch) shifted left 22 bits, OR datacenter_id shifted left 17, OR worker_id shifted left 12, OR sequence.

## 6. What Happens When Things Fail?
- **ZooKeeper down**: Existing workers continue generating (worker_id is already assigned). New workers use pre-assigned IDs (e.g., Kubernetes pod ordinal) as fallback.
- **Clock skew (NTP adjustment)**: Small backward jumps (under 5ms) are waited out. Large jumps (over 1 second) halt generation and alert -- this indicates serious misconfiguration.
- **Worker ID conflict**: ZooKeeper ephemeral nodes guarantee at most one process per worker_id. Session timeout (30s) ensures dead processes release their ID.

## 7. Scaling
- **10x**: Expand worker_id field to 10 bits (1,024 workers). Use thread-local generators with sub-worker-id bits. Pre-generate IDs into local buffers.
- **100x**: Expand to 128-bit layout with more timestamp precision, workers, and sequence space. Regional independence with region prefix bits. Hardware timestamping to eliminate clock skew.

## 8. Key Trade-offs to Mention
| Decision | Trade-off |
|----------|-----------|
| Snowflake (64-bit) vs UUID v4 (128-bit random) | Snowflake is half the size, time-sortable, better for DB indexes; UUID is simpler but larger and unsorted |
| Custom epoch (2020) vs Unix epoch (1970) | Custom epoch extends usable range by not wasting 50 years of timestamp space |
| Synchronized generation vs lock-free CAS | Synchronized is simpler and fast enough (under 100ns overhead); CAS needed only at extreme throughput |

## 9. Closing (30s)
> Snowflake generates 64-bit, time-ordered, globally unique IDs without coordination. The 41-bit timestamp gives 69 years, 10 bits identify 1,024 nodes, and 12 bits of sequence allow 4,096 IDs per millisecond per node. Each node generates IDs in-process in under 1 microsecond with zero network calls. Worker IDs are assigned at startup via ZooKeeper. Clock skew is handled by waiting for small jumps and halting for large ones. The uniqueness guarantee comes from the combination of unique worker_id and incrementing sequence within each millisecond.
