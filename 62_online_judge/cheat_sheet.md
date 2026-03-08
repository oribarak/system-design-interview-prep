# Online Judge (LeetCode) -- Cheat Sheet

## Key Numbers
- 2M submissions/day, 150 peak sub/sec (contests)
- 30 test cases/submission, ~60s total execution time
- 9,000 concurrent sandbox workers at contest peak
- 3,000 problems, ~1 GB test case storage
- Feedback target: < 30 seconds from submission to verdict

## Core Components
- Submission Queue: Redis with priority lanes (contest high, practice normal)
- Dispatcher: pulls from queue, assigns to available workers
- Execution Workers: auto-scaled pool of sandbox instances (containers + cgroups + seccomp)
- Sandbox: no network, syscall whitelist, CPU/memory/PID limits, unprivileged user namespace
- Checker: exact match or custom judge program for output validation
- Leaderboard: Redis sorted set with composite score (solved count + penalty)
- WebSocket Gateway: streams per-test-case judging progress to user

## Architecture in One Sentence
Users submit code to a priority queue; a dispatcher assigns submissions to auto-scaled sandbox workers that compile and run code against test cases under strict resource limits, streaming results via WebSocket and updating Redis sorted set leaderboards for contests.

## Top 3 Trade-offs
1. Container sandbox vs. lightweight VM (Firecracker): containers start faster (~200ms vs ~1s) but offer slightly weaker isolation; acceptable with seccomp + cgroups hardening.
2. Priority queue (Redis) vs. Kafka: Redis supports simple priority lanes for contest fairness; Kafka adds durability but complicates priority ordering.
3. Sequential vs. parallel test case execution: sequential is simpler and uses less resources per submission; parallel reduces judging latency but needs distributed coordination.

## Scaling Story
10x: spot/preemptible worker instances, shard queue by problem_id, regional worker pools.
100x: tiered execution (sample tests first, full suite only if samples pass), parallel test case execution across workers, custom lightweight sandbox (isolate).

## Closing Statement
A multi-layer sandboxed execution system with priority-queued dispatch, auto-scaled container workers enforcing CPU/memory/syscall limits, WebSocket result streaming, and Redis sorted set leaderboards delivers secure, fair code evaluation for 150 concurrent submissions per second across contests with 50K participants.
