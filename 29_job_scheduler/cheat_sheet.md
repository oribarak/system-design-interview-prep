# Job Scheduler (Cron at Scale) — Cheat Sheet

## Key Numbers
- Registered jobs: 10 million
- Peak triggers/sec: 50,000 (burst at minute/hour boundaries)
- Average triggers/sec: 5,000
- Trigger accuracy: < 5 seconds delay
- Job metadata: ~5 GB total; execution records: ~86 GB/day
- Scheduler nodes: 10-20; Workers: 500+ (auto-scaled)

## Core Components (one line each)
- **Scheduler Engine (partitioned)**: scans Redis trigger cache for due jobs on a 1-second tick
- **Trigger Cache (Redis)**: sorted set of (next_trigger_at, job_id) per partition for fast scanning
- **Trigger Queue (Kafka)**: durable buffer between scheduler and workers, absorbs burst traffic
- **Worker Fleet**: auto-scaled pool that dequeues and executes job handlers
- **Job Store (Postgres)**: source of truth for job definitions and execution history
- **Reaper**: background process detecting stuck/timed-out executions for retry
- **Result Processor**: updates execution status and triggers dependent jobs

## Architecture in One Sentence
A partitioned scheduler scans Redis-cached trigger times every second, dispatches due jobs through Kafka to auto-scaled workers, and uses atomic CAS to prevent double-triggers with configurable retry and misfire policies.

## Top 3 Trade-offs
1. Database scanning (simple, slow at scale) vs. Redis trigger cache (fast, extra component to maintain)
2. At-least-once triggering with idempotent handlers (practical) vs. exactly-once (complex, distributed transactions)
3. Jitter for burst smoothing (reduces peak load) vs. precise timing (simpler but creates thundering herd)

## Scaling Story
- **10x**: more scheduler partitions (1,000), Kafka partitions, auto-scale workers to 5,000, shard Postgres
- **100x**: hierarchical scheduling, pre-computed trigger tables, inline execution for lightweight jobs, spot instances for workers

## Closing Statement
A partitioned, tick-based scheduler using Redis for trigger scanning, Kafka for durable dispatch, atomic CAS for deduplication, and a reaper for stuck execution recovery.
