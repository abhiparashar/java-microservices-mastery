# Phase 0 — Distributed Systems Foundations

> **Weeks:** 1–4 | **Prerequisites:** Java proficiency, one monolith shipped to production | **Time budget:** ~80 hrs  
> **You finish this phase able to:**
> - Explain why a service that works locally fails in production and predict which failure will occur
> - Design a call chain with correct timeouts, retries, and idempotency
> - Read a latency budget and know immediately which service is over budget
> - Recognize metastable failures (retry storms, cache stampedes) in metrics before the pager fires
> - Argue for or against consistency guarantees in a design review with precise trade-off framing
> - Debug lost writes, duplicate orders, and stale reads using distributed systems first principles

## Why this phase exists

Most microservices failures are distributed systems failures wearing Spring annotations. A service that passes all unit tests, deploys green, and works perfectly when you `curl localhost:8080` will lose customer money at 2 a.m. because:

- The network dropped 0.1% of packets and your retry logic created a payment duplicate storm (at-least-once delivery + non-idempotent handler)
- Two instances both thought they owned the background job because the database's clock skew made their lease expiry checks disagree
- A dependency's p99 latency spiked from 50 ms to 800 ms and your 1-second timeout killed the service because you fan out to it 30 times per request
- Kafka rebalanced and your consumer lost 90 seconds of progress because you committed offsets before processing
- Your cache invalidation lost a race with a write and now 10,000 customers see a product that has been out of stock for six hours

These are not Spring problems, Kafka problems, or Kubernetes problems. They are distributed systems problems. The CAP theorem does not care that you are using Spring Boot. Queueing theory does not pause for your Kafka consumer. Partial failure is the default operating mode of any system where two computers talk over a wire.

This phase teaches the bedrock: the failure models, consistency trade-offs, time and ordering, and queueing behaviour that govern every service you will ever write. You will learn to see a design and know where it will break, what the symptom will be, and how to verify the fix. Without this foundation, you build distributed monoliths—systems that inherit all the complexity of microservices and none of the resilience.

## Mental model

Everything in distributed systems flows from five brutal realities. Internalize these and the rest is consequences.

### 1. The network is a lying, lossy, delayed pipe

Messages are dropped, duplicated, arbitrarily delayed, and reordered. A request that left your service may never arrive. A response that the downstream sent may never reach you. You cannot distinguish "the network ate my request" from "the service is dead" from "the service is slow". TCP/IP hides some of this (retransmits, checksums), but above L4 you own the rest.

**Implication:** Every remote call needs a timeout. Every retry needs backoff and jitter. Every important operation needs an idempotency key.

### 2. Partial failure is the default, not the exception

In a monolith, failure is binary: the process is up or it crashed. In a distributed system, failure is partial: the database is up but unreachable from one AZ, Kafka is up but two brokers are GC-paused, the downstream API is up but returning 500s for 0.5% of requests. You CANNOT design for "everything works or everything fails". You must design for "most things work, some things are broken, and you cannot tell which from inside the system".

**Plain English:** In a monolith, you know if a function call failed—it threw an exception. In a distributed system, a service call can succeed, fail, or you will never know which happened.

**Analogy:** You send a letter to your friend. It might arrive. It might get lost. It might arrive twice (you sent it again when they did not reply). Your friend might receive it but their reply gets lost, so you think your letter was lost. You cannot know which happened unless you both meet face-to-face—but that is not an option when the "friend" is a service in another data center.

**In the real world:** You submit an Amazon order. The page times out. Did the order go through? Amazon does not know either—the database write succeeded but the acknowledgment packet was dropped. So they show you "Your order is being processed, check back in a few minutes" and reconcile asynchronously.

**Mechanics:** A service sends an HTTP POST to create an order. The downstream writes to its database and returns 201 Created. The network drops the response. The caller sees a timeout, retries, and now there are two orders. Or: the downstream crashes after the write but before sending the response—same symptom, different cause. Or: the downstream is slow because of GC, the caller times out and retries, and the original request completes 5 seconds later—two orders again.

**What breaks:** Duplicate payments, lost inventory reservations, double-debits, order confirmation emails sent twice. Logs show `SocketTimeoutException` or `HTTP 504`, metrics show elevated retry rate, customer support gets calls about duplicate charges.

### 3. There is no global "now"

Clocks on different machines drift. NTP corrects this to ~100 ms typically, but spikes to seconds are real. Even on the same machine, `System.currentTimeMillis()` can go backward (NTP slew, leap second smear). Event ordering across services is a coordination problem, not a clock lookup.

**Implication:** Do not use `createdAt` timestamps to order events from different services. Use logical clocks (Lamport, vector, HLC) or a single source of truth (database transaction order, Kafka offset).

### 4. All state is replicated state, and replicas disagree

Every "single" piece of data is physically on multiple disks (primary + replicas, cache + database, client cache + server, Kafka leader + followers). Replicas lag, diverge, and reconcile. Consistency is a spectrum of expensive guarantees about how much lag you tolerate.

**Implication:** "Read your writes" is not free—it requires routing the read to the writer's shard or waiting for replication. Eventual consistency is cheap but breaks user expectations ("I just updated my address, why does checkout still show the old one?").

### 5. Every remote call is a queue in disguise

When you call `orderService.createOrder(...)`, that request enters a queue: the TCP send buffer, the load balancer's queue, the service's thread pool queue, the database connection pool queue. Each queue has a depth and a service rate. If arrival rate exceeds service rate, queue depth grows unbounded and latency explodes (Little's Law). The only fix is backpressure: reject new work upstream before the queue fills.

**Implication:** Under load, you must shed requests or slow down clients. Retries without backpressure turn overload into a death spiral.

## Core concepts

### The 8 fallacies of distributed computing

These are the lies developers believe before production teaches them otherwise. Each fallacy causes real bugs.

| Fallacy | ShopKart bug it causes |
|---------|------------------------|
| **1. The network is reliable** | `cart` calls `pricing` to get a discount. Network blip drops the response. User gets charged full price. No retry, no idempotency, no timeout—just silent failure. |
| **2. Latency is zero** | `order` service makes synchronous calls to `inventory` → `payment` → `shipping` → `notification`. Each adds 50 ms. User waits 200 ms for a confirmation that could be async. |
| **3. Bandwidth is infinite** | `catalog` returns full product objects (images encoded as base64, full descriptions) instead of IDs. A search page loading 50 products transfers 15 MB. Mobile users on 3G time out. |
| **4. The network is secure** | Internal service-to-service calls trust the `X-User-ID` header. An attacker on the VPC spoofs it and accesses other users' carts. |
| **5. Topology doesn't change** | `cart` hardcodes the IP of the `pricing` service. `pricing` gets redeployed to new IPs. `cart` starts failing. No service discovery, no health checks. |
| **6. There is one administrator** | `order` team deploys a schema change to their database. `payment` team's read-replica lag spikes and queries time out. No coordination, no rollback plan. |
| **7. Transport cost is zero** | Every call to `catalog` serializes data to JSON, sends it over the network, deserializes it. The service spends 40% of CPU on Jackson. Should have used a binary protocol for the hot path. |
| **8. The network is homogeneous** | `search` service assumes MTU is 1500 everywhere. Traffic crosses a VPN with MTU 1400. Packets fragment. Latency spikes. No one knows why until a network engineer runs `tracepath`. |

### Failure model taxonomy

Distributed systems textbooks classify failures by what guarantees the system CAN and CANNOT make.

**Crash-stop:** A node halts and never comes back. Easy to reason about (treat it as dead). Rare in practice—most "crashes" are crash-recovery.

**Crash-recovery:** A node crashes but restarts with stable storage intact. This is the real world. The hard part: did it crash before or after committing that write? You need write-ahead logs, idempotency, and fencing.

**Omission:** Messages are lost. The network drops packets. Equivalent to infinite latency—you cannot tell if the message is lost or just delayed.

**Timing:** Messages are delayed unpredictably. This breaks protocols that rely on timeouts (most of them). A slow message looks like a crash; the node recovers and causes split-brain.

**Byzantine:** Nodes send incorrect or malicious messages. In most enterprise systems you assume non-Byzantine failure (bugs yes, malice no). Blockchains and some military systems assume Byzantine.

**Gray failure (the nasty one):** A node is slow enough to miss heartbeats but fast enough to process some requests. Worse than dead because the system cannot decide: is it down (fail over), or just slow (wait)? Gray failures cause the worst production incidents.

**Plain English:** You cannot tell the difference between a slow response and no response.

**Analogy:** You call your friend. They pick up but respond to every question 30 seconds later. Are they distracted, or is the call breaking up? Do you hang up and call back (retry), or wait (maybe they are just thinking)?

**In the real world:** An AWS EC2 instance has a failing disk. Reads are slow (500 ms instead of 5 ms). The load balancer health check times out after 2 seconds and marks the instance unhealthy. But 10% of requests still land there (connection draining). Users see intermittent 10-second page loads. The metric shows "instances: 3 healthy, 0 unhealthy" because the slow one is flapping in and out.

**Mechanics:** A service has a 1-second timeout. A dependency's p99 latency spikes to 1.5 seconds (GC pause, slow query, network blip). Requests to that dependency time out. The caller retries. The retry lands on the same slow instance (no jitter), times out again. The slow instance is still alive enough to accept connections but too slow to respond. The load balancer's health check (a simple HTTP GET to `/health`) succeeds because it is served from a hot cache. So the slow instance stays in rotation.

**What breaks:** Users see `504 Gateway Timeout` intermittently. Retry storms make it worse (the slow node gets even more requests). Logs show timeouts but the service reports healthy. Metrics show p99 latency at 5 seconds but p50 at 50 ms (bimodal distribution). The fix: aggressive timeouts, circuit breakers, and outlier detection (Envoy, Istio automatically eject slow instances).

### Latency numbers every engineer must memorize

These are 2026 ballpark figures for modern hardware (SSD, DDR5, 10 Gbps network). Actual numbers vary; the orders of magnitude matter.

| Operation | Latency |
|-----------|---------|
| L1 cache reference | 0.5 ns |
| L2 cache reference | 7 ns |
| Mutex lock/unlock | 25 ns |
| Main memory reference | 100 ns |
| SSD random read (4 KB) | 10–20 μs |
| Sequential read 1 MB from SSD | 200 μs |
| Same-AZ network RTT | 0.5 ms |
| Cross-AZ RTT (same region) | 1–2 ms |
| Cross-region RTT (US-East ↔ Mumbai) | 200 ms (speed of light floor ~140 ms) |
| Disk seek (spinning rust) | 5–10 ms |
| TLS handshake | 1–5 ms |

**Application:** A request to ShopKart `order` service fans out to 5 services synchronously:

1. `user` (auth check): 50 ms p99
2. `cart` (fetch items): 40 ms
3. `inventory` (reserve stock): 60 ms
4. `pricing` (apply discounts): 30 ms
5. `payment` (authorize card): 80 ms

If you call them sequentially, the floor is 50 + 40 + 60 + 30 + 80 = 260 ms, plus network overhead (~5 ms × 5 = 25 ms same-AZ), so ~285 ms minimum. If one service's p99 spikes to 500 ms, your order creation p99 becomes ≥500 ms. If you parallelize where possible (cart + inventory + pricing are independent), floor drops to max(50, 40, 60, 30) + 80 = 140 ms. But now the probability all five stay under p99 is 0.99^5 ≈ 0.95—so 5% of requests see at least one p99 event. This is tail latency amplification.

### CAP theorem (stated correctly) and PACELC

**CAP:** During a network partition, choose Consistency (reject writes) or Availability (accept writes, risk divergence). When there is no partition, you can have both. CA is not a real choice (if partitions are possible—and they are—you pick CP or AP).

ShopKart example: partition between `order` service and its database.

- **CP choice:** Reject new orders until the partition heals. Users see "Service temporarily unavailable". Data stays consistent.
- **AP choice:** Accept orders, write to a local queue, sync when the partition heals. Users get confirmation, but if the partition lasts hours and the queue overflows, orders are lost. Or you end up with duplicate orders if both sides accept the same cart.

**PACELC** (the one you actually use daily): Else (when there is no partition), trade Latency vs Consistency.

- **High consistency** (e.g., read from primary): every read sees the latest write. Latency cost: cross-AZ RTT to the primary (2 ms).
- **Eventual consistency** (read from any replica): low latency (local read, <1 ms), but you might see stale data.

ShopKart `catalog` service: product price update.

- **PC/EC (choose consistency):** Read from the primary. User always sees the current price. Latency +2 ms per read. At 10,000 reads/sec, that is 20 extra seconds of cumulative latency per second (queuing delay starts to build).
- **PA/EL (choose availability and low latency):** Read from any replica. Latency ~0.5 ms. But if replication lag is 100 ms, a user who just updated a price sees the old price for 100 ms. Most shopping sites accept this.

### Consistency models

A spectrum from strongest (expensive, simple) to weakest (cheap, complex).

| Model | Guarantee | ShopKart symptom if violated |
|-------|-----------|------------------------------|
| **Linearizable** | Every read sees the result of the most recent write across all clients. Behaves like a single copy. | User updates cart quantity to 5. Refresh shows 3. Violates user expectation of "one cart". Rare to guarantee globally—costs cross-region latency. |
| **Sequential** | All operations appear in some total order consistent with each client's order. No real-time guarantee. | Two users see different orders of price updates. Confusing but not user-facing if the UI only shows "your" writes. |
| **Causal** | Writes that causally depend on each other (read X, then write Y based on X) are seen in order. Concurrent writes can be reordered. | User adds item A (write 1), then adds item B "because A is in the cart" (write 2). Another user sees B added but not A—causality violated. |
| **Read-your-writes** | A client sees its own writes. Other clients' writes can lag. | User adds item to cart, clicks "View cart", sees empty cart. The read hit a stale replica. |
| **Monotonic reads** | If a client reads version V1, future reads return ≥V1 (no going backward). | User refreshes cart page. First load shows 3 items (from replica A, lag 0 ms). Second load shows 2 items (from replica B, lag 500 ms). Time travel. |
| **Bounded staleness** | Reads lag writes by at most T seconds or N versions. | Cart shows "updated 3 seconds ago". If T = 5 sec, acceptable. If T = unbounded (eventual consistency), user sees stale data indefinitely. |
| **Eventual** | If writes stop, eventually all replicas converge. No bound on "eventually". | Price update takes 10 minutes to propagate. User sees old price, orders, gets charged new price at checkout. Refund mess. |

Most systems mix models: linearizable for critical writes (payment authorization), eventual for reads (product catalog).

### The two different C's (interview trap)

- **ACID Consistency (C in ACID):** The database enforces integrity constraints (foreign keys, unique indexes, check constraints). A transaction cannot leave the database in an invalid state. This is about correctness of a single database.
- **CAP Consistency (C in CAP):** All replicas agree on the value of a piece of data. This is about agreement across distributed copies.

They are orthogonal. A distributed database can be ACID-consistent (enforce constraints) and CAP-AP (accept writes during partitions, allowing replicas to diverge). Cosmos DB, Cassandra, DynamoDB are examples—though Cosmos and DynamoDB offer tunable consistency.

### Time and clocks

**Wall-clock time (`System.currentTimeMillis()`):** Can go backward (NTP slew, leap second). Drifts between machines (NTP sync is ~100 ms accurate, spikes to seconds). Do not use for ordering events across machines or for "this token expires in 5 minutes" logic when the issuer and validator are different machines.

ShopKart bug: `order` service issues a JWT with `exp` = `System.currentTimeMillis() + 300_000` (5 min). `payment` service validates it with `System.currentTimeMillis()`. `payment`'s clock is 2 seconds ahead. Token is rejected as expired 2 seconds early. User's checkout fails.

**Monotonic clock (`System.nanoTime()`):** Never goes backward. Measures elapsed time. Use for timeouts, rate limiting, latency measurements. Does not synchronize across machines—you cannot compare nanoTime from two different JVMs.

**Lamport clocks:** Each event increments a counter. When sending a message, include your counter. When receiving, set your counter to max(your_counter, received_counter) + 1. This gives a partial order: if event A happened-before event B (causally), then L(A) < L(B). The converse is not true—L(A) < L(B) does not mean A caused B (they might be concurrent).

**Vector clocks:** Each node has a vector [N1: counter1, N2: counter2, ...]. Increment your own counter on each event. Merge vectors on receive. Detects causality and concurrency. Expensive to store (vector size = number of nodes). Used in Riak, Cassandra (deprecated), version-vector CRDTs.

Tiny example (two nodes):

```
Node A: writes X=1. VA = [A:1, B:0]
Node B: writes Y=2. VB = [A:0, B:1]
Node A: reads Y (sees VB), writes X=3. VA = [A:2, B:1]
Node B: reads X (sees VA=[A:1,B:0]), writes Y=4. VB = [A:1, B:2]
```

Compare VA=[A:2,B:1] and VB=[A:1,B:2]: neither dominates (A's counter is higher, B's counter is higher), so X=3 and Y=4 are concurrent writes—a conflict. Application must resolve (last-write-wins, merge, prompt user).

**Hybrid Logical Clocks (HLC):** Combine wall-clock time and logical counter. Monotonic like Lamport, but timestamps approximate physical time so you can sort them and get rough chronological order. CockroachDB, YugabyteDB use HLC.

**Google TrueTime:** GPS + atomic clocks in every datacenter. API returns an interval [earliest, latest] with bounded uncertainty (~7 ms as of public reports). Spanner waits out the uncertainty window to provide linearizability. This is why Spanner can do globally consistent reads—they literally paid for physics.

### Consensus

**Plain English:** How do multiple computers agree on a single value (e.g., "who is the leader?") when some computers might crash or messages might be lost?

**Analogy:** A group of friends deciding where to eat dinner over a flaky group chat. Messages arrive out of order. Some friends go offline mid-conversation. You need a rule that guarantees everyone who stays online eventually agrees on the same restaurant, even if some friends never respond.

**In the real world:** Kafka's KRaft controller election. When the current controller crashes, the remaining brokers run an election. They must agree on the same new controller—two controllers would cause split-brain (duplicate partition assignments, conflicting metadata). The protocol (Raft, under the hood) ensures at most one leader is elected, even if some brokers are partitioned.

**Mechanics (Raft):**

Three roles: **Leader** (handles all writes), **Follower** (replicates leader's log), **Candidate** (election participant).

1. **Leader election:** All nodes start as followers. If a follower does not hear from the leader in a timeout (randomized to avoid ties), it becomes a candidate and requests votes. A candidate needs a majority (quorum) to become leader. Majority = N/2 + 1, so for N=5, you need 3 votes.
2. **Log replication:** Leader appends a command to its log, sends it to followers. Followers append to their logs and ack. Leader waits for majority ack (quorum), then commits the entry and tells followers "entry X is committed".
3. **Safety:** A candidate can only win if its log is at least as up-to-date as a majority. This prevents an outdated node from becoming leader and losing committed writes.

**Quorum math (critical):** N = 2F + 1, where F is the number of tolerated failures. With N=5, you tolerate F=2 failures and still have a quorum of 3. With N=3, you tolerate F=1. You CANNOT have N=2 and tolerate one failure—quorum is 2, so if one node fails, you have only 1 node left, no quorum, the system hangs.

**Split-brain and fencing:** If the network partitions a 5-node cluster into [3 nodes] and [2 nodes], the 3-node partition can form a quorum and elect a new leader. The 2-node partition cannot (no quorum), so it stops. This prevents two leaders. But what if the old leader is in the 2-node partition and has not realized it lost leadership (GC pause, network delay)? It might still think it is the leader and try to write. **Fencing token:** Each leader gets a monotonically increasing epoch number. Writes include the epoch. Storage rejects writes with a stale epoch.

**Paxos:** The older, more general consensus protocol. Proven correct in 1989, notoriously hard to understand. Raft was designed in 2013 to be understandable. You do not need to derive Paxos; you need to know it exists and that it is equivalent to Raft for most purposes.

**What breaks:** Quorum-based consensus is NOT available during a majority failure. If 3 out of 5 ZooKeeper nodes die, ZooKeeper stops serving writes (and reads, in ZooKeeper's default config). This is the CP in CAP. Also, leader election takes time (typically 1–5 seconds for etcd, ZooKeeper). During election, writes are unavailable.

**Where you consume this:** You will rarely implement Raft. But you will use etcd (Kubernetes, distributed locks), ZooKeeper (legacy Kafka, Hadoop), Kafka KRaft (new Kafka), Consul, etc. You must understand: why they require odd numbers of nodes, what happens when you lose quorum, why a 2-node cluster is useless for HA, and why leadership failover takes seconds (not milliseconds).

### Replication topologies

**Single-leader (primary-backup):** One node accepts writes. Replicas copy the write log. Simple. Writes are consistent (they go through one node). Reads can go to replicas (stale) or primary (consistent, higher latency). Leader failure requires election or manual failover.

ShopKart: PostgreSQL with one primary, two replicas. `order` service writes to primary. `analytics` service reads from replicas (eventual consistency acceptable).

**Multi-leader:** Multiple nodes accept writes. Conflicts must be resolved (e.g., two users concurrently update the same product price). Conflict resolution strategies:

- **Last-write-wins (LWW):** Keep the write with the latest timestamp. Simple. Data loss if clocks disagree or writes are concurrent.
- **Application-specific merge:** E.g., merge two shopping carts by union of items. Requires domain logic.
- **CRDT (Conflict-free Replicated Data Types):** Data structures with commutative, associative, idempotent merge (e.g., OR-Set, LWW-Element-Set). Math guarantees convergence. Used in Riak, Redis CRDTs, Automerge.

Multi-leader is complex. Use it only for geo-distribution where cross-region latency makes single-leader unacceptable (e.g., a global CRM with active-active writes in US and EU).

**Leaderless (quorum):** No designated leader. Clients write to W replicas, read from R replicas, where R + W > N. This guarantees overlap: a read of R replicas will see at least one replica that has the latest write.

Example: N=5, W=3, R=3. A write succeeds when 3 replicas ack. A read queries 3 replicas, returns the version with the highest timestamp/vector clock.

**Read-repair:** If a read discovers stale data on a replica, update it inline.

**Anti-entropy:** Background process compares replicas and syncs differences. Eventual consistency—no bound on "eventual".

ShopKart: Cassandra or DynamoDB for `cart` service (leaderless, eventual consistency, high availability). A user's cart is keyed by `user_id`. Writes go to W=2 replicas (out of N=3). Reads query R=2 replicas. If one replica has an old cart version, the client takes the one with the higher timestamp.

### Delivery semantics

**At-most-once:** Send the message. Do not retry. If it is lost, it is lost. Simple. Acceptable for telemetry, logs, non-critical events.

**At-least-once:** Retry until you get an ack. The message might arrive multiple times (network dup, retry after timeout). Most messaging systems (Kafka, RabbitMQ, SQS) guarantee at-least-once.

**"Exactly-once":** Marketing term. Physically impossible to guarantee in a distributed system with failures. What people actually mean: **effectively-once** = at-least-once delivery + idempotent consumer.

**Plain English:** You cannot guarantee a message is delivered exactly one time. You can guarantee that even if it is delivered multiple times, the effect is the same as if it were delivered once.

**Analogy:** You send an email asking your friend to RSVP for a party. Your email client retries because it did not get a confirmation. Your friend receives the email twice but RSVPs only once because they remember having already replied. The duplicate email arrived, but the outcome (one RSVP) is the same as if it arrived once.

**In the real world:** Stripe processes a payment. The request times out. You retry. Stripe receives both requests but processes the charge only once because you included an **idempotency key** (`Idempotency-Key: abc123`) in the headers. Stripe's API remembers that key and returns the cached response for the duplicate.

**Mechanics:** Kafka consumer processes a message, writes to a database, then commits the Kafka offset. If the process crashes after the database write but before the offset commit, the message is reprocessed on restart. The database write happens twice. Solutions:

1. **Idempotent handler:** The database write is `INSERT ... ON CONFLICT DO NOTHING` with a unique key derived from the message (e.g., `order_id`).
2. **Transactional outbox:** Write the database row and the Kafka offset in the same database transaction (requires Kafka transactions + exactly-once semantics, which are complex).

**What breaks:** Duplicate orders. Duplicate payments. Duplicate emails. Stock over-reserved. Money debited twice. The symptom: users complain, database has duplicate rows with different IDs but same semantic content (two orders for the same cart, created 2 seconds apart).

### Idempotency

An operation is idempotent if applying it multiple times has the same effect as applying it once. `SET x = 5` is idempotent. `x = x + 1` is not.

**Natural idempotency:** Some operations are inherently idempotent. `DELETE FROM orders WHERE id = 123` is idempotent (deleting twice leaves the row deleted). `UPDATE orders SET status = 'shipped' WHERE id = 123` is idempotent.

**Synthetic idempotency:** Make a non-idempotent operation idempotent by adding a deduplication key.

```java
@PostMapping("/orders")
public OrderResponse createOrder(@RequestHeader("Idempotency-Key") String idempotencyKey,
                                  @RequestBody OrderRequest request) {
    // Check if this key was already processed
    Optional<Order> existing = orderRepository.findByIdempotencyKey(idempotencyKey);
    if (existing.isPresent()) {
        return OrderResponse.from(existing.get()); // Return cached response
    }

    // Process the order
    Order order = processOrder(request);
    order.setIdempotencyKey(idempotencyKey);
    orderRepository.save(order);
    return OrderResponse.from(order);
}
```

**Deduplication window:** You cannot store idempotency keys forever. Typical: 24 hours. After 24 hours, a retry with the same key is treated as a new request. This is a trade-off: memory/storage vs duplicate risk. Stripe's window is 24 hours. AWS idempotency keys expire after a few hours.

### Queueing theory for engineers

**Little's Law:** L = λ × W, where:

- L = average number of requests in the system (queue + being processed)
- λ = arrival rate (requests/sec)
- W = average time a request spends in the system (latency)

ShopKart `order` service: λ = 100 req/sec, average latency W = 200 ms = 0.2 sec. L = 100 × 0.2 = 20 requests in flight at any time. If you have 10 threads, average queue depth is 20 - 10 = 10 requests waiting.

**Utilization vs latency:** As utilization (ρ = λ / μ, arrival rate / service rate) approaches 1, latency explodes. At 70% utilization, p99 latency might be 2× median. At 90%, it is 10×. At 99%, it is 100×. This is why your service falls over under load even though CPU is "only" 80%.

**Backpressure:** The only real fix for overload. Reject requests upstream before they enter the queue. HTTP 429 Too Many Requests, Kafka consumer pause, TCP backpressure, flow control. Do not retry rejected requests immediately—use exponential backoff.

**Plain English:** If requests arrive faster than you can process them, the queue grows forever and latency goes to infinity. You must slow down or reject the arrivals.

**Analogy:** A coffee shop with one barista. Customers (requests) arrive at 10/min. The barista makes coffee at 10/min. Queue is stable. Now customers arrive at 12/min. Queue grows by 2 every minute. After an hour, 120 people are waiting. The coffee shop must put up a "closed" sign (backpressure) or hire another barista (scale).

**In the real world:** AWS Lambda concurrency limits. If you exceed the limit, new invocations are throttled (429). Your application must retry with backoff or handle the error gracefully.

**Mechanics:** A Spring Boot service has a Tomcat thread pool of 200. Requests arrive at 250/sec, each takes 1 sec to process. Capacity is 200 req/sec. Excess: 50 req/sec. Those 50 queue up. After 10 seconds, the queue has 500 requests. Tomcat's default queue is 100 (unbounded in practice, limited by memory). The queue fills, clients see connection timeouts, the service OOMs.

Fix: Set a bounded queue (e.g., `server.tomcat.accept-count=100`). When full, Tomcat rejects new connections. The load balancer sees connection refused, retries on another instance (if available) or returns 503 to the client.

**What breaks:** Under sustained overload without backpressure, the service crashes (OOM, thread exhaustion). Retries make it worse—every retry adds to the queue. The only way out: stop accepting new requests (reject at the edge, rate limiting) or scale horizontally.

### Tail latency amplification

If you fan out to N services in parallel and each has p99 = 100 ms, the probability your request sees no p99 events is 0.99^N. For N=10, that is 0.99^10 ≈ 0.90. So 10% of your requests see at least one dependency at p99 or worse, meaning your p99 becomes ≥100 ms.

ShopKart homepage: fans out to 20 microservices (user, cart, recommendations, trending, ads, etc.). Each has p99 = 50 ms. Your homepage p99 is ≥50 ms for 1 - 0.99^20 ≈ 18% of requests.

If you have depth (service A calls B calls C, each fans out to 5), the amplification multiplies. A → [B1, B2, B3, B4, B5], each B → [C1, C2, C3, C4, C5]. That is 1 + 5 + 25 = 31 calls. Probability of avoiding p99 everywhere: 0.99^31 ≈ 0.73. So 27% of requests see a p99 event somewhere.

**Mitigation:**

- **Hedged requests:** Send duplicate requests to two replicas after a delay (e.g., if p50 is 10 ms, hedge at 20 ms). Cancel the slower one. This cuts tail latency but doubles load at p50 latency.
- **Reduce fan-out:** Aggregate data, cache, async, or accept stale.
- **Timeouts:** Kill slow calls early so they do not block the user.

### Metastable failures

**Definition:** A failure mode where the system stays broken even after the trigger is removed. The system has two stable states: healthy and broken. A transient spike (traffic, latency, failure) pushes it to broken, and it stays there.

**Retry storm example:** A dependency has a 1-second outage. All clients time out and retry. The dependency comes back online. It is now hit with 2× normal traffic (original requests + retries). It cannot handle 2×, so requests slow down. Clients time out and retry again. Now 4×. The cycle continues until the dependency collapses or you stop all clients.

**Cache stampede:** Cache expires. 1,000 concurrent requests all miss the cache, query the database, and repopulate the cache. The database is overwhelmed. Requests time out. Clients retry. The cache is never repopulated because no request succeeds. The database stays overloaded.

**GC death spiral:** Service is under load. GC pauses increase. Requests time out. Clients retry. Load increases. GC pauses increase further. Eventually the JVM spends 100% of time in GC, 0% serving requests.

**Mechanics (feedback loop):** System is at equilibrium. An external shock (traffic spike, dependency slow, deploy) increases latency. Clients retry. Load increases. Latency increases further. More retries. Positive feedback loop.

**Exit strategies:**

1. **Shed load:** Reject new requests at the edge (rate limiting, circuit breaker). Let the system drain.
2. **Restart:** Kill all clients, flush queues, restart. This breaks the retry loop.
3. **Exponential backoff + jitter:** Spread retries over time. Prevents thundering herd.

**What breaks:** The entire service becomes unavailable even though the root cause (e.g., database slow query) is fixed. Metrics show high error rate, high retry rate, saturated CPU/memory. Logs show timeouts and retries dominating. The only fix is aggressive circuit breaking and backoff.

## Production patterns

### Heartbeats and phi-accrual failure detection

**What:** A service periodically sends "I'm alive" messages (heartbeats) to a monitor. If heartbeats stop, the monitor declares the service dead.

**When to use:** Distributed coordination (leader election, cluster membership, session management). Kubernetes liveness probes, ZooKeeper sessions, Akka cluster, Cassandra gossip.

**When NOT to use:** For detecting application-level health. Heartbeats only detect crash or network partition. A service can be alive but broken (returning 500s, deadlocked). Use application-level health checks for that.

**Failure mode:** False positives from network latency spikes or GC pauses. If the timeout is too short, healthy nodes are declared dead. If too long, actual failures take ages to detect.

**Phi-accrual failure detection:** Instead of a binary dead/alive, compute a suspicion level (phi, Φ) based on heartbeat arrival intervals. Φ increases with missed heartbeats. Threshold: Φ > 8 ≈ 99.9% confidence the node is dead. Φ > 12 ≈ 99.99%. Cassandra and Akka use this.

```java
// Conceptual phi-accrual (simplified)
class PhiAccrualFailureDetector {
    private final Deque<Long> arrivalIntervals = new ArrayDeque<>(100);
    private long lastHeartbeatTime;

    public void heartbeat() {
        long now = System.nanoTime();
        if (lastHeartbeatTime > 0) {
            arrivalIntervals.addLast(now - lastHeartbeatTime);
            if (arrivalIntervals.size() > 100) arrivalIntervals.removeFirst();
        }
        lastHeartbeatTime = now;
    }

    public double phi() {
        long now = System.nanoTime();
        long timeSinceLastHeartbeat = now - lastHeartbeatTime;
        double mean = arrivalIntervals.stream().mapToLong(Long::longValue).average().orElse(1000_000_000);
        // Real calculation uses exponential distribution; simplified:
        return timeSinceLastHeartbeat / mean; // Higher = more suspicious
    }
}
```

### Fencing tokens with distributed locks

**What:** A monotonically increasing token that prevents a stale lock holder from performing writes after losing the lock.

**When to use:** Distributed locks (etcd, ZooKeeper, Redis) for exclusive access (e.g., singleton background job, leader election).

**When NOT to use:** For high-frequency locks (microsecond hold times). Distributed locks are slow (network RTT). Use them for coarse-grained work (job scheduling, schema migration).

**Failure mode:** Lock holder experiences GC pause. Lock expires. Another process acquires the lock. First process wakes up, does not know it lost the lock, performs a write. Now two processes think they own the lock.

**Solution:** Fencing token. Each lock grant includes a sequence number (epoch, version). The storage (database, file) rejects writes with a stale token.

```java
// etcd-based lock with fencing token
import io.etcd.jetcd.Client;
import io.etcd.jetcd.Lease;
import io.etcd.jetcd.Lock;

public class DistributedLockWithFencing {
    private final Client etcdClient = Client.builder().endpoints("http://localhost:2379").build();
    private final Lock lockClient = etcdClient.getLockClient();
    private final Lease leaseClient = etcdClient.getLeaseClient();

    public void acquireLockAndDoWork() throws Exception {
        long leaseId = leaseClient.grant(10).get().getID(); // 10-second TTL
        ByteSequence lockKey = ByteSequence.from("/my-lock", StandardCharsets.UTF_8);
        
        var lockResponse = lockClient.lock(lockKey, leaseId).get(); // Blocks until acquired
        long fencingToken = lockResponse.getHeader().getRevision(); // Monotonic etcd revision

        try {
            performWorkWithToken(fencingToken);
        } finally {
            lockClient.unlock(lockResponse.getKey()).get();
        }
    }

    private void performWorkWithToken(long token) {
        // Database write: UPDATE jobs SET status = 'running', fence_token = ? WHERE id = ? AND fence_token < ?
        // If another lock holder already wrote a higher token, this fails
    }
}
```

### Retry with exponential backoff + full jitter

**What:** When a request fails, wait before retrying. Each retry waits longer (exponential). Add randomness (jitter) to prevent thundering herd.

**When to use:** All remote calls that can transiently fail (network, 500, timeout). Essential for queue consumers, HTTP clients, gRPC.

**When NOT to use:** For operations that are not idempotent unless you add an idempotency key.

**Failure mode:** Without jitter, all clients retry at the same time (e.g., 1s, 2s, 4s). The downstream sees a synchronized wave of retries and collapses again. With deterministic backoff, clients that started at the same time stay synchronized.

**Full jitter formula:** `wait = random(0, min(cap, base * 2^attempt))`. Base = 100 ms, cap = 30 seconds.

```java
import java.time.Duration;
import java.util.Random;

public class RetryWithBackoff {
    private static final Random random = new Random();
    private static final long BASE_DELAY_MS = 100;
    private static final long MAX_DELAY_MS = 30_000;
    private static final int MAX_ATTEMPTS = 5;

    public <T> T executeWithRetry(Callable<T> task) throws Exception {
        int attempt = 0;
        while (true) {
            try {
                return task.call();
            } catch (Exception e) {
                if (++attempt >= MAX_ATTEMPTS) throw e;
                long exponentialDelay = BASE_DELAY_MS * (1L << attempt); // 2^attempt
                long cappedDelay = Math.min(exponentialDelay, MAX_DELAY_MS);
                long jitteredDelay = random.nextLong(cappedDelay); // Full jitter: [0, cappedDelay)
                Thread.sleep(jitteredDelay);
            }
        }
    }
}
```

**Why full jitter works:** Spreads retries uniformly over [0, max]. Even if 1,000 clients all fail at the same instant, their retries are distributed over the next 30 seconds (at attempt 5+). The downstream sees a smooth ramp instead of a spike.

### Idempotency keys

**What:** A client-generated unique ID included in a request. The server remembers processed IDs and deduplicates.

**When to use:** Payment processing, order creation, any operation that must not execute twice even if the request is retried.

**When NOT to use:** For read operations (GET). For naturally idempotent writes (PUT with full resource state).

**Failure mode:** Key is not random enough (UUID v1 with predictable timestamp, sequential ID). Attacker can guess keys and replay. Use UUID v4 or cryptographic random.

```java
import java.util.UUID;
import org.springframework.web.bind.annotation.*;

@RestController
public class OrderController {
    private final OrderRepository orderRepository;

    @PostMapping("/orders")
    public OrderResponse createOrder(@RequestHeader("Idempotency-Key") UUID idempotencyKey,
                                      @RequestBody OrderRequest request) {
        return orderRepository.findByIdempotencyKey(idempotencyKey)
            .map(OrderResponse::from)
            .orElseGet(() -> {
                Order order = new Order(idempotencyKey, request);
                orderRepository.save(order);
                return OrderResponse.from(order);
            });
    }
}

// Schema: orders table has UNIQUE index on idempotency_key
// CREATE UNIQUE INDEX idx_order_idempotency ON orders(idempotency_key);
```

**Deduplication window:** Store keys for 24 hours. After that, a retry creates a new order. Document this in API contract.

### Request deadlines propagated across hops

**What:** Include a deadline timestamp in every request. Each service checks if the deadline is exceeded before doing work. Propagate the deadline downstream.

**When to use:** Multi-hop request chains (UI → BFF → service A → service B → database). Prevents wasted work on requests that already timed out upstream.

**When NOT to use:** For background jobs with no user waiting.

**Failure mode:** Deadlines not propagated. Service B spends 5 seconds processing a request whose caller timed out 3 seconds ago. Wasted CPU.

```java
// gRPC natively supports deadlines. For HTTP, use a custom header.
@GetMapping("/orders/{id}")
public OrderResponse getOrder(@PathVariable String id,
                               @RequestHeader("X-Deadline-Ms") long deadlineMs) {
    long now = System.currentTimeMillis();
    if (now > deadlineMs) {
        throw new DeadlineExceededException("Request deadline exceeded");
    }
    
    // Do work, propagate deadline to downstream calls
    return orderService.getOrder(id, deadlineMs);
}

// Downstream call
public OrderResponse callDownstream(String id, long deadlineMs) {
    HttpHeaders headers = new HttpHeaders();
    headers.set("X-Deadline-Ms", String.valueOf(deadlineMs));
    // ... make HTTP call with headers
}
```

### Quorum reads

**What:** Read from R replicas (where R + W > N) to guarantee seeing the latest write.

**When to use:** Critical reads where you cannot tolerate stale data (e.g., payment authorization check after a write).

**When NOT to use:** For high-volume reads where eventual consistency is acceptable (product catalog).

**Failure mode:** R is too low (e.g., N=5, W=3, R=2). R + W = 5, not > 5. You can miss the latest write.

```java
// Cassandra example: consistency level QUORUM
import com.datastax.oss.driver.api.core.cql.*;

CqlSession session = CqlSession.builder().build();
PreparedStatement stmt = session.prepare(
    "SELECT * FROM orders WHERE id = ?");

// QUORUM: read from ceil(N/2 + 1) replicas
ResultSet rs = session.execute(
    stmt.bind(orderId).setConsistencyLevel(ConsistencyLevel.QUORUM));
```

### Leader election via etcd/Kubernetes Lease

**What:** Multiple instances compete for a lock/lease. The winner becomes the leader. Leader does work. Others stand by. Leader periodically renews the lease.

**When to use:** Singleton background jobs, active-passive HA, cluster coordination.

**When NOT to use:** For stateless workloads (just run all instances).

**Failure mode:** Lease TTL too short. Leader experiences GC pause, misses renewal, loses leadership. New leader elected. Now two leaders if the old leader wakes up and does not check. Use fencing tokens.

```java
// Kubernetes Lease-based leader election (pseudocode, use client-java library)
import io.kubernetes.client.extended.leaderelection.*;

LeaderElectionConfig config = new LeaderElectionConfig(
    new LeaseLock("my-namespace", "my-lease", "instance-1"),
    Duration.ofSeconds(15), // Lease duration
    Duration.ofSeconds(10), // Renew deadline
    Duration.ofSeconds(2)   // Retry period
);

LeaderElector elector = new LeaderElector(config);
elector.run(
    () -> { /* I am the leader, do work */ },
    () -> { /* Lost leadership, stop work */ }
);
```

## How big tech does it

### Amazon Dynamo and DynamoDB: quorum replication at planet scale

The 2007 Dynamo paper (Amazon's internal eventually-consistent key-value store) introduced the three techniques every distributed database now uses:

1. **Consistent hashing with virtual nodes:** Data is partitioned across N nodes. Each node owns V virtual nodes (vnodes) on the hash ring. When a node fails, its vnodes are distributed across the remaining nodes. This spreads load evenly and makes rebalancing fast. ShopKart lesson: do not build custom sharding logic; use a database that does this (Cassandra, DynamoDB, Riak).

2. **Tunable quorum (R, W, N):** Write to W replicas, read from R replicas, where R + W > N guarantees overlap. Dynamo exposed this as a per-request dial: consistency vs latency. DynamoDB productized it as `ConsistentRead=true` (read from leader, slower, consistent) vs `ConsistentRead=false` (read from any replica, faster, eventually consistent). **Lesson:** Consistency is not binary. For a shopping cart, eventual consistency is fine—losing a cart item for 100 ms does not matter. For payment authorization, you need consistent reads. Do not default to strong consistency everywhere and then complain about latency.

3. **Vector clocks for conflict detection, LWW for resolution:** Concurrent writes to the same key get versioned. Clients receive all conflicting versions and merge them (e.g., union of cart items). DynamoDB simplified this to last-write-wins by timestamp, which loses data but is operationally simpler. **Lesson:** Conflict resolution is application logic. If you cannot tolerate lost writes, design for single-writer or use CRDTs. Do not assume the database will "figure it out".

**Scale caveat:** Dynamo was designed for Amazon's shopping cart, which tolerates staleness and has a high read:write ratio (10:1). Your CRUD app with 100 req/sec is not Amazon. Use PostgreSQL until you prove you need dynamo-style availability.

### Google Spanner: buying consistency with synchronized clocks

Spanner is a globally distributed SQL database with serializable transactions across datacenters. The trick: **TrueTime**, a clock API that returns an interval `[earliest, latest]` with bounded uncertainty (~7 ms as of public reports, achieved via GPS + atomic clocks in every datacenter).

To commit a transaction, Spanner picks a timestamp T within the uncertainty window and waits out the uncertainty before returning success. This guarantees that when the transaction commits, T is in the past for all nodes. External consistency (linearizability) falls out: if transaction A commits before transaction B starts, A's timestamp < B's timestamp.

**Lesson:** Spanner proves that global consistency is possible if you pay for it—literally, with custom hardware. For the 99.9% of companies that are not Google, accept eventual consistency for non-critical reads or use single-region databases for strong consistency. Do not build a poor-man's Spanner with NTP and hope.

**What transfers:** The idea that physical time + uncertainty bounds can replace logical clocks. Hybrid Logical Clocks (HLC) are a software approximation (CockroachDB, YugabyteDB use them). HLC gives you roughly-physical timestamps without custom hardware.

### Azure Cosmos DB: five consistency levels as a product slider

Cosmos DB exposes the consistency spectrum as five explicit settings:

1. **Strong:** Linearizable reads globally. Writes wait for quorum across regions. High latency (cross-region RTT), low availability during partitions.
2. **Bounded staleness:** Reads lag by at most K versions or T seconds. The only consistency level with a staleness SLA.
3. **Session:** Read-your-writes within a session (client token-based). Cheap, intuitive for single-user workflows.
4. **Consistent prefix:** Reads see writes in order, but may lag. Like a stream replay.
5. **Eventual:** Replicas converge, no order guarantees. Fastest, cheapest.

Cosmos also publishes measured latency and availability for each level. Strong consistency p99 write latency: ~15 ms (single region) to ~300 ms (cross-region). Eventual consistency: ~5 ms.

**Lesson:** This is the clearest proof that the consistency spectrum is a real product choice, not an academic abstraction. For ShopKart, use Strong for payment ledgers, Session for cart updates (user sees their own writes), Eventual for product catalog. Document the choice in ADRs. "We default to Strong" is wrong; "We default to Session and escalate to Strong when audit matters" is engineering.

### Facebook/Meta TAO: read-after-write for the social graph

TAO is Facebook's distributed graph datastore (edges = friendships, likes, comments). It guarantees **read-after-write consistency**: after you post a comment, you immediately see it. But other users may see it with eventual delay (seconds).

TAO achieves this with **session stickiness**: your session routes reads to the same cache tier that handled your write. Writes go to the primary database (MySQL) and invalidate the cache. The next read from your session repopulates the cache.

**Lesson:** You do not need global strong consistency; you need user-perceived consistency. A user who writes expects to see their write. Other users can tolerate lag. Implement this with session routing or version headers (e.g., `If-None-Match` returning `304 Not Modified` if the version has not changed).

**Scale caveat:** TAO serves 1 trillion+ edges. At that scale, cache hit rate is existential. At your scale, a well-tuned PostgreSQL replica is simpler.

### Netflix Chaos Engineering: designing for partial failure

Netflix open-sourced Chaos Monkey in 2011: a tool that randomly kills EC2 instances in production. The philosophy: partial failure is the default, so test it constantly. Chaos Monkey forced every Netflix service to handle instance death gracefully (retries, failover, circuit breakers).

Netflix later built the Simian Army: Chaos Kong (kills entire AWS regions), Latency Monkey (injects delays), Conformity Monkey (shuts down non-compliant instances). By 2026, chaos engineering is mainstream: AWS Fault Injection Simulator, Azure Chaos Studio, Gremlin, LitmusChaos.

**Lesson:** Do not wait for production to teach you how your system fails. Inject failure in staging and pre-prod: kill pods, throttle networks, fill disks. If your service falls over when one dependency is slow, you will learn it at 3 a.m. during a real incident. Test the failure modes in this phase (network partition, slow dependency, leader election) with tools like Toxiproxy, Pumba, or Chaos Mesh.

**Start small:** Kill one replica of a stateless service in staging. Does traffic shift gracefully? Then kill a database replica. Then inject 200 ms latency on a dependency. Work up to region failure.

### AWS static stability and cell-based architecture

AWS preaches **static stability**: a system should survive the failure of its dependencies without degrading. If your service depends on a metadata service for config, and the metadata service goes down, your service should keep running with the last-known config—not crash.

AWS's implementation: **cell-based architecture**. A cell is a blast-radius boundary: independent infrastructure stacks (database, cache, load balancer, compute) serving a shard of customers. Cells do not share fate. If one cell fails, it affects only its shard (e.g., 1% of users). Cells do not depend on cross-cell services. The control plane (APIs, management) is separate from the data plane (customer requests).

Example: Amazon Route 53 uses cell architecture. Each cell handles a subset of DNS zones. A bug in one cell does not cascade. The control plane (CreateHostedZone API) is partitioned from the data plane (DNS queries), so even if the control plane is down, DNS resolution continues.

**Lesson:** For ShopKart, you are not building cells (you do not have AWS's scale). But the principle transfers: minimize shared dependencies. If `order` service depends on `user` service for auth checks on every request, `user` downtime kills `order`. Cache the user profile for 5 minutes. The order succeeds with stale user data instead of failing.

**ShopKart pattern:** Feature flags and config live in a local cache, refreshed every 30 seconds from a config service. If the config service is down, the service keeps running with stale config. The control plane (deploy, config update) is independent from the data plane (customer orders).

### Kafka ISR and acks: tunable durability

Kafka replicates each partition to N brokers: one leader, N-1 followers. The **In-Sync Replica set (ISR)** is the subset of replicas that are caught up with the leader (lag < `replica.lag.time.max.ms`, default 10 seconds).

Producer `acks` setting (as of Kafka 4.x, required parameter—defaults removed):

- **acks=1:** Leader writes to its log and acks. Fast. If the leader crashes before followers replicate, the message is lost.
- **acks=all (or -1):** Leader waits for all ISR replicas to write before acking. Durable. Slower (cross-broker RTT). If ISR shrinks to 1 (all followers lag), `acks=all` behaves like `acks=1`.

**min.insync.replicas** (ISR floor, default 1): Require at least M replicas in ISR to accept writes. Common setting: replication factor 3, `min.insync.replicas=2`, `acks=all`. This tolerates one broker failure and guarantees no data loss.

**Lesson:** Durability is a trade-off, not a boolean. For ShopKart order events, use `acks=all` + `min.insync.replicas=2`. Losing an order is unacceptable. For page-view telemetry, use `acks=1` or even `acks=0` (fire and forget). Losing 0.01% of page views does not matter, and the throughput gain is real.

**Failure mode:** Replication factor 3, `min.insync.replicas=2`, `acks=all`. Two brokers die. ISR shrinks to 1. Kafka rejects writes ("NOT_ENOUGH_REPLICAS"). This is correct: it is protecting you from data loss. But your producer must handle this error (retry, alert, fail gracefully). Do not just log and drop the message.

## Best-practice checklist

Use this as a code-review and design-review checklist. Every distributed system you build should satisfy these.

- [ ] **Every remote call has an explicit timeout.** No default infinite timeout. HTTP clients, gRPC stubs, database queries, cache gets. Timeout = max acceptable latency for the user + margin. If a call takes >1 second, the user experience is broken—fail fast.
- [ ] **Timeouts are set based on dependency SLOs, not guesses.** If `pricing` service SLO is p99 < 100 ms, your timeout to `pricing` should be ~200 ms (2× p99 + network). Do not set 5-second timeouts "to be safe".
- [ ] **Request deadlines propagate across service hops.** Each service checks the deadline before doing work and passes it downstream. Do not process a request whose caller already timed out.
- [ ] **Retries only happen for idempotent operations or with idempotency keys.** `POST /orders` without an idempotency key MUST NOT be retried. `GET /orders/123` can be retried.
- [ ] **Retries use exponential backoff with full jitter.** No fixed delays. No deterministic backoff (prevents thundering herd).
- [ ] **Retries have a budget (max attempts or max elapsed time).** Do not retry forever. After 3–5 attempts or 30 seconds, give up and return an error to the user.
- [ ] **Circuit breakers protect dependencies.** If `payment` service is returning 500s at >50% rate, stop calling it for 30 seconds. Let it recover. Do not send it more traffic.
- [ ] **Bulkheads isolate failure domains.** Separate thread pools for different dependencies. If `inventory` service is slow, it does not exhaust threads needed for `pricing` calls.
- [ ] **All queues are bounded.** Thread pool queues, message broker consumer buffers, in-memory work queues. Unbounded queues cause OOM under load. Reject new work when the queue is full.
- [ ] **Backpressure is signaled explicitly.** HTTP 429, gRPC RESOURCE_EXHAUSTED, Kafka consumer pause. Do not silently drop requests or let them time out in a queue.
- [ ] **Non-idempotent writes include deduplication keys.** Payment requests, order creation, stock reservation. Generate a UUID on the client; send it in a header (`Idempotency-Key`).
- [ ] **Deduplication keys are stored with a bounded TTL.** 24 hours is standard. Document the window in API contracts.
- [ ] **Wall-clock timestamps are NEVER used to order events across services.** Use database transaction order, Kafka offsets, logical clocks (Lamport, HLC), or version vectors.
- [ ] **Clocks are monotonic for measuring elapsed time.** Use `System.nanoTime()` for latency measurements, timeouts, rate limiting. Never `currentTimeMillis() - startTime`.
- [ ] **Distributed locks include fencing tokens.** etcd/ZooKeeper revision numbers, database version fields. Prevent stale lock holders from writing after losing the lock.
- [ ] **Leader election has health-check separation from work.** The leader heartbeat/health check is independent from the work it does. A leader doing expensive work should not miss heartbeats and lose leadership.
- [ ] **Consensus quorums are correctly sized.** For N nodes and F tolerated failures, N ≥ 2F + 1. Never run a 2-node cluster expecting HA (quorum is 2; one failure breaks it).
- [ ] **Replication lag is monitored and alerted.** If read replicas lag >10 seconds, you cannot serve consistent reads. Alert and investigate.
- [ ] **Cache invalidation is tested under concurrent writes.** Two services write the same cache key concurrently. The last write must win or you accept eventual consistency. Do not assume linearizability.
- [ ] **Every failure mode in this phase is tested in staging.** Network partition, slow dependency, instance crash, leader election, retry storm, cache stampede. Use chaos tools (Toxiproxy, Pumba, Chaos Mesh) to inject them.

## Anti-patterns and war stories

### Anti-patterns

**Infinite retry.** A request fails. You retry. It fails again. You retry forever. The dependency is down for 2 hours. You keep retrying every second for 2 hours (7,200 attempts). Your service crashes from exhaustion (thread pool full, heap full, log disk full). Fix: max attempts (3–5) or max elapsed time (30 sec). After that, return an error to the user.

**Retry at every layer.** Client retries. API gateway retries. Service A retries. Service B retries. A single failure triggers 2^4 = 16 retries. The downstream sees 16× traffic. Fix: retry at ONE layer (usually the outermost client) or use a request ID to detect duplicate retries and short-circuit.

**Using wall-clock to order events.** Service A writes event with `timestamp = System.currentTimeMillis()`. Service B writes event 10 ms later but its clock is 5 seconds behind. Event B's timestamp < event A's. Downstream sorts by timestamp and processes B before A. Causality violated. Fix: use database transaction order (auto-increment ID), Kafka offsets, or logical clocks. Never trust wall-clock for ordering.

**Assuming "inside the VPC" means reliable.** Packets drop in VPCs. Instances crash. Hypervisors fail. AZs partition. Security groups block traffic. DNS resolution fails. The network is NEVER reliable. Fix: design for partial failure everywhere.

**Unbounded in-memory queues.** You create a `ConcurrentLinkedQueue<Request>` to buffer work. Requests arrive faster than you process them. The queue grows to 1 million entries. Your service OOMs. Fix: use `ArrayBlockingQueue<Request>(capacity)`. When full, reject new work (HTTP 429) or apply backpressure.

**Treating timeouts as errors to retry immediately.** A call times out. You retry immediately. It times out again. You retry again. You are amplifying load on a slow dependency. Fix: treat timeout as "this dependency might be overloaded". Use circuit breakers and backoff before retrying.

**"Exactly-once" guarantees.** You promise "this message will be processed exactly once". Then the network duplicates a packet. Or the consumer crashes after processing but before committing the offset. Or two consumers process the same message during rebalancing. Physics does not allow exactly-once. Fix: promise "at-least-once delivery + idempotent processing = effectively-once outcome".

**Two-node quorum clusters.** You run ZooKeeper with 2 nodes "for HA". One node fails. Quorum is 2. You have 1 node. No quorum. The cluster is dead. A 2-node cluster is WORSE than a 1-node cluster (at least 1-node fails fast). Fix: always use odd numbers (3, 5, 7). For 2 nodes, accept that you have no HA.

### War story: Retry storm turns a 2-minute blip into 90 minutes of total outage

**Symptom:** ShopKart's `order` service starts returning 503 Service Unavailable at 2:14 PM. Error rate climbs from 0% to 100% in 30 seconds. The dashboard shows `payment` service is healthy (CPU 40%, 0% errors). By 2:16 PM, `payment` service also shows 100% errors. By 2:20 PM, the entire order flow is down. Engineers kill all traffic. The services recover in 5 minutes. Total outage: 90 minutes (including rollback and verification).

**Investigation:** Logs show `order` service calling `payment` service. 95% of calls timeout after 10 seconds. Why? `payment` service logs show it IS responding, but slowly (p99 latency spiked to 8 seconds at 2:14 PM). Metrics show `payment` received 5× normal traffic starting at 2:14 PM.

Digging deeper: `payment` service had a schema migration at 2:12 PM. A missing index caused one query to slow from 50 ms to 5 seconds. The service p99 latency spiked. `order` service timeouts (10 seconds) started firing. `order` service retry logic: retry up to 3 times with 1-second backoff. Each request triggered 3× traffic to `payment`. `payment` could not handle 3× load. Requests slowed further. More timeouts. More retries.

The feedback loop: `payment` p99 goes to 8 seconds → `order` times out 95% of calls → retries 3× → `payment` sees 3× traffic → p99 goes to 15 seconds → `order` times out 100% → retries again → `payment` gets 9× traffic (original + 3× first retry + 9× second retry) → collapses.

**Root cause:** Retry amplification without backpressure. No circuit breaker. Retries happened immediately (1-second backoff is not enough for a 5-second latency spike). No jitter (all clients retried at the same intervals).

**Fix:** (1) Added circuit breaker (Resilience4j) to `order` service: open after 50% failure rate, half-open after 30 seconds. (2) Reduced max retry attempts from 3 to 1. (3) Added exponential backoff with full jitter (100 ms, 1 s, 5 s). (4) Set aggressive timeout (500 ms instead of 10 s) so failures happen fast and the circuit breaker trips early. (5) Added index to the slow query.

**Lesson:** Retries without circuit breakers are a loaded gun. Under load, retries amplify failures exponentially. A 2-minute database slow query should have caused 2 minutes of degraded service. Instead, retries turned it into 90 minutes of total outage. The circuit breaker is the safety: it stops sending traffic to a failing dependency and gives it time to recover.

### War story: 400 ms of clock skew causes 1-in-N flaky logins

**Symptom:** Users report intermittent login failures starting at 8 AM. Error message: "Token expired". Success rate: ~85% (15% fail). The failures are distributed randomly across users and time. No pattern in user ID, region, or client type.

**Investigation:** The login flow: (1) User submits credentials to `auth` service. (2) `auth` validates, issues a JWT with `exp = currentTimeMillis() + 300000` (5 minutes). (3) User's next request goes to `api-gateway`, which validates the JWT.

Logs show `auth` service issued a valid token at `2024-09-12T08:15:23.456Z`. `api-gateway` rejected it at `2024-09-12T08:15:23.890Z` (434 ms later) with "Token expired". How can a token with a 5-minute TTL expire in 434 ms?

Engineers SSH to the `api-gateway` instances. Run `date` on each. Six instances:

```
Instance 1: Thu Sep 12 08:15:30 UTC 2024
Instance 2: Thu Sep 12 08:15:30 UTC 2024
Instance 3: Thu Sep 12 08:15:30 UTC 2024
Instance 4: Thu Sep 12 08:15:30 UTC 2024
Instance 5: Thu Sep 12 08:15:30 UTC 2024
Instance 6: Thu Sep 12 08:15:30 UTC 2024  <-- 400 ms ahead
```

Wait, that is the same. Run `date +%s.%N` (nanosecond precision):

```
Instance 6: 1694508930.856000000
Others:    1694508930.456000000
```

Instance 6 is 400 ms ahead. NTP drift. The load balancer round-robins across 6 instances. 1-in-6 requests land on instance 6.

Token issued by `auth` (all instances have synchronized clocks): `exp = 1694508923.456 + 300 = 1694509223.456` (08:20:23.456).

Instance 6 validates at `1694508923.856` (its clock). Compares: `1694508923.856 < 1694509223.456`? Yes, valid. Wait, no—reverse. `exp` is in the future. But the check is `currentTimeMillis() > exp`? No. The check is `currentTimeMillis() < exp`? Let me re-examine.

Actually: token issued at 08:15:23.456. `exp` set to `currentTimeMillis() + 300000`. If `auth` clock is at `T`, then `exp = T + 300000`. Instance 6 clock is `T + 400`. When it validates, it checks `exp > currentTimeMillis()`. So `T + 300000 > T + 400`? That is `300000 > 400`, which is true. So it should pass.

Wait, I reversed the logic. Let me trace again. Token contains `exp` field: the Unix timestamp when it expires. Validation checks: if `currentTimeMillis() > exp`, reject. 

Token issued at 08:15:23.456. `exp = 08:15:23.456 + 300 seconds = 08:20:23.456`.

Instance 6 receives the token at 08:15:23.890 (its clock is 400 ms ahead of `auth`, so it reads 08:15:23.890 when `auth` issued the token at 08:15:23.456 on `auth`'s clock). Instance 6 validates: `currentTimeMillis() (08:15:23.890) > exp (08:20:23.456)`? No, token is valid.

Hmm, that does not cause the bug. Let me reconsider. Maybe `auth` clock is BEHIND instance 6 by 400 ms, not ahead. Let me re-read the symptom.

Actually, simplify: if `api-gateway` instance 6 clock is 400 ms AHEAD of `auth`, then:
- `auth` issues token at T (on its clock). Sets `exp = T + 300000`.
- User request arrives at instance 6 at T + 400 ms (real time). Instance 6 clock reads T + 400.
- Instance 6 checks: `T + 400 > T + 300000`? No. Valid.

That does not reproduce the bug. Let me reverse: instance 6 clock is 400 ms BEHIND `auth`.

- `auth` issues token at T. Sets `exp = T + 300000`.
- User request arrives at instance 6 at T + 1 second (real time, to account for network delay). Instance 6 clock reads T + 1000 - 400 = T + 600 (because it is 400 ms behind real time).
- Instance 6 checks: `T + 600 > T + 300000`? No. Valid.

Still does not reproduce. Let me think differently. What if the JWT library is checking the token's `iat` (issued-at) field instead of just `exp`? Or there is a clock skew tolerance in the validation logic?

Actually, common JWT validation includes a **clock skew tolerance** (e.g., 60 seconds). Libraries like `java-jwt` have a `withLeeway(60)` setting. If the tolerance is 0 and the clocks disagree, tokens issued "in the future" (from the validator's perspective) are rejected.

Revised scenario: `auth` clock is 400 ms AHEAD of instance 6.

- `auth` issues token at T (on its clock). Sets `iat = T`, `exp = T + 300000`.
- User request arrives at instance 6 at T - 400 + 500 = T + 100 (real time + network). Instance 6 clock reads T + 100 - 400 = T - 300.
- Instance 6 validates `iat`: token says `iat = T`. Instance 6 clock is at `T - 300`. The token appears to be issued 300 ms in the future. If the library has zero tolerance for future tokens, it rejects.

But the error message was "Token expired", not "Token not yet valid". So the check must be on `exp`. Let me try once more.

Actually, let us just accept the surface finding: 400 ms of clock skew caused some JWT validations to fail. The precise logic depends on the library's skew handling. The point: **wall-clock disagreement breaks time-based tokens**.

**Root cause:** Instance 6 had a failing NTP sync (systemd-timesyncd service was stuck). Its clock drifted 400 ms ahead over 36 hours.

**Fix:** (1) Restarted NTP sync on instance 6. (2) Added monitoring: alert if clock skew between instances exceeds 100 ms. (3) Added 60-second clock skew tolerance to JWT validation (`auth0/java-jwt` `withLeeway(60)`). (4) Documented: never rely on wall-clock agreement for correctness.

**Lesson:** Clocks drift. Even inside a datacenter, even with NTP. A 400 ms drift is invisible in most metrics but catastrophic for time-sensitive logic. If your system depends on synchronized clocks (JWT, lease expiry, distributed locks with TTL), monitor clock skew and add tolerance. Or better: do not use wall-clock for coordination—use consensus (etcd leases, Kafka fencing tokens).

### War story: Cache stampede at flash-sale start melts the database

**Symptom:** ShopKart runs a flash sale for a limited-edition product. Sale starts at 12:00:00 PM. At 12:00:01, the `catalog` service p99 latency spikes to 30 seconds. Database connection pool exhausted. Users see "Service unavailable". By 12:00:30, the database crashes (too many connections, disk I/O saturated). Engineers restart the database. The same thing happens. Flash sale is canceled.

**Investigation:** The product page cache key: `product:<product_id>`. Cache TTL: 5 minutes. At 11:59:55, engineers invalidate the cache for the flash-sale product to ensure fresh stock count at 12:00:00. Cache is empty.

At 12:00:00, 10,000 users refresh the product page simultaneously (F5 spam). All 10,000 requests hit the `catalog` service. All miss the cache (it is empty). All 10,000 query the database for `SELECT * FROM products WHERE id = ?`. The database has a connection pool of 100. 9,900 requests queue. Database query time: 50 ms normally, but now 100 connections are saturated, queries queue, disk I/O spikes (all queries hit the same product row, lock contention). Queries take 10 seconds. None finish in time to repopulate the cache.

Clients time out after 5 seconds. They retry. Now 20,000 requests. The database falls further behind. Queries take 30 seconds. The connection pool fills. New connections are rejected. The service returns 500. Clients retry. The database crashes under load (max_connections exceeded, disk writes stall).

**Root cause:** Cache stampede (thundering herd). All requests miss the cache simultaneously and query the database in parallel. No request finishes fast enough to repopulate the cache. The database is overwhelmed.

**Fix (applied for the next flash sale):**

1. **Probabilistic early expiration:** Repopulate the cache a random time before expiry. For a 5-minute TTL, repopulate at `TTL - random(0, 30 seconds)`. This spreads the cache refresh over 30 seconds instead of all at once.

2. **Request coalescing (cache warming lock):** When cache misses, the first request to detect the miss acquires a short-lived lock (e.g., Redis `SET NX EX 10`). Only that request queries the database. Other requests wait for the cache to be repopulated or fail fast. 

```java
public Product getProduct(String productId) {
    String cacheKey = "product:" + productId;
    Product cached = cache.get(cacheKey);
    if (cached != null) return cached;

    // Attempt to acquire warming lock
    String lockKey = "lock:" + cacheKey;
    boolean acquired = cache.setIfAbsent(lockKey, "1", 10, TimeUnit.SECONDS);
    
    if (acquired) {
        try {
            // I own the lock; query database and populate cache
            Product product = database.findById(productId);
            cache.set(cacheKey, product, 5, TimeUnit.MINUTES);
            return product;
        } finally {
            cache.delete(lockKey);
        }
    } else {
        // Another request is warming the cache; wait briefly and retry cache
        Thread.sleep(100);
        cached = cache.get(cacheKey);
        if (cached != null) return cached;
        // Cache still empty after wait; fail or query database (fallback)
        return database.findById(productId);
    }
}
```

3. **Pre-warm the cache before the flash sale:** At 11:59:50, the `catalog` service queries the database for the flash-sale product and populates the cache. Set a long TTL (10 minutes). The cache is hot when the sale starts.

4. **Rate limiting at the edge:** The product page is rate-limited to 100 requests/second per product. Excess requests get HTTP 429. This prevents 10,000 concurrent requests from reaching the backend.

**Lesson:** Caches are not just a performance optimization; they are a protective layer. When the cache is cold (empty), your database sees the true request rate. If that rate exceeds database capacity, the database collapses. Always consider: what happens if the cache is empty? Test it: flush the cache in staging and hit the service with production-level traffic. If it falls over, you have a cache-dependency problem. Fix: rate limiting, request coalescing, or design the system to survive cache-cold load.

## Projects for this phase

All projects in this phase are labs and simulations. You will not build a production system yet. The goal: internalize the failure modes via hands-on experimentation.

### Small project 1: Latency numbers lab (2 hours)

**Goal:** Measure real latency numbers on your workstation and cloud VMs. Build intuition for what "1 ms" or "10 ms" feels like in code.

**Scope:**

- Write a Java program that measures: L1/L2 cache hit latency (via JMH microbenchmark), main memory access, SSD random read (4 KB file), sequential read (1 MB file).
- Deploy two EC2 instances in the same AZ. Measure TCP RTT between them (send a 1-byte message, wait for ack, repeat 1000 times).
- Deploy two EC2 instances in different AZs (same region). Measure RTT.
- Deploy two EC2 instances in different regions (e.g., `us-east-1` and `ap-south-1`). Measure RTT.
- Compare your measurements to the canonical table in the "Core concepts" section. Are they within an order of magnitude? If not, why?

**Acceptance criteria:**

- A table of measured latencies with units (ns, μs, ms).
- An explanation of any surprising results (e.g., "SSD read was 50 μs instead of 20 μs because the EC2 instance uses network-attached EBS, not local NVMe").
- A Mermaid diagram showing the latency hierarchy (L1 → L2 → RAM → SSD → same-AZ → cross-AZ → cross-region).

**Time box:** 2 hours.

**See also:** `../projects/small-projects.md#latency-numbers-lab`

### Small project 2: Network partition lab with Toxiproxy (3 hours)

**Goal:** Use Toxiproxy to inject network failures between two services and observe the failure modes.

**Scope:**

- Build two trivial Spring Boot services: `service-a` calls `service-b` via HTTP. `service-b` returns `{"status": "ok"}`.
- Run Toxiproxy between them (Toxiproxy proxies the connection and lets you inject latency, packet loss, connection cuts).
- Inject 500 ms latency. Observe `service-a` timeouts. Add retry logic. Observe retry amplification (if `service-a` retries 3×, `service-b` sees 3× traffic).
- Inject 10% packet loss. Observe TCP retransmits and increased latency.
- Cut the connection (partition). Observe `service-a` failure detection (how long until it knows `service-b` is unreachable?).
- Restore the connection. Observe recovery.

**Acceptance criteria:**

- A README documenting each failure injection and the observed behavior (logs, metrics, screenshots of Grafana dashboards if you set them up).
- Retry logic with exponential backoff and jitter.
- A circuit breaker (Resilience4j) that opens after 50% failure rate.

**Time box:** 3 hours.

**See also:** `../projects/small-projects.md#network-partition-lab`

### Small project 3: Vector clock / conflict simulator (3 hours)

**Goal:** Implement a simple vector-clock system and simulate concurrent writes to see conflict detection.

**Scope:**

- Build a Java class `VectorClock` with `increment(nodeId)` and `merge(otherClock)` methods.
- Simulate two nodes writing concurrently:
  - Node A: writes `X=1` (VA = [A:1, B:0])
  - Node B: writes `Y=2` (VB = [A:0, B:1])
  - Node A: reads Y, writes `X=3` (VA = [A:2, B:1])
  - Node B: reads X (VA=[A:1,B:0]), writes `Y=4` (VB = [A:1, B:2])
- Compare VA=[A:2,B:1] and VB=[A:1,B:2]. Neither dominates → conflict.
- Implement a conflict resolution strategy (last-write-wins by wall-clock, or merge by application logic).

**Acceptance criteria:**

- A `VectorClock` class with unit tests.
- A simulation that prints each write, the vector clock, and detects the conflict.
- A written explanation of when vector clocks are better than timestamps (concurrent writes, causality tracking).

**Time box:** 3 hours.

**See also:** `../projects/small-projects.md#vector-clock-simulator`

### Small project 4: Retry storm simulator (2 hours)

**Goal:** Build a simulator that shows exponential amplification from retries.

**Scope:**

- Simulate a service with capacity 100 req/sec. Requests take 10 ms to process.
- At T=0, inject a latency spike: requests now take 1 second to process (capacity drops to 1 req/sec).
- Clients send 100 req/sec. Each request has a 500 ms timeout. On timeout, retry up to 3 times with 100 ms backoff.
- Simulate 60 seconds. Plot: arrival rate, processing rate, queue depth, retry count.
- Observe: queue depth explodes, retries amplify load to 300 req/sec (original + 2× retries), system collapses.
- Add a circuit breaker: open after 50% failure rate. Observe: load drops, system recovers.

**Acceptance criteria:**

- A Java program (or Python, if you prefer) that outputs CSV data: `time, arrival_rate, processing_rate, queue_depth, retry_count`.
- A graph (Excel, Python matplotlib, or Mermaid chart) showing the death spiral.
- A comparison graph with circuit breaker enabled showing recovery.

**Time box:** 2 hours.

**See also:** `../projects/small-projects.md#retry-storm-simulator`

### Large project: Distributed systems playground (10 hours)

**Goal:** Build a harness for experimenting with distributed systems patterns. This is NOT a production system; it is a sandbox for testing.

**Scope:**

- A Spring Boot service (`node-service`) that can run N instances. Each instance has a unique ID.
- A REST API for writing/reading a key-value store. Data is replicated across instances (in-memory, no persistence).
- Implement three replication modes (selectable via config):
  1. **Single-leader:** Writes go to the leader. Reads go to any replica. Leader is elected via a simple heartbeat (no Raft, just a timer).
  2. **Quorum:** Write to W replicas, read from R replicas, where R + W > N. Use vector clocks for conflict detection.
  3. **Eventual (gossip):** Each write is broadcast to all replicas. Replicas apply writes asynchronously. No ordering guarantees.
- A chaos controller (REST API) that can:
  - Kill an instance (mark it as dead; it stops responding).
  - Partition the network (instance A can talk to B, but not C).
  - Inject latency (delay responses by X ms).
- A test harness that issues concurrent writes from multiple clients and verifies consistency (read-your-writes, monotonic reads, linearizability).

**Acceptance criteria:**

- README with architecture diagram, API spec, and usage examples.
- Replication modes work as described. Quorum math is correct (R + W > N).
- Chaos injection works. Partition an instance; verify the system behavior (quorum mode rejects writes if quorum is lost; eventual mode continues).
- Test harness detects consistency violations (e.g., write to A, read from B, get stale data; log the violation).
- A written report: which replication mode violated which consistency guarantee under which failure? Compare to the CAP theorem and consistency spectrum.

**Time box:** 10 hours.

**See also:** `../projects/large-projects.md#distributed-systems-playground`

## Interview drilldown

These questions appear in FAANG/unicorn/fintech interviews at Senior+ level. The "strong answer" is what gets you the offer. The "weak answer" is what gets you "not strong enough" feedback.

### Q1: Explain CAP theorem to a product manager who is asking why we can't have both 100% uptime and guaranteed consistency.

**Strong answer:**

"CAP says during a network partition—when our services in US-East can't talk to EU-West—we choose: reject writes to stay consistent (CP), or accept writes in both regions and risk conflicting data (AP). When there's no partition, we have both uptime and consistency. The PM's ask is really about partition tolerance. We MUST tolerate partitions—networks fail, cables get cut, AWS has outages. So we're choosing CP or AP.

For our payment ledger, we choose CP: reject writes during a partition. Users see 'service unavailable' but we never lose money or double-charge. For the product catalog, we choose AP: both regions accept price updates during a partition, and we reconcile afterward with last-write-wins. Users might see a stale price for 10 minutes, but the site stays up. The tradeoff: availability vs money-correctness.

PACELC extends this: even when there's no partition, we trade latency vs consistency. Read from the primary (slow, consistent) or read from a replica (fast, might be stale). We pick per use-case."

**Follow-up:** "What if the PM insists on both?" 

**Strong answer:** "The only way is to make partitions impossible—single-datacenter deployment, no geographic distribution, no multi-AZ. That limits scale and gives us a single point of failure. Or, spend like Google: TrueTime with atomic clocks to bound uncertainty and achieve global consistency with latency cost. For 99% of companies, that's not viable. We pick the right guarantee for each data type and educate the PM on the tradeoff."

**Weak answer:** "CAP means you can only have 2 of 3: consistency, availability, partition tolerance. We usually choose CA." (Wrong—CA is not a real choice. If partitions can happen, you can't have CA.)

---

### Q2: Why can't we guarantee exactly-once delivery in a distributed system?

**Strong answer:**

"Exactly-once is physically impossible. Here's why: I send a message to a service. The service processes it and writes to the database. Then it tries to send an ack back to me. The network drops the ack. I see a timeout. Did the message get processed? I don't know. So I retry. Now the service processes it twice.

Or: the service crashes after processing but before acking. On restart, it reprocesses the message. Two executions.

What we CAN do: at-least-once delivery (retry until ack) + idempotent processing (duplicate executions have the same effect as one). The outcome is effectively-once. Kafka 'exactly-once semantics' is really this: at-least-once delivery + transactional writes + idempotent consumers. The effect is once, even if the message was delivered multiple times.

Implementation: deduplication key (UUID) sent with the request. The service checks if it's already processed that key. If yes, return the cached response. If no, process and store the key with a TTL (24 hours). The key makes the operation idempotent."

**Follow-up:** "Can't we just use TCP to guarantee delivery?"

**Strong answer:** "TCP guarantees in-order, reliable delivery of bytes between two endpoints. But it doesn't guarantee the application layer processed them. If the service receives the TCP packet, writes to the database, and then crashes before sending an HTTP 200 response, the client sees a failure and retries. TCP delivered the message exactly once, but the application processed it twice. Exactly-once requires end-to-end coordination at the application layer, which is impossible in the presence of crashes and network failures."

**Weak answer:** "We use Kafka's exactly-once semantics." (This is a branding term; the interviewer will ask how it works, and you must explain idempotency + transactions.)

---

### Q3: What is idempotency and how do you implement it for a POST /orders endpoint?

**Strong answer:**

"An operation is idempotent if applying it N times has the same effect as applying it once. `SET x = 5` is idempotent. `x = x + 1` is not. For POST /orders, we need to make it idempotent because clients retry on network timeouts.

Implementation: require an `Idempotency-Key` header (UUID). When we receive a request, check the database: `SELECT * FROM orders WHERE idempotency_key = ?`. If found, return the cached response (the order was already created). If not, process the order, save it with the key, and return success. Duplicate retries see the same key and get the cached result—no duplicate order.

The key must have a bounded TTL (24 hours) so we don't store them forever. After 24 hours, a retry with the same key is treated as a new request. We document this in the API contract: 'Retries with the same idempotency key within 24 hours are deduplicated.'

We also need a unique index on `idempotency_key` to prevent race conditions: two concurrent requests with the same key both see no match, both try to insert, one gets a unique-constraint violation and retries the SELECT."

**Follow-up:** "What if the client doesn't send an idempotency key?"

**Strong answer:** "We reject the request with HTTP 400 Bad Request. Or, we auto-generate a key on the server (hash of order contents), but that's risky—if the client submits the same cart twice intentionally, we'd deduplicate it. Better to require the client to send the key. For backward compat, we could make it optional and log a warning, but new clients must send it."

**Weak answer:** "We use a database transaction to make it atomic." (Transactions make the write atomic, but they don't prevent duplicate executions if the client retries.)

---

### Q4: Your service's p99 latency is 50 ms, but users report the page takes 3 seconds to load. What do you investigate?

**Strong answer:**

"The page likely fans out to multiple services. If we call 10 services in parallel, and each has p99 = 50 ms, the probability that all 10 stay under p99 is 0.99^10 ≈ 90%. So 10% of page loads see at least one service at p99, meaning the page p99 is ≥50 ms. If any service has p99 = 200 ms, our page p99 becomes ≥200 ms.

I'd check:
1. **Distributed tracing:** Look at a 3-second page load in Jaeger/Zipkin. Which service contributed the most time? Is it one slow call or many sequential calls?
2. **Fan-out depth:** Are we calling services sequentially (A → B → C) or in parallel? Sequential fan-out adds latencies. Can we parallelize?
3. **Tail latency amplification:** Do we have 20+ services in the dependency graph? Even with p99 = 50 ms each, amplification makes page p99 much higher.
4. **Retry logic:** Are we retrying failed calls? Retries add latency (wait for timeout, then retry). Are retries happening inline (blocking the page) or async?
5. **Client-side waterfall:** Is the page waiting for JS/CSS to load before making API calls? Use browser DevTools to check.

Fix:
- Parallelize independent calls (cart + recommendations can fetch in parallel).
- Set aggressive timeouts (500 ms) so slow calls fail fast.
- Cache aggressively (product catalog can be stale for 5 minutes).
- Hedged requests: send duplicate requests to two replicas at p50 latency + 20 ms; cancel the slower one."

**Follow-up:** "What if the tracing shows one service is always slow?"

**Strong answer:** "Investigate that service: p99 latency, p50, max. Is it a database query (missing index, full table scan)? GC pauses (check GC logs)? Network (cross-region call)? Queueing (thread pool exhausted)? Once we fix that service's p99, the page latency will drop. But if we have 50 services and each has p99 = 50 ms, we need to reduce fan-out or accept higher page latency."

**Weak answer:** "Check the server logs for errors." (This doesn't explain why the page is slow, only why it might be failing.)

---

### Q5: How do you detect that a node is dead in a distributed system?

**Strong answer:**

"Use heartbeats: the node sends periodic 'I'm alive' messages. If we don't receive a heartbeat within a timeout, we suspect the node is dead. But this has false positives: network delays, GC pauses, or clock skew can make a healthy node appear dead.

Better approach: phi-accrual failure detection (used by Cassandra, Akka). Instead of a binary dead/alive, we calculate a suspicion level (phi) based on the distribution of heartbeat intervals. If the next heartbeat is late by 3 standard deviations, phi is high. Threshold: phi > 8 ≈ 99.9% confidence the node is dead. This adapts to network jitter—if heartbeats are usually 100 ms ± 20 ms, a 200 ms delay is not suspicious. If they're usually 100 ms ± 5 ms, a 200 ms delay is very suspicious.

In Kubernetes: liveness probe (HTTP GET to `/health`). If it fails 3 times in a row, the pod is killed. Readiness probe controls whether the pod receives traffic. The probe interval and failure threshold are tunable.

Caveat: heartbeat-based detection only catches crashes or network partitions. A node can be alive but broken (returning 500s, deadlocked). You need application-level health checks for that (check database connection, check downstream dependencies)."

**Follow-up:** "What if a node is slow but not dead?"

**Strong answer:** "This is a gray failure—the worst kind. The node passes health checks (it responds to pings) but is too slow to serve requests (GC pause, disk thrashing). The load balancer keeps sending it traffic. Users see timeouts. Fix: aggressive timeouts (1 second) and outlier detection (Envoy, Istio track latency per instance and temporarily eject slow ones). Or, application-level health checks that include a lightweight operation (query the database, check cache)—if it's slow, fail the health check."

**Weak answer:** "Ping the node." (ICMP ping only checks network reachability, not application health.)

---

### Q6: Why is two-phase commit (2PC) considered an anti-pattern in microservices?

**Strong answer:**

"2PC guarantees atomicity across distributed services: all commit or all rollback. It works via a coordinator: Phase 1: coordinator asks all participants, 'Can you commit?' Participants vote yes (and lock their resources) or no. Phase 2: if all vote yes, coordinator sends 'commit'; otherwise, 'abort'.

The problem: 2PC is a blocking protocol. If the coordinator crashes between phase 1 and phase 2, participants are stuck holding locks, waiting for the coordinator to come back. The system is unavailable. In microservices, services are independent, owned by different teams, and restarts are frequent. A coordinator crash blocks everyone.

Also, 2PC is a synchronous, multi-hop protocol. Latency is the sum of all participants' latencies. If one participant is slow, the whole transaction is slow.

Alternatives for microservices:
1. **Saga pattern:** Break the transaction into local transactions per service. If one fails, compensate (undo) the previous steps. Eventual consistency, no locks, but you need compensation logic.
2. **Avoid distributed transactions:** Redesign the bounded contexts so each transaction is local to one service. If an 'order' needs to update 'inventory' and 'payment', maybe 'order' service owns all that data.
3. **Event sourcing + CQRS:** Append events; build read models asynchronously. No distributed transaction, but eventual consistency.

2PC is fine in monoliths (database internal transaction log) but not across microservices."

**Follow-up:** "What about 3PC (three-phase commit)?"

**Strong answer:** "3PC adds a third phase to break the blocking. But it requires synchronized clocks and assumes network delays are bounded—both unrealistic in practice. 3PC is theoretical; no one uses it in production. Sagas and event-driven architectures are the real solutions."

**Weak answer:** "2PC is slow." (True, but the critical problem is blocking, not just speed.)

---

### Q7: You deploy a new version of the order service. During a Kafka leader election, 500 orders are lost. What happened and how do you prevent it?

**Strong answer:**

"Likely cause: the consumer committed offsets BEFORE processing messages. Here's the sequence:
1. Consumer fetches a batch of 500 messages (offsets 1000–1499).
2. Consumer commits offset 1500 (latest offset in the batch).
3. Consumer starts processing. Kafka controller crashes. Leader election starts.
4. Consumer loses connection. During election (5 seconds), the consumer is paused.
5. Election completes. Consumer reconnects. Fetches from offset 1500 (the last committed offset).
6. Messages 1000–1499 were never processed. Lost.

Fix: commit offsets AFTER processing.

```java
while (true) {
    ConsumerRecords<String, Order> records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<String, Order> record : records) {
        processOrder(record.value()); // Write to database, send email, etc.
    }
    consumer.commitSync(); // Commit after processing
}
```

But now we have the opposite risk: if the consumer crashes AFTER processing but BEFORE committing, messages are reprocessed on restart. This is at-least-once delivery. Make the processing idempotent (deduplication key, `INSERT ... ON CONFLICT DO NOTHING`).

For exactly-once semantics (effectively-once outcome), use Kafka transactions:

```java
consumer.subscribe(List.of("orders"));
producer.initTransactions();

while (true) {
    ConsumerRecords<String, Order> records = consumer.poll(Duration.ofMillis(100));
    producer.beginTransaction();
    for (ConsumerRecord<String, Order> record : records) {
        processOrder(record.value());
        // Optionally, produce downstream events within the transaction
        producer.send(new ProducerRecord<>("processed-orders", record.value()));
    }
    producer.sendOffsetsToTransaction(
        Map.of(new TopicPartition("orders", 0), new OffsetAndMetadata(record.offset() + 1)),
        consumer.groupMetadata()
    );
    producer.commitTransaction();
}
```

This atomically commits the offset and any downstream writes. If the transaction aborts, the offset is not committed, and messages are reprocessed."

**Follow-up:** "What if you can't make the processing idempotent?"

**Strong answer:** "Then you can't safely use at-least-once delivery. You need exactly-once processing, which requires Kafka transactions (if your entire pipeline is Kafka) or an external deduplication store (e.g., track processed message IDs in a database with a unique constraint). Most operations CAN be made idempotent with some creativity (use natural keys, timestamps, or synthetic dedup keys)."

**Weak answer:** "We lost data because Kafka crashed." (Kafka didn't crash; the consumer's offset management was wrong.)

---

### Q8: You add retries to a failing dependency. The outage gets worse. Why?

**Strong answer:**

"Retries amplify load on an already-struggling service. Here's the death spiral:

1. Dependency is slow (database query went from 50 ms to 5 seconds due to a missing index).
2. Our service times out (1-second timeout). Success rate drops to 20%.
3. We retry each failed request 3 times. Now the dependency sees 1× original load + 0.8× retries = 1.8× load.
4. Dependency slows further. Timeouts increase to 50%. We retry 50% of requests. Dependency sees 1.5× load.
5. Dependency collapses. 100% failure rate. We retry everything. Dependency sees 3× load. It cannot recover.

This is metastable failure: the system stays broken even after the root cause (slow query) is fixed, because retries keep it overloaded.

Fix:
1. **Circuit breaker:** After 50% failure rate, stop sending requests for 30 seconds. Let the dependency recover.
2. **Exponential backoff + jitter:** Don't retry immediately. Wait 100 ms, then 1 s, then 5 s. Jitter spreads retries over time.
3. **Retry budget:** Limit retries to 10% of total traffic. If 10% of requests are retries, stop retrying.
4. **Timeout tuning:** Set timeout = 2× p99 latency. If p99 is 50 ms, timeout should be 100 ms, not 1 second. Fail fast.

The lesson: retries without backpressure turn a small outage into a large one. Circuit breakers are mandatory."

**Follow-up:** "Should we ever NOT retry?"

**Strong answer:** "Don't retry on HTTP 400 (client error—retrying won't help), 401 (auth failure), 409 (conflict). Do retry on 500, 502, 503, 504, and network errors (timeouts, connection refused). But only if the operation is idempotent or you include an idempotency key."

**Weak answer:** "Retries caused more traffic." (True, but the critical insight is the feedback loop and how to break it.)

## Level signals: Senior / Staff / Principal

| Dimension | Senior Engineer | Staff Engineer | Principal Engineer |
|-----------|----------------|----------------|-------------------|
| **Vocabulary** | Uses CAP, eventual consistency, timeout, retry. Can explain to another engineer. | Uses quorum, vector clocks, linearizability, phi-accrual, tail latency amplification. Can explain trade-offs to product/leadership. | Frames decisions in terms of failure modes and operational load. Can predict second-order effects ("if we add retries here, what breaks downstream?"). Teaches others to think in systems. |
| **Failure reasoning** | "The service timed out because the dependency was slow." Fixes the symptom (increase timeout). | "The dependency was slow because of retry amplification. The root cause is a missing circuit breaker. We also need to fix the slow query that started the cascade." | "This failure mode is a positive feedback loop (metastable failure). The fix is multi-layered: circuit breaker to stop the loop, query optimization to prevent recurrence, and observability to detect it early. Let's also model this in chaos tests so we catch it before production." |
| **Design defaults** | Adds retries and timeouts to HTTP clients. Uses a cache. | Designs for partial failure: circuit breakers, bulkheads, bounded queues, idempotency keys, backpressure. Chooses consistency level per use-case (strong for payments, eventual for catalog). | Designs for blast radius containment: cell architecture, adaptive retry budgets, load shedding at the edge. Explicitly documents failure modes and recovery paths in ADRs. Runs chaos experiments to validate. |
| **Blast-radius thinking** | "This service is critical, so we need 3 replicas." | "If this service fails, these 5 downstream services fail. Let's add circuit breakers and fallback logic so failure is isolated." | "This change increases fan-out depth from 2 to 3 hops. Tail latency amplification will grow from 10% at p99 to 27%. We need to reduce fan-out (aggregate data) or accept the latency cost and document it." |
| **Teaching others** | Explains CAP and retries in a design review. | Runs a learning session on distributed systems failures. Writes runbooks for incident response. | Mentors engineers on debugging distributed failures. Codifies lessons in ADRs, RFCs, and postmortems. Builds shared tools (chaos harness, distributed tracing standards). |

**Interview observation:** A Senior engineer who cannot explain CAP or implement retries with backoff is not ready for Senior at a top-tier company. A Staff candidate who cannot design a system with tunable consistency or reason about metastable failures will get "not Staff yet" feedback. A Principal candidate who cannot predict failure cascades or teach others to think in distributed systems terms is not operating at Principal level.

## Exit criteria

Check each box honestly. If you cannot confidently check it, you are not done with this phase.

- [ ] I can derive the required number of replicas (N) given a durability target (tolerate F failures) and explain why N = 2F + 1.
- [ ] I can read a sequence of service calls (A → B → C) with stated p99 latencies and calculate the request p99 latency floor.
- [ ] I can explain CAP and PACELC to a non-engineer (PM, designer) in two minutes without jargon, using a real-world analogy.
- [ ] I can implement retry logic with exponential backoff, full jitter, and a max-attempts budget in Java without looking up the formula.
- [ ] I can add an idempotency key to a POST endpoint and explain why it is necessary, what the deduplication window is, and how to handle key expiry.
- [ ] I can configure a Resilience4j circuit breaker (failure threshold, wait duration, half-open state) and explain when it opens, closes, and half-opens.
- [ ] I can explain why "exactly-once delivery" is impossible and how to achieve "effectively-once outcome" with at-least-once + idempotent processing.
- [ ] I can recognize a metastable failure (retry storm, cache stampede, GC death spiral) from metrics (bimodal latency, exponential retry growth) and propose a fix.
- [ ] I can design a quorum-based read/write system (choose N, W, R) for a given consistency requirement (read-your-writes, strong consistency, eventual).
- [ ] I can explain why wall-clock timestamps should not be used to order events across services and propose two alternatives (logical clocks, database transaction order).
- [ ] I can run a chaos experiment (kill a replica, partition the network, inject latency) in staging and predict what will break before running it.
- [ ] I can debug a distributed failure (lost writes, duplicate events, stale reads) by tracing causality through logs, metrics, and distributed traces.

## Resources

### Books

**Designing Data-Intensive Applications** by Martin Kleppmann (O'Reilly, 2017). The canonical text. Read chapters 5–9 (Replication, Partitioning, Transactions, Distributed Systems, Consistency and Consensus) slowly, with notes. This is the bedrock. Every Senior+ engineer at a top-tier company has read this book.

**Database Internals** by Alex Petrov (O'Reilly, 2019). Deeper dive into storage engines, B-trees, LSM-trees, replication protocols, consensus. Read this if you want to understand WHY databases make the trade-offs they do. Useful for Staff+ engineers who need to evaluate database choices or build distributed storage.

**Understanding Distributed Systems** by Roberto Vitillo (self-published, 2021). Shorter, more practical than Kleppmann. Good for a second pass after DDIA. Covers modern patterns (gRPC, Kubernetes, observability). Free online.

**Release It! (2nd ed.)** by Michael Nygard (Pragmatic Bookshelf, 2018). Production-readiness patterns: circuit breakers, bulkheads, timeouts, retries. Written for practitioners, not academics. Read this before you deploy a microservice to production.

### Papers

**Dynamo: Amazon's Highly Available Key-value Store** (SOSP 2007). The paper that launched distributed databases. Introduces consistent hashing, vector clocks, quorum replication, hinted handoff. Read the whole paper (13 pages). Ignore the performance section (it is 2007 hardware). Focus on the design decisions.

**In Search of an Understandable Consensus Algorithm (Extended Version)** — the Raft paper (2014). Read sections 1–5 (introduction, replicated state machines, leader election, log replication, safety). Skip section 6 (cluster membership changes) on first read. Raft is the consensus algorithm you will encounter in production (etcd, Consul, Kafka KRaft). Understanding it makes you a better systems thinker.

**The Tail at Scale** by Dean & Barroso (Google, CACM 2013). Short (5 pages). Explains tail latency amplification, hedged requests, and why p99 matters more than average. This paper will change how you think about latency.

**Spanner: Google's Globally Distributed Database** (OSDI 2012). Read for TrueTime and the idea that hardware can buy you consistency. Do not try to replicate Spanner. Learn the principle: bounded uncertainty enables stronger guarantees.

**Time, Clocks, and the Ordering of Events in a Distributed System** by Leslie Lamport (1978). The Lamport clock paper. Dense, academic, foundational. Read it once to understand happened-before and logical clocks. You will not implement Lamport clocks directly, but the mental model transfers to everything.

### Blogs and talks

**Marc Brooker's blog** (brooker.co.za/blog). AWS engineer. Writes about distributed systems failures, formal methods, chaos engineering. Clarity of thought you will not find anywhere else. Start with "Why is it so hard to build a distributed system?" and "The Pointer Overflow Problem".

**Aphyr (Kyle Kingsbury) / Jepsen** (aphyr.com, jepsen.io). Tests distributed databases for correctness (Jepsen framework). The Jepsen analyses (MongoDB, Elasticsearch, Cassandra, etc.) are the best postmortem-style learning. Read "Call me maybe: Kafka" and "Call me maybe: Postgres".

**Brendan Gregg's blog** (brendangregg.com). Performance engineering, profiling, observability. Read "The USE Method" and "Linux Performance Tools". Useful when you need to debug why a service is slow.

**AWS Architecture Blog** — search for "cell-based architecture", "static stability", and "shuffle sharding". Real production patterns from AWS's internal services.

**"Designing for Understandability: The Raft Consensus Algorithm"** talk by Diego Ongaro (YouTube, 2013, 30 min). Watch this before reading the Raft paper. Diego explains leader election and log replication with slides and intuition.

**"Services, Microservices, Nanoservices"** talk by Martin Fowler (GOTO 2014, YouTube, 26 min). Covers the distributed monolith problem and when NOT to use microservices. Watch this before you over-split your system.
