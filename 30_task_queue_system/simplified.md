# Task Queue System -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
We need a distributed task queue that decouples producers from consumers -- producers enqueue tasks (send email, resize image, process payment) and workers asynchronously dequeue and execute them. The hard part is ensuring at-least-once delivery without loss, handling worker crashes mid-processing, and supporting priorities and delayed execution at 500K tasks/sec.

## 2. Requirements
**Functional:** Enqueue tasks with payload, optional priority, delay, and dedup key. Dequeue with visibility timeout (task invisible to others while being processed). Acknowledge on success or release for retry. Dead letter queue (DLQ) after N failures. Named queues for different task types. Batch operations.

**Non-functional:** At-least-once delivery (no task loss), enqueue p99 under 10ms, dequeue p99 under 20ms, 500K enqueue and 500K dequeue per second, replicated for durability (factor 3), 7-day retention for unprocessed tasks.

## 3. Core Concept: Visibility Timeout for At-Least-Once Delivery
The key insight is the visibility timeout mechanism. When a worker dequeues a task, the task becomes invisible to other workers for a configurable duration (e.g., 5 minutes). If the worker acknowledges within that window, the task is permanently deleted. If the worker crashes or times out, the task automatically becomes visible again and is redelivered to another worker. This simple mechanism guarantees at-least-once delivery without complex distributed transactions.

## 4. High-Level Architecture
```
Producers --> [API Gateway] --> Broker Cluster
                                 (partitioned, replicated log)
                                      |
                              [Visibility Timer]
                              [Delay Scheduler]
                              [DLQ Router]
                                      |
                              Workers (long-poll)
```
- **Broker Cluster**: stores messages in a replicated append-only log with queue semantics layered on top.
- **Visibility Timer**: background process transitioning expired IN_FLIGHT messages back to AVAILABLE.
- **Delay Scheduler**: holds delayed messages and transitions them to AVAILABLE when their delay expires.
- **DLQ Router**: moves messages exceeding max_receives to the dead letter queue for manual inspection.
- **Workers**: long-poll the broker for available messages; process and acknowledge.

## 5. How Task Processing Works
1. Producer enqueues a task with payload and optional priority/delay. Broker writes to replicated log and ACKs.
2. Worker long-polls the broker (waits up to 20 seconds for a message to arrive).
3. Broker selects the highest-priority AVAILABLE message, sets state to IN_FLIGHT, sets invisible_until = now + visibility_timeout, generates a unique receipt_handle, and returns it.
4. Worker processes the task. During processing, the message is invisible to other workers.
5. Worker calls acknowledge with the receipt_handle; message is permanently deleted.
6. If no ACK within the visibility timeout, the Visibility Timer resets the message to AVAILABLE for redelivery.
7. After max_receives failed attempts, the DLQ Router moves the message to the dead letter queue.

## 6. What Happens When Things Fail?
- **Worker crashes mid-processing**: Visibility timeout expires, message automatically becomes available for another worker. No task is lost.
- **Poison message (always fails)**: After max_receives attempts (default 3), routed to the DLQ. Can be inspected and replayed manually.
- **Broker node failure**: Partition replicas on other brokers take over. Consumer polling discovers the new partition leader automatically.

## 7. Scaling
- **10x**: Scale to 500 partitions across 100 broker nodes. Batch 100 messages per request to reduce RPC overhead by 100x. Tiered storage: recent messages on SSD, older on HDD.
- **100x**: Regional broker clusters for geo-distributed producers and consumers. Zero-copy delivery via sendfile/io_uring. Dedicated clusters per high-volume queue to prevent noisy-neighbor effects. Compress message batches with LZ4.

## 8. Key Trade-offs to Mention
| Decision | Trade-off |
|----------|-----------|
| Pull/long-poll over push delivery | Workers control consumption rate with natural backpressure, but slightly higher latency |
| Visibility timeout over explicit locking | Simpler, self-healing on failure, but can cause duplicate processing if timeout is too short |
| Log-based storage with queue semantics | High throughput and durability from the log, with per-message ACK and DLQ from queue semantics |

## 9. Closing (30s)
> We designed a distributed task queue with at-least-once delivery using visibility timeouts. Messages are stored in a replicated, partitioned log with queue semantics layered on top -- visibility management, per-message acknowledgment, priority levels, and dead letter routing. Workers long-poll for tasks, process them, and acknowledge. Failed or timed-out tasks are automatically retried up to a configurable limit before routing to the DLQ. The system scales horizontally by adding broker partitions and workers, with batching for throughput and tiered storage for retention.
