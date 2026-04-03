# Payment System (Stripe / Square / Adyen)

## 1. Problem Statement & Requirements

Design a payment processing platform that handles the full lifecycle of financial transactions — accepting payments from customers, routing to card networks, managing settlements, handling refunds, and preventing fraud — similar to Stripe, Square, Adyen, or PayPal.

### Functional Requirements
- **Charge a payment method** — credit/debit cards, bank transfers (ACH), digital wallets (Apple Pay, Google Pay)
- **Payment intents** — two-phase: authorize (hold funds) → capture (collect funds), or single-phase direct charge
- **Refunds** — full or partial refund of a completed payment
- **Payment methods** — store, retrieve, and delete customer payment methods (tokenized)
- **Idempotency** — every API call accepts an idempotency key; retries are safe and produce identical results
- **Webhooks** — notify merchants of async events (payment succeeded, refund completed, dispute opened)
- **Multi-currency** — accept payments in 135+ currencies; settle in merchant's preferred currency
- **Subscriptions / Recurring billing** — charge customers on a schedule (monthly, annual)
- **Dispute / Chargeback handling** — manage the lifecycle of customer-initiated disputes
- **Payout / Settlement** — aggregate funds and transfer to merchant bank accounts on a schedule
- **Ledger** — double-entry bookkeeping for every money movement; audit trail

### Non-Functional Requirements
- **Correctness above all** — money must never be created, lost, or double-counted
- **Exactly-once processing** — no duplicate charges, no lost refunds
- **High availability** — 99.999% for payment acceptance (< 5 min downtime/year)
- **Low latency** — payment authorization in < 2 seconds end-to-end
- **PCI DSS compliance** — Level 1; card data encrypted at rest and in transit; tokenization
- **Strong consistency** — payment state transitions must be ACID
- **Auditability** — every state change logged immutably for regulatory compliance
- **Scalability** — handle Black Friday traffic (10x normal peak)

### Out of Scope
- Detailed card network protocol internals (ISO 8583 message format)
- KYC/AML onboarding flow for merchants
- Tax calculation engine
- Full invoicing system

---

## 2. Scale Estimations

### Traffic

| Metric | Value |
|--------|-------|
| Payments processed / day | 50M |
| Payments / second (avg) | ~580 TPS |
| Payments / second (peak — Black Friday) | ~6,000 TPS |
| Refunds / day | 2.5M (5% refund rate) |
| Webhook deliveries / day | 200M (avg 4 events per payment lifecycle) |
| API calls / second (all endpoints) | ~10K RPS |
| Active merchants | 5M |
| Stored payment methods (tokens) | 2B |

### Storage

| Metric | Value |
|--------|-------|
| Payment record size | ~2 KB (metadata, status, amounts, timestamps) |
| Payments per year | 50M × 365 = ~18.25B |
| Payment storage (1 year) | 18.25B × 2 KB = **~36.5 TB** |
| Ledger entries (2 per payment + refunds + fees) | ~50B/year |
| Ledger storage (1 year) | 50B × 500 bytes = **~25 TB** |
| Event log (webhooks, audits) | **~20 TB/year** |
| Total storage (1 year) | **~80 TB** |
| Retention | **7+ years** (regulatory) |

### Bandwidth

| Metric | Value |
|--------|-------|
| API ingress | 10K RPS × 2 KB = ~20 MB/s |
| Webhook egress | 200M/day × 1 KB = ~2.3 MB/s avg, 50 MB/s peak |
| Card network traffic | ~580 TPS × 1 KB = ~0.6 MB/s |

### Hardware Estimate

| Component | Spec |
|-----------|------|
| API servers | 20-50 (stateless, auto-scaling) |
| Payment processing workers | 30-100 (orchestrate payment flow) |
| Database (primary) | 10-20 shards (PostgreSQL, SSD-backed) |
| Ledger database | 5-10 dedicated nodes (append-only, SSD) |
| Card network gateway | 10-20 (connection pooling to Visa/MC) |
| Webhook delivery fleet | 20-50 (async, retry-capable) |
| Redis (idempotency + rate limiting) | 3-6 nodes |

---

## 3. High-Level Architecture

```
+----------+       +-----------+       +--------------+       +-----------+
| Merchant | ----> | API Layer | ----> | Payment      | ----> | Card      |
| (client) |       | (gateway) |       | Orchestrator |       | Networks  |
+----------+       +-----------+       +--------------+       | (Visa/MC) |
                                             |                +-----------+
                                             v
                                       +----------+
                                       |  Ledger  |
                                       +----------+

Detailed:

Merchant App / Checkout
        |
        v
+--------------------+
|    API Gateway       |
| • TLS termination    |
| • Rate limiting      |
| • Auth (API keys)    |
| • Idempotency check  |
+--------+-----------+
         |
+--------v-----------+
| Payment Service     |
| (Orchestrator)      |
|                     |
| • Create intent     |
| • Validate inputs   |
| • Route to PSP      |
| • State machine     |
+---+-----+-----+---+
    |     |     |
    v     v     v
+-----+ +---+ +--------+
|Token| |Risk| |Ledger  |
|Vault| |Eng | |Service |
+-----+ +---+ +--------+
    |     |
    v     v
+--------------------+
| Payment Gateway     |
| (Network Routing)   |
|                     |
| • Connection pool   |
| • Protocol adapter  |
| • Retry / failover  |
+--------+-----------+
         |
    +----+----+----+
    v         v    v
+------+  +------+  +------+
| Visa |  |Master|  | ACH  |
|      |  | card |  |      |
+------+  +------+  +------+
```

### Component Breakdown

```
+======================================================================+
|                        CONTROL PLANE                                  |
|                                                                       |
|  +-------------------+  +-------------------+  +------------------+  |
|  | Merchant Config    |  | Routing Engine     |  | Webhook Service  |  |
|  | (API keys, payout  |  | (smart routing to  |  | (reliable async  |  |
|  |  schedule, fees,   |  |  best-performing   |  |  delivery with   |  |
|  |  currency prefs)   |  |  network/acquirer) |  |  retry + DLQ)    |  |
|  +-------------------+  +-------------------+  +------------------+  |
|                                                                       |
|  +-------------------+  +-------------------+  +------------------+  |
|  | Subscription Mgr   |  | Dispute Manager    |  | Settlement /     |  |
|  | (recurring billing, |  | (chargeback flow,  |  | Payout Engine    |  |
|  |  dunning, proration)|  |  evidence upload,  |  | (batch, daily)   |  |
|  +-------------------+  |  representment)    |  +------------------+  |
|                          +-------------------+                        |
+======================================================================+

+======================================================================+
|                         DATA PLANE                                    |
|                                                                       |
|  +----------------------------------------------------------------+  |
|  |                    Payment Service (Core)                       |  |
|  |                                                                 |  |
|  |  +------------------+  +------------------+                    |  |
|  |  | Payment State     |  | Idempotency       |                    |  |
|  |  | Machine            |  | Service            |                    |  |
|  |  | (intent→auth→     |  | (Redis-backed,     |                    |  |
|  |  |  capture→settle)  |  |  48h TTL)          |                    |  |
|  |  +------------------+  +------------------+                    |  |
|  +----------------------------------------------------------------+  |
|                                                                       |
|  +----------------------------------------------------------------+  |
|  |                    Token Vault (PCI Scope)                      |  |
|  |                                                                 |  |
|  |  Card number → Token mapping (AES-256-GCM encrypted)           |  |
|  |  HSM-backed key management                                     |  |
|  |  Isolated network segment (PCI CDE)                             |  |
|  +----------------------------------------------------------------+  |
|                                                                       |
|  +----------------------------------------------------------------+  |
|  |                    Risk / Fraud Engine                           |  |
|  |                                                                 |  |
|  |  • ML model scoring (real-time, < 50ms)                        |  |
|  |  • Rules engine (velocity, geolocation, device fingerprint)    |  |
|  |  • 3D Secure challenge trigger                                  |  |
|  +----------------------------------------------------------------+  |
|                                                                       |
|  +----------------------------------------------------------------+  |
|  |                    Ledger (Double-Entry)                         |  |
|  |                                                                 |  |
|  |  Every money movement = debit one account + credit another     |  |
|  |  Append-only, immutable log                                     |  |
|  |  Sum of all debits = Sum of all credits (always balanced)      |  |
|  +----------------------------------------------------------------+  |
+======================================================================+
```

---

## 4. API Design

### Create Payment Intent (Authorize)

```
POST /v1/payment_intents
Idempotency-Key: pi_req_abc123
Authorization: Bearer sk_live_...

Request:
{
  "amount": 2999,                        // cents (avoid floating point!)
  "currency": "usd",
  "payment_method": "pm_card_visa_4242",
  "capture_method": "manual",            // "automatic" for single-phase
  "description": "Order #1234",
  "metadata": {"order_id": "ord-1234"},
  "customer": "cus_abc123",
  "receipt_email": "alice@example.com"
}

Response (200 OK):
{
  "id": "pi_3MtwBwLkdIwHu7ix",
  "object": "payment_intent",
  "amount": 2999,
  "currency": "usd",
  "status": "requires_confirmation",
  "payment_method": "pm_card_visa_4242",
  "client_secret": "pi_3MtwBw_secret_YkS2...", // for frontend confirmation
  "created": 1712150400
}
```

### Confirm Payment (Execute Authorization)

```
POST /v1/payment_intents/{id}/confirm
Idempotency-Key: pi_confirm_abc123

Response (200 OK):
{
  "id": "pi_3MtwBwLkdIwHu7ix",
  "status": "requires_capture",     // if manual capture
  // OR
  "status": "succeeded",            // if automatic capture
  "charges": {
    "data": [{
      "id": "ch_abc123",
      "amount": 2999,
      "currency": "usd",
      "status": "succeeded",
      "authorization_code": "A12345",
      "network_transaction_id": "NT98765",
      "risk_score": 12,
      "outcome": {
        "network_status": "approved_by_network",
        "risk_level": "normal"
      }
    }]
  }
}
```

### Capture Payment

```
POST /v1/payment_intents/{id}/capture
Idempotency-Key: pi_capture_abc123

Request:
{
  "amount_to_capture": 2999           // can be less than authorized (partial capture)
}

Response (200 OK):
{
  "id": "pi_3MtwBwLkdIwHu7ix",
  "status": "succeeded",
  "amount_captured": 2999
}
```

### Create Refund

```
POST /v1/refunds
Idempotency-Key: ref_req_xyz789

Request:
{
  "payment_intent": "pi_3MtwBwLkdIwHu7ix",
  "amount": 1500,                     // partial refund (in cents)
  "reason": "customer_request"        // customer_request | duplicate | fraudulent
}

Response (200 OK):
{
  "id": "re_abc123",
  "amount": 1500,
  "currency": "usd",
  "status": "pending",               // pending → succeeded (async from network)
  "payment_intent": "pi_3MtwBwLkdIwHu7ix"
}
```

### Webhook Event

```
POST https://merchant.example.com/webhooks/stripe
Stripe-Signature: t=1712150400,v1=sha256_hmac_signature

{
  "id": "evt_abc123",
  "type": "payment_intent.succeeded",
  "data": {
    "object": {
      "id": "pi_3MtwBwLkdIwHu7ix",
      "amount": 2999,
      "status": "succeeded"
    }
  },
  "created": 1712150400
}

Merchant must respond 2xx within 10 seconds.
No 2xx → retry with exponential backoff (up to 3 days, ~20 attempts).
```

---

## 5. Data Model

### Payment Intent

```
+--------------------------------------------------------------+
|                    payment_intents                             |
+-----------------------+-----------+--------------------------+
| Field                 | Type      | Notes                    |
+-----------------------+-----------+--------------------------+
| payment_intent_id     | UUID      | PRIMARY KEY              |
| merchant_id           | UUID      | FK to merchants          |
| customer_id           | UUID      | FK to customers          |
| amount                | BIGINT    | Cents (never float!)     |
| currency              | CHAR(3)   | ISO 4217 (usd, eur)     |
| status                | ENUM      | See state machine below  |
| payment_method_id     | UUID      | FK (tokenized)           |
| capture_method        | ENUM      | automatic / manual       |
| amount_captured       | BIGINT    | May be < amount          |
| amount_refunded       | BIGINT    | Running refund total     |
| description           | TEXT      |                          |
| metadata              | JSONB     | Merchant-defined KV      |
| idempotency_key       | VARCHAR   | UNIQUE per merchant      |
| network_txn_id        | VARCHAR   | From card network        |
| authorization_code    | VARCHAR   | From issuer              |
| risk_score            | SMALLINT  | 0-100 from fraud engine  |
| failure_code          | VARCHAR   | "card_declined", etc.    |
| failure_message       | TEXT      |                          |
| client_secret         | VARCHAR   | For frontend SDK         |
| created_at            | TIMESTAMP |                          |
| updated_at            | TIMESTAMP |                          |
| captured_at           | TIMESTAMP |                          |
| cancelled_at          | TIMESTAMP |                          |
+-----------------------+-----------+--------------------------+

Index: (merchant_id, created_at DESC) — merchant dashboard
Index: (idempotency_key, merchant_id) — UNIQUE dedup
Index: (status, updated_at) — for settlement batch queries
```

### Ledger Entry (Double-Entry Bookkeeping)

```
+--------------------------------------------------------------+
|                    ledger_entries                              |
+-----------------------+-----------+--------------------------+
| Field                 | Type      | Notes                    |
+-----------------------+-----------+--------------------------+
| entry_id              | UUID      | PRIMARY KEY              |
| transaction_id        | UUID      | Groups debit+credit pair |
| account_id            | UUID      | FK to ledger accounts    |
| entry_type            | ENUM      | DEBIT / CREDIT           |
| amount                | BIGINT    | Always positive (cents)  |
| currency              | CHAR(3)   |                          |
| payment_intent_id     | UUID      | Reference to payment     |
| event_type            | VARCHAR   | "charge", "refund", etc. |
| created_at            | TIMESTAMP | Immutable                |
+-----------------------+-----------+--------------------------+

Constraint: entries are APPEND-ONLY. No updates, no deletes.
            Corrections are done by adding reversing entries.

Accounts:
  merchant:acme:usd          — merchant's balance
  platform:fees:usd          — platform fee revenue
  network:visa:usd           — liability to Visa
  customer:alice:usd         — customer's refund balance
  reserve:acme:usd           — held funds (fraud reserve)

Example — $29.99 payment with 2.9% + $0.30 fee:
  Transaction T1 (charge):
    DEBIT  network:visa:usd         $29.99  (we owe Visa for settlement)
    CREDIT merchant:acme:usd        $28.82  (merchant receives)
    CREDIT platform:fees:usd        $1.17   (our fee: 2.9% + $0.30)

  Invariant: total debits = total credits = $29.99 ✓
```

### Payment State Machine

```
+------------------------------------------------------------------+
|                Payment Intent Lifecycle                            |
|                                                                   |
|  +---------------------+                                          |
|  | requires_payment_    |  (created, no payment method yet)       |
|  | method               |                                          |
|  +----------+----------+                                          |
|             |                                                      |
|             | attach payment method                                |
|             v                                                      |
|  +----------+----------+                                          |
|  | requires_             |                                          |
|  | confirmation          |  (ready to submit to network)           |
|  +----------+----------+                                          |
|             |                                                      |
|        +----+----+                                                 |
|        |         |                                                 |
|        v         v                                                 |
|  +-----+--+ +---+----------+                                     |
|  |requires | |requires_     |                                     |
|  |_action  | |capture       |  (auth succeeded, capture pending)  |
|  |(3DS)    | |(manual mode) |                                     |
|  +----+---+ +---+-----+---+                                     |
|       |         |      |                                          |
|       v         v      v                                          |
|  +----+---+ +--+---+ ++----------+                               |
|  |succeeded| |cancel| |  expired  |  (auth expires: 7 days card, |
|  |         | |led   | |           |   29 days for some networks)  |
|  +----+---+ +------+ +-----------+                               |
|       |                                                           |
|       v (refund initiated)                                        |
|  +----+---+                                                       |
|  |partially|                                                      |
|  |_refunded|                                                      |
|  +----+---+                                                       |
|       |                                                           |
|       v (full amount refunded)                                    |
|  +----+---+                                                       |
|  |refunded |                                                      |
|  +---------+                                                      |
+------------------------------------------------------------------+
```

---

## 6. Core Design Decisions

### Decision 1: How to Guarantee Exactly-Once Payment Processing

This is the single most important design problem. Charging a customer twice is catastrophic.

```
+------------------------------------------------------------------+
|  The Problem:                                                     |
|                                                                   |
|  Merchant → "charge $29.99" → our API → card network → issuer   |
|                                                                   |
|  Failures can happen at EVERY hop:                                |
|  1. Merchant's request times out → retries → duplicate?          |
|  2. Our API crashes after sending to network → retry → double?   |
|  3. Network returns ambiguous response (timeout) → ???           |
|  4. Our API crashes after network approves, before responding    |
|     → Merchant retries → we see new request → double charge?     |
+------------------------------------------------------------------+
```

#### Solution: Multi-Layer Idempotency

```
+------------------------------------------------------------------+
|  Layer 1: Client-Side Idempotency Key                            |
|                                                                   |
|  Every API request includes: Idempotency-Key: "order-1234-charge"|
|                                                                   |
|  Processing:                                                      |
|  1. SETNX in Redis: idempotency:{merchant}:{key} → "processing" |
|     TTL = 48 hours                                                |
|  2. If key already exists:                                        |
|     → Check status: "processing" → return 409 (in progress)     |
|     → Check status: "completed" → return stored result (200)     |
|  3. Process payment                                               |
|  4. Store result: idempotency:{merchant}:{key} → {result JSON}  |
|  5. Return result to client                                       |
|                                                                   |
|  Merchant retries same key → gets identical response              |
|  No duplicate charge possible                                     |
+------------------------------------------------------------------+

+------------------------------------------------------------------+
|  Layer 2: Internal Request ID for Network Calls                  |
|                                                                   |
|  Our system assigns unique request_id for each network call       |
|                                                                   |
|  If we crash after sending to network:                            |
|  1. On restart, find "in-flight" payments (status=PROCESSING)    |
|  2. Query network: "what happened to request_id=REQ-123?"        |
|  3. Network returns: "approved" or "declined" or "not found"     |
|  4. Update our state accordingly                                  |
|                                                                   |
|  This is called "recovery/reconciliation"                        |
+------------------------------------------------------------------+

+------------------------------------------------------------------+
|  Layer 3: Database Transaction for State Changes                 |
|                                                                   |
|  BEGIN TRANSACTION;                                               |
|    UPDATE payment_intents SET status='succeeded' WHERE id=...;   |
|    INSERT INTO ledger_entries (debit) VALUES (...);              |
|    INSERT INTO ledger_entries (credit) VALUES (...);             |
|    INSERT INTO events (payment_intent.succeeded) VALUES (...);   |
|  COMMIT;                                                          |
|                                                                   |
|  All state changes are atomic:                                    |
|  Payment status + ledger + event log committed together           |
|  If any fails, all roll back → consistent state guaranteed       |
+------------------------------------------------------------------+
```

### Decision 2: The Double-Entry Ledger

```
+------------------------------------------------------------------+
|  Why Double-Entry Bookkeeping Is Non-Negotiable                  |
|                                                                   |
|  Single-entry (naive):                                            |
|    "Merchant ACME received $29.99"                               |
|    → Where did the money come from? Where did the fee go?        |
|    → How do you verify nothing is missing?                       |
|    → How do you reconcile with bank statements?                  |
|                                                                   |
|  Double-entry:                                                    |
|    Every transaction has EQUAL debits and credits:                |
|                                                                   |
|    Payment of $29.99 (fee: $1.17):                               |
|    +------------------+--------+---------+                        |
|    | Account          | Debit  | Credit  |                        |
|    +------------------+--------+---------+                        |
|    | network:visa     | $29.99 |         | (funds inbound)       |
|    | merchant:acme    |        | $28.82  | (merchant owes)       |
|    | platform:fees    |        |  $1.17  | (our revenue)         |
|    +------------------+--------+---------+                        |
|    | TOTAL            | $29.99 | $29.99  | ✓ BALANCED            |
|    +------------------+--------+---------+                        |
|                                                                   |
|    Refund of $15.00:                                              |
|    +------------------+--------+---------+                        |
|    | Account          | Debit  | Credit  |                        |
|    +------------------+--------+---------+                        |
|    | merchant:acme    | $14.42 |         | (deduct from merchant)|
|    | platform:fees    |  $0.58 |         | (refund our fee too)  |
|    | network:visa     |        | $15.00  | (send back to Visa)   |
|    +------------------+--------+---------+                        |
|    | TOTAL            | $15.00 | $15.00  | ✓ BALANCED            |
|    +------------------+--------+---------+                        |
|                                                                   |
|  Invariant: SELECT SUM(debit) - SUM(credit) FROM ledger = 0     |
|  If ever non-zero → something is broken → ALERT IMMEDIATELY     |
|                                                                   |
|  This invariant is checked continuously (every 5 min).           |
|  It catches bugs, race conditions, and data corruption.          |
+------------------------------------------------------------------+
```

### Decision 3: Smart Routing to Maximize Authorization Rate

```
+------------------------------------------------------------------+
|  Payment Routing Engine                                           |
|                                                                   |
|  Problem: Not all card networks / acquirers are equal.            |
|  Visa via Acquirer A: 95% auth rate, $0.10 per txn               |
|  Visa via Acquirer B: 92% auth rate, $0.08 per txn               |
|  → Route to A for higher success, B for lower cost?              |
|                                                                   |
|  Routing decisions based on:                                      |
|  +---------------------------------------------------+          |
|  | Signal               | How It Affects Routing      |          |
|  +---------------------------------------------------+          |
|  | Card brand (Visa/MC) | Which networks available    |          |
|  | Card country          | Local vs cross-border fees  |          |
|  | Card type (credit/    | Different interchange rates |          |
|  |   debit/prepaid)      |                             |          |
|  | Transaction amount    | Some acquirers cap amounts  |          |
|  | Merchant category     | Risk profile per MCC        |          |
|  | Historical auth rate  | Prefer higher-performing    |          |
|  |   per acquirer        |   acquirer for this BIN     |          |
|  | Acquirer health       | Circuit breaker if failing  |          |
|  | Currency              | Local processing preferred  |          |
|  +---------------------------------------------------+          |
|                                                                   |
|  Cascade / Failover:                                              |
|  1. Try primary acquirer (best expected auth rate)               |
|  2. If soft decline (insufficient funds, do-not-honor):          |
|     → Don't retry (issuer decision is final)                    |
|  3. If network error / timeout:                                   |
|     → Retry with secondary acquirer (different network path)    |
|  4. If hard decline (stolen card, fraud):                        |
|     → Don't retry (flag for review)                              |
|                                                                   |
|  This "smart routing" can improve auth rates by 2-5%             |
|  At $10B annual volume, 2% = $200M more revenue for merchants    |
+------------------------------------------------------------------+
```

---

## 7. Detailed Flow Diagrams

### Payment Authorization Flow

```
Merchant        API Gateway      Payment Svc      Fraud Engine    Token Vault    Card Network
   |                |                |                |               |               |
   |-- POST ------->|                |                |               |               |
   |   /payment_    |                |                |               |               |
   |   intents/     |                |                |               |               |
   |   {id}/confirm |                |                |               |               |
   |   Idempotency- |                |                |               |               |
   |   Key: abc123  |                |                |               |               |
   |                |                |                |               |               |
   |                |-- check ------>|                |               |               |
   |                |  idempotency   |                |               |               |
   |                |  (Redis SETNX) |                |               |               |
   |                |                |                |               |               |
   |                |                |-- risk check ->|               |               |
   |                |                |   (amount, IP, |               |               |
   |                |                |    device, geo)|               |               |
   |                |                |                |               |               |
   |                |                |<-- score: 12 --|               |               |
   |                |                |   (low risk,   |               |               |
   |                |                |    proceed)    |               |               |
   |                |                |                |               |               |
   |                |                |-- detokenize --|-------------->|               |
   |                |                |   pm_card_4242 |               |               |
   |                |                |                |               |               |
   |                |                |<-- card: 4242..|--------------|               |
   |                |                |   (decrypted   |               |               |
   |                |                |    in secure    |               |               |
   |                |                |    enclave)     |               |               |
   |                |                |                |               |               |
   |                |                |-- authorize ---|---------------|-------------->|
   |                |                |   $29.99 USD   |               |   (ISO 8583  |
   |                |                |   card: 4242...|               |    message)   |
   |                |                |                |               |               |
   |                |                |<-- approved ---|---------------|---------------|
   |                |                |   auth_code:   |               |               |
   |                |                |   A12345       |               |               |
   |                |                |                |               |               |
   |                |                |-- BEGIN TXN:   |               |               |
   |                |                |   UPDATE payment status=authorized             |
   |                |                |   INSERT ledger entries (debit+credit)         |
   |                |                |   INSERT event (payment.authorized)            |
   |                |                |   COMMIT                                       |
   |                |                |                |               |               |
   |                |                |-- store result |               |               |
   |                |                |   in idempotency cache                         |
   |                |                |                |               |               |
   |<-- 200 OK -----|<---------------|                |               |               |
   |   status:      |                |                |               |               |
   |   authorized   |                |                |               |               |
   |                |                |                |               |               |
   |                |  async: emit webhook event                      |               |
   |                |  "payment_intent.requires_capture"              |               |
```

### Refund Flow

```
Merchant          Payment Svc         Ledger          Card Network      Webhook Svc
   |                  |                  |                  |                |
   |-- POST refund -->|                  |                  |                |
   |  amount: $15.00  |                  |                  |                |
   |                  |                  |                  |                |
   |                  |-- validate:      |                  |                |
   |                  |  payment exists? |                  |                |
   |                  |  status=succeeded?                  |                |
   |                  |  refund <= remaining?               |                |
   |                  |                  |                  |                |
   |                  |-- BEGIN TXN:     |                  |                |
   |                  |  INSERT refund   |                  |                |
   |                  |   (status=pending)                  |                |
   |                  |  UPDATE payment  |                  |                |
   |                  |   amount_refunded += $15            |                |
   |                  |  INSERT ledger --|->               |                |
   |                  |   entries        |  DEBIT merchant |                |
   |                  |                  |  DEBIT platform |                |
   |                  |                  |  CREDIT network |                |
   |                  |  COMMIT          |                  |                |
   |                  |                  |                  |                |
   |<-- 200 refund ---|                  |                  |                |
   |   status=pending |                  |                  |                |
   |                  |                  |                  |                |
   |                  |-- submit refund -|----------------->|                |
   |                  |   to network     |                  |                |
   |                  |   (async)        |                  |                |
   |                  |                  |                  |                |
   |                  |     ... hours to days later ...     |                |
   |                  |                  |                  |                |
   |                  |<-- refund -------|------ confirmed -|                |
   |                  |   completed      |                  |                |
   |                  |                  |                  |                |
   |                  |-- UPDATE refund  |                  |                |
   |                  |  status=succeeded|                  |                |
   |                  |                  |                  |                |
   |                  |-- emit webhook --|------------------|--------------->|
   |                  |                  |                  |                |
   |                  |                  |                  |    deliver to  |
   |                  |                  |                  |<-- merchant ---|
   |                  |                  |                  |  "refund.      |
   |                  |                  |                  |   succeeded"   |
```

### Settlement / Payout Flow

```
+------------------------------------------------------------------+
|  Daily Settlement Batch                                           |
|                                                                   |
|  Nightly batch job (runs at 00:00 UTC):                          |
|                                                                   |
|  1. Query all payments with status=succeeded                     |
|     settled_at IS NULL                                            |
|     created_at < (now - settlement_delay)                        |
|     → settlement_delay = 2 days (fraud detection window)         |
|                                                                   |
|  2. For each merchant, calculate net payout:                     |
|     +-----------------------------------------------+            |
|     | Merchant: ACME Corp                           |            |
|     |                                                |            |
|     | Gross charges:        $45,230.00               |            |
|     | - Refunds:            -$1,200.00               |            |
|     | - Platform fees:      -$1,341.57               |            |
|     | - Dispute losses:       -$150.00               |            |
|     | - Reserve hold (10%): -$4,253.84               |            |
|     |                                                |            |
|     | Net payout:           $38,284.59               |            |
|     +-----------------------------------------------+            |
|                                                                   |
|  3. Create payout record + ledger entries:                       |
|     DEBIT  merchant:acme:usd     $38,284.59                     |
|     CREDIT payout:acme:batch-456 $38,284.59                     |
|                                                                   |
|  4. Submit ACH/wire transfer to merchant's bank                  |
|                                                                   |
|  5. ACH settles in 1-2 business days                              |
|     → Update payout status to "paid"                              |
|     → Webhook: "payout.paid"                                      |
+------------------------------------------------------------------+
```

---

## 8. Tokenization & PCI Compliance

```
+------------------------------------------------------------------+
|  PCI DSS Scope Minimization via Tokenization                     |
|                                                                   |
|  Problem: Handling raw card numbers (PAN) puts entire             |
|  infrastructure in PCI scope → massive compliance burden          |
|                                                                   |
|  Solution: Token Vault isolates card data                        |
|                                                                   |
|  +------------------+         +---------------------------+      |
|  | PCI Zone          |         | Non-PCI Zone               |      |
|  | (Token Vault)     |         | (Everything else)           |      |
|  |                   |         |                             |      |
|  | • Receives raw    |         | • Only sees tokens          |      |
|  |   card numbers    |         |   (pm_card_visa_4242)       |      |
|  | • Encrypts with   |         | • Cannot reverse token      |      |
|  |   AES-256-GCM     |         |   to card number            |      |
|  | • Stores in HSM-  |         | • Dramatically reduces      |      |
|  |   backed vault    |         |   PCI audit scope           |      |
|  | • Returns token   |         |                             |      |
|  |                   |         |                             |      |
|  | < 5 servers       |         | 100+ servers (out of scope) |      |
|  +------------------+         +---------------------------+      |
|                                                                   |
|  Tokenization flow:                                               |
|  1. Client (browser) sends card to Stripe.js endpoint            |
|     (card number never touches merchant's server!)               |
|  2. Token Vault encrypts + stores → returns token "pm_abc123"   |
|  3. Merchant uses token for all subsequent operations             |
|  4. On authorization: Payment Svc sends token to Token Vault     |
|     → Vault decrypts in secure enclave → sends to network        |
|     → Card number exists in memory for < 100ms                   |
|                                                                   |
|  Key management:                                                  |
|  • Master keys stored in HSM (Hardware Security Module)          |
|  • Key rotation every 90 days (automatic, zero-downtime)         |
|  • Envelope encryption: data key encrypts card,                  |
|    master key encrypts data key                                   |
|  • Destruction: when card deleted, data key destroyed            |
|    → encrypted card data becomes irrecoverable                   |
+------------------------------------------------------------------+
```

---

## 9. Fraud Detection Engine

```
+------------------------------------------------------------------+
|  Real-Time Fraud Scoring Pipeline                                 |
|                                                                   |
|  Every payment passes through fraud engine BEFORE network call:   |
|                                                                   |
|  Input signals:                                                    |
|  +-----------------------------------------------------------+  |
|  | Signal                  | Weight | Example                  |  |
|  +-------------------------+--------+--------------------------+  |
|  | Card BIN country vs IP  | High   | US card from Nigeria IP  |  |
|  | Transaction velocity    | High   | 20 charges in 5 min      |  |
|  | Amount anomaly          | Medium | $5K on card that avg $50 |  |
|  | Device fingerprint      | Medium | New device, no history   |  |
|  | Email domain age        | Low    | Created 2 hours ago      |  |
|  | Shipping ≠ billing addr | Medium | Different countries       |  |
|  | Historical fraud rate   | High   | This BIN range: 8% fraud |  |
|  | 3DS authentication      | High   | 3DS passed = lower risk  |  |
|  +-------------------------+--------+--------------------------+  |
|                                                                   |
|  ML Model (real-time scoring):                                    |
|  • Gradient boosted trees (XGBoost) — < 10ms inference           |
|  • Trained on billions of historical transactions                |
|  • Updated daily with latest fraud patterns                      |
|  • Output: risk score 0-100                                      |
|                                                                   |
|  Decision:                                                        |
|  Score 0-25:  ALLOW (proceed with authorization)                 |
|  Score 25-75: CHALLENGE (trigger 3D Secure verification)         |
|  Score 75-100: BLOCK (reject, notify merchant)                   |
|                                                                   |
|  Thresholds configurable per merchant (Radar rules in Stripe)    |
+------------------------------------------------------------------+
```

---

## 10. Handling Edge Cases

### Partial Capture / Over-Capture

```
Authorized: $100.00

Case 1: Partial capture (customer removed items from cart)
  → Capture $75.00. Remaining $25.00 auth released to cardholder.
  → Ledger: DEBIT network $75, CREDIT merchant $72.82, CREDIT fees $2.18

Case 2: Multi-capture (ship items separately)
  → Capture $40.00 (shipment 1)
  → Capture $35.00 (shipment 2)
  → Total captured: $75.00. Remaining $25.00 released.
  → Each capture is a separate ledger transaction.
```

### Authorization Expiry

```
Problem: Merchant authorizes $100 but never captures

  Card network rules:
    Visa: auth expires in 7 days
    Mastercard: auth expires in 7-30 days (varies)
    If not captured → funds released to cardholder automatically

  Our handling:
    Background job scans for old uncaptured auths
    After 6 days: send webhook warning merchant
    After 7 days: mark as expired, release ledger hold
    Merchant can re-authorize if needed
```

### Network Timeout (Ambiguous Response)

```
Problem: We send auth request to Visa → timeout after 30s
         Did Visa approve it? Decline it? Never receive it?

  This is the HARDEST edge case in payments.

  Solution: Status Inquiry + Reconciliation

  1. Immediately: mark payment as "processing" (not succeeded/failed)
  2. After 30s: send "status inquiry" to Visa
     → "I sent request REQ-123, what happened?"
     → Visa responds: approved / declined / not found
  3. If "not found": safe to retry with new request ID
  4. If approved/declined: update our state accordingly
  5. If status inquiry also times out:
     → Mark as "requires_manual_review"
     → Nightly reconciliation batch compares our records
       with network settlement files
     → Catches ALL discrepancies within 24 hours
```

### Double-Charge Prevention on Retry

```
Scenario:
  1. Merchant calls POST /confirm (Idempotency-Key: IK-123)
  2. We auth with Visa → approved → our server crashes BEFORE responding
  3. Merchant times out → retries with same IK-123
  4. We see IK-123 exists with status="processing" (not completed)

  Handling:
  → Check: is there a network_txn_id for this payment?
  → Yes: we already sent to Visa and it was approved
  → Return the original approval result (don't send to Visa again)
  → Update idempotency cache with final result

  If network_txn_id is NULL:
  → We crashed before sending to Visa
  → Safe to send authorization now (first attempt from network's POV)
```

### Dispute / Chargeback Handling

```
Customer calls bank: "I didn't make this purchase!"

Timeline:
  Day 0:   Issuer initiates dispute, debits merchant's account
  Day 1:   We receive dispute notification from network
           → Webhook: "charge.dispute.created"
           → Ledger: DEBIT merchant, CREDIT dispute_reserve
  Day 1-21: Merchant submits evidence (receipt, tracking, logs)
           → We forward evidence to network (representment)
  Day 30-75: Issuer reviews evidence
           → Won: funds returned to merchant (CREDIT merchant)
           → Lost: funds stay with customer (dispute_reserve cleared)
           → Webhook: "charge.dispute.closed" with outcome

  Financial impact tracking:
  dispute_rate = disputes / total_charges (by merchant)
  If > 1%: warn merchant (Visa/MC programs penalize high dispute rates)
  If > 2%: risk of losing processing privileges entirely
```

---

## 11. Tradeoffs & Design Decisions Summary

| Decision | Option A | Option B | Chosen | Why |
|----------|----------|----------|--------|-----|
| **Idempotency** | DB-based (check before every write) | Redis SETNX + DB | Redis + DB | Redis for fast dedup (< 1ms); DB as durable backup |
| **Ledger** | Single-entry (simple balance tracking) | Double-entry bookkeeping | Double-entry | Provable correctness; audit trail; catches bugs via balance invariant |
| **Money representation** | Float/decimal | Integer cents (BIGINT) | Integer cents | Floats have rounding errors; $29.99 = 2999 cents; zero ambiguity |
| **Card storage** | Encrypt in main DB | Isolated Token Vault + HSM | Token Vault | Minimizes PCI scope from 100+ servers to < 5; massive compliance savings |
| **Network routing** | Single acquirer | Smart routing with fallback | Smart routing | 2-5% auth rate improvement = millions in recovered revenue |
| **State transitions** | Eventual consistency | ACID transactions | ACID | Money requires absolute correctness; payment + ledger + event = atomic |
| **Webhook delivery** | Synchronous (block on merchant) | Async with retry queue | Async + retry | Merchant downtime shouldn't block our pipeline; retry up to 3 days |
| **Settlement** | Real-time (per-transaction) | Daily batch | Daily batch | Reduces ACH costs; provides fraud review window; industry standard |
| **Fraud scoring** | Rules-only | ML model + rules | ML + rules | ML catches patterns rules can't; rules handle explicit policies |
| **Consistency** | Optimistic (retry on conflict) | Pessimistic (lock on payment) | Pessimistic | Payment state transitions must be serialized; no lost updates |

---

## 12. Reliability & Fault Tolerance

### Single Points of Failure & Mitigations

```
+--------------------------------------------------------------+
| Component          | SPOF Risk | Mitigation                  |
+--------------------+-----------+-----------------------------+
| API Gateway        | Low       | Stateless, multi-AZ, LB    |
|                    |           | auto-scaling                |
+--------------------+-----------+-----------------------------+
| Payment Service    | Medium    | Stateless; idempotency      |
|                    |           | ensures safe retries;       |
|                    |           | multi-instance active-active|
+--------------------+-----------+-----------------------------+
| Token Vault        | Critical  | HSM-backed; multi-AZ with   |
|                    |           | synchronous replication;    |
|                    |           | key escrow for DR           |
+--------------------+-----------+-----------------------------+
| Database (primary) | Critical  | Synchronous standby +       |
|                    |           | automatic failover (< 30s); |
|                    |           | point-in-time recovery      |
+--------------------+-----------+-----------------------------+
| Card network conn  | High      | Multiple acquirer           |
|                    |           | connections; circuit breaker;|
|                    |           | failover routing            |
+--------------------+-----------+-----------------------------+
| Redis (idempotency)| Medium    | Cluster with replicas;      |
|                    |           | on failure: fall back to DB |
|                    |           | idempotency check (slower)  |
+--------------------+-----------+-----------------------------+
| Fraud engine       | Medium    | Fallback to rules-only mode |
|                    |           | if ML service is down;      |
|                    |           | never block payments due to |
|                    |           | fraud engine failure        |
+--------------------+-----------+-----------------------------+
```

### Disaster Recovery

```
+------------------------------------------------------------------+
|  RPO and RTO by Component                                        |
|                                                                   |
|  +------------------+---------+---------+----------------------+ |
|  | Component        | RPO     | RTO     | Strategy             | |
|  +------------------+---------+---------+----------------------+ |
|  | Payment DB       | 0 (sync)| < 30s   | Sync standby + auto  | |
|  |                  |         |         | failover              | |
|  +------------------+---------+---------+----------------------+ |
|  | Ledger DB        | 0 (sync)| < 30s   | Same as above; ledger| |
|  |                  |         |         | MUST have zero loss   | |
|  +------------------+---------+---------+----------------------+ |
|  | Token Vault      | 0 (sync)| < 60s   | HSM replication;     | |
|  |                  |         |         | encrypted backup      | |
|  +------------------+---------+---------+----------------------+ |
|  | Redis            | < 1s    | < 15s   | Cluster failover;    | |
|  |                  |         |         | AOF everysec          | |
|  +------------------+---------+---------+----------------------+ |
|  | Event store      | < 1s    | < 60s   | Async replica; events| |
|  |                  |         |         | can be replayed       | |
|  +------------------+---------+---------+----------------------+ |
|                                                                   |
|  Cross-region DR:                                                 |
|  • Active-passive across regions                                 |
|  • Async replication (RPO: seconds)                              |
|  • Manual failover (RTO: minutes)                                |
|  • Card network connections in both regions (pre-established)    |
+------------------------------------------------------------------+
```

### Reconciliation — The Ultimate Safety Net

```
+------------------------------------------------------------------+
|  Daily Reconciliation (runs every night)                          |
|                                                                   |
|  Three-way reconciliation:                                        |
|                                                                   |
|  Source 1: Our payment records                                    |
|  Source 2: Card network settlement files (received daily)        |
|  Source 3: Our ledger balances                                    |
|                                                                   |
|  Compare:                                                         |
|  ✓ Every payment in our DB has a matching network record         |
|  ✓ Every network record has a matching payment in our DB         |
|  ✓ Amounts match exactly (to the cent)                           |
|  ✓ Ledger sum(debits) = sum(credits) for every account           |
|  ✓ Merchant balances = sum of their ledger entries               |
|                                                                   |
|  Mismatches → alert on-call engineer IMMEDIATELY                 |
|  Common causes:                                                   |
|  • Timeout: we think declined, network approved                  |
|  • Race condition: double processing                              |
|  • Network error: settlement file had wrong amount               |
|                                                                   |
|  This is the LAST LINE OF DEFENSE.                               |
|  Even if every other safeguard fails,                            |
|  reconciliation catches it within 24 hours.                      |
+------------------------------------------------------------------+
```

---

## 13. Full System Architecture (Production-Grade)

```
                              +---------------------------+
                              |      Load Balancer         |
                              |   (L7, multi-AZ, TLS 1.3) |
                              +------------+--------------+
                                           |
              +----------------------------+----------------------------+
              |                            |                            |
     +--------v--------+         +--------v--------+         +--------v--------+
     | API Gateway (a)  |         | API Gateway (b)  |         | API Gateway (c)  |
     | • Rate limit     |         | • Rate limit     |         | • Rate limit     |
     | • API key auth   |         | • API key auth   |         | • API key auth   |
     | • Idempotency    |         | • Idempotency    |         | • Idempotency    |
     +--------+---------+         +--------+---------+         +--------+---------+
              |                            |                            |
              +----------------------------+----------------------------+
                                           |
                    +----------------------+----------------------+
                    |                      |                      |
     +--------------v-----------+  +-------v----------+  +-------v-----------+
     |    Payment Service        |  | Fraud Engine      |  | Webhook Service   |
     |    (Orchestrator)         |  |                   |  |                    |
     |                           |  | • ML scoring      |  | • Async delivery  |
     | • State machine           |  |   (< 10ms)        |  | • Retry w/ backoff|
     | • Network routing         |  | • Rules engine    |  | • Signature HMAC  |
     | • Retry / recovery        |  | • 3DS trigger     |  | • DLQ for failures|
     | • Ledger writes (atomic)  |  |                   |  |                    |
     +-----------+---------------+  +-------+-----------+  +--------+----------+
                 |                          |                        |
    +------------+------------+             |                        |
    |            |            |             |                        |
    v            v            v             v                        v
+--------+ +---------+ +----------+ +----------+           +---------+
|Idempot.| | Ledger  | | Payment  | | Feature  |           | Event   |
|Cache   | | Service | | DB       | | Store    |           | Store   |
|(Redis) | | (append | |(Postgres | |(real-time |           |(Kafka)  |
|        | |  only)  | | sharded) | | signals)  |           |         |
+--------+ +---------+ +----------+ +----------+           +---------+
                                |
                    +-----------+-----------+
                    |                       |
           +-------v--------+     +--------v--------+
           | Token Vault     |     | Payment Gateway  |
           | (PCI Zone)      |     | (Network Router)  |
           |                 |     |                    |
           | • HSM keys      |     | • Visa             |
           | • AES-256-GCM   |     | • Mastercard       |
           | • < 5 servers   |     | • Amex              |
           +-----------------+     | • ACH               |
                                   | • Apple/Google Pay  |
                                   +--------------------+
                                           |
                              +------------+------------+
                              |            |            |
                         +----v---+   +----v---+  +----v---+
                         | Visa   |   |Master  |  | ACH    |
                         | Net    |   | card   |  | Fed    |
                         +--------+   +--------+  +--------+


Background Services:
+------------------------------------------------------------------+
|  +-------------------+  +-------------------+  +----------------+ |
|  | Settlement Engine  |  | Reconciliation     |  | Subscription   | |
|  | (daily payout      |  | (3-way nightly     |  | Engine          | |
|  |  batch → ACH)      |  |  match: our DB vs  |  | (recurring      | |
|  |                    |  |  network files vs   |  |  billing,       | |
|  |                    |  |  ledger balances)   |  |  dunning)       | |
|  +-------------------+  +-------------------+  +----------------+ |
|                                                                    |
|  +-------------------+  +-------------------+  +----------------+ |
|  | Auth Expiry        |  | Dispute Manager    |  | Reporting /     | |
|  | Watchdog           |  | (chargeback flow,  |  | Analytics       | |
|  | (release expired   |  |  evidence submit,  |  | (merchant       | |
|  |  authorizations)   |  |  representment)    |  |  dashboard)     | |
|  +-------------------+  +-------------------+  +----------------+ |
+------------------------------------------------------------------+


Monitoring:
+------------------------------------------------------------------+
|  Key Metrics:                                                     |
|  • Authorization rate (by network, by BIN, by merchant)          |
|  • Payment latency (p50, p99) — target < 2s e2e                 |
|  • Decline rate and reasons                                      |
|  • Fraud score distribution                                      |
|  • Refund rate (by merchant)                                     |
|  • Dispute rate (by merchant — alert if > 1%)                   |
|  • Ledger balance invariant (debits = credits)                   |
|  • Webhook delivery success rate                                  |
|  • Network health (per acquirer latency, error rate)             |
|  • Idempotency cache hit rate                                    |
|                                                                   |
|  Alerts:                                                          |
|  RED   — Ledger imbalance detected (debits ≠ credits)           |
|  RED   — Auth rate drops > 10% in 5 min (network issue)         |
|  RED   — Token Vault unreachable                                 |
|  RED   — Reconciliation mismatch > $1,000                        |
|  WARN  — Webhook delivery failure rate > 5%                      |
|  WARN  — Payment p99 latency > 5 seconds                        |
|  WARN  — Merchant dispute rate approaching 1%                    |
+------------------------------------------------------------------+
```

---

## 14. Subscription / Recurring Billing

```
+------------------------------------------------------------------+
|  Subscription Lifecycle                                           |
|                                                                   |
|  1. Create subscription:                                          |
|     customer=cus_123, plan=monthly_pro ($49/mo)                  |
|     billing_anchor=1st of month                                   |
|                                                                   |
|  2. Billing engine (runs daily):                                  |
|     SELECT subscriptions                                          |
|     WHERE next_billing_date <= NOW()                              |
|       AND status = 'active'                                       |
|                                                                   |
|  3. For each due subscription:                                    |
|     → Create payment intent ($49.00)                              |
|     → Attempt charge on stored payment method                    |
|     → If succeeds: advance next_billing_date by 1 month          |
|     → If fails: enter dunning cycle                               |
|                                                                   |
|  Dunning (failed payment recovery):                               |
|  +---+--------------------------------------------------+        |
|  |Day| Action                                            |        |
|  +---+--------------------------------------------------+        |
|  | 0 | Charge fails → retry in 3 days, email customer   |        |
|  | 3 | Retry charge → fails → retry in 5 days           |        |
|  | 8 | Retry charge → fails → email "update card"       |        |
|  |14 | Final retry → fails → cancel subscription        |        |
|  |   | → webhook: "customer.subscription.deleted"       |        |
|  +---+--------------------------------------------------+        |
|                                                                   |
|  Proration (mid-cycle plan change):                               |
|  Customer upgrades from $49 → $99 on day 15 of 30:              |
|  Unused on old plan: $49 × (15/30) = $24.50 credit              |
|  Remaining on new plan: $99 × (15/30) = $49.50 charge           |
|  Net charge: $49.50 - $24.50 = $25.00                           |
+------------------------------------------------------------------+
```

---

## 15. Interview Tips

1. **Lead with correctness, not performance** — "The #1 requirement is money never gets lost or double-counted." Explain idempotency keys, ACID transactions, double-entry ledger. Performance is secondary to correctness for payments.

2. **Draw the payment lifecycle state machine** — requires_payment_method → requires_confirmation → requires_capture → succeeded → refunded. Show you understand the two-phase (auth + capture) model and why it exists (hotel holds, preorders).

3. **Explain idempotency concretely** — Walk through the retry scenario: merchant sends charge, times out, retries with same key. Show how SETNX in Redis prevents duplicate processing, and how stored results return the original response.

4. **Double-entry ledger is the showstopper answer** — Most candidates say "store balances in a column." Explain: every money movement creates debit + credit entries; sum must always be zero; corrections are reversing entries, never updates. This invariant catches bugs, fraud, and data corruption.

5. **Use integers for money, never floats** — $29.99 = 2999 cents as BIGINT. Floating point: 0.1 + 0.2 = 0.30000000000000004. In payments, one cent off in a billion transactions = audit nightmare.

6. **Tokenization minimizes PCI scope** — Card numbers never touch your main infrastructure. Token Vault (< 5 servers in PCI scope) vs entire platform (100+ servers). Dramatically reduces audit cost and security risk.

7. **Network timeout is the hardest edge case** — Don't handwave "just retry." Explain: timeout is ambiguous (approved? declined? lost?). Solution: status inquiry to network + nightly reconciliation as safety net.

8. **Smart routing improves auth rates** — Not all acquirers are equal. Routing based on BIN, country, card type, historical performance can lift auth rates 2-5%. At scale, this is millions in recovered revenue.

9. **Reconciliation is the last line of defense** — Three-way nightly comparison: our records vs network settlement files vs ledger balances. Any mismatch = alert. This catches everything that all other safeguards missed.

10. **End with fraud and disputes** — Mention ML scoring (< 10ms real-time), 3DS challenges for medium-risk, and the dispute lifecycle. Show you understand the business impact: > 1% dispute rate = card network penalties.
