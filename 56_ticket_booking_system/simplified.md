# Ticket Booking System (Ticketmaster) -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
We need a ticket booking system that handles extreme traffic spikes when popular events go on sale -- millions of users competing for thousands of seats simultaneously. The hard part is preventing overselling (each seat sold exactly once) under massive contention while keeping the system responsive, and fairly managing access through a virtual queue.

## 2. Requirements
**Functional:** Event discovery and search. Real-time seat map display. Temporary seat holds (8-minute TTL) during checkout. Payment processing within the hold window. Virtual waiting queue for high-demand events. Inventory management preventing overselling.

**Non-functional:** No overselling -- strong consistency on inventory. 500K QPS during hot on-sales. Seat hold acquisition under 200ms. 99.99% for browsing, 99.9% for checkout. Fair access through randomized virtual queue.

## 3. Core Concept: FOR UPDATE SKIP LOCKED + Virtual Queue
Two mechanisms work together. The virtual queue controls admission rate, letting only N users per second into the seat selection page, preventing thundering herd. Once admitted, seat reservation uses PostgreSQL's FOR UPDATE SKIP LOCKED -- this acquires row locks without blocking on already-locked seats. If someone else already locked a seat, the query skips it instead of waiting, eliminating deadlocks. Combined with a version column for optimistic concurrency, this guarantees exactly-once seat sales under massive contention.

## 4. High-Level Architecture
```
Users --> [Virtual Queue] -- Redis Sorted Set, randomized positions
               |
        (admitted at controlled rate)
               |
         [Seat Map Service] -- Redis-cached availability (eventual consistency)
               |
         [Inventory Service] -- PostgreSQL with row-level locks (strong consistency)
               |
         [Hold Expiration Worker] -- Redis TTL triggers seat release
               |
         [Order Service] --> [Payment Service]
```
- **Virtual Queue**: Redis Sorted Set with randomized scores. Admits users at a rate calibrated to inventory and checkout capacity.
- **Inventory Service**: PostgreSQL with FOR UPDATE SKIP LOCKED for contention-free seat locking.
- **Hold Expiration Worker**: Redis TTL detects expired holds; worker updates PostgreSQL to release seats.
- **Seat Map Service**: Redis-cached view refreshed every 2 seconds (eventual consistency is fine for display).

## 5. How Ticket Purchase Works
1. User joins the virtual queue 30 minutes before on-sale; gets a randomized position (not FIFO, to prevent bot advantage).
2. Queue admits users at a controlled rate (e.g., 500/sec based on inventory and checkout capacity).
3. Admitted user receives a time-limited access token (JWT) required for seat selection APIs.
4. User selects seats; Inventory Service runs FOR UPDATE SKIP LOCKED to atomically hold them (8-minute TTL).
5. User proceeds to checkout; payment is processed within the hold window.
6. On payment success: seats marked SOLD, order created. On failure: hold remains active for retry.
7. If hold expires (user abandons): Hold Expiration Worker releases seats back to AVAILABLE.

## 6. What Happens When Things Fail?
- **PostgreSQL primary failure**: Synchronous standby promotes. No committed holds are lost.
- **Payment timeout**: Retry with idempotency key. Hold remains active during the retry window. Idempotency key prevents double-charge.
- **Hold expiration worker crash**: Database-level expiration query runs as backup every 30 seconds: seats with expired holds are released.

## 7. Scaling
- **10x**: Shard inventory by event_id (each event on its own DB shard). Read replicas for seat map display. Multi-level caching: CDN for event pages, Redis for seat maps.
- **100x**: Represent seat maps as bitmaps (50K seats = 6.25KB). Pre-partition large events into independent sections with separate DB shards. Dedicated infrastructure for mega-events.

## 8. Key Trade-offs to Mention
| Decision | Trade-off |
|----------|-----------|
| PostgreSQL with row locks vs Redis for inventory | PostgreSQL gives ACID for financial transactions; Redis cannot guarantee durability for inventory |
| SKIP LOCKED vs regular SELECT FOR UPDATE | SKIP LOCKED prevents deadlocks and queue buildup; regular locking causes cascading waits |
| Randomized queue vs FIFO | Randomized prevents bot advantage from network latency; less intuitive for users |

## 9. Closing (30s)
> The system handles extreme on-sale concurrency through two mechanisms: a virtual queue that randomizes positions and rate-limits admission, and PostgreSQL's FOR UPDATE SKIP LOCKED for contention-free seat locking that guarantees no overselling. Holds are time-limited (8 minutes) with Redis TTL expiration, automatically releasing abandoned seats. The system handles 500K QPS through CDN caching, Redis seat maps, and queue-controlled admission rates calibrated to downstream capacity.
