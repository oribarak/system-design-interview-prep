# Real-Time Stream Processing -- Cheat Sheet

## Key Numbers
- 2M events/sec, 500 bytes/event = 1 GB/s ingestion
- 128 partitions per topic, 20K total partitions
- 7-day retention = ~605 TB (x3 replication = 1.8 PB)
- Checkpoint interval: 30s, incremental size: ~50 MB/job
- Stateful job state: up to 500 GB per job (RocksDB)
- Recovery time: <30 seconds from last checkpoint

## Core Components
- **Broker Cluster**: Partitioned, replicated commit log (Kafka-style)
- **Job Manager**: Coordinates DAG scheduling, checkpoints, failure recovery
- **Task Managers**: Execute operators, manage RocksDB state, consume/produce
- **Checkpoint Coordinator**: Injects barriers for Chandy-Lamport snapshots
- **Object Storage (S3)**: Durable checkpoint and savepoint storage
- **Schema Registry**: Validates event schemas, enforces compatibility

## Architecture in One Sentence
Producers write events to a partitioned replicated broker log; Task Managers consume assigned partitions, execute a DAG of operators with RocksDB-backed state, and achieve exactly-once via Chandy-Lamport checkpoint barriers combined with Kafka transactions.

## Top 3 Trade-offs
1. **Event-at-a-time vs. micro-batch**: Lower latency but more per-record overhead
2. **Embedded RocksDB vs. external state store**: No network hop but state is local (needs checkpointing)
3. **Aligned vs. unaligned checkpoints**: Aligned is simpler and smaller; unaligned reduces latency spikes

## Scaling Story
- 10x: Increase partitions to 1024, scale Task Managers proportionally, enable tiered broker storage
- 100x: Multi-cluster federation, disaggregated storage (stateless brokers + S3), adaptive auto-scaling

## Closing Statement
A distributed stream processing platform with exactly-once semantics via Chandy-Lamport checkpoint barriers, RocksDB-backed stateful operators, watermark-driven windowing for out-of-order events, and horizontal scaling through partition-level parallelism -- handling 2M+ events/sec with sub-second latency.
