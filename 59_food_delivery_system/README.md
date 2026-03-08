# System Design Interview: Food Delivery System (DoorDash) -- "Perfect Answer" Playbook

## 0) Opening: Frame the Problem (30-60s)

"We are designing a food delivery platform that connects customers, restaurants, and delivery drivers. The system must handle restaurant discovery and search, real-time order management, driver dispatch and routing, and live order tracking. I will focus on the order lifecycle, the dispatch algorithm for matching drivers to orders, and the real-time location tracking system that ties everything together."

## 1) Clarifying Questions (2-5 min)

1. **Scope**: Full platform or specific subsystem? -- Full platform with deep dives on dispatch and tracking.
2. **Geography**: Single city or multi-region? -- Multi-city, national scale.
3. **Order volume**: How many orders per day? -- 10 million orders/day.
4. **Restaurant count**: How many active restaurants? -- 500,000.
5. **Driver count**: How many concurrent drivers? -- 200,000 peak concurrent.
6. **Menu management**: Do restaurants manage their own menus? -- Yes, via a merchant portal.
7. **Payment**: In scope? -- Briefly; assume integration with a payment gateway.
8. **Real-time tracking**: Must customers see driver location in real time? -- Yes, with location updates every 5 seconds.
9. **ETA accuracy**: How important is delivery time estimation? -- Critical for customer experience and driver dispatch.

## 2) Back-of-Envelope Estimation (3-5 min)

| Metric | Estimate |
|---|---|
| Daily orders | 10M |
| Peak orders/hour (lunch/dinner) | 2M (peak factor ~5x average) |
| Peak orders/sec | 2M / 3600 ~ 556 OPS |
| Concurrent active orders | ~500K (avg order lifecycle 45 min) |
| Active drivers (peak) | 200K |
| Driver location updates | 200K x 1 update/5s = 40K updates/sec |
| Customer tracking requests | 500K active orders x 1 req/10s = 50K reads/sec |
| Restaurant count | 500K |
| Average order size | $30 |
| Daily GMV | $300M |
| Menu items per restaurant | ~100 |
| Total menu items | 50M |

## 3) Requirements (3-5 min)

### Functional Requirements
- **Restaurant Discovery**: Search and browse restaurants by location, cuisine, rating, delivery time.
- **Menu Browsing**: View restaurant menus with items, prices, modifiers, availability.
- **Order Placement**: Cart management, order submission, payment processing.
- **Order Management**: Restaurant receives orders, confirms, marks items as preparing/ready.
- **Driver Dispatch**: Automatically match available drivers to orders based on proximity, ETA, and capacity.
- **Real-Time Tracking**: Live driver location on a map for the customer.
- **ETA Prediction**: Accurate estimated delivery time at order placement and during delivery.
- **Notifications**: Push notifications for order status changes (confirmed, picked up, arriving).

### Non-Functional Requirements
- **Latency**: Restaurant search < 200 ms; order placement < 1 s; location update processing < 1 s.
- **Availability**: 99.99% for order placement; 99.9% for tracking.
- **Consistency**: Orders and payments must be strongly consistent (no double charges, no lost orders).
- **Scalability**: Handle 10x order volume during promotions or holidays.
- **Location freshness**: Driver position visible to customer within 5-10 seconds of actual position.

## 4) API Design (2-4 min)

### Restaurant Search
```
GET /restaurants?lat=<lat>&lng=<lng>&cuisine=<type>&sort=<rating|eta|distance>&page=<n>
Response: { restaurants: [{ id, name, cuisine, rating, eta_minutes, distance_km, image_url }] }
```

### Place Order
```
POST /orders
Body: {
  customer_id, restaurant_id,
  items: [{ item_id, quantity, modifiers: [] }],
  delivery_address: { lat, lng, address_text },
  payment_method_id, tip_amount
}
Response: { order_id, estimated_delivery_time, total_amount }
```

### Order Status
```
GET /orders/{order_id}/status
Response: { order_id, status, driver: { id, name, lat, lng }, eta_minutes, timeline: [] }
```

### Driver Location Update (driver app)
```
POST /drivers/location
Body: { driver_id, lat, lng, heading, speed, timestamp }
```

### Driver Accept/Reject
```
POST /drivers/orders/{order_id}/accept
POST /drivers/orders/{order_id}/reject
```

## 5) Data Model (3-5 min)

### Core Entities

**Orders**
| Field | Type | Notes |
|---|---|---|
| order_id | uuid | Primary key |
| customer_id | uuid | FK |
| restaurant_id | uuid | FK |
| driver_id | uuid | FK, assigned after dispatch |
| status | enum | PLACED, CONFIRMED, PREPARING, READY, PICKED_UP, DELIVERING, DELIVERED, CANCELLED |
| items | json | Order items with modifiers |
| delivery_address | point + text | Lat/lng + address string |
| total_amount | decimal | |
| created_at | timestamp | |
| estimated_delivery_at | timestamp | |

**Restaurants**
| Field | Type | Notes |
|---|---|---|
| restaurant_id | uuid | Primary key |
| name | string | |
| location | point | Lat/lng |
| cuisine_type | string[] | |
| rating | float | Aggregate rating |
| operating_hours | json | Per-day open/close times |
| avg_prep_time_min | int | Historical average |

**Drivers**
| Field | Type | Notes |
|---|---|---|
| driver_id | uuid | Primary key |
| status | enum | AVAILABLE, EN_ROUTE_PICKUP, AT_RESTAURANT, DELIVERING, OFFLINE |
| current_location | point | Latest lat/lng |
| current_order_ids | uuid[] | Batched orders |
| vehicle_type | enum | Car, bike, scooter |

### Storage Choices
- **Orders**: PostgreSQL with sharding by customer_id. Strong consistency for order state transitions.
- **Restaurants / Menus**: PostgreSQL for metadata; Elasticsearch for search and filtering.
- **Driver locations**: Redis with geospatial indexing (GEOADD/GEOSEARCH) for real-time proximity queries.
- **Order events**: Kafka for event streaming (status changes, location updates).
- **Analytics**: Data warehouse (BigQuery/Snowflake) for historical analysis.

## 6) High-Level Architecture (5-8 min)

**One-sentence summary**: Customers search restaurants via an Elasticsearch-backed service, place orders through a strongly consistent order service, a dispatch engine matches drivers using geospatial proximity from Redis, and a real-time tracking system streams driver locations to customers via WebSocket connections.

### Components

1. **API Gateway** -- Rate limiting, auth, routing.
2. **Restaurant Service** -- Restaurant metadata, menus, search (backed by Elasticsearch).
3. **Order Service** -- Order lifecycle management, state machine, idempotent order creation.
4. **Payment Service** -- Integrates with payment gateway; handles authorization, capture, refunds.
5. **Dispatch Service** -- Matches drivers to orders; runs the dispatch algorithm.
6. **Driver Location Service** -- Ingests driver GPS updates, stores in Redis geospatial index.
7. **ETA Service** -- Predicts delivery time using ML model (prep time + pickup travel + delivery travel).
8. **Tracking Service** -- Streams driver location to customers via WebSocket/SSE.
9. **Notification Service** -- Push notifications for order status changes.
10. **Event Bus (Kafka)** -- Decouples services; carries order events, location events, analytics events.

## 7) Deep Dive #1: Driver Dispatch System (8-12 min)

### The Dispatch Problem

When a restaurant marks an order as almost ready, the system must assign a driver. The goal is to minimize total delivery time while balancing driver utilization. This is a variant of the online bipartite matching problem.

### Dispatch Algorithm

**Batch Dispatch**: Rather than greedily assigning each order to the nearest driver, we batch orders and drivers in a time window (e.g., 30 seconds) and solve an assignment problem.

1. **Candidate Selection**: Query Redis GEOSEARCH for drivers within a radius (e.g., 5 km) of the restaurant. Filter by driver status (AVAILABLE or about to finish current delivery).
2. **Scoring**: For each (driver, order) pair, compute a score:
   ```
   score = w1 * pickup_eta + w2 * delivery_eta + w3 * driver_idle_time + w4 * order_wait_time
   ```
   - pickup_eta: Time for driver to reach restaurant (from ETA service, considers traffic)
   - delivery_eta: Time from restaurant to customer (routing API)
   - driver_idle_time: Penalize keeping drivers idle too long
   - order_wait_time: Penalize orders waiting too long (food getting cold)

3. **Assignment**: Solve the weighted bipartite matching using the Hungarian algorithm (for small batches) or a greedy approximation (for large batches). Output: optimal driver-order assignments.

4. **Offer to Driver**: Send assignment to driver app. Driver has 30 seconds to accept. If rejected, re-enter the order into the next dispatch batch.

### Order Batching (Stacking)

For efficiency, one driver can pick up multiple orders from the same restaurant or nearby restaurants:
- If two orders are at the same restaurant and both delivery addresses are in a similar direction, batch them.
- Constraint: second delivery must add no more than 10 minutes to the first order's ETA.
- Reduces cost per delivery by 20-30%.

### Handling Edge Cases

- **No drivers available**: Expand search radius incrementally (5km -> 8km -> 12km). If still none, queue the order and retry every 30 seconds. Notify customer of delay.
- **Driver cancels after acceptance**: Re-dispatch immediately with priority boost.
- **Restaurant delays**: If prep time exceeds estimate, delay driver dispatch to avoid driver idle time at restaurant.
- **Peak demand (surge)**: Increase driver incentives (surge pay). Show longer ETAs to customers. Temporarily expand delivery radius.

### Geospatial Index for Drivers

- Redis GEOADD stores each driver's location with their driver_id.
- GEOSEARCH finds drivers within radius of a given point.
- Updates: 40K writes/sec (200K drivers / 5 sec interval).
- Reads: ~500 dispatch queries/sec, each finding drivers within radius.
- Redis handles this easily; partition by city/region for multi-region scale.

## 8) Deep Dive #2: Real-Time Order Tracking (5-8 min)

### Architecture

1. **Driver App** sends GPS location every 5 seconds via HTTP POST to the Driver Location Service.
2. **Driver Location Service** writes to Redis (geospatial update) and publishes to Kafka topic `driver-locations`.
3. **Tracking Service** subscribes to `driver-locations`, filters for orders that have active tracking sessions.
4. **Customer App** opens a WebSocket connection to the Tracking Service when viewing order status.
5. **Tracking Service** pushes driver location updates to the connected customer in real time.

### WebSocket Management

- 500K concurrent active orders -> up to 500K simultaneous WebSocket connections.
- Tracking Service is horizontally scaled across many nodes.
- Use consistent hashing on order_id to route WebSocket connections, so each order's updates go to the correct server.
- If a server fails, clients reconnect and are routed to a new server.

### ETA Updates

- ETA is recomputed every time a driver location update is received.
- Uses a routing engine (OSRM or Google Maps API) to compute remaining travel time.
- Cache route segments to reduce API calls; recompute full route only when driver deviates significantly.
- Push updated ETA to customer along with driver position.

### Data Flow

```
Driver GPS -> Location Service -> Redis (geospatial) + Kafka
Kafka -> Tracking Service -> WebSocket -> Customer App
Kafka -> ETA Service -> recompute ETA -> push via Tracking Service
```

### Optimization

- **Location smoothing**: Filter GPS noise; interpolate between updates for smooth map animation on client.
- **Battery optimization**: Reduce GPS update frequency when driver is stationary (at restaurant).
- **Connection pooling**: Multiplex multiple tracking sessions per WebSocket server to reduce connection overhead.

## 9) Trade-offs and Alternatives (3-5 min)

| Decision | Chosen | Alternative | Rationale |
|---|---|---|---|
| Dispatch approach | Batch dispatch (30s windows) | Greedy nearest-driver | Batch produces globally better assignments; slight delay is acceptable |
| Driver location store | Redis geospatial | PostGIS | Redis is faster for high-frequency point updates and radius queries |
| Tracking protocol | WebSocket | SSE / Polling | WebSocket supports bidirectional communication; lower overhead than polling |
| Order database | PostgreSQL (sharded) | DynamoDB | Strong consistency needed for financial transactions; Postgres offers ACID |
| Search | Elasticsearch | PostgreSQL full-text + PostGIS | ES handles complex faceted search with geo-filtering much better at scale |

## 10) Scaling: 10x and 100x (3-5 min)

### 10x (100M orders/day)
- Shard Order Service by city/region. Each region operates semi-independently.
- Scale Redis geospatial index per city (one Redis cluster per major metro).
- Add read replicas for restaurant search (Elasticsearch).
- Dispatch Service runs per-city; no cross-city coordination needed.
- WebSocket servers scale horizontally; use Redis Pub/Sub for cross-server event routing.

### 100x (1B orders/day)
- Multi-region deployment with data sovereignty (EU data stays in EU).
- Pre-compute restaurant recommendations and ETAs during off-peak for popular areas.
- Move dispatch to ML-based assignment (reinforcement learning for long-horizon optimization).
- Replace single Kafka cluster with region-local clusters + cross-region replication for analytics.
- Custom location ingestion pipeline replacing Redis with a purpose-built geospatial data store.

## 11) Reliability and Fault Tolerance (3-5 min)

- **Order Service**: PostgreSQL with synchronous replication. Idempotent order creation (idempotency key from client). Outbox pattern for reliable event publishing to Kafka.
- **Payment**: Two-phase approach: authorize at order placement, capture at delivery confirmation. Refund on cancellation. Reconciliation job catches discrepancies.
- **Dispatch Service failure**: Orders queue in Kafka. When dispatch recovers, it processes the backlog. Drivers remain available and will be matched.
- **Tracking Service failure**: Customers fall back to polling the order status API. No data loss since driver locations continue flowing into Redis/Kafka.
- **Driver app connectivity**: Offline mode queues location updates locally; batch-sends when connectivity returns. Order acceptance requires connectivity.

## 12) Observability and Operations (2-4 min)

- **Order funnel metrics**: Orders placed -> confirmed -> picked up -> delivered. Track drop-off at each stage.
- **Dispatch metrics**: Time to assign driver, acceptance rate, reassignment rate.
- **ETA accuracy**: Compare predicted vs. actual delivery time. Track by city, time of day, restaurant.
- **Driver utilization**: % time active vs. idle. Orders per hour per driver.
- **System health**: API latency p50/p99, Kafka consumer lag, Redis memory usage, WebSocket connection count.
- **Alerting**: Page on order placement error rate > 0.1%, dispatch failure rate > 1%, ETA error > 15 minutes.

## 13) Security (1-3 min)

- **Payment security**: PCI-DSS compliance. Tokenized payment methods; no raw card data stored.
- **Location privacy**: Driver locations only visible to customers with active orders. Exact home addresses encrypted at rest.
- **Authentication**: OAuth2 with JWT tokens for all APIs. Separate auth for customer, driver, and merchant apps.
- **Fraud detection**: Detect and block fake orders, promo abuse, GPS spoofing by drivers.
- **Data encryption**: TLS for all communication; AES-256 for sensitive data at rest.

## 14) Team and Operational Considerations (1-2 min)

- **Marketplace team**: Owns dispatch, pricing, driver incentives. Focused on supply-demand balance.
- **Order team**: Owns order lifecycle, cart, checkout. Focused on conversion and reliability.
- **Restaurant platform team**: Owns merchant portal, menu management, restaurant onboarding.
- **Driver platform team**: Owns driver app, location services, earnings/payments.
- **Infrastructure team**: Owns Kafka, Redis, database operations, deployment pipelines.

## 15) Common Follow-up Questions

1. **How do you handle surge pricing?** -- Dynamic pricing based on supply (available drivers) vs. demand (order volume) per zone. Increase delivery fee and driver pay during high demand. Show customer the fee before order confirmation.
2. **How do you predict ETA?** -- ML model trained on historical delivery data. Features: distance, time of day, traffic, restaurant prep time history, weather. Separate models for prep time, pickup travel, and delivery travel.
3. **How do you handle refunds and disputes?** -- Automated refund for common issues (missing items, late delivery > 30 min). Escalation to support for complex disputes. Charge-back to restaurant or driver as appropriate.
4. **How do you ensure food quality?** -- Minimize time food waits at restaurant (dispatch driver close to food-ready time). Insulated bags required. Rating system for drivers and restaurants.
5. **How do you onboard restaurants?** -- Self-service portal for menu upload. Photo service for menu items. Integration with restaurant POS systems for order injection.

## 16) Closing Summary (30-60s)

"We designed a food delivery platform with four core subsystems: a search-powered restaurant discovery service, a strongly consistent order management system, a batch-based dispatch engine that optimally matches drivers to orders using geospatial proximity from Redis, and a WebSocket-based real-time tracking system that streams driver locations to customers. The architecture handles 10M orders/day with sub-second order placement, 40K driver location updates per second, and 500K concurrent tracking sessions, scaling horizontally by partitioning per city and region."

## Appendix A: Configuration / Tuning Knobs

| Knob | Default | Tuning Guidance |
|---|---|---|
| Dispatch batch window | 30 seconds | Shorter = lower latency, worse matching quality |
| Driver search radius | 5 km | Wider in suburbs, tighter in dense cities |
| Max orders per driver (batching) | 2 | Higher saves cost but increases delivery time |
| GPS update interval | 5 seconds | Lower = smoother tracking, higher battery/bandwidth |
| Driver acceptance timeout | 30 seconds | Shorter = faster dispatch, more rejections |
| ETA padding | +5 minutes | Under-promise, over-deliver; adjust by confidence |
| WebSocket ping interval | 30 seconds | Keep-alive frequency for connection health |

## Appendix B: Vocabulary Cheat Sheet

| Term | Definition |
|---|---|
| Dispatch | The process of assigning a delivery driver to an order |
| Bipartite Matching | Optimal assignment between two sets (drivers and orders) |
| Geofencing | Virtual geographic boundary for triggering location-based events |
| GEOSEARCH | Redis command for finding points within a geographic radius |
| Outbox Pattern | Reliable event publishing by writing events to a DB table before Kafka |
| Surge Pricing | Dynamic pricing that increases during high demand periods |
| Order Batching / Stacking | Assigning multiple orders to one driver for efficiency |
| ETA | Estimated Time of Arrival |

## Appendix C: References

- DoorDash Engineering Blog: "How DoorDash Optimizes Delivery" series
- Uber Engineering: "Uber's Real-Time Push Platform"
- Redis documentation: Geospatial commands
- Kuhn, H. "The Hungarian Method for the Assignment Problem" (1955)
- Li et al. "Large-Scale Order Dispatch in On-Demand Ride-Hailing Platforms" (KDD 2018)
