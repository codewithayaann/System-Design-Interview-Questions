# Redis Failure Handling in a Distributed Rate Limiter

## 1. Problem: What happens when Redis goes down?

A distributed rate limiter often uses Redis as shared state.

Every application server may receive requests, but the rate-limit
counter must be consistent across servers. Redis provides a central,
low-latency place to read and update that state.

``` mermaid
flowchart LR
    C[Client] --> G[API Gateway]
    G --> R[Rate Limiter]
    R -->|Read / Update Counter| REDIS[(Redis)]
    R --> B[Backend Services]

    style C fill:#111827,stroke:#94a3b8,color:#fff
    style G fill:#0b2948,stroke:#2196f3,color:#fff
    style R fill:#063b2b,stroke:#00e676,color:#fff
    style REDIS fill:#3b0b0b,stroke:#ff3b30,color:#fff
    style B fill:#17113b,stroke:#8b5cf6,color:#fff
```

If Redis becomes unavailable, the rate limiter can no longer reliably
read or update distributed counters.

The important system-design question is not simply:

> "What do we do when Redis fails?"

It is:

> **"What should happen to requests when the rate-limit state is
> unavailable?"**

The two primary choices are:

-   **Fail Open** --- allow requests when the rate limiter cannot make a
    decision.
-   **Fail Closed** --- reject requests when the rate limiter cannot
    make a decision.

------------------------------------------------------------------------

# 2. The Failure Scenario

Under normal conditions:

``` mermaid
sequenceDiagram
    participant C as Client
    participant G as API Gateway
    participant R as Rate Limiter
    participant X as Redis
    participant B as Backend

    C->>G: HTTP Request
    G->>R: Check rate limit
    R->>X: Read / update counter
    X-->>R: Counter state
    R->>B: Allow request
    B-->>C: Response
```

When Redis fails:

``` mermaid
sequenceDiagram
    participant C as Client
    participant G as API Gateway
    participant R as Rate Limiter
    participant X as Redis
    participant B as Backend

    C->>G: HTTP Request
    G->>R: Check rate limit
    R->>X: Read / update counter
    X--xR: Timeout / connection failure

    alt Fail Open
        R->>B: Allow request
        B-->>C: Response
    else Fail Closed
        R-->>G: Reject request
        G-->>C: Error / retry response
    end
```

------------------------------------------------------------------------

# 3. Why Redis Failure Is Dangerous

Suppose an API normally allows:

**100 requests/minute per user**

Under normal operation, Redis maintains the shared counter.

``` mermaid
flowchart LR
    U[Client] --> RL[Rate Limiter]
    RL --> R[(Redis)]
    R -->|count = 100| RL
    RL -->|Limit reached| X[Reject]
```

If Redis disappears and the limiter simply allows everything:

``` mermaid
flowchart LR
    U[Many Clients] --> RL[Rate Limiter]
    RL -. Redis unavailable .-> R[(Redis ❌)]
    RL -->|No distributed counter| B[Backend]
    B --> DB[(Database)]

    style R fill:#3b0b0b,stroke:#ff3b30,color:#fff
    style B fill:#17113b,stroke:#8b5cf6,color:#fff
```

Traffic can change from:

``` text
100 req/min
       ↓
10,000+ req/min
```

The backend, database, downstream APIs, and infrastructure may suddenly
receive traffic that the rate limiter was designed to prevent.

------------------------------------------------------------------------

# 4. Fail Open

## Definition

**Fail Open means:**

> If the rate limiter cannot access Redis, allow the request to
> continue.

The priority is **availability**.

``` mermaid
flowchart LR
    C[Client] --> G[API Gateway]
    G --> RL[Rate Limiter]
    RL -->|Check counter| R[(Redis ❌)]
    R -. unavailable .-> RL
    RL -->|ALLOW| B[Backend Services]

    style RL fill:#063b2b,stroke:#00e676,color:#fff
    style R fill:#3b0b0b,stroke:#ff3b30,color:#fff
    style B fill:#17113b,stroke:#8b5cf6,color:#fff
```

### What you gain

-   Better availability
-   Users can continue using the service
-   A Redis outage does not automatically become an application outage
-   Useful for low-risk APIs

### What you lose

-   Rate-limit enforcement is weakened
-   Backend traffic may spike
-   Abuse protection may temporarily disappear
-   Database load can increase
-   Expensive downstream calls may become uncontrolled

------------------------------------------------------------------------

# 5. Fail Open --- Good Use Cases

Fail Open is generally more appropriate when temporary excess traffic is
acceptable.

## `/search`

``` mermaid
flowchart LR
    C[Client] --> G[Gateway]
    G --> RL[Rate Limiter]
    RL --> R[(Redis ❌)]
    RL -->|Fail Open| S[Search Service]

    style R fill:#3b0b0b,stroke:#ff3b30,color:#fff
    style S fill:#063b2b,stroke:#00e676,color:#fff
```

Why?

-   Search is often read-heavy.
-   A temporary increase in traffic may be preferable to making the
    entire search feature unavailable.
-   Search can often be protected further using caching, query limits,
    or backend autoscaling.

## `/public-content`

``` mermaid
flowchart LR
    C[Client] --> G[Gateway]
    G --> RL[Rate Limiter]
    RL --> R[(Redis ❌)]
    RL -->|Fail Open| P[Public Content]

    style R fill:#3b0b0b,stroke:#ff3b30,color:#fff
    style P fill:#063b2b,stroke:#00e676,color:#fff
```

Why?

-   Content is already public.
-   Authentication or transaction protection may not be involved.
-   Availability may be more important than strict request counting.

------------------------------------------------------------------------

# 6. Fail Closed

## Definition

**Fail Closed means:**

> If the rate limiter cannot determine whether a request is allowed,
> reject the request.

The priority is **safety and protection**.

``` mermaid
flowchart LR
    C[Client] --> G[API Gateway]
    G --> RL[Rate Limiter]
    RL -->|Check counter| R[(Redis ❌)]
    R -. unavailable .-> RL
    RL -->|REJECT| X[Request Blocked]

    style R fill:#3b0b0b,stroke:#ff3b30,color:#fff
    style X fill:#3b0b0b,stroke:#ff3b30,color:#fff
```

### What you gain

-   Strong protection during dependency failures
-   Prevents uncontrolled traffic
-   Protects authentication and transaction systems
-   Protects expensive downstream operations

### What you lose

-   Availability
-   Legitimate users may be blocked
-   Redis becomes part of the availability path
-   A Redis outage can become a user-facing outage

------------------------------------------------------------------------

# 7. Fail Closed --- Good Use Cases

## `/login`

``` mermaid
flowchart LR
    C[Client] --> G[Gateway]
    G --> RL[Rate Limiter]
    RL --> R[(Redis ❌)]
    RL -->|Fail Closed| X[Reject Login]

    style R fill:#3b0b0b,stroke:#ff3b30,color:#fff
    style X fill:#3b0b0b,stroke:#ff3b30,color:#fff
```

Why?

Login endpoints are common targets for brute-force attacks.

If rate limiting disappears during a Redis outage, an attacker may be
able to send a very large number of authentication attempts.

## `/payment`

``` mermaid
flowchart LR
    C[Client] --> G[Gateway]
    G --> RL[Rate Limiter]
    RL --> R[(Redis ❌)]
    RL -->|Fail Closed| X[Reject Transaction]

    style R fill:#3b0b0b,stroke:#ff3b30,color:#fff
    style X fill:#3b0b0b,stroke:#ff3b30,color:#fff
```

Why?

Payment APIs can trigger expensive or sensitive operations.

Allowing uncontrolled traffic may create:

-   Duplicate transaction attempts
-   Excessive downstream calls
-   Fraud or abuse opportunities
-   High financial or operational risk

------------------------------------------------------------------------

# 8. Fail Open vs Fail Closed

``` mermaid
flowchart TD
    A[Redis unavailable] --> Q{What matters more?}

    Q -->|Availability| FO[Fail Open]
    Q -->|Safety / Protection| FC[Fail Closed]

    FO --> FO1[Allow request]
    FO1 --> FO2[Backend may receive more traffic]

    FC --> FC1[Reject request]
    FC1 --> FC2[Backend remains protected]

    style FO fill:#063b2b,stroke:#00e676,color:#fff
    style FC fill:#3b0b0b,stroke:#ff3b30,color:#fff
```

  Decision             Fail Open                Fail Closed
  -------------------- ------------------------ ----------------
  Main priority        Availability             Safety
  Redis unavailable    Allow                    Reject
  Backend protection   Weaker                   Stronger
  User availability    Higher                   Lower
  Abuse protection     Weaker                   Stronger
  Best for             Low-risk APIs            Sensitive APIs
  Typical examples     Search, public content   Login, payment

------------------------------------------------------------------------

# 9. Endpoint-Specific Failure Policy

A common mistake is to apply one global policy to every endpoint.

Instead, define the failure behavior based on endpoint risk.

``` mermaid
flowchart TD
    API[API Request] --> P{Endpoint Policy}

    P --> L["/login"]
    P --> PAY["/payment"]
    P --> S["/search"]
    P --> PC["/public-content"]

    L --> FC1[Fail Closed]
    PAY --> FC2[Fail Closed]

    S --> FO1[Fail Open]
    PC --> FO2[Fail Open]

    style FC1 fill:#3b0b0b,stroke:#ff3b30,color:#fff
    style FC2 fill:#3b0b0b,stroke:#ff3b30,color:#fff
    style FO1 fill:#063b2b,stroke:#00e676,color:#fff
    style FO2 fill:#063b2b,stroke:#00e676,color:#fff
```

This gives the system a more granular policy.

``` text
/login          → Fail Closed
/payment        → Fail Closed
/search         → Fail Open
/public-content → Fail Open
```

------------------------------------------------------------------------

# 10. Redis High Availability

Fail Open and Fail Closed define what happens **when Redis fails**.

A second question is:

> Can we reduce the chance of Redis failing in the first place?

Use Redis high availability.

``` mermaid
flowchart TD
    RL[Rate Limiter] --> P[(Redis Primary)]
    P -->|Replication| R1[(Redis Replica)]
    P -->|Replication| R2[(Redis Replica)]

    P -. failure .-> F[Failover]
    F --> R1

    style P fill:#063b2b,stroke:#00e676,color:#fff
    style R1 fill:#0b2948,stroke:#2196f3,color:#fff
    style R2 fill:#0b2948,stroke:#2196f3,color:#fff
    style F fill:#3b0b0b,stroke:#ff3b30,color:#fff
```

The objective is to avoid a single Redis instance becoming a single
point of failure.

Typical building blocks include:

-   Primary
-   Replicas
-   Automatic failover
-   Health checks
-   Connection timeouts
-   Monitoring

------------------------------------------------------------------------

# 11. Redis Cluster

When scale becomes large, Redis can also be distributed across multiple
nodes.

``` mermaid
flowchart TD
    RL[Rate Limiter] --> C[(Redis Cluster)]

    C --> A[Node A]
    C --> B[Node B]
    C --> D[Node C]

    A --- A1[Shard]
    B --- B1[Shard]
    D --- D1[Shard]

    style C fill:#17113b,stroke:#8b5cf6,color:#fff
    style A fill:#0b2948,stroke:#2196f3,color:#fff
    style B fill:#063b2b,stroke:#00e676,color:#fff
    style D fill:#063b2b,stroke:#00d9ff,color:#fff
```

Redis Cluster can help with:

-   Horizontal scaling
-   Data partitioning
-   Higher throughput
-   Distribution of rate-limit keys

But clustering does not mean the application can ignore failure
handling.

The rate limiter still needs a strategy for connection errors, timeouts,
failover, and unavailable nodes.

------------------------------------------------------------------------

# 12. Local In-Memory Fallback

Another strategy is to temporarily use a local counter if Redis is
unavailable.

``` mermaid
flowchart TD
    RL[Rate Limiter] --> R{Redis available?}

    R -->|Yes| REDIS[(Redis)]
    R -->|No| LOCAL[Local In-Memory Counter]

    REDIS --> ALLOW[Allow / Reject]
    LOCAL --> ALLOW

    LOCAL -. Redis recovers .-> REDIS

    style REDIS fill:#063b2b,stroke:#00e676,color:#fff
    style LOCAL fill:#17113b,stroke:#8b5cf6,color:#fff
```

A simplified fallback sequence:

``` mermaid
flowchart LR
    A[1. Redis request fails] --> B[2. Local counter]
    B --> C[3. Apply local limit]
    C --> D[4. Redis recovers]
    D --> E[5. Resume distributed limiting]
```

This can preserve some protection while Redis is unavailable.

However, it introduces an important tradeoff.

------------------------------------------------------------------------

# 13. The Local Counter Problem

Imagine three application servers.

Each server has its own local counter.

``` mermaid
flowchart TD
    LB[Load Balancer]

    LB --> A[Server A]
    LB --> B[Server B]
    LB --> C[Server C]

    A --> CA[Local Counter A]
    B --> CB[Local Counter B]
    C --> CC[Local Counter C]
```

Suppose the intended global limit is:

``` text
100 requests/minute
```

If each server independently allows 100:

``` text
Server A → 100
Server B → 100
Server C → 100

Effective total → 300 requests/minute
```

The more servers you have, the larger the difference can become.

``` mermaid
flowchart LR
    A[100/min Server A] --> T[Global traffic]
    B[100/min Server B] --> T
    C[100/min Server C] --> T

    T --> X[Up to 300/min]

    style X fill:#3b0b0b,stroke:#ff3b30,color:#fff
```

So local fallback improves availability, but **weakens global rate-limit
accuracy**.

------------------------------------------------------------------------

# 14. When Local Fallback Makes Sense

Local fallback can be useful when:

-   A temporary Redis outage should not completely stop traffic.
-   The endpoint is relatively low risk.
-   Some protection is better than no protection.
-   The local limit is intentionally conservative.
-   The business can tolerate imperfect global enforcement.

Example:

``` text
Normal distributed limit:
100 req/min

Temporary local fallback:
30 req/min/server
```

This is not globally equivalent to 100 req/min, but it can reduce the
impact of a Redis outage.

The exact fallback limit should be selected based on the endpoint and
traffic model.

------------------------------------------------------------------------

# 15. Timeout Is Important

Never allow the rate limiter to wait indefinitely for Redis.

Bad:

``` mermaid
flowchart LR
    RL[Rate Limiter] --> R[(Redis)]
    R -->|Hangs| RL
    RL --> B[Backend]
```

If Redis is slow, rate-limit checks can consume application resources
and increase request latency.

Instead:

``` mermaid
flowchart LR
    RL[Rate Limiter] -->|Short timeout| R[(Redis)]
    R -->|Timeout| T[Failure Policy]
    T --> FO[Fail Open]
    T --> FC[Fail Closed]

    style R fill:#3b0b0b,stroke:#ff3b30,color:#fff
    style T fill:#17113b,stroke:#8b5cf6,color:#fff
```

The rate limiter should have a bounded Redis timeout.

The exact timeout depends on latency requirements and infrastructure,
but the key principle is:

> **A dependency failure should fail fast rather than block the request
> path indefinitely.**

------------------------------------------------------------------------

# 16. Circuit Breaker

A circuit breaker prevents repeatedly sending requests to a
known-failing Redis dependency.

## State machine

``` mermaid
stateDiagram-v2
    [*] --> CLOSED

    CLOSED --> OPEN: Failure threshold exceeded
    OPEN --> HALF_OPEN: Timeout elapsed
    HALF_OPEN --> CLOSED: Recovery successful
    HALF_OPEN --> OPEN: Recovery failed
```

### CLOSED

Normal operation.

``` text
Rate Limiter → Redis
```

Requests are sent to Redis normally.

### OPEN

Redis is considered unhealthy.

``` text
Rate Limiter → Circuit Breaker → Fast Failure
```

The system avoids repeatedly waiting for Redis.

### HALF-OPEN

After a timeout, allow limited test requests.

If Redis responds successfully, return to CLOSED.

If it fails again, return to OPEN.

------------------------------------------------------------------------

# 17. Circuit Breaker + Failure Policy

``` mermaid
flowchart TD
    C[Client] --> G[API Gateway]
    G --> RL[Rate Limiter]
    RL --> CB[Circuit Breaker]
    CB --> R[(Redis)]

    R -->|Success| RL
    R -->|Failures| CB

    CB -->|OPEN| P{Endpoint Policy}

    P -->|Low Risk| FO[Fail Open]
    P -->|High Risk| FC[Fail Closed]

    FO --> B[Backend]
    FC --> X[Reject]

    style R fill:#3b0b0b,stroke:#ff3b30,color:#fff
    style FO fill:#063b2b,stroke:#00e676,color:#fff
    style FC fill:#3b0b0b,stroke:#ff3b30,color:#fff
```

This separates two concerns:

1.  **Circuit breaker** decides whether the Redis dependency should be
    called.
2.  **Failure policy** decides what the API should do when Redis cannot
    provide the rate-limit state.

------------------------------------------------------------------------

# 18. Production Architecture

A more resilient architecture can look like this:

``` mermaid
flowchart LR
    C[Client] --> LB[Load Balancer]
    LB --> G[API Gateway]

    G --> RL[Rate Limiter]

    RL --> T[Timeout]
    T --> CB[Circuit Breaker]

    CB --> R[(Redis HA / Cluster)]

    CB -. failure .-> FP{Failure Policy}

    FP -->|Low Risk| FO[Fail Open]
    FP -->|High Risk| FC[Fail Closed]

    FO --> B[Backend Services]
    FC --> X[Reject Request]

    R --> B

    B --> DB[(Database)]

    RL -.-> M[Monitoring]

    style LB fill:#0b2948,stroke:#2196f3,color:#fff
    style G fill:#063b2b,stroke:#00e676,color:#fff
    style RL fill:#111827,stroke:#94a3b8,color:#fff
    style R fill:#17113b,stroke:#8b5cf6,color:#fff
    style FO fill:#063b2b,stroke:#00e676,color:#fff
    style FC fill:#3b0b0b,stroke:#ff3b30,color:#fff
    style X fill:#3b0b0b,stroke:#ff3b30,color:#fff
```

------------------------------------------------------------------------

# 19. Monitoring

A production system should expose metrics around both Redis and rate
limiting.

``` mermaid
flowchart TD
    RL[Rate Limiter]

    RL --> M1[Redis Availability]
    RL --> M2[Redis Latency]
    RL --> M3[Rate Limit Check Failures]
    RL --> M4[Rejected Requests]
    RL --> M5[Allowed Requests]
    RL --> M6[Circuit Breaker State]
    RL --> M7[Backend QPS]
    RL --> M8[Database Load]

    style RL fill:#063b2b,stroke:#00e676,color:#fff
```

Useful signals include:

-   Redis availability
-   Redis connection errors
-   Redis latency
-   Rate-limit decision latency
-   Rate-limit check failures
-   Number of rejected requests
-   Number of allowed requests
-   Circuit-breaker state
-   Backend request rate
-   Database CPU/load
-   Error rate

A Redis outage should trigger an alert before it becomes a backend
overload incident.

------------------------------------------------------------------------

# 20. Tradeoff Summary

``` mermaid
quadrantChart
    title Failure Policy Tradeoff
    x-axis Lower Availability --> Higher Availability
    y-axis Lower Safety --> Higher Safety
    quadrant-1 Safety + Availability
    quadrant-2 Safety First
    quadrant-3 Risky
    quadrant-4 Availability First
    Fail Open: [0.82, 0.35]
    Fail Closed: [0.25, 0.85]
    Local Fallback: [0.65, 0.55]
```

### Fail Open

**Optimize for:**

-   Availability
-   User experience
-   Continuity of low-risk services

**Tradeoff:**

-   Less protection
-   Potential traffic flood

### Fail Closed

**Optimize for:**

-   Safety
-   Abuse prevention
-   Backend protection

**Tradeoff:**

-   Reduced availability
-   Legitimate requests may fail

### Local Fallback

**Optimize for:**

-   Partial protection
-   Availability during short outages

**Tradeoff:**

-   Global limit becomes approximate
-   Counters are split across servers

------------------------------------------------------------------------

# 21. Decision Framework

When designing the system, ask these questions in order:

``` mermaid
flowchart TD
    A[Redis unavailable] --> B{How critical is the endpoint?}

    B -->|Low risk| C[Prefer Fail Open]
    B -->|High risk| D[Prefer Fail Closed]

    C --> E{Can backend absorb extra traffic?}
    E -->|Yes| F[Allow traffic]
    E -->|No| G[Use conservative fallback / protection]

    D --> H{Can users tolerate temporary failure?}
    H -->|Yes| I[Reject quickly]
    H -->|No| J[Consider controlled fallback]

    G --> K[Monitor closely]
    J --> K
```

------------------------------------------------------------------------

# 22. Practical Endpoint Policy

A simple production policy could be:

``` text
Endpoint             Redis Failure Policy
------------------------------------------------
/login               FAIL CLOSED
/payment             FAIL CLOSED
/transfer             FAIL CLOSED
/password-reset       FAIL CLOSED
/search               FAIL OPEN
/public-content       FAIL OPEN
/product-catalog      FAIL OPEN
/recommendations      FAIL OPEN
```

The exact policy should be decided from the business risk rather than
blindly applying one strategy to every API.

------------------------------------------------------------------------

# 23. Common Design Mistakes

## Mistake 1 --- "Redis goes down, so the rate limiter is down."

This is incomplete.

A production design needs an explicit dependency-failure strategy.

------------------------------------------------------------------------

## Mistake 2 --- Always Fail Open

This protects availability but can expose sensitive endpoints to
uncontrolled traffic.

------------------------------------------------------------------------

## Mistake 3 --- Always Fail Closed

This protects the backend but can turn a Redis outage into a complete
application outage.

------------------------------------------------------------------------

## Mistake 4 --- Unlimited Redis timeout

A slow Redis can cause request threads/connections to pile up.

Use bounded timeouts.

------------------------------------------------------------------------

## Mistake 5 --- Local counters without understanding the tradeoff

Local counters are independent on each server.

They do not automatically provide a globally accurate distributed limit.

------------------------------------------------------------------------

## Mistake 6 --- No monitoring

A rate limiter can technically keep working while the backend or
database is being overloaded.

Monitor the complete request path.

------------------------------------------------------------------------

# 24. Interview Answer

A strong system-design answer can be:

> "If Redis goes down, I wouldn't simply let the rate limiter stop
> working. First, I'd define a failure policy per endpoint. Low-risk
> APIs such as search or public content can fail open to preserve
> availability, while sensitive APIs such as login and payment should
> generally fail closed to protect the system. I'd also use Redis HA or
> Redis Cluster to reduce the chance of a complete Redis failure, add
> short timeouts and a circuit breaker, and optionally use a
> conservative local in-memory fallback where approximate enforcement is
> acceptable. Finally, I'd monitor Redis health, rate-limit failures,
> circuit state, backend QPS, and database load."

------------------------------------------------------------------------

# 25. Final Architecture Mental Model

Think about Redis failure handling as **layers of protection**:

``` mermaid
flowchart TD
    A[Redis Failure]

    A --> B[1. Redis HA / Cluster]
    B --> C[2. Short Timeout]
    C --> D[3. Circuit Breaker]
    D --> E[4. Endpoint Failure Policy]

    E --> F[Fail Open]
    E --> G[Fail Closed]
    E --> H[Local Fallback]

    F --> I[Availability]
    G --> J[Safety]
    H --> K[Partial Protection]

    I --> L[Monitoring]
    J --> L
    K --> L

    style A fill:#3b0b0b,stroke:#ff3b30,color:#fff
    style F fill:#063b2b,stroke:#00e676,color:#fff
    style G fill:#3b0b0b,stroke:#ff3b30,color:#fff
    style H fill:#17113b,stroke:#8b5cf6,color:#fff
    style L fill:#0b2948,stroke:#2196f3,color:#fff
```

The key idea:

> **A resilient rate limiter does not assume Redis will always be
> available. It explicitly defines what happens when the shared state
> disappears.**

------------------------------------------------------------------------

# 26. Quick Cheat Sheet

  Concept               Purpose
  --------------------- --------------------------------------------
  Redis HA              Reduce Redis single-point-of-failure risk
  Redis Cluster         Scale and distribute Redis
  Timeout               Prevent requests from waiting indefinitely
  Circuit Breaker       Stop repeatedly calling a failing Redis
  Fail Open             Prioritize availability
  Fail Closed           Prioritize safety
  Local Fallback        Provide temporary local protection
  Endpoint Policy       Choose behavior based on API risk
  Monitoring            Detect and respond to failures
  Conservative Limits   Reduce damage during fallback

## One-line takeaway

**Design the failure path as carefully as the normal path.**
