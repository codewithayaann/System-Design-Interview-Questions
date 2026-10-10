<p align="center">
  <a href="https://topmate.io/codewithayaan/new/wMSkSWH5su">
    <img src="https://github.com/user-attachments/assets/2f2744d7-3852-4072-b95d-db813b373ea0" alt="thumbnail" width="100%" />
  </a>
</p>


# Message Queues: Complete Practical Guide

A hands-on system-design guide with Mermaid diagrams, problem
statements, RabbitMQ and Kafka setup, Node.js examples, trade-offs,
reliability patterns, and production guidance.

## Table of contents

1.  [Problem statement](#1-problem-statement)
2.  [Core concepts](#2-core-concepts)
3.  [Mermaid diagrams](#3-mermaid-diagrams)
4.  [Trade-offs and use cases](#4-trade-offs-and-use-cases)
5.  [Set up RabbitMQ](#5-set-up-rabbitmq)
6.  [RabbitMQ implementation](#6-rabbitmq-implementation)
7.  [Set up Kafka](#7-set-up-kafka)
8.  [Kafka implementation](#8-kafka-implementation)
9.  [Reliability, retries, and DLQs](#9-reliability-retries-and-dlqs)
10. [Scaling, monitoring, and production
    checklist](#10-scaling-monitoring-and-production-checklist)
11. [Interview questions](#11-interview-questions)
12. [Official references](#12-official-references)

## 1. Problem statement

Imagine an e-commerce API that accepts an order and then sends a
confirmation email, reserves inventory, notifies a warehouse, and
updates analytics. If every task runs before the HTTP response is
returned, users wait for slow services. If the email provider is
unavailable, the whole request may fail even after the order has been
saved. Traffic spikes can overwhelm downstream systems, and retrying the
whole request can duplicate business operations.

A message broker lets the API persist the order, publish a message, and
return promptly. Background consumers handle email, inventory, and
analytics independently. The broker buffers bursts and can retain work
while consumers recover.

``` mermaid
flowchart TD
    U[Customer] --> API[Order API]
    API --> DB[(Orders database)]
    API --> Q[Message broker]
    Q --> E[Email worker]
    Q --> I[Inventory worker]
    Q --> A[Analytics worker]
    E --> ESP[Email provider]
    I --> STOCK[Inventory service]
    A --> DW[Analytics store]
```

**Problem statement:** Design an asynchronous order-processing system
that responds quickly, isolates downstream failures, absorbs traffic
spikes, supports retries, avoids duplicate business effects, and
provides visibility into failed or delayed work.

A queue is not always necessary. Use a direct synchronous call when the
caller needs an immediate result and the operation is quick. A broker
adds infrastructure, eventual consistency, operational overhead, and new
failure modes.

## 2. Core concepts

-   **Producer:** creates/publishes a message.
-   **Broker:** accepts, routes, and stores messages.
-   **Queue:** a work destination, commonly used with RabbitMQ.
-   **Topic/partition:** Kafka's named event stream and its ordered
    subdivisions.
-   **Consumer:** reads and processes messages.
-   **Acknowledgment:** confirms successful handling in RabbitMQ
    manual-ack workflows.
-   **Offset:** Kafka consumer position; committing it records where
    consumption should resume.
-   **Retry:** another attempt after a transient failure.
-   **Dead-letter queue/topic (DLQ):** holds messages that exhaust the
    retry policy or cannot be handled.
-   **Consumer group:** Kafka consumers that share partition
    assignments.
-   **Idempotency:** processing the same message again does not repeat
    the business effect.

### Example event envelope

``` json
{
  "id": "evt_01JABC123",
  "type": "order.created",
  "version": 1,
  "occurredAt": "2026-10-10T10:30:00.000Z",
  "correlationId": "req_8f2b",
  "payload": {
    "orderId": "ord_123",
    "customerId": "cus_456",
    "total": 1499
  }
}
```

Prefer stable IDs, explicit event types, schema versions, and
correlation IDs. Avoid secrets and unnecessary personal data in
messages.

### Delivery semantics

  --------------------------------------------------------------------------
  Guarantee               Meaning                 Trade-off
  ----------------------- ----------------------- --------------------------
  At-most-once            Delivered zero or one   Loss is possible
                          time                    

  At-least-once           Retried according to    Duplicates are possible
                          policy until handled    

  Exactly-once business   The final business      Requires
  effect                  effect occurs once      transactional/idempotent
                          despite retries         design; end-to-end
                                                  exactly-once is difficult
  --------------------------------------------------------------------------

A common practical target is **at-least-once delivery plus idempotent
consumers**. Broker guarantees alone do not make external side effects
exactly once.

## 3. Mermaid diagrams

### Basic message flow

``` mermaid
flowchart LR
    P[Producer] -->|Publish| B[Broker]
    B --> Q[(Queue or topic)]
    Q --> C[Consumer]
    C -->|Success| ACK[Acknowledge or commit]
    C -->|Temporary failure| R[Bounded retry with backoff]
    R --> Q
    C -->|Retries exhausted| DLQ[(DLQ)]
```

### Synchronous versus asynchronous

``` mermaid
flowchart TD
    subgraph SYNC["Synchronous"]
      U1[User] --> API1[API]
      API1 --> TASK1[Slow task]
      TASK1 --> U1
    end
    subgraph ASYNC["Asynchronous"]
      U2[User] --> API2[API]
      API2 --> Q2[(Broker)]
      API2 -->|Fast response| U2
      Q2 --> W[Background worker]
      W --> S[External service]
    end
```

### RabbitMQ routing

RabbitMQ commonly routes from a producer through an exchange; the
exchange uses bindings and routing keys to choose queues.

``` mermaid
flowchart LR
    P[Producer] --> X{Topic exchange}
    X -->|order.created| Q1[(Email queue)]
    X -->|order.created| Q2[(Inventory queue)]
    X -->|order.*| Q3[(Audit queue)]
    Q1 --> C1[Email worker]
    Q2 --> C2[Inventory worker]
    Q3 --> C3[Audit worker]
```

### Kafka partitions and consumer group

``` mermaid
flowchart LR
    P[Producer] --> T[Topic: order-events]
    T --> P0[Partition 0]
    T --> P1[Partition 1]
    T --> P2[Partition 2]
    subgraph G["Consumer group"]
      C0[Consumer A]
      C1[Consumer B]
      C2[Consumer C]
    end
    P0 --> C0
    P1 --> C1
    P2 --> C2
```

A partition is assigned to at most one consumer in a group at a time.
More consumers than partitions in a group will leave some consumers
idle. Different groups can independently consume the same retained
events.

### Transactional outbox

``` mermaid
flowchart LR
    API[Application] --> TX[Single database transaction]
    TX --> DB[(Business tables)]
    TX --> OUT[(Outbox table)]
    OUT --> PUB[Outbox publisher]
    PUB --> Q[Broker]
    Q --> C[Idempotent consumer]
```

The outbox pattern prevents the gap where the database commits but the
event is never published. The business change and outbox row are written
in one transaction; a publisher sends pending rows later. It can publish
duplicates after uncertain failures, so consumers must be idempotent.

## 4. Trade-offs and use cases

  -----------------------------------------------------------------------
  Dimension               RabbitMQ                Apache Kafka
  ----------------------- ----------------------- -----------------------
  Core model              Broker, exchanges,      Distributed retained
                          bindings, queues        log, topics, partitions

  Typical fit             Task queues and         Event streaming,
                          flexible routing        replay, data pipelines

  Consumption             Messages commonly acked Consumers track
                          and removed from queue  offsets; events remain
                                                  per retention policy

  Routing                 Direct, topic, fanout,  Topics/partitions;
                          headers exchanges       event fan-out through
                                                  consumer groups

  Replay                  Requires deliberate     Natural fit: reread
                          design                  retained events from
                                                  offsets

  Scaling                 Queue topology,         Partition count and
                          consumers, broker setup consumer groups are
                                                  central

  Ordering                Depends on queue and    Within a partition, not
                          consumer topology;      globally across
                          retries can reorder     partitions

  Operational focus       Queue depth, unacked    Consumer lag,
                          messages, consumer      partitions,
                          capacity                replication, disk,
                                                  retention
  -----------------------------------------------------------------------

### Choose RabbitMQ when

-   You need background jobs, task distribution, or flexible routing.
-   A worker should acknowledge successful completion.
-   Multiple queues need different subsets of messages.
-   Long-term replay of a large event history is not the main
    requirement.

### Choose Kafka when

-   Multiple systems independently consume the same events.
-   Replaying retained events is important.
-   You are building event pipelines, analytics, logs, or stream
    processing.
-   High throughput and partition-based scaling are core requirements.

Neither is automatically faster or more reliable. Performance depends on
message size, batching, persistence, replication, acknowledgments,
topology, hardware, and consumer work. Benchmark the actual workload.

### Other use cases

Email/SMS notifications, webhook delivery, PDF/report generation,
image/video processing, background jobs, search indexing,
service-to-service events, data synchronization, audit streams,
analytics, and buffering bursty workloads.

## 5. Set up RabbitMQ

For local development, create `docker-compose.rabbitmq.yml`:

``` yaml
services:
  rabbitmq:
    image: rabbitmq:4-management
    hostname: rabbitmq
    ports:
      - "5672:5672"
      - "15672:15672"
    environment:
      RABBITMQ_DEFAULT_USER: app
      RABBITMQ_DEFAULT_PASS: local_dev_password
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "-q", "ping"]
      interval: 10s
      timeout: 5s
      retries: 10

volumes:
  rabbitmq_data:
```

Start it:

``` bash
docker compose -f docker-compose.rabbitmq.yml up -d
docker compose -f docker-compose.rabbitmq.yml ps
docker compose -f docker-compose.rabbitmq.yml logs -f rabbitmq
```

Management UI: `http://localhost:15672`\
Local credentials: `app` / `local_dev_password`\
AMQP port: `5672`

Stop it with `docker compose -f docker-compose.rabbitmq.yml down`. Use
`down -v` only when you intentionally want to delete the local data
volume. These credentials are for local development only.

Install Node.js packages:

``` bash
mkdir rabbitmq-demo && cd rabbitmq-demo
npm init -y
npm install amqplib
npm install -D typescript tsx @types/node @types/amqplib
mkdir src
export AMQP_URL='amqp://app:local_dev_password@localhost:5672'
```

## 6. RabbitMQ implementation

### Producer: `src/producer.ts`

``` typescript
import amqp from "amqplib";
import { randomUUID } from "node:crypto";

const url = process.env.AMQP_URL ?? "amqp://guest:guest@localhost:5672";
const queue = "email-jobs";

async function main() {
  const connection = await amqp.connect(url);
  const channel = await connection.createConfirmChannel();

  await channel.assertQueue(queue, { durable: true });

  const job = {
    id: randomUUID(),
    type: "send_email",
    payload: {
      to: "learner@example.com",
      subject: "Welcome",
      body: "Your account is ready."
    }
  };

  channel.sendToQueue(queue, Buffer.from(JSON.stringify(job)), {
    persistent: true,
    contentType: "application/json",
    messageId: job.id
  });

  // Wait for broker publisher confirms. sendToQueue's boolean alone
  // is only client-side flow-control information.
  await channel.waitForConfirms();
  console.log("Published", job.id);

  await channel.close();
  await connection.close();
}

main().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
```

### Consumer: `src/worker.ts`

``` typescript
import amqp from "amqplib";

const url = process.env.AMQP_URL ?? "amqp://guest:guest@localhost:5672";
const queue = "email-jobs";

type EmailJob = {
  id: string;
  type: "send_email";
  payload: { to: string; subject: string; body: string };
};

async function processJob(job: EmailJob) {
  // Replace with a real email provider call.
  // Use job.id as an idempotency key where the provider supports it.
  console.log(`Email to ${job.payload.to}: ${job.payload.subject}`);
}

async function main() {
  const connection = await amqp.connect(url);
  const channel = await connection.createChannel();

  await channel.assertQueue(queue, { durable: true });
  await channel.prefetch(10);

  console.log(`Waiting for jobs on ${queue}`);

  await channel.consume(queue, async (message) => {
    if (!message) return;

    try {
      const job = JSON.parse(message.content.toString()) as EmailJob;
      await processJob(job);
      channel.ack(message);
    } catch (error) {
      console.error("Processing failed", error);

      // Reject without requeue to avoid an infinite hot loop.
      // Configure a dead-letter exchange/queue before using this in production.
      channel.nack(message, false, false);
    }
  }, { noAck: false });

  const shutdown = async () => {
    await channel.close().catch(() => undefined);
    await connection.close().catch(() => undefined);
  };
  process.once("SIGINT", () => void shutdown());
  process.once("SIGTERM", () => void shutdown());
}

main().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
```

Add scripts to `package.json`:

``` json
{
  "scripts": {
    "producer": "tsx src/producer.ts",
    "worker": "tsx src/worker.ts"
  }
}
```

Run `npm run worker` in one terminal and `npm run producer` in another.
The example is intentionally small. The `nack(..., false, false)` path
may discard a message if no dead-letter exchange is configured; add a
DLQ before using it for real workloads.

### RabbitMQ reliability notes

-   Use durable queues and persistent messages where required.
-   Use publisher confirms to learn whether the broker accepted a
    publication.
-   Use manual acknowledgments and ack only after processing succeeds.
-   Use prefetch to limit unacknowledged work.
-   Configure bounded retries and a dead-letter exchange/queue.
-   Avoid immediate infinite requeue loops.
-   For high availability, choose an appropriate replicated queue
    strategy and verify the behavior under node failure.

## 7. Set up Kafka

This single-node KRaft configuration is for local development, not
production high availability. Create `docker-compose.kafka.yml`:

``` yaml
services:
  kafka:
    image: apache/kafka:4.1.0
    hostname: kafka
    ports:
      - "9092:9092"
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: PLAINTEXT://:9092,CONTROLLER://:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_NUM_PARTITIONS: 3
    volumes:
      - kafka_data:/var/lib/kafka/data

volumes:
  kafka_data:
```

Start and inspect:

``` bash
docker compose -f docker-compose.kafka.yml up -d
docker compose -f docker-compose.kafka.yml ps
docker compose -f docker-compose.kafka.yml logs -f kafka
```

Create a topic with three partitions:

``` bash
docker compose -f docker-compose.kafka.yml exec kafka \
  /opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create --topic order-events --partitions 3 --replication-factor 1
```

List topics:

``` bash
docker compose -f docker-compose.kafka.yml exec kafka \
  /opt/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --list
```

Test with the console producer:

``` bash
docker compose -f docker-compose.kafka.yml exec -it kafka \
  /opt/kafka/bin/kafka-console-producer.sh \
  --bootstrap-server localhost:9092 --topic order-events
```

Enter a JSON line and press Enter. In another terminal:

``` bash
docker compose -f docker-compose.kafka.yml exec -it kafka \
  /opt/kafka/bin/kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 --topic order-events --from-beginning
```

Stop with `docker compose -f docker-compose.kafka.yml down`. Use
`down -v` only to intentionally remove local data.

**Networking note:** `localhost:9092` works for host-run applications.
From another Docker container, `localhost` refers to that container, not
Kafka. Use an internal Docker hostname and configure separate
internal/external advertised listeners if needed.

Install the Node.js client:

``` bash
mkdir kafka-demo && cd kafka-demo
npm init -y
npm install kafkajs
npm install -D typescript tsx @types/node
mkdir src
export KAFKA_BROKERS='localhost:9092'
```

## 8. Kafka implementation

### Producer: `src/producer.ts`

``` typescript
import { Kafka } from "kafkajs";
import { randomUUID } from "node:crypto";

const brokers = (process.env.KAFKA_BROKERS ?? "localhost:9092").split(",");
const kafka = new Kafka({ clientId: "orders-api", brokers });
const producer = kafka.producer();

async function main() {
  await producer.connect();

  const event = {
    id: randomUUID(),
    type: "order.created",
    version: 1,
    occurredAt: new Date().toISOString(),
    payload: { orderId: "ord_123", customerId: "cus_456", total: 1499 }
  };

  await producer.send({
    topic: "order-events",
    acks: -1,
    messages: [{
      key: event.payload.orderId,
      value: JSON.stringify(event),
      headers: { eventType: event.type, eventId: event.id }
    }]
  });

  console.log("Published", event.id);
  await producer.disconnect();
}

main().catch(async (error) => {
  console.error(error);
  await producer.disconnect().catch(() => undefined);
  process.exitCode = 1;
});
```

`acks: -1` requests acknowledgment from all in-sync replicas. With the
single-node local setup, it does not provide high availability. The
message key helps keep events for the same order in the same partition
under the default partitioner.

### Consumer: `src/consumer.ts`

``` typescript
import { Kafka } from "kafkajs";

const brokers = (process.env.KAFKA_BROKERS ?? "localhost:9092").split(",");
const kafka = new Kafka({ clientId: "email-worker", brokers });
const consumer = kafka.consumer({ groupId: "email-workers" });

type OrderCreated = {
  id: string;
  type: "order.created";
  payload: { orderId: string; customerId: string; total: number };
};

async function processEvent(event: OrderCreated) {
  // Replace with real work. Use event.id for deduplication/idempotency.
  console.log("Processing order", event.payload.orderId);
}

async function main() {
  await consumer.connect();
  await consumer.subscribe({ topic: "order-events", fromBeginning: false });

  await consumer.run({
    autoCommit: false,
    eachMessage: async ({ topic, partition, message }) => {
      if (!message.value) return;

      const event = JSON.parse(message.value.toString()) as OrderCreated;
      await processEvent(event);

      // Commit the NEXT offset, only after successful processing.
      await consumer.commitOffsets([{
        topic,
        partition,
        offset: (BigInt(message.offset) + 1n).toString()
      }]);
    }
  });
}

async function shutdown() {
  await consumer.disconnect();
}
process.once("SIGINT", () => void shutdown());
process.once("SIGTERM", () => void shutdown());

main().catch(async (error) => {
  console.error(error);
  await consumer.disconnect().catch(() => undefined);
  process.exitCode = 1;
});
```

Add scripts:

``` json
{
  "scripts": {
    "producer": "tsx src/producer.ts",
    "consumer": "tsx src/consumer.ts"
  }
}
```

Run `npm run consumer` and then `npm run producer`. Create the topic
first with the CLI command in section 7.

**Important:** this minimal consumer illustrates explicit offset
commits; it is not a complete retry/DLQ implementation. If processing
fails, do not blindly skip/commit the event. Define bounded retries, a
terminal dead-letter topic, malformed-message handling, and replay
procedures. A crash after a side effect but before the offset commit can
cause duplicate processing, so make handlers idempotent.

### Consumer groups and scaling

-   Instances with the same group ID share partitions.
-   Different group IDs can each consume the same retained stream.
-   A group cannot gain useful partition-level parallelism beyond the
    number of partitions.
-   Ordering is guaranteed within a partition, not across the topic.
-   Changing partition count can change key-to-partition mapping for
    future records.

## 9. Reliability, retries, and DLQs

``` mermaid
flowchart TD
    M[Receive message] --> P[Process]
    P -->|Success| ACK[Acknowledge or commit]
    P -->|Transient error| R[Retry with exponential backoff and jitter]
    R --> P
    R -->|Attempt limit reached| D[(DLQ or dead-letter topic)]
    P -->|Permanent invalid message| D
    D --> ALERT[Alert and inspect]
    ALERT --> FIX[Fix code or data]
    FIX --> REPLAY[Controlled replay]
```

### Retry rules

1.  Retry transient errors; do not retry permanent validation errors
    indefinitely.
2.  Set a maximum attempt count.
3.  Use exponential backoff and jitter.
4.  Avoid retry storms and head-of-line blocking.
5.  Include message ID, attempt count, error class, and timestamps in
    failure metadata.
6.  Alert on DLQ growth and assign an owner for investigation.
7.  Make replay safe through idempotency.

### RabbitMQ versus Kafka acknowledgment

-   RabbitMQ manual `ack` confirms successful processing of a delivery.
    `nack`/reject behavior and dead-lettering depend on the call and
    queue configuration.
-   Kafka commits offsets for a partition; the committed offset is the
    next record to read. Committing before a side effect can lose work;
    committing afterward can cause duplicates after a crash.

### Idempotent consumer

For database-backed processing, apply the business update and record the
processed event ID in one database transaction:

``` text
BEGIN
  If event ID already exists in processed_events:
      do nothing
  Else:
      apply business update
      insert event ID into processed_events
COMMIT
```

Use a unique constraint on the event ID. A separate check followed by an
update can race. For external APIs, use provider idempotency keys where
supported and reconcile uncertain outcomes.

### Transactional outbox

When a database update and event publication must be consistent, write
the business record and an outbox record in the same database
transaction. A background publisher sends pending outbox records. Since
the publisher can resend after uncertain failures, downstream consumers
must deduplicate.

## 10. Scaling, monitoring, and production checklist

### Scaling consumers

-   RabbitMQ: add consumers/concurrency carefully; tune prefetch and
    watch queue depth and unacknowledged deliveries.
-   Kafka: add partitions when the key/order model allows it, then scale
    consumers up to useful partition parallelism.
-   Limit in-flight work and memory buffers.
-   Apply backpressure or rate limits when downstream systems are
    saturated.
-   Separate heavy jobs from latency-sensitive jobs.
-   Scale based on queue age/lag and processing rate, not CPU alone.

A rough worker estimate:

`workers ≈ arrival_rate_per_second × average_processing_seconds ÷ target_utilization`

Example: 200 messages/s × 0.1 seconds ÷ 0.7 utilization ≈ 29 concurrent
processing slots. Validate this with load tests, tail latency, retries,
and downstream limits.

### Monitor

**Broker:** availability, disk, publish errors/confirm latency, queue
depth, oldest-message age, unacknowledged deliveries, replication
health.

**Consumers:** throughput, latency percentiles, error rate, retry count,
DLQ depth, consumer health, Kafka lag and rebalances, CPU/memory,
downstream connection pools.

**Business:** orders stuck pending, notification delay, outbox rows
waiting, and duplicate operations prevented.

### Production checklist

-   [ ] Choose a broker based on routing, replay, throughput, ordering,
    and operational skills.
-   [ ] Define event IDs, types, versions, correlation IDs, and schema
    compatibility.
-   [ ] Define acceptable message loss/duplication and ordering scope.
-   [ ] Configure persistence and replication for the real failure
    model.
-   [ ] Use publisher confirms or suitable producer acknowledgments.
-   [ ] Acknowledge/commit only after successful processing.
-   [ ] Make consumers idempotent.
-   [ ] Add bounded retries, backoff, jitter, and a DLQ/dead-letter
    topic.
-   [ ] Use an outbox for critical database-to-event publication.
-   [ ] Set concurrency and buffer limits.
-   [ ] Monitor queue age, lag, failures, and DLQ growth.
-   [ ] Secure credentials and network access; use TLS/authentication in
    production.
-   [ ] Define retention and privacy rules.
-   [ ] Test broker restart, consumer crash, network failure,
    duplicates, and replay.
-   [ ] Document controlled DLQ replay and disaster recovery.

## 11. Interview questions

**Why use a message queue?**\
To decouple services, buffer bursts, run slow tasks asynchronously, and
scale producers and consumers independently.

**Does a queue guarantee no loss?**\
Not by itself. Producer confirms, persistence, replication,
acknowledgments/offsets, and consumer behavior all matter.

**What is the difference between RabbitMQ and Kafka?**\
RabbitMQ is often used for routed work queues; Kafka is often used for
retained, replayable event streams and partition-based scale.

**How do you prevent duplicate processing?**\
Use stable IDs, idempotent operations, and a deduplication record
protected by a transaction and unique constraint.

**What is a DLQ?**\
A destination for messages that fail the configured processing policy,
allowing investigation and controlled replay.

**How do you preserve ordering?**\
Define the key and scope. Kafka uses partition ordering; task queues may
need limited concurrency and careful retry design.

**What is the transactional outbox?**\
A pattern that stores a business update and an event record in one
database transaction, then publishes asynchronously.

**How do you scale Kafka consumers?**\
Use partitions and consumer group instances; useful parallelism is
limited by the partition count.

**What is backpressure?**\
Mechanisms that prevent incoming work from overwhelming consumers or
downstream services, such as prefetch, concurrency limits, pausing, and
rate limits.

## 12. Official references

-   RabbitMQ docs: https://www.rabbitmq.com/docs
-   RabbitMQ tutorials: https://www.rabbitmq.com/tutorials
-   RabbitMQ publisher confirms: https://www.rabbitmq.com/docs/confirms
-   RabbitMQ dead lettering: https://www.rabbitmq.com/docs/dlx
-   Apache Kafka docs: https://kafka.apache.org/documentation/
-   Kafka quickstart: https://kafka.apache.org/quickstart
-   Kafka design: https://kafka.apache.org/documentation/#design
-   KafkaJS docs: https://kafka.js.org/docs
-   amqplib docs: https://amqp-node.github.io/amqplib/

## Final takeaway

A message broker is a tool for asynchronous, decoupled systems---not a
substitute for failure handling. Start from the problem, choose RabbitMQ
for flexible task routing or Kafka for retained event streams when those
models fit, and design for duplicates, retries, observability, and
recovery from the beginning.
