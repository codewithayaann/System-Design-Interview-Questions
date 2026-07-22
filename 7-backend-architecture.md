# 7 Backend Architecture Patterns Every Modern Engineer Should Know

# 1. Request / Response Pattern

## Overview

The Request/Response pattern is the backbone of traditional web
applications and REST APIs. In this model, a client sends an HTTP
request to a server and waits until the server finishes processing
before receiving a response. This synchronous communication model is
simple, predictable, and easy to debug, which is why it powers
authentication, CRUD operations, dashboards, payment confirmation, and
most business APIs. During the lifecycle of a request, the server may
validate authentication, execute business logic, query one or more
databases, and finally return JSON or HTML to the client. Because the
client waits for completion, slow database queries or external API calls
directly impact user experience. Request/Response works extremely well
for operations where immediate feedback is required, but it becomes
inefficient for long-running jobs like report generation, image
processing, or sending thousands of emails. In production systems it is
commonly combined with caching, CDNs, asynchronous workers, and queues
to reduce latency. Although microservices and event-driven architectures
have become popular, almost every modern backend still exposes
synchronous APIs because browsers, mobile apps, and third-party
integrations depend on them. Understanding this pattern is fundamental
before learning more advanced backend architectures.

## Architecture

``` mermaid
sequenceDiagram
participant Client
participant API
participant DB
Client->>API: HTTP Request
API->>DB: Query
DB-->>API: Data
API-->>Client: HTTP Response
```

## Best Use Cases

-   Production systems
-   Scalable backend services
-   System design interviews

## Advantages

-   Improves architecture for its intended purpose
-   Widely adopted in industry
-   Can be combined with other patterns

## Best Practices

-   Keep implementations simple.
-   Monitor failures and performance.
-   Choose the pattern based on business requirements, not trends.

------------------------------------------------------------------------

# 2. Event-Driven Pattern

## Overview

The Event-Driven pattern allows services to communicate asynchronously
through an event broker instead of directly calling one another. When an
important business event occurs, such as 'Order Created' or 'Payment
Completed', the producer publishes an event to Kafka, RabbitMQ, Amazon
SQS, or another messaging platform. Consumer services subscribe only to
the events they care about and process them independently. This removes
tight coupling between services and makes systems easier to scale and
extend. New consumers can be added without modifying the producer,
making this architecture ideal for notifications, analytics, audit
logging, recommendation engines, and background processing. Since
communication is asynchronous, users don't wait for every downstream
task to finish. However, engineers must design for eventual consistency,
duplicate events, retries, idempotency, and monitoring. Event-driven
systems power many large-scale applications because they improve
resilience and throughput while enabling independent deployment of
services.

## Architecture

``` mermaid
flowchart LR
Producer-->Broker((Kafka))
Broker-->Inventory
Broker-->Email
Broker-->Analytics
Broker-->Notification
```

## Best Use Cases

-   Production systems
-   Scalable backend services
-   System design interviews

## Advantages

-   Improves architecture for its intended purpose
-   Widely adopted in industry
-   Can be combined with other patterns

## Best Practices

-   Keep implementations simple.
-   Monitor failures and performance.
-   Choose the pattern based on business requirements, not trends.

------------------------------------------------------------------------

# 3. Cache-Aside Pattern

## Overview

Cache-Aside is one of the most widely used caching strategies for
backend applications. Instead of querying the database for every
request, the application first checks a fast in-memory cache such as
Redis or Memcached. If the requested data exists, it is immediately
returned, dramatically reducing response time. If the cache misses, the
application retrieves the data from the database, stores it in the
cache, and then returns it to the client. This approach significantly
reduces database load, improves scalability, and delivers better user
experiences. It is commonly used for user profiles, product catalogs,
dashboards, trending content, and configuration data. The biggest
engineering challenge is cache invalidation because stale data can lead
to inconsistent user experiences. Choosing an appropriate TTL, updating
cache after writes, and avoiding cache stampedes are essential best
practices.

## Architecture

``` mermaid
flowchart LR
Client-->App
App-->Cache[(Redis)]
Cache--Hit-->Client
Cache--Miss-->DB[(Database)]
DB-->Cache
Cache-->Client
```

## Best Use Cases

-   Production systems
-   Scalable backend services
-   System design interviews

## Advantages

-   Improves architecture for its intended purpose
-   Widely adopted in industry
-   Can be combined with other patterns

## Best Practices

-   Keep implementations simple.
-   Monitor failures and performance.
-   Choose the pattern based on business requirements, not trends.

------------------------------------------------------------------------

# 4. CQRS

## Overview

Command Query Responsibility Segregation separates read and write
operations into independent models. Commands modify application state
through a write model, while queries retrieve data from an optimized
read model. This separation enables each side to scale independently and
use different storage technologies if necessary. Large systems with
significantly more reads than writes benefit greatly because read models
can be optimized without affecting transactional writes. CQRS is
commonly paired with Event Sourcing, where every change generates an
event used to update read projections. Although the architecture
introduces more infrastructure and eventual consistency, it simplifies
complex domains and improves performance in banking, e-commerce,
booking, and analytics systems. Teams adopting CQRS should understand
synchronization, event replay, and monitoring to ensure read models
remain accurate.

## Architecture

``` mermaid
flowchart LR
Client-->CommandAPI
Client-->QueryAPI
CommandAPI-->WriteDB
WriteDB-->Events((Event Bus))
Events-->ReadDB
QueryAPI-->ReadDB
```

## Best Use Cases

-   Production systems
-   Scalable backend services
-   System design interviews

## Advantages

-   Improves architecture for its intended purpose
-   Widely adopted in industry
-   Can be combined with other patterns

## Best Practices

-   Keep implementations simple.
-   Monitor failures and performance.
-   Choose the pattern based on business requirements, not trends.

------------------------------------------------------------------------

# 5. Strangler Fig Pattern

## Overview

The Strangler Fig pattern is an incremental migration strategy for
replacing legacy monolithic applications with modern services. Rather
than rewriting an entire application at once, engineers introduce new
services around the existing system. An API Gateway gradually routes
traffic to new services while legacy modules continue serving remaining
requests. Over time, more functionality moves into microservices until
the monolith can be safely retired. This approach minimizes downtime,
lowers migration risk, and enables continuous delivery throughout
modernization. It is commonly used by enterprises modernizing ERP
systems, banking platforms, and large SaaS applications where a full
rewrite would be too risky. Success depends on identifying clear service
boundaries, maintaining backward compatibility, and migrating features
incrementally.

## Architecture

``` mermaid
flowchart LR
Client-->Gateway
Gateway-->Monolith
Gateway-->UserSvc
Gateway-->OrderSvc
Gateway-->PaymentSvc
Monolith-.Replace.->UserSvc
```

## Best Use Cases

-   Production systems
-   Scalable backend services
-   System design interviews

## Advantages

-   Improves architecture for its intended purpose
-   Widely adopted in industry
-   Can be combined with other patterns

## Best Practices

-   Keep implementations simple.
-   Monitor failures and performance.
-   Choose the pattern based on business requirements, not trends.

------------------------------------------------------------------------

# 6. Saga Pattern

## Overview

The Saga pattern manages distributed business transactions across
multiple microservices without relying on a single ACID transaction.
Each service performs its own local transaction and publishes an event
or invokes the next service. If every step succeeds, the workflow
completes successfully. If any step fails, compensating transactions
undo previous operations to restore consistency. For example, during
e-commerce checkout, payment may succeed, inventory may be reserved, but
shipping creation may fail. Instead of rolling back a global
transaction, the system refunds payment and restores inventory through
compensation logic. Saga improves scalability because services remain
independent, but designing reliable compensation workflows, monitoring
execution, and handling retries require careful engineering.

## Architecture

``` mermaid
flowchart LR
Order-->Payment-->Inventory-->Shipping
Shipping-.Failure.->Refund
Refund-->RestoreInventory
```

## Best Use Cases

-   Production systems
-   Scalable backend services
-   System design interviews

## Advantages

-   Improves architecture for its intended purpose
-   Widely adopted in industry
-   Can be combined with other patterns

## Best Practices

-   Keep implementations simple.
-   Monitor failures and performance.
-   Choose the pattern based on business requirements, not trends.

------------------------------------------------------------------------

# 7. Database per Service Pattern

## Overview

Database per Service is a core microservices principle where every
service owns its own datastore. No service directly accesses another
service's database; instead, data is shared through APIs or asynchronous
events. This creates loose coupling, allows teams to deploy
independently, and lets each service choose the most appropriate
database technology, such as PostgreSQL for users, MongoDB for
documents, or ClickHouse for analytics. While this architecture improves
autonomy and fault isolation, it introduces challenges around reporting,
cross-service queries, distributed joins, and maintaining consistency.
Patterns such as API Composition, CQRS, and Event-Driven Architecture
are often combined with Database per Service to solve these challenges.

## Architecture

``` mermaid
flowchart TB
Gateway-->User
Gateway-->Order
Gateway-->Payment
Gateway-->Analytics
User-->PG[(PostgreSQL)]
Order-->MySQL[(MySQL)]
Payment-->Mongo[(MongoDB)]
Analytics-->Click[(ClickHouse)]
```

## Best Use Cases

-   Production systems
-   Scalable backend services
-   System design interviews

## Advantages

-   Improves architecture for its intended purpose
-   Widely adopted in industry
-   Can be combined with other patterns

## Best Practices

-   Keep implementations simple.
-   Monitor failures and performance.
-   Choose the pattern based on business requirements, not trends.
