# Phase 5 - Events, Messaging and Streaming

> **Weeks:** 31-36 | **Prerequisites:** Phase 4 (Data and Consistency) | **Time budget:** 90 hrs  
> **You finish this phase able to:**
> - Design event schemas and partition strategies that prevent hot partitions and preserve ordering guarantees
> - Implement exactly-once semantics end-to-end using Kafka transactions and idempotent consumers
> - Operate Kafka clusters: tune for no data loss, detect consumer lag, handle rebalances, size partitions
> - Build saga orchestration and choreography flows with failure recovery and compensation
> - Design schema evolution strategies that prevent runtime failures across service versions
> - Distinguish when to use queues vs logs vs stream processors and choose the right managed service

## Why this phase exists

Asynchronous messaging is how services stay up when their neighbours are down. It is also how teams accidentally build an untraceable, unversioned, globally coupled mess. The difference is discipline.

Synchronous RPC couples you to uptime; async messaging couples you to contract stability. A breaking schema change in Kafka takes down every consumer. A slow consumer blocks nothing but creates lag that becomes data loss when retention expires. A partition key chosen wrong creates hot spots that no amount of horizontal scaling fixes. These are not hypothetical: they are the Tuesday incidents at every company running microservices at scale.

This phase teaches you to design message flows that decouple services without creating chaos — the contracts, the error handling, the observability, the operational reality. You will learn why Kafka is not a queue, why idempotence is not exactly-once, and why event sourcing solves problems you probably do not have.

## Mental model

The core mental model for this entire phase:

**Plain English:** A queue is a to-do list where each task is consumed once and then deleted. A log is a permanent history where every reader keeps their own bookmark and can re-read from any earlier position.

**Analogy:** A hospital emergency room triage queue vs a legal court transcript. The triage queue assigns each patient to one doctor and marks them as seen — the patient does not get triaged twice. The court transcript is written once and read by the defense, prosecution, judge, jury, court reporter, and appellate reviewers, each at their own pace and position. The transcript does not vanish when the defense finishes reading it; the defense can re-read it. If a new appellate reviewer joins months later, they start from the beginning.

Where the analogy breaks: the court transcript is ordered globally; Kafka partitions are ordered only per partition, not across partitions. A triage queue is FIFO across all patients; SQS FIFO is FIFO per message group, not globally.

**In the real world:** Gmail uses a log model for your inbox: every device (phone, laptop, tablet) reads the same message history from its own position and can re-sync from scratch. Slack uses a similar model: new devices download message history. Amazon SQS processes each order-confirmation message exactly once — one worker claims it, processes it, deletes it; no other worker sees it.

**Mechanics:** Kafka is an append-only distributed commit log. Producers append records to the end of a partition. Consumers track an offset (position in the log) and read sequentially from that offset. The log is retained for a configurable time or size; consumers can rewind and replay. Consumer groups coordinate so each partition is read by one consumer in the group at a time, but multiple groups can independently read the same partition at different offsets. SQS is a queue: a message is invisible to other consumers once claimed, deleted after processing, and cannot be replayed.

**What breaks:** Choosing a queue when you need replay (SQS) means you cannot reprocess history after a bug is fixed — the messages are deleted. Choosing a log when you need exclusive processing (Kafka without consumer groups) means duplicate processing unless you implement idempotency. Treating Kafka like a queue and never replaying means you pay for a log's storage and complexity without using its strength.

This is the single most important distinction in this phase. Every decision flows from whether you need replayability and independent consumption (log) or guaranteed single processing and message deletion (queue).

## Core concepts

### Messaging vs streaming vs RPC-over-a-queue

**Messaging:** fire-and-forget, eventual delivery, single consumer per message (queue) or broadcast (pub-sub). Use cases: task distribution, command dispatch, service integration. Examples: RabbitMQ, SQS, Azure Service Bus.

**Streaming:** durable log, replayable, ordered per partition, multiple independent consumers. Use cases: event sourcing, CQRS projections, stream processing, audit trail, CDC. Examples: Kafka, Kinesis, Pulsar.

**RPC-over-a-queue:** synchronous semantics (request/reply correlation) over async transport. Anti-pattern unless mandated by legacy systems. You pay queue latency without gaining decoupling — the caller still blocks waiting for the reply.

Decision matrix:

| Need | Use |
|------|-----|
| Replay historical events | Log (Kafka) |
| Each message processed exactly once by one worker | Queue (SQS standard, RabbitMQ) |
| Ordered processing per entity (order, user) | Log with partition key |
| High throughput (>100k msg/sec per topic) | Log (Kafka, Kinesis) |
| Per-message TTL or priority routing | Queue (RabbitMQ) |
| Stream joins, windowed aggregations | Stream processor (Kafka Streams, Flink) |
| Cheapest managed option for <1k msg/sec | SQS standard |

### Kafka internals: the decisions that shape your architecture

#### Topics, partitions, and offsets

A **topic** is a logical event stream (e.g., `order.created`). A **partition** is the physical unit: an ordered, immutable sequence of records, stored as a set of segment files on disk. Each record gets a monotonically increasing **offset** within its partition. Offsets are partition-local, not global — offset 100 in partition 0 is unrelated to offset 100 in partition 1.

The partition count is set at topic creation and is hard to change (requires creating a new topic and migrating). It determines:
- **Parallelism ceiling:** a consumer group can have at most N active consumers for N partitions. 50 partitions = max 50 parallel consumers.
- **Ordering boundary:** Kafka guarantees order only within a partition, not across partitions.
- **Rebalance blast radius:** adding/removing consumers triggers partition reassignment.

Rule of thumb: start with `max(expected consumer count, expected throughput MB/sec ÷ partition throughput limit)`. Partition throughput ceiling is roughly 10-50 MB/sec depending on replication, batching, and compression. Too few partitions = underutilized consumers; too many = rebalance overhead and ZooKeeper/controller load (less of an issue post-KRaft).

Offsets are committed to the internal `__consumer_offsets` topic. A consumer tracks its position and resumes from the last committed offset after a restart or rebalance.

#### Replication, leaders, and durability

Each partition has a **leader** (handles all reads and writes) and N **replicas** (followers). The **in-sync replica set (ISR)** is the leader plus followers that are caught up (not lagging beyond `replica.lag.time.max.ms`). A follower falls out of ISR if it lags; it rejoins when caught up.

Durability contract for zero data loss:

```yaml
# Producer config
acks: all  # Wait for all ISR replicas to acknowledge
enable.idempotence: true  # Prevents duplicates on retry
retries: 2147483647  # Retry indefinitely (idempotence prevents duplication)
max.in.flight.requests.per.connection: 5  # Safe with idempotence
```

```yaml
# Topic config (set at creation or alter)
min.insync.replicas: 2  # At least 2 replicas (leader + 1 follower) must ack
replication.factor: 3  # Total replicas
```

`acks=all` + `min.insync.replicas=2` + `replication.factor=3` means: writes block until at least the leader and one follower confirm. If `min.insync.replicas=1` (just the leader), you accept data loss if the leader dies before replication. If `acks=1` (leader only), the write succeeds before followers replicate, risking loss on leader failover.

**What breaks:** `acks=1` with leader crash before follower replication → data loss. `min.insync.replicas=2` with only 2 replicas total and one broker down → writes fail (not enough ISR). Set `replication.factor` ≥ `min.insync.replicas + 1` to tolerate one broker failure.

#### Producer idempotence and transactions

**Idempotent producer** (`enable.idempotence=true`, default since Kafka 3.0): the broker deduplicates retries using sequence numbers. Guarantees exactly-once per partition per producer session. Does NOT cover multiple partitions or external state (database writes).

**Transactions** (`transactional.id=<unique-per-producer>`): atomic writes across multiple partitions and consumer offset commits. Enables exactly-once semantics (EOS) within Kafka: read, process, write outputs and commit offset in one atomic unit. Consumer must set `isolation.level=read_committed` to see only committed messages.

What EOS covers:
- Duplicate suppression across retries.
- Atomic multi-partition writes (e.g., write to `order.created` and `inventory.reserved` together).
- Atomic consume-transform-produce loop.

What EOS does NOT cover:
- External side effects: if you write to Postgres and then publish to Kafka, a crash after the DB write but before the Kafka commit creates an orphan DB row. Solution: transactional outbox pattern (see Production Patterns).
- Cross-broker failures during transaction coordinator failover can delay completion; EOS is not instant.

```java
// Transactional producer
Properties props = new Properties();
props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
props.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "order-service-producer-1");  // Unique per instance
props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
KafkaProducer<String, String> producer = new KafkaProducer<>(props);
producer.initTransactions();

try {
    producer.beginTransaction();
    producer.send(new ProducerRecord<>("order.created", orderId, orderJson));
    producer.send(new ProducerRecord<>("inventory.reserved", orderId, inventoryJson));
    producer.commitTransaction();
} catch (Exception e) {
    producer.abortTransaction();
    throw e;
}
```

#### Consumer groups, partition assignment, and rebalancing

A **consumer group** is a set of consumers that cooperate to consume a topic. Kafka assigns each partition to exactly one consumer in the group. Multiple groups can independently consume the same topic.

**Partition assignment strategies:**
- **Range:** sorts partitions and consumers, divides partitions evenly. Default. Can create unbalanced assignment across topics.
- **Round-robin:** assigns partitions in round-robin order. Balances better but causes more partition shuffling on rebalance.
- **Sticky:** minimizes partition movement on rebalance. Reduces state transfer for stateful processors.
- **Cooperative-sticky** (incremental cooperative rebalancing, default since Kafka 3.2): stops-the-world rebalance is replaced with incremental reassignment. Only partitions being moved stop processing; others continue. Dramatically reduces rebalance impact for large consumer groups.

**Static membership:** set `group.instance.id` to a stable unique identifier per consumer. On restart, the consumer rejoins with the same partitions instead of triggering a full rebalance. Useful for stateful stream processors with large local state stores.

**Rebalance triggers:**
- Consumer joins or leaves the group.
- Consumer misses heartbeat (crashes, GC pause, network partition).
- Consumer exceeds `max.poll.interval.ms` without calling `poll()` (blocked processing).

**max.poll.interval.ms:** maximum time between `poll()` calls. If a consumer does not call `poll()` within this interval, the broker assumes it is dead and triggers rebalance. Default 300 seconds. If your per-message processing can take longer (e.g., complex transformation, external API call), increase this or process asynchronously and poll more frequently.

**What breaks:** A consumer doing expensive synchronous processing per message takes 10 minutes per batch. `max.poll.interval.ms=300000` (5 min) triggers rebalance. Partition is reassigned to another consumer, which processes the same messages again. Original consumer finishes, commits offset, but it is no longer the owner — commit fails. Infinite rebalance loop.

Fix: process asynchronously and poll every few seconds, or increase `max.poll.interval.ms` to greater than worst-case batch processing time.

#### Offset commit strategies

**Auto-commit** (`enable.auto.commit=true`, `auto.commit.interval.ms=5000`): offsets are committed automatically every 5 seconds. Simple but risks message loss or duplication on crashes. If a consumer processes messages, crashes before the next auto-commit, the offset is not saved — messages are reprocessed (at-least-once). If auto-commit happens before processing completes, messages are skipped on crash (at-most-once, data loss).

**Manual commit (sync):**

```java
consumer.poll(Duration.ofMillis(100));
// process records
consumer.commitSync();  // Blocks until commit completes
```

Guarantees at-least-once: offset is committed only after processing. Crash before commit → reprocess. Slower: blocks on every commit.

**Manual commit (async):**

```java
consumer.poll(Duration.ofMillis(100));
// process records
consumer.commitAsync((offsets, exception) -> {
    if (exception != null) {
        log.error("Offset commit failed", exception);
    }
});
```

Non-blocking but risks offset commit failure going unnoticed. Use for throughput; fall back to sync commit on consumer shutdown.

**At-least-once positioning:** commit offset after processing. On crash, uncommitted messages are reprocessed. Requires idempotent consumers.

**At-most-once positioning (data loss):** commit offset before processing. On crash, processed-but-uncommitted messages are lost. Never acceptable for business data.

### Partition key design: ordering vs hot partitions

Kafka routes records to partitions by key hash: `partition = hash(key) % partition_count`. Records with the same key always land in the same partition, preserving order for that key.

**ShopKart example:** `order.created` events.

**Option 1: key by `orderId`**
- **Pros:** all events for one order are ordered (create → item-added → payment → shipped).
- **Cons:** orders are distributed across partitions; no hot partitions unless one order generates vastly more events than others (rare).
- **Verdict:** Correct for most use cases.

**Option 2: key by `customerId`**
- **Pros:** all events for one customer are ordered.
- **Cons:** high-volume customers (bulk buyers, bots, resellers) create hot partitions. One partition handles 80% of traffic; other partitions idle. Cannot scale horizontally.
- **Verdict:** Only if customer-level ordering is required and traffic is evenly distributed across customers.

**Option 3: no key (null key)**
- **Pros:** even partition distribution.
- **Cons:** no ordering guarantees. Event `order.shipped` might be consumed before `order.created` for the same order.
- **Verdict:** Only for events where order does not matter.

**Hot partition detection:** monitor `kafka.server:type=BrokerTopicMetrics,name=BytesInPerSec` per partition. If one partition's throughput is 10x the average, you have a hot partition. Fix: repartition data with a composite key (e.g., `customerId-shardId` where `shardId = customerId.hashCode() % 10`) or split high-volume entities into multiple topics.

### Consumer error handling: retries, DLQs, and poison pills

A message fails processing. You have three options: retry immediately, retry later, or give up.

**Transient errors:** network timeouts, database deadlocks, rate limits. Retry with backoff. If the consumer blocks on retry, the entire partition stops processing — one poison message blocks all subsequent messages.

**Poison errors:** malformed JSON, schema incompatibility, business rule violations. Retrying will never succeed. If the consumer keeps crashing and restarting, it reprocesses the same message infinitely.

**Blocking retry:** consume message, fail, retry in a loop before calling `poll()` again. Blocks the partition until success or max retries. Acceptable for transient errors with short retry windows (seconds). Fatal for persistent failures.

**Non-blocking retry (retry topic pattern):**

```java
@RetryableTopic(
    attempts = "4",
    backoff = @Backoff(delay = 1000, multiplier = 2.0, maxDelay = 10000),
    dltTopicSuffix = "-dlt",
    include = {RetryableException.class}
)
@KafkaListener(topics = "order.created")
public void handleOrder(OrderCreatedEvent event) {
    // throws RetryableException -> retried via retry topics
    // throws NonRetryableException -> sent to DLT immediately
}
```

Spring Kafka's `@RetryableTopic` (Boot 2.7+) auto-creates retry topics (`order.created-retry-0`, `-retry-1`, `-retry-2`) and a dead-letter topic (`order.created-dlt`). Failed messages are published to retry topics with exponential backoff delays. After max retries, the message lands in the DLT.

The partition is never blocked: the consumer commits the offset and moves on. Retry happens asynchronously via separate retry-topic consumers.

**Dead-letter queue (DLQ) / dead-letter topic (DLT):**

The final resting place for poison messages. Required fields in DLT messages:

```java
@Header(KafkaHeaders.ORIGINAL_TOPIC) String originalTopic,
@Header(KafkaHeaders.ORIGINAL_PARTITION) int originalPartition,
@Header(KafkaHeaders.ORIGINAL_OFFSET) long originalOffset,
@Header(KafkaHeaders.EXCEPTION_MESSAGE) String exceptionMessage,
@Header(KafkaHeaders.EXCEPTION_STACKTRACE) String stacktrace,
@Header("X-Retry-Count") int retryCount,
@Header("traceparent") String traceId  // OpenTelemetry trace context
```

**DLT triage runbook:**
1. Alert fires: DLT message count > threshold.
2. Engineer queries DLT: read exception message, trace ID, original offset.
3. Classify: schema bug (fix producer), business logic bug (fix consumer), data quality issue (fix upstream source).
4. Fix deployed.
5. Replay DLT: produce corrected messages back to the original topic (or a replay topic), skipping unfixable messages.

**What breaks:** A DLT nobody monitors is a data-loss bucket. Messages silently pile up; customers report missing orders weeks later. Attach a runbook, an on-call rotation, and an SLO (e.g., "P1 DLT messages triaged within 4 hours").

### Schema management: preventing runtime disasters

JSON without a schema is a contract written in invisible ink. Avro/Protobuf/JSON Schema make contracts explicit and machine-verifiable.

**Schema Registry:** central service (Confluent Schema Registry, AWS Glue Schema Registry, Apicurio) that stores schemas, enforces compatibility, and assigns schema IDs. Producers embed the schema ID in each message; consumers fetch the schema by ID and deserialize.

**Compatibility modes:**

| Mode | Allowed producer change | Allowed consumer change | Who upgrades first |
|------|-------------------------|-------------------------|--------------------|
| **BACKWARD** | Remove field, add optional field | Add field, remove optional field | Consumers first |
| **FORWARD** | Add field, remove optional field | Remove field, add optional field | Producers first |
| **FULL** | Add/remove optional field only | Add/remove optional field only | Either |
| **NONE** | Anything | Anything | Chaos |

**BACKWARD (most common):** old consumers can read new messages. New schema can remove fields or add optional fields. Upgrade consumers first, then producers. Use case: you control the consumers and can upgrade them before producers.

**FORWARD:** new consumers can read old messages. New schema can add fields or remove optional fields. Upgrade producers first, then consumers. Use case: long-lived consumers (data warehouse ingestion) that you cannot upgrade frequently.

**FULL (transitive FULL is gold standard):** both backward and forward compatible. Only optional field additions allowed. Upgrade in any order. Use case: strict schema evolution discipline.

**Enforcement:** CI job runs schema compatibility check before merging producer changes:

```bash
curl -X POST http://schema-registry:8081/compatibility/subjects/order.created-value/versions/latest \
  -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  -d '{"schema": "..."}'
# Returns {"is_compatible": true/false}
```

**Allowed changes (BACKWARD):**
- Add optional field (with default value).
- Remove required field (consumers ignore unknown fields).

**Forbidden changes (BACKWARD):**
- Add required field → old consumers crash on missing field.
- Remove optional field → breaks if old consumers reference it.
- Change field type → deserializer fails.

**What breaks:** Producer adds required field `customerId` without a default. Old consumers crash with `NullPointerException`. All 50 consumer instances restart simultaneously, trigger rebalance storm, partition lag skyrockets, alerts fire, incident declared.

### Event design: contracts that age well

**Event types:**

1. **Notification event (thin):** "something happened; go look it up."
   - Example: `{"eventType": "OrderCreated", "orderId": "12345"}`
   - Consumer calls `GET /orders/12345` to fetch current state.
   - **Coupling:** consumer depends on the API being available and the order still existing.

2. **Event-carried state transfer (fat):** "something happened; here is everything you need."
   - Example: `{"eventType": "OrderCreated", "orderId": "12345", "customerId": "C1", "items": [...], "total": 99.99}`
   - Consumer has all data in the event; no API call needed.
   - **Coupling:** payload size grows; schema compatibility becomes critical; old events might lack new fields.

3. **Event sourcing:** events are the source of truth; current state is derived by replaying events.
   - Example: `OrderCreated`, `ItemAdded`, `ItemRemoved`, `OrderPlaced`, `PaymentReceived`, `OrderShipped`.
   - Requires event store, replay infrastructure, snapshotting for performance.
   - **Use case:** auditing, temporal queries ("what was the order state at 3pm yesterday?"), complex state machines.

**Decision:** Start with event-carried state transfer for most use cases. Notification events couple you to API availability. Event sourcing is overkill unless you need audit or time-travel queries.

**Event naming:** past tense, domain language. `OrderPlaced`, not `PlaceOrder` (that is a command). `PaymentFailed`, not `PaymentFailure`.

**Required envelope fields:**

```json
{
  "id": "evt_a1b2c3d4",           // Unique event ID (UUID)
  "type": "order.created",        // Event type
  "version": "1.0",               // Schema version
  "occurredAt": "2026-09-12T10:15:30Z",  // ISO 8601 timestamp
  "traceparent": "00-abc123...",  // OpenTelemetry trace context (W3C)
  "source": "order-service",      // Producing service
  "tenantId": "tenant-42",        // Multi-tenancy key (if applicable)
  "aggregateId": "order-12345",   // Entity ID (partition key)
  "data": { /* payload */ }
}
```

**PII in events:** never put raw PII (email, phone, SSN, credit card) in events unless the topic has restricted access and encrypted at rest. Use tokenized references or exclude PII entirely.

**Event catalog:** maintain a central registry (Confluence wiki, AsyncAPI spec, or dedicated tool) listing every event type, schema, producing service, consuming services, retention policy, and owner. Treat it like an API catalog.

### Ordering: what Kafka guarantees and what it does not

**Kafka's ordering guarantee:** records with the same partition key are ordered within a partition. No global order across partitions.

**Cross-entity ordering problem:** Order A ships before Order B, but their events arrive out of order because they landed in different partitions. Consumer sees `OrderBShipped` before `OrderAShipped`. If the consumer displays a global "recent shipments" list sorted by event time, the list is wrong.

**Solutions:**
1. **Accept eventual consistency:** display "recent shipments" with a disclaimer ("updates may be delayed"). Most systems live with this.
2. **Sequence numbers:** attach a global sequence number to each event (generated by a single writer). Consumer buffers events and processes them in sequence order, detecting gaps. Complex; requires gap detection and buffering.
3. **Single partition:** one partition for the entire topic. Order is global. Throughput limited to one partition's ceiling (~10-50 MB/sec). Not horizontally scalable.

**Out-of-order arrival:** even within a partition, network delays or producer retries can reorder events if `max.in.flight.requests.per.connection > 1`. Set it to 1 for strict ordering (kills throughput) or use idempotence + sequence numbers.

**What breaks:** Consumer processes `OrderShipped` before `OrderCreated` because they landed in different partitions. Downstream service crashes on foreign key violation (order does not exist yet).

### Idempotent consumption: the dedupe window

Exactly-once delivery is impossible in distributed systems. At-least-once is achievable. Idempotent consumers make at-least-once behave like exactly-once.

**Idempotency:** processing the same message N times has the same effect as processing it once.

**Natural idempotency:** `SET inventory = 10` is idempotent; `UPDATE inventory SET qty = qty - 1` is not.

**Synthetic idempotency:** store processed event IDs in a dedupe table. Before processing, check if the event ID exists. If yes, skip. If no, process and insert the ID.

```java
@Transactional
public void handleOrderCreated(OrderCreatedEvent event) {
    if (dedupeRepository.existsById(event.id())) {
        log.info("Duplicate event {}, skipping", event.id());
        return;
    }
    orderRepository.save(new Order(event.orderId(), event.items()));
    dedupeRepository.save(new ProcessedEvent(event.id(), Instant.now()));
}
```

**Dedupe window:** how long do you keep event IDs? Forever is unbounded growth. 7 days is typical: retention period + buffer. Delete IDs older than 7 days.

**What breaks:** Dedupe window is 24 hours. A consumer crashes, falls behind by 3 days, replays from a 3-day-old offset. Events from 3 days ago are reprocessed because their IDs are no longer in the dedupe table. Duplicate processing occurs.

Fix: set dedupe window ≥ max expected consumer lag + retention period.

### Choreography vs orchestration revisited

Choreography: services react to events; no central coordinator.  
Orchestration: a central coordinator calls services in order.

**Decision test:** "Who can answer 'where is my order?'"

- **Choreography:** no single service knows. You reconstruct the state by querying multiple services or replaying events. Works for loosely coupled flows where no one needs a global view. Example: `OrderCreated` → inventory service reserves stock, payment service charges card, shipping service creates label, all independently.

- **Orchestration:** the orchestrator (saga coordinator) knows. It tracks state and can report "payment pending" or "shipment created". Required for flows with complex dependencies or user-visible status. Example: order service orchestrates "reserve inventory → charge payment → ship order" and rolls back on failures.

**When choreography fails:** 10 services each react to `OrderCreated`. One service's handler is slow. Order processing takes 5 minutes. Customer calls support: "where is my order?" Support queries order service: "created." Queries inventory: "reserved." Queries payment: "pending." No single source of truth. Debugging requires distributed tracing.

**When orchestration fails:** the orchestrator becomes a god service that knows too much about every domain. Changes to payment logic require updating the orchestrator.

**Hybrid:** choreography for independent side effects (send notification email, update search index), orchestration for critical path with dependencies (reserve → pay → ship).

### RabbitMQ: when you need a queue, not a log

**Core model:**
- **Exchange:** receives messages from producers, routes them to queues based on routing rules.
  - **Direct:** routes by exact routing key.
  - **Topic:** routes by pattern (e.g., `order.*.created` matches `order.us.created`, `order.eu.created`).
  - **Fanout:** broadcasts to all bound queues.
  - **Headers:** routes by message headers (rarely used).
- **Queue:** FIFO buffer, one consumer per message.
- **Binding:** links exchange to queue with a routing key.

**Quorum queues** (default since RabbitMQ 3.8): Raft-based replication, survives node failures. Prefer over classic mirrored queues.

**Dead-letter exchange (DLX):** messages that are rejected, nack'd, or TTL-expired are routed to the DLX. Equivalent to Kafka DLT but built-in.

**Message TTL:** per-message or per-queue expiration. Use case: "retry this message in 5 minutes" (publish to a TTL queue with DLX pointing to the retry queue).

**Message priority:** 0-255 priority levels. Higher-priority messages jump the queue. Use case: VIP customer orders processed before regular orders.

**Consumer prefetch (`basic.qos`):** limits unacknowledged messages per consumer. Prevents one fast consumer from hogging all messages and starving others.

**When RabbitMQ beats Kafka:**
- Per-message routing (route order events to different queues by region: US, EU, APAC).
- Low message volume (<10k msg/sec) where Kafka's partition overhead is overkill.
- Per-message TTL or priority (Kafka lacks these).
- Simpler ops: RabbitMQ cluster setup is easier than Kafka (no KRaft controller setup, fewer knobs).

**When Kafka beats RabbitMQ:**
- Replay: RabbitMQ deletes consumed messages; Kafka retains them.
- High throughput (>100k msg/sec): Kafka scales horizontally via partitions.
- Stream processing: Kafka Streams, ksqlDB, Flink integrations. RabbitMQ has no native stream processor.
- Multiple independent consumers: Kafka consumer groups allow N independent readers of the same topic. RabbitMQ requires N queues and fanout exchange.

### Cloud-managed brokers: the build-vs-buy decision

| Service | Type | Ordering | Retention | Replay | Throughput | Cost shape |
|---------|------|----------|-----------|--------|------------|------------|
| **SQS Standard** | Queue | None | 14 days max | No (deleted on consume) | Unlimited (throttle per msg) | $0.40 per million msgs |
| **SQS FIFO** | Queue | Per message group | 14 days max | No | 3000 msg/sec per queue (batchable to 30k) | $0.50 per million |
| **SNS** | Pub/Sub | None | Retry only | No | Unlimited | $0.50 per million msgs + delivery cost |
| **EventBridge** | Event bus | None | 24h archive | Yes (archive/replay) | 10k events/sec default | $1 per million events |
| **Kinesis Data Streams** | Log | Per shard | 1-365 days | Yes | 1 MB/sec or 1000 rec/sec per shard | $0.015/hr per shard + $0.014/GB PUT |
| **MSK (Kafka)** | Log | Per partition | Configurable | Yes | Depends on broker size | $0.50+/hr per broker (t3.small) |
| **Pub/Sub (GCP)** | Queue/Log hybrid | Per ordering key | 7 days default | Yes (seek to timestamp) | Unlimited (regional limit) | $40/TiB ingress + $8/TiB egress |
| **Service Bus (Azure)** | Queue | FIFO per session | 14 days max | Limited (message browsing) | 1000 ops/sec standard tier | $0.05/million ops (standard) |
| **Event Hubs (Azure)** | Log | Per partition | 1-90 days | Yes | 1 MB/sec per TU | $0.028/hr per TU + $0.028/GB ingress |

**Decision:**
- **SQS standard:** cheapest for async tasks, no ordering needed, low volume. Use case: image resize jobs, email dispatch.
- **SQS FIFO:** ordering per customer, low volume. Use case: financial transactions per account.
- **Kinesis/Event Hubs:** AWS/Azure native log, but Kafka (MSK/self-managed) is more feature-rich (transactions, exactly-once, rich ecosystem).
- **MSK vs self-managed Kafka:** MSK = simpler (AWS handles brokers, KRaft, patching) but costs 2-3x more and lacks some advanced configs. Self-managed = full control, cheaper at scale, operational burden.
- **Pub/Sub:** GCP-native, scales to planet-scale, but lock-in and less ecosystem tooling than Kafka.

**Managed vs self-managed threshold:** if your team lacks deep Kafka operational expertise (broker tuning, partition reassignment, KRaft troubleshooting), start with MSK/Confluent Cloud. If you have a platform team and >20 brokers, self-managed is cheaper.

### Stream processing: Kafka Streams and when to use Flink

**Kafka Streams:** Java library that runs in your application JVM. Not a separate cluster. Reads from Kafka topics, processes, writes back to Kafka.

**KStream vs KTable:**
- **KStream:** unbounded event stream. Each record is an independent event.
- **KTable:** changelog stream. Each record is an update to a key's current value. Semantically a materialized view.

**Joins:**
- **KStream-KStream:** join events that occur within a time window (e.g., click event + purchase event within 10 minutes).
- **KStream-KTable:** join event with current state (e.g., enrich order event with current product price).
- **KTable-KTable:** join two tables (e.g., customer + address).

**Windowing:** group events into time buckets (tumbling, hopping, session windows). Use case: "orders per minute", "active users in 5-minute window".

**State stores:** local RocksDB embedded in each stream instance. Backs KTables and windowed aggregations. Replicated via changelog topics (compacted Kafka topics). On restart, state is restored from the changelog.

**Standby replicas:** replicate state stores to other instances. On instance failure, standby promotes to active instantly without full restore. Reduces recovery time but doubles memory.

**Rebalance cost:** stateful streams must restore local state from changelog on rebalance. For 10 GB state and 50 MB/sec restore speed, rebalance takes 200 seconds. Mitigate: static membership, standby replicas.

**Exactly-once semantics (EOS):** Kafka Streams supports EOS (`processing.guarantee=exactly_once_v2`). Combines consumer offset commits and output topic writes into a transaction. Requires Kafka 2.5+ and read_committed isolation.

**When to use Kafka Streams:**
- Simple stateful transformations: aggregations, joins, filtering.
- Java/JVM application.
- Tight Kafka integration; no other data sources.
- Small-to-medium state (<100 GB per instance).

**When to use Flink instead:**
- Complex event processing: CEP patterns, multi-source joins, late-arrival handling.
- Large state (>100 GB), requiring distributed state backend (RocksDB on S3).
- Non-Kafka sources: JDBC, file systems, Iceberg, Hudi.
- Lower-latency requirements (Flink's pipelined execution beats Kafka Streams' microbatching).
- Need SQL interface (Flink SQL).

Flink is a separate cluster (TaskManagers, JobManager). Operational overhead is higher. Kafka Streams is simpler ops: just deploy JAR.

### CDC with Debezium: turning your database into an event stream

**Change Data Capture (CDC):** read database transaction logs (Postgres WAL, MySQL binlog) and publish each change as an event.

**Debezium:** Kafka Connect source connector that reads DB logs and writes to Kafka.

**How it works:**
1. Debezium takes initial snapshot of table (SELECT * WHERE snapshot_scn < current).
2. Connects to DB replication log.
3. Streams each INSERT/UPDATE/DELETE as a Kafka event.
4. Event schema: `{before: {...}, after: {...}, op: "c|u|d", ts_ms: ...}`.

**Outbox pattern (correct use of CDC):**
1. Service writes to DB + outbox table in the same transaction:
   ```sql
   INSERT INTO orders (id, ...) VALUES (...);
   INSERT INTO outbox (event_type, aggregate_id, payload) VALUES ('order.created', '12345', '...');
   ```
2. Debezium captures outbox table changes.
3. Kafka Connect SMT (Single Message Transform) extracts `payload` and publishes to topic.
4. Outbox rows are deleted after processing (or compacted by aggregate_id).

**Anti-pattern:** expose DB schema directly via CDC. Internal schema changes break downstream consumers. CDC is an implementation detail; events are the public contract.

**Operational gotchas:**
- **Postgres replication slot:** Debezium creates a replication slot. If Debezium stops or falls behind, the slot retains WAL segments, filling disk. Monitor `pg_replication_slots.confirmed_flush_lsn`.
- **Schema changes:** adding a column to a table creates a new Avro schema. Debezium forwards it to Schema Registry. Consumers must handle schema evolution.
- **Initial snapshot:** locks table (PostgreSQL `ACCESS SHARE` lock, doesn't block writes). For huge tables, snapshot can take hours. Run during low-traffic window.

### Backpressure and flow control

**Backpressure:** downstream cannot keep up with upstream; buffer fills; system must slow down or drop messages.

**Kafka's built-in backpressure:** consumers poll at their own pace. If a consumer is slow, partition lag grows, but the producer is unaffected. Lag is monitored; alerting fires; you scale consumers or optimize processing.

**What breaks without backpressure handling:** producer writes 10k msg/sec, consumer processes 1k msg/sec. Lag grows unbounded. After retention period (7 days), oldest messages are deleted before the consumer reads them. Data loss.

**Flow control mechanisms:**
- **Consumer scaling:** add consumers (up to partition count).
- **Batch size tuning:** increase `max.poll.records` to process larger batches per poll.
- **Autoscaling based on queue depth:** Kubernetes HPA triggers on `kafka_consumer_lag` metric. Lag >10k → scale out.
- **Circuit breaker:** if lag exceeds threshold, stop accepting new requests upstream (e.g., return 503 Service Unavailable).

## Production patterns

### Outbox → Kafka relay (transactional consistency)

**Problem:** write to DB and publish to Kafka atomically. Direct `save()` + `producer.send()` risks inconsistency: DB write succeeds, Kafka write fails → orphan DB record.

**Solution:** transactional outbox pattern.

```java
@Transactional
public void createOrder(CreateOrderRequest req) {
    Order order = new Order(req.items(), req.total());
    orderRepository.save(order);
    
    OutboxEvent event = new OutboxEvent(
        UUID.randomUUID(),
        "order.created",
        order.getId(),
        objectMapper.writeValueAsString(new OrderCreatedEvent(order))
    );
    outboxRepository.save(event);
}
```

**Relay (Debezium + Kafka Connect):**
```json
{
  "name": "outbox-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "postgres",
    "database.dbname": "orders",
    "table.include.list": "public.outbox",
    "transforms": "outbox",
    "transforms.outbox.type": "io.debezium.transforms.outbox.EventRouter",
    "transforms.outbox.table.field.event.type": "event_type",
    "transforms.outbox.table.field.event.key": "aggregate_id",
    "transforms.outbox.table.field.event.payload": "payload",
    "transforms.outbox.route.topic.replacement": "${routedByValue}"
  }
}
```

**When to use:** every service writing to DB and Kafka. This is the gold standard for dual-write atomicity.

**When NOT:** pure Kafka Streams apps (no DB), read-only services.

**Failure mode:** Debezium connector crashes. Outbox events pile up in DB. Alert fires on outbox table row count. Operator restarts connector; events are replayed.

### Idempotent consumer with dedupe table

**Pattern:**

```java
@Service
public class OrderEventConsumer {
    @Autowired ProcessedEventRepository dedupeRepo;
    @Autowired OrderService orderService;

    @KafkaListener(topics = "order.created")
    @Transactional
    public void handleOrderCreated(OrderCreatedEvent event) {
        if (dedupeRepo.existsById(event.id())) {
            return; // Already processed
        }
        orderService.createOrder(event);
        dedupeRepo.save(new ProcessedEvent(event.id(), Instant.now()));
    }
    
    @Scheduled(cron = "0 0 2 * * ?") // Daily at 2am
    public void cleanOldDedupeRecords() {
        dedupeRepo.deleteByProcessedAtBefore(Instant.now().minus(7, ChronoUnit.DAYS));
    }
}
```

**When to use:** at-least-once delivery with non-idempotent side effects (DB inserts, API calls).

**When NOT:** naturally idempotent operations (upserts, set operations).

**Failure mode:** dedupe table fills disk. Scheduled cleanup fails. Alert on table size. Operator investigates; cleanup query is slow (missing index). Add index, re-run cleanup.

### Retry topics + DLQ with Spring Kafka

```java
@Configuration
public class KafkaConfig {
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, String> kafkaListenerContainerFactory() {
        // Standard factory setup
    }
}

@Service
public class PaymentEventConsumer {
    @RetryableTopic(
        attempts = "4",
        backoff = @Backoff(delay = 2000, multiplier = 2, maxDelay = 60000),
        autoCreateTopics = "false", // Create topics via IaC
        include = {PaymentGatewayTimeoutException.class, DatabaseDeadlockException.class},
        dltStrategy = DltStrategy.FAIL_ON_ERROR
    )
    @KafkaListener(topics = "payment.requested")
    public void processPayment(PaymentRequestedEvent event) {
        paymentGateway.charge(event.amount(), event.customerId());
    }
    
    @DltHandler
    public void handleDlt(PaymentRequestedEvent event,
                          @Header(KafkaHeaders.EXCEPTION_MESSAGE) String exception,
                          @Header(KafkaHeaders.ORIGINAL_OFFSET) long offset) {
        log.error("Payment event landed in DLT: {} at offset {}, reason: {}", 
                  event.paymentId(), offset, exception);
        alertingService.triggerP2Alert("Payment DLT threshold exceeded");
    }
}
```

**When to use:** all async consumers with transient failures.

**When NOT:** fully synchronous flows (no async layer).

**Failure mode:** retry backoff too aggressive (immediate retries). Payment gateway rate-limits you. Every retry hits the rate limit. DLT fills in seconds. Reduce retry attempts, increase backoff.

### Saga orchestration over events

```java
@Service
public class OrderSagaOrchestrator {
    @Autowired KafkaTemplate<String, String> kafka;
    @Autowired SagaStateRepository sagaRepo;

    @KafkaListener(topics = "order.created")
    @Transactional
    public void onOrderCreated(OrderCreatedEvent event) {
        SagaState saga = new SagaState(event.orderId(), "INVENTORY_PENDING");
        sagaRepo.save(saga);
        kafka.send("inventory.reserve", event.orderId(), 
                   new ReserveInventoryCommand(event.items()));
    }

    @KafkaListener(topics = "inventory.reserved")
    @Transactional
    public void onInventoryReserved(InventoryReservedEvent event) {
        SagaState saga = sagaRepo.findById(event.orderId()).orElseThrow();
        saga.setState("PAYMENT_PENDING");
        sagaRepo.save(saga);
        kafka.send("payment.charge", event.orderId(), 
                   new ChargePaymentCommand(event.orderId(), event.total()));
    }

    @KafkaListener(topics = "payment.failed")
    @Transactional
    public void onPaymentFailed(PaymentFailedEvent event) {
        kafka.send("inventory.release", event.orderId(), 
                   new ReleaseInventoryCommand(event.orderId()));
        SagaState saga = sagaRepo.findById(event.orderId()).orElseThrow();
        saga.setState("FAILED");
        sagaRepo.save(saga);
    }
}
```

**When to use:** complex flows with compensation logic, user-visible status.

**When NOT:** simple fire-and-forget notifications.

**Failure mode:** orchestrator crashes mid-saga. Inventory reserved, payment not attempted. On restart, orchestrator replays from `inventory.reserved` offset, retries payment. Idempotent handlers prevent double-charging.

### Event-carried state transfer to eliminate sync calls

**Before (synchronous coupling):**

```java
// Order service calls Catalog service synchronously
@GetMapping("/orders/{id}")
public OrderDTO getOrder(@PathVariable String id) {
    Order order = orderRepo.findById(id);
    Product product = catalogClient.getProduct(order.productId()); // Sync HTTP call
    return new OrderDTO(order, product.name(), product.imageUrl());
}
```

**After (event-carried state transfer):**

```java
// Order service maintains a local product cache, updated via events
@KafkaListener(topics = "catalog.product.updated")
public void onProductUpdated(ProductUpdatedEvent event) {
    productCacheRepo.upsert(new ProductCache(event.productId(), event.name(), event.imageUrl()));
}

@GetMapping("/orders/{id}")
public OrderDTO getOrder(@PathVariable String id) {
    Order order = orderRepo.findById(id);
    ProductCache product = productCacheRepo.findById(order.productId()); // Local read
    return new OrderDTO(order, product.name(), product.imageUrl());
}
```

**When to use:** high-volume reads of slowly changing reference data (products, users, pricing).

**When NOT:** data changes frequently; cache invalidation complexity exceeds sync call simplicity.

**Failure mode:** product cache stale (event consumer lagged). Order displays old product name. Acceptable for non-critical display data. Not acceptable for pricing (use sync call for money).

### CQRS projection consumer

```java
@KafkaListener(topics = "order.created,order.shipped,order.cancelled")
@Transactional
public void projectOrderStatus(OrderEvent event) {
    OrderStatusView view = statusViewRepo.findById(event.orderId())
        .orElse(new OrderStatusView(event.orderId()));
    
    switch (event) {
        case OrderCreatedEvent e -> view.setStatus("CREATED");
        case OrderShippedEvent e -> view.setStatus("SHIPPED").setTrackingNumber(e.trackingNumber());
        case OrderCancelledEvent e -> view.setStatus("CANCELLED");
    }
    
    statusViewRepo.save(view);
}
```

**When to use:** read-heavy query models derived from multiple event streams.

**When NOT:** single-entity views with no aggregation (just query the source DB).

**Failure mode:** projection consumer lags. Read model is stale. Customer sees "order not found" for 30 seconds after creation. Alert on lag >10k. Scale consumers.

### Compacted topic as distributed lookup table

```yaml
# Topic config
cleanup.policy: compact
min.cleanable.dirty.ratio: 0.5
delete.retention.ms: 86400000  # 1 day tombstone retention
```

```java
// Producer: publish latest state per key
kafka.send("product.lookup", productId, productJson); // Upsert
kafka.send("product.lookup", productId, null);        // Delete (tombstone)

// Consumer: build local cache
KTable<String, Product> productTable = builder.table("product.lookup");
```

**When to use:** slowly changing configuration or reference data, accessed frequently.

**When NOT:** large datasets (compacted topics still retain all keys; 10M products = 10M messages).

**Failure mode:** compaction lags. Deleted keys remain visible. Consumer sees deleted product. Set `min.cleanable.dirty.ratio=0.1` for more aggressive compaction.

### Claim-check pattern for large payloads

```java
// Producer: upload payload to S3, publish reference
String s3Key = s3Client.upload(largePayload);
kafka.send("order.created", orderId, new OrderCreatedEvent(orderId, s3Key));

// Consumer: download payload from S3
@KafkaListener(topics = "order.created")
public void handleOrder(OrderCreatedEvent event) {
    byte[] payload = s3Client.download(event.s3Key());
    processOrder(payload);
}
```

**When to use:** payloads >1 MB (Kafka max message size is 1 MB default, tunable to ~10 MB but wasteful).

**When NOT:** payloads <100 KB (S3 round-trip adds latency).

**Failure mode:** S3 object deleted before consumer reads it. Consumer crashes with S3 404. Set S3 lifecycle policy: retain objects for retention period + buffer.

### Dead-letter triage and replay tool

```java
@RestController
public class DltReplayController {
    @PostMapping("/dlt/replay")
    public ReplayResult replayDlt(@RequestParam String topic, 
                                   @RequestParam long startOffset,
                                   @RequestParam long endOffset) {
        Consumer<String, String> dltConsumer = createConsumer("dlt-replay-tool");
        dltConsumer.assign(List.of(new TopicPartition(topic + "-dlt", 0)));
        dltConsumer.seek(new TopicPartition(topic + "-dlt", 0), startOffset);
        
        int replayed = 0;
        while (dltConsumer.position(new TopicPartition(topic + "-dlt", 0)) < endOffset) {
            var records = dltConsumer.poll(Duration.ofSeconds(1));
            for (var record : records) {
                // Re-publish to original topic or a replay topic
                kafka.send(topic + "-replay", record.key(), record.value());
                replayed++;
            }
        }
        return new ReplayResult(replayed);
    }
}
```

**When to use:** every production system with DLTs.

**When NOT:** you never inspect DLTs (then delete the DLT and accept data loss).

**Failure mode:** replay tool re-publishes messages to original topic. Consumers process them immediately. If the bug is not fixed, messages land in DLT again. Publish to `-replay` topic, verify fix, then cut over.

### Consumer lag alerting

```yaml
# Prometheus alert rule
- alert: KafkaConsumerLagHigh
  expr: kafka_consumer_group_lag{topic="order.created"} > 10000
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "Consumer group {{ $labels.group }} lagging >10k on {{ $labels.topic }}"
```

```java
// Expose lag via Micrometer
@Component
public class KafkaLagMetrics {
    @Scheduled(fixedRate = 30000)
    public void recordLag() {
        AdminClient admin = AdminClient.create(Map.of(
            AdminClientConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092"
        ));
        var groups = admin.listConsumerGroups().all().get();
        for (var group : groups) {
            var offsets = admin.listConsumerGroupOffsets(group.groupId()).partitionsToOffsetAndMetadata().get();
            // Calculate lag = log-end-offset - committed-offset
            // Publish to Micrometer Gauge
        }
    }
}
```

**When to use:** all production Kafka consumers.

**When NOT:** development environments (noise).

**Failure mode:** alert threshold too low (1k). Normal processing variance triggers false alarms. Tune threshold to 3σ above baseline lag.

### Schema compatibility gate in CI

```groovy
// Gradle task
task checkSchemaCompatibility {
    doLast {
        def response = new URL("http://schema-registry:8081/compatibility/subjects/order.created-value/versions/latest")
            .openConnection()
            .with {
                requestMethod = 'POST'
                doOutput = true
                setRequestProperty('Content-Type', 'application/vnd.schemaregistry.v1+json')
                outputStream.withWriter { it << """{"schema": "${file('src/main/avro/OrderCreated.avsc').text}"}""" }
                inputStream.text
            }
        def result = new groovy.json.JsonSlurper().parseText(response)
        if (!result.is_compatible) {
            throw new GradleException("Schema is not backward compatible")
        }
    }
}
build.dependsOn checkSchemaCompatibility
```

**When to use:** all services publishing Avro/Protobuf events.

**When NOT:** JSON without schema (you are already in trouble).

**Failure mode:** CI passes, engineer merges breaking schema. Forgot to run `./gradlew build`. Deploy to prod. Consumers crash. Rollback. Add pre-merge CI enforcement: block merge if task fails.

## How big tech does it

**LinkedIn:** Kafka's birthplace. Built in 2011 to replace fragmented point-to-point messaging. LinkedIn publicly describes Kafka as the "central nervous system" of their infrastructure: 7+ trillion messages per day across 100+ Kafka clusters (as of 2024 public talks). Every service integration, activity tracking event, metrics pipeline, and data replication flow runs over Kafka. Key decisions: tiered storage (hot data in broker memory, warm data on disk, cold data in HDFS/S3) reduces broker disk requirements; schema governance via centralized Schema Registry is mandatory — no producer ships without a registered schema; consumer lag dashboards are the primary production health signal alongside request latency. LinkedIn's Brooklin project replicates Kafka data across datacenters for disaster recovery and geo-distribution. The lesson: Kafka at scale requires treating it as critical infrastructure — dedicated SRE team, capacity planning quarters ahead, and automated tooling for partition reassignment and cluster balancing.

**Uber:** operates one of the largest Kafka deployments outside LinkedIn. Public engineering blog posts describe 1+ trillion messages per day (2022 figure), multi-region clusters with cross-region replication for disaster recovery, and tiered storage that moves 80% of historical data to S3 within 24 hours. Uber developed uReplicator (open-sourced) to replace Kafka MirrorMaker for more reliable cross-cluster replication with better performance under backlog. The ride-matching pipeline is event-driven: location updates from millions of drivers flow through Kafka, joined with rider requests in Flink stream processors, matched, and order events published back to Kafka for dispatch services to consume. Schema governance is centralized: teams cannot publish new event types without schema review and backward-compatibility validation. Consumer lag SLOs are strict: P1 services must process events within 10 seconds 99.9% of the time; lag >1 minute triggers paging. The takeaway: at planetary scale, Kafka is not fire-and-forget — it requires active management, custom tooling, and obsessive monitoring.

**Netflix:** the Keystone real-time data pipeline processes 2+ trillion events per day (reported 2023). Every playback event (play, pause, seek, buffer), recommendation interaction, and A/B test impression flows through Kafka into multiple downstream systems: analytics (Druid, S3/Iceberg), machine learning feature stores, real-time dashboards, and alerting. Keystone is multi-tenant: hundreds of producer teams publish to thousands of topics. Schema enforcement via Confluent Schema Registry is mandatory; compatibility checks run in CI. Netflix developed Delta (a Flink-based stream processor) to route, filter, and transform events before materialization. Consumer lag is monitored per consumer group; alerts escalate to team-specific on-call rotations. Replay is a first-class operation: when a model retraining job needs 90 days of playback history, teams submit replay jobs that read from compacted long-retention topics without impacting real-time consumers. The pattern: treat Kafka topics as immutable event logs with multi-year retention where economically viable; derive specialized views rather than mutating the source.

**DoorDash:** publicly documented their migration from synchronous request/reply order processing to event-driven choreography. The legacy system: order service called restaurant service (sync), payment service (sync), driver dispatch (sync) — total 99th percentile latency 8+ seconds, frequent cascading failures when one service degraded. The rewrite: `order.created` event published to Kafka; independent consumers in restaurant, payment, and dispatch services react asynchronously. Latency to customer confirmation dropped to <2 seconds (order confirmed after payment auth, shipping happens async). Key challenges: debugging distributed flows required investing in OpenTelemetry trace propagation through Kafka headers (`traceparent` attached to every event); handling failures required saga orchestration with compensation (if payment fails after restaurant accepted, publish `order.cancelled` to trigger restaurant cancellation); schema evolution mistakes once took down 40% of consumers overnight when a required field was added without a default — now all schemas are validated for backward compatibility in CI before merge. The lesson: event-driven reduces coupling but increases observability and schema-discipline requirements.

**Shopify:** uses Kafka to handle flash-sale traffic spikes. Public engineering blog describes a Black Friday where 10k orders/second spiked to 70k orders/second in under 60 seconds. The event pipeline: order events published to Kafka, consumed by inventory reservation, payment processing, fraud detection, shipping label creation, and notification services, each autoscaling independently based on consumer lag. Kafka absorbs the spike as a buffer; consumers scale horizontally (Kubernetes HPA watching `kafka_consumergroup_lag` metric) over 2–3 minutes. Before Kafka, synchronous order processing meant scaling all services simultaneously, which was slower and more expensive. Kafka's partitioning strategy: orders keyed by `order_id`, not `customer_id`, to avoid hot partitions from bulk buyers. Compacted topics store product catalog and pricing data; consumer services maintain local caches refreshed from these topics, eliminating sync calls to the catalog service. The pattern: Kafka as a load-leveling buffer that decouples producer spikes from consumer scaling delays.

**Zalando:** built Nakadi, an HTTP-based event streaming service on top of Kafka. Publicly described as "Kafka as a service with governance built in." Teams publish events via HTTP POST to Nakadi; Nakadi validates schemas, enforces partitioning rules, and writes to Kafka. Consumers read via HTTP streaming or Kafka consumer API. Key governance: every event type requires an owner team, schema definition, retention policy, and audience declaration (public/internal/sensitive). Nakadi rejects events that violate schema compatibility. Audit logs track every event type access for GDPR compliance. The pattern: wrap Kafka with a governed API layer to enforce standards at scale. Over 150 teams publish events; without centralized enforcement, schema chaos would be unmanageable.

**Confluent:** as the commercial Kafka vendor, publishes reference architectures. Key patterns: event-driven microservices use Kafka as the source of truth (event sourcing), CQRS projections consume events to build optimized read models, stream processing with Kafka Streams or Flink transforms and enriches events in-flight, and CDC with Debezium publishes database changes as events. Confluent Cloud's managed offering includes automatic partition balancing, tiered storage to S3, and schema validation. The reference architecture for e-commerce: orders written to DB + outbox table → Debezium publishes `order.created` → inventory service consumes and reserves stock → payment service consumes and charges → shipping service consumes and prints label. Failures trigger compensating events (`inventory.released`, `payment.refunded`). Every service maintains local projections of data it needs, updated via events, eliminating sync cross-service calls.

**Alibaba:** developed RocketMQ, an alternative to Kafka optimized for transactional workloads and message tracing. RocketMQ powers Alibaba's e-commerce platform: 1.4+ billion orders on Singles' Day (11.11 sales event, public figures). Key differences from Kafka: message filtering at the broker level (consume only messages matching a tag), transactional messages (half-send a message, commit after local transaction, rollback on failure — solves the dual-write problem without outbox tables), and message tracing built-in (every message assigned a trace ID, stored in a separate trace topic). RocketMQ is used where Kafka's log semantics are overkill and queue-like consumption patterns fit better. The lesson: Kafka is not the only answer; RocketMQ, Pulsar, and AWS EventBridge have different tradeoffs.

**Cross-cutting observation:** Every company invested in three operational pillars before scaling producer count:

1. **Schema governance:** Schema Registry + CI compatibility checks. No exceptions. Breaking a schema in production is a SEV-1 incident.
2. **Lag monitoring:** consumer lag is a first-class metric, dashboards per consumer group, alerts on thresholds, autoscaling tied to lag.
3. **Replay tooling:** purpose-built tools to replay from DLTs, replay date ranges for backfills, and dry-run replays against test environments.

The common failure mode: teams adopt Kafka for its scalability, skip governance, and hit a wall at 50+ producers when debugging cross-service flows becomes impossible and schema changes break production weekly. The discipline comes first; the scale follows.

## Best-practice checklist

### Schema and contracts
- [ ] Every event type has a versioned schema registered in Schema Registry (Avro, Protobuf, or JSON Schema)
- [ ] Schema compatibility mode is set to BACKWARD or FULL (never NONE)
- [ ] CI pipeline validates schema compatibility before merge; breaking changes block deployment
- [ ] Event schemas include required envelope fields: `id`, `type`, `version`, `occurredAt`, `traceparent`, `source`, `aggregateId`
- [ ] PII is excluded from events or tokenized; sensitive topics have restricted ACLs and encryption at rest
- [ ] Event catalog documents every event type: schema, producing service, consuming services, retention, owner, purpose

### Partition key and ordering
- [ ] Partition key chosen to distribute load evenly (avoid `customerId` if a few customers dominate traffic)
- [ ] Events requiring ordering share the same partition key (e.g., all order lifecycle events keyed by `orderId`)
- [ ] Hot partition monitoring in place: alert if one partition's throughput exceeds 2x the average
- [ ] Partition count set to `max(expected consumers, throughput ÷ 10 MB/sec)` to allow horizontal scaling

### Durability and reliability
- [ ] Producer config: `acks=all`, `enable.idempotence=true`, `retries=2147483647`
- [ ] Topic config: `min.insync.replicas=2`, `replication.factor=3` (or ≥ `min.insync.replicas + 1`)
- [ ] Producers use transactional outbox pattern for DB + Kafka dual writes (not direct `save()` + `send()`)
- [ ] Consumers commit offsets only after processing completes (manual commit sync/async, never auto-commit for critical paths)

### Consumer idempotency and error handling
- [ ] All consumers are idempotent: deduplication table, natural idempotency, or deterministic side effects
- [ ] Deduplication window ≥ max expected lag + retention period (typically 7 days)
- [ ] Retry strategy uses non-blocking retry topics with exponential backoff (not in-line retries blocking the partition)
- [ ] Dead-letter topic (DLT) configured for every consumer; includes original offset, exception, stacktrace, trace ID
- [ ] DLT has an alert, a runbook, and an on-call owner; SLO: P1 DLT messages triaged within 4 hours
- [ ] Replay tooling exists to re-publish DLT messages after fixes, with dry-run capability

### Consumer scaling and lag management
- [ ] Consumer lag monitored per consumer group; exported to Prometheus/Datadog/CloudWatch
- [ ] Lag SLO defined (e.g., 95th percentile lag <10 seconds for critical topics, <5 minutes for analytics)
- [ ] Autoscaling configured based on lag metric (Kubernetes HPA or KEDA watching `kafka_consumergroup_lag`)
- [ ] `max.poll.interval.ms` set to >worst-case batch processing time to prevent rebalance loops
- [ ] Static membership (`group.instance.id`) used for stateful consumers to minimize rebalance state transfer

### Observability and tracing
- [ ] OpenTelemetry trace context (`traceparent` header) propagated from producer to consumer to downstream calls
- [ ] Consumer processing time, offset commit latency, and rebalance frequency logged as metrics
- [ ] Distributed traces link synchronous requests → Kafka events → async processing → downstream calls
- [ ] Kafka broker metrics monitored: bytes in/out per topic, partition count, under-replicated partitions, ISR shrink/expand rate

### Operational readiness
- [ ] Retention policy set based on replay requirements and storage budget (7-30 days typical, 90+ days for audit/CQRS)
- [ ] Compacted topics use `cleanup.policy=compact`, `min.cleanable.dirty.ratio=0.1` for aggressive cleanup
- [ ] Partition reassignment runbook exists and tested (broker decommissioning, rebalancing across availability zones)
- [ ] Disaster recovery tested: cross-region replication (MirrorMaker 2, uReplicator, or managed replication), RTO/RPO documented

## Anti-patterns and war stories

### Anti-patterns

**Kafka as a database:** using Kafka's infinite retention and compacted topics as the primary data store, querying by scanning partitions. Kafka is optimized for sequential writes and reads, not random access. Lookups are O(N) scans. Joins require co-partitioning or shuffling. Use a database for queries; use Kafka for event transport.

**Kafka as RPC:** request/reply pattern with correlation IDs, where the caller publishes a request event and blocks waiting for a reply event on a separate topic. You pay Kafka's latency (10-100 ms) without gaining async decoupling — the caller still blocks. If you need synchronous semantics, use HTTP. If you need async, commit to fire-and-forget and eventual consistency.

**Topic per consumer:** creating `order.created.inventory-service`, `order.created.payment-service` to isolate consumers. Kafka's consumer groups already isolate consumers; this creates N topics for one logical event type. Schema evolution becomes N schema updates. Use one topic with multiple consumer groups.

**Giant events with base64 blobs:** embedding 5 MB PDFs or images as base64 strings in events. Bloats broker memory, slows serialization, risks hitting max message size. Use the claim-check pattern: upload blob to S3, include S3 key in the event.

**No schema registry:** JSON events with no schema, relying on "just parse it and see." Producer adds a required field; consumers crash at runtime. No compile-time safety, no compatibility validation. Use Avro/Protobuf and Schema Registry.

**Publishing inside a DB transaction:** wrapping Kafka `send()` in a JPA `@Transactional` boundary. Kafka does not participate in the DB transaction (no XA protocol). DB commits, Kafka send fails → orphan DB row. Use transactional outbox instead.

**Unbounded retries blocking the partition:** consumer catches exception, retries in a loop 1000 times before calling `poll()` again. A single poison message blocks the partition for 10 minutes. Use non-blocking retry topics.

**DLQ nobody reads:** configuring a DLT, never alerting on it, never triaging it. Messages silently accumulate; data loss discovered weeks later during an audit. A DLT without a runbook and an owner is a silent failure mode.

**Consumer count > partition count:** scaling a consumer group to 20 instances for a 10-partition topic. 10 instances are active; 10 are idle, wasting resources. Kafka assigns at most one consumer per partition within a group. Scale consumers up to partition count; if you need more, increase partitions.

**Auto-commit with async processing:** `enable.auto.commit=true` with a consumer that processes messages asynchronously (hands off to a thread pool). Offset commits every 5 seconds regardless of whether processing finished. Crash before async processing completes → messages are lost. Use manual commit after async work completes.

**"Event-driven" architecture that is a synchronous chain:** service A publishes event → service B consumes, processes, publishes event → service C consumes, processes, publishes event. If B or C are down, the chain stalls. This is synchronous request/reply with extra hops. True event-driven means independent reactions, not sequential dependencies. For sequential flows, use saga orchestration or rethink whether async is appropriate.

### War story 1: The rebalance storm

**Incident:** A payment processing consumer group had 50 instances processing 100 partitions. Each message required calling a payment gateway API (200 ms p50, 2 seconds p99). `max.poll.interval.ms` was set to the default 300 seconds (5 minutes). During a payment gateway degradation, API latency spiked to 10 seconds p50. The consumer's processing loop took 10 seconds × 500 messages per poll = 5000 seconds. The consumer did not call `poll()` again within 5 minutes. The broker assumed the consumer was dead and triggered a rebalance. All 50 consumers stopped processing; partitions were reassigned. Each consumer restored its committed offset and resumed polling. The slow processing repeated. Another 5 minutes passed without a poll; another rebalance triggered. This entered an infinite loop: constant rebalancing, zero messages actually completing.

**Detection:** Kafka broker metrics showed 10+ rebalances per minute. Consumer lag spiked to 500k messages. P1 alert fired for lag >50k. On-call engineer saw rebalance events in consumer logs: `Revoke partitions`, `Assign partitions`, repeating every 5 minutes.

**Diagnosis:** Engineer correlated rebalance timing with payment gateway latency spike (both started at the same time). Realized slow external call was blocking the poll loop. Checked `max.poll.interval.ms=300000` and batch size of 500. Math: 500 messages × 10 sec = 5000 sec > 300 sec.

**Fix (immediate):** Set `max.poll.records=10` to reduce batch size. Each poll now processed 10 messages in 100 seconds, under the 300-second limit. Rebalances stopped. Lag began dropping.

**Fix (durable):** Increased `max.poll.interval.ms=600000` (10 minutes) and restored `max.poll.records=500`. Refactored consumer to process messages asynchronously: poll, submit batch to thread pool, poll again every 30 seconds. Thread pool processes messages in parallel; consumer does not block.

**Lesson:** Never block the poll loop with long-running work. Either process synchronously and tune `max.poll.interval.ms` to >worst-case batch time, or process asynchronously and poll frequently. Monitor rebalance rate; frequent rebalances indicate poll starvation.

### War story 2: The schema massacre

**Incident:** A catalog team deployed a new version of their product service that changed the `product.updated` event schema. They added a required field `categoryId` without a default value. The change passed their local testing because their test consumer was updated in the same deployment. The schema was not registered in Schema Registry (they were using JSON without enforcement). Deployment went to production at 2 AM. Within 10 minutes, 15 downstream consumer services (inventory, search, pricing, recommendations, analytics) began crashing with `NullPointerException: categoryId cannot be null`. Each consumer instance restarted, processed the same poison message, crashed again. The DLT filled with 2 million messages overnight (backlog of 6 hours × 80k events/sec).

**Detection:** 3 AM: paging storm. Every team with a consumer on `product.updated` received alerts. Incident declared; SEV-1. PagerDuty escalated to the VP of Engineering because 15 teams were paged simultaneously.

**Diagnosis:** Incident commander checked Kafka lag: all consumer groups on `product.updated` were at 100% lag. Checked consumer logs: all crashing on deserializing `product.updated` events. Identified the new field in recent events. Checked git history: catalog team deployed a schema change 1 hour before the crash.

**Fix (immediate):** Rolled back the catalog service deployment. New events stopped flowing; old events resumed. Consumers caught up on old events. But the DLT still had 2 million poison messages.

**Fix (DLT triage):** Platform team wrote a one-off script to read the DLT, inject a default `categoryId="UNKNOWN"`, re-publish to a replay topic. Each consumer team updated their consumer to handle `categoryId="UNKNOWN"` gracefully, deployed, and consumed from the replay topic. Took 16 hours to fully drain the DLT.

**Fix (prevent recurrence):** Mandated Schema Registry for all event schemas. CI pipeline now runs `curl -X POST .../compatibility/.../versions/latest` to validate backward compatibility. Breaking changes are blocked at PR review. Catalog team's new schema was resubmitted with `categoryId` as an optional field with default `null`; consumers updated to handle `null`.

**Lesson:** Schema compatibility is not optional. JSON without a schema registry is production roulette. The cost of adding Schema Registry is 1 day of setup; the cost of breaking 15 services is 16 hours of incident response plus lost trust.

### War story 3: The replay that double-charged 40,000 customers

**Incident:** An order service team needed to backfill a new analytics projection. They wrote a replay job: read 30 days of `order.created` events, transform them, write to a new analytics topic. They pointed the replay job at the production `order.created` topic and consumed from offset 0. The replay job ran successfully and wrote 10 million events to the analytics topic. But the replay job also triggered side effects. The `order.created` consumer in the notification service sent confirmation emails for every order. The payment service, which listened to the same topic for fraud-detection logging, submitted 40,000 duplicate payment authorizations (idempotency was missing because the payment processor's API was assumed to be idempotent — it was not).

**Detection:** Customer support received 400 calls in 2 hours: "I was charged twice for an order from 3 weeks ago." Payment processor sent an alert: "unusual spike in authorization volume."

**Diagnosis:** Payment team correlated the duplicate charges with the replay job's start time. Checked the replay job config: it consumed from the production topic and processed all events, including calling the payment processor. The replay job had no filtering.

**Fix (immediate):** Killed the replay job. Payment team submitted reversal requests for all duplicate charges (took 3 days; manual CSV upload to payment processor). Notification team sent apology emails to 40,000 customers who received duplicate confirmation emails.

**Fix (prevent recurrence):** Established a replay protocol: (1) replay jobs run against a dedicated replay topic or a copy of the production topic; (2) replay jobs tag events with a `X-Replay: true` header; (3) all consumers check the header and skip side effects (emails, payment authorizations) if replaying; (4) replay jobs require approval from a platform lead.

**Lesson:** Replay is a production operation with blast radius. Never replay against consumers with side effects unless you have explicit replay-awareness (header checks, idempotency). Test replay jobs in staging with a subset of data. Dry-run capability is mandatory.

### War story 4: The tombstone that deleted a billion-dollar key

**Incident:** A pricing service maintained a compacted topic `pricing.current` with the latest price for every product (10 million products). The topic was configured with `delete.retention.ms=3600000` (1 hour). A pricing batch job published updated prices nightly. During one run, a bug caused the batch job to publish a `null` value (tombstone) for product `SKU-PREMIUM-001`, the company's flagship product. The tombstone was intended to delete the old price and would be followed immediately by a new price. But the batch job crashed mid-run after publishing the tombstone, before publishing the new price. The tombstone sat in the topic for 90 minutes (batch job restart took longer than expected due to infra issues). Kafka compaction cleaned the topic after 1 hour, deleting the tombstone and the original price record. When the batch job restarted and published the new price, it appeared as if `SKU-PREMIUM-001` had never had a price before. Downstream consumers (e-commerce frontend, recommendation engine) saw no price for the product and hid it from search results. Revenue impact: $2 million in lost sales over 6 hours before the issue was detected.

**Detection:** Business analyst noticed a 30% drop in sales for `SKU-PREMIUM-001` in the hourly report. Escalated to engineering. Pricing team checked the compacted topic: no historical price for `SKU-PREMIUM-001`, only the newly published price.

**Diagnosis:** Checked Kafka broker logs: compaction had cleaned the topic and removed the tombstone after `delete.retention.ms` expired. The original price was deleted because the tombstone was the last message for that key. Checked the batch job logs: job crashed after publishing tombstone, restarted 90 minutes later.

**Fix (immediate):** Manually published the historical price from a database backup.

**Fix (prevent recurrence):** Changed batch job logic to publish new prices first (upsert), never publish tombstones mid-batch. Increased `delete.retention.ms=86400000` (24 hours) to give more time for recovery if a tombstone is accidentally published. Added monitoring: alert if a tombstone is published for a high-value product key.

**Lesson:** Compacted topics with tombstones are dangerous. Tombstone retention is a clock; if you do not publish a replacement before it expires, the key is gone forever. Upserts are safer than delete-then-insert. For critical keys, consider never using tombstones — mark records as deleted with a flag instead.

## Projects for this phase

### Small project 1: Transactional outbox → Kafka pipeline

**Goal:** Implement dual-write atomicity for a service that writes to PostgreSQL and publishes to Kafka.

**Scope:**
- Spring Boot service with JPA entity and outbox table (schema: `id`, `event_type`, `aggregate_id`, `payload`, `created_at`).
- Service writes to entity table + outbox in the same transaction.
- Debezium PostgreSQL connector configured to capture outbox table changes.
- Kafka Connect SMT (Single Message Transform) routes outbox events to topic `<event_type>`.
- Consumer validates that zero orphan DB rows exist (rows in entity table without corresponding Kafka events).

**Architecture sketch:** Service → PostgreSQL (entity + outbox tables) ← Debezium connector → Kafka → downstream consumer.

**Acceptance criteria:**
- Service writes 1000 orders; `order.created` topic has exactly 1000 events.
- Kill the service mid-transaction (during DB write); no incomplete events published.
- Kill Debezium connector; restart after 5 minutes; all outbox events eventually published.

**Stretch goals:**
- Outbox table cleanup job: delete processed events after 1 hour.
- Schema Registry integration with Avro schemas for outbox payload.

**Time box:** 12 hours.

### Small project 2: Retry and DLT topology with replay CLI

**Goal:** Build a consumer with non-blocking retry and a CLI tool to replay DLT messages.

**Scope:**
- Kafka consumer listens to `order.payment-requested`.
- Transient failures (timeout, rate limit) → retry topic with exponential backoff (3 retries: 2s, 4s, 8s).
- Permanent failures (invalid card) → DLT immediately.
- DLT messages include original offset, exception message, stacktrace, trace ID.
- CLI tool accepts date range, reads DLT, re-publishes to `order.payment-requested-replay` topic.

**Architecture sketch:** Producer → `payment-requested` topic → consumer (with retry topics `payment-requested-retry-0/1/2`) → DLT → replay tool → `payment-requested-replay` topic.

**Acceptance criteria:**
- Inject a transient failure; message retries 3 times before landing in DLT.
- Inject a permanent failure; message lands in DLT immediately (no retries).
- Replay tool re-publishes 100 DLT messages; replayed messages appear in replay topic with original payload intact.

**Stretch goals:**
- Replay tool supports dry-run mode (logs messages without publishing).
- Metrics: retry count per message, DLT size, replay job duration.

**Time box:** 10 hours.

### Small project 3: Schema Registry compatibility gate in CI

**Goal:** Prevent breaking schema changes from merging.

**Scope:**
- Avro schemas for 3 event types: `order.created`, `order.shipped`, `inventory.reserved`.
- Gradle/Maven task checks schema compatibility against Schema Registry before build.
- CI pipeline (GitHub Actions, Jenkins) runs compatibility check; fails build if incompatible.
- Intentionally introduce a breaking change (add required field without default); verify CI fails.
- Fix the schema (make field optional); verify CI passes.

**Architecture sketch:** Developer → commit → CI → schema compatibility check (curl to Schema Registry) → fail/pass → merge gate.

**Acceptance criteria:**
- Breaking change blocks merge.
- Backward-compatible change passes.
- Schema Registry logs show compatibility check requests.

**Stretch goals:**
- Pre-commit hook runs compatibility check locally before push.
- Slack notification on schema registration or compatibility failure.

**Time box:** 6 hours.

### Small project 4: Kafka Streams enrichment job

**Goal:** Enrich `order.created` events with product details from a compacted `product.catalog` topic.

**Scope:**
- Produce 1000 product catalog entries to `product.catalog` (compacted topic).
- Produce 500 `order.created` events (each references a `productId`).
- Kafka Streams app joins `order.created` (KStream) with `product.catalog` (KTable).
- Enriched events (order + product name, price, imageUrl) written to `order.enriched` topic.

**Architecture sketch:** `product.catalog` (KTable) + `order.created` (KStream) → Kafka Streams join → `order.enriched`.

**Acceptance criteria:**
- All 500 enriched events contain product details.
- Product catalog updated mid-stream (update price); new enriched events reflect updated price.
- Kafka Streams state store backed by changelog topic; survives instance restart.

**Stretch goals:**
- Windowed join: only join orders and products updated within 10 minutes of each other.
- Expose state store as queryable REST API (interactive queries).

**Time box:** 14 hours.

### Small project 5: Consumer lag autoscaler with KEDA

**Goal:** Autoscale Kafka consumers based on lag metric.

**Scope:**
- Kubernetes deployment of a consumer service (initial replicas: 2).
- KEDA ScaledObject configured to scale based on `kafka_consumergroup_lag` metric.
- Lag threshold: scale out if lag >5000, scale in if lag <1000.
- Producer floods the topic with 50k messages; observe consumer scaling from 2 → 10 replicas.
- Producer stops; observe consumer scaling back to 2 replicas.

**Architecture sketch:** Kafka → consumer deployment (KEDA HPA) ← Prometheus (scraping lag metrics) ← KEDA scaler.

**Acceptance criteria:**
- Consumer scales out under load.
- Consumer scales in when idle.
- Scaling completes within 2 minutes of lag threshold breach.

**Stretch goals:**
- Configure min/max replicas and cooldown period.
- Alert if lag exceeds threshold for >5 minutes despite autoscaling.

**Time box:** 8 hours.

### Large project 1: ShopKart event backbone

**Goal:** Build the foundational event-driven infrastructure for ShopKart: event catalog, schemas, topics, CDC, projections, replay tooling, lag SLOs.

**Scope:**
- **Event catalog:** document 10 event types (`order.created`, `order.shipped`, `payment.received`, `inventory.reserved`, `product.updated`, `user.registered`, `cart.updated`, `pricing.changed`, `shipment.delivered`, `notification.sent`). Each entry: schema, producing service, consuming services, partition key, retention, owner.
- **Schema Registry:** register Avro schemas for all 10 event types with BACKWARD compatibility.
- **Topics:** create topics with appropriate partition counts (order: 20, inventory: 10, notifications: 5) and retention (orders: 30 days, notifications: 7 days).
- **CDC:** Debezium connector for `orders` and `products` tables; publish changes to corresponding topics.
- **Outbox pattern:** order service and product service use outbox tables for DB + Kafka dual writes.
- **Projections:** build 2 CQRS projections:
  1. Order status view: consumes `order.*` events, materializes current order status in PostgreSQL read model.
  2. Customer order history: consumes `order.*` events, builds per-customer denormalized order list.
- **Retry/DLQ:** configure retry topics and DLTs for all consumers.
- **Replay tooling:** CLI tool to replay DLT messages, replay date ranges for backfills.
- **Observability:** Prometheus metrics for lag, DLT size, rebalance rate; Grafana dashboard; alerts for lag >10k and DLT >100.
- **Lag SLOs:** define SLOs (order processing: 95th <10s, inventory: 95th <30s, notifications: 95th <60s); monitor compliance.

**Architecture sketch:** Services (order, product, inventory, payment, notification) → PostgreSQL (with outbox) ← Debezium → Kafka (20+ topics) → consumers (projections, downstream services) → Prometheus/Grafana.

**Acceptance criteria:**
- Event catalog is complete and versioned in git.
- All schemas registered; CI enforces compatibility.
- CDC captures and publishes DB changes within 2 seconds.
- CQRS projections are eventually consistent (lag <30 seconds).
- Replay tool successfully backfills 7 days of order events to a new projection.
- Lag SLO compliance >99% over a 1-week test period.

**Stretch goals:**
- Multi-region replication: replicate critical topics to a second Kafka cluster.
- Event-driven saga orchestration for order fulfillment.
- Stream processing job: windowed aggregation of orders per hour per region.

**Time box:** 60 hours.

### Large project 2: Notification platform with fan-out and deduplication

**Goal:** Build a multi-channel notification system (email, SMS, push) that consumes events, deduplicates, fan-outs to channels, and tracks delivery status.

**Scope:**
- **Event sources:** consumes events from `order.created`, `payment.received`, `shipment.delivered`.
- **Notification rules:** define rules (e.g., `order.created` → send email + push; `shipment.delivered` → send SMS).
- **Deduplication:** prevent duplicate notifications if the same event is reprocessed.
- **Fan-out:** route each notification to appropriate channels (email, SMS, push).
- **Delivery tracking:** publish `notification.sent`, `notification.delivered`, `notification.failed` events.
- **Retry/DLQ:** transient failures (email provider timeout) → retry with backoff; permanent failures (invalid phone number) → DLT.
- **Rate limiting:** SMS provider limits 100 req/sec; implement token bucket to stay under limit.
- **Templates:** use templating engine (Thymeleaf, Mustache) for email/SMS content.
- **Observability:** metrics for notifications sent/failed per channel; alert on failure rate >5%.

**Architecture sketch:** Event sources → Kafka → notification service (dedup + rule evaluation) → fan-out to email/SMS/push workers → external providers (SendGrid, Twilio, FCM) → delivery status events → Kafka.

**Acceptance criteria:**
- 1000 order events → exactly 1000 email notifications (no duplicates, even if events replayed).
- Email provider times out → notification retried 3 times before landing in DLT.
- Invalid phone number → notification lands in DLT immediately.
- Rate limiting prevents >100 SMS/sec to Twilio.
- Delivery status tracked: 98% delivered, 2% failed.

**Stretch goals:**
- User preferences: allow users to opt out of specific channels.
- A/B testing: send 10% of emails with variant subject line, track open rate.
- Template versioning: support multiple template versions; route based on user segment.

**Time box:** 50 hours.

## Interview drilldown

**Q1: How does Kafka guarantee ordering?**

**Strong answer:** Kafka guarantees ordering only within a partition, not across partitions. Records with the same partition key (determined by `hash(key) % partition_count`) always land in the same partition and are appended sequentially. A consumer reading from a partition receives records in the order they were written. However, if records for the same logical entity are split across partitions (e.g., different order events land in different partitions because no partition key was provided), there is no ordering guarantee. To preserve ordering for an entity, all events for that entity must share the same partition key (e.g., `orderId`). If you need global ordering across all records, you must use a single partition, which limits throughput to that partition's ceiling (~10-50 MB/sec).

**Follow-up:** What if I have `max.in.flight.requests.per.connection > 1` and a producer retries a failed write?

**Strong answer:** If `max.in.flight.requests.per.connection > 1` and `enable.idempotence=false`, retries can reorder messages. Example: producer sends message A (offset 100) and B (offset 101). A fails, B succeeds. A is retried and succeeds, landing at offset 102. Consumer reads B before A. To prevent this, enable idempotence (`enable.idempotence=true`), which assigns sequence numbers to messages and allows the broker to reject out-of-order retries. Or set `max.in.flight.requests.per.connection=1`, which serializes all sends but reduces throughput.

**Weak answer:** Kafka keeps everything in order because it is a log. (Fails to mention partition-local ordering, partition key hashing, or the reordering risk with retries.)

---

**Q2: Does Kafka provide exactly-once delivery?**

**Strong answer:** Kafka provides exactly-once semantics (EOS) within the Kafka ecosystem, not end-to-end. Idempotent producers (`enable.idempotence=true`) deduplicate retries per partition per session. Transactional producers (`transactional.id=<id>`) provide atomic writes across multiple partitions and atomic consume-transform-produce loops. Consumers set `isolation.level=read_committed` to see only committed messages. This guarantees exactly-once processing within Kafka: a message is consumed once, processed, outputs written, and offset committed atomically. However, if you write to an external database or call an external API, Kafka cannot enforce atomicity. A crash after the DB write but before the Kafka offset commit leads to reprocessing. To achieve end-to-end exactly-once, you need idempotent consumers: either natural idempotency (upserts) or synthetic idempotency (deduplication table).

**Follow-up:** What are the performance implications of EOS?

**Strong answer:** Transactional writes add latency (typically 10-20 ms) due to coordinator communication and two-phase commit. Throughput is also slightly lower because the producer must wait for transaction commits before starting the next batch. For most use cases, the latency cost is acceptable. For latency-sensitive paths (<5 ms p99), consider at-least-once with idempotent consumers instead.

**Weak answer:** Yes, Kafka has exactly-once. (Fails to distinguish idempotence, transactions, and external side effects. Does not mention consumer idempotency.)

---

**Q3: How do you choose a partition key?**

**Strong answer:** Choose a key that (1) distributes load evenly and (2) preserves ordering where needed. For example, keying by `orderId` distributes orders evenly across partitions and ensures all events for one order are ordered. Keying by `customerId` preserves customer-level ordering but risks hot partitions if a few customers dominate traffic (e.g., bulk buyers). Keying by `region` or `tenantId` can create imbalance if regions/tenants vary widely in volume. If no ordering is required, use `null` (or a random key) for round-robin distribution. Monitor partition throughput (bytes in/out per partition); if one partition consistently handles 5x the average, you have a hot partition. Fix: repartition with a composite key (e.g., `customerId-shardId` where `shardId = hash(customerId) % 10`) or split high-volume entities.

**Follow-up:** What if I need global ordering across all events?

**Strong answer:** Use a single partition. This guarantees global ordering but limits throughput to one partition's ceiling. Acceptable for low-volume topics (<10 MB/sec). For high volume, accept partition-local ordering or attach sequence numbers and reconstruct order downstream.

**Weak answer:** Just use the entity ID. (Fails to consider hot partitions or the tradeoff between ordering and load distribution.)

---

**Q4: What happens during a consumer rebalance?**

**Strong answer:** A rebalance occurs when a consumer joins, leaves, or is detected as dead (missed heartbeat or `max.poll.interval.ms` exceeded). The broker's group coordinator reassigns partitions to the remaining consumers. During rebalance, all consumers in the group stop processing (stop-the-world) until reassignment completes. Consumers with stateful processing (e.g., Kafka Streams state stores) must restore state from changelog topics, which can take seconds to minutes depending on state size. After rebalance, each consumer resumes from its last committed offset. Rebalances are disruptive: they increase latency, stall processing, and can trigger a rebalance storm if consumers repeatedly exceed `max.poll.interval.ms`. Mitigation: use cooperative sticky assignment (default since Kafka 3.2), which incrementally reassigns only the partitions being moved; enable static membership (`group.instance.id`) for stateful consumers to skip rebalance on restart.

**Follow-up:** How do you detect and debug frequent rebalances?

**Strong answer:** Monitor the `kafka.consumer:type=consumer-coordinator-metrics,client-id=<id>,rebalance-rate-per-hour` metric. Frequent rebalances (>10/hour) indicate instability. Check consumer logs for `Revoke` and `Assign` messages; correlate timestamps with application logs (GC pauses, long processing times). Common causes: `max.poll.interval.ms` too low, blocking I/O in poll loop, GC pauses exceeding `session.timeout.ms`, network partitions. Fix: tune `max.poll.interval.ms`, process asynchronously, reduce batch size, or optimize GC.

**Weak answer:** Rebalance happens when a consumer crashes. (Fails to mention partition reassignment, stop-the-world, or state restoration.)

---

**Q5: How do you handle a poison message?**

**Strong answer:** A poison message is one that cannot be processed successfully (malformed JSON, schema incompatibility, business rule violation). If the consumer retries indefinitely, it blocks the partition. Solution: non-blocking retry with a dead-letter topic (DLT). On transient errors (timeouts, rate limits), publish the message to a retry topic with a delay (e.g., 2s, 4s, 8s exponential backoff) and commit the original offset immediately. On permanent errors (schema mismatch, validation failure), publish directly to the DLT with the original message, exception, stacktrace, and trace ID. The DLT has an alert and a runbook: engineers triage, fix the root cause (schema bug, validation logic), and replay the corrected messages. Spring Kafka's `@RetryableTopic` automates this. Never retry in-line in the poll loop; it blocks the partition.

**Follow-up:** What if the DLT fills up and nobody notices?

**Strong answer:** That is silent data loss. The fix: (1) alert on DLT message count >threshold, (2) attach an on-call runbook, (3) define an SLO (e.g., "P1 DLT messages triaged within 4 hours"), (4) build replay tooling so fixing the bug and replaying is a standard operation.

**Weak answer:** Catch the exception and log it. (Fails to prevent partition blockage or provide a recovery path.)

---

**Q6: Design a notification system.**

**Strong answer:** Requirements: send email, SMS, push notifications for events like `order.created`, `shipment.delivered`. Constraints: handle 10k events/sec, deduplicate, support multiple channels, track delivery status. Design: (1) consume events from Kafka topics (`order.*`, `shipment.*`); (2) apply rules to determine notification type (e.g., `order.created` → email + push); (3) deduplicate using event ID (store in Redis or DB with 7-day TTL); (4) fan out to channel-specific workers (email worker, SMS worker, push worker); (5) workers call external providers (SendGrid, Twilio, FCM); (6) publish `notification.sent`, `notification.delivered`, `notification.failed` events back to Kafka for tracking. Use retry topics for transient failures (provider timeout); DLT for permanent failures (invalid email). Rate-limit calls to providers (token bucket). Scale workers independently based on lag per channel.

**Follow-up:** How do you prevent duplicate notifications if Kafka reprocesses an event?

**Strong answer:** Deduplicate on event ID. Before sending a notification, check if the event ID exists in a deduplication store (Redis with 7-day TTL, or DB table). If yes, skip. If no, send and record the ID. Even if the event is reprocessed, the second attempt finds the ID and skips.

**Weak answer:** Use a queue and send the notification. (No deduplication, no fan-out, no retry/DLT, no observability.)

---

**Q7: How do you replay events safely?**

**Strong answer:** Replaying means reprocessing historical events, either for backfills (new projection) or recovery (DLT triage). Risks: duplicate side effects (emails, payments), performance impact (replay floods consumers). Safe replay: (1) replay to a dedicated topic or environment, not production consumers; (2) tag replayed events with a header (`X-Replay: true`); (3) consumers check the header and skip side effects (emails, API calls) during replay; (4) test replay with a small date range first (dry-run); (5) monitor lag and scale consumers if needed; (6) replay during low-traffic windows to avoid resource contention. For DLT replay: fix the bug, deploy the fix, replay DLT messages to a `-replay` topic, verify success, then cut over. Never replay directly to the original topic without fixing the root cause — it just refills the DLT.

**Follow-up:** What if the replay takes days and you need to serve live traffic simultaneously?

**Strong answer:** Run replay consumers in a separate consumer group. The original consumer group continues processing live traffic at its own pace. The replay group reads from offset 0 (or a specific timestamp) and writes to a backfill table or topic. When the replay completes, cut over queries to the backfilled data. Separate consumer groups decouple replay from live traffic.

**Weak answer:** Just read from offset 0. (Fails to address duplicate side effects, performance, or isolation.)

---

**Q8: Choreography or orchestration for an order fulfillment flow?**

**Strong answer:** It depends on whether you need visibility and compensation. Choreography: each service reacts to events independently (`order.created` → inventory reserves stock, payment charges card, shipping prints label). No central coordinator; loose coupling. Pro: services are independent. Con: no single service knows order status; debugging requires distributed tracing; compensating a failed step is harder (who triggers the rollback?). Orchestration: a central orchestrator (saga coordinator) calls each step in order, tracks state, and handles failures. Pro: order status is queryable; compensation logic is centralized. Con: orchestrator becomes a god service; tight coupling. Recommendation: choreography for independent side effects (send notification, update search index), orchestration for critical paths with dependencies and user-visible status (order fulfillment, payment processing). Hybrid: orchestrate the critical path, choreograph the side effects.

**Follow-up:** How do you implement saga orchestration with Kafka?

**Strong answer:** Store saga state in a database (current step, compensation actions taken). Orchestrator consumes events from each step (e.g., `inventory.reserved`, `payment.charged`), updates saga state, and publishes the next command (e.g., `shipping.create-label`). On failure (e.g., `payment.failed`), orchestrator publishes compensating commands (`inventory.release`). Each step is a separate service listening to its command topic. The orchestrator does not call services synchronously; it publishes commands and waits for events. This is async orchestration.

**Weak answer:** Orchestration is always better because you can see the status. (Ignores coupling, god-service risk, and use-case tradeoffs.)

---

**Q9: Kafka vs RabbitMQ vs SQS for these three cases: (a) task queue, (b) audit log, (c) high-throughput stream processing.**

**Strong answer:**  
**(a) Task queue (image resize, email sending):** SQS standard. Simplest ops, cheapest for low-medium volume (<10k msg/sec), auto-scaling, no partition management. RabbitMQ if you need priorities or per-message TTL. Kafka is overkill — you are paying for log retention and replay you do not need.  
**(b) Audit log (compliance, re-processing):** Kafka. Replayable, long retention (90+ days), multiple independent consumers (analytics, compliance, fraud detection). SQS deletes messages on consumption — cannot replay. RabbitMQ has limited replay via message browsing.  
**(c) High-throughput stream processing (100k+ msg/sec, joins, aggregations):** Kafka. Horizontal scaling via partitions, Kafka Streams/Flink integration, ordered processing. RabbitMQ tops out at ~50k msg/sec per cluster. SQS has no stream processor; you would poll and process in application code, which is inefficient for stateful operations.

**Follow-up:** What about EventBridge or SNS?

**Strong answer:** EventBridge is for event routing with built-in schema registry and archive/replay, but costs 2x Kafka for high volume and has lower throughput (10k events/sec default). SNS is pub/sub for fan-out (one message to many subscribers), not a durable log. Use SNS to fan out notifications; use Kafka for event sourcing and stream processing.

**Weak answer:** Kafka for everything. (Ignores simpler, cheaper options for simple use cases.)

---

**Q10: How do you evolve an event schema without breaking consumers?**

**Strong answer:** Use backward compatibility: new schemas can remove fields or add optional fields (with defaults). This allows old consumers (unaware of new fields) to read new messages. Upgrade path: (1) register new schema in Schema Registry with compatibility mode BACKWARD; (2) deploy consumers first (they ignore unknown fields); (3) deploy producers (they emit new schema). Never add required fields without a default — old consumers crash. Never change field types — deserializers fail. Use Schema Registry's compatibility check in CI to block breaking changes at PR time. For major breaking changes (rename field, change semantics), version the event type (`order.created.v2`) and run both schemas in parallel during migration.

**Follow-up:** What if you need to rename a field?

**Strong answer:** Renaming is a breaking change. Options: (1) add the new field as optional, deprecate the old field, populate both during transition, remove old field after all consumers migrate — slow but safe. (2) Version the event (`order.created.v2`), publish both versions in parallel, migrate consumers one by one, sunset v1 after full migration. (3) Use Avro field aliases to map old field names to new ones — Avro can read old messages with the new schema.

**Weak answer:** Just change the field and deploy. (No schema registry, no compatibility validation, guaranteed production breakage.)

---

**Q11: How do you trace a request across Kafka?**

**Strong answer:** Propagate OpenTelemetry trace context in message headers. Producer: extract current trace context, serialize to W3C `traceparent` header, attach to Kafka record headers. Consumer: extract `traceparent` from headers, start a new span as a child of the producer's span. Downstream calls from the consumer inherit the same trace ID. This creates an end-to-end trace: HTTP request → producer span → Kafka message → consumer span → downstream HTTP call. Use auto-instrumentation (OpenTelemetry Java agent or Micrometer Tracing) to minimize manual code. Export traces to Jaeger, Tempo, or Datadog. Critically, every consumer must extract and propagate the trace context; if one consumer breaks the chain, downstream calls lose trace correlation.

**Follow-up:** What if the producer does not include a trace context?

**Strong answer:** The consumer starts a new root span with a new trace ID. The trace is isolated to the consumer and downstream; it does not link back to the producer. Fix: mandate trace propagation in producer libraries or intercept at serialization to inject trace context automatically.

**Weak answer:** Use correlation IDs. (Correlation IDs are custom, not standardized; they do not integrate with distributed tracing tools.)

---

**Q12: How do you scale Kafka consumers?**

**Strong answer:** Kafka scales consumers horizontally within a consumer group up to the partition count. If you have 20 partitions, you can scale to 20 consumers (one per partition). Scaling beyond partition count wastes resources (idle consumers). To scale further: (1) increase partition count (requires creating a new topic and migrating data, or using repartitioning), (2) reduce per-message processing time (optimize code, use batch processing), (3) process asynchronously (poll frequently, submit messages to a thread pool, commit after async processing completes). Autoscaling: monitor consumer lag (`kafka_consumergroup_lag` metric); scale out if lag >threshold, scale in if lag <threshold. Use Kubernetes HPA or KEDA to automate scaling. Rebalance is triggered on scale-up/down; use cooperative sticky assignment to minimize disruption. For stateful consumers (Kafka Streams), rebalance requires state restoration — standby replicas reduce recovery time.

**Follow-up:** What if lag keeps growing despite scaling?

**Strong answer:** You have hit the partition ceiling or the consumer is fundamentally too slow. Options: (1) increase partition count (repartition topic), (2) profile consumer code to find bottlenecks (slow DB query, external API call), (3) batch processing (consume 500 messages, process in parallel, commit once), (4) offload heavy work (write to a separate queue for downstream batch processing). If the partition count is already high (>100), the problem is likely consumer throughput, not parallelism.

**Weak answer:** Add more consumers. (Ignores partition count limit and fails to diagnose root cause.)

## Level signals: Senior / Staff / Principal

| Dimension | Senior | Staff | Principal |
|-----------|--------|-------|-----------|
| **Schema design** | Designs event schemas for their service; understands backward compatibility. | Defines schema governance standards across teams; reviews schemas for contract stability; builds CI tooling to enforce compatibility. | Architects event catalog and schema evolution strategy for the org; makes build-vs-buy decisions on schema registry vendors; defines migration paths for legacy systems. |
| **Partition key strategy** | Chooses partition keys that avoid hot partitions for their use case. | Identifies hot partition risks across services during design reviews; defines key cardinality guidelines; monitors partition throughput and rebalances. | Designs partitioning strategies that scale to billions of events/day; audits partition strategies org-wide; dictates repartitioning runbooks. |
| **Idempotency** | Implements deduplication tables for their consumers. | Designs idempotent consumer patterns as platform primitives; provides libraries/templates; enforces idempotency in code reviews. | Defines org-wide at-least-once vs exactly-once trade-offs; architects transactional outbox as default; measures idempotency compliance. |
| **Error handling** | Configures retry topics and DLT for their service. | Standardizes retry/DLT topology org-wide; builds replay tooling; defines DLT SLOs and triage runbooks. | Architects error-handling strategy across event-driven systems; decides when to fail fast vs retry; owns incident response playbooks. |
| **Observability** | Adds lag metrics and alerts for their consumer group. | Designs lag dashboards and SLOs for all consumers; builds autoscaling based on lag; correlates lag with business metrics. | Defines event-driven observability standards: tracing, metrics, alerts; integrates with incident management; measures MTTR for event-related incidents. |
| **Saga patterns** | Implements choreography or orchestration for one flow. | Chooses choreography vs orchestration based on failure mode analysis; designs saga state machines; reviews compensating transaction logic. | Architects saga framework for the platform; decides when to use orchestration engines (Temporal, Camunda) vs hand-rolled; defines testing strategies for sagas. |
| **Kafka operations** | Tunes consumer configs (`max.poll.interval.ms`, `fetch.min.bytes`). | Plans partition count and retention policies; manages topic lifecycle; runs partition reassignments and cluster upgrades. | Owns Kafka capacity planning; decides managed vs self-hosted; architects multi-region replication; defines disaster recovery RTO/RPO. |
| **Stream processing** | Writes simple Kafka Streams jobs (filter, map, aggregation). | Designs stateful stream processors with joins and windowing; tunes state stores and standby replicas; debugs rebalance issues. | Chooses Kafka Streams vs Flink vs custom; architects streaming platforms; defines stream processing patterns (lambda, kappa); reviews complex CEP pipelines. |
| **Incident response** | Debugs consumer lag in their service; restarts consumers; scales up. | Leads incident response for event-driven failures; diagnoses rebalance storms, schema breakages, DLT overflows; coordinates cross-team fixes. | Defines incident severity for event pipeline failures; owns post-incident reviews; drives platform improvements (circuit breakers, automated rollbacks). |

## Exit criteria

- [ ] I can explain partition-local ordering vs global ordering and choose the right partition key for entity-level ordering
- [ ] I can configure a producer for zero data loss (`acks=all`, `min.insync.replicas=2`, idempotence, retries)
- [ ] I can implement exactly-once semantics end-to-end using Kafka transactions and idempotent consumers
- [ ] I can design event schemas with backward compatibility and validate them in CI using Schema Registry
- [ ] I can implement the transactional outbox pattern with Debezium to achieve DB + Kafka dual-write atomicity
- [ ] I can configure non-blocking retry topics with exponential backoff and route poison messages to a DLT
- [ ] I can build a DLT triage runbook and a replay tool to safely reprocess failed messages
- [ ] I can tune consumer configs to prevent rebalance loops (`max.poll.interval.ms`, `max.poll.records`, static membership)
- [ ] I can monitor consumer lag, set SLOs, and configure autoscaling based on lag metrics
- [ ] I can propagate OpenTelemetry trace context through Kafka headers for distributed tracing
- [ ] I can design a saga with choreography or orchestration and implement compensating transactions
- [ ] I can distinguish when to use Kafka vs RabbitMQ vs SQS based on replay, throughput, and ordering requirements
- [ ] I can write a Kafka Streams job with KTable joins, windowing, and state stores
- [ ] I can choose partition count and retention policies based on throughput, parallelism, and replay needs
- [ ] I can implement event-carried state transfer to eliminate synchronous calls and reduce coupling
- [ ] I can use a compacted topic as a distributed lookup table with appropriate cleanup policies
- [ ] I can implement CDC with Debezium and transform outbox events into properly formed Kafka messages
- [ ] I can debug hot partitions by monitoring per-partition throughput and repartition using composite keys
- [ ] I can distinguish idempotent producer semantics, transactional semantics, and consumer idempotency
- [ ] I can explain the risks of auto-commit with async processing and choose the correct commit strategy
- [ ] I can design message retention, tombstone retention, and compaction policies for different use cases
- [ ] I can implement the claim-check pattern for large payloads and trade off S3 latency vs Kafka message size
- [ ] I can choose cooperative sticky assignment to minimize rebalance disruption in large consumer groups
- [ ] I can configure KRaft-based Kafka clusters and migrate from ZooKeeper-based deployments
- [ ] I have built a complete event backbone with schemas, topics, CDC, projections, retry/DLT, and observability

## Resources

**Books:**
- *Kafka: The Definitive Guide*, 2nd edition (Narkhede, Shapira, Palino) — the authoritative Kafka reference; covers producers, consumers, Kafka Streams, operations.
- *Designing Event-Driven Systems* (Ben Stopford, O'Reilly) — event sourcing, CQRS, Kafka as a platform; Confluent-centric but conceptually strong.
- *Enterprise Integration Patterns* (Hohpe, Woolf) — pattern catalog for messaging; predates Kafka but defines the vocabulary (message router, content filter, dead letter channel).
- *Building Event-Driven Microservices* (Bellemare, O'Reilly) — event-driven architecture patterns, stream processing, governance.

**Official documentation:**
- Apache Kafka documentation — [kafka.apache.org/documentation](https://kafka.apache.org/documentation) — producer/consumer configs, broker tuning, Kafka Streams API.
- Debezium documentation — [debezium.io/documentation](https://debezium.io/documentation) — CDC connectors, outbox pattern, schema evolution.
- Confluent Platform documentation — [docs.confluent.io](https://docs.confluent.io) — Schema Registry, Kafka Connect, ksqlDB, control center.
- Spring for Apache Kafka — [spring.io/projects/spring-kafka](https://spring.io/projects/spring-kafka) — `@KafkaListener`, `@RetryableTopic`, transactional support.

**Blogs and talks:**
- Confluent blog — *"Exactly-Once Semantics Are Possible"*, *"Kafka as a Platform"*, *"Should You Put Multiple Event Types in the Same Topic?"*
- Martin Fowler — *"What do you mean by Event-Driven?"* — clarifies event notification, event-carried state transfer, event sourcing, CQRS.
- Uber Engineering blog — *"uReplicator"*, *"Data Infrastructure at Uber"*, *"Kafka at Scale"*.
- Netflix Tech Blog — *"Keystone Real-Time Stream Processing Platform"*, *"Optimizing Kafka for Observability"*.
- DoorDash Engineering blog — *"Building Scalable Real-Time Event Processing with Kafka"*.
- LinkedIn Engineering blog — *"Kafka's origin story"*, *"Brooklin: Multi-Datacenter Replication"*, *"Kafka at LinkedIn's Scale"*.

**Tools and frameworks:**
- KEDA (Kubernetes Event-Driven Autoscaling) — [keda.sh](https://keda.sh) — autoscale consumers based on Kafka lag.
- Schema Registry UI — Confluent Schema Registry UI or Conduktor for browsing schemas.
- Kafka UI tools — Conduktor, Redpanda Console, Kafdrop, AKHQ for browsing topics, consumer groups, schemas.
- AsyncAPI — [asyncapi.com](https://asyncapi.com) — event-driven API specification format; defines events, schemas, channels.

**Courses:**
- Confluent Fundamentals for Apache Kafka — official training; covers architecture, producers, consumers, Kafka Streams.
- Udemy: "Apache Kafka Series" (Stéphane Maarek) — hands-on producer/consumer setup, Kafka Connect, Schema Registry.

**Repositories and examples:**
- Confluent examples — [github.com/confluentinc/examples](https://github.com/confluentinc/examples) — microservices, event sourcing, CQRS, Kafka Streams.
- Debezium examples — [github.com/debezium/debezium-examples](https://github.com/debezium/debezium-examples) — outbox pattern, CDC pipelines.
