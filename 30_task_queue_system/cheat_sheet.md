# Task Queue System — Cheat Sheet

## Key Numbers
- Enqueue/dequeue rate: 500K tasks/sec each
- Average payload: 1 KB; max: 256 KB
- Queue depth: ~5 million in-flight at peak
- Visibility timeout: 300 seconds (5 min default)
- DLQ threshold: 3 failed attempts
- Retention: 7 days for unprocessed messages
- Workers: 10,000 concurrent

## Core Components (one line each)
- **Broker Cluster**: partitioned message store with replicated append-only logs
- **Visibility Timer**: scans for IN_FLIGHT messages past their timeout and reactivates them
- **Delay Scheduler**: activates delayed messages when their visible_at time arrives
- **DLQ Router**: moves messages exceeding max_receives to the dead letter queue
- **Partition Manager**: assigns and rebalances queue partitions across brokers
- **Consumer SDKs**: handle long-polling, batching, ack, and graceful shutdown
- **Dedup Cache**: prevents duplicate enqueues within a 5-minute window

## Architecture in One Sentence
A partitioned broker cluster stores messages in replicated logs with queue semantics (visibility timeout, per-message ack, priorities, DLQ), delivering tasks to long-polling workers with at-least-once guarantees.

## Top 3 Trade-offs
1. At-least-once (simple, requires idempotent consumers) vs. exactly-once (complex, distributed transactions)
2. Long-polling (simple, slight latency) vs. push delivery (low latency, complex consumer management)
3. Queue-based (per-message ack, good for tasks) vs. log-based like Kafka (replay, good for streaming)

## Scaling Story
- **10x**: more partitions and brokers, batch operations (100/request), tiered storage, consumer auto-scaling
- **100x**: regional clusters, zero-copy delivery, dedicated clusters per high-volume queue, LZ4 compression

## Closing Statement
A distributed task queue using visibility timeouts for at-least-once delivery, priority queues for task ordering, dead letter routing for poison messages, and partitioned brokers for horizontal scale.
