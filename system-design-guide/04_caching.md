<p align="center">
  <a href="https://topmate.io/codewithayaan/new/wMSkSWH5su">
    <img src="https://github.com/user-attachments/assets/2f2744d7-3852-4072-b95d-db813b373ea0" alt="thumbnail" width="100%" />
  </a>
</p>





# How to Implement Caching with Redis

A practical guide to implementing production-ready caching with Redis,
Node.js, cache-aside patterns, TTLs, invalidation, stampede protection,
monitoring, and scaling.

## 1. What caching solves

Caching stores frequently accessed data in a faster layer so repeated
requests avoid expensive database work.

Without cache:

``` text
Client → API → Database → Response
```

With Redis:

``` text
Client → API → Redis
              ├─ HIT  → Response
              └─ MISS → Database → Redis → Response
```

Main benefits:

-   Lower API latency
-   Lower database CPU and I/O
-   Higher read throughput
-   Better scalability
-   Lower infrastructure cost

Caching should improve performance; it should not automatically become
the source of truth.

------------------------------------------------------------------------

## 2. When to cache

Good candidates:

-   Frequently read records
-   Expensive database queries
-   Product/catalog data
-   User profiles
-   Configuration
-   Aggregations
-   Search results with short freshness requirements
-   Expensive API responses

Avoid or carefully evaluate caching when:

-   Data changes constantly
-   Strong consistency is mandatory
-   Data is rarely requested
-   Fetching the data is already cheap
-   Cache invalidation is harder than the performance benefit

Ask:

> Is the cost of fetching this data greater than the cost and complexity
> of maintaining its cache?

------------------------------------------------------------------------

## 3. Why Redis?

Redis is an in-memory data store with low-latency operations and useful
structures for caching.

Common structures:

  Redis type   Typical cache use
  ------------ -----------------------------------------
  String       JSON object, API response, simple value
  Hash         Object fields
  Set          Unique collections
  Sorted Set   Ranking, time/score based data
  List         Ordered collections

For a normal API cache, Redis Strings containing serialized JSON are
often enough.

------------------------------------------------------------------------

# 4. Cache-Aside: the recommended starting pattern

Cache-aside is one of the most common application caching patterns.

### Read

``` text
Request
   ↓
Build cache key
   ↓
Redis GET
   ↓
 ┌───────────────┐
 │ Cache exists? │
 └──────┬────────┘
     YES│       NO
        ↓        ↓
     Return    Database
                ↓
             Redis SET
                ↓
             Return
```

High-level pseudocode:

``` text
receive request

check cache

if cached data exists:
    return cached data

fetch data from database

if data exists:
    store data in cache with TTL

return data
```

### Write

``` text
Update request
      ↓
Update database
      ↓
Invalidate cache
      ↓
Return success
```

This is usually a simple and safe starting point because the database
remains authoritative.

------------------------------------------------------------------------

# 5. Cache key design

A cache key must uniquely identify the data being cached.

Bad:

``` text
123
```

Better:

``` text
product:123
```

For a multi-tenant SaaS:

``` text
tenant:123:product:456
```

For a user:

``` text
user:123:profile
```

For an API response:

``` text
api:products:page:1:limit:20:sort:price
```

Every parameter that changes the response should be represented in the
key.

For example, these must not share one key:

``` text
page=1
page=2
sort=price
sort=name
```

Recommended convention:

``` text
<domain>:<resource>:<identifier>
```

Use versioning when the cached representation changes:

``` text
product:v1:123
product:v2:123
```

------------------------------------------------------------------------

# 6. TTL --- Time To Live

TTL determines how long a cache entry remains valid.

Example:

``` text
product:123 → TTL 300 seconds
```

After five minutes, the key expires.

Typical starting points:

  Data                        Example TTL
  ----------------------- ---------------
  Product data                  5--30 min
  User profile                  5--15 min
  Search result             30 sec--5 min
  Feature configuration         5--30 min
  Exchange rate                  1--5 min
  Static configuration      30 min--24 hr

These are starting points, not universal rules.

Choose TTL based on:

1.  How often data changes
2.  How bad stale data would be
3.  Query cost
4.  Traffic volume
5.  Business requirements

Do not use one TTL for everything.

------------------------------------------------------------------------

# 7. Cache invalidation

Suppose:

``` text
Database = price 899
Redis    = price 999
```

The cache is stale.

Common strategies:

### Delete

``` text
Update database
      ↓
Delete cache key
```

The next read rebuilds the cache.

### Update

``` text
Update database
      ↓
Update cache with new value
```

### TTL

Let the entry expire automatically.

### Versioned keys

``` text
product:123:v1
product:123:v2
```

For many CRUD systems, a practical approach is:

``` text
1. Update database
2. Delete affected cache key
3. Return success
```

The next read gets fresh data.

------------------------------------------------------------------------

# 8. Redis + Node.js setup

Create a project:

``` bash
mkdir redis-cache-demo
cd redis-cache-demo
npm init -y
npm install express redis
```

Run Redis locally:

``` bash
docker run --name redis-cache -p 6379:6379 -d redis
```

Verify:

``` bash
docker exec -it redis-cache redis-cli
```

Then:

``` text
PING
```

Expected:

``` text
PONG
```

Recommended project structure:

``` text
src/
├── redis.js
├── cache.js
├── product.service.js
└── server.js
```

------------------------------------------------------------------------

# 9. Redis connection

`src/redis.js`

``` javascript
const { createClient } = require("redis");

const redis = createClient({
  url: process.env.REDIS_URL || "redis://localhost:6379"
});

redis.on("error", (error) => {
  console.error("Redis error:", error);
});

async function connectRedis() {
  if (!redis.isOpen) {
    await redis.connect();
  }
}

module.exports = {
  redis,
  connectRedis
};
```

Do not create a new Redis connection for every HTTP request.

Create the connection once and reuse it.

------------------------------------------------------------------------

# 10. Reusable cache helper

`src/cache.js`

``` javascript
const { redis } = require("./redis");

async function getCache(key) {
  const value = await redis.get(key);

  if (!value) {
    return null;
  }

  return JSON.parse(value);
}

async function setCache(key, value, ttlSeconds = 300) {
  await redis.set(
    key,
    JSON.stringify(value),
    { EX: ttlSeconds }
  );
}

async function deleteCache(key) {
  await redis.del(key);
}

module.exports = {
  getCache,
  setCache,
  deleteCache
};
```

This keeps Redis-specific operations in one place.

------------------------------------------------------------------------

# 11. Implement cache-aside

Suppose the API is:

``` text
GET /products/:id
```

Service logic:

``` javascript
async function getProduct(id) {
  const cacheKey = `product:${id}`;

  const cached = await getCache(cacheKey);

  if (cached) {
    return cached;
  }

  const product = await database.products.findById(id);

  if (!product) {
    return null;
  }

  await setCache(cacheKey, product, 300);

  return product;
}
```

Flow:

``` text
GET /products/123
       ↓
Redis GET product:123
       ↓
     HIT?
   /      \
 YES      NO
  ↓        ↓
Return   Database
           ↓
        Redis SET
           ↓
         Return
```

------------------------------------------------------------------------

# 12. Implement invalidation

For:

``` text
PUT /products/:id
```

``` javascript
async function updateProduct(id, data) {
  const product = await database.products.update(id, data);

  await deleteCache(`product:${id}`);

  return product;
}
```

This produces:

``` text
Client
  ↓
API
  ↓
Database update
  ↓
Redis DEL product:123
  ↓
Success
```

The next read rebuilds the cache.

------------------------------------------------------------------------

# 13. Cache hit and miss metrics

Track both.

``` javascript
const cached = await getCache(key);

if (cached) {
  metrics.cacheHits++;
  return cached;
}

metrics.cacheMisses++;
```

Hit ratio:

``` text
Hit Ratio =
Hits / (Hits + Misses)
```

Example:

``` text
Hits   = 90,000
Misses = 10,000

Hit Ratio = 90%
```

Do not optimize only for hit ratio. Correctness, latency, memory, and
database protection also matter.

------------------------------------------------------------------------

# 14. Cache stampede / thundering herd

Suppose a popular key expires:

``` text
Popular key expires
       ↓
10,000 requests arrive
       ↓
10,000 cache misses
       ↓
10,000 database queries
       ↓
Database overloaded
```

Solutions:

### TTL jitter

Instead of:

``` text
TTL = 300 seconds
```

Use:

``` text
TTL = 300 + random small value
```

This prevents many keys expiring simultaneously.

### Distributed lock

``` text
Request A → miss → acquire lock → DB → cache
Request B → miss → wait
Request C → miss → wait
```

Only one request rebuilds the key.

### Background refresh

Refresh popular entries before they expire.

------------------------------------------------------------------------

# 15. Cache penetration

A client may repeatedly request a record that does not exist.

``` text
GET /users/999999
      ↓
Redis miss
      ↓
Database miss
```

Repeated requests keep reaching the database.

Use negative caching:

``` text
user:999999 → NOT_FOUND
TTL = 30 seconds
```

Use a short TTL because the resource could be created later.

------------------------------------------------------------------------

# 16. Cache avalanche

An avalanche happens when many keys expire together.

Bad:

``` text
100,000 keys
     ↓
same TTL
     ↓
expire together
     ↓
huge database load
```

Protection:

-   TTL jitter
-   Different TTLs
-   Cache warming
-   Refresh-ahead
-   Rate limiting expensive rebuilds
-   Background refresh

------------------------------------------------------------------------

# 17. Cache warming

Cache warming preloads important data before users request it.

``` text
Scheduled job / startup
        ↓
Fetch popular data
        ↓
Store in Redis
        ↓
Users arrive
        ↓
Cache hit
```

Useful for:

-   Popular products
-   Home-page data
-   Leaderboards
-   Frequently used configuration
-   Known high-traffic resources

------------------------------------------------------------------------

# 18. Refresh-ahead

Refresh-ahead proactively refreshes a cache entry before its TTL
expires.

Example:

``` text
TTL = 10 minutes

After 8 minutes
      ↓
Background refresh
      ↓
Fetch latest DB data
      ↓
Update Redis
      ↓
Reset TTL
```

This is especially useful for frequently accessed and relatively stable
data.

------------------------------------------------------------------------

# 19. Write-through cache

Write-through means the cache and database are updated as part of the
write path.

``` text
Application
    ├──→ Redis
    └──→ Database
```

High-level idea:

``` text
Receive update

update cache
update database

return success
```

Benefits:

-   Cache is populated after writes
-   Reads can immediately use cache
-   Useful when read-after-write behavior matters

Trade-off:

-   Writes become more expensive
-   Failure coordination is more complicated

------------------------------------------------------------------------

# 20. Write-back / write-behind

Write-back writes to cache first and persists to the database
asynchronously.

``` text
Application
    ↓
Redis
    ↓
Background worker
    ↓
Database
```

High-level flow:

``` text
Receive write
store in cache
mark data dirty
return quickly

background worker
write dirty data to database
```

This can provide high write throughput but introduces more complexity
and potential data-loss risk.

Use it only when the system can tolerate asynchronous persistence.

------------------------------------------------------------------------

# 21. Redis eviction policies

Redis memory is finite.

Common policies:

-   `noeviction`
-   `allkeys-lru`
-   `allkeys-lfu`
-   `volatile-lru`
-   `volatile-lfu`
-   `allkeys-random`
-   `volatile-ttl`

### LRU

Least Recently Used.

Remove entries that have not been accessed recently.

### LFU

Least Frequently Used.

Remove entries accessed least often.

Choose based on real traffic behavior.

------------------------------------------------------------------------

# 22. Distributed caching

With multiple application servers:

``` text
             Load Balancer
             /     |     \
           App1   App2   App3
             \      |      /
                  Redis
                    ↓
                Database
```

A shared Redis cache ensures all application instances can access the
same cached data.

Local-only caches can cause:

-   Duplicate cache data
-   Inconsistent values
-   Uneven hit ratios
-   More memory usage across servers

Start with shared Redis unless local caching is clearly justified.

------------------------------------------------------------------------

# 23. Multi-level caching

For very latency-sensitive systems:

``` text
Request
  ↓
Local memory cache
  ↓ miss
Redis
  ↓ miss
Database
```

This can be fast, but it introduces another invalidation layer.

Now you must manage:

-   Local TTL
-   Redis TTL
-   Local invalidation
-   Redis invalidation
-   Memory limits
-   Cross-instance consistency

Do not add this complexity without a measured need.

------------------------------------------------------------------------

# 24. API response caching

You can cache a complete response.

Example:

``` text
GET /products?category=electronics&page=1
```

Key:

``` text
api:products:electronics:page:1
```

Value:

``` text
{
  "products": [...],
  "page": 1,
  "total": 1000
}
```

This can dramatically reduce expensive query work.

But invalidation is harder because one database change may affect many
cached responses.

------------------------------------------------------------------------

# 25. Query result caching

Instead of caching the entire HTTP response:

``` text
Database query
      ↓
Redis
      ↓
Multiple APIs
```

This can allow multiple endpoints to reuse expensive computed data.

------------------------------------------------------------------------

# 26. Pagination and filters

Every parameter affecting the response should be part of the key.

Bad:

``` text
products
```

Better:

``` text
products:page:1:limit:20:sort:price:category:electronics
```

For multi-tenant systems:

``` text
tenant:123:products:page:1:limit:20
```

Never let page 1 overwrite page 2.

------------------------------------------------------------------------

# 27. Security and cache isolation

Never expose Redis directly to the public internet.

Recommended:

``` text
Internet
   ↓
Load Balancer
   ↓
Application
   ↓
Private Redis
```

Use:

-   Private networking
-   Authentication
-   TLS when appropriate
-   Firewall/security groups
-   Access controls
-   Strong credentials

For personalized data, include identity in the key:

``` text
user:123:profile
```

Never cache all users under:

``` text
profile
```

------------------------------------------------------------------------

# 28. Cache poisoning

Cache poisoning occurs when incorrect or attacker-controlled content is
stored and served to other users.

Protect against:

-   Unsafe cache keys
-   Missing authorization context
-   Unvalidated query parameters
-   Shared keys for personalized responses
-   Excessively long user-controlled keys

Validate input before constructing keys.

------------------------------------------------------------------------

# 29. Protect Redis from cache explosion

Attackers or poorly designed endpoints can generate millions of unique
keys.

Example:

``` text
search:<random-value-1>
search:<random-value-2>
search:<random-value-3>
...
```

Protect Redis by:

-   Limiting key size
-   Validating query parameters
-   Using short TTLs
-   Rate limiting expensive endpoints
-   Monitoring memory
-   Avoiding unnecessary high-cardinality caches

------------------------------------------------------------------------

# 30. Redis failure strategy

Redis should not automatically become a single point of failure.

For many read caches:

``` text
Redis unavailable
      ↓
Short timeout
      ↓
Fallback to database
```

But this can overload the database during Redis outages.

Therefore combine fallback with:

-   Timeouts
-   Circuit breakers
-   Rate limits
-   Database protection
-   Query limits

Never allow Redis requests to wait forever.

------------------------------------------------------------------------

# 31. Monitoring

Track application metrics:

``` text
Cache hit ratio
Cache miss ratio
Cache latency
Redis errors
Database query rate
API latency
```

Track Redis metrics:

``` text
Memory usage
Evictions
Commands/sec
Connected clients
CPU
Network traffic
Replication health
Cluster health
```

Business metrics can also be useful:

``` text
Product page latency
Search latency
Checkout latency
Database cost
Error rate
```

------------------------------------------------------------------------

# 32. Testing checklist

### Cache hit

``` text
Redis has data
→ return cached data
→ database is not queried
```

### Cache miss

``` text
Redis has no data
→ query database
→ write Redis
→ return result
```

### Database miss

``` text
Redis miss
→ database has no record
→ return 404/null
→ optionally negative-cache
```

### Update

``` text
Update database
→ invalidate cache
```

### Redis failure

``` text
Redis unavailable
→ timeout/fallback strategy
```

### Expiration

``` text
TTL expires
→ cache miss
→ database
→ rebuild cache
```

------------------------------------------------------------------------

# 33. Load testing

Compare before and after caching.

Before:

``` text
API latency
Database CPU
Database queries/sec
Database connections
```

After:

``` text
API latency
Redis latency
Cache hit ratio
Database CPU
Database queries/sec
```

Example:

``` text
Before:
p95 API latency = 300ms
DB CPU = 80%

After:
p95 API latency = 80ms
DB CPU = 35%
Cache hit ratio = 92%
```

The actual numbers depend on the workload.

------------------------------------------------------------------------

# 34. Production architecture

A practical starting architecture:

``` text
                    Users
                      │
                      ▼
                  CDN / WAF
                      │
                      ▼
                Load Balancer
                      │
             ┌────────┴────────┐
             │                 │
           App #1            App #2
             │                 │
             └────────┬────────┘
                      │
                      ▼
                 Redis Cache
                      │
                      ▼
                   Database
```

For many systems, start here:

``` text
Cache-Aside + Redis + TTL + Invalidation + Monitoring
```

Add advanced mechanisms only when needed.

------------------------------------------------------------------------

# 35. Redis Cluster

When one Redis node cannot provide enough memory or throughput, Redis
Cluster can distribute keys across nodes.

``` text
Application
     ↓
Redis Cluster
   /    |    \
Node1 Node2 Node3
```

Benefits:

-   Horizontal scaling
-   More available memory
-   Higher throughput

Trade-offs:

-   More operational complexity
-   More complicated failure handling
-   Key distribution considerations

Do not introduce a cluster before capacity requires it.

------------------------------------------------------------------------

# 36. Event-driven invalidation

For larger systems:

``` text
Application
    ↓
Database update
    ↓
Publish event
    ↓
Event bus
    ↓
Cache invalidation worker
    ↓
Redis
```

Example:

``` text
product.updated
```

A worker can invalidate:

``` text
product:123
products:category:10
products:search:laptop
```

This becomes useful when many services depend on the same data.

------------------------------------------------------------------------

# 37. Common mistakes

### 1. Caching everything

Only cache data where the performance benefit is meaningful.

### 2. No invalidation strategy

Every cache needs a plan for becoming stale.

### 3. Same TTL everywhere

Different data has different freshness requirements.

### 4. Incomplete keys

All response-changing parameters must be represented.

### 5. Personalized data under shared keys

This can leak data between users.

### 6. No Redis failure strategy

The application should have deliberate behavior when Redis is
unavailable.

### 7. Ignoring memory limits

Redis is memory-based.

### 8. No cache stampede protection

Popular keys can overload the database after expiration.

### 9. Making Redis the source of truth unnecessarily

For normal caching, the database should remain authoritative.

### 10. Adding too much complexity too early

Start with cache-aside and evolve based on measurements.

------------------------------------------------------------------------

# 38. Recommended implementation roadmap

``` text
1. Identify expensive/frequent reads
        ↓
2. Measure baseline latency + DB load
        ↓
3. Design cache keys
        ↓
4. Add Redis
        ↓
5. Implement cache-aside reads
        ↓
6. Add TTL
        ↓
7. Add invalidation
        ↓
8. Add hit/miss monitoring
        ↓
9. Test Redis failure
        ↓
10. Load test
        ↓
11. Add stampede protection if needed
        ↓
12. Add warming/refresh-ahead if needed
        ↓
13. Scale Redis when capacity requires it
```

------------------------------------------------------------------------

# 39. Production checklist

Before shipping:

-   [ ] Redis connection is reused
-   [ ] Cache keys are deterministic
-   [ ] Tenant/user isolation is correct
-   [ ] TTL is configured
-   [ ] Invalidation is implemented
-   [ ] Cache hit/miss metrics exist
-   [ ] Redis timeout exists
-   [ ] Redis failure strategy exists
-   [ ] Memory limits are configured
-   [ ] Eviction policy is intentional
-   [ ] Sensitive data is handled safely
-   [ ] Cache stampede is considered
-   [ ] Cache penetration is considered
-   [ ] Load tests have been performed
-   [ ] Database is still the source of truth

------------------------------------------------------------------------

# 40. Final mental model

When designing a Redis cache, answer these questions:

``` text
1. What should I cache?
2. Why should I cache it?
3. What is the cache key?
4. What is the TTL?
5. What happens on a cache hit?
6. What happens on a cache miss?
7. How is the cache invalidated?
8. What happens when Redis fails?
9. What happens when thousands of requests miss together?
10. How much memory is required?
11. How will I monitor it?
12. When should I scale Redis?
```

The recommended starting point is:

``` text
Cache-Aside
    +
Redis
    +
TTL
    +
Explicit Invalidation
    +
Monitoring
```

Then evolve only when the workload requires it:

``` text
Cache-Aside
   ↓
TTL
   ↓
Invalidation
   ↓
Monitoring
   ↓
Negative Caching
   ↓
TTL Jitter
   ↓
Cache Warming
   ↓
Refresh-Ahead
   ↓
Request Coalescing / Locking
   ↓
Distributed Redis
   ↓
Redis Cluster
```

> **Cache for performance, but keep correctness in the source of
> truth.**
