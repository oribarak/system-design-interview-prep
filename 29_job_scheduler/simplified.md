# Job Scheduler (Cron at Scale) -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
We need a distributed job scheduler that reliably triggers millions of scheduled jobs -- essentially Cron at scale. The hard part is ensuring no triggers are missed even during scheduler failures, handling the "minute boundary" burst where thousands of cron jobs fire simultaneously, and preventing double-triggers when scheduler partition ownership changes.

## 2. Requirements
**Functional:** Register jobs with cron expressions, fixed intervals, or one-time schedules with timezone support. Trigger jobs at their scheduled time and dispatch to workers. Support retries with configurable backoff, job dependencies, and lifecycle management (pause, resume, delete). Record execution history.

**Non-functional:** Jobs fire within 5 seconds of scheduled time. No missed triggers even during scheduler failures. At-least-once execution (combined with idempotent handlers for exactly-once). Handle 10M registered jobs and 50K triggers/sec at peak. Multi-tenant isolation.

## 3. Core Concept: Partitioned Scanning with Atomic Compare-and-Swap
The key insight is partitioning the job space across scheduler nodes, where each node scans its partition for due triggers using a pre-computed next_trigger_at index. To prevent double-triggers during partition reassignment, each trigger uses an atomic compare-and-swap: UPDATE jobs SET next_trigger_at = :next WHERE job_id = :id AND next_trigger_at = :expected. Only one scanner can succeed, and the triggered job is dispatched to a durable task queue.

## 4. High-Level Architecture
```
API --> [Job Store (Postgres)] <-- Scheduler Nodes (10-20)
                                        |
                              [Next-Trigger Cache (Redis)]
                                        |
                                [Trigger Queue (Kafka)]
                                        |
                                Worker Fleet (500+)
                                        |
                                [Result Processor]
```
- **Scheduler Nodes**: each owns a partition of jobs and scans for due triggers every 1 second.
- **Job Store (Postgres)**: persistent storage for job definitions and execution history.
- **Next-Trigger Cache (Redis)**: sorted set of (next_trigger_at, job_id) for efficient scanning.
- **Trigger Queue (Kafka)**: durable buffer between scheduler and workers, absorbs burst traffic.
- **Worker Fleet**: auto-scaled based on queue depth, pulls and executes jobs.
- **Result Processor**: updates execution records and triggers dependent jobs.

## 5. How a Job Gets Triggered
1. When a job is created, the cron parser computes next_trigger_at and stores it (indexed in Redis sorted set).
2. Every 1 second, each scheduler node scans its partition for jobs where next_trigger_at <= now().
3. For each due job, the scheduler atomically sets next_trigger_at to the next occurrence (compare-and-swap to prevent double-triggers).
4. An execution record is created with status QUEUED and published to the Kafka trigger queue.
5. A worker dequeues the execution, invokes the job handler, and reports success/failure.
6. On failure, the result processor checks retry policy and re-enqueues with exponential backoff delay.
7. A background reaper detects executions stuck in RUNNING state beyond their timeout and retries or marks as failed.

## 6. What Happens When Things Fail?
- **Scheduler node failure**: Partition ownership is reassigned via coordination service within 10-30 seconds. On startup, the new owner scans for overdue jobs (next_trigger_at < now) and fires them based on the misfire policy (fire immediately, skip and reschedule, or fire all missed).
- **Worker crashes mid-execution**: Execution stays in RUNNING state. The reaper detects the timeout and retries on a different worker.
- **Kafka unavailable**: Scheduler buffers triggers locally and retries. Database remains the source of truth.

## 7. Scaling
- **10x**: 1,000 partitions across 100 scheduler nodes. Redis sorted set per partition for sub-millisecond scanning. Scale Kafka partitions and auto-scale workers from 500 to 5,000 based on queue depth.
- **100x**: Hierarchical scheduling -- per-tenant scheduler instances for large tenants. Pre-compute all triggers for the next hour in a dedicated pending triggers table. Fast-path for lightweight jobs (under 1 second) that execute inline on the scheduler node.

## 8. Key Trade-offs to Mention
| Decision | Trade-off |
|----------|-----------|
| Pull-based workers (from queue) over push | Natural load balancing and easy to scale, but slightly higher latency than push |
| Redis sorted set + Postgres (hybrid) | Fast scanning via Redis with Postgres as durable source of truth, but two systems to keep in sync |
| At-least-once triggering (not exactly-once) | Simpler and more reliable, but job handlers must be idempotent |

## 9. Closing (30s)
> We designed a distributed job scheduler managing 10M jobs with reliable at-least-once trigger semantics. Partitioned scheduler nodes scan a Redis-cached time index for due triggers, use atomic compare-and-swap to prevent double-triggers, and dispatch through a Kafka queue to an auto-scaled worker fleet. Minute-boundary bursts are smoothed with pre-computation and jitter. Missed triggers are recovered on scheduler restart, and stuck executions are detected by a timeout reaper. The system scales by independently adding scheduler partitions, queue partitions, and workers.
