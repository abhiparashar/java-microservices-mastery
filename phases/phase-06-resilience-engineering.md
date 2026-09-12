# Phase 6 - Resilience and Reliability Engineering

> **Weeks:** 37–41 | **Prerequisites:** Phases 2, 3, 4 | **Time budget:** 45–50 hrs  
> **You finish this phase able to:**
> - Design systems that degrade gracefully under partial failure rather than cascading
> - Size timeouts, retries, circuit breakers, and bulkheads from first principles, not cargo-culted defaults
> - Calculate error budgets and availability targets from dependency trees
> - Run controlled chaos experiments to validate resilience hypotheses
> - Distinguish between transient, persistent, and gray failures in production telemetry
> - Implement adaptive load shedding and admission control under saturation

## Why this phase exists

In a monolith a slow database query is a slow web page. The user waits, the page renders, the connection closes. In a distributed system that same slow query becomes an exhausted thread pool, cascading timeouts across service boundaries, retries that amplify load 10×, circuit breakers tripping in a metastable failure mode that persists after the database recovers, and a 3 a.m. page for every team that touches the call path.

Distributed systems fail in ways monoliths never do. A network partition is not a crash—it's a half-alive system that looks healthy locally but cannot coordinate globally. A GC pause is not downtime—it's 500 ms of queued requests that all time out simultaneously when the service resumes. A configuration push is not a deployment—it's a synchronized thundering herd when 300 pods all reload their connection pools at the same instant.

This phase teaches you to design for the failure that **will** happen rather than the uptime you hope for. The goal is not to prevent failure—you cannot prevent a datacenter losing power, an upstream API returning 500s, or a query scanning 10 million rows. The goal is to contain failure: isolate it, limit its blast radius, degrade functionality gracefully, and recover automatically when the underlying fault clears.

## Mental model

**Reliability is not the absence of failure; it is the containment of failure.** A reliable system continues to serve traffic—perhaps degraded, perhaps slower, perhaps with reduced functionality—when dependencies fail. An unreliable system amplifies a single slow service into a complete outage across every caller.

**Every dependency is a liability you must be able to survive without**, at least temporarily. If the pricing service is down, ShopKart's catalog should serve product listings with stale prices or no prices, not return 500 to every user. If the recommendation engine times out, the homepage should fall back to a static featured list, not hang for 30 seconds waiting for personalization.

**Redundancy adds; dependencies multiply.** Serial dependencies multiply their failure rates: 10 services each at 99.9% availability compose to ~99.0% (3.6 days/year downtime) if any one failure kills the request. Redundant independent replicas add: three instances each at 99% compose to 99.9999% (32 seconds/year) if you tolerate two failures. The shape of your architecture determines whether nines add or subtract.

**Timeouts are not optional error handling; they are your last line of defense against infinite wait.** A missing timeout is a thread leak waiting to happen. A timeout without retry logic and a fallback is just a faster way to fail.

**What you do not exercise rots.** An untested circuit breaker never trips in production because the threshold was misconfigured. An untested DR failover does not work when you need it because the replica was not actually replicating for six months and no one noticed. Chaos engineering is not breaking things for fun—it is the only way to know your resilience patterns actually work.

## Core concepts

### Availability arithmetic and error budgets

Availability is the fraction of time a service successfully responds to requests. If you measure 43,195 successful requests and 5 errors in a five-minute window, your availability for that window is 43,195 / 43,200 ≈ 99.988%. Annualized, **99.9% ("three nines") = 8.76 hours/year downtime**; **99.99% ("four nines") = 52.6 minutes/year**; **99.999% ("five nines") = 5.26 minutes/year**.

**Serial dependencies multiply failure rates.** If ShopKart's order flow calls `payment` (99.9%), `inventory` (99.9%), `pricing` (99.9%), and `notification` (99.9%) sequentially, and any one failure aborts the order, the **end-to-end availability is 0.999⁴ ≈ 0.996 = 99.6%**, or 35 hours/year—four times worse than any individual service. Twenty services at 99.9% compose to 98.0% (7.3 days/year) if you require all twenty to succeed.

**Redundancy adds nines.** If you deploy three independent replicas and tolerate two failures (N=3, K=1 quorum), availability is 1 - (1 - 0.99)³ = 1 - 0.000001 = 99.9999% (five nines). Independence is critical—if replicas share a database, a power supply, or a config service, they fail together and you get no benefit.

**Error budgets make reliability measurable.** A 99.9% SLO means you have a budget of 43.2 minutes/month to spend on errors, downtime, or degradation. Spend it on controlled experiments, risky deployments, or dependency failures. When the budget is exhausted, stop deploying and harden. An error budget is not a target—it is the difference between the SLO you promise and the SLI you achieve.

```
SLO:    99.9% of requests succeed within 500 ms (43.2 min error budget/month)
SLI:    Measured availability this month: 99.93% (30.2 min consumed)
Budget: 13.0 minutes remaining → safe to deploy; 0 minutes → feature freeze, focus on reliability
```

Google's SRE book canonicalized this framing. Error budgets align business and engineering: product managers want features; SREs want stability; the error budget is the currency that balances both.

### Failure taxonomy

Not all failures look the same in production. Knowing the difference changes how you detect, mitigate, and recover.

**Crash failure:** the service stops responding entirely. Kubernetes detects it via liveness probe, restarts the pod, load balancer removes it from rotation. Easy to detect, easy to handle, rare in mature systems.

**Slow failure:** the service responds, but 10× slower than normal. p99 latency spikes from 50 ms to 5 seconds. Your timeouts were set to 3 seconds so requests queue, threads exhaust, upstream callers time out, retries amplify load. Slow failure is more dangerous than crash failure because it looks alive to basic health checks.

**Partial failure:** the service is healthy for 95% of requests and broken for 5%. A shard key hash collision means one specific user ID always hits the corrupted replica; everyone else works fine. Aggregate metrics look normal. The angry user's ticket sits in support for a week.

**Gray failure:** the service appears healthy to its own health check but is unreachable or degraded from the caller's perspective. Happens with asymmetric network partitions, DNS propagation lag, split-brain scenarios, or when the health check does not exercise the actual request path.

**Plain English:** Gray failure is when a system lies to you about being healthy.

**Analogy:** A restaurant's phone line is working—you can call and the host answers—but the kitchen is on fire and no food is coming out. The host does not know the kitchen is broken because they are in the front of house. To a caller making a reservation the restaurant seems open; to a diner waiting for food it is closed.

**In the real world:** A Kubernetes pod passes its readiness check (HTTP 200 on `/actuator/health`) because Spring Boot reports "UP", but the outbound network policy silently drops all egress traffic to the database. The pod stays in rotation, requests come in, every database query times out, the service returns 500s, but Kubernetes never removes it because `/actuator/health` never fails.

**Mechanics:** Gray failures happen when your health check is not on the critical path. If the readiness probe calls a `/health` endpoint that returns a static response or only checks the JVM is alive, it will never detect a downstream dependency failure, a network partition, or a corrupted local cache. The probe must exercise the actual failure domain you care about—but if it exercises deep dependencies (calls the database), it becomes a thundering herd that kills the database during a blip.

**What breaks:** Symptoms: steady stream of 500s in logs, caller-side timeouts, elevated p99, but the pod is not restarting and Kubernetes metrics show all replicas "ready". Detection: compare service-reported health with caller-observed error rate. Fix: shallow readiness (JVM liveness), deep periodic probes logged but not used for routing, or circuit-breaker-aware readiness that sheds traffic when downstream is failing.

**Dependency degradation:** the upstream API is up, but rate-limiting you, returning partial data, or serving from a stale cache. You retry, amplify load, get throttled harder. Requires adaptive backoff and fallback logic.

**Correlated failure:** multiple independent-looking services fail simultaneously because they share a hidden dependency—a config service, a DNS server, a certificate expiration, a shared database connection pool. Correlated failure turns "redundancy adds nines" into "redundancy is an illusion."

**Metastable failure:** the system has two equilibrium states—healthy and collapsed—and once it flips into the collapsed state it stays there even after the triggering fault clears. 

**Plain English:** Metastable failure is a system that breaks, does not unbreak itself when you fix the cause, and requires manual intervention to recover even though the original problem is gone.

**Analogy:** Imagine a river with a narrow bridge. Normally, traffic flows. One day a truck breaks down on the bridge, traffic backs up for miles. Eventually the truck gets towed away—the bridge is now empty and clear—but drivers, seeing the jam on their GPS, take an alternate route. The bridge stays empty for hours. The equilibrium flipped from "flowing" to "avoided" and will not flip back without active intervention (signs, traffic police, time).

**In the real world:** Netflix publicly described a metastable failure in their SPS (subscription processing service). Under normal load SPS was fast. A downstream service became slow, SPS requests queued, memory filled with queued work, GC pauses increased, throughput dropped further, the queue grew larger, GC paused longer—even after the downstream service recovered, SPS stayed in a degraded state with high GC overhead processing the backlog until they manually drained the queue.

**Mechanics:** Metastable failure requires a positive feedback loop. Example: elevated latency → retry storms → higher load → more latency → more retries. Or: queue buildup → GC pressure → slower processing → longer queues → more GC. The system cannot self-recover because the recovery path (draining the queue, reducing GC) requires spare capacity the system no longer has.

**What breaks:** Metrics show the original fault is resolved but throughput remains depressed. Memory or CPU is saturated. Restarting pods "fixes" it—until the next trigger. Prevention: admission control, bounded queues with overflow shedding, retry budgets, load shedding before saturation. Detection: compare current throughput to historical baseline under similar load. Recovery: force a state reset—restart, drain the queue, shed load.

### Timeouts: the full chain

A timeout is not "how long I am willing to wait." A timeout is "how long I am willing to let my threads sit idle before I fail fast and attempt recovery."

**How to choose:** Start with the callee's p99.9 latency from production telemetry. Add network latency (typically 1–5 ms intra-cluster, 20–100 ms inter-region). Add buffer (20–50%). Round up, not down. If the service's p99.9 is 200 ms, your timeout should be 300–400 ms, not 1000 ms and definitely not 30 seconds.

**Why not a round number?** Because "5 seconds" or "30 seconds" is a guess disconnected from reality. It is either much too generous (waste threads waiting for something that is clearly stuck) or much too aggressive (kill requests that are legitimately slow under load). Measure first, then decide.

**The full timeout chain:**

1. **Connection timeout:** how long to wait for TCP handshake. Typical: 2–5 seconds. Too short and you fail on transient network blips; too long and a dead service holds threads.
2. **TLS timeout:** handshake overhead. Usually same as connection timeout.
3. **Read timeout:** how long to wait for the first byte of response body after request sent. This is your main application timeout. Set it to callee p99.9 + buffer.
4. **Total timeout (deadline):** absolute upper bound for the entire request lifecycle. If read timeout is 500 ms, total might be 1 second to account for retries.
5. **Connection pool acquisition timeout:** how long to wait for a connection from the pool before failing. If this is infinite, a saturated pool blocks threads forever.

```yaml
# application.yaml: Spring Boot RestClient timeout configuration
spring:
  threads:
    virtual:
      enabled: true  # Reduces thread exhaustion risk but does not eliminate need for timeouts

resilience4j:
  timelimiter:
    instances:
      paymentService:
        timeoutDuration: 800ms  # From payment p99.9 = 600ms + 200ms buffer
        cancelRunningFuture: true
```

```java
// Spring Boot 4.x RestClient with full timeout chain
@Configuration
public class HttpClientConfig {
    
    @Bean
    public RestClient paymentClient(RestClient.Builder builder) {
        var clientConfig = ClientHttpRequestFactorySettings.DEFAULTS
            .withConnectTimeout(Duration.ofSeconds(3))
            .withReadTimeout(Duration.ofMillis(800));  // Our calculated timeout
        
        return builder
            .baseUrl("http://payment-service")
            .requestFactory(new JdkClientHttpRequestFactory(
                HttpClient.newBuilder()
                    .connectTimeout(Duration.ofSeconds(3))
                    .build()))
            .defaultStatusHandler(HttpStatusCode::is5xxServerError,
                (request, response) -> {
                    throw new PaymentServiceException("Payment service error: " + 
                        response.getStatusCode());
                })
            .build();
    }
}
```

**Deadline propagation:** If the catalog service has a 1-second timeout for the user request, and it calls pricing (500 ms timeout) which calls the database (200 ms timeout), each layer must subtract elapsed time from the deadline and pass the remainder downstream. Otherwise the database query gets 200 ms even though the user request has 50 ms remaining.

gRPC and OpenTelemetry support deadline propagation natively. HTTP requires custom headers (`X-Deadline`, `grpc-timeout`). Without it you get "timeout inversion"—a downstream service spending time on work the upstream caller already abandoned.

**The rule:** A timeout without a retry policy and a fallback is just a faster failure. You have failed fast—now what? Return an error to the user? Retry? Serve stale data? The timeout is the detection mechanism, not the recovery mechanism.

### Retries: idempotency, jitter, and budgets

Retries turn transient failures into successes and permanent failures into retry storms that amplify load 10×.

**Only retry idempotent operations.** If retrying might double-charge a credit card, do not retry. If you must retry non-idempotent writes, use idempotency tokens: client generates a UUID, server deduplicates by token.

**Exponential backoff with full jitter:**

```
base_delay = 100ms
max_delay = 30s
attempt = 0, 1, 2, 3, ...

# Exponential backoff without jitter (WRONG)
delay = min(base_delay * 2^attempt, max_delay)
# Result: 100ms, 200ms, 400ms, 800ms, 1600ms, 3200ms, ...
# Problem: Every client retries at exactly 100ms, 200ms, 400ms → thundering herd

# Full jitter (CORRECT)
temp = min(base_delay * 2^attempt, max_delay)
delay = random(0, temp)
# Result: random between 0–100ms, 0–200ms, 0–400ms, ...
# Effect: Spreads retries over time, breaks synchronization
```

Why full jitter beats "backoff without jitter": when 1,000 clients all fail at the same instant (deployment, network blip, service restart), exponential backoff without jitter means they all retry at T+100ms, overwhelming the service again. Jitter desynchronizes the retry wave.

AWS published a study showing full jitter completes requests ~30% faster than non-jittered backoff under load. Resilience4j defaults to full jitter.

**Retry budgets:** Only retry if recent requests are succeeding. Idea from Google SRE: track a success rate over a sliding window (e.g., last 100 requests). If success rate < 90%, stop retrying—the service is genuinely down and retries will only make it worse. Resilience4j does not implement this natively; you must track it with custom metrics.

**Token bucket for retries:**

```java
// Custom retry budget enforcer
public class RetryBudget {
    private final AtomicInteger successCount = new AtomicInteger(0);
    private final AtomicInteger totalCount = new AtomicInteger(0);
    private final double threshold = 0.9;  // 90% success required
    
    public void recordSuccess() {
        successCount.incrementAndGet();
        totalCount.incrementAndGet();
    }
    
    public void recordFailure() {
        totalCount.incrementAndGet();
    }
    
    public boolean allowRetry() {
        int total = totalCount.get();
        if (total < 10) return true;  // Not enough samples
        double rate = (double) successCount.get() / total;
        
        // Reset every 100 requests to keep window fresh
        if (total > 100) {
            totalCount.set(0);
            successCount.set(0);
        }
        
        return rate >= threshold;
    }
}
```

**Which failures to retry:**

- **Yes:** 503 Service Unavailable, 429 Too Many Requests (with backoff respecting `Retry-After`), network timeout, connection refused, DNS resolution failure (transient).
- **No:** 400 Bad Request (client error, will not improve), 401/403 (auth error), 404 (data not found), 409 Conflict, 5xx on non-idempotent writes.
- **Maybe:** 500 Internal Server Error on reads—if the service is overloaded 500 means "I am saturated, do not retry immediately"; if it is a bug 500 is permanent.

**Retry amplification across layers:** If the API gateway retries 3 times, the backend service retries 3 times, and the database client retries 3 times, a single user request can become 3 × 3 × 3 = 27 database queries. Each layer must know the total budget and subtract its consumption. Or: only the outermost layer retries, inner layers fail fast.

**Hedged requests (tail-at-scale):** For latency-critical reads, issue the same request to two replicas simultaneously and use whichever responds first. Cuts p99 latency significantly (Google reported 40%+ reduction in some services) but doubles load. Only viable if you have spare capacity and the operation is read-only and idempotent.

```java
// Hedged request example
public CompletableFuture<Product> getProductHedged(String productId) {
    var request1 = callReplica1Async(productId);
    var request2 = CompletableFuture.delayedExecutor(50, TimeUnit.MILLISECONDS)
        .execute(() -> callReplica2Async(productId));
    
    return CompletableFuture.anyOf(request1, request2)
        .thenApply(result -> (Product) result);
}
// After 50ms if replica1 hasn't responded, send to replica2.
// Return first success. Cancel the slower request.
```

### Circuit breaker: states, thresholds, and half-open probes

A circuit breaker protects **your threads**, not the failing service. When a dependency is down, stop calling it so your thread pool does not exhaust waiting for timeouts.

**States:**

1. **Closed (normal):** Requests pass through. Failures are counted.
2. **Open (tripped):** All requests fail immediately with `CallNotPermittedException`. No traffic to the failing service. Lasts for a configured duration (`waitDurationInOpenState`).
3. **Half-Open (probing):** After the wait duration, allow a few test requests (`permittedNumberOfCallsInHalfOpenState`) through to see if the service recovered. If they succeed, transition to Closed. If they fail, return to Open.

**Sliding window:** Track recent requests in a window (count-based or time-based).

- **Count-based:** last N requests (e.g., 100). When failures / total > threshold, trip. Simple, but bursty—one bad second can look fine if averaged over the last 100 requests.
- **Time-based:** last T seconds (e.g., 60s). More stable under variable load. Resilience4j default.

**Thresholds:**

```yaml
resilience4j:
  circuitbreaker:
    instances:
      pricingService:
        slidingWindowType: TIME_BASED
        slidingWindowSize: 60                # 60-second window
        minimumNumberOfCalls: 10             # Need at least 10 calls before evaluating
        failureRateThreshold: 50             # Trip if >50% fail
        slowCallRateThreshold: 80            # Also trip if >80% are slow
        slowCallDurationThreshold: 2s        # "Slow" = >2s
        waitDurationInOpenState: 30s         # Stay open for 30s before half-open
        permittedNumberOfCallsInHalfOpenState: 5  # Test with 5 requests
        automaticTransitionFromOpenToHalfOpenEnabled: true
        recordExceptions:
          - java.net.ConnectException
          - java.util.concurrent.TimeoutException
        ignoreExceptions:
          - com.shopkart.BusinessValidationException  # Not a failure
```

**Common misconfigurations:**

- `minimumNumberOfCalls` too high → circuit never trips during low traffic.
- `failureRateThreshold` too high (e.g., 90%) → you exhaust threads before it trips.
- `waitDurationInOpenState` too short (e.g., 1s) → constant open/half-open/open flapping.
- `permittedNumberOfCallsInHalfOpenState` = 1 → a single slow probe keeps you open forever.

**What it protects:** Your thread pool from blocking on a dead service. Does NOT protect the downstream service—if it is drowning, your circuit breaker opening reduces load, but that is a side effect, not the design goal.

**What breaks:** If every pod has an independent circuit breaker and they trip at different times, you get inconsistent behavior—some pods return errors, some forward requests. Solution: coordinated circuit breaking (share state in Redis) or accept the inconsistency as temporary during failure.

```java
// Spring Boot integration
@Service
public class PricingService {
    
    private final RestClient pricingClient;
    private final CircuitBreakerRegistry circuitBreakerRegistry;
    
    @CircuitBreaker(name = "pricingService", fallbackMethod = "getPriceFallback")
    @TimeLimiter(name = "pricingService")
    public CompletableFuture<Price> getPrice(String productId) {
        return CompletableFuture.supplyAsync(() -> 
            pricingClient.get()
                .uri("/prices/{id}", productId)
                .retrieve()
                .body(Price.class)
        );
    }
    
    private CompletableFuture<Price> getPriceFallback(String productId, Throwable t) {
        log.warn("Pricing service unavailable for product {}, using fallback", productId, t);
        return CompletableFuture.completedFuture(
            priceCache.getOrDefault(productId, Price.unavailable())
        );
    }
}
```

### Bulkhead: thread-pool and semaphore isolation

Bulkhead pattern: isolate resources so one failing dependency cannot exhaust the entire pool.

**Thread-pool bulkhead:** Each dependency gets a dedicated thread pool. If the payment service hangs, only the payment pool exhausts; catalog and search remain healthy.

```yaml
resilience4j:
  thread-pool-bulkhead:
    instances:
      paymentService:
        maxThreadPoolSize: 10
        coreThreadPoolSize: 5
        queueCapacity: 20  # Reject if >20 queued
        keepAliveDuration: 20s
```

**Semaphore bulkhead:** Limit concurrent calls without dedicated threads. Cheaper (no thread overhead) but less isolation—slow calls block the caller's thread.

```yaml
resilience4j:
  bulkhead:
    instances:
      searchService:
        maxConcurrentCalls: 25
        maxWaitDuration: 100ms  # Wait up to 100ms for a permit
```

**When thread-pool vs semaphore:**

- Thread-pool: external I/O (HTTP, database), where you want complete isolation and can afford thread overhead.
- Semaphore: in-process calls, virtual threads, or high-frequency paths where thread-pool overhead is prohibitive.

**Connection-pool bulkhead (the one people forget):** Even if you isolate threads, a single shared connection pool means one slow query can exhaust all database connections. Use per-service HikariCP pools or per-tenant schemas.

```yaml
# Separate HikariCP pool per service
spring:
  datasource:
    payment:
      jdbc-url: jdbc:postgresql://payment-db:5432/payments
      hikari:
        maximum-pool-size: 20
        minimum-idle: 5
        connection-timeout: 5000
        pool-name: payment-pool
    inventory:
      jdbc-url: jdbc:postgresql://inventory-db:5432/inventory
      hikari:
        maximum-pool-size: 15
        pool-name: inventory-pool
```

**What it prevents:** Payment service times out → all payment threads block → API gateway cannot forward any requests → entire system down. With bulkhead: payment threads exhaust, payment returns errors, but search and catalog continue serving traffic.

### Rate limiting and throttling

**Token bucket:**

```
capacity = 100 tokens
refill_rate = 10 tokens/second

if tokens_available > 0:
    tokens_available -= 1
    allow_request()
else:
    reject_with_429()

# Refill: every 100ms, add 1 token (10/sec), max 100
```

Allows bursts up to capacity, then throttles to refill rate. Most common algorithm. Used by AWS, Google Cloud, Stripe.

**Leaky bucket:**

```
queue_capacity = 100
drain_rate = 10 requests/second

if queue.size < capacity:
    queue.add(request)
else:
    reject_with_429()

# Drain: every 100ms, process 1 request from queue
```

Smooths bursts into a steady outflow. Harder to implement correctly (need background thread draining queue). Less common in practice.

**Fixed window:**

```
window = current_minute
counter[window] += 1
if counter[window] > limit:
    reject_with_429()
```

Simple but allows 2× bursts at window boundaries. If limit is 100/min, you can send 100 requests at 12:00:59 and 100 more at 12:01:00 → 200 requests in 2 seconds.

**Sliding window log:**

```
timestamps = deque()
timestamps.append(now())
# Remove all entries older than window
while timestamps and timestamps[0] < now() - window_duration:
    timestamps.popleft()
if len(timestamps) > limit:
    reject_with_429()
```

Accurate but memory-heavy (stores every request timestamp).

**Sliding window counter (best of both):**

```
current_window_count = counter[current_minute]
previous_window_count = counter[previous_minute]
elapsed = seconds_into_current_minute

estimated = previous_window_count * (1 - elapsed/60) + current_window_count
if estimated > limit:
    reject_with_429()
```

Approximates sliding window log with O(1) memory. Used by Redis rate limiters. Close enough to exact for production.

| Algorithm         | Burst tolerance | Accuracy | Memory | Typical use           |
|-------------------|-----------------|----------|--------|-----------------------|
| Token bucket      | High            | Good     | O(1)   | API gateways, AWS     |
| Leaky bucket      | None (smoothed) | Exact    | O(N)   | Traffic shaping       |
| Fixed window      | 2× at boundary  | Poor     | O(1)   | Simple counters       |
| Sliding log       | Low             | Exact    | O(N)   | Audit logs            |
| Sliding counter   | Medium          | Good     | O(1)   | Redis-based limiters  |

**Distributed rate limiting with Redis + Lua atomicity:**

```lua
-- rate_limit.lua (loaded into Redis)
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local current = redis.call('INCR', key)
if current == 1 then
    redis.call('EXPIRE', key, window)
end
if current > limit then
    return 0  -- Rejected
else
    return 1  -- Allowed
end
```

```java
@Service
public class RedisRateLimiter {
    
    private final RedisTemplate<String, String> redis;
    private final RedisScript<Long> script;
    
    public boolean allowRequest(String clientId, int limit, int windowSeconds) {
        String key = "rate_limit:" + clientId + ":" + Instant.now().getEpochSecond() / windowSeconds;
        Long result = redis.execute(script, List.of(key), limit, windowSeconds);
        return result != null && result == 1;
    }
}
```

**Per-tenant and per-key quotas:** Different limits for free vs paid users, by API key, by IP. Store limits in a config service or database, key by tenant ID.

**Fair queuing:** When multiple tenants share a service, process requests in round-robin across tenants so one heavy tenant cannot starve others. Requires separate queues per tenant—complex, usually done in the API gateway or service mesh.

**429 + Retry-After semantics:**

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 30
Content-Type: application/json

{"error": "Rate limit exceeded", "retryAfter": 30}
```

Clients must respect `Retry-After`. If they do not, rate limiting turns into a retry storm.

**Where to enforce:**

- **Edge (API gateway):** Protects the entire backend, prevents abuse, but does not handle internal inter-service limits.
- **Gateway (per service ingress):** Service-specific limits, useful for multi-tenant systems.
- **Service:** Application-layer logic, can enforce business rules ("max 5 orders/minute per user").
- **Database:** Last line of defense. PostgreSQL `pg_bouncer` connection limits, MySQL `max_user_connections`. Prevents runaway queries from bringing down the DB.

### Load shedding and admission control

**Load shedding:** Intentionally drop requests before the system collapses. Better to reject 10% of traffic cleanly than to fail 100% of traffic with timeouts and retries.

**Shed before you collapse:** Monitor CPU, memory, queue depth, or active requests. When saturation nears (e.g., 80% CPU), start rejecting low-priority traffic with 503.

**Prioritized shedding:** Drop lowest-priority work first.

1. Batch jobs, analytics, background exports
2. Free-tier users
3. Non-critical features (recommendations, A/B tests)
4. Critical reads (product catalog, search)
5. Critical writes (checkout, payment)

**Queue timeouts and CoDel:** Track time-in-queue for each request. If a request has been waiting >500 ms and the service is saturated, reject it immediately—it is probably already past the client's timeout. This is the CoDel (Controlled Delay) algorithm from networking applied to request queues. Prevents queue buildup from turning into metastable failure.

```java
public class QueueTimeoutFilter implements Filter {
    
    private static final Duration MAX_QUEUE_TIME = Duration.ofMillis(500);
    private final Meter.Counter queueTimeoutCounter;
    
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) {
        request.setAttribute("enqueueTime", Instant.now());
        
        Instant enqueued = (Instant) request.getAttribute("enqueueTime");
        Duration queued = Duration.between(enqueued, Instant.now());
        
        if (queued.compareTo(MAX_QUEUE_TIME) > 0) {
            queueTimeoutCounter.increment();
            ((HttpServletResponse) response).setStatus(503);
            return;  // Drop request
        }
        
        chain.doFilter(request, response);
    }
}
```

**Adaptive concurrency limits (Netflix gradient/AIMD):** Instead of fixed thread pools, dynamically adjust the concurrency limit based on observed latency. If latency increases, reduce limit (additive decrease); if latency is stable and success rate is high, increase limit (multiplicative increase). Prevents thread exhaustion during slowdowns.

**Plain English:** The service watches its own latency. When requests start slowing down, it stops accepting new work until the current work finishes faster.

**Analogy:** A restaurant kitchen has a capacity—say, 20 orders at once. If they accept 30 orders and the kitchen gets backed up, every dish takes longer. Smart approach: when ticket times start rising, the host stops seating new tables until the kitchen catches up.

**In the real world:** Netflix's SPS (Subscription Processing Service) used this. Under normal load it accepted 200 concurrent requests. When latency spiked (database slow, GC pause), the limit dropped to 50, rejecting new work with 503 until latency returned to normal. Prevented cascading failure.

**Mechanics (simplified gradient algorithm):**

```
current_limit = 100
min_limit = 10
max_limit = 500
latency_p99 = measure_p99_latency()

if latency_p99 > target_latency * 1.2:  # 20% over target
    current_limit = max(min_limit, current_limit * 0.9)  # Reduce by 10%
elif latency_p99 < target_latency and success_rate > 0.95:
    current_limit = min(max_limit, current_limit + 1)  # Increase slowly

if active_requests >= current_limit:
    reject_with_503()
```

**What breaks:** If the limit drops too fast, you shed too much load and underutilize capacity. If it increases too fast, you re-saturate before recovery. Tuning requires production telemetry. Libraries: Netflix Concurrency Limits (Java), Envoy adaptive concurrency filter.

### Graceful degradation and fallbacks

When a dependency fails, **degrade gracefully** rather than failing completely.

**Static fallbacks:** Return a hardcoded default. ShopKart recommendations fail → show a curated "featured products" list.

**Stale-cache fallbacks:** If the pricing service is down, serve prices from the cache even if expired. Better to show a 10-minute-old price than no price.

**Reduced functionality modes:** Payment gateway down → allow "save for later" and email the cart link; do not allow checkout. Notification service down → complete the order, log the notification failure, retry async.

**Feature-flag kill switches:** A/B test causing load spikes → kill switch disables the test, falls back to baseline experience.

**The critical rule:** A fallback that silently returns wrong data for money or inventory is worse than an error. If you cannot guarantee correctness, return an explicit error. Do not charge a credit card with a cached price that is off by 50%. Do not confirm an order if inventory is uncertain. Fail safe.

```java
@Service
public class ProductService {
    
    private final RestClient catalogClient;
    private final Cache<String, Product> productCache;
    
    public Product getProduct(String productId) {
        try {
            Product fresh = catalogClient.get()
                .uri("/products/{id}", productId)
                .retrieve()
                .body(Product.class);
            productCache.put(productId, fresh);
            return fresh;
        } catch (Exception e) {
            log.warn("Catalog service unavailable, serving stale product {}", productId, e);
            Product stale = productCache.getIfPresent(productId);
            if (stale != null) {
                return stale.withWarning("Price may be outdated");
            }
            throw new ProductUnavailableException("Catalog service down and no cached data");
        }
    }
}
```

### Health checks: liveness, readiness, and the dependency trap

Kubernetes probes:

- **Liveness:** "Is the process alive?" Failure → restart the pod. Should check JVM responsiveness, not dependencies.
- **Readiness:** "Can this pod serve traffic?" Failure → remove from load balancer. Can check critical dependencies, but carefully.
- **Startup:** "Has the process finished initializing?" Failure → keep retrying; success → switch to liveness/readiness. Used for slow-starting JVMs.

**The cardinal sin:** Liveness probe that calls a dependency. If the database blips, every pod's liveness probe fails, Kubernetes restarts the entire fleet simultaneously, thundering herd kills the database on restart. **Liveness must never check dependencies.**

**Readiness that sheds traffic correctly:**

```yaml
# Shallow readiness: only checks if the app can handle requests
management:
  endpoint:
    health:
      probes:
        enabled: true
      show-details: never
  health:
    livenessState:
      enabled: true
    readinessState:
      enabled: true

# Kubernetes probes
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
```

**Deep vs shallow checks:**

- **Shallow:** JVM is alive, HTTP server is responsive. Fast, no thundering herd risk.
- **Deep:** Can connect to database, can reach Redis, circuit breakers are not open. Accurate but risky—if 50 pods all health-check the database every 5 seconds and it hiccups, you amplify load 50×.

**Hybrid approach:** Readiness does shallow checks plus circuit-breaker state. If the circuit breaker to the database is open, readiness fails → pod is removed from rotation → traffic flows to healthy pods → the circuit breaker eventually closes → readiness passes → pod returns to rotation. The circuit breaker does the deep checking on actual request traffic; readiness just reads the state.

**Health endpoints as an attack surface:** If `/actuator/health` is publicly exposed and returns detailed dependency status, an attacker learns your topology and can DoS a critical dependency to cascade failures. Either:

- Keep health internal (Kubernetes service mesh only).
- Return minimal info on the public endpoint (`{"status": "UP"}`), detailed info on a separate internal-only endpoint.

### Queueing theory and the utilization wall

**Little's Law:** `L = λ × W`

- L = average number of requests in the system (queue + processing)
- λ = arrival rate (requests/second)
- W = average time in system (latency)

If you are processing 100 req/s at 50 ms average latency, you have L = 100 × 0.05 = 5 requests in-flight. If latency jumps to 500 ms (10× slower), you need 100 × 0.5 = 50 in-flight capacity. If your thread pool is 20, requests queue, latency increases further, positive feedback loop.

**The utilization wall:** As CPU/thread pool utilization approaches 100%, latency spikes exponentially. Queueing theory (M/M/1 model): average wait time = (ρ / (1 - ρ)) × service_time, where ρ = utilization.

| Utilization | Queue multiplier | Latency impact            |
|-------------|------------------|---------------------------|
| 50%         | 1×               | Baseline                  |
| 75%         | 3×               | Noticeable                |
| 90%         | 9×               | Degraded                  |
| 95%         | 19×              | User-impacting            |
| 99%         | 99×              | Unusable (metastable)     |

**Why adding threads to a saturated system makes latency worse:** More threads → more context switching → less CPU for actual work → slower processing → longer queues → higher latency. The optimal pool size is usually `cores × (1 + wait_time / compute_time)`. For I/O-bound (high wait/compute ratio), you can afford more threads. For CPU-bound, more threads than cores is counterproductive. Virtual threads reduce context-switch overhead but do not eliminate queuing effects.

**How to stay below the wall:** Autoscale before 70–80% utilization, shed load above it, or use adaptive concurrency limits.

### Chaos engineering

**Hypothesis-driven experiments.** Not "let's break things and see what happens." Formulate a hypothesis: "If we terminate one database replica, the application continues serving reads with <100 ms p99 increase because the connection pool fails over to the remaining replicas within 5 seconds."

**Steady-state definition:** Define observable metrics that represent "healthy"—p99 latency <200 ms, error rate <0.1%, throughput >1000 req/s. The experiment succeeds if steady state is maintained during the fault injection.

**Blast radius control:** Start small (one pod in staging), increase gradually (one AZ in production during low traffic), never run untested experiments in production at peak load.

**Game days:** Scheduled chaos exercises where the whole team watches. Inject failures, observe detection/alerting/recovery, document surprises, fix gaps. Amazon runs annual "Game Day" events where they intentionally fail datacenters.

**Tooling:**

- **Chaos Mesh** (Kubernetes): inject pod kill, network delay, disk I/O errors, DNS failures via CRDs.
- **Litmus Chaos** (Kubernetes): similar to Chaos Mesh, more focused on SRE workflows.
- **Toxiproxy:** TCP proxy that injects latency, connection drops, bandwidth limits. Run it as a sidecar in tests.
- **AWS Fault Injection Simulator:** managed chaos for AWS services (terminate EC2, throttle API calls, fail AZ).
- **Service mesh fault injection (Istio):** inject HTTP delays, aborts, header manipulation via VirtualService config.

**Organizational prerequisites:** You may not run chaos experiments until you have:

1. Observability to detect the failure (metrics, logs, traces).
2. Runbooks to respond.
3. Rollback capability (feature flags, deployment rollback).
4. Management buy-in (chaos is not cowboy culture, it is disciplined risk reduction).

Chaos without observability is just breaking things. Chaos without rollback is gambling.

### Disaster recovery: RTO, RPO, and the uncomfortable truth

**RTO (Recovery Time Objective):** How long can you tolerate downtime? 1 hour? 4 hours? 24 hours?

**RPO (Recovery Point Objective):** How much data loss is acceptable? 0 (zero data loss)? 5 minutes? 1 hour?

RTO and RPO determine architecture:

| RTO      | RPO      | Strategy                                 | Cost   |
|----------|----------|------------------------------------------|--------|
| 24 hrs   | 4 hrs    | Backup/restore from snapshots            | Low    |
| 4 hrs    | 1 hr     | Warm standby (replica in another region) | Medium |
| 1 hr     | 15 min   | Active-passive with fast failover        | High   |
| Seconds  | 0        | Active-active multi-region               | Very high |

**Backup vs replication vs DR:**

- **Backup:** Snapshot to S3, restore when needed. RPO = backup frequency. RTO = restore time (hours).
- **Replication:** Database continuously replicates to a standby. RPO = replication lag (seconds to minutes). RTO = failover time (minutes to hours, depending on automation).
- **DR (Disaster Recovery):** Full duplicate environment in another region, always ready. RPO near-zero (if replicating). RTO depends on failover mechanism.

**Multi-AZ vs multi-region:**

- **Multi-AZ:** Availability zones in the same region. Protects against datacenter failure, not regional disasters (hurricane, power grid, AWS region outage). RTO: minutes (automatic failover). RPO: zero (synchronous replication within region is fast).
- **Multi-region:** Protects against regional disasters. RTO: higher (DNS cutover, database promotion). RPO: higher (cross-region replication lag). Cost: 2× infrastructure.

**Active-active vs active-passive:**

- **Active-active:** Traffic routed to both regions. Failover is instant (just remove failed region from DNS/load balancer). Requires conflict-free data replication (CRDTs, last-write-wins, or partitioned by region). Expensive, complex.
- **Active-passive:** Primary region serves traffic, secondary is idle standby. Failover: promote secondary, update DNS. Cheaper, simpler, but slower failover.
- **Pilot light:** Minimal infrastructure running in secondary (database replica only), scale up on failover. Cheaper than full standby, slower than active-passive.

**Data-layer constraints are the real limiter:** You can run stateless services in multiple regions trivially. Databases are hard. PostgreSQL streaming replication cross-region has lag (50–200 ms typical). If you promote the replica during an outage, you might lose the last few seconds of writes. Synchronous cross-region replication is slow (adds RTT to every write). Distributed databases (CockroachDB, Spanner) solve this with consensus at a latency and complexity cost.

**Failover testing:** If you have never tested failover, it does not work. Schedule quarterly DR drills: shut down the primary region, promote the replica, verify the application works, fail back. Common discoveries during drills:

- Replica was not actually replicating for six months (monitoring gap).
- DNS TTL was set to 1 hour, failover took 1 hour to propagate.
- Hardcoded primary endpoint in config, application could not reach the replica.
- Prometheus/Grafana was in the failed region, no metrics during failover.

**The uncomfortable truth:** Untested DR is not DR—it is disaster theater. If the runbook has not been executed in production-like conditions in the last 6 months, assume it is broken.

### Blast radius reduction: cells and shuffle sharding

**Cell-based architecture:** Partition the infrastructure into isolated cells, each serving a subset of users. A failure in one cell affects only that cell's users.

Example: ShopKart has 10 million users. Instead of one giant deployment, deploy 10 cells of 1 million users each. Users are hashed to a cell by user ID. Cell 3 fails → 1 million users affected, 9 million unaffected.

**Shuffle sharding (AWS framing):** Instead of a single cell per user, each user is assigned to multiple independent cells (shards), chosen pseudo-randomly. A failure only affects users whose entire shard set failed.

**Plain English:** Instead of putting all your eggs in one basket, give each customer a random mix of several baskets. A basket breaking only hurts customers whose other baskets also broke—which is statistically rare if baskets fail independently.

**In the real world:** AWS Route 53 uses shuffle sharding for name servers. Each hosted zone is assigned 4 name servers chosen from a pool of hundreds. For two customers to both fail, all 4 of customer A's name servers AND all 4 of customer B's name servers must fail. If name servers fail independently at 1% rate, the odds of all 4 failing is 0.01⁴ = 0.0001% = 1 in 100 million.

**Mechanics (worked probability example):**

```
Total shards: 100
Shards per customer: 8 (randomly chosen from the 100)
Shard failure rate: 1% (1 out of 100 shards fails)

Probability that all 8 of a customer's shards fail: 0.01⁸ ≈ 10⁻¹⁶ (effectively zero)
Probability that a customer is affected if ANY of their 8 shards fails: ~8% (acceptable)

Compare to static assignment (1 shard per customer):
If a shard fails, 100% of its customers are down.

Shuffle sharding spreads risk: minor degradation for many, catastrophic failure for almost none.
```

**Implementation:**

```java
public class ShuffleShardRouter {
    
    private final List<String> shardEndpoints;  // 100 endpoints
    private final int shardsPerCustomer = 8;
    
    public List<String> getShardsForCustomer(String customerId) {
        // Seed RNG with customer ID for deterministic assignment
        Random rng = new Random(customerId.hashCode());
        
        return rng.ints(0, shardEndpoints.size())
            .distinct()
            .limit(shardsPerCustomer)
            .mapToObj(shardEndpoints::get)
            .toList();
    }
    
    public String routeRequest(String customerId, int attemptNumber) {
        List<String> shards = getShardsForCustomer(customerId);
        // Round-robin or random across the customer's assigned shards
        return shards.get(attemptNumber % shards.size());
    }
}
```

**Static stability:** The system does not depend on a control plane to route during an outage. If the shard assignment service is down, the client can recompute shard assignments locally using the same hash function. Netflix's EVCache clients embed this logic—no coordinator needed.

## Production patterns

### Pattern: Circuit breaker with fallback chain

**What:** Wrap calls to unreliable dependencies in a circuit breaker; when open, execute fallback logic (cache, static default, degraded mode) instead of failing immediately.

**When to use:**

- External HTTP APIs (payment gateways, third-party data providers).
- Internal services that have occasional instability.
- Any synchronous dependency where a failure should not propagate as an error to the user.

**When NOT to use:**

- Database calls within a transaction (circuit breaker cannot help—transaction must succeed or roll back).
- Critical writes where a fallback would silently corrupt data (payment processing, inventory deduction).

**Failure modes:**

- Circuit trips too easily → false positives, unnecessary degradation.
- Fallback is stale/incorrect → users see wrong data, support tickets.
- Circuit never closes → manual intervention required, system stays degraded.

**Code:**

```yaml
# application.yaml
resilience4j:
  circuitbreaker:
    instances:
      recommendationService:
        slidingWindowSize: 100
        failureRateThreshold: 50
        waitDurationInOpenState: 30s
        permittedNumberOfCallsInHalfOpenState: 10
  timelimiter:
    instances:
      recommendationService:
        timeoutDuration: 1s
```

```java
@Service
public class RecommendationService {
    
    private final RestClient recommendationClient;
    private final Cache<String, List<Product>> recommendationCache;
    
    @CircuitBreaker(name = "recommendationService", fallbackMethod = "getRecommendationsFallback")
    @TimeLimiter(name = "recommendationService")
    public CompletableFuture<List<Product>> getRecommendations(String userId) {
        return CompletableFuture.supplyAsync(() -> {
            var response = recommendationClient.get()
                .uri("/recommendations/{userId}", userId)
                .retrieve()
                .body(new ParameterizedTypeReference<List<Product>>() {});
            
            recommendationCache.put(userId, response);  // Update cache on success
            return response;
        });
    }
    
    private CompletableFuture<List<Product>> getRecommendationsFallback(String userId, Throwable t) {
        log.warn("Recommendation service unavailable for user {}, using fallback", userId, t);
        
        // Try cache first
        List<Product> cached = recommendationCache.getIfPresent(userId);
        if (cached != null) {
            return CompletableFuture.completedFuture(cached);
        }
        
        // Fall back to static featured list
        return CompletableFuture.completedFuture(getFeaturedProducts());
    }
    
    private List<Product> getFeaturedProducts() {
        // Static curated list, refreshed daily by a background job
        return featuredProductRepository.findAll();
    }
}
```

### Pattern: Retry with exponential backoff and jitter

**What:** Automatically retry failed requests with increasing delays, randomized to avoid thundering herds.

**When to use:**

- Transient network failures (503, connection timeout).
- Rate-limited APIs (429).
- Idempotent reads where eventual success is acceptable.

**When NOT to use:**

- Non-idempotent writes (POST /orders) without idempotency tokens.
- Permanent failures (400, 404, 401).
- Latency-critical synchronous paths (retries add tail latency).

**Failure modes:**

- Retry amplification: 3 retries at each of 5 layers = 3⁵ = 243× load on the database.
- Backoff too short → overwhelm recovering service.
- Backoff too long → user waits unnecessarily.

**Code:**

```yaml
resilience4j:
  retry:
    instances:
      inventoryService:
        maxAttempts: 3
        waitDuration: 100ms
        enableExponentialBackoff: true
        exponentialBackoffMultiplier: 2
        enableRandomizedWait: true  # Full jitter
        randomizedWaitFactor: 0.5   # Wait between 50%–150% of calculated delay
        retryExceptions:
          - java.net.ConnectException
          - java.util.concurrent.TimeoutException
          - org.springframework.web.client.HttpServerErrorException$ServiceUnavailable
        ignoreExceptions:
          - org.springframework.web.client.HttpClientErrorException
```

```java
@Service
public class InventoryService {
    
    private final RestClient inventoryClient;
    
    @Retry(name = "inventoryService")
    public StockLevel checkStock(String productId) {
        return inventoryClient.get()
            .uri("/inventory/{productId}", productId)
            .retrieve()
            .body(StockLevel.class);
    }
}
```

### Pattern: Bulkhead isolation with dedicated thread pools

**What:** Assign each dependency a separate thread pool or semaphore limit to prevent one slow service from exhausting all threads.

**When to use:**

- Multiple external dependencies with different latency profiles.
- Multi-tenant systems where one tenant's heavy usage should not starve others.

**When NOT to use:**

- Virtual threads (built-in isolation via cheap thread creation).
- Single-threaded reactive systems (use semaphore bulkhead instead).

**Failure modes:**

- Pool too small → artificial throttling, rejected requests.
- Pool too large → total threads exceed JVM capacity, OOM.
- Shared connection pool bypasses thread isolation.

**Code:**

```yaml
resilience4j:
  thread-pool-bulkhead:
    instances:
      paymentService:
        maxThreadPoolSize: 10
        coreThreadPoolSize: 5
        queueCapacity: 20
      searchService:
        maxThreadPoolSize: 20
        coreThreadPoolSize: 10
        queueCapacity: 50
```

```java
@Configuration
public class BulkheadConfig {
    
    @Bean
    public ThreadPoolBulkheadRegistry bulkheadRegistry() {
        return ThreadPoolBulkheadRegistry.ofDefaults();
    }
}

@Service
public class PaymentService {
    
    @Bulkhead(name = "paymentService", type = Bulkhead.Type.THREADPOOL)
    public CompletableFuture<PaymentResult> processPayment(PaymentRequest request) {
        return CompletableFuture.supplyAsync(() -> {
            // Call payment gateway
            return paymentGateway.charge(request);
        });
    }
}
```

### Pattern: Rate limiting with Redis and distributed counters

**What:** Enforce per-client or per-tenant request limits using a centralized Redis counter, preventing abuse and protecting downstream services.

**When to use:**

- Multi-tenant APIs with tiered pricing (free vs paid).
- Public APIs exposed to third parties.
- Protection against accidental retry loops or misbehaving clients.

**When NOT to use:**

- Single-tenant internal services (local in-memory limiters are cheaper).
- Latency-critical paths where Redis RTT is unacceptable (use local approximation).

**Failure modes:**

- Redis down → either fail open (no limiting) or fail closed (reject all requests).
- Clock skew between app servers causes inconsistent window boundaries.
- High cardinality keys (per-user limiting with millions of users) → Redis memory pressure.

**Code:**

```lua
-- rate_limit.lua
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local now = tonumber(ARGV[3])

local current_window = math.floor(now / window)
local current_key = key .. ":" .. current_window
local previous_key = key .. ":" .. (current_window - 1)

local current_count = tonumber(redis.call('GET', current_key) or '0')
local previous_count = tonumber(redis.call('GET', previous_key) or '0')

local elapsed = now % window
local weight = 1 - (elapsed / window)
local estimated = (previous_count * weight) + current_count

if estimated >= limit then
    return {0, estimated}  -- Rejected
else
    redis.call('INCR', current_key)
    redis.call('EXPIRE', current_key, window * 2)
    return {1, estimated}  -- Allowed
end
```

```java
@Service
public class DistributedRateLimiter {
    
    private final StringRedisTemplate redis;
    private final RedisScript<List> script;
    private final MeterRegistry metrics;
    
    public boolean allowRequest(String clientId, int limit, int windowSeconds) {
        String key = "rate_limit:" + clientId;
        long now = System.currentTimeMillis() / 1000;
        
        List<Long> result = redis.execute(script, 
            Collections.singletonList(key), 
            String.valueOf(limit), 
            String.valueOf(windowSeconds), 
            String.valueOf(now));
        
        boolean allowed = result.get(0) == 1;
        double current = result.get(1);
        
        metrics.counter("rate_limiter", 
            "client", clientId, 
            "allowed", String.valueOf(allowed))
            .increment();
        
        return allowed;
    }
}

@RestController
public class ApiController {
    
    private final DistributedRateLimiter limiter;
    
    @GetMapping("/api/products")
    public ResponseEntity<List<Product>> getProducts(@RequestHeader("X-Client-ID") String clientId) {
        if (!limiter.allowRequest(clientId, 100, 60)) {  // 100 req/min
            return ResponseEntity.status(HttpStatus.TOO_MANY_REQUESTS)
                .header("Retry-After", "60")
                .build();
        }
        
        return ResponseEntity.ok(productService.findAll());
    }
}
```

### Pattern: Time limiter with deadline propagation

**What:** Enforce an absolute timeout on an operation, canceling it if it exceeds the deadline, and propagate the remaining time to downstream calls.

**When to use:**

- Fan-out calls where the aggregate must complete within a user-facing SLA.
- Nested service chains where each layer consumes part of the total budget.

**When NOT to use:**

- Operations that cannot be safely canceled mid-execution (partially written files, database transactions).

**Failure modes:**

- Canceled operation leaves inconsistent state (file half-written).
- Deadline not propagated → downstream consumes full timeout even when upstream has <100 ms remaining.

**Code:**

```java
@Service
public class ProductAggregator {
    
    @TimeLimiter(name = "productAggregation")
    public CompletableFuture<ProductDetails> getProductDetails(String productId) {
        Instant deadline = Instant.now().plus(Duration.ofMillis(500));
        
        var productFuture = catalogService.getProduct(productId, deadline);
        var priceFuture = pricingService.getPrice(productId, deadline);
        var reviewsFuture = reviewService.getReviews(productId, deadline);
        
        return CompletableFuture.allOf(productFuture, priceFuture, reviewsFuture)
            .thenApply(v -> new ProductDetails(
                productFuture.join(),
                priceFuture.join(),
                reviewsFuture.join()
            ))
            .orTimeout(500, TimeUnit.MILLISECONDS)  // Enforce overall deadline
            .exceptionally(ex -> {
                log.error("Product details aggregation timed out for {}", productId, ex);
                return ProductDetails.partial(productId);  // Fallback
            });
    }
}

// Downstream service respects deadline
@Service
public class PricingService {
    
    public CompletableFuture<Price> getPrice(String productId, Instant deadline) {
        Duration remaining = Duration.between(Instant.now(), deadline);
        if (remaining.isNegative()) {
            return CompletableFuture.failedFuture(new TimeoutException("Deadline exceeded"));
        }
        
        return pricingClient.get()
            .uri("/prices/{id}", productId)
            .header("X-Deadline", deadline.toString())  // Propagate
            .retrieve()
            .body(Price.class)
            .toFuture()
            .orTimeout(remaining.toMillis(), TimeUnit.MILLISECONDS);
    }
}
```

### Pattern: Kill switch with feature flags

**What:** Remotely disable a feature or dependency via a feature flag, bypassing it entirely to shed load or mitigate an incident.

**When to use:**

- Experimental features that might cause load spikes.
- Non-critical features that can be disabled during an outage to reduce load.
- Gradual rollout with the ability to instantly roll back without redeploying.

**When NOT to use:**

- Critical business logic where disabling would break core functionality.
- Without observability to detect that the flag is active (silent degradation).

**Failure modes:**

- Flag misconfigured → feature permanently disabled, revenue impact.
- Flag service down → fail-open (feature enabled) or fail-closed (feature disabled)?

**Code:**

```yaml
# Feature flag config (external service or config file)
features:
  recommendations:
    enabled: true
  email-notifications:
    enabled: true
  advanced-search:
    enabled: false  # Kill switch active
```

```java
@Service
public class SearchService {
    
    private final FeatureFlagService flags;
    
    public SearchResults search(String query) {
        SearchResults basic = performBasicSearch(query);
        
        if (flags.isEnabled("advanced-search")) {
            try {
                return enhanceWithAdvancedFeatures(basic, query);
            } catch (Exception e) {
                log.error("Advanced search failed, falling back to basic", e);
                return basic;  // Degrade gracefully
            }
        }
        
        return basic;
    }
}

// Feature flag service with cached values and fallback
@Service
public class FeatureFlagService {
    
    private final RestClient configClient;
    private final Map<String, Boolean> cache = new ConcurrentHashMap<>();
    
    @Scheduled(fixedRate = 10_000)  // Refresh every 10s
    public void refreshFlags() {
        try {
            Map<String, Boolean> flags = configClient.get()
                .uri("/feature-flags")
                .retrieve()
                .body(new ParameterizedTypeReference<>() {});
            cache.putAll(flags);
        } catch (Exception e) {
            log.warn("Failed to refresh feature flags, using cached values", e);
        }
    }
    
    public boolean isEnabled(String flag) {
        return cache.getOrDefault(flag, false);  // Fail closed
    }
}
```

### Pattern: Dependency criticality tiering

**What:** Classify dependencies by criticality; apply different resilience strategies to critical vs degradable vs optional dependencies.

**ShopKart example:**

| Dependency       | Tier       | Strategy                                  | Failure mode                     |
|------------------|------------|-------------------------------------------|----------------------------------|
| `payment`        | Critical   | Circuit breaker, 3 retries, no fallback   | Order fails, user retries        |
| `inventory`      | Critical   | Circuit breaker, 2 retries, no fallback   | Order fails if out of stock      |
| `catalog`        | Critical   | Circuit breaker, cache fallback           | Show stale product data          |
| `pricing`        | Critical   | Circuit breaker, cache fallback (5 min)   | Show slightly outdated prices    |
| `recommendations`| Degradable | Circuit breaker, static fallback          | Show featured products           |
| `reviews`        | Degradable | Circuit breaker, omit reviews             | Product page missing reviews     |
| `notification`   | Optional   | Fire-and-forget, retry async              | User does not get email          |
| `analytics`      | Optional   | No retries, log failure                   | Event lost, acceptable           |

**Code:**

```java
public enum DependencyTier {
    CRITICAL,    // Must succeed for request to succeed
    DEGRADABLE,  // Can substitute with fallback
    OPTIONAL     // Can omit entirely
}

@Service
public class OrderService {
    
    public Order createOrder(OrderRequest request) {
        // Critical: must succeed
        Payment payment = paymentService.charge(request.getPaymentInfo());
        Inventory inventory = inventoryService.reserve(request.getItems());
        
        // Degradable: use fallback if unavailable
        Price price = Try.of(() -> pricingService.calculateTotal(request.getItems()))
            .recover(ex -> pricingCache.getOrDefault(request.getItems(), Price.unknown()))
            .get();
        
        // Optional: fire-and-forget
        CompletableFuture.runAsync(() -> {
            try {
                notificationService.sendOrderConfirmation(request.getUserId(), order);
            } catch (Exception e) {
                log.warn("Failed to send notification, will retry async", e);
                retryQueue.enqueue(new NotificationTask(request.getUserId(), order));
            }
        });
        
        return new Order(payment, inventory, price);
    }
}
```

## How big tech does it

### Netflix: From Hystrix to adaptive systems

Netflix open-sourced **Hystrix** in 2012—the circuit breaker library that defined resilience patterns for a generation. By 2018 they placed it in maintenance mode and by 2024 it was fully deprecated. Why?

Hystrix required explicit configuration per dependency: thread pool size, timeout, circuit breaker thresholds. At Netflix's scale (hundreds of services, thousands of dependency relationships) this became unmaintainable. A database gets slow, you need to tune 47 Hystrix configurations across 20 services. Configuration drift was constant. The fixed thread pools wasted resources at low load and still exhausted at high load.

Netflix replaced Hystrix with **adaptive concurrency limits** (the gradient algorithm described earlier). Instead of fixed pools, the system measures latency and dynamically adjusts the concurrency limit. When a service slows down, the limit drops; when it is healthy, the limit rises. No configuration required—the system self-tunes based on observed behavior. Netflix reported that this cut operational toil significantly and improved efficiency (fewer threads idle at low load, better protection at high load).

**Chaos Monkey and the Simian Army** are Netflix's chaos engineering tools. Chaos Monkey randomly terminates EC2 instances during business hours. Chaos Gorilla simulates an entire AWS availability zone failing. Chaos Kong takes down an entire AWS region. Latency Monkey injects artificial latency. They run these continuously in production—not as one-off experiments, but as constant background validation that the system can survive failures.

Netflix's **ChAP (Chaos Automation Platform)** is the modern successor: automated chaos experiments with hypothesis testing, blast radius control, and rollback. Before deploying a change, ChAP runs a controlled experiment: inject a fault (kill a service, add latency), measure steady-state metrics (error rate, p99 latency), verify the system degraded gracefully. If the experiment fails, the deployment is blocked.

**Regional failover exercises:** Netflix practices full region failovers quarterly. They reroute all traffic from us-east-1 to us-west-2, verify the application works, fail back. These drills have uncovered dozens of hidden dependencies—hardcoded region names in config, DNS propagation delays, certificate expirations that only triggered in the secondary region. The exercises are scheduled, announced, and monitored by the entire engineering org.

### AWS: Cells, shuffle sharding, and static stability

AWS's **Builders' Library** is their public engineering blog. Key resilience articles:

**"Avoiding insurmountable queue backlogs"**: Use bounded queues with overflow shedding. A saturated queue that grows unbounded turns into a metastable failure—even when load decreases, the system spends all its time draining the backlog instead of serving new requests. AWS services use fixed-size queues and reject excess load with 503 instead of queuing infinitely.

**"Timeouts, retries, and backoff with jitter"**: The article that popularized full jitter. AWS measured retry patterns across their fleet and found that clients without jitter synchronized into thundering herds. Full jitter spreads retries evenly, completing requests 30% faster than non-jittered backoff.

**"Static stability"**: A system is statically stable if it can operate during a control-plane outage. Example: Route 53 name servers do not depend on a central database to answer queries—they cache zone data locally. If the control plane fails (you cannot update DNS records), existing queries continue to work. Contrast with a system where every request requires a real-time lookup to a config service—if the config service is down, all requests fail.

**"Cell-based architecture"**: AWS partitions large-scale services into isolated cells. Each cell serves a subset of customers and has its own compute, storage, and control plane. A cell failure affects only that cell's customers. DynamoDB, S3, and Route 53 all use cell-based designs. The tradeoff: operational complexity (managing N cells) for blast radius reduction (1/N of customers affected by a single-cell failure).

**Shuffle sharding** (detailed earlier): Each customer is assigned to multiple randomly chosen shards. A shard failure affects only customers whose entire shard set failed—statistically improbable. Route 53 uses this for name servers; AWS Direct Connect uses it for redundant connections.

### Google SRE: Error budgets and graceful degradation

Google's **Site Reliability Engineering** book canonicalized the error budget framework. An SLO (Service Level Objective) is not a goal—it is a contract. If you promise 99.9% availability, you have a budget of 43.2 minutes/month to spend on errors. Spend it on risky deployments, dependency failures, or planned maintenance. When the budget is exhausted, stop deploying new features and focus on hardening.

Error budgets align incentives: product managers want features (which add risk); SREs want stability (which slows innovation). The error budget is the negotiation mechanism. If you have budget left, deploy. If you are out, stabilize first. This prevents both excessive caution (we never deploy anything) and recklessness (we deploy broken code every day).

**"Tail at Scale"** is a Google paper on reducing p99 latency in distributed systems. Key technique: **hedged requests**. For latency-critical reads, send the same request to two replicas simultaneously. Return whichever responds first. Cancel the slower one. This cuts p99 latency by 40–60% because you avoid waiting for the single slow replica. Cost: 2× load, so only viable for read-heavy workloads with spare capacity.

**Graceful degradation at scale:** Google services have explicit degraded modes. If the personalization backend is down, YouTube serves a generic homepage. If the spell-checker is slow, Search returns results for the literal query instead of waiting for corrections. The user experience degrades, but the core functionality (search, video playback) remains available.

### Stripe: Published rate-limiter design

Stripe's engineering blog published their rate limiter design. Key details:

- **Token bucket per API key**, stored in Redis.
- **Four limits simultaneously**: per-second burst, per-hour sustained, per-day quota, and concurrent requests. Each limit is independent; hitting any one returns 429.
- **Distributed enforcement with Lua scripts** for atomic increment-and-check. A naive implementation (read count, increment, check) has a race condition where two requests simultaneously read the same count and both pass the limit. Lua scripts run atomically in Redis.
- **Retry-After header** tells clients when to retry. Stripe's SDKs respect this automatically.
- **Exemptions for trusted partners**: High-volume integrations get higher limits or bypass rate limiting entirely. This is configured per API key in a database.

Stripe reported that rate limiting reduced abusive traffic by 99% and prevented several accidental retry loops that could have taken down the API.

### Slack: Public incident writeups on degradation

Slack publishes detailed incident postmortems. A recurring theme: **graceful degradation is hard to test**.

**January 2021 outage**: A database failover took 3 minutes—well within their RTO—but the application did not reconnect automatically. Clients saw "disconnected" for the full 3 minutes. Root cause: connection pooling logic assumed the primary database IP never changed. The failover changed the IP; the app did not re-resolve DNS. Fix: connection pools now re-resolve DNS on connection failure. This only surfaced during a real failover because their failover tests were scripted (they manually updated the connection string instead of actually failing over the database).

**Load shedding during message surges**: Slack spikes to 10× normal load during major events (company all-hands, breaking news). Their strategy: shed low-priority work first—presence updates, read receipts, typing indicators—before shedding messages. Messages are tier-1 critical; presence is tier-3 degradable. Users tolerate missing "is typing" indicators; they do not tolerate lost messages.

### Shopify: Flash-sale queueing and load shedding

Shopify powers e-commerce for high-traffic flash sales—limited-edition sneaker drops, concert ticket releases. These events generate 50–100× normal load in the first 60 seconds.

**Queue-Fair**: Shopify's waiting room system. When load spikes, put users in a queue with a estimated wait time. Admit them at a controlled rate (e.g., 1,000 users/minute) instead of letting 100,000 users simultaneously hammer checkout. The queue is statically stable—implemented in Cloudflare Workers at the edge, so even if Shopify's origin is down, the queue continues to function.

**Load shedding before saturation:** Shopify monitors active requests, CPU, and database connection pool saturation. At 80% saturation, they start rejecting low-priority requests (analytics tracking, third-party webhooks, product recommendations). At 90%, they reject non-critical features (discounts, gift cards). At 95%, they serve a static "We're experiencing high traffic" page instead of processing checkout. The goal: maintain core functionality (add to cart, checkout) even if peripheral features degrade.

### Common patterns across all

1. **Explicit dependency tiering**: Every dependency is classified as critical, degradable, or optional. Critical dependencies have no fallback—failure fails the request. Degradable dependencies have fallbacks (cache, static defaults). Optional dependencies are fire-and-forget.
2. **Practiced failure**: Netflix's Chaos Monkey, AWS's game days, Google's DiRT (Disaster Recovery Testing). Failures are not theoretical—they are triggered regularly in production-like environments to validate that runbooks, alerting, and recovery mechanisms work.
3. **Strong platform defaults**: AWS's service clients have built-in retries, backoff, and timeouts. Google's gRPC has deadline propagation. Netflix's libraries ship with adaptive concurrency limits. Developers do not configure resilience from scratch—they inherit battle-tested defaults.

### What a 10-person team should copy first

Do not start with a service mesh. Do not build a custom chaos engineering platform. Copy these, in this order:

1. **Timeouts on every call**: HTTP clients, database connections, message queue consumers. No infinite waits.
2. **Bulkheads (thread pools or semaphores)**: Prevent one slow dependency from exhausting all threads.
3. **Kill switches (feature flags)**: The ability to remotely disable a feature without redeploying is a tactical nuclear option during an incident. Build this early.
4. **Explicit fallbacks**: For degradable dependencies, define the fallback behavior (cache, static list, skip the feature) before the dependency fails in production.
5. **Liveness probes that do not check dependencies**: Liveness should only verify the JVM is alive, not that the database is reachable.
6. **DR runbooks that you actually run**: Quarterly failover drills, even if your "DR" is just restoring from a backup.

Avoid premature investment in service meshes (adds operational complexity for unclear benefit at small scale), distributed tracing (useful but not a resilience primitive), or chaos engineering in production (chaos without observability is just breaking things). Build observability first (metrics, logs, structured traces), resilience second, chaos third.

## Best-practice checklist

Evaluate your system against this list. Each "no" is a production incident waiting to happen.

**Timeouts and deadlines:**
- [ ] Every HTTP client call has an explicit timeout (not the default infinite or 30-second guess).
- [ ] Every database query has a timeout (via JDBC `setQueryTimeout` or connection-level timeout).
- [ ] Every message queue consumer has a processing timeout (reject messages that take too long).
- [ ] Timeouts are derived from production telemetry (p99.9 + buffer), not round numbers.
- [ ] Deadline propagation is implemented for nested calls (downstream consumes remaining time, not full budget).

**Retries:**
- [ ] Only idempotent operations are retried, or non-idempotent writes use idempotency tokens.
- [ ] Retries use exponential backoff with full jitter (not fixed intervals).
- [ ] Retry budgets are tracked (stop retrying if success rate drops below threshold).
- [ ] Retries are counted and exposed as metrics (to detect retry amplification).
- [ ] Only the outermost layer retries, or each layer subtracts from a shared budget.

**Circuit breakers:**
- [ ] Every external dependency is protected by a circuit breaker.
- [ ] Circuit breaker thresholds are tuned (not copy-pasted defaults from a blog).
- [ ] `minimumNumberOfCalls` is low enough to trip during low-traffic hours.
- [ ] Circuit breakers have fallback logic (they protect your threads; fallbacks protect the user).

**Bulkheads:**
- [ ] Each critical dependency has a dedicated thread pool or semaphore limit.
- [ ] Database connection pools are per-dependency, not one global pool.
- [ ] Thread pool sizes are sized from Little's Law or empirical load testing, not guesses.

**Dependency management:**
- [ ] Every dependency is classified by tier (critical / degradable / optional).
- [ ] Degradable dependencies have documented fallback behavior (cache, static default, skip).
- [ ] Optional dependencies are fire-and-forget with async retry queues.
- [ ] Dependency graph is documented and reviewed (to avoid serial dependency chains that multiply failure rates).

**Health checks:**
- [ ] Liveness probes check only JVM health, never dependencies.
- [ ] Readiness probes are shallow or circuit-breaker-aware (not deep dependency checks on every probe).
- [ ] Health endpoints do not expose internal topology to public traffic.

**Rate limiting and load shedding:**
- [ ] Public APIs have rate limiting per client/tenant.
- [ ] Rate limits are enforced in a distributed manner (Redis, API gateway).
- [ ] Queue depths are bounded (reject excess work with 503, do not queue infinitely).
- [ ] Load shedding activates before saturation (80–90% utilization threshold).
- [ ] Low-priority traffic is identified and shed first (analytics, webhooks, background jobs).

**Observability:**
- [ ] Every timeout, retry, circuit breaker trip, and load-shed event is logged and counted in metrics.
- [ ] SLIs (error rate, latency) are measured and tracked against SLOs.
- [ ] Error budgets are calculated and visible to the team.

**Chaos and DR:**
- [ ] Chaos experiments have been run in staging (not just planned).
- [ ] DR runbooks exist and are versioned in the repo.
- [ ] DR drills are scheduled quarterly (not "we'll do it when we have time").
- [ ] Failover has been tested end-to-end in the last 6 months.

**Feature flags and kill switches:**
- [ ] Feature flags exist for experimental or high-risk features.
- [ ] Kill switches can be activated without redeploying.
- [ ] Flag service failure defaults to a safe state (fail-closed for new features, fail-open for critical paths).

## Anti-patterns and war stories

### Anti-pattern: Retry at every layer without coordination

**What it looks like:** API gateway retries failed requests 3 times. The backend service retries database queries 3 times. The database client library retries connections 2 times. A single user request becomes 3 × 3 × 2 = 18 database queries.

**Why it is wrong:** Retry amplification. One slow query cascades into a stampede that overwhelms the database. The database gets slower, triggering more retries, positive feedback loop.

**War story: The 47× amplification cascade**

A ShopKart-like e-commerce platform had this exact setup. During Black Friday, a single slow product page query (3 seconds instead of 50 ms—someone forgot an index after a schema change) triggered:

- 3 retries at the API gateway layer
- 3 retries at the catalog service layer
- 2 retries at the database connection pool layer
- Total: 3 × 3 × 2 = 18 database queries per user request

The product page was popular (featured in the homepage banner). 500 users/second clicked it. 500 × 18 = 9,000 queries/second hit the database, overwhelming it. The database's query queue filled, every query took 10+ seconds, timeouts fired everywhere, circuit breakers tripped across the entire estate.

Detection took 8 minutes (the first on-call assumed it was a traffic spike, not retry amplification). Diagnosis took another 12 minutes (database metrics showed 9,000 queries/sec but only 500 distinct session IDs—the signature of retry amplification). Fix: emergency feature flag disabled the problematic product page, traffic dropped instantly, database recovered in 2 minutes. Total outage: 22 minutes. Lost revenue: estimated $400K. Root cause: no coordination between retry layers.

**Fix pattern:** Only the outermost layer (API gateway) retries. Inner layers fail fast and propagate the error. Or: use a shared retry budget (token bucket) across all layers—once the budget is exhausted, no layer retries.

### Anti-pattern: Circuit breaker with cargo-culted defaults

**What it looks like:** Copy-pasting Resilience4j config from a tutorial:

```yaml
resilience4j:
  circuitbreaker:
    instances:
      myService:
        failureRateThreshold: 50
        slidingWindowSize: 100
        minimumNumberOfCalls: 100
        waitDurationInOpenState: 60s
```

**Why it is wrong:**

- `minimumNumberOfCalls: 100` means the circuit never trips during off-peak hours (if you get 10 requests/minute, you need 10 minutes of failures before evaluation).
- `failureRateThreshold: 50` means half your requests fail before the circuit trips—your thread pool might already be exhausted.
- `waitDurationInOpenState: 60s` is a guess. Maybe your service recovers in 5 seconds (unnecessarily long outage), or maybe it takes 5 minutes (constant open/half-open/open flapping).

**War story: The circuit breaker that never tripped**

A payment service had a circuit breaker protecting calls to a fraud-detection API. Configuration was copied from the Resilience4j README: `minimumNumberOfCalls: 100`. During business hours (9 AM–5 PM) the service handled 200 requests/minute—circuit breaker worked fine, tripped after 30 seconds of failures.

At 11 PM on a Saturday, a fraud-detection API deployment went wrong, returning 500s. The payment service was processing 8 requests/minute (overnight batch). With `minimumNumberOfCalls: 100`, it took 12.5 minutes to accumulate enough calls to evaluate the failure rate. During those 12 minutes, every payment request waited for the full 10-second timeout, then failed. Thread pool exhausted. The batch job backed up. By the time the circuit tripped (12 minutes later), 96 payments had timed out and were stuck in a manual review queue.

**Fix:** Tune `minimumNumberOfCalls` based on actual traffic: `max(10, expected_requests_per_minute / 6)`. At 8 req/min, use `minimumNumberOfCalls: 10`, trip after 75 seconds. At 200 req/min, use `minimumNumberOfCalls: 33`, trip after 10 seconds.

### Anti-pattern: Liveness probe that checks the database

**What it looks like:**

```yaml
livenessProbe:
  httpGet:
    path: /actuator/health
    port: 8080
  periodSeconds: 10
```

```java
// Health check hits database
@Component
public class DatabaseHealthIndicator implements HealthIndicator {
    @Override
    public Health health() {
        jdbcTemplate.queryForObject("SELECT 1", Integer.class);  // BAD
        return Health.up().build();
    }
}
```

**Why it is wrong:** The database has a 5-minute failover window (primary crashes, replica is promoted). Kubernetes liveness probe fails, restarts all 50 pods simultaneously. All 50 pods come up, liveness probe runs, all 50 hit the database simultaneously (thundering herd). The database, already struggling with the failover, is overwhelmed. Pods restart again. Loop continues for 45 minutes until someone manually scales the deployment to 1 pod.

**War story: The 50-minute outage from a 5-minute failover**

An inventory service used AWS RDS Multi-AZ PostgreSQL. The primary instance had a hardware failure at 2:47 AM. RDS automatically failed over to the standby replica in another AZ—documented failover time is 1–2 minutes, actual time was 4 minutes due to the crash requiring log replay.

The inventory service had 50 pods in Kubernetes. Liveness probe: `SELECT 1 FROM inventory LIMIT 1`. Probe runs every 10 seconds, fails after 3 consecutive failures (30 seconds). At 2:48 AM (1 minute into the failover), all 50 pods' liveness probes failed. Kubernetes restarted all 50 pods simultaneously.

Pods started, Spring Boot initialized, liveness probe ran immediately. 50 pods × `SELECT 1` query = 50 simultaneous connections to the database, which was still in the middle of promoting the replica. RDS connection limit was 100; the burst consumed 50, leaving 50 for actual application traffic. But the pods also opened their HikariCP pools (20 connections each) = 50 × 20 = 1,000 connection attempts. RDS rejected them (max connections exceeded). Pods failed to start. Kubernetes tried again. Loop.

The standby was fully promoted by 2:51 AM (4 minutes)—but the pod restart loop prevented traffic from reaching it. On-call engineer paged at 2:53 AM. Diagnosed at 3:05 AM. Fix: manually scaled deployment to 5 pods, which successfully started. Gradually scaled back to 50 over 10 minutes. Service fully recovered at 3:37 AM. Outage duration: 50 minutes. Database failover duration: 4 minutes. The liveness probe turned a 4-minute failover into a 50-minute outage.

**Fix:** Liveness checks only JVM health (HTTP server responsive, memory not exhausted). Readiness can check circuit breaker state (if circuit to database is open, mark pod unready).

### Anti-pattern: Fallback that serves stale inventory

**What it looks like:**

```java
@CircuitBreaker(name = "inventory", fallbackMethod = "checkStockFallback")
public StockLevel checkStock(String productId) {
    return inventoryClient.get().uri("/stock/{id}", productId).retrieve().body(StockLevel.class);
}

public StockLevel checkStockFallback(String productId, Throwable t) {
    return stockCache.getOrDefault(productId, new StockLevel(100));  // BAD: assume 100 in stock
}
```

**Why it is wrong:** Fallback returns cached inventory. Cache says "100 units in stock", actual inventory is 0 (sold out 10 minutes ago during a flash sale). You accept 120 orders. You oversell by 20 units. Customers are angry. Customer support is overwhelmed. You lose money on expedited shipping to fulfill the mistake or you refund and lose credibility.

**War story: The 4,000-unit oversell**

A limited-edition product launch (concert tickets for a popular artist). Inventory service had 5,000 tickets. Circuit breaker on the inventory service with a 10-minute cache fallback. At 10:02 AM, inventory service experienced a GC pause (old-gen collection, 8 seconds—later traced to a memory leak). Circuit breaker tripped. Fallback served cached stock levels.

Problem: The cache was populated at 10:00 AM (before the sale started). It said "5,000 available". Reality at 10:02 AM: only 1,000 tickets left (4,000 sold in the first 2 minutes). The circuit stayed open for 30 seconds (configured `waitDurationInOpenState`). During those 30 seconds, 3,000 more orders went through, all seeing "5,000 available" from the cache. Total orders: 7,000. Actual inventory: 5,000. Oversell: 2,000 tickets.

At 10:03 AM, the circuit closed, inventory service responded with "sold out", but 2,000 customers already received order confirmations. The company had to honor the confirmations (bad faith to cancel), negotiated with the venue to open additional sections, absorbed the cost difference. Financial loss: ~$150K. Reputation damage: trending on Twitter for "botched ticket sale".

**Fix:** For money/inventory operations, the fallback is to fail explicitly. Return a "We cannot confirm availability right now, please try again in a moment" message instead of guessing stock levels. If you must fallback, use a pessimistic assumption (assume out of stock) rather than optimistic (assume in stock).

## Projects for this phase

**Small projects (individual, 8–12 hours each):**

1. **Resilience4j configuration playground**: Deploy a simple Spring Boot app with a flaky downstream service (randomly returns 500 or delays 5 seconds). Configure circuit breaker, retry, bulkhead, and time limiter. Use JMeter or Gatling to generate load. Compare:
   - No resilience patterns (baseline: how badly does it fail?)
   - Circuit breaker only (trips after how many failures? recovers how fast?)
   - Circuit breaker + retry with jitter (does retry amplification happen? measure with Micrometer)
   - Circuit breaker + retry + bulkhead (does the bulkhead isolate the failure?)
   Deliverable: written report comparing failure modes, metrics screenshots, configuration tuning notes.

2. **Distributed rate limiter with Redis + Lua**: Implement token bucket and sliding window counter algorithms. Deploy Redis. Write Lua scripts for atomic increment-and-check. Build a REST API (`POST /api/action`) that enforces 100 requests/minute per API key. Test with concurrent clients (5 threads, 200 requests each). Measure false negatives (requests allowed over limit due to race conditions—should be zero with Lua atomicity). Deliverable: working rate limiter, test results, performance measurements.

3. **Adaptive load shedder**: Implement the Netflix gradient/AIMD concurrency limit algorithm. Deploy a service with a CPU-intensive endpoint (`/expensive`). Use JMeter to gradually increase load. Service should measure p99 latency and dynamically reduce concurrency when latency spikes. Plot: load (req/s) vs concurrency limit vs p99 latency. Deliverable: implementation, graphs showing adaptive limit responding to load, threshold tuning notes.

4. **Toxiproxy fault-injection test suite**: Deploy a multi-service setup (API gateway → catalog service → PostgreSQL). Run Toxiproxy as a proxy between catalog and database. Inject faults: 500 ms latency, connection drops, bandwidth limits. Verify circuit breakers trip, retries trigger, fallbacks activate. Automate as integration tests. Deliverable: Toxiproxy config, test suite, CI pipeline that runs fault injection on every PR.

5. **DR runbook + failover drill**: Document a disaster recovery procedure for a database (e.g., PostgreSQL with streaming replication). Write the runbook: steps to promote replica, update connection strings, verify data integrity, fail back. Execute the drill in a staging environment. Measure RTO (time from failure to recovery) and RPO (data loss). Deliverable: runbook (Markdown), drill report (what went wrong, what was learned, updated runbook).

**Large project (team of 2–3, 40–50 hours total): Resilience harness and scored game day**

Goal: Build a resilience testing harness for ShopKart (catalog, cart, order, payment, inventory services). Then run a game day where you inject failures and score the system's response.

Architecture:
- Deploy ShopKart services (use Docker Compose or Kubernetes).
- Add instrumentation: Prometheus metrics, structured logging, distributed tracing (OpenTelemetry).
- Build a fault injector: control plane that can trigger failures (kill pods, add latency, return errors, throttle resources).
- Build a steady-state monitor: dashboard showing SLIs (error rate, p99 latency, throughput).

Fault scenarios (inject each for 5 minutes, measure impact):
1. Payment service crashes (1 replica out of 3).
2. Database connection pool exhausted (inventory service).
3. Network latency spikes (200 ms added between catalog and pricing).
4. Inventory service returns 500s (simulate downstream API failure).
5. Rate limit triggered (API gateway throttles requests).
6. Full region failure (entire deployment killed, failover to secondary).

Scoring rubric (0–100 points):
- SLIs maintained (error rate <1%, p99 <500 ms): 30 points
- Graceful degradation (reduced functionality, not total failure): 20 points
- Alerts fired within 1 minute of fault: 15 points
- Recovery without manual intervention: 20 points
- No customer data loss/inconsistency: 15 points

Acceptance criteria:
- Score ≥80 points on 5 out of 6 scenarios.
- Runbook documents each scenario: detection, mitigation, recovery.
- Metrics/traces captured for post-incident review.

Stretch goals:
- Automate fault injection (scheduled chaos experiments in CI/CD).
- Implement shuffle sharding (partition users across independent cells).
- Multi-region active-passive DR with <5-minute RTO.

Deliverable: working harness, game day report (score per scenario, gaps identified, fixes implemented), updated architecture with resilience patterns documented.

## Interview drilldown

### 1. Design a rate limiter

**Initial ask:** "Design a rate limiter that enforces 100 requests per minute per user. How would you implement it?"

**Strong answer:** Start with algorithm choice: token bucket is most common (allows bursts, smooth limiting). Store per-user counters in Redis (distributed state). Increment-and-check must be atomic—use a Lua script in Redis to avoid race conditions. Return 429 with `Retry-After` header when limit is exceeded. Consider edge cases: what if Redis is down (fail open or fail closed?), what if users have different limits (store limit per user in config service), what if you need per-second and per-hour limits simultaneously (independent counters).

**Follow-up 1:** "How do you handle distributed enforcement? What if multiple instances race to increment the counter?"

**Strong answer:** Lua scripts in Redis execute atomically. All app instances call the same Redis key (`rate_limit:user123`), Lua script increments and checks in one atomic operation. No race condition. Alternative: use Redis INCR (atomic) but this only handles count-based, not time-windowed limits.

**Follow-up 2:** "The system has 10 million active users. How do you scale this?"

**Strong answer:** 10 million keys in Redis is manageable (a single Redis instance can hold 100M+ keys). If you need more scale, shard by user ID (consistent hashing to multiple Redis instances). Watch memory: a counter per user per window is ~20 bytes, 10M users = 200 MB—cheap. Bigger issue: request rate to Redis. If you have 100K req/s, Redis can handle it (single-threaded but fast, ~100K ops/s per instance). If you need more, shard or use Redis Cluster.

**Follow-up 3:** "Some users are free tier (100 req/min), some are paid tier (1000 req/min). How do you enforce per-tenant limits?"

**Strong answer:** Store limit per user in a config database or cache. Lua script takes the limit as an argument. When a request arrives, look up the user's tier, fetch the limit, pass it to the rate limiter. Caching the limit locally (in-memory with TTL) reduces database load. Fair queuing: if you want to prevent one heavy user from starving others, maintain separate queues per user and round-robin across them (complex, usually done in the API gateway).

**Weak answer signals:** "Just use a counter in the database and check it on every request" (database is too slow, race conditions, no atomicity). "Use a thread-local variable" (does not work in distributed systems). Not mentioning Retry-After header. Not considering what happens when Redis is down.

---

### 2. How do you prevent cascading failures?

**Strong answer:** Cascading failure happens when one service's failure overwhelms upstream callers, which then fail and overwhelm their callers. Prevention:

1. **Timeouts**: Every call has a timeout. If a service hangs, callers fail fast instead of waiting indefinitely and exhausting threads.
2. **Bulkheads**: Isolate dependencies with separate thread pools. If payment service is slow, only the payment pool exhausts—search and catalog remain healthy.
3. **Circuit breakers**: Stop calling a failing service. When the circuit is open, fail immediately instead of wasting threads on calls that will fail.
4. **Fallbacks**: Degrade gracefully. Recommendations fail → show featured products. Do not propagate the failure as an error.
5. **Load shedding**: Reject low-priority traffic before the system collapses. Better to serve 90% of traffic successfully than fail 100% with timeouts.
6. **Retry budgets**: Stop retrying when success rate is low. Retries amplify load; if the service is genuinely down, retrying makes it worse.

**Follow-up:** "You have a chain: API gateway → catalog → pricing → database. Pricing is slow. How does this cascade, and what breaks first?"

**Strong answer:** API gateway calls catalog (times out after 1 second). Catalog calls pricing (times out after 500 ms). Pricing calls database (query takes 5 seconds). Pricing's thread pool exhausts first (threads waiting for the database). Catalog's calls to pricing time out, retry, amplify load. API gateway's calls to catalog time out. Users see errors. Without bulkheads: all threads in the API gateway are blocked waiting for catalog, which is blocked waiting for pricing, which is blocked waiting for the database. System is deadlocked. With bulkheads: pricing's thread pool exhausts, circuit breaker trips, pricing returns fallback (cached prices), catalog continues serving, API gateway continues serving.

---

### 3. When do you retry and when do you not?

**Strong answer:** Retry only:

- **Idempotent operations** (reads, idempotent writes with tokens).
- **Transient failures** (503, connection timeout, network blip). Not permanent failures (400, 404, 401).
- **When you have time budget** (if the user request has a 1-second deadline and you have used 900 ms, do not retry).
- **When the service is recovering** (check retry budget—if success rate is <90%, stop retrying).

Do not retry:

- **Non-idempotent writes** (POST /orders without idempotency token → might double-charge).
- **Permanent errors** (400 Bad Request will not improve, 401 Unauthorized means auth token is invalid).
- **On the critical path for latency-sensitive operations** (retry adds tail latency; if p99 must be <100 ms, retries push you to 200 ms).
- **If downstream is overloaded** (retrying amplifies load; if they are returning 503, back off or stop).

**Follow-up:** "The service returns 500 Internal Server Error. Should you retry?"

**Strong answer:** It depends. 500 is ambiguous. Could be a transient bug (NullPointerException on one specific input), a saturated service (thread pool exhausted, returning 500 to shed load), or a persistent bug (every request with this data will fail). Best practice: on reads, retry with backoff (might be transient). On writes, do not retry unless you have an idempotency token. Monitor retry success rate—if retries are failing too, stop.

---

### 4. What is a circuit breaker actually protecting?

**Strong answer:** A circuit breaker protects **your threads**, not the failing service. When a dependency is down, calling it wastes threads waiting for timeouts. The circuit breaker detects the failure pattern and fails fast—immediately returns an error or calls a fallback instead of waiting for the timeout. This prevents thread pool exhaustion.

Side effect: reducing traffic to the failing service might help it recover (less load), but that is not the primary goal. Primary goal: keep the caller healthy.

**Follow-up:** "How does the circuit breaker know when to trip and when to recover?"

**Strong answer:** Sliding window tracks recent requests (count-based or time-based). When failure rate exceeds threshold (e.g., >50% failures in last 60 seconds) and minimum call count is met, trip to Open. Stay Open for a configured duration (wait period). Then transition to Half-Open, allow a few test requests. If they succeed, close the circuit (back to normal). If they fail, reopen.

**Follow-up:** "All pods have independent circuit breakers. Does this cause problems?"

**Strong answer:** Yes—inconsistent behavior. Some pods trip at different times, some stay closed. Users get inconsistent responses (one pod serves traffic, another returns errors). Solutions: (1) accept the inconsistency as temporary (it will converge once all pods trip), (2) centralize circuit breaker state in Redis (all pods share the same state—trips and closes together), (3) use a service mesh (Istio/Linkerd can coordinate circuit breakers). Tradeoff: shared state adds complexity and a single point of failure (if Redis is down, no circuit breaker).

---

### 5. How do you set a timeout?

**Strong answer:** Not a guess—derive from production telemetry. Start with the service's p99.9 latency (99.9th percentile from metrics). Add network latency (1–5 ms intra-AZ, 20–100 ms cross-region). Add buffer (20–50%). Round up. If service p99.9 is 200 ms intra-AZ, timeout = 200 + 5 + 50 = 255 ms → round to 300 ms.

Why p99.9 and not p99? Because p99 means 1 in 100 legitimate requests will timeout. At 10K req/s, that is 100 timeouts/second—too many false positives.

Why buffer? Because load increases latency. If you set the timeout exactly at p99.9, any load spike will cause timeouts.

**Follow-up:** "What if the service has a bimodal latency distribution—fast queries (10 ms) and slow queries (2 seconds)?"

**Strong answer:** Two approaches: (1) separate endpoints for fast and slow operations (GET /products fast, POST /reports slow), each with its own timeout, or (2) adaptive timeouts—track per-operation latency, adjust timeout dynamically. Most systems use (1) because it is simpler.

**Follow-up:** "Should timeout be higher in production than in staging?"

**Strong answer:** Ideally no—staging should mirror production latency. If staging is slower (smaller instances, different database), that is a staging environment problem, not a reason to inflate timeouts. Production timeouts should be based on production SLIs. Staging timeouts should be based on staging SLIs (and you should fix staging to match production).

---

### 6. Explain error budgets to a product manager

**Strong answer:** "We promise customers 99.9% uptime—that is our SLO. That means we have a budget of 43.2 minutes per month where we can be down or degraded. We can spend that budget on risky deployments, experiments, or accepting some failures from dependencies. If we use up the budget (we have had 43 minutes of errors this month), we stop deploying new features and focus on reliability—fix bugs, harden the system, reduce risk. If we have budget left, we can deploy. This balances innovation (new features) with stability (uptime). It is a contract: you (PM) get to push features as long as we stay within budget; we (engineering) get to freeze features when the budget is exhausted."

**Follow-up:** "Why not aim for 100% uptime?"

**Strong answer:** "100% uptime is impossible—dependencies fail, hardware fails, bugs exist. Also, 100% uptime is expensive. Going from 99.9% to 99.99% (from 8 hours to 52 minutes downtime per year) might require doubling infrastructure (multi-region active-active), doubling operational cost, slowing down feature velocity. The error budget lets us negotiate the tradeoff: is the extra 7 hours of uptime per year worth the cost?"

---

### 7. Your dependency is down—what does your service do?

**Strong answer:** Depends on the dependency's tier:

- **Critical dependency (payment, inventory):** Fail the request. Return an error to the user. Do not fake it—accepting an order when inventory is unknown leads to overselling.
- **Degradable dependency (recommendations, reviews):** Serve a fallback. Recommendations down → show featured products. Reviews down → omit the reviews section, show the product without them.
- **Optional dependency (analytics, notifications):** Fire-and-forget, queue for async retry. Analytics down → log the event locally, retry later. User does not care if their pageview was not tracked.

Circuit breaker trips to protect your threads. Fallback logic ensures the user gets a response (even if degraded).

**Follow-up:** "What if the dependency is up but slow (5 seconds instead of 50 ms)?"

**Strong answer:** Timeout fires. Treat it like a failure—circuit breaker counts it as a failure, might trip. Slow is often worse than down because it exhausts threads. Retry? No—retrying a slow service makes it slower. Log it, alert on it, let the owning team fix it.

---

### 8. How do you do multi-region?

**Strong answer:** Active-passive or active-active, depending on RTO/RPO requirements.

**Active-passive:** Primary region serves traffic, secondary is a warm standby (database replicating, compute minimal or idle). Failover: promote secondary database, scale up compute, update DNS to point to secondary. RTO: 5–30 minutes (depends on automation). RPO: seconds to minutes (replication lag). Cheaper, simpler, sufficient for most systems.

**Active-active:** Both regions serve traffic simultaneously. Requires conflict-free replication (CRDTs, last-write-wins, or partition by region—e.g., US users → us-east-1, EU users → eu-west-1). Failover is instant (just remove failed region from DNS). RTO: seconds. RPO: near-zero (synchronous replication or quorum writes). Expensive, complex.

**Follow-up:** "How do you handle data consistency in active-active?"

**Strong answer:** Hard problem. Options: (1) partition by region (US data in us-east, EU data in eu-west—no conflicts, but users must hit the correct region), (2) use a distributed database with consensus (CockroachDB, Spanner—handles conflicts automatically via quorum writes, adds latency), (3) last-write-wins (eventual consistency, accept that conflicting writes might overwrite each other—okay for some use cases, unacceptable for money/inventory).

---

### 9. What is shuffle sharding?

**Strong answer:** Instead of assigning each customer to one shard, assign them to multiple randomly chosen shards. If a shard fails, only customers whose entire shard set failed are affected—statistically rare.

Example: 100 shards, each customer is assigned 8 random shards. If a shard fails (1% failure rate), a customer is only affected if all 8 of their shards fail: 0.01⁸ ≈ 10⁻¹⁶ (effectively never). Contrast with static assignment (each customer on 1 shard): if a shard fails, 100% of its customers are down.

Benefit: blast radius reduction. Used by AWS Route 53 (name servers) and DynamoDB.

**Follow-up:** "How do you assign customers to shards deterministically?"

**Strong answer:** Hash the customer ID, seed a random number generator with the hash, generate N random shard indices. Same customer ID always produces the same shard set. No coordination needed—clients can recompute the assignment locally.

---

### 10. How do you test resilience?

**Strong answer:** Chaos engineering. Hypothesis-driven fault injection:

1. Define steady state (error rate <1%, p99 <500 ms).
2. Form a hypothesis ("If we kill one payment service replica, error rate stays <1% because load balancer fails over").
3. Inject the fault (kill the pod).
4. Measure impact (did SLIs hold?).
5. Document (what broke, what mitigations worked, what to fix).

Start small (staging environment), increase blast radius gradually (one pod, one AZ, one region). Use tools: Chaos Mesh, Litmus, Toxiproxy, Istio fault injection.

**Follow-up:** "When is it safe to run chaos in production?"

**Strong answer:** After you have:

1. Observability (metrics, logs, traces—you must be able to detect the failure).
2. Rollback capability (can you stop the experiment instantly?).
3. Runbooks (if the experiment causes an outage, do you know how to recover?).
4. Management buy-in (chaos is not cowboy culture—it is disciplined risk reduction).

Never run untested experiments at peak traffic. Start during low-traffic windows. Gradually increase to business hours.

---

### 11. What is a metastable failure and how do you prevent it?

**Strong answer:** Metastable failure is when a system has two equilibrium states (healthy and collapsed), and once it flips into collapsed, it cannot recover without manual intervention, even though the triggering fault cleared. Caused by positive feedback loops: slow requests → retries → more load → slower requests → more retries.

Examples: queue buildup → GC pressure → slower processing → longer queues. Connection pool exhaustion → requests queue → threads exhaust → circuit breaker trips → traffic floods to other pods → they exhaust.

**Prevention:**

1. Admission control (shed load before saturation—reject at 80% capacity).
2. Bounded queues (reject overflow instead of queueing infinitely).
3. Retry budgets (stop retrying when success rate is low).
4. GC tuning (young-gen-only GCs to avoid long pauses).

**Detection:** Throughput depressed even after load decreases. Memory or CPU saturated. Restart "fixes" it temporarily.

---

### 12. You are in an incident. The service is down. Walk me through your response.

**Strong answer:**

1. **Triage (first 2 minutes):** Check metrics—error rate, latency, throughput. Check recent changes (deployments, config pushes). Check dependencies (are they healthy?). Form an initial hypothesis.
2. **Mitigate (next 5–10 minutes):** Stop the bleeding. Rollback the bad deployment, kill the feature flag, scale up if it is a capacity issue, fail over to secondary region. Goal: restore service, not find root cause yet.
3. **Communicate (continuously):** Post status updates every 5–10 minutes. Tell users "we are aware, investigating." Tell stakeholders ETA.
4. **Verify (after mitigation):** Metrics green? Users able to transact? Run smoke tests.
5. **Post-incident review (next day):** Write a blameless postmortem. What broke, why, how we detected it, how we fixed it, what we will change to prevent recurrence. Share widely.

**Follow-up:** "Metrics show high latency, but you cannot find the cause. What do you do?"

**Strong answer:** Shed load. If the system is saturated and you do not know why, shedding low-priority traffic often restores health. Feature-flag off non-critical features. Scale up (if it is capacity). If it is a dependency, circuit breaker should have tripped—check circuit breaker state. If a circuit is stuck closed, manually trip it (feature flag, config push). Buy time to investigate.

## Level signals: Senior / Staff / Principal

**Senior Engineer (L4/L5 equivalent):**

- Configures Resilience4j circuit breakers, retries, timeouts for a single service. Reads metrics to tune thresholds.
- Responds to incidents using a runbook. Escalates when unsure.
- Writes basic chaos experiments in staging (kill a pod, verify app recovers).
- Understands error budgets conceptually but does not drive SLO definition.

**Staff Engineer (L6/L7 equivalent):**

- Designs resilience patterns across multiple services. Audits dependency graphs to identify serial dependencies that multiply failure rates. Advocates for bulkhead isolation.
- Writes runbooks. Leads incident response, coordinates cross-team mitigation. Drives postmortem process.
- Designs chaos experiments with hypothesis-driven methodology. Runs game days. Integrates chaos into CI/CD.
- Defines SLOs and error budgets for their domain. Tracks burn rate, recommends deployment freezes when budget is exhausted.
- Recognizes metastable failures and designs admission control to prevent them.

**Principal Engineer (L7/L8+ equivalent):**

- Architects multi-region DR strategy. Decides active-passive vs active-active, sizes RTO/RPO, justifies cost. Owns the org's resilience posture.
- Teaches resilience patterns org-wide. Reviews other teams' designs for hidden SPOFs and retry amplification. Mandates timeout standards.
- Drives chaos engineering culture—makes chaos experiments a requirement for critical-path launches. Negotiates game day schedules with leadership.
- Sets org-level SLOs. Builds tooling to calculate composite SLOs across dependency trees. Reports error budget burn to execs.
- Publishes thought leadership (conference talks, blog posts, internal tech specs). Is consulted during major incidents as "break-glass" escalation.
- Identifies systemic failure modes (retry amplification, thundering herds, metastable collapse) and drives platform-level fixes (mandatory timeouts in client libraries, org-wide retry budgets).

## Exit criteria

You have completed this phase when you can check every box:

- [ ] I can calculate end-to-end availability from a dependency tree and explain why serial dependencies multiply failure rates.
- [ ] I can size a timeout from production telemetry (p99.9 + network + buffer), not a guess.
- [ ] I can configure Resilience4j circuit breaker, retry, bulkhead, and time limiter with values derived from service SLIs, not defaults.
- [ ] I can explain when to retry (idempotent, transient, budget remaining) and when not to (permanent failure, non-idempotent, saturated downstream).
- [ ] I have implemented exponential backoff with full jitter and can explain why jitter prevents thundering herds.
- [ ] I can classify every dependency as critical/degradable/optional and document its fallback behavior.
- [ ] I can design a rate limiter (token bucket, sliding window, Redis Lua atomicity, Retry-After semantics).
- [ ] I can explain shuffle sharding and its blast-radius benefits.
- [ ] I have written health checks that distinguish liveness (JVM alive, no dependencies) from readiness (can serve traffic, shallow or circuit-aware).
- [ ] I can describe metastable failure, give a production example, and name two prevention strategies (admission control, bounded queues).
- [ ] I have run at least one chaos experiment (killed a pod, injected latency, observed metrics, verified graceful degradation).
- [ ] I have participated in a DR drill (promoted replica, updated config, verified recovery) and measured RTO.
- [ ] I can explain error budgets to a non-engineer and use burn rate to decide whether to deploy or stabilize.
- [ ] I can respond to an incident: triage, mitigate, communicate, verify, postmortem.
- [ ] I can review a service's resilience posture and identify gaps (missing timeouts, unbounded queues, retry amplification, liveness checking dependencies).

## Resources

**Books:**

- **"Release It! Second Edition"** by Michael T. Nygard — The canonical text for resilience engineering. Stability patterns (circuit breaker, bulkhead, timeout), capacity anti-patterns (unbounded queues, slow responses), and war stories from production. Read this first.
- **"Site Reliability Engineering"** (Google SRE Book) — Error budgets, toil reduction, postmortems, monitoring. Free online. Chapters 3 (Embracing Risk), 4 (Service Level Objectives), and 26 (Data Integrity) are most relevant.
- **"The Site Reliability Workbook"** — Practical SRE. Implementing SLOs, error budgets, incident response. Complements the SRE book.
- **"Chaos Engineering"** by Casey Rosenthal and Nora Jones (O'Reilly) — Hypothesis-driven fault injection, game days, organizational prerequisites for chaos.

**Papers:**

- **"Metastable Failures in Distributed Systems"** (Bronson et al., HotOS 2021) — Defines metastable failure, gives production examples (Netflix SPS, Facebook TAO), proposes mitigations. Essential reading.
- **"Tail at Scale"** (Dean & Barroso, CACM 2013) — Google's techniques for reducing p99 latency: hedged requests, tied requests, canary requests. Short, high-impact paper.
- **"Amazon's DynamoDB"** (OSDI 2007 for the original design; re-Invent talks for current architecture) — Describes shuffle sharding, cell-based isolation, conflict-free replication.

**Blogs and articles:**

- **AWS Builders' Library** (aws.amazon.com/builders-library) — "Timeouts, retries, and backoff with jitter", "Avoiding insurmountable queue backlogs", "Static stability using availability zones", "Caching challenges and strategies". All articles are production-grounded.
- **Marc Brooker's blog** (brooker.co.za/blog) — AWS Principal Engineer. Deep dives on shuffle sharding, metastable failures, formal methods, distributed systems theory applied to real systems.
- **Netflix Tech Blog** (netflixtechblog.com) — Hystrix postmortems, adaptive concurrency limits, Chaos Monkey evolution, regional failover stories.
- **Stripe Engineering Blog** (stripe.com/blog/engineering) — Rate limiter design, idempotency keys, API reliability patterns.
- **Google SRE Blog** (cloud.google.com/blog/products/devops-sre) — Error budgets in practice, SLO burn alerts, postmortem culture.

**Documentation:**

- **Resilience4j** (resilience4j.readme.io) — Comprehensive docs for circuit breaker, retry, bulkhead, rate limiter, time limiter. Includes Spring Boot integration examples.
- **Spring Boot Actuator** (docs.spring.io/spring-boot/reference/actuator) — Health checks, metrics, readiness/liveness probes.
- **OpenTelemetry Java** (opentelemetry.io/docs/languages/java) — Observability instrumentation (required for chaos engineering validation).

**Tools to install and learn:**

- **Resilience4j** (io.github.resilience4j:resilience4j-spring-boot3)
- **Chaos Mesh** (chaos-mesh.org) — Kubernetes-native chaos engineering.
- **Toxiproxy** (github.com/Shopify/toxiproxy) — TCP proxy for fault injection in tests.
- **Redis + Lua** — Distributed rate limiting, locking.
- **Prometheus + Grafana** — SLI dashboards, error budget tracking.
- **k6** or **Gatling** — Load testing to validate resilience under stress.
