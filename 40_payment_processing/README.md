# System Design Interview: Payment Processing System (like Stripe) -- "Perfect Answer" Playbook

---

## 0) Opening: Frame the Problem (30-60s)

> "We need to design a payment processing system -- similar to Stripe -- that enables merchants to accept online payments from customers. The core challenges are ensuring exactly-once payment execution (no double-charges, no lost payments), maintaining strong consistency across distributed components, handling PCI DSS compliance for sensitive card data, and supporting multiple payment methods and currencies. I will cover the payment flow, idempotency guarantees, the ledger system, fraud detection, and settlement/reconciliation."

---

## 1) Clarifying Questions (2-5 min)

**Scope**
- Are we designing a payment gateway (like Stripe) that integrates with card networks, or an internal payment service for a single platform?
- Should we support multiple payment methods (cards, bank transfers, digital wallets) or just card payments?
- Do we need to support recurring payments (subscriptions)?

**Scale**
- How many transactions per day? (Assume 100 million transactions/day.)
- What is the average transaction value? (Assume $50.)
- What is the peak QPS? (Assume 3,000 transactions/sec at peak.)

**Policy / Constraints**
- PCI DSS compliance level? (Level 1 -- highest, for > 6M transactions/year.)
- What payment consistency guarantee? (Exactly-once semantics -- no double-charges.)
- Multi-currency? (Yes -- support 50+ currencies with real-time exchange rates.)
- Settlement timeline? (T+1 for card payments to merchant accounts.)

---

## 2) Back-of-Envelope Estimation (3-5 min)

| Metric | Value |
|---|---|
| Transactions per day | 100,000,000 |
| Transaction QPS (avg) | 100M / 86,400 = ~1,160 QPS |
| Transaction QPS (peak, 3x) | ~3,500 QPS |
| Average transaction value | $50 |
| Daily transaction volume | 100M x $50 = $5 billion/day |
| Yearly volume | ~$1.8 trillion/year |
| Payment record size | ~2 KB |
| Daily storage (transactions) | 100M x 2 KB = 200 GB/day |
| Yearly storage | ~73 TB/year |
| Ledger entries per transaction | 4 (debit/credit x 2 accounts) |
| Ledger entries per day | 400M |
| Fraud detection latency budget | < 100ms per transaction |

---

## 3) Requirements (3-5 min)

### Functional
- **Charge**: Authorize and capture a payment from a customer's payment method to a merchant's account.
- **Refund**: Return funds (full or partial) from merchant to customer.
- **Tokenization**: Securely store payment method details (card numbers) and return a reusable token.
- **Multi-currency**: Accept payments in 50+ currencies; settle in the merchant's preferred currency.
- **Webhooks**: Notify merchants of payment events (succeeded, failed, refunded) via reliable webhooks.
- **Reconciliation**: Provide detailed transaction reports and enable settlement to merchant bank accounts.

### Non-functional
- **Exactly-once**: No double-charges; no lost payments. Every payment request produces exactly one outcome.
- **Consistency**: Strong consistency for payment state. A payment is either fully processed or not at all.
- **Availability**: 99.99% for payment processing (4.3 minutes downtime/month max).
- **Latency**: Payment authorization < 2 seconds; webhook delivery < 30 seconds.
- **Security**: PCI DSS Level 1 compliance. Card data encrypted end-to-end. No plaintext PAN storage.
- **Auditability**: Complete audit trail for every transaction; immutable ledger.

---

## 4) API Design (2-4 min)

### Create Payment Intent
```
POST /api/v1/payment_intents
Headers: Authorization: Bearer sk_live_xxx (merchant secret key)
Body: {
  amount: 5000,           // in smallest currency unit (cents)
  currency: "usd",
  payment_method: "pm_card_visa_4242",
  customer: "cus_abc123",
  description: "Order #1234",
  idempotency_id: "order-1234-attempt-1",
  metadata: { order_id: "1234" }
}

Response: {
  id: "pi_xyz789",
  status: "requires_confirmation",
  amount: 5000,
  currency: "usd",
  client_token: "pi_xyz789_EXAMPLE"   // for client-side confirmation
}
```

### Confirm Payment
```
POST /api/v1/payment_intents/{id}/confirm
Body: { payment_method: "pm_card_visa_4242" }

Response: {
  id: "pi_xyz789",
  status: "succeeded",
  amount: 5000,
  charge: { id: "ch_abc", amount: 5000, status: "succeeded" }
}
```

### Refund
```
POST /api/v1/refunds
Body: {
  payment_intent: "pi_xyz789",
  amount: 2500,     // partial refund
  reason: "customer_request",
  idempotency_id: "refund-pi-xyz789-1"
}

Response: { id: "re_def456", status: "succeeded", amount: 2500 }
```

### Tokenize Payment Method
```
POST /api/v1/payment_methods
Body: {
  type: "card",
  card: { number: "4242424242424242", exp_month: 12, exp_year: 2027, cvc: "123" }
}

Response: { id: "pm_card_visa_4242", type: "card", card: { last4: "4242", brand: "visa", exp_month: 12, exp_year: 2027 } }
```

### List Transactions (for Reconciliation)
```
GET /api/v1/transactions?merchant_id={id}&start_date={date}&end_date={date}&status={status}

Response: { transactions: [{ id, type, amount, currency, status, created_at, settled_at }], has_more, next_page }
```

---

## 5) Data Model (3-5 min)

### Payment Intents Table
| Column | Type | Notes |
|---|---|---|
| payment_intent_id | STRING (PK) | Globally unique |
| idempotency_key | STRING (UNIQUE) | Per-merchant idempotency |
| merchant_id | STRING (FK) | |
| customer_id | STRING (FK) | |
| amount | BIGINT | In smallest currency unit (cents) |
| currency | STRING(3) | ISO 4217 |
| status | ENUM | CREATED, REQUIRES_CONFIRMATION, PROCESSING, SUCCEEDED, FAILED, CANCELLED |
| payment_method_id | STRING | Tokenized reference |
| description | STRING | |
| metadata | JSON | Merchant-provided key-value pairs |
| created_at | TIMESTAMP | |
| updated_at | TIMESTAMP | |

### Charges Table
| Column | Type | Notes |
|---|---|---|
| charge_id | STRING (PK) | |
| payment_intent_id | STRING (FK) | |
| amount | BIGINT | |
| currency | STRING(3) | |
| status | ENUM | PENDING, SUCCEEDED, FAILED |
| gateway_response | JSON | Raw response from card network |
| auth_code | STRING | Authorization code from issuer |
| decline_reason | STRING | If failed |
| created_at | TIMESTAMP | |

### Ledger Entries Table (Double-Entry)
| Column | Type | Notes |
|---|---|---|
| entry_id | STRING (PK) | |
| transaction_id | STRING | Groups related entries |
| account_id | STRING | Which account (merchant, platform, customer) |
| entry_type | ENUM | DEBIT, CREDIT |
| amount | BIGINT | Always positive |
| currency | STRING(3) | |
| balance_after | BIGINT | Running balance for reconciliation |
| created_at | TIMESTAMP | Immutable, append-only |

### Payment Methods Table (Tokenized)
| Column | Type | Notes |
|---|---|---|
| payment_method_id | STRING (PK) | Token |
| customer_id | STRING (FK) | |
| type | ENUM | CARD, BANK_ACCOUNT, WALLET |
| card_last4 | STRING | Last 4 digits only |
| card_brand | STRING | visa, mastercard, etc. |
| card_exp_month | INT | |
| card_exp_year | INT | |
| card_fingerprint | STRING | Hash for dedup across tokens |
| encrypted_pan | BYTES | AES-256 encrypted, stored in HSM-backed vault |
| created_at | TIMESTAMP | |

**Storage choices**: Payment intents and charges in sharded PostgreSQL (shard by merchant_id) with synchronous replication. Ledger in a separate append-only PostgreSQL database (write-optimized, no updates or deletes). Payment method vault in an isolated, PCI-scoped database with HSM-backed encryption. Idempotency keys in Redis with 24-hour TTL.

---

## 6) High-Level Architecture (5-8 min)

### Dataflow
1. **Merchant's backend** creates a Payment Intent via the API.
2. **Payment API** validates the request, checks the idempotency key, and creates the payment record.
3. On confirmation, the **Payment Orchestrator** coordinates the payment flow:
   a. **Fraud Detection Service** scores the transaction (< 100ms).
   b. **Token Vault** retrieves the encrypted card details for the payment method.
   c. **Payment Gateway Router** selects the optimal acquirer/processor and sends the authorization request to the **Card Network** (Visa, Mastercard).
4. The card network routes to the **Issuing Bank**, which approves or declines.
5. Authorization result flows back. The orchestrator updates the payment status and creates **Ledger Entries** (double-entry bookkeeping).
6. **Webhook Service** delivers the payment event to the merchant's endpoint.
7. End of day: **Settlement Service** batches authorized transactions, submits capture requests to acquirers, and records settlement in the ledger.
8. **Payout Service** transfers settled funds to merchant bank accounts (T+1).

### Components
- **Payment API**: RESTful API with idempotency support. Entry point for all payment operations.
- **Payment Orchestrator**: State machine managing the payment lifecycle.
- **Fraud Detection Service**: ML-based real-time fraud scoring.
- **Token Vault**: PCI-scoped service storing encrypted card data. HSM-backed key management.
- **Payment Gateway Router**: Routes to optimal acquirer based on card type, currency, success rate.
- **Card Network Interface**: Adapters for Visa, Mastercard, Amex, etc. (ISO 8583 protocol).
- **Ledger Service**: Append-only double-entry bookkeeping for all financial movements.
- **Webhook Service**: Reliable webhook delivery with retries and exponential backoff.
- **Settlement Service**: Batches and settles transactions with acquirers.
- **Payout Service**: Transfers settled funds to merchant bank accounts.
- **Reconciliation Service**: Compares internal ledger with acquirer/bank statements.
- **Reporting Service**: Generates transaction reports, dashboards, and financial statements.

> **One-sentence summary**: A payment processing system where merchant API requests flow through an idempotent orchestrator that coordinates fraud detection, tokenized card retrieval, and card network authorization, with all financial movements recorded in a double-entry ledger and settled to merchant accounts via batch payout.

---

## 7) Deep Dive #1: Exactly-Once Payment Execution (8-12 min)

The most critical requirement in payment processing is ensuring that every payment is executed exactly once. A double-charge is a serious financial and trust issue; a lost payment means lost revenue.

### Sources of Duplication
1. **Client retries**: Network timeout causes the merchant to retry the API call.
2. **Internal retries**: The orchestrator retries a failed step (e.g., network error to card network).
3. **At-least-once message delivery**: Kafka/queue delivers the same event twice.

### Idempotency Key Pattern
Every payment API call requires an `idempotency_key` (provided by the merchant).

**Flow**:
1. Payment API receives request with idempotency_key.
2. Check Redis: `SETNX idempotency:{merchant_id}:{key} -> {payment_intent_id, status}` with 24h TTL.
3. If the key already exists:
   - If the previous request succeeded, return the cached result (same response, no re-processing).
   - If the previous request is still processing, return 409 Conflict.
   - If the previous request failed, allow retry (update the cached entry).
4. If the key is new, proceed with payment processing.

### Payment State Machine
```
CREATED -> REQUIRES_CONFIRMATION -> PROCESSING -> SUCCEEDED
                                               -> FAILED
                                               -> CANCELLED
```

- Each state transition is persisted in the database within a transaction.
- The orchestrator only advances the state forward (no backward transitions except explicit cancellation).
- Each state transition is idempotent: re-applying the same transition is a no-op.

### Atomic Operations with the Ledger
When a payment succeeds:
1. In a single database transaction:
   - Update payment_intent status to SUCCEEDED.
   - Insert charge record.
   - Insert ledger entries (debit customer, credit merchant, credit platform fee).
2. If any step fails, the entire transaction rolls back. The payment remains in PROCESSING state and can be retried.

### Handling Network Failures with Card Networks
The most dangerous scenario: we sent an authorization request to the card network, but the response was lost (network timeout).

**Resolution**:
1. Mark the payment as `PROCESSING` before sending to the card network.
2. If the response is lost, do NOT retry the authorization (could cause a double-charge).
3. Instead, send a **status inquiry** to the card network to check if the original authorization was processed.
4. If the inquiry shows it was authorized, update our records accordingly.
5. If the inquiry shows it was not received, retry the authorization.
6. Implement a **reconciliation job** that periodically compares our PROCESSING payments with card network records to catch any discrepancies.

### Distributed Locking
- For each payment_intent_id, acquire a distributed lock (Redis SETNX with TTL) before processing.
- This prevents concurrent processing of the same payment by multiple workers.
- Lock TTL = 30 seconds (longer than the longest expected processing time).

---

## 8) Deep Dive #2: Double-Entry Ledger System (5-8 min)

A double-entry ledger is essential for financial integrity. Every monetary movement is recorded as a pair of entries that balance to zero.

### Double-Entry Principle
For every transaction, total debits = total credits. Example for a $50 payment with a 2.9% + $0.30 platform fee:

| Entry | Account | Type | Amount |
|---|---|---|---|
| 1 | Customer (cardholder) | DEBIT | $50.00 |
| 2 | Merchant | CREDIT | $48.25 |
| 3 | Platform (revenue) | CREDIT | $1.75 |

Verification: $50.00 debit = $48.25 + $1.75 credits. Balanced.

### Refund Example ($25 partial refund):

| Entry | Account | Type | Amount |
|---|---|---|---|
| 1 | Customer (cardholder) | CREDIT | $25.00 |
| 2 | Merchant | DEBIT | $24.13 |
| 3 | Platform (revenue) | DEBIT | $0.87 |

### Ledger Properties
- **Append-only**: Entries are never modified or deleted. Corrections are made by adding new entries.
- **Immutable**: Once written, an entry cannot be changed. This provides a complete audit trail.
- **Strongly consistent**: All entries for a transaction are written atomically.
- **Partitioned**: Ledger is partitioned by account_id for read performance (balance lookups).

### Balance Computation
- **Real-time balance**: Maintained as a running total in a separate `account_balances` table, updated atomically with each ledger entry.
- **Verification**: Periodically recompute balances from the ledger entries and compare with the `account_balances` table. Any discrepancy triggers an alert.

### Reconciliation
Three levels of reconciliation:
1. **Internal**: Verify all ledger entries balance (sum of debits = sum of credits per transaction).
2. **Acquirer reconciliation**: Compare our settlement records with acquirer's settlement files (received daily).
3. **Bank reconciliation**: Compare our payout records with actual bank statement credits.

Discrepancies are flagged for manual investigation with a tolerance window (timing differences are expected; amount differences are not).

---

## 9) Trade-offs and Alternatives (3-5 min)

| Decision | Chosen | Alternative | Rationale |
|---|---|---|---|
| Idempotency | Client-provided idempotency key + Redis cache | Server-generated + DB check | Client key gives merchant control; Redis is faster than DB lookup |
| Ledger | Append-only PostgreSQL | Immutable ledger DB (QLDB, TigerBeetle) | PostgreSQL is proven; QLDB adds cryptographic verification but is less flexible |
| Payment orchestration | State machine (orchestrator) | Choreography (event-driven saga) | Orchestrator provides clearer control flow and easier debugging for financial operations |
| Card data storage | Tokenized vault with HSM | Third-party tokenization (Stripe/Adyen) | In-house gives control; third-party reduces PCI scope |
| Settlement | Batch (end of day) | Real-time settlement | Batch is standard in card industry; real-time adds complexity |

---

## 10) Scaling: 10x and 100x (3-5 min)

### 10x (1B transactions/day, 35K peak QPS)
- **Database sharding**: Shard payment database by merchant_id. Ledger sharded by account_id.
- **Gateway routing optimization**: Route to acquirers with highest approval rates per card type/region.
- **Read replicas**: Reporting and reconciliation queries run on read replicas, not primary.
- **Horizontal API scaling**: Stateless API servers behind load balancers; add more instances.
- **Caching**: Cache merchant configurations, exchange rates, and routing rules in Redis.

### 100x (10B transactions/day)
- **Multi-region active-active**: Process payments in the region closest to the merchant. Ledger entries replicated asynchronously (eventual consistency for reporting, strong for individual transactions).
- **Dedicated ledger database**: TigerBeetle or a custom-built financial database optimized for double-entry accounting at extreme scale.
- **Event sourcing for payments**: Instead of mutable payment records, store an append-only event log per payment. Rebuild state by replaying events.
- **Pre-authorized payments**: For repeat customers, pre-authorize and store authorization codes, enabling sub-100ms payment processing.
- **Acquirer load balancing**: Spread transactions across multiple acquirers to avoid single-acquirer bottlenecks.

---

## 11) Reliability and Fault Tolerance (3-5 min)

- **Database failure**: Synchronous replication with automatic failover. RPO = 0 (no transaction loss). The ledger is the source of truth; it must never lose data.
- **Card network timeout**: Status inquiry pattern (described in Deep Dive #1). Never blindly retry authorizations.
- **Payment orchestrator crash**: The orchestrator is stateless; payment state is in the database. On restart, it resumes from the last persisted state. Distributed lock prevents concurrent processing.
- **Webhook delivery failure**: Retry with exponential backoff (1s, 2s, 4s, ... up to 24 hours). Store webhook events persistently. Merchants can query for missed events via API.
- **Fraud service failure**: If the fraud service is unavailable, use a simplified rule-based fallback (transaction amount limits, velocity checks). Never block all payments due to fraud service outage.
- **Reconciliation catches everything**: Even if a failure causes a subtle inconsistency, the daily reconciliation job detects it. This is the safety net.
- **Disaster recovery**: Full database backups every 6 hours with point-in-time recovery. Cross-region replication for the ledger.

---

## 12) Observability and Operations (2-4 min)

- **Payment success rate**: The most critical metric. Track overall and per-acquirer, per-card-type, per-region. Any drop triggers immediate investigation.
- **Authorization latency**: p50, p95, p99. Breakdown by acquirer and card network. Alert if p99 > 3s.
- **Decline rate**: Track reasons (insufficient funds, fraud, technical). High decline rate may indicate routing issues.
- **Fraud detection performance**: False positive rate (legitimate transactions flagged) and false negative rate (fraud that slipped through).
- **Ledger integrity**: Automated balance verification runs every hour. Any imbalance triggers a P0 alert.
- **Webhook delivery rate**: Percentage of webhooks delivered on first attempt. Track retry backlog.
- **Settlement reconciliation**: Daily discrepancy count and amount. Zero discrepancies is the target.
- **PCI compliance monitoring**: Continuous scanning for unauthorized access to cardholder data environment.

---

## 13) Security (1-3 min)

- **PCI DSS Level 1**: Cardholder data environment (CDE) is isolated. Only the Token Vault handles raw card data. All other services use tokens.
- **Encryption**: AES-256 for card data at rest, managed by HSM (Hardware Security Module). TLS 1.3 for all transit.
- **Tokenization**: Card numbers are replaced with non-reversible tokens immediately upon receipt. The mapping is stored only in the Token Vault.
- **Network segmentation**: CDE is in a separate VPC with strict firewall rules. No direct internet access.
- **Access control**: Least-privilege IAM. No engineer has access to production card data. All access logged and audited.
- **API security**: Merchant API keys (publishable + secret). Secret keys never exposed to client-side code. Webhook signatures (HMAC) verify authenticity.
- **3D Secure**: Supports 3DS2 for strong customer authentication (SCA) in European markets (PSD2 compliance).
- **Fraud prevention**: Real-time ML scoring, velocity checks, AVS (address verification), CVV verification.

---

## 14) Team and Operational Considerations (1-2 min)

- **Core payments team**: Payment API, orchestrator, state machine logic.
- **Risk/Fraud team**: ML fraud models, rule engine, manual review queue.
- **Ledger/Finance team**: Double-entry ledger, reconciliation, settlement, payouts.
- **Vault/Security team**: Token Vault, HSM management, PCI compliance, security audits.
- **Integrations team**: Card network adapters (Visa, MC), acquirer integrations, payment method support.
- **Deployment**: Zero-downtime deploys mandatory. Canary releases for payment logic changes. Feature flags for new payment methods. Extensive pre-production testing with card network sandboxes.

---

## 15) Common Follow-up Questions

**Q: How do you handle recurring payments (subscriptions)?**
A: Store the customer's tokenized payment method. A Subscription Service maintains billing schedules. At each billing cycle, it creates a Payment Intent using the stored token. Failed payments are retried with exponential backoff (day 1, day 3, day 7) before cancelling the subscription. Dunning emails notify the customer.

**Q: How do you handle disputes (chargebacks)?**
A: When a cardholder disputes a charge, the card network sends a chargeback notification. The system automatically debits the merchant's account for the disputed amount. The merchant can submit evidence via API. If the dispute is resolved in the merchant's favor, the debit is reversed. All dispute activity is recorded in the ledger.

**Q: How does multi-currency work?**
A: Accept payments in the customer's currency. Convert to the merchant's settlement currency using a locked exchange rate (at payment time, not settlement time). Store both the original and converted amounts. The spread between market rate and our rate is a revenue source.

**Q: What happens if the database fails mid-transaction?**
A: The database transaction (payment status update + ledger entries) is atomic. If the database fails mid-write, the transaction rolls back. The payment remains in PROCESSING state. On database recovery, the orchestrator detects the incomplete payment, performs a status inquiry with the card network, and completes or cancels it.

**Q: How do you test payment systems?**
A: (1) Card network sandboxes for integration testing. (2) Special test card numbers that trigger specific responses (approve, decline, timeout). (3) Chaos engineering to simulate acquirer failures. (4) Shadow mode: run new routing logic in parallel with production without affecting real transactions.

---

## 16) Closing Summary (30-60s)

> "We designed a payment processing system handling 100M transactions/day ($5B daily volume). The system guarantees exactly-once payment execution through client-provided idempotency keys, a state-machine orchestrator, and distributed locking. Card data is tokenized immediately and stored in an HSM-backed vault (PCI DSS Level 1). The authorization flow routes through optimal acquirers to card networks and issuing banks. All financial movements are recorded in an append-only double-entry ledger, enabling complete auditability and three-level reconciliation (internal, acquirer, bank). Settlement batches daily with T+1 payout to merchant accounts. The fraud detection service scores every transaction in < 100ms using ML models. The system scales by sharding the payment database by merchant and the ledger by account."

---

## Appendix A: Configuration / Tuning Knobs

| Knob | Default | Tune When |
|---|---|---|
| `idempotency_key_ttl` | 24 hours | Shorter if key collisions are a concern; longer for slow merchants |
| `payment_lock_ttl` | 30s | Longer if card network responses are slow |
| `fraud_score_threshold` | 0.75 | Lower blocks more fraud; higher reduces false positives |
| `webhook_retry_schedule` | 1s, 2s, 4s, ... 24h | Adjust backoff based on merchant endpoint reliability |
| `settlement_batch_time` | 23:00 UTC | Align with acquirer processing windows |
| `payout_schedule` | T+1 | T+0 for premium merchants; T+7 for high-risk |
| `exchange_rate_cache_ttl` | 5 min | Lower for volatile currencies |
| `3ds_threshold` | All EU transactions | Expand/reduce based on regulation |
| `max_retry_attempts` | 3 | For card network timeouts before marking as failed |

## Appendix B: Vocabulary Cheat Sheet

| Term | Definition |
|---|---|
| **Payment Intent** | An object representing the lifecycle of a payment from creation to completion |
| **Authorization** | Reserving funds on a cardholder's account without transferring money |
| **Capture** | Finalizing the transfer of authorized funds from cardholder to merchant |
| **Settlement** | The actual transfer of funds between banks, typically batched daily |
| **Acquirer** | The merchant's bank that processes card transactions on their behalf |
| **Issuer** | The cardholder's bank that issued the payment card |
| **PAN** | Primary Account Number; the full card number (must never be stored in plaintext) |
| **HSM** | Hardware Security Module; tamper-resistant device for cryptographic key management |
| **PCI DSS** | Payment Card Industry Data Security Standard; compliance framework for handling card data |
| **Double-Entry** | Accounting system where every transaction has equal debits and credits |
| **Idempotency Key** | Unique identifier ensuring a request is processed at most once |
| **Chargeback** | A dispute where the cardholder asks their bank to reverse a charge |
| **3D Secure** | Additional authentication step (e.g., OTP) for online card payments |
| **ISO 8583** | International standard for financial transaction messaging (card networks) |

## Appendix C: References

- Stripe's approach to idempotency and reliable systems
- TigerBeetle: Financial accounting database designed for mission-critical safety
- PCI DSS v4.0 Requirements and Security Assessment Procedures
- ISO 8583: Financial transaction card originated messages
- Designing Data-Intensive Applications (Martin Kleppmann) -- chapter on distributed transactions
- Square's double-entry bookkeeping system architecture
