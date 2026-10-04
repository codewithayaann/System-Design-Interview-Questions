# Database Scaling --- Step-by-Step Deep Dive

A practical guide to scaling a database from a simple single-node setup
to a distributed, highly available architecture.

The key principle:

> **Do not jump to sharding first. Fix the bottleneck at the simplest
> layer that can solve it.**

A common progression is:

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
Sharding + Replication
        ↓
Multi-Region / Distributed Database
```

------------------------------------------------------------------------

# 1. Understand the Database Bottleneck

Before scaling, determine what is actually limiting the system.

Common bottlenecks:

-   Slow queries
-   Missing indexes
-   Too many database connections
-   High CPU
-   High memory usage
-   Disk I/O
-   Lock contention
-   Large table scans
-   Too many reads
-   Too many writes
-   Dataset too large for one node
-   Replication lag

A useful first question is:

> **Is the database slow because it is doing too much work, or because
> it has reached its infrastructure limit?**

If a query can be reduced from 2 seconds to 50 ms, adding another
database server may be unnecessary.

------------------------------------------------------------------------

# 2. Step 1 --- Query Optimization

Query optimization should normally be the first step.

## 2.1 Find Slow Queries

Monitor:

-   Query duration
-   Query frequency
-   Rows scanned
-   Rows returned
-   CPU time
-   Disk reads
-   Lock wait time

A query executed 1,000 times per second at 100 ms can be more important
than a query executed once per minute at 2 seconds.

Think about:

``` text
Impact ≈ Query Cost × Query Frequency
```

## 2.2 Use EXPLAIN

For SQL databases, inspect the query execution plan.

``` sql
EXPLAIN
SELECT *
FROM orders
WHERE user_id = 1001;
```

For supported databases:

``` sql
EXPLAIN ANALYZE
SELECT *
FROM orders
WHERE user_id = 1001;
```

Look for:

-   Full table scans
-   Expensive joins
-   Large sort operations
-   Temporary tables
-   Poor index usage
-   Excessive rows examined

## 2.3 Avoid SELECT \*

Instead of:

``` sql
SELECT *
FROM users
WHERE id = 1001;
```

prefer:

``` sql
SELECT id, name, email
FROM users
WHERE id = 1001;
```

Benefits:

-   Less data read
-   Less network transfer
-   Less serialization
-   Less application memory

## 2.4 Avoid N+1 Queries

Bad pattern:

``` text
Get 100 users
    ↓
100 additional queries for orders
```

Potentially:

``` text
101 database queries
```

Better:

``` text
Get users
    ↓
Fetch required orders in a batch
```

Or use an appropriate join/query strategy.

## 2.5 Optimize Joins

Check:

-   Join columns are indexed when appropriate
-   Large intermediate result sets
-   Join order
-   Filtering before joining
-   Whether denormalization is justified

## 2.6 Query Optimization Checklist

``` text
Find slow query
      ↓
EXPLAIN / EXPLAIN ANALYZE
      ↓
Check rows scanned
      ↓
Check indexes
      ↓
Reduce returned data
      ↓
Optimize joins
      ↓
Measure again
```

------------------------------------------------------------------------

# 3. Step 2 --- Indexing

Indexes speed up data lookup by creating an additional access structure.

Without a useful index:

``` text
Query
  ↓
Scan many rows
  ↓
Find matching rows
```

With an index:

``` text
Query
  ↓
Index
  ↓
Matching rows
```

## 3.1 Basic Index

Example:

``` sql
CREATE INDEX idx_orders_user_id
ON orders(user_id);
```

Now:

``` sql
SELECT id, total, status
FROM orders
WHERE user_id = 1001;
```

can use the index to locate relevant records more efficiently.

## 3.2 Composite Index

Suppose queries commonly use:

``` sql
WHERE tenant_id = ?
AND status = ?
```

A composite index may help:

``` sql
CREATE INDEX idx_orders_tenant_status
ON orders(tenant_id, status);
```

Column order matters.

A composite index on:

``` text
(tenant_id, status)
```

is not equivalent to:

``` text
(status, tenant_id)
```

for every query pattern.

Design indexes around actual access patterns.

## 3.3 Covering Index

A covering index can contain enough information for the database to
answer a query without accessing the full table.

Conceptually:

``` text
Query
  ↓
Index contains required values
  ↓
Return result
```

This can reduce table access for suitable workloads.

## 3.4 Index Trade-offs

Indexes are not free.

They consume:

-   Storage
-   Memory
-   Write time
-   Maintenance resources

Every insert/update/delete may need to update relevant indexes.

Therefore:

> **More indexes do not automatically mean better performance.**

## 3.5 Index Checklist

``` text
Identify common queries
        ↓
Check execution plans
        ↓
Add targeted indexes
        ↓
Measure read performance
        ↓
Measure write overhead
        ↓
Remove unused/problematic indexes
```

------------------------------------------------------------------------

# 4. Step 3 --- Partitioning

Partitioning divides a large logical table into smaller physical
partitions.

The application can still treat it as one logical table while the
database manages the underlying partitions.

``` mermaid
flowchart TD
    T[Large Orders Table] --> P1[2024 Partition]
    T --> P2[2025 Partition]
    T --> P3[2026 Partition]
```

## 4.1 Range Partitioning

Data is divided into ranges.

Example:

``` text
orders_2024
orders_2025
orders_2026
```

Useful for:

-   Time-series data
-   Logs
-   Events
-   Historical records

## 4.2 Hash Partitioning

A hash function distributes records.

``` text
hash(user_id) % N
```

Conceptually:

``` mermaid
flowchart LR
    U[User ID] --> H[Hash Function]
    H --> P1[Partition 1]
    H --> P2[Partition 2]
    H --> P3[Partition 3]
    H --> P4[Partition 4]
```

This can provide more even distribution than simple ranges for some
workloads.

## 4.3 List Partitioning

Records are grouped by predefined values.

Example:

``` text
US → Partition US
EU → Partition EU
IN → Partition IN
```

## 4.4 Partition Pruning

A major benefit is allowing the database to avoid irrelevant partitions.

Example:

``` sql
SELECT *
FROM orders
WHERE created_at >= '2026-01-01'
AND created_at < '2026-02-01';
```

If the table is partitioned by month, the database may only need the
relevant partition.

## 4.5 Partitioning Is Not Sharding

Important distinction:

``` text
Partitioning
= Splitting a logical table into partitions

Sharding
= Splitting data across separate database nodes/clusters
```

Partitioning can exist inside one database server.

Sharding distributes data across multiple database systems.

------------------------------------------------------------------------

# 5. Step 4 --- Optimize the Primary Database

After query and schema optimization, make sure the primary database
itself is appropriately sized and configured.

The primary commonly handles writes and may also handle reads.

``` mermaid
flowchart LR
    APP[Application] --> P[(Primary Database)]
```

Monitor:

-   CPU
-   Memory
-   Disk I/O
-   Storage capacity
-   Connections
-   Transactions/sec
-   Query latency
-   Locks
-   Buffer/cache hit rate

## 5.1 Connection Pooling

Do not create a new database connection for every request if the
database/client stack supports pooling.

Conceptually:

``` mermaid
flowchart LR
    A[Application Instances] --> CP[Connection Pool]
    CP --> DB[(Database)]
```

Connection pooling can reduce connection overhead and prevent connection
storms.

## 5.2 Vertical Database Scaling

If the primary is resource constrained, increase:

-   CPU
-   RAM
-   Storage performance
-   Network capacity

This is often easier than immediately introducing distributed
complexity.

------------------------------------------------------------------------

# 6. Step 5 --- Add Read Replica #1

If the workload is read-heavy, replication can separate reads from
writes.

``` mermaid
flowchart LR
    APP[Application] -->|Writes| P[(Primary)]
    P -->|Replication| R1[(Read Replica 1)]
    APP -->|Reads| R1
```

The primary handles writes.

The replica handles eligible reads.

## 6.1 Why Read Replicas?

Suppose:

``` text
10,000 requests/sec

8,000 reads
2,000 writes
```

If one primary is overloaded by reads, moving some reads to replicas can
reduce primary pressure.

## 6.2 Replication Lag

Replication is often asynchronous.

Therefore:

``` text
Primary
  ↓
Write = 500
  ↓
Replica
  ↓
May temporarily still return 450
```

Applications must understand this possibility.

## 6.3 Read-After-Write Problem

Example:

``` text
User updates profile
       ↓
Write → Primary
       ↓
Immediately read profile
       ↓
Read → Replica
       ↓
Replica has not caught up
```

The user may temporarily see stale data.

Solutions depend on the database and architecture and may include:

-   Read from primary after critical writes
-   Session/consistency routing
-   Synchronous replication where appropriate
-   Waiting for replication position
-   Application-level consistency strategy

------------------------------------------------------------------------

# 7. Step 6 --- Add Read Replicas #2 and #3

As read traffic grows, add more replicas.

``` mermaid
flowchart TD
    APP[Application] --> P[(Primary)]

    P --> R1[(Replica 1)]
    P --> R2[(Replica 2)]
    P --> R3[(Replica 3)]

    APP --> RL[Read Router]
    RL --> R1
    RL --> R2
    RL --> R3
```

## 7.1 Read Distribution

A read router/load balancer can distribute traffic:

``` text
Read 1 → Replica 1
Read 2 → Replica 2
Read 3 → Replica 3
Read 4 → Replica 1
```

Possible strategies:

-   Round robin
-   Weighted routing
-   Least connections
-   Health-aware routing
-   Replica-lag-aware routing

## 7.2 Replica Health

Do not send traffic to a replica that is:

-   Down
-   Severely lagging
-   Overloaded
-   Running out of storage

Monitor:

``` text
Replica lag
CPU
Memory
Disk I/O
Connections
Read QPS
Errors
```

## 7.3 The Read Replica Limit

Read replicas primarily solve read scalability.

They do not automatically solve:

-   Write bottlenecks
-   Huge dataset size
-   Hot keys
-   Cross-region write conflicts
-   Single-primary write limits

When the write workload or dataset becomes too large, another strategy
may be required.

------------------------------------------------------------------------

# 8. Step 7 --- Introduce Sharding

Sharding distributes data across multiple database nodes.

``` mermaid
flowchart TD
    APP[Application] --> ROUTER[Shard Router]
    ROUTER --> S1[(Shard 1)]
    ROUTER --> S2[(Shard 2)]
    ROUTER --> S3[(Shard 3)]
```

Instead of:

``` text
One database
  └── 3 billion rows
```

you might have:

``` text
Shard 1 → subset of data
Shard 2 → subset of data
Shard 3 → subset of data
```

Each shard handles only part of the workload.

------------------------------------------------------------------------

# 9. Choosing a Shard Key

The shard key is one of the most important decisions in a sharded
architecture.

Common candidates:

-   user_id
-   tenant_id
-   account_id
-   region_id
-   organization_id

## 9.1 Good Shard Key Properties

A good shard key should ideally provide:

### High Cardinality

Many possible values.

``` text
user_id → usually high cardinality
gender → very low cardinality
```

### Even Distribution

Avoid sending most records to one shard.

### Query Locality

Common queries should be able to identify the correct shard.

### Stable Distribution

The distribution should remain reasonable as the system grows.

------------------------------------------------------------------------

# 10. Avoid Hot Shards

A hot shard receives disproportionately high traffic.

Example:

``` text
Shard 1 → 25%
Shard 2 → 25%
Shard 3 → 25%
Shard 4 → 25%
```

Good distribution.

Bad:

``` text
Shard 1 → 10%
Shard 2 → 10%
Shard 3 → 10%
Shard 4 → 70%  ← Hot shard
```

Hot shards can happen because of:

-   Poor shard key
-   Celebrity users
-   Large tenants
-   Geographic concentration
-   Sequential IDs
-   Traffic spikes

A good architecture must plan for this.

------------------------------------------------------------------------

# 11. Sharding Strategies

## 11.1 Range-Based Sharding

``` text
user_id 1–1M       → Shard A
user_id 1M–2M      → Shard B
user_id 2M–3M      → Shard C
```

Simple but can create hot ranges.

## 11.2 Hash-Based Sharding

``` text
hash(user_id) → shard
```

Generally provides better distribution.

However, changing the number of shards can require significant data
movement depending on the implementation.

## 11.3 Directory-Based Sharding

A routing layer maintains a mapping:

``` text
tenant_101 → shard_1
tenant_102 → shard_3
tenant_103 → shard_2
```

This provides flexible placement but introduces another metadata/routing
dependency.

------------------------------------------------------------------------

# 12. Cross-Shard Queries

One of the biggest challenges with sharding is querying data across
multiple shards.

``` mermaid
flowchart TD
    APP[Application] --> ROUTER[Shard Router]

    ROUTER --> S1[(Shard 1)]
    ROUTER --> S2[(Shard 2)]
    ROUTER --> S3[(Shard 3)]

    S1 --> M[Merge Results]
    S2 --> M
    S3 --> M
```

Example:

``` sql
SELECT COUNT(*)
FROM orders;
```

If orders are sharded, the system may need to:

``` text
Query Shard 1
Query Shard 2
Query Shard 3
       ↓
Combine results
```

This can be slower and operationally more complex.

## 12.1 Design Around Shard Locality

Prefer APIs that include the shard key.

Instead of:

``` http
GET /orders/123
```

you may design around:

``` http
GET /users/1001/orders
```

The `user_id` immediately identifies the likely shard.

------------------------------------------------------------------------

# 13. Sharding + Replication

At larger scale, each shard can have replicas.

``` mermaid
flowchart TD
    R[Shard Router]

    R --> S1[Shard 1 Primary]
    R --> S2[Shard 2 Primary]
    R --> S3[Shard 3 Primary]

    S1 --> S1R1[Shard 1 Replica]
    S1 --> S1R2[Shard 1 Replica]

    S2 --> S2R1[Shard 2 Replica]
    S2 --> S2R2[Shard 2 Replica]

    S3 --> S3R1[Shard 3 Replica]
    S3 --> S3R2[Shard 3 Replica]
```

Now you can independently scale:

-   Data distribution
-   Write capacity
-   Read capacity
-   Availability

But operational complexity increases substantially.

------------------------------------------------------------------------

# 14. Rebalancing

As data grows, shards can become uneven.

Example:

``` text
Shard A → 2 TB
Shard B → 2 TB
Shard C → 12 TB  ← Too large
```

Rebalancing moves data between shards.

``` mermaid
flowchart LR
    A[Shard A] -->|Move data| B[Shard B]
    C[Hot/Large Shard] -->|Redistribute| A
    C -->|Redistribute| B
```

Important considerations:

-   Data movement cost
-   Production traffic impact
-   Dual writes / migration strategy
-   Consistency
-   Routing changes
-   Rollback
-   Monitoring

------------------------------------------------------------------------

# 15. Advanced Distributed Scaling

At very large scale, combine:

``` text
Sharding
+
Replication
+
Failover
+
Rebalancing
+
Multi-region deployment
+
Traffic routing
```

Example:

``` mermaid
flowchart TD
    U[Global Users] --> G[Global Traffic Router]

    G --> US[US Region]
    G --> EU[EU Region]
    G --> AS[Asia Region]

    US --> USDB[(Sharded DB + Replicas)]
    EU --> EUDB[(Sharded DB + Replicas)]
    AS --> ASDB[(Sharded DB + Replicas)]
```

------------------------------------------------------------------------

# 16. Multi-Region Database Considerations

Multi-region databases can reduce latency and improve regional
availability.

But they introduce additional complexity:

-   Replication latency
-   Data residency
-   Conflict resolution
-   Failover
-   Regional outages
-   Consistency
-   Network partitions

Ask:

> Does the business actually need multi-region writes?

Do not introduce multi-region writes simply because the application has
global users.

Sometimes a single write region with regional read replicas is much
simpler.

------------------------------------------------------------------------

# 17. Database Failover

A resilient database architecture needs a plan for primary failure.

Conceptually:

``` mermaid
flowchart LR
    P[Primary] --> R1[Replica 1]
    P --> R2[Replica 2]

    P -. Failure .-> X[Primary Down]

    R1 --> NEW[Promoted Primary]
```

Failover mechanisms may include:

-   Automated health checks
-   Leader election
-   Replica promotion
-   DNS/service discovery changes
-   Connection redirection

Consider:

-   Recovery Time Objective (RTO)
-   Recovery Point Objective (RPO)
-   Data loss tolerance
-   Failover duration

------------------------------------------------------------------------

# 18. Consistency Considerations

Scaling through replication introduces consistency decisions.

### Stronger Consistency

Reads should reflect recent writes according to the chosen consistency
guarantee.

Useful for:

-   Payments
-   Account balances
-   Critical permissions
-   Inventory in some workflows

### Eventual Consistency

Replicas may temporarily differ but converge.

Useful for:

-   Feeds
-   Analytics
-   Search indexes
-   Some recommendation systems

The correct choice depends on business requirements.

------------------------------------------------------------------------

# 19. Database Connection Management

Scaling application servers can accidentally overload the database.

Example:

``` text
20 App Servers
×
50 Connections each
=
1000 DB connections
```

If the database can safely support only 300 connections, the application
fleet can create a database outage.

Use:

-   Connection pooling
-   Pool limits
-   Timeouts
-   Backpressure
-   Query timeouts
-   Connection monitoring

``` mermaid
flowchart LR
    A1[App 1] --> P[Connection Pool]
    A2[App 2] --> P
    A3[App 3] --> P
    P --> DB[(Database)]
```

------------------------------------------------------------------------

# 20. Caching Around the Database

Caching can delay or reduce the need for more database capacity.

``` mermaid
flowchart LR
    APP[Application] --> C[(Redis)]
    C -->|Cache Miss| DB[(Database)]
    DB --> C
```

Typical candidates:

-   User profiles
-   Product catalog
-   Configuration
-   Frequently accessed metadata
-   Expensive computed results

But avoid caching everything.

Ask:

-   Is the data read frequently?
-   Is it expensive to compute?
-   Can stale data be tolerated?
-   How will invalidation work?

------------------------------------------------------------------------

# 21. Database Scaling Decision Tree

``` mermaid
flowchart TD
    START[Database is slow] --> Q{Slow queries?}

    Q -->|Yes| OPT[Optimize Queries]
    OPT --> IDX{Missing indexes?}

    Q -->|No| RES{Resource bottleneck?}
    IDX -->|Yes| INDEX[Add / Tune Indexes]
    IDX -->|No| PART{Large tables?}

    INDEX --> MEASURE[Measure Again]
    PART -->|Yes| PARTITION[Partition Tables]
    PART -->|No| RES

    RES -->|CPU / RAM / I/O| VERT[Vertical Scale]
    RES -->|Read load| REPLICA[Add Read Replica]

    REPLICA --> MORE{Read load still high?}
    MORE -->|Yes| REPLICAS[Add More Replicas]
    MORE -->|No| DONE[Monitor]

    REPLICAS --> WRITE{Write or dataset limit?}
    WRITE -->|No| DONE
    WRITE -->|Yes| SHARD[Introduce Sharding]

    SHARD --> DIST[Sharding + Replication]
    DIST --> GLOBAL{Need global scale?}
    GLOBAL -->|Yes| MULTI[Multi-Region]
    GLOBAL -->|No| DONE
```

------------------------------------------------------------------------

# 22. What Each Scaling Technique Solves

  Technique               Primary Problem Solved
  ----------------------- -----------------------------------------------
  Query optimization      Excessive database work
  Indexing                Slow lookups
  Partitioning            Very large tables / partition pruning
  Vertical scaling        Resource limits on one node
  Read replicas           High read traffic
  Multiple replicas       Further read scaling / availability
  Sharding                Dataset or write workload exceeds one cluster
  Replication per shard   Read scaling + availability for shards
  Rebalancing             Uneven shard distribution
  Multi-region            Global latency / regional availability

------------------------------------------------------------------------

# 23. Common Mistakes

## Mistake 1 --- Sharding Too Early

Sharding adds:

-   Routing complexity
-   Operational overhead
-   Cross-shard queries
-   Rebalancing
-   Migration complexity

Optimize first.

## Mistake 2 --- Adding Indexes Everywhere

Indexes improve some reads but make writes and storage more expensive.

## Mistake 3 --- Assuming Replicas Solve Everything

Replicas primarily help read scalability.

They do not automatically solve write bottlenecks.

## Mistake 4 --- Ignoring Replication Lag

A replica is not necessarily an up-to-date copy at every instant.

## Mistake 5 --- Choosing a Poor Shard Key

A bad shard key can create:

-   Hot shards
-   Uneven storage
-   Expensive migrations
-   Cross-shard queries

## Mistake 6 --- Ignoring Connection Limits

More application servers can mean more database connections.

## Mistake 7 --- Scaling Without Measurement

Always measure before and after a scaling change.

------------------------------------------------------------------------

# 24. Practical Scaling Journey

Imagine a SaaS product.

### Stage 1 --- Early Product

``` text
App
 ↓
Primary DB
```

Optimize queries and indexes.

### Stage 2 --- Growing Traffic

``` text
App Servers
     ↓
Primary DB
```

Scale application servers horizontally.

### Stage 3 --- Read-Heavy

``` text
             ┌→ Replica 1
App → Primary ├→ Replica 2
             └→ Replica 3
```

Move appropriate reads to replicas.

### Stage 4 --- Huge Tables

``` text
Primary
  ↓
Partitioned Tables
```

Partition large datasets.

### Stage 5 --- Database Limit

``` text
Application
     ↓
Shard Router
  ↙   ↓   ↘
S1    S2    S3
```

Introduce sharding.

### Stage 6 --- Large Distributed System

``` text
Global Router
   ↓     ↓     ↓
 US     EU    Asia
  ↓      ↓      ↓
Sharded + Replicated Databases
```

Add multi-region capabilities only when requirements justify them.

------------------------------------------------------------------------

# 25. Final Mental Model

When a database becomes slow, think in this order:

``` text
1. Measure
   ↓
2. Optimize queries
   ↓
3. Add / tune indexes
   ↓
4. Partition large tables
   ↓
5. Scale the primary vertically
   ↓
6. Add Read Replica #1
   ↓
7. Add Read Replicas #2, #3...
   ↓
8. Identify write / storage bottlenecks
   ↓
9. Introduce sharding
   ↓
10. Replicate each shard
   ↓
11. Rebalance as data grows
   ↓
12. Add multi-region architecture if required
```

The most important lesson:

> **Database scaling is a progression, not a single technology.**

Start with the cheapest and simplest optimization.

Only introduce distributed complexity when the current architecture has
reached a real limit.

``` text
Optimize
   ↓
Index
   ↓
Partition
   ↓
Replicate
   ↓
Scale Reads
   ↓
Shard
   ↓
Replicate Shards
   ↓
Distribute Globally
```

A good database architecture is not the one with the most nodes.

It is the one that handles the required workload reliably while keeping
complexity, cost, and operational risk under control.
