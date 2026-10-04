# System Design --- Core Concepts You Must Understand
PDF: https://drive.google.com/file/d/1x-i5K_EUo9-yyGEMCCGom8kTyIELMA3t/view?usp=sharing

A practical, step-by-step guide to the seven system design concepts that
form the foundation of scalable distributed systems.

------------------------------------------------------------------------

## 1. Scalability

Scalability is the ability of a system to handle increasing traffic,
data, and workload while maintaining acceptable performance,
reliability, and cost.

A useful way to think about scalability is:

> More users should not automatically mean proportionally worse
> performance.

### 1.1 Vertical Scaling

Vertical scaling means increasing the capacity of an existing machine.

Examples:

-   More CPU
-   More RAM
-   Faster SSD/NVMe storage
-   Higher network bandwidth
-   Larger database instance

``` text
Before

        ┌──────────────┐
        │   Server     │
        │  4 CPU       │
        │  16 GB RAM   │
        └──────────────┘

After

        ┌──────────────┐
        │   Server     │
        │  16 CPU      │
        │  64 GB RAM   │
        └──────────────┘
```

### Advantages

-   Simple operational model
-   Usually requires fewer application changes
-   Useful for databases and stateful workloads
-   Easier to reason about initially

### Limitations

-   Hardware has an upper limit
-   Larger machines become expensive
-   Scaling may require downtime depending on infrastructure
-   A single machine can remain a failure point

### 1.2 Horizontal Scaling

Horizontal scaling means adding more instances instead of making one
machine larger.

``` mermaid
flowchart LR
    U[Users] --> LB[Load Balancer]
    LB --> A[App Server 1]
    LB --> B[App Server 2]
    LB --> C[App Server 3]
```

Each instance handles a portion of the workload.

### Advantages

-   Higher capacity
-   Better fault tolerance
-   Can scale incrementally
-   Works well with cloud infrastructure
-   Enables rolling deployments

### Challenges

Once an application becomes distributed, you need to think about:

-   Shared state
-   Session management
-   Load balancing
-   Distributed caching
-   Database connections
-   Service discovery
-   Consistency
-   Failure handling

### 1.3 Stateless Application Servers

Horizontal scaling becomes much easier when application servers are
stateless.

Instead of storing user session state inside one server:

``` text
User → Server A → Local Session
```

store shared state externally:

``` mermaid
flowchart LR
    U[User] --> LB[Load Balancer]
    LB --> A[Server A]
    LB --> B[Server B]
    LB --> C[Server C]
    A --> R[(Redis)]
    B --> R
    C --> R
```

Now any server can handle the next request.

### 1.4 Capacity Planning

Capacity planning estimates how much infrastructure is required for
expected workload.

Consider:

-   Requests per second (RPS)
-   Peak RPS
-   Average response time
-   CPU utilization
-   Memory utilization
-   Database throughput
-   Storage growth
-   Network bandwidth
-   Number of concurrent users

A basic planning model:

``` text
Required capacity ≈ Peak workload / Capacity per instance
```

For example, if one server safely handles 500 RPS and peak traffic is
5,000 RPS:

``` text
5000 / 500 = 10 servers
```

In production, you would add headroom rather than operating exactly at
maximum capacity.

### 1.5 Autoscaling

Autoscaling automatically adds or removes infrastructure based on
demand.

``` mermaid
flowchart LR
    M[Metrics] --> AS[Autoscaler]
    AS -->|Scale Out| N[More Instances]
    AS -->|Scale In| F[Fewer Instances]
    N --> APP[Application Fleet]
    F --> APP
```

Common signals:

-   CPU utilization
-   Memory utilization
-   Request rate
-   Queue depth
-   Response latency
-   Custom business metrics

### 1.6 Scaling Strategy

A practical progression is:

``` text
Single server
    ↓
Vertical scaling
    ↓
Stateless application
    ↓
Horizontal scaling
    ↓
Load balancing
    ↓
Caching
    ↓
Database scaling
    ↓
Autoscaling
```

------------------------------------------------------------------------

# 2. Load Balancing

A load balancer distributes incoming traffic across multiple backend
instances.

Without load balancing:

``` text
Users ───────────────> One Server
```

With load balancing:

``` mermaid
flowchart LR
    U[Users] --> LB[Load Balancer]
    LB --> A[Server A]
    LB --> B[Server B]
    LB --> C[Server C]
```

The goal is to prevent one server from becoming a bottleneck or single
point of failure.

## 2.1 Layer 4 Load Balancing

Layer 4 operates primarily at the transport layer.

Common protocols:

-   TCP
-   UDP

It makes routing decisions using information such as:

-   Source IP
-   Destination IP
-   Port
-   Connection information

``` text
Client
   ↓
L4 Load Balancer
   ↓
TCP connection
   ↓
Backend
```

### Advantages

-   Fast
-   Low overhead
-   Protocol independent at the application level
-   Good for high-throughput traffic

### Limitations

It does not understand application-level details such as:

-   HTTP paths
-   HTTP headers
-   Cookies
-   Request content

## 2.2 Layer 7 Load Balancing

Layer 7 operates at the application layer and can understand HTTP/HTTPS.

It can route based on:

-   URL path
-   Hostname
-   Headers
-   Cookies
-   HTTP method
-   Other application-level information

Example:

``` text
/api/users/*     → User Service
/api/orders/*    → Order Service
/api/payments/*  → Payment Service
```

``` mermaid
flowchart TD
    U[Client] --> L7[L7 Load Balancer]
    L7 -->|/users| US[User Service]
    L7 -->|/orders| OS[Order Service]
    L7 -->|/payments| PS[Payment Service]
```

## 2.3 Routing Strategies

### Round Robin

Requests are distributed sequentially:

``` text
Request 1 → A
Request 2 → B
Request 3 → C
Request 4 → A
```

### Weighted Round Robin

More powerful servers receive more traffic.

``` text
Server A → Weight 5
Server B → Weight 3
Server C → Weight 2
```

### Least Connections

Send traffic to the server with the fewest active connections.

### IP Hash

A client's IP is mapped consistently to a backend.

Useful when some degree of session affinity is required.

### Consistent Hashing

Often used in distributed systems to reduce remapping when nodes are
added or removed.

## 2.4 Health Checks

A load balancer should avoid sending traffic to unhealthy instances.

``` mermaid
flowchart LR
    LB[Load Balancer] --> A[Healthy A]
    LB --> B[Healthy B]
    LB -.-> C[Unhealthy C]
```

Health checks can test:

``` text
GET /health
GET /ready
```

A good health system distinguishes between:

-   Liveness: is the process alive?
-   Readiness: can it safely receive traffic?

## 2.5 Traffic Distribution

For production systems, load balancing often works together with:

-   Autoscaling
-   Health checks
-   Service discovery
-   TLS termination
-   Rate limiting
-   Connection draining
-   Failover

------------------------------------------------------------------------

# 3. Caching

Caching stores frequently accessed data closer to the application or
user so the system does not repeatedly perform expensive work.

Instead of:

``` text
User → Application → Database
```

we can use:

``` mermaid
flowchart LR
    U[User] --> A[Application]
    A --> C[(Cache)]
    C -->|Miss| DB[(Database)]
    DB --> C
```

## 3.1 Why Caching Helps

Caching can reduce:

-   Database load
-   Network latency
-   CPU usage
-   Expensive computation
-   External API calls

It can improve:

-   Response latency
-   Throughput
-   Availability during some dependency failures

## 3.2 Cache-Aside

The application checks the cache first.

``` mermaid
sequenceDiagram
    participant App
    participant Cache
    participant DB

    App->>Cache: Get data
    alt Cache hit
        Cache-->>App: Return data
    else Cache miss
        Cache-->>App: Miss
        App->>DB: Query data
        DB-->>App: Data
        App->>Cache: Store data
        App-->>App: Return data
    end
```

This is one of the most common patterns.

## 3.3 Write-Through

The application writes to the cache and the cache synchronously writes
to the database.

``` text
Application
    ↓
Cache
    ↓
Database
```

This can keep cache contents relatively fresh but adds write-path
complexity.

## 3.4 Write-Behind

The cache accepts the write first and persists it to the database
asynchronously.

``` text
Application
    ↓
Cache
    ↓
Async persistence
    ↓
Database
```

It can provide fast writes but introduces durability and consistency
considerations.

## 3.5 TTL

TTL means Time To Live.

Example:

``` text
User profile cache
TTL = 10 minutes
```

After expiration, the value must be refreshed.

TTL prevents stale data from living forever.

## 3.6 Cache Invalidation

A common challenge is:

> How do you know when cached data is no longer valid?

Common approaches:

-   TTL expiration
-   Explicit deletion
-   Versioned keys
-   Event-driven invalidation
-   Write-through updates

A famous engineering problem is cache invalidation because stale data
can create incorrect application behavior.

## 3.7 CDN Caching

A CDN caches content near users geographically.

``` mermaid
flowchart LR
    U1[User US] --> CDN[CDN]
    U2[User EU] --> CDN
    U3[User Asia] --> CDN
    CDN --> ORIGIN[Origin Server]
```

Useful for:

-   Images
-   JavaScript
-   CSS
-   Videos
-   Static HTML
-   Cacheable API responses

## 3.8 In-Memory Caching

Examples include:

-   Redis
-   Memcached
-   Local process memory

Distributed cache:

``` mermaid
flowchart LR
    A1[App 1] --> R[(Redis)]
    A2[App 2] --> R
    A3[App 3] --> R
```

Local memory is extremely fast but is not automatically shared across
application instances.

------------------------------------------------------------------------

# 4. Database Scaling

Databases frequently become one of the most important bottlenecks as
systems grow.

The practical scaling sequence should usually start with optimization
before introducing distributed complexity.

``` text
Query Optimization
       ↓
Indexing
       ↓
Partitioning
       ↓
Primary Database Optimization
       ↓
Read Replica #1
       ↓
Read Replicas #2, #3...
       ↓
Sharding
       ↓
Distributed / Multi-region Architecture
```

## 4.1 Step 1 --- Query Optimization

Before adding hardware, determine whether the database is doing
unnecessary work.

Look for:

-   Slow queries
-   Full table scans
-   N+1 queries
-   Expensive joins
-   Unnecessary columns
-   Missing filters
-   Repeated queries

Use tools such as:

``` sql
EXPLAIN
EXPLAIN ANALYZE
```

Example:

``` sql
SELECT *
FROM orders
WHERE user_id = 1001;
```

Prefer selecting only what is required:

``` sql
SELECT id, status, total
FROM orders
WHERE user_id = 1001;
```

## 4.2 Step 2 --- Indexing

Indexes allow the database to locate rows more efficiently.

Example:

``` sql
CREATE INDEX idx_orders_user_id
ON orders(user_id);
```

For multi-column access patterns:

``` sql
CREATE INDEX idx_orders_user_status
ON orders(user_id, status);
```

Important considerations:

-   Index selectivity
-   Query patterns
-   Column ordering
-   Write overhead
-   Storage overhead
-   Unused indexes

Do not blindly index every column.

## 4.3 Step 3 --- Partitioning

Partitioning divides one logical table into smaller physical partitions.

Common approaches:

-   Range
-   Hash
-   List

Example:

``` mermaid
flowchart TD
    T[Orders Table] --> P1[2024 Partition]
    T --> P2[2025 Partition]
    T --> P3[2026 Partition]
```

With partition pruning, a query for 2026 may only need the 2026
partition.

## 4.4 Step 4 --- Primary Database

Start with a well-configured primary database.

It handles:

-   Writes
-   Transactions
-   Reads, when appropriate

Optimize:

-   CPU
-   Memory
-   Storage
-   Connection pools
-   Database configuration
-   Buffer/cache settings

Monitor:

-   CPU
-   Memory
-   Disk I/O
-   Connections
-   Query latency
-   Lock contention
-   Transaction throughput

## 4.5 Step 5 --- Read Replica #1

When reads become the bottleneck, introduce a read replica.

``` mermaid
flowchart LR
    APP[Application] -->|Writes| P[(Primary)]
    P -->|Replication| R1[(Read Replica)]
    APP -->|Reads| R1
```

The primary continues handling writes while the replica handles read
traffic.

Important issue:

### Replication Lag

The replica may temporarily be behind the primary.

Therefore, applications must decide whether a particular read requires:

-   Strong consistency
-   Eventual consistency

## 4.6 Step 6 --- Multiple Read Replicas

As read traffic grows:

``` mermaid
flowchart TD
    APP[Application] --> P[(Primary)]
    P --> R1[(Replica 1)]
    P --> R2[(Replica 2)]
    P --> R3[(Replica 3)]

    APP --> LB[Read Load Balancer]
    LB --> R1
    LB --> R2
    LB --> R3
```

Use a load balancer or replica-aware routing layer to distribute reads.

Monitor:

-   Replica lag
-   Read QPS
-   CPU
-   Memory
-   Disk I/O
-   Replica health

## 4.7 Step 7 --- Sharding

When a single database cluster cannot handle the dataset or workload,
split data across multiple database nodes.

``` mermaid
flowchart TD
    APP[Application] --> ROUTER[Shard Router]
    ROUTER --> S1[(Shard 1)]
    ROUTER --> S2[(Shard 2)]
    ROUTER --> S3[(Shard 3)]
```

Possible shard keys:

-   user_id
-   tenant_id
-   region_id
-   account_id

A good shard key should generally provide:

-   Even distribution
-   High enough cardinality
-   Predictable access patterns
-   Low likelihood of hot spots

### Hot Shard

A hot shard receives disproportionately high traffic.

For example:

``` text
Shard 1 → 20% traffic
Shard 2 → 20% traffic
Shard 3 → 60% traffic  ← Hot shard
```

## 4.8 Cross-Shard Queries

Queries involving multiple shards can become expensive.

``` text
Application
   ↓
Shard 1 ──┐
Shard 2 ──┼──> Merge Results
Shard 3 ──┘
```

Design APIs and data ownership so common queries can often be served
from one shard.

## 4.9 Distributed Database Strategy

At very large scale, combine:

``` mermaid
flowchart TD
    ROUTER[Global Traffic Router]
    ROUTER --> R1[Region A]
    ROUTER --> R2[Region B]
    ROUTER --> R3[Region C]

    R1 --> S1[(Shard + Replicas)]
    R2 --> S2[(Shard + Replicas)]
    R3 --> S3[(Shard + Replicas)]
```

Additional concerns include:

-   Failover
-   Rebalancing
-   Multi-region replication
-   Consistency
-   Disaster recovery
-   Backup/restore
-   Data residency
-   Operational complexity

------------------------------------------------------------------------

# 5. CAP Theorem

CAP is a framework for reasoning about distributed data systems.

The three properties are:

-   Consistency
-   Availability
-   Partition Tolerance

A distributed system experiencing a network partition cannot
simultaneously guarantee both perfect consistency and availability for
every operation.

## 5.1 Consistency

Every successful read receives the latest committed value according to
the system's consistency model.

Example:

``` text
Write balance = 500

Read from any relevant node
→ 500
```

## 5.2 Availability

Every request to a non-failing node receives a response.

Availability does not necessarily mean the response contains the newest
value.

## 5.3 Partition Tolerance

The system continues operating despite communication failures between
nodes.

``` mermaid
flowchart LR
    A[Node A] x--x B[Node B]
    A --- C[Network Partition]
    C --- B
```

In distributed systems, network partitions are considered unavoidable
possibilities, so partition tolerance becomes a fundamental requirement.

## 5.4 CAP Trade-off

During a partition, you generally choose between:

``` text
Consistency
      OR
Availability
```

This is a simplification of a deeper set of consistency and availability
guarantees, but it is useful for architectural reasoning.

## 5.5 CP Systems

A CP-oriented system favors consistency during a partition.

It may reject or delay operations rather than return potentially stale
data.

Useful when correctness is more important than continuous availability.

Examples of use cases:

-   Financial transactions
-   Metadata
-   Leader election
-   Coordination

## 5.6 AP Systems

An AP-oriented system favors availability during a partition.

It may accept operations and reconcile divergent data later.

Useful for:

-   Social feeds
-   Distributed content
-   Some shopping experiences
-   Large-scale eventually consistent systems

## 5.7 CAP Is Not a Database Selection Checklist

Do not simply label databases "CA," "CP," or "AP" without considering
their actual configuration and consistency model.

Real systems can expose different guarantees depending on:

-   Replication strategy
-   Quorum settings
-   Read/write configuration
-   Failure conditions
-   Data model

------------------------------------------------------------------------

# 6. Messaging & Event-Driven Architecture

Synchronous communication makes one service wait for another.

``` mermaid
sequenceDiagram
    participant A as Service A
    participant B as Service B

    A->>B: Request
    B-->>A: Response
    A->>A: Continue
```

Asynchronous messaging decouples producers from consumers.

``` mermaid
flowchart LR
    P[Producer] --> Q[(Message Queue)]
    Q --> C1[Consumer 1]
    Q --> C2[Consumer 2]
```

## 6.1 Why Messaging?

Messaging can provide:

-   Loose coupling
-   Asynchronous processing
-   Traffic smoothing
-   Retry capabilities
-   Better failure isolation
-   Independent scaling

## 6.2 Queue

A queue typically distributes work to consumers.

Example:

``` text
Order Created
     ↓
Queue
     ↓
Worker
     ↓
Send Email
```

If traffic spikes, messages can wait in the queue instead of
overwhelming the worker.

## 6.3 Pub/Sub

Pub/Sub allows multiple subscribers to consume an event.

``` mermaid
flowchart LR
    P[Order Service] --> T[Event Topic]
    T --> E[Email Service]
    T --> A[Analytics Service]
    T --> N[Notification Service]
```

One producer does not need direct knowledge of every consumer.

## 6.4 Event Streaming

Event streaming platforms maintain an ordered stream of events that
consumers can process.

Concepts commonly include:

-   Topics
-   Partitions
-   Offsets
-   Consumer groups
-   Retention

``` mermaid
flowchart LR
    P[Producer] --> T[Topic]
    T --> P1[Partition 0]
    T --> P2[Partition 1]
    T --> P3[Partition 2]
```

Partitions allow parallel consumption.

## 6.5 Consumer Groups

A consumer group allows multiple workers to share partitions.

``` mermaid
flowchart LR
    T[Topic] --> P0[Partition 0]
    T --> P1[Partition 1]
    T --> P2[Partition 2]

    P0 --> C1[Consumer 1]
    P1 --> C2[Consumer 2]
    P2 --> C3[Consumer 3]
```

This enables horizontal scaling of event processing.

## 6.6 Asynchronous Processing

Use asynchronous processing when the caller does not need an immediate
result.

Example:

``` text
User uploads file
       ↓
API stores metadata
       ↓
Queue
       ↓
Background worker
       ↓
Image processing
       ↓
Notification
```

The user does not need to wait for the entire workflow.

## 6.7 Reliability Considerations

Distributed messaging requires thinking about:

-   Retries
-   Dead-letter queues
-   Duplicate messages
-   Idempotency
-   Ordering
-   Delivery semantics
-   Backpressure
-   Consumer failures
-   Poison messages

### Idempotency

If a message is delivered twice, processing it twice should not corrupt
the system.

Example:

``` text
Payment ID = 123

First processing → payment captured
Duplicate delivery → detect ID 123 already processed
```

## 6.8 Event-Driven Architecture

A larger event-driven system might look like:

``` mermaid
flowchart TD
    U[User Action] --> API[API Service]
    API --> DB[(Database)]
    API --> BUS[Event Bus]

    BUS --> W1[Email Worker]
    BUS --> W2[Analytics Worker]
    BUS --> W3[Notification Worker]
    BUS --> W4[Search Index Worker]
```

The event bus becomes a communication boundary between independent
capabilities.

------------------------------------------------------------------------

# 7. Architectural Styles

Architecture style describes how an application is structured and how
its components communicate.

There is no universally best architecture.

The right choice depends on:

-   Team size
-   Product complexity
-   Scale
-   Deployment requirements
-   Reliability requirements
-   Organizational structure
-   Operational maturity

------------------------------------------------------------------------

## 7.1 Monolith

A monolith packages most application functionality into one deployable
unit.

``` mermaid
flowchart LR
    U[Users] --> APP[Monolithic Application]
    APP --> DB[(Database)]
```

### Advantages

-   Simple deployment
-   Simple local development
-   Easy function calls between modules
-   Lower operational overhead
-   Good for early-stage products

### Disadvantages

-   Large codebase over time
-   Tightly coupled modules
-   Entire application may need deployment
-   Scaling individual components is harder

A well-modularized monolith can be an excellent starting point.

------------------------------------------------------------------------

## 7.2 Layered Architecture

A layered architecture separates responsibilities.

Common layers:

``` text
Presentation
     ↓
Application / Service
     ↓
Domain
     ↓
Data Access
     ↓
Database
```

``` mermaid
flowchart TD
    UI[Presentation Layer]
    S[Application / Service Layer]
    D[Domain Layer]
    DA[Data Access Layer]
    DB[(Database)]

    UI --> S
    S --> D
    D --> DA
    DA --> DB
```

Benefits include:

-   Separation of concerns
-   Easier testing
-   Clear responsibilities
-   Easier onboarding

------------------------------------------------------------------------

## 7.3 Microservices

Microservices split an application into independently deployable
services.

``` mermaid
flowchart TD
    U[Users] --> G[API Gateway]

    G --> US[User Service]
    G --> OS[Order Service]
    G --> PS[Payment Service]
    G --> NS[Notification Service]

    US --> UDB[(User DB)]
    OS --> ODB[(Order DB)]
    PS --> PDB[(Payment DB)]
```

### Advantages

-   Independent deployment
-   Independent scaling
-   Service-level ownership
-   Fault isolation
-   Technology flexibility

### Costs

-   Network communication
-   Distributed debugging
-   Service discovery
-   Observability requirements
-   Data consistency challenges
-   Deployment complexity
-   More infrastructure

Microservices should generally solve a real organizational or scaling
problem rather than being adopted only because they are popular.

------------------------------------------------------------------------

## 7.4 Serverless

Serverless platforms execute functions or workloads without requiring
the team to manage servers directly.

``` mermaid
flowchart LR
    U[User] --> API[API Gateway]
    API --> F1[Function]
    API --> F2[Function]
    F1 --> DB[(Database)]
    F2 --> Q[Queue]
```

### Advantages

-   Automatic scaling
-   Pay-per-use models
-   Reduced infrastructure management
-   Good for event-driven workloads

### Challenges

-   Cold starts
-   Runtime limits
-   Vendor-specific services
-   Distributed debugging
-   Cost can become difficult to predict at sustained high volume

------------------------------------------------------------------------

## 7.5 Event-Driven Architecture

Components communicate through events instead of tightly coupled
synchronous calls.

``` mermaid
flowchart LR
    O[Order Service] --> E[OrderCreated Event]
    E --> P[Payment Service]
    E --> N[Notification Service]
    E --> A[Analytics Service]
```

Advantages:

-   Loose coupling
-   Async processing
-   Independent scaling
-   Good failure isolation
-   Natural integration with background processing

Trade-offs:

-   Eventual consistency
-   Harder debugging
-   Duplicate events
-   Ordering concerns
-   Schema evolution
-   Operational complexity

------------------------------------------------------------------------

# Putting Everything Together

A realistic scalable system may combine all seven concepts rather than
choosing only one.

``` mermaid
flowchart TD
    U[Users] --> CDN[CDN]
    CDN --> LB[Load Balancer]

    LB --> A1[App Server 1]
    LB --> A2[App Server 2]
    LB --> A3[App Server 3]

    A1 --> C[(Redis Cache)]
    A2 --> C
    A3 --> C

    A1 --> P[(Primary Database)]
    A2 --> P
    A3 --> P

    P --> R1[(Read Replica 1)]
    P --> R2[(Read Replica 2)]
    P --> R3[(Read Replica 3)]

    A1 --> BUS[Event Bus]
    A2 --> BUS
    A3 --> BUS

    BUS --> W1[Email Worker]
    BUS --> W2[Analytics Worker]
    BUS --> W3[Notification Worker]

    P --> S[Shard Router]
    S --> S1[(Shard 1)]
    S --> S2[(Shard 2)]
    S --> S3[(Shard 3)]
```

The components solve different problems:

  Problem                             Typical Solution
  ----------------------------------- --------------------------------------
  More application traffic            Horizontal scaling
  Uneven traffic                      Load balancing
  Repeated expensive reads            Caching
  Slow queries                        Query optimization
  Large table scans                   Indexing / partitioning
  Read-heavy database                 Read replicas
  Dataset too large for one cluster   Sharding
  Service decoupling                  Messaging / events
  Background work                     Queues / workers
  Regional latency                    CDN / multi-region architecture
  Component failure                   Replication / failover
  Large system complexity             Appropriate architectural boundaries

# Recommended Learning Order

Do not try to learn everything at once.

Follow this progression:

``` mermaid
flowchart TD
    A[1. Scalability] --> B[2. Load Balancing]
    B --> C[3. Caching]
    C --> D[4. Database Scaling]
    D --> E[5. CAP Theorem]
    E --> F[6. Messaging & Event-Driven]
    F --> G[7. Architectural Styles]
    G --> H[Design Real Systems]
```

## Phase 1 --- Scaling Fundamentals

Learn:

-   Vertical vs horizontal scaling
-   Stateless services
-   Capacity planning
-   Autoscaling
-   Load balancing

Practice by designing:

-   Simple web application
-   URL shortener
-   Image upload service

## Phase 2 --- Performance

Learn:

-   Caching
-   CDN
-   Cache invalidation
-   TTL
-   Database indexing
-   Query optimization

Practice:

-   Design a high-traffic product catalog
-   Design a news feed
-   Design a content delivery platform

## Phase 3 --- Distributed Data

Learn:

-   Replication
-   Partitioning
-   Sharding
-   CAP
-   Consistency
-   Failover

Practice:

-   Design a large multi-tenant SaaS database
-   Design a globally distributed user service

## Phase 4 --- Asynchronous Systems

Learn:

-   Queues
-   Pub/Sub
-   Event streaming
-   Consumer groups
-   Retries
-   Idempotency
-   Dead-letter queues

Practice:

-   Order processing system
-   Notification platform
-   Payment event pipeline

## Phase 5 --- Architecture

Compare:

``` text
Monolith
   ↓
Layered Monolith
   ↓
Modular Monolith
   ↓
Microservices
   ↓
Event-Driven Systems
   ↓
Serverless / Hybrid
```

Do not treat this as a mandatory migration path. Choose architecture
based on actual requirements.

# System Design Interview Checklist

When designing a system, walk through these questions:

### 1. Requirements

-   What does the system do?
-   Who uses it?
-   What are the most important features?
-   What are the availability requirements?

### 2. Scale

-   How many users?
-   Requests per second?
-   Peak traffic?
-   Data size?
-   Data growth rate?

### 3. API

-   What endpoints are required?
-   What are the request/response shapes?
-   Is pagination required?
-   Is idempotency required?

### 4. Application

-   Stateless or stateful?
-   How will it scale horizontally?
-   Where should business logic live?

### 5. Database

-   SQL or NoSQL?
-   What are the access patterns?
-   What indexes are needed?
-   Will replication be required?
-   Will partitioning or sharding eventually be needed?

### 6. Cache

-   What data is frequently read?
-   What is the TTL?
-   What happens on cache miss?
-   How is stale data invalidated?

### 7. Traffic

-   Is a load balancer required?
-   L4 or L7?
-   What routing strategy?
-   What health checks?

### 8. Async Processing

-   Which operations can be asynchronous?
-   Do we need queues?
-   What happens if a consumer fails?
-   How do we handle duplicates?

### 9. Reliability

-   What happens when a server fails?
-   What happens when the database fails?
-   How does failover work?
-   What happens during a network partition?

### 10. Observability

Monitor:

-   Latency
-   Error rate
-   Throughput
-   CPU
-   Memory
-   Database load
-   Cache hit rate
-   Queue depth
-   Replication lag

A useful mental model is:

``` text
Scale
 ↓
Distribute
 ↓
Cache
 ↓
Optimize Data
 ↓
Replicate
 ↓
Partition / Shard
 ↓
Decouple with Events
 ↓
Design for Failure
 ↓
Observe Everything
```

# Final Takeaway

Good system design is not about adding the maximum number of
technologies.

It is about identifying the bottleneck, understanding the trade-off, and
introducing the simplest mechanism that solves the actual problem.

Start simple.

Measure.

Optimize.

Scale the bottleneck.

Then introduce distributed complexity only when the system genuinely
needs it.
