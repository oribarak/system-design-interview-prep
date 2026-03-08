# Stream Processing System -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
We need to build a real-time stream processing platform that ingests millions of events per second, applies transformations and windowed aggregations, and outputs results with exactly-once guarantees. The hard part is maintaining correctness across failures while keeping latency under one second.

## 2. Requirements
**Functional:** Publish events to partitioned topics. Apply stateless transforms (map, filter) and stateful operations (windowed aggregations, stream-stream joins). Support event replay by resetting offsets. Guarantee exactly-once output semantics.

**Non-functional:** Process 2M+ events/sec. End-to-end p99 latency under 1 second for stateless operations. Recover from worker failures within 30 seconds. Maintain per-partition ordering. 7-day event retention for replay.

## 3. Core Concept: Chandy-Lamport Checkpoint Barriers
The system injects special barrier markers into the event stream. As a barrier flows through the processing DAG, each operator snapshots its state when barriers from all inputs arrive. The checkpoint captures a consistent cut: all events before the barrier are reflected in state, none after. On failure, the system restores the last checkpoint and replays from those offsets -- events are re-processed but produce identical results because state is also restored.

## 4. High-Level Architecture
```
Producers --> Broker Cluster (partitioned log) --> Task Managers (operators + RocksDB state)
                                                          |
                                                   Checkpoint Coordinator
                                                          |
                                                   Object Storage (S3)
                                                          |
                                                   Output Sinks
```
- **Broker Cluster**: Distributed commit log with partitioned, replicated topics.
- **Job Manager**: Schedules operator DAGs across Task Managers, coordinates checkpoints.
- **Task Managers**: Execute operators, maintain local RocksDB state, consume/produce events.
- **Checkpoint Coordinator**: Injects barriers, tracks completion, stores snapshots in S3.

## 5. How a Windowed Aggregation Works
1. Producer publishes click events to a "clicks" topic, partitioned by user ID.
2. Task Manager consumes assigned partitions and applies a filter operator.
3. Events are keyed by user ID and assigned to a 1-minute tumbling window.
4. Each window's count is maintained in RocksDB on the local SSD.
5. A watermark tracks event-time progress; when it passes a window's end time, the window fires.
6. The aggregated result is produced to an output topic.
7. Late events arriving within the allowed-lateness window trigger an updated result.

## 6. What Happens When Things Fail?
- **Task Manager crash**: Job Manager detects via heartbeat timeout. Reassigns partitions to healthy workers, downloads state from S3, replays from checkpointed offsets. Recovery within 30 seconds.
- **Broker failure**: Replicated partitions (RF=3) with ISR. Leader election promotes an in-sync replica. Producers retry with no data loss when using acks=all.
- **Poison pill events**: Malformed events that cause exceptions are routed to a dead-letter topic for investigation rather than retried infinitely.

## 7. Scaling
- **10x**: Increase topic partitions from 128 to 1024 and add proportional Task Managers. Use tiered storage (hot SSD + warm S3) for the broker log. Incremental checkpoints become critical at this scale.
- **100x**: Multi-cluster federation with cross-region replication. Disaggregated storage where brokers become stateless compute. Edge pre-aggregation pushes simple counts to edge nodes before the central cluster.

## 8. Key Trade-offs to Mention
| Decision | Trade-off |
|---|---|
| Event-at-a-time processing (Flink-style) | Lower latency than micro-batch, but more complex state management |
| Embedded RocksDB for state | Co-located (no network hop) with incremental checkpoints, but tied to local disk and harder to inspect |
| Checkpoint barriers for exactly-once | Consistent distributed snapshots without stopping the pipeline, but barrier alignment can cause brief backpressure spikes |

## 9. Closing (30s)
> We designed a stream processing platform handling 2M events/sec with exactly-once semantics. Events flow through a partitioned, replicated broker into Task Managers running operator DAGs with RocksDB-backed state. Chandy-Lamport checkpoint barriers create consistent snapshots stored in S3, enabling recovery within 30 seconds. Windowed aggregations use watermarks to handle out-of-order events. The system scales by adding partitions and workers, and uses Kafka transactions for atomic offset and output commits.
