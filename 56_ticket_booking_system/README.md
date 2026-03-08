# System Design Interview: Ticket Booking System (Ticketmaster) — "Perfect Answer" Playbook

## 0) Opening: Frame the Problem (30-60s)

"I'll design a ticket booking system like Ticketmaster that handles event discovery, seat selection, reservation, and payment for concerts, sports, and live events. The core challenge is handling extreme traffic spikes when popular events go on sale -- millions of users competing for thousands of seats simultaneously. I'll focus on the inventory management system that prevents overselling, the virtual waiting queue that handles traffic bursts, and how we maintain a responsive experience under massive contention."

## 1) Clarifying Questions (2-5 min)

| # | Question | Likely Answer |
|---|----------|---------------|
| 1 | What types of events? Assigned seating or general admission? | Both: venue maps with assigned seats and GA sections |
| 2 | How many concurrent users during a hot on-sale? | Up to 10M users for a Taylor Swift-class event |
| 3 | How many tickets per event? | 10K - 100K seats per event |
| 4 | Is there a virtual queue for high-demand events? | Yes, queue-based access to prevent thundering herd |
| 5 | Ticket hold time? How long to complete purchase? | 5-10 minute hold after seat selection |
| 6 | Dynamic pricing? | Yes, some events have demand-based pricing |
| 7 | Resale/transfer? | Secondary market support, but not primary focus |
| 8 | Payment processing? | Integrated, with hold-and-charge pattern |

## 2) Back-of-Envelope Estimation (3-5 min)

| Metric | Estimate |
|--------|----------|
| Events per day | 5,000 events with active sales |
| Total tickets sold/day | 2M tickets |
| Avg daily QPS | 2M / 86,400 ~ 23 QPS |
| Peak QPS (hot on-sale) | 500K QPS (10M users hitting refresh) |
| Hot event inventory | 50,000 seats |
| Hold duration | 8 minutes |
| Active holds at peak | 50,000 (entire inventory held) |
| Hold expiration rate | 50,000 / 8 min ~ 100 expirations/sec |
| Payment processing/sec (peak) | ~5,000 TPS |
| Seat map payload | ~200 KB (venue with 50K seats) |
| Ticket record size | ~1 KB |
| Daily ticket storage | 2M * 1 KB = 2 GB/day |

## 3) Requirements (3-5 min)

### Functional Requirements
1. **Event browsing**: search and discover events by artist, venue, date, genre
2. **Seat map display**: show available/taken seats in real-time
3. **Seat reservation (hold)**: temporarily hold selected seats for checkout (8-minute TTL)
4. **Checkout and payment**: complete purchase within hold window
5. **Virtual queue**: manage access for high-demand events to prevent system overload
6. **Inventory management**: prevent overselling -- each seat sold exactly once
7. **Order management**: view purchased tickets, transfers, refunds

### Non-Functional Requirements
- **Consistency**: no overselling -- seats must never be double-sold (strong consistency on inventory)
- **Availability**: 99.99% for event browsing; 99.9% for checkout
- **Latency**: seat map load < 500ms; hold acquisition < 200ms
- **Scalability**: handle 500K QPS during hot on-sales
- **Fairness**: virtual queue ensures first-come-first-served access
- **Hold expiration**: abandoned holds must be released promptly

## 4) API Design (2-4 min)

### Event Discovery
```
GET /v1/events?query=taylor+swift&city=NYC&date_from=2024-06-01
Response: { "events": [{ "event_id": "evt_123", "name": "...", "venue": "...", "date": "...", "price_range": "$99-$500" }] }
```

### Queue and Access
```
POST /v1/queue/join
{ "event_id": "evt_123", "user_id": "user_456" }
Response: { "queue_token": "qt_abc", "position": 45230, "estimated_wait_min": 12 }

GET /v1/queue/status?queue_token=qt_abc
Response: { "position": 12400, "status": "WAITING" | "ADMITTED", "access_token": "at_xyz" }
```

### Seat Selection and Hold
```
GET /v1/events/{event_id}/seats?section=A
Response: { "seats": [{ "seat_id": "A-101", "status": "AVAILABLE", "price": 250 }] }

POST /v1/holds
{ "event_id": "evt_123", "seat_ids": ["A-101", "A-102"], "access_token": "at_xyz" }
Response: { "hold_id": "hold_789", "expires_at": "2024-03-07T16:08:00Z", "total": 500 }
```

### Checkout
```
POST /v1/orders
{ "hold_id": "hold_789", "payment_method_id": "pm_abc" }
Response: { "order_id": "ord_123", "status": "CONFIRMED", "tickets": [...] }
```

## 5) Data Model (3-5 min)

### Core Entities

**Event**
| Column | Type | Notes |
|--------|------|-------|
| event_id | UUID | PK |
| name | string | |
| venue_id | UUID | FK |
| event_date | timestamp | |
| on_sale_date | timestamp | |
| status | enum | UPCOMING, ON_SALE, SOLD_OUT, COMPLETED |
| total_capacity | int | |

**Seat**
| Column | Type | Notes |
|--------|------|-------|
| seat_id | UUID | PK |
| event_id | UUID | FK, partition key |
| section | string | |
| row | string | |
| number | int | |
| status | enum | AVAILABLE, HELD, SOLD |
| price | decimal | |
| hold_id | UUID | nullable |
| hold_expires_at | timestamp | nullable |
| version | int | optimistic locking |

**Hold**
| Column | Type | Notes |
|--------|------|-------|
| hold_id | UUID | PK |
| event_id | UUID | |
| user_id | UUID | |
| seat_ids | UUID[] | |
| status | enum | ACTIVE, EXPIRED, CONVERTED |
| expires_at | timestamp | |
| created_at | timestamp | |

**Order**
| Column | Type | Notes |
|--------|------|-------|
| order_id | UUID | PK |
| user_id | UUID | |
| event_id | UUID | |
| hold_id | UUID | |
| seat_ids | UUID[] | |
| total_amount | decimal | |
| payment_id | string | external payment reference |
| status | enum | PENDING, CONFIRMED, REFUNDED |
| created_at | timestamp | |

### Storage Choices
- **Seats/Inventory**: PostgreSQL with row-level locking -- strong consistency for inventory operations
- **Events**: PostgreSQL -- relational, ACID
- **Queue state**: Redis Sorted Sets -- fast position tracking, atomic operations
- **Holds**: Redis with TTL -- automatic expiration, fast lookups
- **Orders**: PostgreSQL -- ACID for financial records
- **Event search**: Elasticsearch -- full-text search, faceted filtering
- **Seat map cache**: Redis -- cache rendered seat availability maps

## 6) High-Level Architecture (5-8 min)

When a hot event goes on sale, users enter a virtual queue. The Queue Service admits users at a controlled rate. Admitted users can view the seat map, select seats, and create holds. The Inventory Service manages seat state transitions with strong consistency. Holds expire after 8 minutes if not converted to orders. The Payment Service handles charges, and the Order Service records confirmed purchases.

### Components
- **API Gateway**: auth, rate limiting, routes to appropriate service
- **Queue Service**: virtual waiting room for high-demand events, controls admission rate
- **Event Service**: event CRUD, search, venue management
- **Inventory Service**: seat state management (available/held/sold), hold creation/expiration
- **Seat Map Service**: renders and caches seat availability views
- **Order Service**: checkout flow, order management
- **Payment Service**: payment processing, refunds
- **Notification Service**: confirmation emails, hold expiration warnings
- **Hold Expiration Worker**: background process that releases expired holds

## 7) Deep Dive #1: Inventory Management and Preventing Overselling (8-12 min)

### The Core Problem
50,000 seats, millions of concurrent users, each seat must be sold exactly once. This is a high-contention distributed concurrency problem.

### Approach: Database Row-Level Locking with Optimistic Concurrency

```sql
-- Acquire hold on seats (atomic)
BEGIN;

SELECT seat_id, status, version
FROM seats
WHERE event_id = ? AND seat_id IN (?, ?) AND status = 'AVAILABLE'
FOR UPDATE SKIP LOCKED;

-- If all requested seats are available and locked:
UPDATE seats
SET status = 'HELD', hold_id = ?, hold_expires_at = NOW() + INTERVAL '8 minutes', version = version + 1
WHERE event_id = ? AND seat_id IN (?, ?) AND status = 'AVAILABLE' AND version = ?;

-- If affected rows != requested count: rollback (some seats were taken)
COMMIT;
```

**Key techniques:**
- `FOR UPDATE SKIP LOCKED`: acquires row locks without blocking. If a seat is already locked by another transaction, skip it instead of waiting. This prevents deadlocks and reduces contention.
- `version` column: optimistic concurrency control. Prevents lost updates.
- Atomic multi-seat hold: either all requested seats are held or none (all-or-nothing).

### Hold Expiration

**Approach 1: Redis TTL (chosen)**
- When a hold is created, store in Redis with TTL = 8 minutes
- Hold Expiration Worker polls Redis for expired keys (using keyspace notifications or periodic scan)
- On expiration: update seat status back to AVAILABLE in PostgreSQL

**Approach 2: Database-based expiration**
- Background job every 10 seconds: `UPDATE seats SET status = 'AVAILABLE', hold_id = NULL WHERE status = 'HELD' AND hold_expires_at < NOW()`
- Simpler but creates periodic write storms

**Chosen: Hybrid**
- Redis TTL for fast expiration detection
- Database as source of truth for seat status
- Worker updates DB when Redis TTL fires

### Converting Hold to Order (Checkout)

```sql
BEGIN;

-- Verify hold is still active
SELECT * FROM holds WHERE hold_id = ? AND status = 'ACTIVE' AND expires_at > NOW();

-- Charge payment (external call, idempotent)
-- payment_result = payment_service.charge(...)

-- If payment succeeds:
UPDATE seats SET status = 'SOLD' WHERE hold_id = ?;
UPDATE holds SET status = 'CONVERTED' WHERE hold_id = ?;
INSERT INTO orders (...) VALUES (...);

COMMIT;
```

### Handling Edge Cases
- **User selects seats that become unavailable**: return error immediately, show updated seat map
- **Payment fails during checkout**: hold remains active, user can retry until expiration
- **Hold expires during payment processing**: payment charge should be idempotent; if seat already released, refund and show error
- **Database failover during hold**: replicated PostgreSQL with synchronous replication to prevent lost holds

## 8) Deep Dive #2: Virtual Queue System (5-8 min)

### Why a Queue?
Without a queue, 10M users hitting the system simultaneously would overwhelm the inventory service. The queue controls the rate of users accessing the seat selection page.

### Queue Architecture

**Join Phase (before on-sale time):**
1. Users enter the waiting room starting 30 minutes before on-sale
2. Each user gets a random queue position (randomized, not first-come -- prevents bot advantage from network proximity)
3. Queue state stored in Redis Sorted Set: `ZADD queue:{event_id} {random_score} {user_id}`

**Admission Phase (on-sale time):**
1. Queue Service admits users at a controlled rate (e.g., 1,000 users/second)
2. Admission rate calibrated based on: checkout conversion rate, hold duration, and available inventory
3. Admitted user receives an `access_token` (JWT, single-use, time-limited)
4. Access token required for all seat selection and hold APIs

**Rate Calculation:**
```
target_admission_rate = available_inventory / (avg_checkout_duration_min * 60) * (1 / conversion_rate)

Example:
  50,000 seats / (3 min * 60 sec) * (1 / 0.5 conversion) = 555 users/sec admitted
```

### Queue Fairness and Anti-Bot Measures
- **Randomized position**: prevents bots with fast connections from getting first positions
- **CAPTCHA on queue join**: human verification before entering queue
- **Device fingerprinting**: detect and block multiple sessions from same device
- **IP rate limiting**: cap queue joins per IP
- **Access token binding**: token tied to session/device; cannot be transferred

### Queue Status Updates
- Users poll `/queue/status` every 5 seconds (or use WebSocket)
- Response includes estimated wait time based on current position and admission rate
- Position updates in Redis Sorted Set are O(1) reads

### Overflow Handling
- If all inventory is held or sold, stop admitting from queue
- Show "currently no available seats, please wait" -- users may get in when holds expire
- When holds expire, inventory opens up, and queue resumes admission

## 9) Trade-offs and Alternatives (3-5 min)

| Decision | Alternative | Why This Choice |
|----------|-------------|-----------------|
| PostgreSQL with row locks | Redis for inventory | PostgreSQL gives ACID for financial transactions; Redis can't guarantee durability for inventory |
| SKIP LOCKED | Regular SELECT FOR UPDATE | SKIP LOCKED prevents deadlocks and queue buildup on hot rows |
| Virtual queue (randomized) | First-come-first-served | Randomized prevents bot advantage from network latency |
| Redis for holds with TTL | Database-only holds | Redis TTL auto-expires holds; DB requires polling |
| Seat-level inventory | Section-level counters | Seat-level supports assigned seating; section counters only for GA |

## 10) Scaling: 10x and 100x (3-5 min)

### 10x Scale (5M tickets/day, 5M QPS peak)
- **Shard inventory by event_id**: each event's seats on a dedicated DB shard
- **Read replicas for seat maps**: serve seat availability from replicas (eventual consistency OK for display)
- **Multi-level caching**: CDN for event pages, Redis for seat maps, DB for authoritative state
- **Queue service sharding**: separate Redis instance per event

### 100x Scale (50M tickets/day)
- **Seat map as bitmap**: 50K seats = 50K bits = 6.25 KB; broadcast bitmap updates via WebSocket
- **Pre-partitioned inventory**: split large events into independent sections, each with its own DB shard
- **Async order processing**: hold is synchronous (consistency required), but order creation and payment can be async
- **Edge admission**: push queue admission logic to edge CDN to reduce round-trip latency
- **Hot event isolation**: deploy dedicated infrastructure for mega-events (separate cluster)

## 11) Reliability and Fault Tolerance (3-5 min)

| Failure Mode | Mitigation |
|--------------|------------|
| PostgreSQL primary failure | Synchronous standby promotes; no committed holds lost |
| Redis (queue) crash | Queue state lost; re-admit users based on access tokens already issued |
| Hold expiration worker crash | Database-level expiration query runs as backup (every 30 seconds) |
| Payment service timeout | Retry with idempotency key; hold remains active during retry window |
| Double-charge | Payment idempotency key prevents duplicate charges |
| Overselling due to race condition | `FOR UPDATE SKIP LOCKED` + version check makes this impossible under correct operation |
| CDN failure during on-sale | Direct-to-origin fallback; capacity planned for this scenario |

### Consistency Guarantees
- **Inventory**: strong consistency (PostgreSQL ACID)
- **Seat map display**: eventual consistency (cached, refreshed every 2 seconds)
- **Queue position**: eventual consistency (Redis, good enough for UX)
- **Orders**: strong consistency (PostgreSQL ACID)

## 12) Observability and Operations (2-4 min)

### Key Metrics
- Queue depth and admission rate per event
- Hold creation rate and conversion rate (holds that become orders)
- Hold expiration rate (indicates user drop-off)
- Seat map cache hit rate
- Inventory service latency p50/p99 (target: < 200ms for hold acquisition)
- Payment success rate and latency
- Overselling count (should always be 0)

### Pre-Event Checklist
- Load test with expected traffic (simulated queue + holds)
- Pre-warm caches (seat maps, event pages)
- Verify queue service capacity
- Confirm payment service rate limits
- Set up dedicated monitoring dashboard for the event

### Alerts
- Any overselling detected (P1, impossible but critical to monitor)
- Hold conversion rate drops below 30% (possible checkout issues)
- Queue admission rate below target (possible downstream bottleneck)
- Payment error rate exceeds 5%
- Inventory service p99 exceeds 1 second

## 13) Security (1-3 min)

- **Bot prevention**: CAPTCHA, device fingerprinting, IP rate limiting, behavioral analysis
- **Access token security**: JWT with short TTL, bound to user session, single-use for hold operations
- **Payment security**: PCI-DSS compliance, tokenized card data, no card numbers stored
- **Scalping prevention**: purchase limits per user (max 4 tickets), phone number verification
- **DDoS protection**: CDN-level DDoS mitigation, queue absorbs traffic bursts
- **Ticket authenticity**: QR codes with cryptographic signatures, validated at venue entry

## 14) Team and Operational Considerations (1-2 min)

- **Inventory team (4-6 engineers)**: seat management, hold system, consistency guarantees
- **Queue team (2-3 engineers)**: virtual queue, admission control, anti-bot
- **Commerce team (3-4 engineers)**: checkout, payment integration, order management
- **Platform team (3-5 engineers)**: API gateway, caching, infrastructure scaling
- **Event ops team**: per-event capacity planning, on-sale monitoring
- **On-call**: dedicated war room for mega-events (Taylor Swift, Super Bowl)

## 15) Common Follow-up Questions

**Q: How do you handle general admission (no assigned seats)?**
A: Use a counter-based approach instead of seat-level tracking. Redis atomic DECR on available count. If DECR returns >= 0, ticket granted. If < 0, INCR back and return sold out. Much simpler than seat-level management.

**Q: How do you handle dynamic pricing?**
A: Price tiers based on demand signals: queue depth, sell-through rate, time to event. Pricing engine updates prices every 5 minutes. Price locked when hold is created (no change during checkout).

**Q: How do you prevent scalpers?**
A: Multi-factor: purchase limits (4 per account), phone verification, ID verification at entry for high-demand events, non-transferable tickets, delayed ticket delivery (closer to event date to reduce resale value window).

**Q: What happens if the entire system goes down during an on-sale?**
A: Pause the on-sale time. All active holds are preserved in DB (durable). Queue state is reconstructable from access tokens. Resume when system is healthy. Communicate via social media and email.

**Q: How do you handle seat selection UX when seats are being grabbed quickly?**
A: Seat map refreshes every 2 seconds via polling or WebSocket. When user clicks a seat, immediate optimistic hold attempt. If seat was just taken, instant feedback to select another. Show "limited availability" indicators on sections.

## 16) Closing Summary (30-60s)

"I've designed a ticket booking system that handles the extreme concurrency of popular event on-sales. The virtual queue randomizes and rate-limits admission to prevent thundering herd. The inventory service uses PostgreSQL with `FOR UPDATE SKIP LOCKED` for contention-free seat locking, ensuring no overselling. Holds are time-limited (8 minutes) with Redis TTL expiration, automatically releasing abandoned seats back to inventory. The system handles 500K QPS during peak on-sales through CDN caching of event pages, Redis-cached seat maps, and queue-controlled admission rates calibrated to inventory and checkout capacity."

## Appendix A: Configuration / Tuning Knobs

| Knob | Default | Range | Effect |
|------|---------|-------|--------|
| hold_duration_min | 8 | 5 - 15 | Longer = more time for checkout, slower inventory turnover |
| queue_admission_rate | 500/sec | 100 - 5000 | Higher = more concurrent users at seat selection |
| max_tickets_per_user | 4 | 1 - 8 | Higher = more tickets per user, fewer unique buyers |
| seat_map_cache_ttl_sec | 2 | 1 - 10 | Lower = more accurate display, more DB load |
| queue_randomization | true | true/false | False = strict FIFO, true = randomized for anti-bot |
| hold_expiration_check_sec | 10 | 5 - 30 | How often hold expiration worker runs |
| captcha_threshold | MEDIUM | LOW/MEDIUM/HIGH | Higher = more friction, fewer bots |
| queue_start_before_onsale_min | 30 | 10 - 60 | How early users can enter the queue |

## Appendix B: Vocabulary Cheat Sheet

| Term | Definition |
|------|------------|
| Hold | Temporary reservation of seats during checkout (TTL-based) |
| Virtual Queue | Waiting room that controls rate of user access to seat selection |
| SKIP LOCKED | PostgreSQL clause that skips rows already locked by other transactions |
| Optimistic Concurrency | Using version numbers to detect conflicting updates |
| Thundering Herd | Millions of users hitting the system simultaneously at on-sale time |
| Seat Map | Visual representation of venue with seat availability |
| On-Sale | The moment tickets become available for purchase |
| Idempotency Key | Unique key ensuring a payment is processed exactly once |
| General Admission | Tickets without assigned seats; counter-based inventory |
| Access Token | JWT issued by queue granting permission to browse and purchase |

## Appendix C: References

- Ticketmaster engineering blog: handling high-demand on-sales
- "Designing Data-Intensive Applications" by Martin Kleppmann (Chapter 7: Transactions)
- PostgreSQL SKIP LOCKED documentation
- Redis Sorted Sets for queue management
- Stripe idempotent payments documentation
- Cloudflare Waiting Room (virtual queue product)
- Queue-it virtual waiting room architecture
