# Ticket Booking System (Ticketmaster) — Cheat Sheet

## Key Numbers
- 500K QPS peak during hot on-sale (10M concurrent users)
- 50K seats per large event
- Hold duration: 8 minutes with Redis TTL
- Queue admission rate: ~500 users/sec (calibrated to inventory and checkout capacity)
- Seat map cache TTL: 2 seconds
- Hold acquisition latency: < 200ms
- 2M tickets sold/day across 5,000 events

## Core Components
- **Virtual Queue**: Redis Sorted Set with randomized positions, CAPTCHA on join, controlled admission rate
- **Inventory Service**: PostgreSQL with `FOR UPDATE SKIP LOCKED` for contention-free seat locking
- **Hold System**: Redis TTL for automatic expiration + DB as source of truth
- **Hold Expiration Worker**: releases expired holds back to AVAILABLE, triggers queue re-admission
- **Seat Map Service**: cached availability view (2-sec TTL), refreshed from inventory DB
- **Order Service**: converts hold to order with idempotent payment processing

## Architecture in One Sentence
A virtual queue rate-limits access to seat selection, PostgreSQL SKIP LOCKED provides contention-free seat holds, Redis TTL auto-expires abandoned holds, and the admission rate is calibrated so demand never exceeds inventory service capacity.

## Top 3 Trade-offs
1. **PostgreSQL row locks vs Redis for inventory**: PostgreSQL gives ACID guarantees for financial-grade consistency; Redis can't
2. **Randomized queue vs FIFO**: randomized prevents bot advantage from network proximity but feels less "fair"
3. **Short hold TTL (8 min) vs long (15 min)**: short turns over inventory faster but pressures slow buyers

## Scaling Story
- 10x: shard inventory DB by event_id, read replicas for seat maps, per-event Redis instances
- 100x: bitmap-based seat maps (50K seats = 6.25 KB), pre-partitioned sections, dedicated infrastructure for mega-events

## Closing Statement
"Virtual queue controls admission rate to prevent thundering herd. PostgreSQL SKIP LOCKED ensures contention-free seat holds with zero overselling. Redis TTL auto-expires abandoned holds. Queue admission calibrated to inventory and checkout throughput. System handles 500K QPS peak through CDN caching, queue throttling, and sharded inventory."
