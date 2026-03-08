# System Design Interview: Amazon E-Commerce Backend -- "Perfect Answer" Playbook

---

## 0) Opening: Frame the Problem (30-60s)

> "We need to design the backend for a large-scale e-commerce platform like Amazon, handling product catalog, search, shopping cart, checkout, order management, and inventory. The core challenges are maintaining inventory consistency during high-traffic flash sales, providing sub-second search across hundreds of millions of products, processing millions of orders per day with reliable payment integration, and coordinating across multiple fulfillment centers. I will cover the product catalog, search, cart, checkout/ordering pipeline, inventory management, and fulfillment."

---

## 1) Clarifying Questions (2-5 min)

**Scope**
- Are we designing the customer-facing e-commerce backend, or also the seller/merchant portal?
- Should we include the recommendation engine, or focus on the transactional path (browse, cart, checkout, order)?
- Do we need to handle marketplace (third-party sellers) or just first-party inventory?

**Scale**
- How many products in the catalog? (Assume 500 million SKUs.)
- How many daily active users? (Assume 200 million DAU.)
- How many orders per day? (Assume 10 million orders/day, with peak at 100M during Prime Day.)

**Policy / Constraints**
- Consistency requirements for inventory? (No overselling -- strong consistency for inventory decrement.)
- Payment processing latency? (Checkout completion < 3 seconds.)
- Multi-region? (Yes -- global with regional fulfillment centers.)

---

## 2) Back-of-Envelope Estimation (3-5 min)

| Metric | Value |
|---|---|
| Products (SKUs) | 500,000,000 |
| Daily active users | 200,000,000 |
| Product page views per day | 2,000,000,000 |
| Product view QPS (avg) | 2B / 86,400 = ~23,000 QPS |
| Product view QPS (peak, 5x) | ~115,000 QPS |
| Search queries per day | 500,000,000 |
| Search QPS (avg) | ~5,800 QPS |
| Orders per day (normal) | 10,000,000 |
| Orders per day (peak, Prime Day) | 100,000,000 |
| Order QPS (peak) | 100M / 86,400 = ~1,157 QPS |
| Avg items per order | 3 |
| Inventory updates per second (peak) | 1,157 x 3 = ~3,500 QPS |
| Cart updates per day | 500,000,000 |
| Cart update QPS (avg) | ~5,800 QPS |

---

## 3) Requirements (3-5 min)

### Functional
- **Product catalog**: Browse, filter, and view detailed product pages with pricing, availability, reviews, and seller info.
- **Search**: Full-text search with faceted filtering (category, price range, brand, rating, Prime eligibility).
- **Shopping cart**: Add/remove items, persist cart across sessions and devices.
- **Checkout**: Address selection, payment processing, order placement with inventory reservation.
- **Order management**: Order tracking, status updates, cancellation, returns.
- **Inventory management**: Real-time inventory tracking across multiple fulfillment centers; no overselling.

### Non-functional
- **Availability**: 99.99% for product pages and search; 99.9% for checkout.
- **Latency**: Product page < 200ms; search < 300ms; checkout < 3 seconds.
- **Consistency**: Strong consistency for inventory and orders; eventual consistency acceptable for product catalog and reviews.
- **Scalability**: Handle 100M orders on peak days (10x normal).
- **Durability**: Zero order loss; all financial transactions persisted.
- **Fault tolerance**: Checkout must complete even if some auxiliary services (recommendations, reviews) are degraded.

---

## 4) API Design (2-4 min)

### Search Products
```
GET /api/v1/search?q={query}&category={cat}&min_price={p}&max_price={p}&sort={relevance|price|rating}&page_token={token}

Response: {
  results: [{ product_id, title, price, rating, review_count, image_url, is_prime, in_stock }],
  facets: { categories: [...], brands: [...], price_ranges: [...] },
  total_results: 12345,
  next_page_token
}
```

### Get Product Details
```
GET /api/v1/products/{product_id}

Response: {
  product_id, title, description, price, currency,
  images: [url], category, brand,
  rating: 4.5, review_count: 12345,
  variants: [{ variant_id, color, size, price, in_stock }],
  sellers: [{ seller_id, name, price, shipping, fulfillment_type }],
  delivery_estimate: "Tomorrow"
}
```

### Cart Operations
```
POST /api/v1/cart/items
Body: { product_id, variant_id, quantity }
Response: { cart: { items: [...], subtotal, item_count } }

PUT /api/v1/cart/items/{item_id}
Body: { quantity }

DELETE /api/v1/cart/items/{item_id}
```

### Place Order (Checkout)
```
POST /api/v1/orders
Body: {
  cart_id,
  shipping_address_id,
  payment_method_id,
  shipping_option: "standard"|"express"|"same_day"
}

Response: { order_id, status: "CONFIRMED", estimated_delivery, total, items: [...] }
```

### Get Order Status
```
GET /api/v1/orders/{order_id}

Response: {
  order_id, status, items: [{ product_id, title, quantity, price, status }],
  shipping: { carrier, tracking_number, estimated_delivery },
  payment: { method, amount, status }
}
```

---

## 5) Data Model (3-5 min)

### Products Table
| Column | Type | Notes |
|---|---|---|
| product_id | STRING (PK) | ASIN-like identifier |
| title | STRING | Searchable |
| description | TEXT | |
| category_id | STRING (FK) | Hierarchical category |
| brand | STRING | |
| base_price | DECIMAL | |
| currency | STRING | |
| images | JSON | Array of image URLs |
| attributes | JSON | Color, size, material, etc. |
| rating_avg | FLOAT | Denormalized |
| review_count | INT | Denormalized |
| status | ENUM | ACTIVE, INACTIVE, DISCONTINUED |
| created_at | TIMESTAMP | |

### Inventory Table (per SKU per fulfillment center)
| Column | Type | Notes |
|---|---|---|
| sku_id | STRING (PK) | Product variant identifier |
| fulfillment_center_id | STRING (PK) | Which warehouse |
| available_quantity | INT | Current sellable stock |
| reserved_quantity | INT | Reserved during checkout, not yet shipped |
| reorder_point | INT | Threshold for restocking |
| updated_at | TIMESTAMP | Optimistic locking |

### Cart Table
| Column | Type | Notes |
|---|---|---|
| user_id | STRING (PK) | Partition key |
| item_id | STRING (SK) | Sort key (product+variant) |
| product_id | STRING | |
| variant_id | STRING | |
| quantity | INT | |
| added_at | TIMESTAMP | |
| price_at_addition | DECIMAL | Snapshot for price change detection |

### Orders Table
| Column | Type | Notes |
|---|---|---|
| order_id | STRING (PK) | Globally unique |
| user_id | STRING | Indexed |
| status | ENUM | PENDING, CONFIRMED, PROCESSING, SHIPPED, DELIVERED, CANCELLED, RETURNED |
| total_amount | DECIMAL | |
| currency | STRING | |
| shipping_address | JSON | Snapshot at order time |
| payment_method_id | STRING | |
| payment_status | ENUM | PENDING, AUTHORIZED, CAPTURED, REFUNDED |
| placed_at | TIMESTAMP | |
| updated_at | TIMESTAMP | |

### Order Items Table
| Column | Type | Notes |
|---|---|---|
| order_id | STRING (PK) | |
| item_id | STRING (PK) | |
| product_id | STRING | |
| variant_id | STRING | |
| quantity | INT | |
| unit_price | DECIMAL | Price at order time |
| fulfillment_center_id | STRING | Assigned warehouse |
| status | ENUM | PENDING, PICKED, PACKED, SHIPPED, DELIVERED |

**Storage choices**: Product catalog in sharded MySQL/Vitess (shard by product_id). Inventory in sharded PostgreSQL with row-level locking (shard by sku_id). Cart in DynamoDB (key-value, fast reads/writes). Orders in sharded PostgreSQL (shard by order_id, secondary index on user_id). Search index in Elasticsearch. Product cache in Redis/Memcached.

---

## 6) High-Level Architecture (5-8 min)

### Dataflow
1. **User browses** products via the **Product Catalog Service** (reads from cache -> database -> search).
2. **Search Service** (Elasticsearch) handles full-text queries with faceted filtering.
3. **Cart Service** persists cart in DynamoDB. Cart is validated (stock check, price check) before checkout.
4. **Checkout** triggers an **Order Service** that orchestrates: inventory reservation, payment authorization, order creation, and fulfillment assignment.
5. **Inventory Service** uses optimistic locking / row-level locks to decrement stock atomically. Failed inventory checks return "out of stock."
6. **Payment Service** integrates with payment gateways (Stripe, bank APIs) for authorization and capture.
7. **Order Service** publishes order events to **Kafka**, consumed by fulfillment, notification, and analytics systems.
8. **Fulfillment Service** routes order items to the optimal warehouse, triggers picking/packing, and generates shipping labels.
9. **Notification Service** sends order confirmation, shipping updates, and delivery notifications.

### Components
- **API Gateway**: Auth, rate limiting, request routing.
- **Product Catalog Service**: Product CRUD, pricing, reviews aggregation.
- **Search Service**: Elasticsearch-based product search with ranking.
- **Cart Service**: DynamoDB-backed persistent cart.
- **Order Service**: Orchestrates checkout flow; owns order lifecycle.
- **Inventory Service**: Atomic stock management with strong consistency.
- **Payment Service**: Payment gateway integration, authorization, capture, refund.
- **Fulfillment Service**: Warehouse assignment, shipping optimization.
- **Notification Service**: Email, SMS, push notifications for order updates.
- **Recommendation Service**: "Customers also bought" and personalized suggestions.
- **Pricing Service**: Dynamic pricing, promotions, coupons.
- **Event Bus (Kafka)**: Asynchronous communication between services.
- **Cache (Redis/Memcached)**: Product data, session data, frequently accessed inventory.

> **One-sentence summary**: A microservices-based e-commerce platform where product search is powered by Elasticsearch, carts are persisted in DynamoDB, checkout orchestrates atomic inventory reservation and payment authorization, and order events flow through Kafka to fulfillment and notification services.

---

## 7) Deep Dive #1: Checkout and Inventory Consistency (8-12 min)

The checkout flow is the most critical path. It must handle concurrent purchases without overselling, process payments reliably, and recover gracefully from partial failures.

### Checkout Flow (Saga Pattern)
The checkout is a distributed transaction across multiple services. We use the **Saga pattern** (choreography or orchestration) instead of 2PC:

**Orchestrated Saga Steps**:
1. **Validate cart**: Check all items are still available and prices haven't changed.
2. **Reserve inventory**: For each item, decrement `available_quantity` and increment `reserved_quantity`. This is a conditional update: `UPDATE inventory SET available_quantity = available_quantity - :qty, reserved_quantity = reserved_quantity + :qty WHERE sku_id = :sku AND fulfillment_center_id = :fc AND available_quantity >= :qty`.
3. **Authorize payment**: Call payment gateway to authorize (not capture) the total amount.
4. **Create order**: Persist order record with status CONFIRMED.
5. **Publish order event**: Kafka event triggers fulfillment, notifications.

**Compensating transactions** (if any step fails):
- Payment authorization fails -> Release inventory reservation.
- Order creation fails -> Void payment authorization + release inventory.
- Inventory reservation fails (out of stock) -> Return error to user; no compensation needed.

### Inventory Reservation Model
```
Total stock = available_quantity + reserved_quantity + shipped_quantity

available_quantity: Can be sold
reserved_quantity: Held during checkout (TTL: 15 minutes)
shipped_quantity: In transit or delivered
```

- **Reservation TTL**: If payment or order creation fails, a background job releases expired reservations after 15 minutes.
- **Atomic decrement**: Use `SELECT ... FOR UPDATE` (PostgreSQL) or conditional writes (DynamoDB) to prevent race conditions.
- **Per-fulfillment-center**: Inventory is tracked per warehouse. The system selects the nearest fulfillment center with stock.

### Handling Flash Sales (100x normal traffic)
Flash sales create extreme inventory contention:
1. **Pre-warm inventory cache**: Before the sale, load inventory counts into Redis with atomic `DECR` operations.
2. **Tiered validation**: Redis provides a fast "is in stock?" check (soft check). If Redis says yes, proceed to the database for the authoritative atomic decrement.
3. **Queue-based checkout**: During extreme peaks, checkout requests enter a FIFO queue. Workers process checkouts sequentially per SKU, eliminating contention.
4. **Inventory segmentation**: Split inventory for a hot product across multiple "virtual" partitions (e.g., 1000 units split into 10 partitions of 100). Each partition handles independent concurrent checkouts.

### Idempotency
- Every checkout request includes a client-generated **idempotency key**.
- The Order Service checks if an order with this key already exists before processing.
- Prevents duplicate orders from network retries or double-clicks.

---

## 8) Deep Dive #2: Product Search (5-8 min)

Search must handle 500M queries/day across 500M products with sub-300ms latency and rich faceted filtering.

### Search Architecture
- **Elasticsearch cluster**: Sharded across 50+ nodes, 500M documents.
- **Index structure**: One primary index, sharded by product_id hash. Fields: title, description, category, brand, price, rating, attributes.
- **Ranking**: Combination of text relevance (BM25), product popularity (sales velocity), seller quality, and personalization features.

### Indexing Pipeline
1. Product catalog changes (CRUD) are published to Kafka as CDC events.
2. An **Indexer Service** consumes these events and updates Elasticsearch.
3. Near-real-time indexing lag: < 5 seconds from database change to searchable.
4. Full reindex (periodic, weekly): Rebuilds the entire index from the source of truth for drift correction.

### Query Processing
```
User query: "wireless bluetooth headphones under $50"
```
1. **Query parsing**: Extract structured filters (price < $50) and text query ("wireless bluetooth headphones").
2. **Query expansion**: Add synonyms ("earbuds", "earphones") and correct typos ("bluetoth" -> "bluetooth").
3. **Elasticsearch query**: Bool query with must (text match), filter (price range, in_stock), and should (boosting Prime, high-rated).
4. **Facet computation**: Aggregate by category, brand, price range for the filter sidebar.
5. **Result enrichment**: Fetch live pricing, availability, and delivery estimates from backend services.

### Search Relevance Tuning
- **A/B testing**: Continuously test ranking formula changes against conversion rate.
- **Learning to Rank (LTR)**: ML model trained on click-through data to re-rank search results.
- **Personalization**: Boost products from categories/brands the user has previously purchased.

### Autocomplete / Suggestions
- **Prefix-based**: Trie or Elasticsearch completion suggester for query suggestions.
- **Trending queries**: Track popular queries in Redis sorted sets; show as suggestions.
- **Latency target**: < 50ms for autocomplete responses.

---

## 9) Trade-offs and Alternatives (3-5 min)

| Decision | Chosen | Alternative | Rationale |
|---|---|---|---|
| Checkout consistency | Saga pattern (orchestrated) | Two-phase commit (2PC) | Saga is more resilient; 2PC blocks on coordinator failure |
| Inventory store | Sharded PostgreSQL with row locks | DynamoDB conditional writes | PostgreSQL supports complex queries needed for multi-FC routing |
| Cart storage | DynamoDB | Redis / Session store | DynamoDB persists across sessions; Redis is faster but volatile |
| Search engine | Elasticsearch | Solr / custom inverted index | Elasticsearch excels at faceted search and horizontal scaling |
| Inter-service communication | Kafka (async) + gRPC (sync) | All REST / all async | Sync for checkout flow (latency-sensitive); async for fulfillment (decoupled) |

---

## 10) Scaling: 10x and 100x (3-5 min)

### 10x (100M orders/day consistently)
- **Database sharding**: Increase order DB shards. Separate read replicas for order history queries.
- **Search cluster**: Scale Elasticsearch to 200+ nodes. Add caching layer for popular queries.
- **Inventory hot partitioning**: Virtual inventory partitions for top 1000 products.
- **Multi-region**: Active-active in 3+ regions. Regional databases with cross-region replication for catalog.
- **Checkout queuing**: Queue-based checkout for all high-traffic periods, not just flash sales.

### 100x (1B orders/day -- hypothetical)
- **CQRS everywhere**: Separate read and write models. Read-optimized materialized views served from cache.
- **Event sourcing for orders**: Instead of mutable order records, store an append-only event log. Rebuild order state by replaying events.
- **Edge caching for catalog**: Push product data to CDN edge; serve product pages entirely from edge.
- **Predictive inventory**: ML models predict demand and pre-position inventory at fulfillment centers.
- **Autonomous checkout**: Pre-authorized payment methods + pre-validated addresses enable one-click checkout with no synchronous validations.

---

## 11) Reliability and Fault Tolerance (3-5 min)

- **Checkout partial failure**: Saga compensating transactions undo completed steps. Idempotency keys prevent double-charging.
- **Inventory service failure**: Cart shows "unable to verify stock" but allows users to continue browsing. Checkout is blocked until inventory service recovers. Reserved inventory TTL ensures automatic release.
- **Payment service failure**: Retry with exponential backoff. If the gateway is unreachable, hold the order in PENDING status and retry asynchronously. Notify user to try again or wait.
- **Search degradation**: Fallback to category browsing if Elasticsearch is down. Cache recent popular search results.
- **Database failure**: Synchronous replication with automatic failover. RPO = 0 for orders and inventory. Product catalog uses read replicas; primary failure causes brief write unavailability.
- **Kafka failure**: Producers buffer locally and retry. Consumers track offsets for exactly-once processing. Multi-AZ Kafka cluster with RF=3.

---

## 12) Observability and Operations (2-4 min)

- **Checkout success rate**: Percentage of checkout attempts that result in a confirmed order. Primary business metric.
- **Checkout latency**: p50, p95, p99. Breakdown by step (inventory, payment, order creation).
- **Cart abandonment rate**: Users who add to cart but do not checkout. Track reasons (out of stock, slow checkout).
- **Search latency and click-through rate (CTR)**: Track per query. Low CTR indicates poor relevance.
- **Inventory accuracy**: Compare system inventory vs. physical counts. Alert on discrepancies.
- **Order processing pipeline**: Time from order placement to shipment. Alert if SLA breached.
- **Error rates by service**: Track 5xx per microservice. Circuit breaker states.
- **Revenue metrics**: Orders per minute, GMV per hour (real-time dashboard for business health).

---

## 13) Security (1-3 min)

- **Authentication**: OAuth2 / JWT tokens for user sessions. MFA for account changes.
- **Payment security**: PCI DSS compliance. Payment details tokenized; never stored on our servers. Use payment gateway's vault.
- **Authorization**: RBAC for seller operations. Users can only access their own orders and cart.
- **Fraud detection**: ML model scores each order for fraud risk (unusual shipping address, velocity checks, device fingerprinting). High-risk orders held for review.
- **Data encryption**: TLS in transit. AES-256 at rest for PII and payment data.
- **Rate limiting**: Per-user cart update limits (prevent bot scalping). CAPTCHA on checkout during flash sales.
- **Inventory abuse prevention**: Limit reservation hold time (15 min). Prevent cart-holding attacks.

---

## 14) Team and Operational Considerations (1-2 min)

- **Catalog team**: Product data management, seller onboarding, category taxonomy.
- **Search team**: Elasticsearch cluster, ranking algorithms, autocomplete, A/B testing.
- **Cart/Checkout team**: Cart service, checkout orchestration, payment integration.
- **Inventory team**: Inventory service, warehouse integration, stock reconciliation.
- **Fulfillment team**: Order routing, shipping optimization, carrier integration.
- **Platform/Infra team**: Kafka, databases, caching, API gateway.
- **Deployment**: Feature flags for flash sale configurations. Canary deployments for checkout changes (highest risk).

---

## 15) Common Follow-up Questions

**Q: How do you handle dynamic pricing?**
A: A Pricing Service manages base prices, promotions, coupons, and dynamic adjustments (demand-based, competitor-matching). Cart and checkout always fetch the latest price from the Pricing Service, not from a cached value. The "price at addition" is shown to the user with a note if it changed.

**Q: How do you prevent bot scalping during flash sales?**
A: (1) CAPTCHA on checkout. (2) Device fingerprinting + behavioral analysis to detect bots. (3) Queue-based checkout with randomized entry to prevent fastest-bot-wins. (4) Per-account purchase limits for hot items.

**Q: How does the recommendation engine integrate?**
A: "Customers also bought" uses collaborative filtering on purchase history. The recommendation service reads from the data warehouse (populated by Kafka order events). Results are precomputed and cached; served as a widget on product pages with a separate API call. If the recommendation service is down, the widget is hidden (graceful degradation).

**Q: How do you handle returns and refunds?**
A: Return request creates a RETURN_REQUESTED status. Once the item is received at the warehouse, the Inventory Service increments available_quantity. The Payment Service issues a refund (full or partial). An event is published to update the order status and notify the user.

**Q: How do you handle multi-currency and international orders?**
A: Products have a base currency. The Pricing Service converts to the user's local currency using real-time exchange rates (cached with 1-hour TTL). The order stores both the display currency amount and the base currency amount. Payment is charged in the display currency.

---

## 16) Closing Summary (30-60s)

> "We designed an Amazon-scale e-commerce backend serving 200M DAU with 500M products and 10M daily orders (100M on peak days). The product catalog is served from sharded MySQL with Redis caching, and search is powered by a 50+ node Elasticsearch cluster with BM25 + learning-to-rank. Carts are persisted in DynamoDB for cross-session durability. Checkout uses an orchestrated Saga pattern: atomic inventory reservation (PostgreSQL row locks), payment authorization, and order creation. Flash sales are handled with Redis-backed inventory soft checks and queue-based checkout. Orders flow through Kafka to fulfillment and notification services. The system is globally distributed with regional fulfillment centers and multi-region databases."

---

## Appendix A: Configuration / Tuning Knobs

| Knob | Default | Tune When |
|---|---|---|
| `inventory_reservation_ttl` | 15 min | Shorter for flash sales; longer for B2B orders |
| `search_result_cache_ttl` | 60s | Lower for fast-moving inventory; higher for stable catalog |
| `checkout_timeout` | 30s | Longer during payment gateway slowdowns |
| `max_cart_items` | 100 | Adjust per business policy |
| `flash_sale_queue_size` | 10,000 | Based on expected concurrent demand |
| `inventory_virtual_partitions` | 10 | More for hotter products |
| `elasticsearch_shards` | 50 | Increase with catalog growth |
| `order_event_retention` | 7 days (Kafka) | Longer for replay capability |
| `fraud_score_threshold` | 0.7 | Lower catches more fraud; higher reduces false positives |

## Appendix B: Vocabulary Cheat Sheet

| Term | Definition |
|---|---|
| **SKU** | Stock Keeping Unit; unique identifier for a specific product variant |
| **Saga Pattern** | Distributed transaction pattern using a sequence of local transactions with compensating actions |
| **Idempotency Key** | Client-generated unique ID to prevent duplicate processing on retries |
| **Inventory Reservation** | Temporarily holding stock during checkout; released if order is not completed |
| **Flash Sale** | Time-limited high-discount event causing traffic spikes (10-100x normal) |
| **Fulfillment Center** | Warehouse where orders are picked, packed, and shipped |
| **GMV** | Gross Merchandise Value; total value of goods sold |
| **BM25** | Probabilistic text relevance scoring function used by Elasticsearch |
| **Learning to Rank** | ML model that re-ranks search results based on user behavior data |
| **CQRS** | Command Query Responsibility Segregation; separate read and write data models |
| **PCI DSS** | Payment Card Industry Data Security Standard; compliance for handling card data |

## Appendix C: References

- Amazon Dynamo: Highly Available Key-value Store (2007)
- Amazon Builder's Library: Avoiding fallback in distributed systems
- Saga Pattern: Microservices Patterns by Chris Richardson
- Elasticsearch: The Definitive Guide
- Event-Driven Architecture for E-Commerce
- PCI DSS Compliance for Cloud-Native Applications
