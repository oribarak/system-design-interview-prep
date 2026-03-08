# Amazon E-Commerce -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
We need to design the backend for a large-scale e-commerce platform handling 500M products, 200M daily active users, and 10M orders per day (100M during peak events like Prime Day). The hard parts are preventing overselling during flash sales, sub-300ms product search, and coordinating checkout across inventory, payment, and fulfillment services.

## 2. Requirements
**Functional:** Product catalog with search and faceted filtering. Persistent shopping cart. Checkout with inventory reservation, payment authorization, and order creation. Order tracking and status updates. Real-time inventory across multiple fulfillment centers.

**Non-functional:** 99.99% availability for product pages and search. Product page under 200ms, search under 300ms, checkout under 3 seconds. Strong consistency for inventory (no overselling). Handle 100M orders on peak days. Zero order loss.

## 3. Core Concept: Saga-Based Checkout with Inventory Reservation
Checkout is a distributed transaction across inventory, payment, and order services. Instead of two-phase commit (which blocks on coordinator failure), we use the Saga pattern: a sequence of local transactions with compensating actions. First, inventory is reserved via an atomic conditional update. Then payment is authorized. Then the order is created. If any step fails, previous steps are rolled back (release inventory, void payment). Inventory reservation has a 15-minute TTL to automatically release abandoned checkouts.

## 4. High-Level Architecture
```
Client --> API Gateway --> Product/Search Service (Elasticsearch)
                      --> Cart Service (DynamoDB)
                      --> Order Service (Saga Orchestrator)
                              |
                    +---------+---------+
                    |         |         |
              Inventory   Payment   Fulfillment
              (PostgreSQL) (Gateway)  (Routing)
                              |
                           Kafka --> Notifications / Analytics
```
- **Search Service**: 50+ node Elasticsearch cluster for full-text search with faceted filtering.
- **Cart Service**: DynamoDB for cross-session persistent cart with fast reads/writes.
- **Order Service**: Orchestrates the checkout saga -- inventory reservation, payment auth, order creation.
- **Inventory Service**: Sharded PostgreSQL with row-level locking for atomic stock decrements.

## 5. How Checkout Works (The Saga)
1. User clicks "Place Order"; Order Service validates cart items are still available and prices match.
2. Inventory Service atomically decrements available_quantity and increments reserved_quantity (conditional: only if available >= requested).
3. Payment Service authorizes (not captures) the total amount via the payment gateway.
4. Order Service creates the order record with status CONFIRMED.
5. Order event published to Kafka, triggering fulfillment routing and email confirmation.
6. If payment fails: compensating transaction releases the inventory reservation.
7. If inventory reservation fails: return "out of stock" immediately -- no compensation needed.

## 6. What Happens When Things Fail?
- **Checkout partial failure**: Saga compensating transactions undo completed steps. Idempotency keys prevent double-charging on retries.
- **Inventory service down**: Cart shows "unable to verify stock." Checkout is blocked until recovery. Reserved inventory TTL auto-releases abandoned reservations.
- **Search degradation**: Fall back to category browsing. Cache recent popular search results.

## 7. Scaling
- **10x (100M orders/day consistently)**: Virtual inventory partitions for hot products (split 1000 units into 10 partitions of 100 for parallel checkouts). Queue-based checkout for all high-traffic periods. Multi-region active-active.
- **100x**: CQRS with separate read/write models. Event sourcing for orders. Edge caching for catalog pages. Predictive inventory pre-positioning.

## 8. Key Trade-offs to Mention
| Decision | Trade-off |
|---|---|
| Saga pattern vs. two-phase commit | More resilient to coordinator failure and avoids distributed locks, but requires explicit compensating transactions for every step |
| DynamoDB for cart vs. Redis | Persists across sessions and survives node failures, but slightly higher latency than Redis |
| Elasticsearch for search vs. SQL | Excels at faceted search and horizontal scaling for 500M products, but requires a separate indexing pipeline with ~5s lag |

## 9. Closing (30s)
> We designed an Amazon-scale backend with 500M products and 10M daily orders. Search is powered by Elasticsearch with learning-to-rank. Carts persist in DynamoDB. Checkout uses an orchestrated Saga: atomic inventory reservation via PostgreSQL row locks, payment authorization, and order creation with compensating transactions on failure. Flash sales use Redis-backed soft checks and queue-based checkout. Orders flow through Kafka to fulfillment and notifications.
