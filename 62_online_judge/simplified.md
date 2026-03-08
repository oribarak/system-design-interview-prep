# Online Judge (LeetCode) -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
We need an online judge that accepts code submissions in 10+ languages, compiles and runs them against test cases in a sandboxed environment, and returns verdicts within 30 seconds. The hard part is securely executing arbitrary user code (preventing filesystem access, network calls, and fork bombs) while providing consistent, fair timing for 150 submissions per second during contests with 50K participants.

## 2. Requirements
**Functional:** Accept code submissions in multiple languages. Compile and run against 10-50 test cases per problem with time/memory limits. Return verdicts (Accepted, Wrong Answer, TLE, MLE, Runtime Error, Compile Error). Support real-time contests with leaderboards. Show first failing test case.

**Non-functional:** Feedback within 10-30 seconds. Handle 150 submissions/sec at peak (contests). Multi-layer sandboxing preventing code escape. Fair, consistent timing across submissions. Elastic scaling of execution workers.

## 3. Core Concept: Multi-Layer Sandboxed Execution
Each submission runs in a fresh container with four layers of protection. Container isolation provides a read-only filesystem and no network access. Cgroups enforce CPU time, memory limits (OOM-kill), and PID limits (max 64, prevents fork bombs). Seccomp system call filtering whitelists only safe syscalls (read, write, mmap, exit) and blocks dangerous ones (socket, mount, ptrace). User namespaces run the program as an unprivileged user with no capability escalation. This defense-in-depth means even if one layer is bypassed, others contain the threat.

## 4. High-Level Architecture
```
User --> [API Service] --> [Submission Queue] -- Redis with priority lanes
                                  |
                           [Dispatcher] --> assigns to available workers
                                  |
                      [Execution Worker] -- fresh sandbox per submission
                      (container + cgroups + seccomp + user namespace)
                                  |
                      [Test Case Store] -- S3 with local cache
                                  |
                      [Result Store] -- PostgreSQL
                      [Leaderboard] -- Redis Sorted Set
                      [WebSocket Gateway] -- live judging progress
```
- **Submission Queue**: Redis-backed with priority lanes (contest submissions high priority, practice normal).
- **Execution Workers**: Auto-scaled sandboxed environments. Base pool of 500, scaling to 9,000+ for contests.
- **Leaderboard**: Redis Sorted Set with composite score encoding solved count and penalty time.
- **WebSocket Gateway**: Streams per-test-case progress to connected users.

## 5. How a Submission Gets Judged
1. User submits code; API Service enqueues in the submission queue with priority level.
2. Dispatcher assigns to an available worker.
3. Worker compiles code in a sandbox (30-second limit). If compilation fails, return CE verdict.
4. For each test case: create fresh sandbox, mount input as stdin, run with time/memory limits.
5. Capture stdout, compare against expected output using the checker (exact match or custom judge).
6. Record runtime (CPU time via cgroup accounting) and memory usage per test case.
7. If wrong answer, record first failing test case and stop.
8. Write result to database, push progress via WebSocket, update leaderboard if contest.

## 6. What Happens When Things Fail?
- **Worker failure during execution**: Submission is re-queued after 5-minute timeout. Re-running is idempotent (same code, same test cases, same result).
- **Contest submission backlog**: Contest queue has priority over practice. Auto-scale workers. If still delayed, extend contest duration.
- **Sandbox escape detected**: Kill the worker instance entirely. Alert security team. Worker's ephemeral nature means no persistent damage.

## 7. Scaling
- **10x**: 90,000 workers at peak using spot/preemptible instances. Shard submission queue by problem_id. Regional worker deployment.
- **100x**: Tiered execution: run sample test cases first on lightweight workers; only if sample passes, run full suite (eliminates 60% early). Parallel test case execution across multiple workers for a single submission.

## 8. Key Trade-offs to Mention
| Decision | Trade-off |
|----------|-----------|
| Containers + seccomp vs lightweight VMs (Firecracker) | Containers start in ~100ms vs ~1s for VMs; VMs offer stronger isolation for higher-security scenarios |
| Redis queue with priority lanes vs Kafka | Redis is simpler for priority queuing; Kafka is better for replay and durability |
| CPU time measurement vs wall clock | CPU time is fairer (not affected by co-tenants) but harder to implement; wall clock is simpler but less consistent |

## 9. Closing (30s)
> Submissions are queued with priority lanes (contest over practice), dispatched to sandboxed workers protected by four layers (containers, cgroups, seccomp, user namespaces). Each submission runs against 30+ test cases with enforced time and memory limits, and results stream back via WebSocket. The system scales to 150 submissions/sec by auto-scaling to thousands of workers, supports 50K-participant contests with Redis sorted set leaderboards, and ensures security through defense-in-depth sandboxing with no network access.
