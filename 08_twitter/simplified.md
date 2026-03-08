# Twitter -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
We need to design a Twitter-like platform where users post tweets, follow other users, and read a personalized home timeline. The core challenge is timeline delivery: when a user posts a tweet, it needs to appear in all their followers' timelines, but a celebrity with 50 million followers cannot trigger 50 million writes per tweet.

## 2. Requirements
**Functional:** Post tweets (280 chars + optional media). Follow/unfollow users. Read a home timeline (tweets from people you follow, reverse chronological). View a user's profile timeline. Like and retweet.

**Non-functional:** Home timeline loads in under 200ms (p99), 99.99% availability, support 300M DAU and 500M tweets/day, eventual consistency (timeline delivery within 5 seconds), tweets must never be lost.

## 3. Core Concept: Hybrid Fan-Out (Push for Regular Users, Pull for Celebrities)
The key insight is the hybrid fan-out strategy. Fan-out-on-write (push) pre-computes every user's timeline in a Redis cache -- when you tweet, your tweet ID is pushed into every follower's timeline. This makes reads instant (O(1)) but is disastrous for celebrities with millions of followers. Fan-out-on-read (pull) avoids the write explosion but makes reads expensive (merge N users' tweets). The hybrid combines both: regular users push, celebrities are pulled and merged at read time. This adds only 20-50ms to reads while eliminating the fan-out explosion.

## 4. High-Level Architecture

```
Client --> [ API Gateway ] --> [ Tweet Service ] --> [ Tweet DB ]
                                      |
                                      v
                                  [ Kafka ]
                                      |
                          +-----------+-----------+
                          v                       v
                  [ Fan-out Service ]     [ Search / Notifications ]
                          |
                          v
                  [ Redis (Timeline Cache) ] <-- [ Timeline Service ] <-- Client
```

- **Tweet Service**: Writes tweets to the database and publishes events to Kafka.
- **Fan-out Service**: Consumes tweet events, looks up followers, pushes tweet IDs into each follower's Redis timeline (skips celebrities).
- **Timeline Service**: On read, fetches pre-computed timeline from Redis, merges in celebrity tweets pulled on-demand, hydrates tweet data.
- **Redis Cluster**: Stores pre-computed timelines as sorted sets (tweet_id as score for time ordering).

## 5. How Posting and Reading a Tweet Works
1. User posts a tweet. Tweet Service writes it to the sharded database and publishes to Kafka.
2. Fan-out Service consumes the event, fetches the poster's follower list.
3. If the poster is a regular user (under 500K followers), it pushes the tweet_id into each follower's Redis sorted set.
4. If the poster is a celebrity, fan-out is skipped entirely.
5. When a user reads their timeline, Timeline Service reads their Redis sorted set (pre-computed feed).
6. It also checks which celebrities the user follows, pulls their recent tweets, and merges them by timestamp.
7. Tweet IDs are hydrated into full tweet objects and returned.

## 6. What Happens When Things Fail?
- **Redis timeline cache goes down**: Fall back to fan-out-on-read for all users. Timeline reads become slower (200-500ms instead of 50ms) but still work.
- **Fan-out Service crashes**: Kafka retains events. Consumers restart and resume from the last offset. Timelines catch up once the service recovers. No tweets are lost.
- **Tweet DB shard goes down**: Replica promotion handles failover. The Redis timeline cache absorbs reads during the transition.

## 7. Scaling
- **10x (3B DAU, 5B tweets/day)**: Shard Tweet DB to hundreds of MySQL instances (Vitess). Scale Redis to 600TB cluster. Run 1200 fan-out consumers. Add selective fan-out -- only push to users active in the last 7 days.
- **100x**: Regional deployment with geo-distributed data centers. Tiered timeline cache (hot users in Redis, warm in SSD-backed cache, cold rebuilt on demand). Edge timeline serving for popular feeds.

## 8. Key Trade-offs to Mention

| Decision | Trade-off |
|---|---|
| Fan-out-on-write vs fan-out-on-read | Push makes reads O(1) but writes scale with follower count. Pull makes writes O(1) but reads require merging N feeds. The hybrid gets the best of both. |
| Celebrity threshold (500K followers) | Below this, push is cheap. Above this, the fan-out cost is prohibitive. The threshold is tunable based on infrastructure capacity. |
| Chronological vs ranked timeline | Chronological is simpler and uses Snowflake ID ordering. Ranked (ML-based) improves engagement but adds a ranking service to the read path. Start chronological, add ranking later. |

## 9. Closing (30s)
> "This Twitter design uses a hybrid fan-out architecture. Regular users' tweets are pushed to followers' pre-computed timelines in Redis, giving O(1) reads. Celebrity tweets are pulled and merged at read time, avoiding the fan-out explosion. Snowflake IDs provide globally unique, time-ordered identifiers. The system is event-driven via Kafka, and everything is sharded -- MySQL by user_id, Redis by user_id -- for horizontal scale."
