# System Design Interview: Task Queue System — "Perfect Answer" Playbook

## 0) Opening: Frame the Problem (30-60s)

> "We need to design a distributed task queue system — a general-purpose infrastructure for decoupling work producers from consumers. Think Celery, Sidekiq, or AWS SQS plus workers. Producers enqueue tasks (send email, resize image, process payment), and workers asynchronously dequeue and execute them. I will cover the queue semantics (at-least-once delivery, visibility timeout, dead letter queue), the broker architecture, priority and delay support, and how we handle worker failures and task retries. The key challenges are ensuring reliable delivery without loss, maintaining ordering where needed, and scaling to millions of tasks per second."

## 1) Clarifying Questions (2-5 min)

**Scope**
- Is this a multi-tenant SaaS task queue (like SQS) or an internal infrastructure component?
- Do we need strict FIFO ordering, or is best-effort ordering acceptable?
- Should the system support task priorities, delayed execution, and task deduplication?

**Scale**
- How many tasks per second? Assume 500,000 enqueue operations/sec and 500,000 dequeue operations/sec.
- What is the average task payload size? Assume 1 KB.
- How many concurrent workers? Assume 10,000 workers across multiple services.

**Policy / Constraints**
- What delivery guarantee: at-most-once, at-least-once, or exactly-once?
- How long should unprocessed tasks be retained? Assume 7 days.
- Should we support batch operations (enqueue/dequeue multiple tasks at once)?

## 2) Back-of-Envelope Estimation (3-5 min)

| Metric | Estimate |
|--------|----------|
| Enqueue rate | 500,000 tasks/sec |
| Dequeue rate | 500,000 tasks/sec |
| Average payload size | 1 KB |
| Ingestion bandwidth | 500K x 1KB = 500 MB/s |
| Daily task volume | 500K x 86,400 = ~43 billion tasks |
| Average queue depth (tasks in flight) | 5 million |
| Storage for 7-day retention | 43B x 1KB x 7 = ~300 TB (before compaction) |
| Workers | 10,000 concurrent |
| Visibility timeout | 300 seconds (5 minutes) |
| Dead letter queue threshold | 3 failed attempts |

## 3) Requirements (3-5 min)

### Functional
- Enqueue tasks with a payload, optional priority, optional delay, and optional deduplication key.
- Dequeue tasks: deliver a task to exactly one worker, with a visibility timeout during which the task is invisible to other workers.
- Acknowledge (delete) a task after successful processing, or release it back for retry.
- Dead letter queue: after N failed processing attempts, move the task to a DLQ for manual inspection.
- Support named queues for different task types and routing.
- Support batch enqueue and batch dequeue for efficiency.

### Non-functional
- Delivery guarantee: at-least-once (no task loss). Combined with idempotent consumers for effective exactly-once.
- Throughput: 500K enqueue/sec and 500K dequeue/sec.
- Latency: enqueue < 10 ms (p99), dequeue < 20 ms (p99).
- Durability: enqueued tasks survive broker restarts and node failures (replication factor 3).
- Availability: the system remains operational during minority node failures.
- Retention: unprocessed tasks retained for 7 days before auto-deletion.

## 4) API Design (2-4 min)

```
POST /queues/{queue_name}/messages
  Body: {messages: [{payload: {...}, priority: 5, delay_seconds: 60,
         dedup_key: "order-123-email", group_id: "customer-456"}]}
  Returns: {message_ids: ["m-abc123"]}

POST /queues/{queue_name}/receive
  Body: {max_messages: 10, visibility_timeout_seconds: 300, wait_time_seconds: 20}
  Returns: {messages: [{message_id: "m-abc123", receipt_handle: "rh-xyz", payload: {...},
            approximate_receive_count: 1, enqueued_at: 1700000000}]}

POST /queues/{queue_name}/acknowledge
  Body: {receipt_handles: ["rh-xyz"]}
  Returns: {acknowledged: 1}

POST /queues/{queue_name}/release
  Body: {receipt_handle: "rh-xyz", delay_seconds: 30}
  Returns: {released: true}

POST /queues/{queue_name}/change-visibility
  Body: {receipt_handle: "rh-xyz", visibility_timeout_seconds: 600}
  Returns: {new_timeout: 1700000600}

GET /queues/{queue_name}/stats
  Returns: {approximate_messages: 500000, approximate_in_flight: 12000,
            approximate_delayed: 3000, dlq_messages: 150}
```

## 5) Data Model (3-5 min)

### Message Record

| Field | Type | Description |
|-------|------|-------------|
| message_id | UUID (PK) | Globally unique message identifier |
| queue_name | string | Target queue |
| payload | bytes | Task data (up to 256 KB) |
| priority | int | Higher number = higher priority (default 0) |
| state | enum | AVAILABLE, IN_FLIGHT, DELAYED, DLQ |
| enqueued_at | int64 | When the message was enqueued |
| visible_at | int64 | When the message becomes visible (for delayed or retried messages) |
| invisible_until | int64 | Visibility timeout deadline (set on dequeue) |
| receive_count | int | Number of times this message has been received |
| receipt_handle | string | Unique handle for the current receive (changes each time) |
| dedup_key | string | Optional deduplication key |
| group_id | string | Optional message group for FIFO ordering |
| max_receives | int | Maximum receives before DLQ (default 3) |

### Queue Configuration

| Field | Type | Description |
|-------|------|-------------|
| queue_name | string (PK) | Unique queue identifier |
| visibility_timeout | int | Default visibility timeout in seconds |
| retention_days | int | How long to keep messages |
| max_message_size | int | Maximum payload size |
| dlq_queue_name | string | Dead letter queue name |
| dlq_max_receives | int | Receives before sending to DLQ |
| fifo | boolean | Whether this is a FIFO queue |

**Storage**: Messages stored in a log-structured store (like Kafka topics) for high throughput, or in a database (Postgres with partitioning) for lower scale with richer querying. At our scale, a custom log-based store is necessary.

**Sharding**: queues are sharded by queue_name. Within a queue, messages are partitioned across broker nodes for parallel processing.

## 6) High-Level Architecture (5-8 min)

**Dataflow**: Producers send messages to the queue service via the API. The message is written to a replicated log on the broker and acknowledged. Workers long-poll the broker for available messages. The broker selects the highest-priority available message, marks it as IN_FLIGHT with a visibility timeout, and delivers it. On success, the worker acknowledges and the message is deleted. On timeout, the message becomes available again.

**Components**:
- **API Gateway**: routes enqueue/dequeue requests, handles authentication and rate limiting.
- **Broker Cluster**: the core message store. Each broker handles a set of queue partitions. Replicates messages to peers.
- **Message Store**: append-only log per queue partition. Supports fast sequential writes and indexed reads by visibility time.
- **Visibility Timer**: background process that checks for IN_FLIGHT messages past their timeout and transitions them back to AVAILABLE.
- **Delay Scheduler**: manages delayed messages, transitioning them to AVAILABLE when their delay expires.
- **DLQ Router**: moves messages exceeding max_receives to the dead letter queue.
- **Partition Manager**: assigns queue partitions to brokers, rebalances on broker join/leave.
- **Consumer Group Coordinator**: tracks which consumer in a group is processing which partition (for FIFO queues).
- **Metrics Collector**: tracks queue depths, throughput, latency, DLQ rates.

**One-sentence summary**: A partitioned broker cluster stores messages in a replicated append-only log, delivers them to workers via long-polling with visibility timeouts for at-least-once processing, and routes failed messages to dead letter queues.

## 7) Deep Dive #1: Visibility Timeout and At-Least-Once Delivery (8-12 min)

### How Visibility Timeout Works

The visibility timeout is the core mechanism enabling at-least-once delivery:

1. **Enqueue**: message is written to the log with state=AVAILABLE.
2. **Dequeue**: worker requests a message. The broker selects an AVAILABLE message, sets state=IN_FLIGHT, sets `invisible_until = now + visibility_timeout`, generates a unique `receipt_handle`, and returns the message.
3. **Processing**: worker processes the task. The message is invisible to other workers during this time.
4. **Acknowledge**: worker calls acknowledge with the receipt_handle. Message is permanently deleted (or marked DELETED).
5. **Timeout**: if the worker does not acknowledge within the timeout, the Visibility Timer sets state=AVAILABLE and the message is redelivered to another worker.

### Why "At-Least-Once" and Not "Exactly-Once"

Consider: Worker A receives a message, processes it successfully, but the acknowledge request fails due to a network error. The visibility timeout expires, and Worker B receives the same message. Both workers processed it. This is inherent to any distributed queue without two-phase commit.

**Mitigation**: consumers must be idempotent. Use the `message_id` or a domain-specific idempotency key to detect and skip duplicate processing.

### Visibility Timeout Tuning

- Too short: task takes longer than expected, timeout fires, message is redelivered while still being processed. Results in duplicate processing.
- Too long: if a worker crashes, the message is stuck IN_FLIGHT for the entire timeout before becoming available.
- **Best practice**: set timeout to 2-3x the expected processing time. Workers should call `change-visibility` to extend the timeout if processing takes longer than expected.

### Implementation: Efficient Timeout Tracking

With millions of IN_FLIGHT messages, we need an efficient way to find expired ones:

- **Sorted index on invisible_until**: the Visibility Timer queries `SELECT * FROM messages WHERE state=IN_FLIGHT AND invisible_until < now()`. With a B-tree index, this is efficient.
- **Timer wheel**: in-memory data structure that buckets timeouts by time. Each bucket represents a 1-second window. A pointer advances every second, processing expired entries. O(1) insertion and expiration.
- **Hybrid**: timer wheel for recent timeouts (< 10 minutes), overflow to a sorted index for longer timeouts.

### Deduplication

For producers that might send the same message twice (e.g., during retry):
- Maintain a deduplication cache: `dedup_key -> message_id` with a TTL of 5 minutes.
- On enqueue, check if the dedup_key already exists. If so, return the existing message_id without re-enqueuing.
- Trade-off: deduplication window must be bounded to prevent unbounded memory growth.

## 8) Deep Dive #2: Priority Queues and FIFO Ordering (5-8 min)

### Priority Support

Tasks with different priorities should be processed in priority order:

**Approach 1: Multiple physical queues**
- Create a separate internal queue per priority level (e.g., P0-critical, P1-high, P2-normal, P3-low).
- On dequeue, the broker checks queues in priority order, returning the first available message from the highest-priority non-empty queue.
- **Pros**: simple, efficient, no sorting.
- **Cons**: lower-priority tasks may starve if higher-priority tasks are always available.

**Approach 2: Priority heap per partition**
- Maintain a min-heap (or priority queue) per partition, ordered by priority.
- On dequeue, extract the highest-priority message.
- **Pros**: no starvation with weighted fair queuing.
- **Cons**: heap operations are O(log N) per dequeue, more complex.

**Recommendation**: multiple physical queues with weighted round-robin. For every 10 dequeue operations, serve 5 from P0, 3 from P1, 2 from P2. This prevents starvation while maintaining priority ordering.

### FIFO Ordering

For use cases requiring strict ordering (e.g., process all events for user X in order):

- Messages with the same `group_id` are always delivered to the same consumer and in enqueue order.
- Only one message per group_id can be IN_FLIGHT at a time. The next message in the group is not delivered until the current one is acknowledged.
- **Implementation**: each group_id maps to a partition (via hashing). Within a partition, a pointer tracks the last delivered offset per group_id.

**FIFO trade-offs**:
- Lower throughput (messages within a group are serialized).
- Higher latency (waiting for acknowledgment before delivering the next).
- Head-of-line blocking: if one message in a group is slow, it blocks all subsequent messages in that group.

## 9) Trade-offs and Alternatives (3-5 min)

### Push (Broker Pushes to Workers) vs. Pull (Workers Poll)
- **Pull/Long-poll (our design)**: workers control their consumption rate, natural backpressure, no need for worker registration. Slightly higher latency due to polling interval.
- **Push**: lower latency, but the broker must track worker capacity and handle slow consumers. Risk of overwhelming workers.

### Log-Based (Kafka) vs. Queue-Based (SQS/RabbitMQ)
- **Log-based**: messages are retained after consumption (replay possible), high throughput, but no per-message acknowledgment or visibility timeout. Best for streaming.
- **Queue-based (our design)**: per-message visibility and acknowledgment, dead letter queues, priorities. Better for task processing where each task is processed once.
- **Hybrid**: our system uses a log for storage (durability, throughput) with queue semantics (visibility, ack) layered on top.

### In-Memory (Redis) vs. Persistent (Disk)
- **Redis (e.g., Sidekiq)**: very fast, low latency, but limited by memory. Acceptable for moderate scale (tens of thousands of tasks/sec).
- **Persistent (our design)**: disk-based with memory caching. Handles millions of tasks/sec and multi-day retention.

### Embedded vs. Standalone
- **Embedded (in-process queue)**: no network hop, simplest possible, but limited to single-process workloads.
- **Standalone (our design)**: decoupled, horizontally scalable, multi-consumer support.

## 10) Scaling: 10x and 100x (3-5 min)

### At 10x (5M tasks/sec)
- **More broker nodes and partitions**: scale from 50 to 500 partitions across 100 broker nodes.
- **Batch operations**: producers and consumers batch 100 messages per request, reducing RPC overhead by 100x.
- **Tiered storage**: recent messages on SSD, older messages (for retention) on HDD or object storage.
- **Consumer auto-scaling**: scale workers based on queue depth. If depth > threshold for 2 minutes, add workers.

### At 100x (50M tasks/sec)
- **Regional broker clusters**: geo-distributed clusters with local producers and consumers. Cross-region replication for disaster recovery only, not for live traffic.
- **Zero-copy delivery**: use sendfile/io_uring for transferring message payloads directly from disk to network without copying through user space.
- **Dedicated clusters per high-volume queue**: isolate heavy queues (e.g., notification sending) on dedicated broker clusters to prevent noisy-neighbor effects.
- **Compression**: compress message batches (LZ4) during replication and storage. 3-5x space savings for text-heavy payloads.

## 11) Reliability and Fault Tolerance (3-5 min)

- **Replication**: every message is written to 3 broker nodes before being acknowledged to the producer. Uses a quorum write (2 of 3 must acknowledge).
- **Broker failure**: partition replicas on other brokers take over. Consumer polling automatically discovers the new partition leader.
- **Message durability**: messages are fsynced to disk before acknowledgment (configurable: fsync per message vs. batched fsync for higher throughput).
- **Consumer failure**: visibility timeout ensures unacknowledged messages are redelivered. No message is lost.
- **Poison messages**: messages that repeatedly fail processing (infinite retry loop) are moved to the DLQ after max_receives attempts. The DLQ can be inspected and replayed manually.
- **Idempotency**: producer dedup_key prevents duplicate enqueues. Consumer idempotency (via message_id) prevents duplicate processing.

## 12) Observability and Operations (2-4 min)

- **Metrics**: enqueue rate, dequeue rate, queue depth (available, in-flight, delayed, DLQ), end-to-end latency (enqueue to completion), visibility timeout expirations/sec, acknowledgment rate.
- **Dashboards**: per-queue health, consumer lag (how far behind consumers are), broker partition distribution, DLQ depth trending.
- **Alerts**: queue depth growing for > 5 minutes (consumers cannot keep up), DLQ depth > 1000 (systematic processing failures), consumer lag > 1 hour, broker disk usage > 80%.
- **Operational tools**: replay DLQ messages to the main queue, purge a queue, move messages between queues, inspect message payload.

## 13) Security (1-3 min)

- **Authentication**: API keys or IAM-style policies for producers and consumers.
- **Authorization**: per-queue ACLs (who can enqueue, who can dequeue). Multi-tenant queues require tenant isolation.
- **Encryption**: TLS for all network communication. Encryption at rest for stored messages (AES-256).
- **Payload sanitization**: enforce maximum payload size (256 KB). Reject payloads exceeding the limit.
- **Audit**: log all administrative operations (queue creation, purge, DLQ replay).

## 14) Team and Operational Considerations (1-2 min)

- **Team**: broker team (storage, replication, partition management), API team (producer/consumer SDKs, rate limiting), operations team (capacity planning, monitoring).
- **SDK quality**: producer and consumer SDKs must handle retries, connection pooling, batch optimization, and graceful shutdown (finish in-flight tasks before stopping).
- **Capacity planning**: monitor queue depth trends and consumer throughput to proactively scale before queues back up.

## 15) Common Follow-up Questions

1. **How do you implement delayed messages?** Store delayed messages with `visible_at = now + delay_seconds` and state=DELAYED. The Delay Scheduler transitions them to AVAILABLE when `visible_at <= now`. Uses a timer wheel or sorted index.
2. **How do you handle message ordering across partitions?** Strict global ordering is not supported (too expensive). Use `group_id` for per-group ordering within a partition. For global ordering, use a single partition (sacrifices throughput).
3. **How is this different from Kafka?** Kafka is a log-based system: messages are retained and consumers track offsets. Our system is a queue: messages are individually acknowledged and deleted. Kafka excels at streaming; our system excels at task processing.
4. **How do you handle back pressure?** If the queue depth exceeds a threshold, return HTTP 429 to producers. Consumers naturally apply backpressure by reducing their polling rate.
5. **Can you support scheduled/recurring tasks?** The task queue handles one-time deferred tasks via delay_seconds. For recurring tasks, integrate with the Job Scheduler (topic 29), which triggers jobs and enqueues them into this task queue.

## 16) Closing Summary (30-60s)

> "We designed a distributed task queue with at-least-once delivery semantics using visibility timeouts. Messages are stored in a replicated, partitioned log with queue semantics layered on top — visibility management, per-message acknowledgment, priority levels, and dead letter routing. Producers enqueue tasks with optional priority and delay; workers long-poll for available tasks, process them, and acknowledge. Failed or timed-out tasks are automatically retried up to a configurable limit before being routed to a dead letter queue. The system scales horizontally by adding broker partitions and worker nodes, batching operations for throughput, and using tiered storage for multi-day retention."

## Appendix A: Configuration / Tuning Knobs

| Knob | Default | Effect |
|------|---------|--------|
| `visibility_timeout` | 300s | Time a dequeued message is invisible to other consumers |
| `max_receives` | 3 | Attempts before routing to DLQ |
| `retention_days` | 7 | How long unprocessed messages are kept |
| `max_message_size` | 256 KB | Maximum payload size |
| `long_poll_wait` | 20s | Maximum wait time for long-poll dequeue |
| `batch_size` | 10 | Maximum messages per batch receive |
| `dedup_window` | 5 min | Deduplication cache TTL |
| `fsync_mode` | batched | Per-message fsync vs. batched (throughput vs. durability) |
| `partition_count` | 50 | Number of partitions per queue |
| `replication_factor` | 3 | Number of copies per message |

## Appendix B: Vocabulary Cheat Sheet

| Term | Definition |
|------|-----------|
| **Visibility Timeout** | Period after dequeue during which a message is invisible to other consumers, allowing processing time. |
| **Receipt Handle** | A unique token for the current delivery of a message, used to acknowledge or extend visibility. |
| **Dead Letter Queue (DLQ)** | A secondary queue that receives messages which failed processing after max_receives attempts. |
| **Long Polling** | Consumer blocks up to N seconds waiting for a message, reducing empty poll overhead. |
| **At-Least-Once** | Delivery guarantee: every message is delivered at least once, but possibly more than once. |
| **Backpressure** | Mechanism to slow down producers when the queue is overloaded, preventing unbounded growth. |
| **FIFO Queue** | Queue where messages with the same group_id are delivered strictly in order, one at a time. |
| **Poison Message** | A message that consistently fails processing, causing an infinite retry loop if not routed to DLQ. |

## Appendix C: References

- Amazon SQS architecture and design principles
- RabbitMQ internals: channels, exchanges, and queues
- Sidekiq: Redis-backed task processing for Ruby
- Celery: distributed task queue for Python
- "Building a Distributed Task Queue" — Lyft Engineering
