# Payment Processing System -- Cheat Sheet

## Key Numbers
- 100M transactions/day, ~3,500 peak QPS
- $5 billion daily volume, ~$1.8 trillion/year
- Authorization latency target: < 2 seconds
- Fraud scoring: < 100ms per transaction
- 4 ledger entries per transaction = 400M entries/day
- Settlement: T+1 to merchant bank accounts
- Availability target: 99.99%

## Core Components
- **Payment API + Idempotency Service**: REST API with Redis-cached idempotency keys (24h TTL)
- **Payment Orchestrator**: State machine managing payment lifecycle with distributed locking
- **Fraud Detection Service**: ML scoring in < 100ms; velocity checks, AVS, CVV
- **Token Vault + HSM**: PCI-scoped card tokenization with hardware key management
- **Gateway Router**: Routes to optimal acquirer by card type/region/approval rate
- **Ledger Service**: Append-only double-entry bookkeeping (debits = credits)
- **Settlement + Payout**: Daily batch capture and T+1 merchant bank transfers
- **Webhook Service**: Reliable delivery with exponential backoff retries

## Architecture in One Sentence
Merchant API requests flow through an idempotent state-machine orchestrator that coordinates ML fraud scoring, HSM-backed card token retrieval, and optimal acquirer routing for card network authorization, with all financial movements recorded in an append-only double-entry ledger and settled via daily batch payout.

## Top 3 Trade-offs
1. **Idempotency key (client-provided) vs. server-generated**: Client key gives merchants control over retry semantics
2. **State machine orchestrator vs. event choreography**: Orchestrator is clearer for financial flows but creates a coordination point
3. **Batch settlement vs. real-time**: Batch is industry standard and simpler; real-time would require fundamental card network changes

## Scaling Story
- 10x: Shard payments DB by merchant, ledger by account; optimize acquirer routing; read replicas for reporting
- 100x: Multi-region active-active, dedicated ledger DB (TigerBeetle), event-sourced payments, pre-authorized repeat payments

## Closing Statement
A payment processing system guaranteeing exactly-once execution through idempotency keys, distributed locking, and status inquiry fallback -- with PCI DSS Level 1 card tokenization, ML fraud detection in < 100ms, append-only double-entry ledger for full auditability, and three-level reconciliation (internal, acquirer, bank) -- handling 100M transactions/day at $5B daily volume.
