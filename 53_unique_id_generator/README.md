# System Design Interview: Unique ID Generator (Snowflake) — "Perfect Answer" Playbook

## 0) Opening: Frame the Problem (30-60s)

"I'll design a distributed unique ID generation system inspired by Twitter's Snowflake. The system generates globally unique, roughly time-ordered, 64-bit IDs at massive scale without coordination between nodes. These IDs are used as primary keys for database rows, event ordering, and distributed tracing. I'll cover the bit layout, clock handling, datacenter/worker assignment, and how the system maintains uniqueness guarantees under clock skew and node failures."

## 1) Clarifying Questions (2-5 min)

| # | Question | Likely Answer |
|---|----------|---------------|
| 1 | What uniqueness guarantee? Globally unique across all services? | Yes, no collisions ever |
| 2 | Do IDs need to be sortable by time? | Yes, roughly time-ordered (within 1ms granularity) |
| 3 | What is the target generation rate? | 100K IDs/sec per node, 10M IDs/sec globally |
| 4 | 64-bit integer or variable length? | 64-bit integer (fits in a long) |
| 5 | How many datacenters and machines? | 32 datacenters, up to 1,024 worker nodes per datacenter |
| 6 | Can IDs be predictable? Security concerns? | IDs can reveal rough timestamp; if security matters, add obfuscation layer |
| 7 | What happens if the clock goes backward? | Must handle gracefully -- no duplicate IDs |
| 8 | Coordination allowed? | No coordination for ID generation (no central counter, no distributed lock) |

## 2) Back-of-Envelope Estimation (3-5 min)

| Metric | Estimate |
|--------|----------|
| Global ID generation rate | 10M IDs/sec |
| Per-node generation rate | 100K IDs/sec |
| Number of nodes | ~100 worker nodes |
| ID size | 64 bits = 8 bytes |
| IDs generated per day | 10M * 86,400 = 864B IDs/day |
| Storage for IDs alone per day | 864B * 8 bytes = 6.9 TB/day |
| Timestamp epoch lifespan | 41 bits = 2^41 ms = ~69.7 years |
| Sequence space per ms per node | 12 bits = 4,096 IDs/ms/node = 4.096M IDs/sec/node |
| Theoretical max global rate | 4,096 * 1,024 nodes * 1,000 ms = 4.19B IDs/sec |

## 3) Requirements (3-5 min)

### Functional Requirements
1. **Generate unique IDs**: produce 64-bit globally unique integers
2. **Time ordering**: IDs generated later have higher values (within clock precision)
3. **No coordination**: each node generates IDs independently without contacting others
4. **High throughput**: support 100K+ IDs/sec per node
5. **Embeddable metadata**: ID encodes timestamp, datacenter, and worker identity

### Non-Functional Requirements
- **Uniqueness**: absolute guarantee -- no duplicates under any circumstance
- **Latency**: < 1ms per ID generation (ideally < 1 microsecond)
- **Availability**: 99.999% -- ID generation cannot be a bottleneck
- **No single point of failure**: no centralized counter or coordinator
- **Clock resilience**: handle NTP clock adjustments and skew

## 4) API Design (2-4 min)

### ID Generation (local library call, not network RPC)
```
// In-process call (no network hop)
long generateId()
// Returns: 64-bit unique ID

// Batch generation for efficiency
long[] generateIds(int count)
// Returns: array of count unique IDs
```

### ID Parsing (utility)
```
GET /v1/ids/{id}/parse
Response: {
  "timestamp": 1709827200000,
  "datacenter_id": 5,
  "worker_id": 42,
  "sequence": 1234,
  "generated_at": "2024-03-07T16:00:00.000Z"
}
```

### Worker Registration
```
POST /v1/workers/register
{
  "datacenter_id": 5,
  "hostname": "worker-42.dc5.internal"
}
Response: { "worker_id": 42, "lease_ttl_sec": 300 }
```

## 5) Data Model (3-5 min)

### Snowflake ID Bit Layout (64 bits)

```
| 1 bit  | 41 bits    | 5 bits        | 5 bits     | 12 bits   |
|--------|------------|---------------|------------|-----------|
| unused | timestamp  | datacenter_id | worker_id  | sequence  |
| (sign) | (ms since  | (0-31)        | (0-31)     | (0-4095)  |
|        |  epoch)    |               |            |           |
```

- **Bit 63**: unused (sign bit, always 0 for positive IDs)
- **Bits 62-22 (41 bits)**: milliseconds since custom epoch (e.g., 2020-01-01). Supports ~69.7 years
- **Bits 21-17 (5 bits)**: datacenter ID (0-31, supports 32 datacenters)
- **Bits 16-12 (5 bits)**: worker ID (0-31 per datacenter, 1,024 workers total)
- **Bits 11-0 (12 bits)**: sequence number (0-4095, auto-increments within same millisecond)

### Alternative Layouts

**More workers, fewer sequence numbers:**
```
| 1 | 41 bits timestamp | 10 bits worker_id | 12 bits sequence |
```
- 1,024 workers globally, 4,096 IDs/ms/worker

**More sequence, fewer workers:**
```
| 1 | 41 bits timestamp | 5 bits datacenter | 5 bits worker | 12 bits sequence |
```
- Standard Snowflake layout (chosen above)

### Storage
- **Worker registry**: ZooKeeper or etcd -- stores datacenter_id + worker_id assignments
- **No persistent storage for ID generation itself** -- purely in-memory, stateless computation

## 6) High-Level Architecture (5-8 min)

Each application instance embeds a Snowflake ID generator as an in-process library. The generator maintains a worker_id (assigned at startup via ZooKeeper), the current timestamp, and a sequence counter. ID generation is a single-threaded, lock-free bit manipulation operation taking < 1 microsecond.

### Components
- **Snowflake Generator Library**: embedded in each application process, generates IDs locally
- **Worker ID Registry (ZooKeeper/etcd)**: assigns unique worker_id to each process at startup
- **Clock Monitor**: detects backward clock jumps, pauses generation until clock catches up
- **ID Parser**: utility to decompose IDs back into constituent parts

### Key Design Principle
No network calls during ID generation. The only network interaction is at startup (to lease a worker_id). After that, every ID is generated locally in < 1 microsecond.

## 7) Deep Dive #1: ID Generation Algorithm (8-12 min)

### Core Algorithm (pseudocode)

```java
class SnowflakeGenerator {
    private final long datacenterId;    // 5 bits
    private final long workerId;        // 5 bits
    private long lastTimestamp = -1;
    private long sequence = 0;
    private static final long EPOCH = 1577836800000L;  // 2020-01-01

    public synchronized long generateId() {
        long timestamp = currentTimeMillis();

        // Handle clock going backward
        if (timestamp < lastTimestamp) {
            long offset = lastTimestamp - timestamp;
            if (offset < 5) {
                // Small backward jump: wait it out
                sleep(offset);
                timestamp = currentTimeMillis();
            } else {
                throw new ClockMovedBackwardException(offset);
            }
        }

        if (timestamp == lastTimestamp) {
            // Same millisecond: increment sequence
            sequence = (sequence + 1) & 0xFFF;  // 12-bit mask
            if (sequence == 0) {
                // Sequence exhausted: wait for next millisecond
                timestamp = waitNextMillis(lastTimestamp);
            }
        } else {
            // New millisecond: reset sequence
            sequence = 0;
        }

        lastTimestamp = timestamp;

        return ((timestamp - EPOCH) << 22)  // 41 bits
             | (datacenterId << 17)          // 5 bits
             | (workerId << 12)              // 5 bits
             | sequence;                     // 12 bits
    }
}
```

### Clock Skew Handling

**Problem**: NTP can adjust system clock backward, which would produce duplicate timestamps.

**Strategies (layered):**
1. **Small backward jump (< 5ms)**: sleep until clock catches up. Adds brief latency but preserves uniqueness.
2. **Medium backward jump (5ms - 1s)**: use logical clock extension -- increment a "clock segment" counter stored in memory, temporarily reducing sequence bits.
3. **Large backward jump (> 1s)**: refuse to generate IDs, raise alert. This indicates serious clock misconfiguration.
4. **Prevention**: use `clock_gettime(CLOCK_MONOTONIC)` where possible, combined with periodic NTP offset tracking.

### Sequence Exhaustion

If more than 4,096 IDs are needed in a single millisecond on one worker:
- `waitNextMillis()`: busy-loop until system clock advances to the next millisecond
- At 4,096 IDs/ms, throughput is 4.096M IDs/sec per worker -- far exceeds typical needs
- If consistently hitting this limit, add more workers

### Thread Safety
- The `synchronized` keyword serializes access (simple, correct)
- For higher throughput: use `AtomicLong` CAS operations or thread-local generators with sub-worker-id bits
- In practice, synchronized overhead is < 100ns, which is negligible

## 8) Deep Dive #2: Worker ID Assignment and Coordination (5-8 min)

### Worker ID Assignment via ZooKeeper

**At process startup:**
1. Process connects to ZooKeeper
2. Creates an ephemeral sequential node under `/snowflake/workers/dc-{datacenter_id}/`
3. ZooKeeper assigns a sequence number (0-31) as the worker_id
4. Process reads back its assigned worker_id
5. Ephemeral node exists as long as the process is alive; deleted on disconnect

**Worker ID recycling:**
- When a process dies, its ephemeral node is deleted after ZooKeeper session timeout (~30s)
- New process can claim the freed worker_id
- **Critical**: must ensure old process is truly dead before reusing. ZooKeeper session timeout provides this guarantee.

**Alternative: Lease-based assignment (etcd):**
1. Process writes a key `/workers/{datacenter_id}/{worker_id}` with a TTL lease (e.g., 5 minutes)
2. Process periodically renews the lease
3. If lease expires (process crash), key is deleted, worker_id becomes available
4. Lease renewal must succeed before generating any ID (prevents split-brain)

### Pre-assigned Worker IDs (simpler alternative)
- In containerized environments (Kubernetes), assign worker_id from a pod's ordinal index (StatefulSet ordinal)
- `worker_id = pod_ordinal % 32`
- Datacenter ID from environment variable
- No ZooKeeper needed; simpler but requires careful deployment management

### Datacenter ID Assignment
- Static configuration per datacenter (0-31)
- Set via environment variable or configuration file
- Rarely changes; does not need dynamic assignment

## 9) Trade-offs and Alternatives (3-5 min)

| Decision | Alternative | Why This Choice |
|----------|-------------|-----------------|
| Snowflake (64-bit, embedded) | UUID v4 (128-bit, random) | Snowflake is half the size, sortable by time, and more efficient as DB index |
| Snowflake (64-bit) | UUID v7 (128-bit, time-ordered) | UUID v7 is time-ordered but 2x storage; Snowflake preferred when space matters |
| ZooKeeper for worker ID | Database auto-increment | ZooKeeper provides ephemeral nodes; DB auto-increment requires coordination per ID |
| Custom epoch (2020-01-01) | Unix epoch (1970-01-01) | Custom epoch extends usable range; Unix epoch wastes 50 years of timestamp space |
| 12-bit sequence | Larger sequence field | 4,096/ms/worker is sufficient; more sequence bits means fewer worker bits |
| Synchronized generation | Lock-free CAS | Synchronized is simpler and fast enough (< 100ns overhead) for most use cases |

### When NOT to Use Snowflake
- **Need unpredictable IDs**: Snowflake IDs leak timestamp and worker info. Use UUID v4 or encrypted IDs instead.
- **Need globally sequential IDs**: Snowflake provides rough ordering, not strict sequential. Use a centralized counter (with availability trade-off).
- **Low volume, simple system**: UUID v4 is simpler and good enough. Snowflake adds complexity for minimal benefit.

## 10) Scaling: 10x and 100x (3-5 min)

### 10x Scale (100M IDs/sec globally)
- **More workers**: expand worker_id field to 10 bits (1,024 workers), reduce datacenter_id
- **Multiple generators per process**: use sub-worker-ids from thread-local generators
- **Pre-generation**: batch-generate IDs ahead of time into a local buffer

### 100x Scale (1B IDs/sec globally)
- **Dual-epoch Snowflake**: use two independent Snowflake schemes with different epochs for different services (namespace isolation)
- **128-bit Snowflake**: expand to 128 bits with more timestamp precision, more workers, larger sequence
- **Hardware timestamping**: use FPGA or hardware clock for sub-microsecond precision, eliminating clock skew concerns
- **Regional independence**: each region runs an independent Snowflake scheme with region prefix bits

## 11) Reliability and Fault Tolerance (3-5 min)

| Failure Mode | Mitigation |
|--------------|------------|
| ZooKeeper down (can't register worker) | Pre-assigned worker IDs as fallback; existing workers continue generating |
| Clock skew (NTP adjustment) | Small: wait; Medium: logical clock; Large: halt and alert |
| Worker ID conflict (two processes same ID) | ZooKeeper ephemeral nodes prevent this; lease-based prevents with TTL gap |
| Sequence exhaustion (> 4,096/ms) | Wait for next millisecond; alert if consistently hitting limit |
| Datacenter isolation (network partition) | Each DC operates independently; IDs are unique by datacenter_id bits |
| Process crash mid-generation | No state to corrupt; new process gets new worker_id |

### Uniqueness Proof
Given the bit layout, two IDs can only collide if:
- Same timestamp (1ms granularity) AND
- Same datacenter_id AND
- Same worker_id AND
- Same sequence number

The algorithm guarantees sequence increments within the same millisecond, and worker_id assignment guarantees no two active processes share a worker_id. Therefore, collisions are impossible under correct operation.

## 12) Observability and Operations (2-4 min)

### Key Metrics
- IDs generated per second (per worker, per datacenter, global)
- Sequence utilization (how often sequence wraps within a millisecond)
- Clock skew events (backward jumps detected)
- Worker ID registration latency
- Worker ID pool utilization (how many of 32 are in use)

### Alerts
- Clock backward jump > 1 second
- Sequence exhaustion rate > 10 events/second (worker is overloaded)
- Worker ID pool > 80% utilized (running out of worker slots)
- ZooKeeper session expiry (worker ID may be lost)

### Operational Runbook
- **Clock skew alert**: check NTP configuration, validate `ntpstat`, restart NTP daemon
- **Worker ID exhaustion**: identify idle workers, scale down unused processes, expand worker_id bits
- **Sequence exhaustion**: add more workers to distribute load, or batch pre-generate IDs

## 13) Security (1-3 min)

- **Information leakage**: Snowflake IDs reveal generation timestamp, datacenter, and worker. If this is sensitive, apply format-preserving encryption (FPE) to the ID before exposing externally.
- **Enumeration**: sequential IDs within a millisecond are predictable. For user-facing IDs, hash or encrypt the Snowflake ID.
- **Worker ID spoofing**: worker registration requires authentication to ZooKeeper. Unauthorized processes cannot claim worker IDs.
- **Timestamp manipulation**: processes should not allow user-supplied timestamps in ID generation.

## 14) Team and Operational Considerations (1-2 min)

- **Infrastructure team (1-2 engineers)**: maintain Snowflake library, ZooKeeper cluster, monitoring
- **Client library**: published as a language-specific library (Java, Go, Python) consumed by all services
- **Zero-downtime upgrades**: new library versions backward-compatible; bit layout changes require migration
- **Testing**: property-based tests verifying uniqueness under concurrent generation, clock skew simulation

## 15) Common Follow-up Questions

**Q: How does Snowflake compare to UUID v7?**
A: UUID v7 is also time-ordered and uses 128 bits. Snowflake is 64 bits (fits in a long, better for DB indexes), but has less worker/sequence space. Use Snowflake when storage efficiency matters; UUID v7 when simplicity and standard compliance matter.

**Q: What if we need IDs to be strictly sequential across all nodes?**
A: That requires coordination (e.g., a centralized counter with Raft consensus). This sacrifices availability and throughput. Snowflake provides rough ordering, not strict sequential.

**Q: How do you handle multi-region with potential clock differences?**
A: Each region/datacenter has its own datacenter_id bits. IDs are unique across regions by construction. They may not be perfectly time-ordered across regions (due to clock differences), but they are unique.

**Q: Can Snowflake IDs be used as database primary keys?**
A: Yes, and they are excellent for this purpose. Time-ordered IDs result in sequential B-tree inserts, avoiding random I/O. Much better than UUID v4 for database performance.

**Q: What is the custom epoch and why not use Unix epoch?**
A: Custom epoch (e.g., 2020-01-01) maximizes the 41-bit timestamp range. Using Unix epoch (1970) wastes ~50 years of range, leaving only ~19 years of usable space.

## 16) Closing Summary (30-60s)

"I've designed a Snowflake-style distributed ID generator that produces 64-bit, time-ordered, globally unique IDs without coordination. The bit layout allocates 41 bits for millisecond timestamp (69 years), 10 bits for datacenter/worker identity (1,024 nodes), and 12 bits for sequence (4,096 IDs per millisecond per node). Each node generates IDs in-process in under 1 microsecond. Worker IDs are assigned via ZooKeeper ephemeral nodes. Clock skew is handled through a layered strategy: wait for small jumps, logical clock for medium, halt for large. The system supports 4M+ IDs/sec per node with zero network calls during generation."

## Appendix A: Configuration / Tuning Knobs

| Knob | Default | Range | Effect |
|------|---------|-------|--------|
| custom_epoch | 2020-01-01 | any past date | Earlier = less future range; later = more range |
| datacenter_id_bits | 5 | 3 - 8 | More bits = more DCs, fewer worker bits |
| worker_id_bits | 5 | 3 - 10 | More bits = more workers per DC |
| sequence_bits | 12 | 8 - 16 | More bits = higher throughput per node per ms |
| clock_backward_tolerance_ms | 5 | 1 - 100 | Max backward jump to wait out before erroring |
| zk_session_timeout_ms | 30000 | 10000 - 60000 | ZooKeeper session timeout for worker ID lease |
| sequence_start | 0 | 0 or random | Random start reduces collision risk during clock issues |

## Appendix B: Vocabulary Cheat Sheet

| Term | Definition |
|------|------------|
| Snowflake | Twitter's distributed ID generation scheme using 64-bit time-ordered IDs |
| Custom Epoch | A chosen start date for the timestamp, maximizing usable range |
| Sequence Number | Counter that increments within the same millisecond to ensure uniqueness |
| Worker ID | Unique identifier for each ID-generating process |
| Clock Skew | Difference in system clock between nodes or backward NTP adjustments |
| Ephemeral Node | ZooKeeper node that is deleted when the creating process disconnects |
| Monotonic Clock | System clock that never goes backward (but may not reflect wall time) |
| Format-Preserving Encryption | Encryption that produces output in the same format as input (64-bit to 64-bit) |
| UUID v7 | RFC 9562 standard for time-ordered 128-bit UUIDs |
| B-tree Locality | Sequential IDs insert at the end of B-tree indexes, reducing random I/O |

## Appendix C: References

- Twitter Snowflake (original blog post and open-source implementation)
- Instagram's ID generation with PostgreSQL
- UUID v7 (RFC 9562)
- "Designing Data-Intensive Applications" by Martin Kleppmann (Chapter 8: Sequence Number Ordering)
- Sonyflake (Sony's variant of Snowflake for fewer worker bits)
- ULID specification (Universally Unique Lexicographically Sortable Identifier)
- ZooKeeper ephemeral nodes documentation
