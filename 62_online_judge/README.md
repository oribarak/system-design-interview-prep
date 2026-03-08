# System Design Interview: Online Judge (LeetCode) -- "Perfect Answer" Playbook

## 0) Opening: Frame the Problem (30-60s)

"We are designing an online judge platform where users submit code solutions to programming problems, and the system compiles, executes, and evaluates those solutions against test cases -- returning verdict, runtime, and memory usage. The key challenges are secure code execution in sandboxed environments, managing a queue of submissions during peak contest periods, and providing fast feedback. I will focus on the sandboxed execution engine, the submission queue architecture, and how we scale to handle thousands of concurrent submissions during contests."

## 1) Clarifying Questions (2-5 min)

1. **Problem types**: Algorithm problems only, or also interactive problems, SQL, system design? -- Primarily algorithmic problems with standard I/O.
2. **Languages supported**: How many? -- 10+ languages (C++, Java, Python, Go, Rust, JavaScript, etc.).
3. **Scale**: How many users and submissions? -- 5 million registered users, 500K daily active, 2 million submissions/day.
4. **Contests**: Do we support real-time contests? -- Yes, with up to 50,000 concurrent participants.
5. **Test case count**: How many test cases per problem? -- 10-50 test cases, some with large inputs.
6. **Time/memory limits**: Per-language or per-problem? -- Per-problem, with per-language multipliers (e.g., Python gets 3x time limit vs. C++).
7. **Feedback**: Do users see which test case failed, or just pass/fail? -- Show first failing test case with expected vs. actual output.
8. **Plagiarism detection**: In scope? -- Yes, for contests.

## 2) Back-of-Envelope Estimation (3-5 min)

| Metric | Estimate |
|---|---|
| Daily submissions | 2M |
| Peak submissions/sec (normal) | 2M / 86400 ~ 23 sub/sec average; 100 sub/sec peak |
| Contest peak | 50K users x 5 submissions in 10 min = 25K submissions / 600s ~ 42 sub/sec burst |
| Combined peak | ~150 sub/sec |
| Average test cases per submission | 30 |
| Average execution per test case | 2 seconds |
| Total execution time per submission | 30 x 2s = 60 seconds |
| Required execution parallelism | 150 sub/sec x 60s = 9,000 concurrent executions |
| Worker count (1 execution per worker) | ~9,000 sandbox workers at peak |
| Problem count | 3,000 problems |
| Test case storage | 3K problems x 30 tests x 10 KB avg = ~1 GB |
| Submission code storage | 2M/day x 2 KB avg = 4 GB/day |

## 3) Requirements (3-5 min)

### Functional Requirements
- **Problem Management**: Create, edit, and store problems with descriptions, test cases, constraints, and editorial solutions.
- **Code Submission**: Accept code in multiple languages, compile, run against test cases, and return verdict (Accepted, Wrong Answer, Time Limit Exceeded, Memory Limit Exceeded, Runtime Error, Compilation Error).
- **Judging**: Compare program output against expected output for each test case. Support custom judges for problems with multiple valid outputs.
- **Contest Mode**: Time-limited contests with leaderboard, penalty scoring, and thousands of concurrent participants.
- **User Progress**: Track solved problems, submission history, and statistics.
- **Plagiarism Detection**: Compare submissions in contests for code similarity.

### Non-Functional Requirements
- **Latency**: Feedback within 10-30 seconds for most submissions.
- **Throughput**: Handle 150 submissions/sec at peak.
- **Security**: Sandboxed execution preventing malicious code from escaping, accessing the network, or consuming excessive resources.
- **Fairness**: Consistent execution environment ensuring reproducible timing results.
- **Scalability**: Scale execution workers elastically for contests.
- **Availability**: 99.9% uptime; higher during contests.

## 4) API Design (2-4 min)

### Submit Solution
```
POST /submissions
Body: { problem_id, language, code: string }
Response: { submission_id, status: "queued" }
```

### Poll Submission Result
```
GET /submissions/{submission_id}
Response: {
  submission_id, status: "judging" | "completed",
  verdict: "Accepted" | "Wrong Answer" | "TLE" | "MLE" | "RE" | "CE",
  runtime_ms: int,
  memory_kb: int,
  test_cases_passed: int,
  total_test_cases: int,
  first_failure: { test_case_num, input, expected, actual } | null
}
```

### WebSocket for Live Results
```
ws://judge.example.com/ws/submissions/{submission_id}
Server pushes: { test_case: 3, status: "passed", runtime_ms: 45 }
Server pushes: { verdict: "Accepted", total_runtime_ms: 234 }
```

### Get Problem
```
GET /problems/{problem_id}
Response: { id, title, description_html, constraints, sample_inputs, sample_outputs, time_limit_ms, memory_limit_kb }
```

### Contest Leaderboard
```
GET /contests/{contest_id}/leaderboard?page=<n>
Response: { rankings: [{ rank, user_id, username, solved, penalty, problem_scores }] }
```

## 5) Data Model (3-5 min)

### Problems
| Field | Type | Notes |
|---|---|---|
| problem_id | int | Primary key |
| title | string | |
| description | text | Markdown/HTML |
| difficulty | enum | Easy, Medium, Hard |
| time_limit_ms | int | Per-test-case limit |
| memory_limit_kb | int | |
| test_cases | list | Stored separately in blob storage |
| checker_type | enum | EXACT_MATCH, CUSTOM_JUDGE |
| custom_judge_code | text | For problems with multiple valid outputs |

### Submissions
| Field | Type | Notes |
|---|---|---|
| submission_id | uuid | Primary key |
| user_id | uuid | FK |
| problem_id | int | FK |
| contest_id | int | Nullable, FK |
| language | string | |
| code | text | Source code |
| verdict | enum | AC, WA, TLE, MLE, RE, CE |
| runtime_ms | int | Max across test cases |
| memory_kb | int | Max across test cases |
| test_cases_passed | int | |
| submitted_at | timestamp | |
| judged_at | timestamp | |

### Test Cases (blob storage)
| Field | Type | Notes |
|---|---|---|
| problem_id | int | |
| test_case_num | int | |
| input_file | blob | Input data |
| expected_output_file | blob | Expected output |
| is_sample | bool | Visible to users |

### Storage Choices
- **Problems and submissions metadata**: PostgreSQL.
- **Test case files**: S3/blob storage, cached on worker nodes.
- **Submission code**: PostgreSQL (small) or S3 for archival.
- **Leaderboard**: Redis sorted set for real-time contest rankings.
- **Submission queue**: Redis-backed queue or Kafka.

## 6) High-Level Architecture (5-8 min)

**One-sentence summary**: Users submit code through an API service, submissions are queued and dispatched to sandboxed execution workers that compile, run, and evaluate against test cases, with results streamed back to users via WebSocket and contest leaderboards updated in real time via Redis sorted sets.

### Components

1. **API Service** -- Handles user requests: submit code, poll results, browse problems.
2. **Submission Queue** -- Redis or Kafka queue holding pending submissions, with priority lanes for contests.
3. **Dispatcher** -- Pulls from the queue and assigns submissions to available workers.
4. **Execution Workers** -- Sandboxed environments that compile and run user code. Autoscaled.
5. **Sandbox** -- Container or VM-based isolation (seccomp, cgroups, namespaces) enforcing time/memory/syscall limits.
6. **Test Case Store** -- S3 with local caching on workers for test case input/output files.
7. **Judge / Checker** -- Compares program output to expected output. Supports exact match and custom checkers.
8. **Result Store** -- PostgreSQL for submission results.
9. **WebSocket Gateway** -- Pushes live judging progress to connected clients.
10. **Leaderboard Service** -- Redis sorted sets for contest rankings, updated on each submission result.
11. **Problem Management Service** -- CRUD for problems, test case upload, editorial management.

## 7) Deep Dive #1: Sandboxed Code Execution (8-12 min)

### Why Sandboxing is Critical

Users submit arbitrary code. Without sandboxing, malicious code could:
- Read/write files on the host system.
- Open network connections (exfiltrate data, attack other systems).
- Fork-bomb or consume all CPU/memory.
- Execute system calls to escape the container.
- Read other users' submissions or test case answers.

### Sandbox Architecture (Linux-based)

Each submission runs in an isolated sandbox with multiple layers of protection:

**Layer 1: Container Isolation**
- Each execution runs in a fresh, minimal container (or lightweight VM like Firecracker).
- Read-only root filesystem. No network access (network namespace with no interfaces).
- Mounted volumes: read-only test case input; writable /tmp for program output (tmpfs, size-limited).

**Layer 2: cgroups Resource Limits**
- CPU time limit: enforced via cgroup CPU controller. Hard kill after time_limit + 500 ms grace.
- Memory limit: cgroup memory controller. OOM-killed if exceeded.
- PID limit: max 64 processes (prevents fork bombs).
- Disk I/O limit: throttled via cgroup blkio.

**Layer 3: seccomp System Call Filtering**
- Whitelist allowed system calls (read, write, mmap, brk, exit, etc.).
- Block dangerous syscalls: execve (after initial program start), socket, mount, ptrace, clone (with certain flags).
- Language-specific profiles: Java needs more syscalls than C; Python needs execve for interpreter startup.

**Layer 4: User Namespace**
- Run the program as an unprivileged user (uid 65534) inside a user namespace.
- No capability to escalate privileges.

### Execution Flow

1. **Receive submission**: Worker pulls (problem_id, language, code) from the queue.
2. **Compile** (if compiled language): Run compiler in sandbox with a 30-second time limit. If compilation fails, return CE verdict.
3. **For each test case**:
   a. Create a fresh sandbox container.
   b. Mount test case input as stdin.
   c. Run the compiled binary (or interpreter with the script).
   d. Enforce time limit (wall clock + CPU time) and memory limit.
   e. Capture stdout and stderr.
   f. Compare stdout against expected output using the checker.
   g. Record runtime and memory usage.
   h. If wrong answer, record first failing test case and stop (or continue for full scoring).
4. **Aggregate results**: Determine final verdict, max runtime, max memory.
5. **Report**: Write result to database, push to WebSocket gateway.

### Timing Consistency

For fair comparison of runtimes between submissions:
- Use CPU time (not wall clock time) measured via getrusage or cgroup CPU accounting.
- Workers are dedicated machines (not shared VMs) with fixed CPU frequency (disable turbo boost, frequency scaling).
- Run on homogeneous hardware to ensure consistent benchmarks.
- Alternatively, compare against a reference solution's runtime on the same worker.

### Worker Pool Management

- Base pool: 500 workers for normal traffic (~23 sub/sec, each taking ~60s).
- Contest scaling: Auto-scale to 9,000+ workers for peak contests. Use cloud instances with pre-baked AMIs containing all language runtimes.
- Pre-warm: Before a contest, spin up workers 10 minutes early. Pre-cache test cases for contest problems.
- After contest: Scale down within 30 minutes.

## 8) Deep Dive #2: Contest Mode and Leaderboard (5-8 min)

### Contest Architecture

A contest has:
- Start time and end time (typically 1.5-2 hours).
- 4-8 problems.
- 10,000-50,000 participants.
- Real-time leaderboard with penalty-based scoring.

### Submission Priority During Contests

The submission queue has two priority lanes:
- **High priority**: Contest submissions. These must be judged quickly for fair competition.
- **Normal priority**: Practice submissions. Can tolerate longer wait times during contests.

During a contest, contest submissions are judged first. If the queue backs up, practice submissions may be delayed but contest fairness is maintained.

### Leaderboard

**Scoring**: For each problem, a user either solved it or not. Penalty = time of accepted submission (in minutes from contest start) + 10 minutes per wrong submission.

**Data structure**: Redis sorted set per contest.
- Key: `contest:{id}:leaderboard`
- Score: composite score encoding problems_solved (primary, descending) and penalty (secondary, ascending).
- Score formula: `(max_problems - solved) * 1e8 + penalty_minutes`
- Lower score = higher rank.

**Update flow**:
1. Submission is judged.
2. If verdict is Accepted and this is the user's first AC for this problem in this contest:
   a. Compute penalty for this problem.
   b. Update user's total solved count and penalty.
   c. ZADD to Redis sorted set with updated score.
3. Leaderboard query: ZRANGE with pagination.

**Leaderboard caching**: For large contests, cache the top-100 leaderboard and update every 5 seconds rather than on every submission.

### Anti-Cheating During Contests

1. **Plagiarism detection**: After the contest, run code similarity analysis (MOSS or custom token-based comparison) on all accepted submissions per problem. Flag pairs with > 90% similarity.
2. **Multi-account detection**: IP address and browser fingerprint matching.
3. **Submission timing**: Flag users who solve hard problems in suspiciously short time.
4. **Lockdown mode**: Optionally require participants to share their screen or use a lockdown browser for proctored contests.

## 9) Trade-offs and Alternatives (3-5 min)

| Decision | Chosen | Alternative | Rationale |
|---|---|---|---|
| Sandbox technology | Containers + seccomp + cgroups | Lightweight VMs (Firecracker) | Containers are faster to start (~100 ms vs. ~1 s); VMs offer stronger isolation for high-security scenarios |
| Submission queue | Redis queue with priority lanes | Kafka | Redis is simpler for priority queuing; Kafka is better if we need replay/durability |
| Leaderboard | Redis sorted set | PostgreSQL materialized view | Redis provides O(log N) updates and range queries; Postgres cannot match this for real-time |
| Result delivery | WebSocket + polling fallback | Polling only | WebSocket gives instant feedback; polling fallback for compatibility |
| Test case storage | S3 + local cache on workers | NFS shared mount | S3 is more scalable; local cache avoids network latency during execution |

## 10) Scaling: 10x and 100x (3-5 min)

### 10x (1,500 sub/sec, 500K concurrent contest users)
- 90,000 execution workers at peak. Use spot/preemptible instances for cost savings (tolerate worker loss; requeue submission).
- Shard the submission queue by problem_id to reduce contention.
- Multiple Redis instances for leaderboard (shard by contest_id).
- CDN for problem descriptions, images, and editorial content.
- Regional deployment: workers in multiple regions to reduce latency.

### 100x (15,000 sub/sec)
- Tiered execution: run sample test cases first on lightweight workers. Only if sample passes, run full test suite on full workers. This eliminates ~60% of submissions early.
- Pre-compiled language runtimes in read-only layers to speed up container start.
- Distributed judging: split test cases across multiple workers for a single submission (parallel test case execution).
- Custom lightweight sandbox (like isolate) instead of full containers for faster startup.

## 11) Reliability and Fault Tolerance (3-5 min)

- **Worker failure during execution**: Submission is re-queued after a timeout (e.g., 5 minutes with no result). Idempotent: re-running the same submission produces the same result.
- **Queue durability**: Redis queue with AOF persistence. If Redis fails, submissions in flight are lost; users can resubmit. For stronger guarantees, use Kafka.
- **Database failure**: PostgreSQL with replication. Read replicas for submission history queries.
- **Contest resilience**: If judging is delayed during a contest, extend the contest duration. Leaderboard freezes in the last 30 minutes (standard practice) to maintain excitement and reduce server load.
- **Sandbox escape**: If detected (should be extremely rare), kill the worker instance entirely. Alert security team. The worker's ephemeral nature means no persistent damage.

## 12) Observability and Operations (2-4 min)

- **Queue depth**: Track pending submissions per priority lane. Alert if contest queue depth > 1000.
- **Worker utilization**: % of workers actively judging vs. idle. Auto-scaling trigger.
- **Judging latency**: Time from submission to result. Target < 30 seconds. Alert if p95 > 60 seconds.
- **Verdict distribution**: Track AC/WA/TLE/RE/CE rates per problem. Unusual spikes may indicate problem issues.
- **Sandbox metrics**: Container startup time, OOM kill rate, seccomp violation rate.
- **Leaderboard update latency**: Time from submission result to leaderboard reflection.

## 13) Security (1-3 min)

- **Code execution isolation**: Multi-layer sandbox (container, cgroups, seccomp, user namespace). No network access.
- **Test case confidentiality**: Test cases are not exposed to users. Only sample test cases are visible. Workers fetch test cases over internal network only.
- **Input validation**: Limit code size (< 100 KB). Sanitize language parameter. Rate-limit submissions per user (e.g., 1 per 5 seconds).
- **Authentication**: JWT-based auth for API. Contest submissions require registered and verified accounts.
- **DDoS**: Rate limiting at the API gateway. Captcha for submission if rate exceeds threshold.

## 14) Team and Operational Considerations (1-2 min)

- **Platform team**: Owns submission queue, dispatcher, worker orchestration, auto-scaling. Focused on throughput and reliability.
- **Sandbox team**: Owns execution environment, security hardening, language runtime management. Focused on security and consistency.
- **Product team**: Owns problem management, contest features, leaderboard, user experience.
- **Content team**: Owns problem creation, test case generation, editorial writing, difficulty calibration.
- **On-call**: Critical during live contests. Pre-contest checklist includes worker pool warm-up, test case cache priming, and leaderboard initialization.

## 15) Common Follow-up Questions

1. **How do you support interactive/reactive problems?** -- Two-process sandbox: the user's program and the interactor run simultaneously, communicating via pipes. The interactor sends queries and validates responses.
2. **How do you generate test cases?** -- Problem authors write test generators (programs that produce test input). Validators check input constraints. Model solutions generate expected output. Stress testing catches edge cases.
3. **How do you handle language-specific differences?** -- Per-language time/memory multipliers (e.g., Python 3x time, Java 2x time). Language-specific seccomp profiles. Pre-installed standard libraries.
4. **How do you prevent timing side-channel attacks on test cases?** -- Randomize test case order. Do not reveal which specific test case failed in contest mode (only show sample cases).
5. **How do you support custom checkers?** -- Custom checker is a trusted program (written by problem author) that reads the input, expected output, and user output, then returns AC or WA with an optional message.

## 16) Closing Summary (30-60s)

"We designed an online judge that accepts code submissions via an API, queues them with priority lanes for contests, and dispatches to sandboxed execution workers protected by containers, cgroups, seccomp, and user namespaces. Each submission runs against 30+ test cases with enforced time and memory limits, and results stream back via WebSocket. The system scales to 150 submissions/sec by auto-scaling to thousands of workers, supports contests with 50K participants using Redis sorted sets for real-time leaderboards, and ensures security through multi-layer sandboxing with no network access."

## Appendix A: Configuration / Tuning Knobs

| Knob | Default | Tuning Guidance |
|---|---|---|
| Time limit per test case | 1-5 seconds | Per-problem; with language multipliers |
| Memory limit | 256 MB | Per-problem; increase for graph/DP problems |
| Max code size | 100 KB | Prevent abuse; most solutions < 10 KB |
| Submission rate limit | 1 per 5 seconds | Tighter during contests to prevent brute-force |
| Worker container startup | < 200 ms | Use pre-warmed containers; critical for throughput |
| Contest leaderboard cache TTL | 5 seconds | Balance freshness vs. Redis load |
| Test case cache TTL on workers | 1 hour | Problems rarely change; long TTL is safe |
| Seccomp syscall whitelist | Language-specific | Audit periodically; minimize allowed syscalls |

## Appendix B: Vocabulary Cheat Sheet

| Term | Definition |
|---|---|
| Verdict | The result of judging: AC, WA, TLE, MLE, RE, CE |
| Sandbox | Isolated execution environment preventing malicious code from affecting the host |
| seccomp | Linux kernel feature for system call filtering |
| cgroups | Linux kernel feature for resource limiting (CPU, memory, PIDs) |
| Checker | Program that compares user output to expected output |
| Penalty | Contest scoring: time of AC + penalty for wrong submissions |
| MOSS | Measure of Software Similarity; plagiarism detection tool |
| Firecracker | Lightweight VM technology by AWS for micro-VM isolation |
| Isolate | Lightweight sandbox tool commonly used in competitive programming judges |

## Appendix C: References

- Wasik et al. "A Survey on Online Judge Systems and Their Applications" (ACM Computing Surveys, 2018)
- Codeforces architecture blog posts by Mike Mirzayanov
- Firecracker: Lightweight Virtualization for Serverless Applications (NSDI 2020)
- Linux cgroups v2 documentation
- seccomp-bpf documentation (Linux kernel)
- MOSS (Measure of Software Similarity) by Stanford
