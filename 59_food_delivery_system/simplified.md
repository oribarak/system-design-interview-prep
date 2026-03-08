# Food Delivery System (DoorDash) -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
We need a food delivery platform connecting customers, restaurants, and drivers. The system handles 10M daily orders, 200K concurrent drivers, and real-time order tracking. The hard part is the dispatch algorithm that optimally matches drivers to orders considering pickup ETA, delivery ETA, driver idle time, and food wait time -- all while providing real-time location tracking to 500K concurrent active orders.

## 2. Requirements
**Functional:** Restaurant discovery and search by location, cuisine, and rating. Order placement with cart management and payment. Automatic driver dispatch matching drivers to orders by proximity and ETA. Real-time driver location tracking for customers. Accurate ETA prediction at placement and during delivery.

**Non-functional:** Order placement under 1 second. Location update processing under 1 second. 99.99% availability for order placement. Handle 10x volume during promotions. Driver position visible within 5-10 seconds of actual position.

## 3. Core Concept: Batch Dispatch with Geospatial Matching
Instead of greedily assigning each order to the nearest driver, the system batches orders and drivers in 30-second windows and solves an assignment problem. For each (driver, order) pair, a score combines pickup ETA, delivery ETA, driver idle time, and order wait time. The Hungarian algorithm (or greedy approximation) finds the globally optimal assignment. This produces better overall results than greedy nearest-driver matching. Redis GEOSEARCH finds candidate drivers within a radius for each order.

## 4. High-Level Architecture
```
Customer App --> [API Gateway] --> [Order Service] -- PostgreSQL (ACID)
                                        |
Restaurant App <-- [Order Events via Kafka]
                                        |
Driver App --> [Location Service] -- Redis GEOADD (40K updates/sec)
                    |
             [Dispatch Service] -- batch matching every 30 seconds
                    |
             [Tracking Service] -- WebSocket to customer (500K connections)
                    |
             [ETA Service] -- ML model: prep time + travel time
```
- **Order Service**: Manages order lifecycle state machine with strong consistency in PostgreSQL.
- **Location Service**: Ingests 40K driver GPS updates/sec via Redis GEOADD. Answers proximity queries via GEOSEARCH.
- **Dispatch Service**: Batches orders every 30 seconds, scores driver-order pairs, solves assignment optimization.
- **Tracking Service**: Streams driver location to customers via WebSocket connections (500K concurrent).

## 5. How an Order Gets Delivered
1. Customer searches restaurants (Elasticsearch with geo-filtering), places order.
2. Order Service creates order record, triggers payment authorization.
3. Restaurant confirms and starts preparing. ETA Service predicts prep time.
4. When food is nearly ready, Dispatch Service finds nearby drivers via Redis GEOSEARCH (5km radius).
5. Dispatch batches orders in a 30-second window and solves optimal driver-order assignment.
6. Selected driver gets 30 seconds to accept. On rejection, order enters the next batch.
7. Driver picks up food; customer tracks driver location in real-time via WebSocket.

## 6. What Happens When Things Fail?
- **Dispatch Service failure**: Orders queue in Kafka. When dispatch recovers, it processes the backlog. Drivers remain available.
- **Tracking Service failure**: Customers fall back to polling the order status API. No data loss since driver locations continue flowing into Redis/Kafka.
- **No drivers available**: Expand search radius incrementally (5km to 8km to 12km). Queue and retry every 30 seconds. Notify customer of delay.

## 7. Scaling
- **10x**: Shard by city/region -- each city operates semi-independently. Scale Redis geospatial index per city. Dispatch Service runs per-city with no cross-city coordination.
- **100x**: Multi-region deployment with data sovereignty. ML-based dispatch using reinforcement learning for long-horizon optimization. Custom geospatial data store replacing Redis.

## 8. Key Trade-offs to Mention
| Decision | Trade-off |
|----------|-----------|
| Batch dispatch (30s windows) vs greedy nearest-driver | Batch produces globally better assignments; slight delay is acceptable |
| Redis geospatial vs PostGIS | Redis is faster for high-frequency point updates and radius queries at 40K writes/sec |
| WebSocket for tracking vs polling | WebSocket supports bidirectional real-time communication with lower overhead than polling |

## 9. Closing (30s)
> Four core subsystems: search-powered restaurant discovery, strongly consistent order management, batch-based dispatch that optimally matches drivers to orders using geospatial proximity from Redis, and WebSocket-based real-time tracking. The key insight is batch dispatch over greedy matching -- by grouping orders and drivers in 30-second windows and solving an assignment problem, we get globally better driver-order pairings. The system handles 10M orders/day and 40K driver location updates per second, scaling per-city.
