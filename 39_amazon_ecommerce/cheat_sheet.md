# Amazon E-Commerce Backend -- Cheat Sheet

## Key Numbers
- 200M DAU, 500M products, 2B product page views/day
- 10M orders/day (normal), 100M orders/day (Prime Day peak)
- Search: 500M queries/day, ~5,800 avg QPS
- Cart: 500M updates/day in DynamoDB
- Inventory: ~3,500 atomic updates/sec at peak
- Checkout target: < 3 seconds end-to-end

## Core Components
- **Product Catalog Service**: Sharded MySQL + Redis cache for product data
- **Search Service**: 50+ node Elasticsearch with BM25 + Learning to Rank
- **Cart Service**: DynamoDB for cross-session persistence
- **Order Service**: Saga orchestrator for checkout flow
- **Inventory Service**: Sharded PostgreSQL with row-level locking (no overselling)
- **Payment Service**: PCI-compliant gateway integration (authorize then capture)
- **Fulfillment Service**: Warehouse routing and shipping coordination
- **Kafka**: Event bus for order events, CDC to search, analytics

## Architecture in One Sentence
A microservices e-commerce platform with Elasticsearch-powered search, DynamoDB carts, orchestrated Saga-pattern checkout with atomic PostgreSQL inventory reservation and payment authorization, and Kafka-driven fulfillment and notification pipelines.

## Top 3 Trade-offs
1. **Saga vs. 2PC for checkout**: Saga is more resilient but requires compensating transactions for rollback
2. **DynamoDB cart vs. Redis**: DynamoDB persists across sessions but Redis is faster (volatile)
3. **Strong consistency for inventory vs. eventual**: Strong prevents overselling but limits throughput under contention

## Scaling Story
- 10x: More DB shards, virtual inventory partitions for hot products, queue-based checkout, multi-region
- 100x: CQRS with event sourcing, edge-cached catalog, predictive inventory placement, one-click pre-authorized checkout

## Closing Statement
A global e-commerce platform with 500M searchable products, Saga-pattern checkout ensuring zero overselling through atomic inventory reservation, payment authorization with idempotent retries, and Kafka-driven fulfillment -- serving 200M DAU with sub-3-second checkout and scaling to 100M orders on peak days.
