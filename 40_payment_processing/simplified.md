# Payment Processing System -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
We need to design a payment processing system like Stripe that handles 100M transactions per day ($5B daily volume) with exactly-once execution. The hard parts are preventing double-charges during network failures, maintaining a consistent financial ledger, and complying with PCI DSS while keeping authorization latency under 2 seconds.

## 2. Requirements
**Functional:** Authorize and capture payments. Refund (full or partial). Tokenize card data for reuse. Multi-currency support. Reliable webhook notifications to merchants. Settlement and reconciliation.

**Non-functional:** Exactly-once payment execution -- no double-charges, no lost payments. Strong consistency for payment state. 99.99% availability. Authorization under 2 seconds. PCI DSS Level 1 compliance. Complete audit trail via immutable ledger.

## 3. Core Concept: Idempotency Key + Double-Entry Ledger
Every payment API call requires a merchant-provided idempotency key. The system checks this key before processing: if the key was already used successfully, return the cached result without re-processing. This prevents double-charges from network retries. All financial movements are recorded in an append-only double-entry ledger where total debits always equal total credits. The ledger is the source of truth -- it is never modified, only appended to -- enabling complete auditability and three-level reconciliation (internal, acquirer, bank).

## 4. High-Level Architecture
```
Merchant --> Payment API (idempotency check) --> Payment Orchestrator (state machine)
                                                      |
                                          +-----------+-----------+
                                          |           |           |
                                    Fraud Service  Token Vault  Gateway Router
                                    (ML scoring)   (HSM/PCI)   (card networks)
                                                                    |
                                                             Visa / Mastercard
                                                                    |
Ledger Service (append-only, double-entry)  <----- Settlement Service
                                                         |
Webhook Service --> Merchant                      Payout Service --> Merchant Bank
```
- **Payment Orchestrator**: State machine managing the payment lifecycle with forward-only transitions.
- **Token Vault**: PCI-scoped service with HSM-backed encryption; only component that touches raw card data.
- **Gateway Router**: Selects optimal acquirer based on card type, currency, and success rates.
- **Ledger Service**: Append-only double-entry bookkeeping for all financial movements.

## 5. How a Payment Works
1. Merchant creates a Payment Intent with an idempotency key.
2. On confirmation, Fraud Service scores the transaction (under 100ms).
3. Token Vault retrieves encrypted card details for the payment method.
4. Gateway Router selects the best acquirer and sends the authorization to the card network.
5. Card network forwards to the issuing bank, which approves or declines.
6. Orchestrator atomically updates payment status, creates charge record, and inserts ledger entries (debit customer, credit merchant, credit platform fee) in one database transaction.
7. Webhook Service notifies the merchant of the result.

## 6. What Happens When Things Fail?
- **Network timeout to card network**: Do NOT retry the authorization (could cause a double-charge). Instead, send a status inquiry to check if the original was processed. Reconciliation job catches any remaining discrepancies.
- **Database failure mid-transaction**: The atomic transaction rolls back. Payment stays in PROCESSING. On recovery, the orchestrator detects the incomplete payment and resolves it via status inquiry.
- **Fraud service unavailable**: Fall back to simplified rule-based checks (amount limits, velocity). Never block all payments due to fraud service outage.

## 7. Scaling
- **10x**: Shard payment database by merchant_id, ledger by account_id. Route to acquirers with highest approval rates per card type. Reporting queries on read replicas.
- **100x**: Multi-region active-active with per-region payment processing. Dedicated financial database (like TigerBeetle) optimized for double-entry accounting. Pre-authorized payments for sub-100ms processing.

## 8. Key Trade-offs to Mention
| Decision | Trade-off |
|---|---|
| Client-provided idempotency key | Gives merchants control over dedup and retry behavior, but requires merchants to implement key generation correctly |
| Append-only ledger (PostgreSQL) | Complete audit trail and proven reliability, but no updates or deletes means corrections require additional compensating entries |
| Status inquiry on timeout (vs. retry) | Prevents double-charges, but adds latency on the failure path and requires card network support for inquiry API |

## 9. Closing (30s)
> We designed a payment system handling 100M transactions/day with exactly-once execution. Client-provided idempotency keys and a state-machine orchestrator prevent double-charges. Card data is tokenized in an HSM-backed vault for PCI compliance. All movements are recorded in an append-only double-entry ledger enabling three-level reconciliation. Settlement batches daily with T+1 payout. The system scales by sharding payments by merchant and the ledger by account.
