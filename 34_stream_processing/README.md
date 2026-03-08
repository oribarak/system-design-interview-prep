# System Design Interview: Real-Time Stream Processing (Kafka-style) -- "Perfect Answer" Playbook

---

## 0) Opening: Frame the Problem (30-60s)

> "We need to design a real-time stream processing platform -- think Kafka Streams or Flink -- that ingests high-throughput event streams, processes them with transformations, aggregations, and windowed joins, and outputs results with low latency. The key challenges are exactly-once processing semantics, stateful computation with fault tolerance, handling out-of-order events (watermarks), and scaling to millions of events per second. I will cover the messaging layer, stream processing engine, state management, and output sinks."

---

## 1) Clarifying Questions (2-5 min)

**Scope**
- Are we designing just the processing engine, or also the underlying message broker (Kafka)?
- Should the system support both stateless (map, filter) and stateful (aggregate, join, windowed) operations?
- Do we need SQL-like query support (like KSQL/Flink SQL) or a programmatic API?

**Scale**
- What is the event throughput? (Assume 2M events/sec, average event size 500 bytes = ~1 GB/s.)
- How many concurrent processing jobs? (Assume 500 active stream processing jobs.)
- What are the state sizes for stateful operators? (Assume up to 500 GB per job for windowed aggregations.)

**Policy / Constraints**
- What processing guarantees? (Exactly-once semantics required for financial/billing use cases.)
- What is the acceptable end-to-end latency? (p99 < 1 second from event ingestion to output.)
- How long should events be retained in the message broker? (7 days for replay.)

---

## 2) Back-of-Envelope Estimation (3-5 min)

| Metric | Value |
|---|---|
| Event throughput | 2,000,000 events/sec |
| Average event size | 500 bytes |
| Ingestion bandwidth | 2M x 500B = 1 GB/s |
| Broker retention | 7 days |
| Broker storage | 1 GB/s x 86,400 x 7 = ~605 TB |
| Replication factor | 3 |
| Total broker storage | 605 TB x 3 = ~1.8 PB |
| Active processing jobs | 500 |
| Avg partitions per topic | 128 |
| Total partition count | ~20,000 (across all topics) |
| Stateful job state size | Up to 500 GB per job |
| Checkpoint interval | Every 30 seconds |
| Checkpoint size (incremental) | ~50 MB per job per checkpoint |

---

## 3) Requirements (3-5 min)

### Functional
- **Event ingestion**: Publish events to named topics with partitioning by key.
- **Stream transformations**: Map, filter, flatMap operations on individual events.
- **Stateful aggregations**: Count, sum, average over tumbling, sliding, and session windows.
- **Stream joins**: Join two streams by key within a time window; join a stream with a table (changelog).
- **Exactly-once output**: Guarantee each event affects the output exactly once, even across failures.
- **Event replay**: Reprocess historical events by resetting consumer offsets.

### Non-functional
- **Throughput**: Process 2M+ events/sec across the platform.
- **Latency**: p99 end-to-end < 1 second for stateless operations, < 5 seconds for windowed aggregations.
- **Fault tolerance**: Recover from worker failures within 30 seconds with no data loss.
- **Scalability**: Horizontally scale by adding workers and partitions.
- **Backpressure**: Gracefully handle slow consumers without dropping events.
- **Ordering**: Maintain per-partition event ordering guarantees.

---

## 4) API Design (2-4 min)

### Producer API
```
POST /topics/{topic}/messages
Headers: X-Partition-Key: {key}
Body: { "value": <bytes>, "timestamp": <epoch_ms>, "headers": {...} }

Response: { "partition": 7, "offset": 123456789 }
```

### Job Submission (Programmatic DAG)
```
POST /api/v1/jobs
Body: {
  "name": "click-aggregation",
  "parallelism": 64,
  "dag": {
    "sources": [{"topic": "clicks", "deserializer": "json"}],
    "operators": [
      {"id": "filter", "type": "filter", "predicate": "event.type == 'click'"},
      {"id": "keyBy", "type": "key_by", "key": "event.user_id"},
      {"id": "window", "type": "tumbling_window", "size": "1m"},
      {"id": "agg", "type": "aggregate", "function": "count"}
    ],
    "sinks": [{"topic": "click-counts", "serializer": "json"}]
  }
}
Response: { "job_id": "job-42", "status": "RUNNING" }
```

### Job Management
```
GET /api/v1/jobs/{job_id}            -- Get job status, metrics
POST /api/v1/jobs/{job_id}/savepoint -- Trigger savepoint for upgrade
DELETE /api/v1/jobs/{job_id}         -- Stop job
POST /api/v1/jobs/{job_id}/rescale?parallelism=128  -- Rescale
```

### Consumer API
```
GET /topics/{topic}/partitions/{partition}/messages?offset={offset}&max_bytes={bytes}

Response: { "messages": [{ "offset": 123, "key": <bytes>, "value": <bytes>, "timestamp": <ms> }] }
```

---

## 5) Data Model (3-5 min)

### Message Broker (Kafka-style)

**Topic Partition Segment (on-disk log)**
| Field | Type | Notes |
|---|---|---|
| offset | INT64 | Monotonically increasing per partition |
| timestamp | INT64 | Event time or broker receive time |
| key | BYTES | Partition routing key |
| value | BYTES | Serialized event payload |
| headers | MAP<STRING,BYTES> | Metadata (trace ID, schema version) |
| crc | UINT32 | Checksum for corruption detection |

**Consumer Group Offsets**
| Column | Type | Notes |
|---|---|---|
| group_id | STRING (PK) | Consumer group name |
| topic | STRING (PK) | Topic name |
| partition | INT32 (PK) | Partition number |
| committed_offset | INT64 | Last committed offset |
| metadata | STRING | Application metadata |

### Stream Processing State

**State Store (RocksDB)**
| Column | Type | Notes |
|---|---|---|
| key | BYTES | Composite: (window_start, group_key) |
| value | BYTES | Serialized aggregate (count, sum, etc.) |
| ttl | INT64 | Auto-expire after window close + allowed lateness |

**Checkpoint Metadata**
| Column | Type | Notes |
|---|---|---|
| job_id | STRING (PK) | Processing job identifier |
| checkpoint_id | INT64 | Monotonically increasing |
| timestamp | INT64 | Checkpoint trigger time |
| input_offsets | MAP<STRING, MAP<INT32, INT64>> | Topic -> partition -> offset |
| state_handle | STRING | Path to state snapshot in object storage |
| status | ENUM | PENDING, COMPLETED, FAILED |

**Storage choices**: Broker uses append-only log segments on SSD with page-cache-friendly sequential I/O. State stores use embedded RocksDB (LSM tree) on local SSD with asynchronous checkpoints to S3.

---

## 6) High-Level Architecture (5-8 min)

### Dataflow
1. **Producers** publish events to topic partitions via the **Broker Cluster** using a partition key hash.
2. **Broker Cluster** persists events in replicated, append-only log segments and serves reads to consumers.
3. **Job Manager** receives job submissions, generates an execution DAG, and schedules **Task Managers** (workers).
4. **Task Managers** consume from assigned partitions, apply operators (filter, map, aggregate, join), maintain local state in **RocksDB**, and produce output to sink topics.
5. **Checkpoint Coordinator** (in Job Manager) periodically injects checkpoint barriers into the stream; Task Managers snapshot their state and offsets to **Object Storage** upon receiving barriers.
6. On failure, the Job Manager restores the last checkpoint: reassigns partitions, restores RocksDB state, and resumes from checkpointed offsets.
7. **Output sinks** consume processed results and deliver to databases, dashboards, or downstream topics.

### Components
- **Broker Cluster**: Distributed commit log with partitioned, replicated topics.
- **Schema Registry**: Stores and validates Avro/Protobuf schemas for topics.
- **Job Manager**: Coordinates job lifecycle, scheduling, checkpointing, and failure recovery.
- **Task Manager (Worker)**: Executes operators, manages local state, consumes/produces events.
- **RocksDB State Store**: Embedded key-value store for stateful operators.
- **Checkpoint Coordinator**: Injects barriers, tracks checkpoint completion.
- **Object Storage (S3)**: Durable storage for state checkpoints and savepoints.
- **ZooKeeper / KRaft**: Broker metadata, controller election, partition assignment.
- **Metrics / Monitoring**: Tracks throughput, latency, consumer lag, checkpoint duration.

> **One-sentence summary**: Producers write events to a partitioned, replicated broker log; Task Managers consume partitions, apply a DAG of operators with RocksDB-backed state, and checkpoint offsets and state snapshots to object storage for exactly-once fault recovery.

---

## 7) Deep Dive #1: Exactly-Once Processing with Checkpointing (8-12 min)

Exactly-once semantics (EOS) is the hardest guarantee in stream processing. The system must ensure every event affects the output exactly once, even when workers crash or restart.

### Chandy-Lamport Distributed Snapshots (Flink-style)

**Checkpoint Barriers**:
1. The Checkpoint Coordinator (in Job Manager) periodically injects a special **barrier marker** (not a data event) into each source partition.
2. As a barrier flows through the DAG, each operator processes it:
   - When an operator receives a barrier on one input, it **pauses** that input (barrier alignment).
   - When barriers from all inputs have arrived, the operator snapshots its state to S3 and forwards the barrier downstream.
3. When all sink operators have received the barrier, the checkpoint is **complete**.

**Why this guarantees exactly-once**:
- The checkpoint captures a consistent cut: all events before the barrier are reflected in the state; no events after it are.
- On recovery, the system restores state from the last completed checkpoint and replays events from the checkpointed offsets. Events between the checkpoint and the failure are re-processed, but since state is also restored, the output is identical.

### Barrier Alignment vs. Unaligned Checkpoints
- **Aligned** (default): Operator pauses fast inputs until slow inputs deliver their barriers. Ensures exactly-once but can cause backpressure spikes.
- **Unaligned**: Operator immediately snapshots and buffers in-flight records between barriers. Reduces latency but increases checkpoint size.

### End-to-End Exactly-Once (Kafka Transactions)
For Kafka-to-Kafka pipelines:
1. Task Manager begins a Kafka transaction.
2. Produces output records to sink topics within the transaction.
3. Commits consumer offsets and output records atomically when the checkpoint completes.
4. If the checkpoint fails, the transaction is aborted; output records are invisible to downstream consumers (read_committed isolation).

### Idempotent Sinks (for non-Kafka outputs)
When the sink is a database:
- Include the `(job_id, checkpoint_id)` in the output record.
- The sink performs an upsert keyed on a deterministic ID derived from the input event.
- On replay after failure, the same records are written with the same keys, producing identical results.

### Checkpoint Performance
- **Incremental checkpoints**: Only changed RocksDB SST files are uploaded to S3 (not the full state).
- **Asynchronous snapshots**: RocksDB supports lightweight snapshots without blocking the write path.
- Typical checkpoint: 50 MB incremental upload in < 2 seconds for 500 GB state.

---

## 8) Deep Dive #2: Windowed Aggregations and Watermarks (5-8 min)

Real-world events arrive out of order. A click at 10:00:05 might arrive at 10:00:12 due to network delays. Windowed aggregations must handle this correctly.

### Window Types
- **Tumbling**: Fixed-size, non-overlapping (e.g., every 1 minute: [10:00, 10:01), [10:01, 10:02)).
- **Sliding**: Fixed-size, overlapping (e.g., 5-min window sliding every 1 min).
- **Session**: Dynamic windows that close after an inactivity gap (e.g., 30-min gap between user events).

### Watermarks
A watermark `W(t)` is a declaration: "All events with event time <= t have been observed."

**Watermark generation**:
- Each source partition tracks the maximum event time seen minus an **allowed lateness** (e.g., 10 seconds).
- The global watermark is the minimum across all source partitions.
- When the watermark passes a window's end time, the window is "fired" (output emitted).

**Late events**:
- Events arriving after the watermark passes their window are "late."
- **Policy options**: (1) Drop and count them. (2) Emit an updated result (retractions/updates). (3) Route to a side output for manual processing.
- **Allowed lateness**: Windows are kept in state for an additional period (e.g., 1 hour) to accept late updates.

### State Management for Windows
- Each window is keyed by `(group_key, window_start, window_end)`.
- State stored in RocksDB with a TTL = window_end + allowed_lateness.
- When the watermark passes window_end, the window fires: read state, compute final result, emit output, and schedule state cleanup after allowed_lateness.

### Example: Counting clicks per user per minute
```
clicks_stream
  .keyBy(event -> event.userId)
  .window(TumblingEventTimeWindows.of(Duration.ofMinutes(1)))
  .allowedLateness(Duration.ofMinutes(5))
  .aggregate(new CountAggregator())
  .sinkTo(outputTopic)
```

---

## 9) Trade-offs and Alternatives (3-5 min)

| Decision | Chosen | Alternative | Rationale |
|---|---|---|---|
| Processing model | Event-at-a-time (Flink-style) | Micro-batch (Spark Structured Streaming) | Lower latency; true per-event processing. Micro-batch adds batch-interval latency |
| State backend | Embedded RocksDB | External Redis / DynamoDB | RocksDB is co-located (no network hop); supports incremental checkpoints; no external dependency |
| Checkpointing | Chandy-Lamport barriers | Log-based (Kafka Streams changelog) | Barriers give consistent snapshots across operators; changelogs are simpler but coupled to Kafka |
| Exactly-once | Kafka transactions + checkpoints | At-least-once + idempotent sinks | Transactions give true EOS; idempotent sinks are simpler but require careful key design |
| Coordination | ZooKeeper / KRaft | etcd / Consul | ZooKeeper is proven for Kafka; KRaft removes the dependency entirely |

---

## 10) Scaling: 10x and 100x (3-5 min)

### 10x (20M events/sec)
- **Increase partitions**: Scale topics from 128 to 1,024 partitions; add proportional Task Managers.
- **Broker fleet**: Grow from ~30 to ~300 broker nodes; use tiered storage (hot SSD + warm S3) to manage 18 PB retention.
- **Task Manager resource isolation**: Run each Task Manager in a container with dedicated CPU and memory; use Kubernetes for orchestration.
- **Incremental checkpoints**: Critical at this scale; full checkpoints would be prohibitively large.

### 100x (200M events/sec)
- **Multi-cluster federation**: Regional broker clusters with cross-region replication for critical topics.
- **Disaggregated storage**: Brokers become stateless compute; all log segments stored in S3 with a caching tier (like Confluent's tiered storage or WarpStream).
- **Adaptive parallelism**: Auto-scale Task Managers based on consumer lag metrics. Re-partition on the fly using consistent hashing.
- **Columnar state store**: For analytical workloads, replace RocksDB with a columnar format for faster aggregation scans.
- **Edge pre-aggregation**: Push simple aggregations (counts, sums) to edge nodes to reduce central cluster load.

---

## 11) Reliability and Fault Tolerance (3-5 min)

- **Broker failure**: Replicated partitions (RF=3) with ISR (in-sync replicas). Leader election promotes an ISR member. Producers retry; no data loss if `acks=all`.
- **Task Manager crash**: Job Manager detects failure (heartbeat timeout 30s). Restores last checkpoint: reassigns partitions to healthy workers, downloads state from S3, replays from checkpointed offsets.
- **Job Manager failure**: Standby Job Manager takes over via leader election. Job state (checkpoints, DAG) is stored in ZooKeeper/S3.
- **Network partition**: Brokers use ISR mechanism -- partitions with insufficient ISR members become read-only. Producers get errors and retry.
- **Checkpoint failure**: If a checkpoint fails (e.g., S3 timeout), the system continues with the previous checkpoint. Alert if multiple consecutive checkpoints fail.
- **Poison pill events**: Malformed events that cause operator exceptions are routed to a dead-letter topic for investigation, not retried infinitely.

---

## 12) Observability and Operations (2-4 min)

- **Consumer lag**: The most critical metric. Lag = latest broker offset - consumer committed offset. Alert if lag grows consistently.
- **Checkpoint duration and size**: Track per-job; increasing duration signals growing state or I/O bottlenecks.
- **Throughput per operator**: Records/sec in vs. out for each operator in the DAG. Identify bottleneck operators.
- **Backpressure indicator**: Ratio of time an operator spends waiting for output buffer space. >50% indicates downstream bottleneck.
- **Event time skew**: Difference between watermark and wall clock. Large skew means events are significantly delayed.
- **GC pauses**: JVM-based systems (Flink, Kafka) are susceptible to GC pauses. Monitor and tune G1GC.
- **Operational tools**: Savepoints for planned upgrades (snapshot state, stop job, restart with new code, restore from savepoint).

---

## 13) Security (1-3 min)

- **Authentication**: SASL/SCRAM or mTLS for broker connections. OAuth2 tokens for job submission API.
- **Authorization**: ACLs per topic (produce, consume, describe). Per-job permissions for state access.
- **Encryption**: TLS in transit between all components. SSE-S3 for checkpoints and log segments at rest.
- **Multi-tenancy**: Topic-level isolation. Resource quotas per tenant (bytes/sec produce, consume). Separate Task Manager pools for different trust levels.
- **Schema enforcement**: Schema Registry rejects messages that do not conform to the registered schema, preventing data corruption.

---

## 14) Team and Operational Considerations (1-2 min)

- **Broker team**: Manages Kafka cluster, partition rebalancing, broker upgrades, capacity planning.
- **Processing platform team**: Owns Job Manager, Task Manager, checkpointing infrastructure, and SDK/API.
- **Application teams**: Build and deploy stream processing jobs using the platform's SDK.
- **Deployment**: Rolling upgrades for brokers (one at a time, maintain ISR). Savepoint-based upgrades for processing jobs: take savepoint, deploy new version, restore from savepoint.
- **Capacity planning**: Monitor partition count growth, consumer lag trends, and state store sizes. Proactively add brokers and workers.

---

## 15) Common Follow-up Questions

**Q: How do you handle schema evolution?**
A: Schema Registry with compatibility modes (backward, forward, full). Consumers can read data written with an older schema. Breaking changes require a new topic and migration job.

**Q: How does backpressure work?**
A: TCP-based flow control between Task Managers. If a downstream operator is slow, its input buffer fills up, which propagates back through the network stack to slow the upstream operator. Ultimately, the source slows its consumption from the broker.

**Q: How do you reprocess historical data?**
A: Reset consumer group offsets to the desired timestamp. The processing job replays all events from that point. For stateful jobs, start from a clean state or a savepoint taken at that time.

**Q: Kafka Streams vs. Flink -- when to choose which?**
A: Kafka Streams is a library (runs in your app, no separate cluster) -- good for simpler jobs tightly coupled to Kafka. Flink is a full cluster -- better for complex DAGs, large state, and multi-source joins.

**Q: How do you handle hot partitions?**
A: Detect via per-partition lag monitoring. Solutions: (1) Add a random suffix to the key to spread load, then aggregate in a second stage. (2) Split the hot partition into sub-partitions. (3) Increase parallelism for the specific operator.

---

## 16) Closing Summary (30-60s)

> "We designed a real-time stream processing platform handling 2M events/sec with exactly-once semantics. The messaging layer uses a partitioned, replicated commit log (Kafka-style) with 7-day retention. The processing engine schedules DAGs of operators across Task Managers, each maintaining local RocksDB state. Exactly-once is achieved through Chandy-Lamport checkpoint barriers that create consistent distributed snapshots stored in S3, combined with Kafka transactions for atomic offset and output commits. Windowed aggregations use watermarks to handle out-of-order events. The system scales by adding partitions and workers, and recovers from failures within 30 seconds by restoring checkpoints."

---

## Appendix A: Configuration / Tuning Knobs

| Knob | Default | Tune When |
|---|---|---|
| `num.partitions` | 128 | More partitions = more parallelism but more overhead |
| `replication.factor` | 3 | Lower for cost; higher for durability |
| `acks` | all | "1" for higher throughput with durability trade-off |
| `checkpoint.interval` | 30s | Lower for faster recovery; higher for less I/O |
| `checkpoint.mode` | ALIGNED | UNALIGNED for latency-sensitive jobs |
| `state.backend` | RocksDB | HashMapStateBackend for small state (< 1 GB) |
| `allowed.lateness` | 5m | Longer for data completeness; shorter for faster cleanup |
| `max.parallelism` | 256 | Upper bound for future rescaling |
| `network.buffer.size` | 32 KB | Larger for throughput; smaller for latency |
| `retention.ms` | 604800000 (7d) | Longer for more replay capability |

## Appendix B: Vocabulary Cheat Sheet

| Term | Definition |
|---|---|
| **Partition** | An ordered, immutable sequence of events within a topic; unit of parallelism |
| **Consumer Group** | A set of consumers that share partition assignments for a topic |
| **Offset** | A monotonically increasing position within a partition |
| **Watermark** | A timestamp assertion: all events with time <= watermark have been observed |
| **Checkpoint Barrier** | A special marker injected into the stream to trigger consistent snapshots |
| **Savepoint** | A manually triggered, portable checkpoint used for job upgrades |
| **ISR** | In-Sync Replicas; the set of replicas caught up to the leader |
| **Exactly-Once** | Each event affects the output exactly once, even across failures |
| **Backpressure** | Flow control mechanism that slows upstream operators when downstream is slow |
| **Tumbling Window** | Fixed-size, non-overlapping time window for aggregation |
| **Session Window** | Dynamic window that closes after an inactivity gap |
| **Dead Letter Topic** | Destination for events that fail processing after retries |

## Appendix C: References

- Kafka: A Distributed Messaging System for Log Processing (LinkedIn, 2011)
- Apache Flink: Stream and Batch Processing in a Single Engine (2015)
- Lightweight Asynchronous Snapshots for Distributed Dataflows (Chandy-Lamport adaptation for Flink)
- Kafka Exactly-Once Semantics (KIP-98, KIP-129)
- The Dataflow Model (Google, 2015) -- foundational paper on windowing and watermarks
- Spark Structured Streaming: micro-batch and continuous processing
