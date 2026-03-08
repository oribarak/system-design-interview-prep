# Proximity Service (Nearby Friends) -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
We need a service that shows users which of their friends are nearby in real-time. With 10M concurrent users each updating location every 30 seconds, that is 333K location updates per second. The hard part is computing proximity efficiently -- naively checking each update against 50 online friends means 16.7M distance calculations per second -- and delivering notifications within 30 seconds via persistent connections.

## 2. Requirements
**Functional:** Users opt in to share location with friends. Display nearby friends within a configurable radius (default 5km). Push notifications when friends enter or leave proximity. Privacy controls for per-friend visibility.

**Non-functional:** Nearby friends list updates within 30 seconds of location change. Handle 333K location updates/sec and 16.7M proximity checks/sec. Support 10M concurrent users. Ephemeral location data (not permanently stored). Fail-closed privacy -- deny access when the privacy service is down.

## 3. Core Concept: Channel-Based Pub/Sub with Geohash Filtering
Instead of pull-based checking (read all friends' locations on every update), use push-based Pub/Sub. Each user has a channel. When User A goes online, their friends subscribe to A's channel. When A updates their location, it is published to A's channel and all subscribers receive it. Each subscriber computes distance locally. Geohash bucketing eliminates 90%+ of distance calculations: if a friend's geohash cell is more than 2 cells away, they cannot be within 5km, so skip the computation entirely.

## 4. High-Level Architecture
```
Mobile App <--> [WebSocket Gateway] (100 servers, 100K connections each)
                       |
                [Location Ingestion] --> Redis (in-memory, 320MB total)
                       |
                [Pub/Sub Servers] -- consistent hash ring by user_id
                       |
                [Friendship Service] -- cached friend lists
                       |
                [Privacy Service] -- enforces visibility rules
```
- **WebSocket Gateway**: 100 servers handling 10M persistent connections. Bidirectional: location updates in, proximity notifications out.
- **Pub/Sub Servers**: Consistent hash ring. Each user's channel lives on server H(user_id). Friends subscribe to the channel.
- **Location Cache**: Redis storing all 10M active positions (32 bytes each = 320MB total, fits in memory).
- **Geohash Filter**: Before computing Haversine distance, check if friend is in same or adjacent geohash cells. Eliminates 90%+ of computations.

## 5. How Proximity Detection Works
1. User A goes online; system subscribes to channels of A's 50 online friends and notifies those friends to subscribe to A's channel.
2. User A sends GPS location every 30 seconds via WebSocket.
3. Location is stored in Redis and published to A's Pub/Sub channel.
4. All subscribers (A's friends) receive the update.
5. Each subscriber checks geohash cells first: if A's cell is far away, skip distance computation.
6. For nearby cells, compute Haversine distance. If within the friend's configured radius, display or notify.
7. When A goes offline (no update for 5 minutes), unsubscribe from all channels.

## 6. What Happens When Things Fail?
- **WebSocket gateway failure**: Clients reconnect automatically with exponential backoff to another gateway. Connection registry (Redis) maps user to gateway.
- **Pub/Sub server failure**: Channel ownership moves to next node on the consistent hash ring. Subscribers reconnect. Brief gap in updates (seconds).
- **Privacy service down**: Default to not showing location (fail-closed). Privacy safety is preserved over availability.

## 7. Scaling
- **10x (100M concurrent)**: 1,000 WebSocket gateways geo-distributed. Scale Pub/Sub ring to hundreds of nodes. Pre-compute online friend lists. Geohash filtering becomes critical.
- **100x (1B concurrent)**: Regional deployment where proximity is computed only within the same region. Move distance computation to client devices entirely. Approximate proximity using geohash cell membership.

## 8. Key Trade-offs to Mention
| Decision | Trade-off |
|----------|-----------|
| Push-based Pub/Sub vs pull-based checking | Push eliminates redundant reads and distributes compute; more complex subscription management |
| Custom channel servers vs Redis Pub/Sub | Redis Pub/Sub does not shard well for 10M channels; custom consistent hash ring scales better |
| Geohash coarse filtering | Eliminates 90%+ of distance computations but introduces edge cases at cell boundaries |

## 9. Closing (30s)
> Push-based Pub/Sub where each user's location updates are published to a channel subscribed by their online friends. Geohash bucketing eliminates 90%+ of distance computations by skipping friends in distant cells. 100 WebSocket gateways maintain 10M persistent connections for sub-30-second notification delivery. All location data is ephemeral (5-minute TTL). Privacy is fail-closed -- if the privacy service is down, no locations are shared.
