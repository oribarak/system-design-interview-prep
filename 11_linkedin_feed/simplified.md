# LinkedIn Feed -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
We need to design LinkedIn's professional feed -- a ranked content surface for 100M daily users. Unlike consumer social feeds, LinkedIn must balance engagement with professional quality, deliberately throttling viral spread of low-quality content. The unique challenge is second-degree content: showing users posts that their connections engaged with, even from people they do not directly follow.

## 2. Requirements
**Functional:** Show a ranked feed from connections, followed influencers, companies, and groups. Include second-degree content (posts your connections engaged with) annotated with social proof ("User A liked this"). Support reactions (6 types), comments, reposts. Follow hashtags.

**Non-functional:** p50 under 400ms, p99 under 1 second, 99.95% availability, 100M DAU at 12K peak QPS, professional relevance weighted heavily in ranking, eventual consistency (seconds of delay acceptable).

## 3. Core Concept: Second-Degree Content with Quality-Gated Viral Expansion
The key insight is second-degree content discovery. When your connection likes a post by someone you do not follow, that post becomes a candidate for your feed with social proof annotation. This dramatically expands the candidate pool beyond direct connections. However, LinkedIn deliberately limits viral spread through quality gates: second-degree content must pass a higher quality threshold than first-degree, and engagement velocity is dampened to prevent low-quality content from spreading. The ranking model explicitly weights a content quality score (professional value) alongside engagement predictions.

## 4. High-Level Architecture

```
[ Post Created ] --> [ Kafka ] --> [ Engagement Aggregator ]
                                          |
                                     (engagement events)
                                          v
                              [ Second-Degree Candidate Buffer (Redis) ]

Client --> [ Feed Mixer ] --> [ 1st-degree candidates ] + [ 2nd-degree candidates ]
                  |
                  v
           [ Ranking Service ] --> [ Social Proof Service ] --> Client
```

- **Feed Mixer**: Orchestrates candidate retrieval from both first-degree (direct connections) and second-degree (connections' engagements) sources.
- **Engagement Aggregator**: Consumes engagement events from Kafka, looks up the reactor's connections, and buffers second-degree candidates per user in Redis.
- **Ranking Service**: ML model that scores candidates with a quality-weighted formula, explicitly penalizing clickbait and engagement bait.
- **Social Proof Service**: Generates annotations like "User A and 3 others reacted to this."

## 5. How Second-Degree Content Reaches Your Feed
1. Your connection (User A) likes a post by User C (whom you do not follow).
2. The engagement event goes to Kafka. The Engagement Aggregator looks up User A's connections.
3. For each of User A's connections (including you), the post is added to a second-degree candidate buffer in Redis.
4. When you open your feed, the Feed Mixer pulls both first-degree candidates (posts from your connections) and second-degree candidates from the buffer.
5. The ranking model scores all candidates. Second-degree content must pass a higher quality threshold.
6. Approved second-degree posts are annotated with social proof ("User A liked this") and included in the feed.
7. If multiple connections engaged with the same post, social proof aggregates: "User A, User B, and 5 others."

## 6. What Happens When Things Fail?
- **Second-degree service goes down**: Serve first-degree content only. Feed quality drops (smaller candidate pool) but remains functional. This is the most common degradation tier.
- **Ranking service goes down**: Serve a chronological first-degree feed. No ranking, no second-degree content. Still usable.
- **Connection graph unavailable**: Use cached connection lists (stale but usable). Alert immediately as this affects all feed operations.

## 7. Scaling
- **10x (1B DAU)**: Engagement fan-out becomes expensive. Batch into larger micro-batches (5-minute windows). Pre-compute second-degree candidates offline for the most active users. Model distillation for faster inference.
- **100x**: Replace second-degree push with a pull-based approach (query connections' recent engagements at read time). Regional feed infrastructure. On-device ranking for final re-ranking.

## 8. Key Trade-offs to Mention

| Decision | Trade-off |
|---|---|
| Including second-degree content vs first-degree only | Second-degree expands content diversity and enables professional discovery, but increases candidate set (1,000 vs 300) and compute cost. Core to LinkedIn's value proposition. |
| Quality-weighted ranking vs pure engagement optimization | Weighting quality explicitly prevents clickbait from going viral but may reduce engagement metrics. LinkedIn accepts lower engagement for higher professional value. |
| Engagement velocity dampening for viral content | Throttling fast-spreading content maintains quality but limits legitimate viral reach. The dampening factor is tunable. |

## 9. Closing (30s)
> "LinkedIn's feed is a pull-based, quality-weighted ranking system. The key differentiator is second-degree content -- surfacing posts that your connections engaged with, annotated with social proof. The ranking model explicitly weights content quality alongside engagement predictions, deliberately throttling viral spread of low-quality content. The system degrades gracefully through multiple tiers: from fully ranked with second-degree content, down to chronological first-degree only."
