# WhatsApp -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
We need to design a real-time messaging platform for 2 billion users supporting one-on-one and group chats with end-to-end encryption. The core challenge is guaranteeing zero message loss with exactly-once display, while maintaining 200M concurrent WebSocket connections and ensuring the server never has access to plaintext message content.

## 2. Requirements
**Functional:** One-on-one and group messaging (up to 1,024 members) with text and media. End-to-end encryption for all messages. Delivery status indicators (sent, delivered, read). Offline message queuing. Typing indicators and presence. Multi-device sync.

**Non-functional:** Message delivery in under 200ms for online users, zero message loss, message ordering within conversations, 99.99% availability, 200M concurrent connections, 3M peak QPS, server has zero knowledge of message content.

## 3. Core Concept: Store-and-Forward with End-to-End Encryption
The key insight is the delivery guarantee pipeline combined with end-to-end encryption. The server acts as a store-and-forward relay: it persists encrypted messages until the recipient acknowledges delivery, then deletes them. Zero message loss is achieved through client-generated idempotency keys, server-side persistence before ACK, and client retry on timeout. The Signal Protocol (Double Ratchet) provides forward secrecy -- each message uses a unique key, and compromising one key does not reveal past or future messages. The server only ever sees encrypted blobs.

## 4. High-Level Architecture

```
Sender --> [ Chat Server A ] --> [ Session Service (Redis) ] --> [ Chat Server B ] --> Recipient
                  |                                                     |
                  v                                                     v
           [ Cassandra ]                                    (if offline)
           (message store)                                  [ Offline Queue (Redis) ]
                                                                     |
                                                                     v
                                                            [ Push Notification (APNs/FCM) ]

[ Key Distribution Service ] -- public keys for E2E encryption
```

- **Chat Servers**: Stateful WebSocket servers, each handling ~50K concurrent connections. Route messages between users.
- **Session Service**: Redis mapping of user_id to their connected Chat Server, enabling message routing.
- **Offline Queue**: Redis sorted set storing encrypted messages for offline users. Flushed on reconnection.
- **Cassandra**: Persistent message store (encrypted blobs). Messages deleted after delivery confirmation.
- **Key Distribution Service**: Manages E2E encryption key exchange (public keys, pre-keys).

## 5. How a Message Is Sent and Delivered
1. Sender's device encrypts the message with the recipient's public key (Signal Protocol).
2. Encrypted message is sent via WebSocket to the sender's Chat Server with a client-generated msg_id.
3. Chat Server persists the encrypted blob to Cassandra and the offline queue, then ACKs to sender (one tick).
4. Chat Server looks up the recipient's connected Chat Server via Session Service.
5. If recipient is online: route to their Chat Server, deliver via WebSocket. Wait for device ACK. On ACK, send delivery receipt back to sender (two ticks).
6. If recipient is offline: push notification is sent via APNs/FCM. Message stays in offline queue.
7. When the offline user reconnects, all pending messages are flushed from the queue in order.

## 6. What Happens When Things Fail?
- **Chat Server crashes**: Client reconnects to another server. Session Service updates routing. Messages already persisted in Cassandra/offline queue are safe. In-flight messages not yet ACKed will be retried by the sender.
- **Sender does not receive ACK**: Client retries after 5 seconds. Server deduplicates by msg_id (idempotent). No duplicate delivery.
- **Recipient never comes online**: Messages are stored in the offline queue for up to 30 days, then permanently deleted.

## 7. Scaling
- **10x (2B concurrent connections)**: 40,000+ Chat Servers. Consistent hashing for user-to-server assignment. Overflow offline queue from Redis to Cassandra for users offline more than 1 hour. Shard group message delivery across multiple workers.
- **100x**: Replace WebSocket with a custom binary protocol for lower overhead. Purpose-built Chat Servers in Erlang/C++ for higher connection density (millions per server). Edge-deployed Chat Servers at PoPs for lower latency.

## 8. Key Trade-offs to Mention

| Decision | Trade-off |
|---|---|
| Store-and-forward (delete after delivery) vs store permanently | Store-and-forward preserves privacy and reduces storage costs, but makes multi-device sync harder (history lives on the device, not the server). |
| End-to-end encryption (Signal Protocol) | Server cannot read, scan, or search message content. Maximizes privacy but prevents server-side spam detection and compliance features. |
| At-least-once delivery + client-side dedup | Simpler than exactly-once at the transport layer. The rare duplicate is caught by the client using msg_id, giving exactly-once display. |

## 9. Closing (30s)
> "This WhatsApp design is a stateful WebSocket messaging system with end-to-end encryption. Chat Servers maintain persistent connections, and a Session Service routes messages between servers. Zero message loss is guaranteed through server-side persistence before ACK, client-side retry with idempotency keys, and an offline queue for disconnected users. The Signal Protocol provides forward secrecy, and the server only ever handles encrypted blobs. Messages are stored only until delivered, then deleted, respecting user privacy."
