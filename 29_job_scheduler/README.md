# System Design Interview: Job Scheduler (Cron at Scale) — "Perfect Answer" Playbook

## 0) Opening: Frame the Problem (30-60s)

> "We need to design a distributed job scheduler — essentially Cron at scale — that reliably triggers millions of scheduled jobs across a fleet of workers. Unlike single-machine Cron, we need exactly-once execution semantics, fault tolerance when workers die mid-job, support for complex schedules (cron expressions, intervals, one-time), and horizontal scalability. I will cover the scheduling engine, the job assignment and execution model, how we handle missed triggers and retries, and the persistence layer that ensures no scheduled job is lost. The key challenges are ensuring at-least-once (ideally exactly-once) triggering, distributing work evenly, and handling time-zone-aware scheduling at scale."

## 1) Clarifying Questions (2-5 min)

**Scope**
- Is this a generic job scheduler (like Airflow/Quartz) or a cron-replacement for periodic tasks?
- Do we need DAG-based workflows (job dependencies), or independent scheduled jobs?
- Should the scheduler execute jobs directly or dispatch them to an external task queue?

**Scale**
- How many scheduled jobs? Assume 10 million registered jobs.
- How many jobs trigger per second at peak? Assume 50,000 triggers/sec (many jobs fire on the hour/minute).
- What is the acceptable trigger latency (delay between scheduled time and actual execution start)? Assume < 5 seconds.

**Policy / Constraints**
- Must jobs execute exactly once, or is at-least-once with idempotency acceptable?
- How should we handle long-running jobs (hours)? Timeout and retry, or let them run?
- Do we need multi-tenant isolation (different users with their own job namespaces)?

## 2) Back-of-Envelope Estimation (3-5 min)

| Metric | Estimate |
|--------|----------|
| Registered jobs | 10,000,000 |
| Peak triggers per second | 50,000 (burst at minute/hour boundaries) |
| Average triggers per second | 5,000 |
| Job metadata size | ~500 bytes (schedule, handler, params, state) |
| Total job metadata storage | 10M x 500B = ~5 GB |
| Trigger records per day | 5,000/sec x 86,400s = ~430 million |
| Trigger record size | ~200 bytes |
| Trigger storage per day | 430M x 200B = ~86 GB |
| Workers needed at peak | 50,000 triggers/sec / 100 jobs per worker/sec = 500 workers |
| Scheduler nodes | 10-20 (for scanning and dispatching) |

## 3) Requirements (3-5 min)

### Functional
- Register jobs with cron expressions, fixed intervals, or one-time schedules, including timezone support.
- Trigger jobs at their scheduled time and dispatch to available workers.
- Support job dependencies (job B runs after job A completes successfully).
- Provide job lifecycle management: create, update, pause, resume, delete.
- Record execution history with status (success, failure, timeout), duration, and output.
- Support retries with configurable backoff on failure.

### Non-functional
- Trigger accuracy: jobs fire within 5 seconds of their scheduled time.
- Reliability: no missed triggers, even during scheduler or worker failures.
- At-least-once execution: every triggered job is executed at least once. Combined with idempotent job handlers for effective exactly-once.
- Scalability: handle 10M registered jobs and 50K triggers/sec at peak.
- Multi-tenancy: tenant isolation for job namespaces and resource quotas.
- Observability: dashboards for job success rates, trigger delays, and worker utilization.

## 4) API Design (2-4 min)

```
POST /jobs
  Body: {name: "daily-report", schedule: "0 2 * * *", timezone: "America/New_York",
         handler: "report_generator", params: {report_type: "daily"},
         retry: {max_attempts: 3, backoff: "exponential"}, timeout_sec: 600,
         tenant_id: "team-analytics"}
  Returns: {job_id: "j-abc123", next_trigger_at: 1700010000}

PUT /jobs/{job_id}
  Body: {schedule: "0 3 * * *", params: {report_type: "weekly"}}
  Returns: {job_id: "j-abc123", next_trigger_at: 1700013600}

POST /jobs/{job_id}/pause
  Returns: {paused: true}

POST /jobs/{job_id}/trigger
  (Manual trigger, bypassing schedule)
  Returns: {execution_id: "e-xyz789", status: "queued"}

GET /jobs/{job_id}/executions?limit=20
  Returns: {executions: [{execution_id: "e-xyz789", triggered_at: ..., started_at: ...,
            completed_at: ..., status: "success", duration_ms: 12340, worker: "w-17"}]}

DELETE /jobs/{job_id}
  Returns: 204 No Content
```

## 5) Data Model (3-5 min)

### Jobs Table (Postgres, sharded by job_id)

| Column | Type | Description |
|--------|------|-------------|
| job_id | UUID (PK) | Unique job identifier |
| tenant_id | string | Tenant/team owner |
| name | string | Human-readable job name |
| schedule | string | Cron expression or interval spec |
| timezone | string | IANA timezone for cron evaluation |
| handler | string | Job handler identifier (function/endpoint) |
| params | JSONB | Parameters passed to the handler |
| status | enum | ACTIVE, PAUSED, DELETED |
| next_trigger_at | timestamp | Pre-computed next trigger time (indexed) |
| last_trigger_at | timestamp | Last time this job was triggered |
| retry_policy | JSONB | {max_attempts, backoff_type, backoff_base} |
| timeout_sec | int | Maximum execution time |
| created_at | timestamp | Creation time |

### Executions Table (Postgres, partitioned by triggered_at)

| Column | Type | Description |
|--------|------|-------------|
| execution_id | UUID (PK) | Unique execution identifier |
| job_id | UUID (FK) | Reference to job |
| triggered_at | timestamp | When the trigger fired |
| started_at | timestamp | When a worker picked it up |
| completed_at | timestamp | When execution finished |
| status | enum | QUEUED, RUNNING, SUCCESS, FAILED, TIMEOUT, RETRYING |
| attempt | int | Attempt number (1, 2, 3...) |
| worker_id | string | Which worker executed it |
| result | JSONB | Output or error message |

**Indexes**: `(status, next_trigger_at)` on Jobs for the scheduler scan. `(job_id, triggered_at)` on Executions for history queries.

**Sharding**: Jobs table sharded by `job_id`. The scheduler scans by `next_trigger_at`, which is a range query — use a time-bucketed secondary index or a dedicated trigger queue.

## 6) High-Level Architecture (5-8 min)

**Dataflow**: The Scheduler Engine periodically scans for jobs whose `next_trigger_at` has passed, creates execution records, and enqueues them into a distributed task queue. Workers poll the queue, execute the job handler, and report results. The Scheduler Engine then computes and stores the next trigger time.

**Components**:
- **Scheduler Engine (10-20 nodes)**: scans the jobs table for due triggers, partitioned by job_id range. Each node owns a partition of jobs.
- **Trigger Queue (Kafka/SQS)**: buffer between the scheduler and workers. Provides durability and load leveling for burst traffic.
- **Worker Fleet (500+ nodes)**: pull jobs from the queue, execute handlers, report results. Auto-scaled based on queue depth.
- **Job Store (Postgres)**: persistent storage for job definitions and execution history.
- **Next-Trigger Cache (Redis)**: sorted set of (next_trigger_at, job_id) for efficient scanning. Rebuilt from Postgres on startup.
- **Cron Parser**: evaluates cron expressions with timezone awareness to compute next_trigger_at.
- **Result Processor**: receives execution results from workers, updates execution records, and triggers dependent jobs.
- **API Gateway**: serves the job management API for creating, updating, and querying jobs.

**One-sentence summary**: A partitioned scheduler engine scans a time-indexed job store for due triggers, dispatches executions through a durable task queue to an auto-scaled worker fleet, and computes the next trigger time after each execution.

## 7) Deep Dive #1: Scheduler Engine and Trigger Accuracy (8-12 min)

### The Tick-Based Scanning Approach

The scheduler runs on a configurable tick interval (e.g., every 1 second):

1. **Scan**: query for all jobs where `next_trigger_at <= now()` and `status = ACTIVE`.
2. **Trigger**: for each due job, atomically set `next_trigger_at` to the next occurrence and insert an execution record with status QUEUED.
3. **Enqueue**: push the execution to the trigger queue.

### The Partition Assignment Problem

With 10 million jobs, a single scanner cannot keep up. We partition:

- **Job partitioning**: divide jobs into P partitions (e.g., 100) by `hash(job_id) % P`.
- **Scanner assignment**: each scheduler node is assigned a set of partitions. With 10 nodes, each handles ~10 partitions (~1M jobs).
- **Partition ownership**: use the leader election service (topic 28) or a coordination service to assign partitions. If a scheduler node fails, its partitions are reassigned to surviving nodes.

### Handling the "Minute Boundary" Burst

Many cron jobs are scheduled at `* * * * *` (every minute) or `0 * * * *` (every hour). At the top of each minute, thousands of jobs trigger simultaneously.

Mitigations:
- **Pre-computation**: the scheduler pre-computes triggers up to 30 seconds ahead and stages them in the trigger queue with a "do not execute before" timestamp. This spreads the enqueue work over time.
- **Jitter**: add a small random delay (0-5 seconds) to each trigger time to smooth out spikes. Configurable per job.
- **Queue buffering**: the trigger queue (Kafka) absorbs the burst. Workers consume at their own pace.

### Missed Trigger Recovery

If a scheduler node was down during a job's trigger time:
- On startup, scan for jobs where `next_trigger_at < now()` (overdue jobs).
- For each overdue job, decide based on the job's **misfire policy**:
  - **Fire immediately**: trigger the missed execution now.
  - **Skip and reschedule**: compute the next future trigger time without firing the missed one.
  - **Fire all missed**: trigger all missed executions (useful for data pipelines).
- Default: fire immediately (most common for cron jobs).

### Avoiding Double-Triggers

Two scheduler nodes might both try to trigger the same job if partition ownership is in flux. Prevention:
- Use an atomic compare-and-swap on `next_trigger_at`: `UPDATE jobs SET next_trigger_at = :next WHERE job_id = :id AND next_trigger_at = :expected`. Only one succeeds.
- The trigger queue should also deduplicate by `(job_id, scheduled_time)`.

## 8) Deep Dive #2: Reliable Execution and Failure Handling (5-8 min)

### Worker Execution Lifecycle

1. Worker dequeues an execution from the trigger queue.
2. Worker sets execution status to RUNNING and records `started_at` and `worker_id`.
3. Worker invokes the job handler (HTTP call, function invocation, container launch).
4. On completion, worker updates status to SUCCESS or FAILED with the result.
5. If the worker crashes mid-execution, the execution remains in RUNNING state.

### Detecting Stuck Executions

A background "reaper" process scans for executions that have been in RUNNING state longer than their `timeout_sec`:
- Set status to TIMEOUT.
- If retry policy allows, create a new execution with `attempt + 1` and re-enqueue.
- After `max_attempts`, set final status to FAILED and alert.

### Retry with Backoff

Retry delays are computed as:
- **Exponential**: `min(base * 2^attempt, max_delay)`. E.g., 10s, 20s, 40s, 80s, capped at 300s.
- **Linear**: `base * attempt`. E.g., 30s, 60s, 90s.
- Retries are enqueued back into the trigger queue with a "delay until" timestamp.

### Idempotency

The scheduler guarantees at-least-once triggering. For exactly-once semantics, job handlers must be idempotent:
- Use the `execution_id` as an idempotency key when writing to databases or external systems.
- Check for existing results before performing side effects.

### Heartbeat-Based Progress Tracking

For long-running jobs (e.g., 2-hour data pipeline), a simple timeout is insufficient. Instead:
- Workers send periodic heartbeats (every 30 seconds) to the scheduler, extending the execution's deadline.
- If heartbeats stop, the reaper treats it as a failure after the heartbeat timeout.

## 9) Trade-offs and Alternatives (3-5 min)

### Pull (Workers Pull from Queue) vs. Push (Scheduler Pushes to Workers)
- **Pull (our design)**: workers consume at their own rate, natural load balancing, easy to scale workers independently.
- **Push**: lower latency (no polling), but requires the scheduler to know worker capacity and health. More complex.

### Database Scanning vs. Delay Queue
- **Database scanning**: simple, uses existing Postgres. But scanning millions of rows every second is expensive.
- **Delay queue (our hybrid)**: use a Redis sorted set or Kafka with delayed delivery as the primary trigger mechanism. Database is the source of truth, but the hot path goes through the queue. Best of both worlds.

### Single-Tenant vs. Multi-Tenant
- **Single-tenant**: simpler isolation, dedicated resources per tenant.
- **Multi-tenant (our design)**: shared infrastructure, cost-efficient, but requires tenant-aware scheduling to prevent noisy neighbors.

### Cron-Only vs. DAG Workflows
- **Cron-only**: covers 80% of use cases with minimal complexity.
- **DAG support**: enables job B to run after job A. Requires a DAG executor that tracks completion status. We support simple dependencies (job triggers on parent completion) without a full DAG engine.

## 10) Scaling: 10x and 100x (3-5 min)

### At 10x (100M jobs, 500K triggers/sec)
- **More scheduler partitions**: 1,000 partitions across 100 scheduler nodes.
- **Redis trigger cache**: sorted set per partition for sub-millisecond scanning.
- **Kafka with more partitions**: scale the trigger queue to handle 500K messages/sec.
- **Worker auto-scaling**: scale workers from 500 to 5,000 based on queue depth.
- **Shard Postgres**: shard the jobs table by tenant_id or job_id hash. Partition executions by date.

### At 100x (1B jobs, 5M triggers/sec)
- **Hierarchical scheduling**: per-tenant scheduler instances for large tenants, shared scheduler for small tenants.
- **Pre-computed trigger tables**: batch-compute all triggers for the next hour and store in a dedicated "pending triggers" table. Scheduler nodes just read sequentially.
- **Tiered execution**: fast-path for lightweight jobs (< 1s) that execute inline on the scheduler node. Heavyweight jobs go through the queue.
- **Cost**: at this scale, the trigger queue and worker fleet dominate costs. Use spot/preemptible instances for workers with retry-on-preemption.

## 11) Reliability and Fault Tolerance (3-5 min)

- **Scheduler node failure**: partition ownership is reassigned within 10-30 seconds via the coordination service. Missed triggers during the gap are recovered by scanning for overdue jobs.
- **Worker failure**: execution stays in RUNNING state. The reaper detects the timeout and retries on a different worker.
- **Queue failure**: Kafka replication (factor 3) ensures no message loss. If Kafka is unavailable, the scheduler buffers triggers locally and retries.
- **Database failure**: Postgres with synchronous replication. Read replicas for API queries. Trigger scanning continues from the Redis cache during short outages.
- **Clock synchronization**: all scheduler nodes use NTP. The trigger scan has a 1-second tolerance to account for minor clock differences.

## 12) Observability and Operations (2-4 min)

- **Metrics**: triggers/sec (actual vs. expected), trigger delay (time from scheduled to dispatched), execution success/failure rate, queue depth, worker utilization.
- **Dashboards**: per-tenant job health, overdue trigger count, scheduler partition balance, retry rate by job.
- **Alerts**: trigger delay > 30 seconds, execution failure rate > 5%, queue depth growing for > 5 minutes, scheduler partition unassigned for > 1 minute.
- **Operational tools**: manual trigger (bypass schedule), bulk pause/resume, job execution history search, dry-run mode for new schedules.

## 13) Security (1-3 min)

- **Multi-tenant isolation**: tenant_id is mandatory on every job. API authentication ensures tenants only access their own jobs.
- **Job handler authorization**: the scheduler only invokes pre-registered handlers. No arbitrary code execution.
- **Secret management**: job parameters may reference secrets (API keys, credentials) stored in a vault, never in the job definition.
- **Rate limiting**: per-tenant limits on job count and trigger frequency to prevent abuse.

## 14) Team and Operational Considerations (1-2 min)

- **Team**: scheduler core team (trigger engine, partition management), worker platform team (execution environment, auto-scaling), API team (job management, UI/dashboard).
- **Rollout**: canary scheduler nodes processing a subset of partitions. Blue-green worker deployments.
- **On-call**: page on trigger delay > 1 minute, scheduler partition orphaned, execution failure spike.

## 15) Common Follow-up Questions

1. **How do you handle timezone-aware cron?** Store the IANA timezone with the job. When computing `next_trigger_at`, evaluate the cron expression in the job's timezone, then convert to UTC for storage and comparison. Handle DST transitions correctly (e.g., a 2:30 AM job when 2 AM is skipped).
2. **How do you support job dependencies (DAGs)?** Each job can have a `depends_on` list of parent job IDs. When a parent completes successfully, the result processor checks if all parents are complete and triggers the child. Cycle detection at job creation time.
3. **How do you prevent a single large tenant from monopolizing workers?** Per-tenant queue partitions or priority levels. Fair scheduling ensures each tenant gets a proportional share of worker capacity.
4. **Can you support sub-second scheduling?** Yes, but it requires a different architecture: use a time-wheel (Hashed Timing Wheel) instead of database scanning. Pre-load upcoming triggers into an in-memory wheel with millisecond granularity.
5. **How do you handle job versioning?** Each job update increments a version number. Executions record which version they ran. Rolling updates can specify "drain current executions before switching to new version."

## 16) Closing Summary (30-60s)

> "We designed a distributed job scheduler that manages 10 million scheduled jobs with reliable, at-least-once trigger semantics. The scheduler engine is partitioned across nodes, each scanning a subset of jobs for due triggers using a Redis sorted set cache. Triggered jobs are dispatched through a Kafka-based queue to an auto-scaled worker fleet. We handle the 'minute boundary' burst with pre-computation and jitter, prevent double-triggers with atomic compare-and-swap, recover missed triggers on scheduler restart, and detect stuck executions with a timeout reaper. The system scales by adding scheduler partitions, queue partitions, and workers independently."

## Appendix A: Configuration / Tuning Knobs

| Knob | Default | Effect |
|------|---------|--------|
| `scan_interval` | 1s | How often the scheduler checks for due triggers |
| `trigger_jitter_max` | 5s | Maximum random delay added to trigger time |
| `partition_count` | 100 | Number of job partitions across scheduler nodes |
| `execution_timeout` | 600s | Default timeout for job execution |
| `max_retry_attempts` | 3 | Default retry count on failure |
| `retry_backoff_base` | 10s | Base delay for exponential backoff |
| `reaper_interval` | 30s | How often the reaper scans for stuck executions |
| `heartbeat_interval` | 30s | How often long-running jobs send heartbeats |
| `pre_trigger_window` | 30s | How far ahead to pre-compute triggers |
| `misfire_policy` | fire_immediately | What to do with missed triggers |

## Appendix B: Vocabulary Cheat Sheet

| Term | Definition |
|------|-----------|
| **Trigger** | The event of a scheduled job becoming due for execution. |
| **Cron Expression** | A string (e.g., "0 2 * * *") defining a recurring schedule. |
| **Misfire** | A trigger that was missed because the scheduler was down or overloaded. |
| **Reaper** | Background process that detects and handles stuck/timed-out executions. |
| **Partition** | A subset of jobs assigned to a specific scheduler node for trigger scanning. |
| **Jitter** | Small random delay added to trigger times to prevent thundering herd. |
| **Heartbeat** | Periodic signal from a worker indicating a long-running job is still alive. |
| **Idempotency Key** | The execution_id used by job handlers to ensure exactly-once side effects. |

## Appendix C: References

- Quartz Scheduler architecture and clustering
- Airflow scheduler deep dive
- Uber's Cherami (delay queue for scheduled delivery)
- Hashed Timing Wheels (Varghese & Lauck, 1987)
- Kubernetes CronJob implementation
