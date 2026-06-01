# Payment Gateway : Payment Gateway

An enterprise-grade, high-throughput, low-latency, globally distributed Payment Gateway processing 10,000+ Transactions Per Second (TPS) across 100+ currencies with 99.999% operational availability and strict PCI-DSS Level 1 compliance.

---

## 1. Executive Summary & Core Pillars

A Payment Gateway acts as a secure financial intermediary orchestrating transactions between customers, merchants, acquiring banks, card networks, and issuing banks. Unlike highly available social networking or e-commerce systems that prioritize **eventual consistency**, a payment system operates under strict **strong consistency** requirements for its financial ledger while maintaining ultra-low latency profiles for API operations.

### Key Metrics & Service Level Objectives (SLOs)

* **Peak Throughput:** 10,000+ TPS sustained, architected to auto-scale horizontally up to 100,000+ TPS.
* **Availability:** 99.999% uptime ($\le$ 5.26 minutes of unscheduled downtime per calendar year).
* **API P95/P99 Latency:** Core Authorization response returned to merchant in $\le$ 200ms (P95) and $\le$ 500ms (P99), isolating slow synchronous downstream banking networks ($\approx$ 1–5s) via aggressive timeout barriers, circuit breaking, and asynchronous fulfillment queues.
* **Data Integrity:** Zero-loss ledger execution. Invariant: $\sum \text{Debits} \equiv \sum \text{Credits}$ across all financial domains at any point in time.

---

## 2. End-to-End System Architecture

The architecture relies on a highly decoupled, microservices-driven framework communicating via synchronous internal gRPC endpoints for atomic transactional pathing, and asynchronous event streams (Apache Kafka) for post-authorization lifecycle management.

### Architectural Diagram

```
                     +---------------------------+
                     |    Merchant Application   |
                     +---------------------------+
                                   |
                                   | HTTPS / TLS 1.3
                                   v
                     +---------------------------+
                     |    Envoy API Gateway      |
                     +---------------------------+
                                   |
         +-------------------------+-------------------------+
         | gRPC (Sync)             | gRPC (Sync)             | gRPC (Sync)
         v                         v                         v
+------------------+      +------------------+      +------------------+
| Payment Service  |----->|  Fraud Service   |      |  Ledger Service  |
+------------------+      +------------------+      +------------------+
  |              |                  |                         |
  | Tokenizes    | Read/Write       | Cache Signals           | Acid DB Write
  v              v                  v                         v
+-------+      +---------------+  +---------------+         +---------------+
| Vault |      | Redis Cluster |  | DynamoDB      |         | Aurora Postgre|
+-------+      +---------------+  +---------------+         +---------------+
  |                      |
  | Synchronous Path     +--------------------+
  v                                           |
+------------------------------------+        |
| Acquiring Bank / Card Networks     |        |
| (Visa, Mastercard, Amex, Rail)     |        |
+------------------------------------+        |
  |                                           |
  | Async Event Notification                  |
  v                                           v
+---------------------------------------------------------------------------+
|                            Apache Kafka Cluster                           |
|  Topics: payment-auth, refund-events, settlement-events, DLQ              |
+---------------------------------------------------------------------------+
         |                                                   |
         v (Consumer)                                        v (Consumer)
+-----------------------+                           +-----------------------+
|  Settlement Service   |                           |    Webhook Service    |
+-----------------------+                           +-----------------------+
         |                                                   |
         | Batch Files (NACHA / ISO 20022)                   | Asynchronous HTTP Post
         v                                                   v
+-----------------------+                           +-----------------------+
|  Clearing Houses /    |                           |   Merchant Endpoints  |
|  Partner Banks        |                           |   (Retry with Backoff)|
+-----------------------+                           +-----------------------+

```

### Deconstructed Domain Microservices

1. **Envoy API Gateway Layer:** Handles transport-layer security (TLS 1.3 termination), rate-limiting per merchant using a distributed Token Bucket algorithm in Redis, JWT/OAuth2 payload authentication, and global path routing.
2. **Core Payment Service:** Orchestrates individual transaction lifecycles, maps internal states, coordinates distributed executions, and manages card data tokenization procedures.
3. **Real-Time Fraud & Risk Mitigation Engine:** Executes real-time machine learning inference scoring alongside dynamic regulatory rule evaluations to drop malicious payloads before card network submission.
4. **Immutable Double-Entry Ledger Service:** Maintains the definitive, single-source-of-truth financial record for all asset/liability shifts within the platform.
5. **Asynchronous Settlement Engine:** Periodically groups authorized and captured transactions into discrete, currency-specific clearing files formatted to banking protocols.
6. **Reliable Webhook Notification Gateway:** Delivers programmatic state changes back to external merchant nodes utilizing guarantee frameworks.

---

## 3. Deep-Dive Functional Engineering

### The Core Transactional Lifecycle & State Machine

Transactions flow strictly through a non-reversible state machine backed by persistent storage engine guarantees.

```
 [Created] ---> [Fraud_Reviewing] ---> [Fraud_Blocked] (Terminal)
     |
     v
 [Authorized] ---> [Capture_Pending] ---> [Captured] ---> [Settling] ---> [Settled] (Terminal)
     |                                      |
     v                                      v
 [Auth_Expired] (Terminal)              [Refund_Initiated] ---> [Refunded] (Terminal)

```

* `Created`: Payload verified, internal ID provisioned, and idempotency lock claimed.
* `Fraud_Reviewing` / `Fraud_Blocked`: Undergoing evaluation or marked as high-risk.
* `Authorized`: Funds verified and blocked on the cardholder's account by the issuing bank. Holds a valid authorization code (`auth_code`).
* `Captured`: Request executed by the merchant to pull the authorized funds from the cardholder.
* `Settled`: Funds successfully wired to the merchant's account via clearing mechanisms.

### Strict Idempotency & Concurrency Design

Double-spending via network latency or aggressive client retry behavior represents an existential failure mode in payments. The system prevents duplicate resource generation through an end-to-end Idempotency Framework.

#### The Race Condition Vulnerability Window

If Request A and Request B carry identical keys and strike the API Gateway concurrently, a traditional read-then-write database validation model contains a race condition window allowing both requests to execute downstream.

```
Client Request A (Retry) ----------> [Gateway] ---> DB Lookup (Miss) --------> Lock & Execute
Client Request B (Concurrent) ------> [Gateway] ---> DB Lookup (Miss) --------> Lock & Execute

```

#### Distributed Locking Implementation Pattern

To handle this, we employ a distributed Redis cluster utilizing atomic `SET key value NX PX milliseconds` operations alongside a stateful reservation mapping table:

```python
import redis
import json
import uuid

class IdempotencyManager:
    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client
        
    def process_payment_safely(self, idempotency_key: str, merchant_id: str, payment_payload: dict, execution_lambda) -> tuple:
        lock_key = f"lock:idempotency:{merchant_id}:{idempotency_key}"
        storage_key = f"data:idempotency:{merchant_id}:{idempotency_key}"
        
        # 1. Attempt to acquire distributed lock (Timeout 30s to prevent deadlocks)
        lock_acquired = self.redis.set(lock_key, "PROCESSING", nx=True, px=30000)
        
        if not lock_acquired:
            # Lock exists; client must wait or retrieve existing payload if finished
            cached_data = self.redis.get(storage_key)
            if cached_data:
                record = json.loads(cached_data)
                return record['status'], record['response']
            else:
                return "CONFLICT_RETRY_LATER", {"error": "Transaction is currently being processed."}
        
        try:
            # 2. Re-verify storage key inside lock boundary (Double-checked locking pattern)
            cached_data = self.redis.get(storage_key)
            if cached_data:
                record = json.loads(cached_data)
                return record['status'], record['response']
            
            # 3. Execute downstream financial side-effects via lambda closure
            status, execution_response = execution_lambda(payment_payload)
            
            # 4. Persist result into long-lived cache (e.g., 7-day TTL for audit purposes)
            payload_to_cache = {
                "status": status,
                "response": execution_response,
                "request_hash": uuid.uuid4().hex
            }
            self.redis.set(storage_key, json.dumps(payload_to_cache), ex=604800)
            return status, execution_response
            
        finally:
            # 5. Clean up lock key
            self.redis.delete(lock_key)

```

### Immutable Ledger Design (Double-Entry Bookkeeping)

The Ledger System guarantees financial correctness. It treats data entries as an append-only transaction journal. Balance updates are computed views derived over historical logs; fields containing accounts are never modified in place via `UPDATE` actions.

#### System Balance Invariants

$$\sum \text{Debits} - \sum \text{Credits} = 0$$

#### Schema Layout & Processing Example

When a customer purchases a item costing $100.00 USD from a merchant, the core application populates multiple relational entries within a single ACID transaction isolation layer.

```text
Transaction Context: TXN_99887766, Amount: $100.00 USD, Merchant Fee: 2.50% ($2.50 USD)
---------------------------------------------------------------------------------------
Entry ID  | Account ID                | Type   | Amount  | Currency | Description
----------+---------------------------+--------+---------+----------+------------------
L_001     | ACC_CUSTOMER_CASH_OUT     | DEBIT  | 100.00  | USD      | Card Authorization
L_002     | ACC_GATEWAY_CLEARING      | CREDIT | 100.00  | USD      | Card Authorization
L_003     | ACC_GATEWAY_CLEARING      | DEBIT  |  97.50  | USD      | Merchant Allocation
L_004     | ACC_MERCHANT_RECEIVABLE   | CREDIT |  97.50  | USD      | Merchant Allocation
L_005     | ACC_GATEWAY_CLEARING      | DEBIT  |   2.50  | USD      | Revenue Capture
L_006     | ACC_GATEWAY_REVENUE       | CREDIT |   2.50  | USD      | Processing Margin

```

### Real-Time Fraud & Automated Risk Mitigation

Every authorization payload maps against a sub-50ms inference pipeline executing rule sets and Machine Learning models (Gradient Boosted Decision Trees analyzing contextual graphs).

#### Scoring Boundaries & Mitigations

* **Score 0–30 (Low Risk):** Auto-routed directly to upstream card networks.
* **Score 31–80 (Medium Risk):** Demands step-up authentication mechanisms (3D Secure 2.2 biometric validation, SMS OTP, or manual operations review queue).
* **Score 81–100 (High Risk):** Hard drop at gateway boundary. Triggers fraud alerts and updates localized blocklists.

#### Analyzed High-Velocity Signals

1. **Velocity Tracking:** Detection of $N > 5$ distinct card numbers attempted from the same IP block within a 60-second sliding window.
2. **Impossible Travel Calculations:** Transaction computed in Mumbai, India, followed by an execution under the identical token footprint 4 minutes later from Chicago, USA.
3. **Device Fingerprint Polymorphism:** Identification of synthetic browser canvas attributes masking behind distinct user-agent signatures.

### Multi-Currency Conversion & Foreign Exchange (FX) Engine

Global operations introduce multi-layered exchange pricing components. The system separates the **Presentment Currency** (what the buyer views) from the **Settlement Currency** (what the merchant receives).

```
[Presentment Amount: INR 8,400] 
       |
       v (Fetched from Redis Cached FX Matrix + 150bps spread buffer)
[Internal Base Layer: USD 100.00] 
       |
       v (Card Network Clearing Routing)
[Settlement Amount: EUR 92.50]

```

To shield merchants against currency fluctuation risk during transaction execution times, the FX platform exposes an **FX Quote Lock**. The gateway guarantees an exchange rate for a configurable 20-minute window, embedding the quote ID directly into the ledger allocation payload.

---

## 4. Database Topography & Architecture

To sustain massive data footprints without structural degradation, data storage splits between localized Relational Storage (OLTP) for atomic transactional operations and Columnar Storage (OLAP) for historical search operations and reporting.

### Relational Schema Definitions (PostgreSQL Dialect)

```sql
-- Core Transactions Master Table
CREATE TABLE transactions (
    id BIGINT PRIMARY KEY,
    transaction_token VARCHAR(64) UNIQUE NOT NULL,
    merchant_id BIGINT NOT NULL,
    amount DECIMAL(18, 4) NOT NULL,
    currency VARCHAR(3) NOT NULL,
    status VARCHAR(24) NOT NULL,
    idempotency_key VARCHAR(255) NOT NULL,
    auth_code VARCHAR(32),
    fraud_score SMALLINT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL
);

-- Index optimization strategies for high throughput querying
CREATE UNIQUE INDEX idx_txn_merchant_id_idempotency 
ON transactions(merchant_id, idempotency_key);

CREATE INDEX idx_txn_created_at_status 
ON transactions(created_at, status) 
WHERE status = 'CAPTURE_PENDING';

-- Double Entry Micro-Ledger Table
CREATE TABLE ledger_entries (
    id BIGINT PRIMARY KEY,
    transaction_id BIGINT NOT NULL REFERENCES transactions(id),
    account_id VARCHAR(64) NOT NULL,
    entry_type VARCHAR(6) NOT NULL CHECK (entry_type IN ('DEBIT', 'CREDIT')),
    amount DECIMAL(18, 4) NOT NULL,
    currency VARCHAR(3) NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL
);

CREATE INDEX idx_ledger_account_currency 
ON ledger_entries(account_id, currency, created_at);

-- Multi-Partial Refund Processing Ledger
CREATE TABLE refunds (
    id BIGINT PRIMARY KEY,
    parent_transaction_id BIGINT NOT NULL REFERENCES transactions(id),
    refund_token VARCHAR(64) UNIQUE NOT NULL,
    amount DECIMAL(18, 4) NOT NULL,
    status VARCHAR(24) NOT NULL,
    reason_code VARCHAR(64),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL
);

```

### Database Scaling Strategies (Sharding & Partitioning)

* **Horizontal Sharding:** The `transactions` and `ledger_entries` tables are sharded horizontally via a hash of the `merchant_id`. This prevents cross-shard multi-row transactions, keeping all tables belonging to a single merchant localized within the same logical database node.
* **Time-Series Partitioning:** Within each shard, tables are partitioned by month based on the `created_at` timestamp. This design maintains compact primary indexes, allows swift cold-data archival procedures (detaching historical partitions to AWS S3/Parquet formats), and isolates disk I/O bottlenecks.

---

## 5. Resiliency, High Availability & Fault Tolerance

Operating at 99.999% availability over fragile, unpredictable banking infrastructure requires robust fault-isolation patterns.

### The Circuit Breaker Architecture

Downstream banking links and API connections integrate with a stateful Circuit Breaker mechanism to isolate issues and protect system performance.

```
       +--------------------------------------------+
       |                                            |
       v                                            |  Success Rate
  +----------+         Failure Threshold            |  Meets Target
  |  Closed  |-------- (e.g., >5% timeouts) ----->+ |
  +----------+                                    | |
       ^                                          v |
       |                                     +----------+
       |-------- After Cool-off Window ------|   Open   |
                & Successful Canary Requests +----------+
                                                  |
                                                  |  Immediate Fallback /
                                                  v  Fast-Failure
                                             [Degraded Mode]

```

* **Closed State:** Requests pass unimpeded to the banking network. Latency and error signatures are continuously evaluated across a sliding window.
* **Open State:** If downstream endpoints exhibit a high failure profile ($\ge 5\%$ error or timeout rates over a 100-request window), the circuit opens. Traffic immediately routes to an alternative acquiring bank (**Smart Routing**), or fails fast if no fallback options exist, preventing thread pool exhaustion.
* **Half-Open State:** After a 30-second cool-off window, a small canary fraction ($\approx 2\%$) of production transactions is allowed through. If these succeed, the circuit returns to a Closed state; any failure returns it immediately to Open.

### Graceful Degradation Strategies

If the Fraud Engine or internal secondary analytics nodes suffer complete cluster outages, the gateway drops back to fallback operational configurations defined by business risk configurations:

* *Fail-Open Configuration:* Allows transactions through with a default fraud score of 0, prioritizing transaction flow during massive system outages.
* *Fail-Secure Configuration:* Immediately rejects high-value transactions ($> \$5,000$ USD) while letting low-value checkouts pass with warnings.

---

## 6. Comprehensive Security, Compliance & Governance

### PCI-DSS Level 1 Technical Implementation

The system isolates the primary network infrastructure from toxic credit card data through an independent, hardened **Tokenization Vault**.

```
[Merchant UI / Elements] 
       |
       | Direct POST (Bypasses Merchant Servers Entirely)
       v
[Hardened Tokenization Vault] ---> Generates Ephemeral Token: `tok_v4_99211`
       |
       | Stores encrypted raw Primary Account Number (PAN) inside HSM
       v
[Card Data Returned to Core Payment Service as Token ONLY]

```

1. **Zero Raw Data Footprint:** Raw Primary Account Numbers (PANs), CVVs, or track data never strike the primary application memory pools, log aggregators, or database systems.
2. **Data Storage Isolation:** Card data directly posts to a decoupled, minimal-surface-area Tokenization Service. This layer maps card structures to randomly generated UUIDv4 strings (e.g., `tok_v4_9a8b7c`).
3. **Hardware Level Cryptography:** Within the vault, data is encrypted via hardware security modules (HSM) utilizing AES-256-GCM keys rotated automatically every 90 days.

### Audit Logs & Absolute Traceability

Every data shift, configuration change, or internal gRPC call writes a cryptographically structured entry into a tamper-proof write-once-read-many (WORM) storage bucket. Each log entry embeds a SHA-256 cryptographic chain hash linked to the preceding block payload, creating an immutable audit trail for forensic review.

---

## 7. Asynchronous Operations & Eventual Consistency

### Settlement Pipeline (Automated Batch Operations)

While authorizations occur synchronously within sub-second thresholds, capital settlement runs asynchronously via a nightly batch pipeline.

```
+------------------------------------------+
|  Captured Transactions (Postgres Month)  |
+------------------------------------------+
                     |
                     v (Scheduled EOD Cron / Airflow Trigger)
+------------------------------------------+
|      MapReduce / Spark Settlement Job    |
+------------------------------------------+
                     |
                     +---> Groups records by Acquirer & Settlement Currency
                     +---> Subtracts interchange processing fee structures
                     |
                     v
+------------------------------------------+
|   ISO 20022 XML / NACHA Clearing Files   |
+------------------------------------------+
                     |
                     v (Pushed via Secure SFTP with PGP Encryption)
+------------------------------------------+
|        Federal Reserve / Central Bank    |
+------------------------------------------+

```

### Webhook Engine Architecture & Delivery Guarantees

Merchants rely on real-time callbacks to confirm financial state shifts. The system utilizes an event-driven framework to process these notification lifecycles.

```
[Kafka Event: payment.captured] 
       |
       v
[Webhook Worker Engine] <--- Fetches Merchant Endpoint URL & HMAC Private Sign Key
       |
       v (HTTP POST with Header Signature `X-Gateway-Signature: t=171822,v1=sha256...`)
[Merchant Server Endpoint]

```

#### Resiliency Mechanisms

* **At-Least-Once Delivery Delivery Strategy:** Workers assume execution failure unless the endpoint returns an explicit `200 OK` response payload within a strict 5000ms window.
* **Exponential Jitter Backoff Schedule:** If an endpoint errors, the request re-queues using an exponential delay scale ($t \times 2^n + \text{random\_jitter}$), retrying up to 12 times over a 48-hour timeline before moving to a Dead Letter Queue (DLQ).
* **Replay Attack Defenses:** Every webhook payload includes an asymmetric cryptographic signature generated via a shared merchant secret key, alongside an epoch timestamp header. This enables client verification and blocks message-interception replay attacks.

---

## 8. Daily Financial Reconciliation (Batch Integrity Validation)

A payment gateway must match its internal records against the actual funds moving through corresponding bank accounts. This is accomplished via an automated, three-way financial reconciliation ledger job executed every 24 hours.

### The Three-Way Reconciliation Strategy

```
           +-----------------------------+
           |       Internal Ledger       |
           +-----------------------------+
                     /         \
                    /           \
     Reconciliation Check    Reconciliation Check
                  /               \
                 v                 v
  +--------------------+     +---------------------+
  |   Bank Clearing    |     |    Card Network     |
  |  Statement (Bank)  |     |  Settlement Report  |
  +--------------------+     +---------------------+

```

The system continuously cross-references three distinct data sources:

1. **Internal Database Ledger:** The system's transaction ledger entries recorded over the past 24 hours.
2. **Card Network Clearing Reports:** Raw processing logs delivered daily from card clearing networks (e.g., Visa EPX, Mastercard IPM files).
3. **Bank Account Statements:** Cash balance adjustments reported by downstream partner banking facilities via MT940 or CAMT.053 standard formats.

### Discrepancy Classification & Mitigation Workflow

When anomalies emerge during reconciliation matching runs, they are classified into operational buckets for automated or manual handling:

* **Unmatched Captured Transaction:** An entry is marked captured in the internal ledger but absent from the card network file. The engine flags the transaction for automatic capture re-submission within the next settlement batch.
* **Exchange Rate Discrepancy:** The card network clears a transaction at an exchange value that differs from the internal gateway quote block. The discrepancy is routed to an automated currency adjustment ledger to re-balance the ledger book.
* **Chargeback Event:** The network report shows an unprompted debit due to a customer dispute. The system creates a new transaction entity in the internal ledger under a `CHARGEBACK` status code, and automatically fires an event stream notification to the merchant.

---

## 9. Interview System Design Summary Evaluation

Use this architectural checklist during evaluations to ensure all design components align with core system requirements:

| System Goal | Design Pattern / Strategic Tool | Concrete Implementation Detail |
| --- | --- | --- |
| **Double Spend Prevention** | Distributed Locking + Idempotency Tables | Redis `SET NX` with 30s lock windows; 7-day transaction response caching. |
| **Financial Accuracy** | Append-Only Double-Entry Microledger | No in-place `UPDATE` commands; immutable balance views; $\sum \text{Debits} \equiv \sum \text{Credits}$. |
| **High Availability** | Active-Active Geo Routing + Circuit Breakers | Multi-region deployments with Envoy API routing; automated fallbacks via alternative acquiring networks. |
| **10,000+ TPS Scaling** | Horizontal Database Sharding | Sharding strategies driven by a hash of `merchant_id` combined with monthly time-series table partitioning. |
| **Security Compliances** | Isolated Hardware Tokenization Vaults | PAN data isolated inside specialized networks; underlying storage secured with AES-256-GCM hardware keys. |
| **Downstream Latency Shielding** | Event-Driven Asynchronous Processing | Microservices decoupled via Apache Kafka clusters; transactional notifications pushed via distributed webhook worker pools. |




<img width="1280" height="1600" alt="1" src="https://github.com/user-attachments/assets/c5b66916-9b61-48c6-9be2-a78855f3de48" />
<img width="1280" height="1600" alt="2" src="https://github.com/user-attachments/assets/15fe85ca-a408-4850-9b96-edce819c51a5" />
<img width="1280" height="1600" alt="3" src="https://github.com/user-attachments/assets/42ef4cf6-2bf1-42ba-a6a5-18722bc901af" />
<img width="1280" height="1600" alt="4" src="https://github.com/user-attachments/assets/094dd833-36ec-4547-a29e-93270a1d4aa8" />
<img width="1280" height="1600" alt="5" src="https://github.com/user-attachments/assets/653ed336-358a-4749-92ef-19be84480e96" />
<img width="1280" height="1600" alt="6" src="https://github.com/user-attachments/assets/646ca7b3-7d1b-490d-aedc-ee112bd3b007" />
<img width="1280" height="1600" alt="7" src="https://github.com/user-attachments/assets/f3a65a01-6fe6-45df-97a7-78f741961366" />
<img width="1280" height="1600" alt="8" src="https://github.com/user-attachments/assets/6238fe27-4a83-40e2-8b78-8de13ee667c4" />
<img width="1280" height="1600" alt="9" src="https://github.com/user-attachments/assets/32b4d64f-af87-4cf4-8742-c25c8c068a4b" />
<img width="1280" height="1600" alt="10" src="https://github.com/user-attachments/assets/2a5924ed-3d76-4df5-9dec-ce8b5bed859c" />
<img width="1280" height="1600" alt="11" src="https://github.com/user-attachments/assets/5d16dc18-3f92-4753-9434-516b114bbbb9" />
<img width="1280" height="1600" alt="12" src="https://github.com/user-attachments/assets/8a48bdf3-9297-4c79-aa16-9533218d8c2d" />
<img width="1280" height="1600" alt="13" src="https://github.com/user-attachments/assets/6969c5ce-a0e3-4961-a870-c7fb71503cec" />
<img width="1280" height="1600" alt="14" src="https://github.com/user-attachments/assets/d6d1b06c-24fa-47fc-90ce-d65f7d3d902b" />
<img width="1280" height="1600" alt="15" src="https://github.com/user-attachments/assets/4884723b-b025-4bfe-97b3-ba2a1bbf5dd6" />
<img width="1280" height="1600" alt="16" src="https://github.com/user-attachments/assets/d1e84783-9211-46ab-84dd-40f60f723715" />
<img width="1280" height="1600" alt="17" src="https://github.com/user-attachments/assets/d4e45eed-1bb8-498a-b431-190976be463b" />
<img width="1280" height="1600" alt="18" src="https://github.com/user-attachments/assets/9be43e16-6646-4a43-adc0-b1f3b278ded8" />
<img width="1280" height="1600" alt="19" src="https://github.com/user-attachments/assets/133aa8a7-a5f5-445f-a51e-ada2832e6440" />
<img width="1280" height="1600" alt="20" src="https://github.com/user-attachments/assets/0f42b60f-faec-4210-8ce2-53b590f5b0a7" />





















