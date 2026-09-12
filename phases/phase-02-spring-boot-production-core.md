# Phase 2 — Spring Boot Production Core

> **Weeks:** 11–17 | **Prerequisites:** Phase 1 (service boundaries, domain modeling, REST basics) | **Time budget:** 42–56 hrs
> **You finish this phase able to:**
> - Configure a Spring Boot service that survives deployment without manual intervention
> - Debug auto-configuration conflicts and bean lifecycle issues in under 10 minutes
> - Design health checks, graceful shutdown, and error responses that Kubernetes and load balancers actually understand
> - Size thread pools, connection pools, and timeouts based on load characteristics, not guesses
> - Write configuration that fails fast on startup instead of failing subtly in production
> - Explain why your @Cacheable method runs six times in a six-replica deployment and fix it

## Why this phase exists

Most Spring developers know how to make an endpoint respond; very few know how to make a service survive a bad Tuesday. The gap shows up in production: services that fall over when a database hiccups, return 200 OK for internal errors, hang indefinitely on downstream calls, restart uncleanly and lose in-flight work, or run scheduled jobs N times in an N-replica cluster.

This phase converts "it works on my machine" into "it degrades predictably under load, restarts cleanly, and tells you what it is doing". You will learn the Spring Boot machinery that almost no tutorial covers: how auto-configuration actually works and why your bean override sometimes loses, how to make a service lifecycle-aware so Kubernetes does not kill requests mid-flight, how to handle errors in a way that clients can act on, how to size every pool and timeout, and how to structure configuration so it fails on startup instead of silently using a nonsense default.

The production patterns in this phase are what distinguish a mid-level developer who "knows Spring" from a senior engineer who ships services that stay up. Every concept here has a body count: real outages, real 3 a.m. pages, real RCAs that say "we did not set a timeout" or "we forgot liveness probes query the database".

## Mental model

A production service is four things:

1. **A contract**: the API it exposes, the errors it returns, the configuration it requires, the dependencies it declares. This contract is explicit, versioned, and validated on startup.

2. **A runtime with bounded resources**: thread pools, connection pools, memory, file descriptors, CPU. Every pool has a maximum size. Every blocking call has a timeout. Every retry has a limit. Unbounded anything is an outage waiting to happen.

3. **A lifecycle**: startup (validate config, connect to dependencies), ready (accept traffic), degraded (dependency down but service alive), shutdown (drain requests, close connections, flush logs). Kubernetes and load balancers need to observe this lifecycle through health checks and signal handlers.

4. **A telemetry surface**: logs (what happened), metrics (how much, how fast, how full), traces (where time went). Instrumentation is not optional; it is how the service explains itself to you at 3 a.m.

Everything in this phase attaches to one of those four. Auto-configuration and bean lifecycle govern how the runtime assembles itself. Configuration, validation, and error handling define the contract. Health checks, graceful shutdown, and concurrency patterns make the lifecycle observable and safe. Actuator, logging, and metrics expose the telemetry surface.

## Core concepts

### Auto-configuration mechanics

Spring Boot auto-configuration is how 90% of your beans appear without you asking. Understanding it is the difference between "it just works" and "it just works, except when it does not and I have no idea why".

**How it works:** Every starter JAR has a file `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` listing fully-qualified class names of auto-configuration classes. At startup, Spring Boot loads these classes, evaluates their `@Conditional*` annotations, and registers matching beans. The `@AutoConfiguration` annotation declares ordering constraints (`before`, `after`) so configurations apply in the right sequence.

**The conditional family:**
- `@ConditionalOnClass` / `@ConditionalOnMissingClass` — checks classpath for a class
- `@ConditionalOnBean` / `@ConditionalOnMissingBean` — checks application context for a bean
- `@ConditionalOnProperty` — checks configuration property
- `@ConditionalOnResource` / `@ConditionalOnWebApplication` / `@ConditionalOnCloudPlatform` — environment checks

**The trap:** `@ConditionalOnMissingBean` fires ONLY if no bean of that type exists when the condition evaluates. If your custom `@Configuration` runs after the auto-configuration, your bean loses. Order matters. Fix it with `@AutoConfiguration(before = SomeAutoConfiguration.class)` or `@AutoConfigureBefore`.

**Debugging:** Run with `--debug` to see the condition evaluation report. It lists every auto-configuration, whether it matched, and why. If a bean you expect is missing, the report tells you which condition failed. If a bean you do not expect is present, the report tells you which auto-configuration contributed it.

**Writing a minimal auto-configuration:**

```java
// src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
com.shopkart.platform.tracing.TracingAutoConfiguration

// TracingAutoConfiguration.java
package com.shopkart.platform.tracing;

import io.micrometer.tracing.Tracer;
import org.springframework.boot.autoconfigure.AutoConfiguration;
import org.springframework.boot.autoconfigure.condition.ConditionalOnClass;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@AutoConfiguration
@ConditionalOnClass(Tracer.class) // Only if Micrometer Tracing on classpath
@ConditionalOnProperty(prefix = "shopkart.tracing", name = "enabled", matchIfMissing = true)
public class TracingAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public CorrelationIdFilter correlationIdFilter(Tracer tracer) {
        return new CorrelationIdFilter(tracer);
    }
}
```

This pattern — declare the auto-configuration in `.imports`, guard it with conditions, provide beans only if missing — is how platform teams build shared starters that services opt into without ceremony.

### Starters, BOM, and dependency management

A **starter** is a curated dependency set for a capability. `spring-boot-starter-web` pulls in Spring MVC, embedded Tomcat, Jackson, and validation. You declare the capability; the starter brings the right versions of 20+ transitive dependencies.

**The Bill of Materials (BOM):** Spring Boot ships `spring-boot-dependencies`, a POM that declares versions for hundreds of libraries. When you use the Spring Boot parent POM or import the BOM in your `<dependencyManagement>`, you inherit those versions. You declare `spring-kafka` without a version; Maven resolves it to the version Spring Boot tested.

**The platform BOM pattern:** In a multi-team organization, the platform team publishes a company BOM that imports `spring-boot-dependencies` and pins additional libraries (internal frameworks, approved Kafka/Redis/observability versions, security patches ahead of Spring's cadence). Application teams import the platform BOM. When the platform team updates a library, every service that rebuilds gets the new version. This is how you patch Log4Shell across 200 services in a day instead of a month.

```xml
<!-- Platform team ships this -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-dependencies</artifactId>
            <version>4.1.3</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
        <dependency>
            <groupId>io.micrometer</groupId>
            <artifactId>micrometer-tracing-bom</artifactId>
            <version>1.5.2</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
        <!-- Company libraries with pinned versions -->
        <dependency>
            <groupId>com.shopkart.platform</groupId>
            <artifactId>shopkart-observability-starter</artifactId>
            <version>2.4.0</version>
        </dependency>
    </dependencies>
</dependencyManagement>

<!-- Application team imports the platform BOM -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.shopkart.platform</groupId>
            <artifactId>shopkart-platform-bom</artifactId>
            <version>2026.09.1</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
        <!-- No version — inherited from BOM -->
    </dependency>
    <dependency>
        <groupId>com.shopkart.platform</groupId>
        <artifactId>shopkart-observability-starter</artifactId>
    </dependency>
</dependencies>
```

### Bean lifecycle, scopes, and the proxy trap

**Bean scopes:** `singleton` (one instance per container, the default), `prototype` (new instance per injection), `request`/`session`/`application` (web scopes). Singleton beans are instantiated once, usually at startup. Prototype beans are created on demand.

**Lifecycle hooks:** A bean can implement `InitializingBean` (`afterPropertiesSet()`) or `DisposableBean` (`destroy()`), or use `@PostConstruct` / `@PreDestroy`. The order: constructor → dependency injection → `@PostConstruct` → `afterPropertiesSet()` → bean ready. On shutdown: `@PreDestroy` → `destroy()`.

**The proxy trap:** Many Spring features work by wrapping your bean in a proxy: `@Transactional`, `@Async`, `@Cacheable`, `@Retryable`, `@Scheduled`. The proxy intercepts method calls from outside the bean. **Self-invocation does not go through the proxy.** This breaks things:

```java
@Service
public class OrderService {
    
    @Transactional
    public void processOrder(OrderRequest req) {
        // This call is NOT transactional — self-invocation bypasses proxy
        validateInventory(req.items());
        // Save order
    }
    
    @Transactional
    public void validateInventory(List<Item> items) {
        // Transaction logic
    }
}
```

**Three fixes:**

1. **Extract to another bean** (preferred): Move `validateInventory` to `InventoryService`. Cross-bean calls go through the proxy.

2. **Self-inject and call through the proxy:**

```java
@Service
public class OrderService {
    private final OrderService self;
    
    public OrderService(OrderService self) {
        this.self = self; // Spring injects the proxy
    }
    
    @Transactional
    public void processOrder(OrderRequest req) {
        self.validateInventory(req.items()); // Goes through proxy
    }
    
    @Transactional
    public void validateInventory(List<Item> items) { }
}
```

3. **Use `AopContext.currentProxy()` (invasive, discouraged):** Requires `@EnableAspectJAutoProxy(exposeProxy = true)`.

### Configuration: sources, precedence, and the discipline of fail-fast

**Property source precedence (highest to lowest):**

1. Command-line arguments (`--server.port=9090`)
2. `SPRING_APPLICATION_JSON` environment variable
3. OS environment variables (`SERVER_PORT=9090`)
4. `application-{profile}.properties` or `.yaml` outside the JAR
5. `application-{profile}.properties` or `.yaml` inside the JAR
6. `application.properties` or `.yaml` outside the JAR
7. `application.properties` or `.yaml` inside the JAR
8. `@PropertySource` annotations
9. Default properties (`SpringApplication.setDefaultProperties()`)

**Profiles:** Activate with `spring.profiles.active`. Use **profile groups** to activate related profiles together:

```yaml
spring:
  profiles:
    group:
      production:
        - prod
        - aws
        - high-throughput
```

Activate `production` to get all three.

**@ConfigurationProperties over @Value:**

`@Value("${server.timeout}")` is fine for a single property. For structured config, use `@ConfigurationProperties`:

```java
@ConfigurationProperties(prefix = "shopkart.order")
@Validated
public class OrderProperties {
    
    @NotNull
    @Min(1)
    @Max(300)
    private Duration processingTimeout = Duration.ofSeconds(30);
    
    @NotEmpty
    private String paymentServiceUrl;
    
    @PositiveOrZero
    private int maxRetries = 3;
    
    private RetryBackoff retryBackoff = new RetryBackoff();
    
    public static class RetryBackoff {
        private Duration initial = Duration.ofMillis(100);
        private Duration max = Duration.ofSeconds(5);
        private double multiplier = 2.0;
        
        // getters/setters
    }
    
    // getters/setters
}

@Configuration
@EnableConfigurationProperties(OrderProperties.class)
public class OrderConfig { }
```

**Relaxed binding:** Spring normalizes property names. These all bind to `processingTimeout`:

- `shopkart.order.processing-timeout` (kebab-case, YAML/properties)
- `shopkart.order.processingTimeout` (camelCase)
- `SHOPKART_ORDER_PROCESSINGTIMEOUT` (environment variable)

**Config trees for Kubernetes secrets:** Mount secrets as files; Spring reads them:

```yaml
spring:
  config:
    import: optional:configtree:/etc/secrets/
```

`/etc/secrets/db-password` becomes `db.password`.

**Configuration decision matrix:**

| Scenario | Recommendation | Why |
|----------|---------------|-----|
| Static config per environment | `application-{profile}.yaml` in JAR | Simple, versioned with code |
| Kubernetes | ConfigMap for config, Secret for credentials | Native, no extra services |
| Config changes without redeploy | Spring Cloud Config or AWS AppConfig | Adds complexity; only if genuinely needed |
| Secrets rotation | External Secrets Operator → K8s Secret | Kubernetes-native, auto-rotates |
| Dynamic feature flags | LaunchDarkly / Flagsmith / Unleash | Built for it; not config |

**The fail-fast discipline:** Invalid configuration should crash the service on startup, not fail requests in production. Enable validation:

```java
@Component
@Validated
public class ConfigValidator {
    
    @PostConstruct
    public void validate() {
        // Check invariants that Bean Validation cannot express
        if (properties.maxRetries() < 0) {
            throw new IllegalStateException("maxRetries must be non-negative");
        }
        if (properties.processingTimeout().isNegative()) {
            throw new IllegalStateException("processingTimeout must be positive");
        }
    }
}
```

Use `@Validated` on `@ConfigurationProperties` classes. The service will not start if validation fails. This is what you want: fast, loud failure that a readiness probe will catch before traffic arrives.

### The 12-factor checklist for Spring Boot services

1. **Codebase:** One repo, many deploys. ✓ Standard.
2. **Dependencies:** Explicitly declare; never assume system packages. ✓ Maven/Gradle + BOM.
3. **Config:** Store in environment, not code. ✓ `application-{profile}.yaml` + env vars.
4. **Backing services:** Treat as attached resources. ✓ Externalize URLs/credentials; reconnect on failure.
5. **Build, release, run:** Separate stages. ✓ CI builds immutable JAR/image; config applied at deploy.
6. **Processes:** Stateless, share-nothing. ✓ No local sessions; use Redis/DB for shared state.
7. **Port binding:** Export services via port. ✓ Embedded Tomcat on `server.port`.
8. **Concurrency:** Scale out via process model. ✓ Horizontal pod autoscaling.
9. **Disposability:** Fast startup, graceful shutdown. ✓ Graceful shutdown + preStop hook (covered below).
10. **Dev/prod parity:** Keep environments similar. ✓ Use same DB (Postgres, not H2) and same message broker in dev.
11. **Logs:** Treat as event streams. ✓ Log to stdout; platform collects (Fluent Bit → Loki/Elastic).
12. **Admin processes:** Run as one-off processes. ✓ Kubernetes Jobs for migrations; Spring Boot CLI for ad hoc tasks.

### Actuator: health, metrics, and the management port

**Health checks:** `/actuator/health` returns service health. Kubernetes needs two flavors:

- **Liveness:** Is the service alive? If not, kill it. Must be cheap. **Never check dependencies in liveness.** If your database is down, that is a readiness problem, not a liveness problem. Checking the DB in liveness means Kubernetes kills all replicas when the DB hiccups, making the outage total instead of partial.

- **Readiness:** Is the service ready to accept traffic? May check dependencies. If the payment service is down, the order service should report not-ready so the load balancer stops sending it traffic.

```yaml
management:
  endpoint:
    health:
      probes:
        enabled: true
      group:
        liveness:
          include: livenessState
        readiness:
          include: readinessState, db, kafka
  health:
    livenessstate:
      enabled: true
    readinessstate:
      enabled: true
```

**Custom health indicator:**

```java
@Component
public class PaymentServiceHealthIndicator implements HealthIndicator {
    private final WebClient paymentClient;
    
    @Override
    public Health health() {
        try {
            // Lightweight check: ping endpoint with short timeout
            paymentClient.get()
                .uri("/actuator/health/liveness")
                .retrieve()
                .toBodilessEntity()
                .timeout(Duration.ofMillis(500))
                .block();
            return Health.up().build();
        } catch (Exception e) {
            return Health.down()
                .withDetail("reason", e.getMessage())
                .build();
        }
    }
}
```

**Info endpoint:** `/actuator/info` exposes build metadata. Wire it to `git.properties` and `build-info.properties` generated by the build:

```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <executions>
        <execution>
            <goals>
                <goal>build-info</goal>
            </goals>
        </execution>
    </executions>
</plugin>
<plugin>
    <groupId>pl.project13.maven</groupId>
    <artifactId>git-commit-id-plugin</artifactId>
</plugin>
```

**Metrics:** `/actuator/prometheus` exports metrics in Prometheus format. Spring Boot auto-instruments HTTP requests, JDBC, JVM, Kafka, and more. Custom metrics via `MeterRegistry`:

```java
@Service
public class OrderService {
    private final Counter ordersCreated;
    private final Timer orderProcessingTime;
    
    public OrderService(MeterRegistry registry) {
        this.ordersCreated = registry.counter("orders.created", "type", "online");
        this.orderProcessingTime = registry.timer("orders.processing.time");
    }
    
    public OrderResponse createOrder(OrderRequest req) {
        return orderProcessingTime.record(() -> {
            // Process order
            OrderResponse resp = processOrder(req);
            ordersCreated.increment();
            return resp;
        });
    }
}
```

**Secure actuator on a separate port:**

```yaml
management:
  server:
    port: 9090 # Management port
  endpoints:
    web:
      base-path: /actuator
      exposure:
        include: health, info, prometheus, metrics
server:
  port: 8080 # Application port
```

Expose port 9090 only inside the cluster. Public traffic hits 8080. This prevents accidental exposure of `/actuator/env` or `/actuator/heapdump`.

### Lifecycle: graceful shutdown and the Kubernetes preStop sleep

**Graceful shutdown:** When SIGTERM arrives, the service should:

1. Stop accepting new requests
2. Wait for in-flight requests to complete (up to a timeout)
3. Close connections to dependencies
4. Flush logs and metrics
5. Exit

Spring Boot 4.x enables this by default:

```yaml
server:
  shutdown: graceful
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

On SIGTERM, the embedded server stops accepting new connections and waits up to 30 seconds for active requests to finish.

**The Kubernetes preStop race:** When a pod terminates, two things happen in parallel:

1. Kubernetes sends SIGTERM to the container
2. Kubernetes removes the pod from the service's endpoint list

These are **asynchronous**. The endpoint removal takes time to propagate to kube-proxy and any Ingress controllers. If the pod shuts down before the endpoint removal propagates, some requests hit a closed socket.

**Solution:** Add a `preStop` hook with a short sleep:

```yaml
spec:
  containers:
    - name: order-service
      lifecycle:
        preStop:
          exec:
            command: ["sh", "-c", "sleep 5"]
```

This gives the endpoint removal time to propagate before SIGTERM arrives. The pod stays in `Terminating` state during `preStop`, then receives SIGTERM, then graceful shutdown kicks in.

**Order matters for shutdown:** You want to shut down in this order:

1. HTTP server (stop accepting requests)
2. Kafka consumers (stop pulling messages)
3. Executors (finish async work)
4. Data sources (close connections)

Spring Boot's `SmartLifecycle` allows phasing. Kafka listeners and web servers already use appropriate phases. If you have custom components:

```java
@Component
public class BackgroundTaskExecutor implements SmartLifecycle {
    private final ExecutorService executor = Executors.newFixedThreadPool(10);
    private volatile boolean running = false;
    
    @Override
    public void start() {
        running = true;
    }
    
    @Override
    public void stop() {
        executor.shutdown();
        try {
            if (!executor.awaitTermination(20, TimeUnit.SECONDS)) {
                executor.shutdownNow();
            }
        } catch (InterruptedException e) {
            executor.shutdownNow();
        }
        running = false;
    }
    
    @Override
    public boolean isRunning() {
        return running;
    }
    
    @Override
    public int getPhase() {
        return Integer.MAX_VALUE - 100; // Shut down before data sources
    }
}
```

Lower phase shuts down first. Web servers are around `Integer.MAX_VALUE`. Data sources are at `Integer.MAX_VALUE`. Custom executors should shut down before data sources but after web servers.

### Error handling: RFC 9457 ProblemDetail and error taxonomy

**Do not return this:**

```json
{
  "timestamp": "2026-09-12T10:30:00Z",
  "status": 500,
  "error": "Internal Server Error",
  "message": "java.lang.NullPointerException: Cannot invoke \"String.length()\" because \"name\" is null",
  "path": "/api/orders"
}
```

This leaks implementation details, gives clients no actionable information, and provides no way to distinguish error types.

**Use RFC 9457 ProblemDetail:**

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(OrderNotFoundException.class)
    public ProblemDetail handleOrderNotFound(OrderNotFoundException ex) {
        ProblemDetail pd = ProblemDetail.forStatusAndDetail(
            HttpStatus.NOT_FOUND,
            ex.getMessage()
        );
        pd.setTitle("Order Not Found");
        pd.setProperty("orderId", ex.getOrderId());
        pd.setProperty("errorCode", "ORDER_NOT_FOUND");
        pd.setProperty("retryable", false);
        return pd;
    }
    
    @ExceptionHandler(PaymentServiceUnavailableException.class)
    public ProblemDetail handlePaymentServiceDown(PaymentServiceUnavailableException ex) {
        ProblemDetail pd = ProblemDetail.forStatusAndDetail(
            HttpStatus.SERVICE_UNAVAILABLE,
            "Payment service is temporarily unavailable"
        );
        pd.setTitle("Dependency Unavailable");
        pd.setProperty("errorCode", "PAYMENT_SERVICE_DOWN");
        pd.setProperty("retryable", true);
        pd.setProperty("retryAfter", 30); // seconds
        return pd;
    }
    
    @ExceptionHandler(InsufficientInventoryException.class)
    public ProblemDetail handleInsufficientInventory(InsufficientInventoryException ex) {
        ProblemDetail pd = ProblemDetail.forStatusAndDetail(
            HttpStatus.CONFLICT,
            ex.getMessage()
        );
        pd.setTitle("Business Rule Violation");
        pd.setProperty("errorCode", "INSUFFICIENT_INVENTORY");
        pd.setProperty("requestedQuantity", ex.getRequestedQuantity());
        pd.setProperty("availableQuantity", ex.getAvailableQuantity());
        pd.setProperty("retryable", false);
        return pd;
    }
    
    @ExceptionHandler(Exception.class)
    public ProblemDetail handleGenericError(Exception ex) {
        // Log the full exception server-side
        log.error("Unhandled exception", ex);
        
        ProblemDetail pd = ProblemDetail.forStatusAndDetail(
            HttpStatus.INTERNAL_SERVER_ERROR,
            "An unexpected error occurred"
        );
        pd.setTitle("Internal Server Error");
        pd.setProperty("errorCode", "INTERNAL_ERROR");
        pd.setProperty("retryable", true);
        // NEVER include the stack trace or ex.getMessage() in the response
        return pd;
    }
}
```

**Error taxonomy:**

| Type | HTTP Status | Retryable | Example |
|------|-------------|-----------|---------|
| Client error | 4xx | No | Invalid input, not found, unauthorized |
| Business rule violation | 409 or 422 | No | Insufficient inventory, duplicate order |
| Dependency unavailable | 503 | Yes | Database down, payment service timeout |
| Internal error | 500 | Maybe | Unexpected exception, bug |

**Include `errorCode` always.** Clients can switch on it. HTTP status tells the category; error code tells the specific case. `retryable` tells the client whether to retry. Never leak stack traces, SQL, or internal class names.

### Validation: Jakarta Bean Validation and cross-field validators

**Bean Validation on request DTOs:**

```java
public record CreateOrderRequest(
    @NotBlank(message = "Customer ID is required")
    String customerId,
    
    @NotEmpty(message = "Order must contain at least one item")
    @Valid // Cascade validation to nested objects
    List<OrderItemRequest> items,
    
    @NotNull
    @Pattern(regexp = "STANDARD|EXPRESS", message = "Invalid shipping method")
    String shippingMethod
) {}

public record OrderItemRequest(
    @NotBlank
    String productId,
    
    @Positive(message = "Quantity must be positive")
    int quantity
) {}

@RestController
public class OrderController {
    
    @PostMapping("/api/orders")
    public OrderResponse createOrder(@Valid @RequestBody CreateOrderRequest request) {
        // Validation happens automatically; MethodArgumentNotValidException thrown on failure
    }
}
```

Spring Boot auto-configures a validator. Enable detailed error responses:

```java
@RestControllerAdvice
public class ValidationExceptionHandler {
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ProblemDetail handleValidationErrors(MethodArgumentNotValidException ex) {
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
        pd.setTitle("Validation Failed");
        
        Map<String, String> errors = ex.getBindingResult()
            .getFieldErrors()
            .stream()
            .collect(Collectors.toMap(
                FieldError::getField,
                e -> e.getDefaultMessage() != null ? e.getDefaultMessage() : "Invalid value"
            ));
        
        pd.setProperty("errors", errors);
        pd.setProperty("errorCode", "VALIDATION_FAILED");
        return pd;
    }
}
```

**Cross-field validation:**

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = DiscountValidator.class)
public @interface ValidDiscount {
    String message() default "Invalid discount configuration";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

@ValidDiscount
public record PricingRequest(
    @PositiveOrZero BigDecimal basePrice,
    @PositiveOrZero BigDecimal discountAmount,
    @DecimalMin("0.0") @DecimalMax("100.0") BigDecimal discountPercent
) {}

public class DiscountValidator implements ConstraintValidator<ValidDiscount, PricingRequest> {
    
    @Override
    public boolean isValid(PricingRequest value, ConstraintValidatorContext context) {
        if (value == null) return true;
        
        boolean bothSet = value.discountAmount().compareTo(BigDecimal.ZERO) > 0 
                       && value.discountPercent().compareTo(BigDecimal.ZERO) > 0;
        
        if (bothSet) {
            context.disableDefaultConstraintViolation();
            context.buildConstraintViolationWithTemplate(
                "Cannot specify both discountAmount and discountPercent"
            ).addConstraintViolation();
            return false;
        }
        return true;
    }
}
```

**Validation groups:** Use different constraints per operation:

```java
public interface CreateGroup {}
public interface UpdateGroup {}

public class ProductRequest {
    @Null(groups = CreateGroup.class, message = "ID must be null when creating")
    @NotNull(groups = UpdateGroup.class, message = "ID is required when updating")
    private String id;
    
    @NotBlank(groups = {CreateGroup.class, UpdateGroup.class})
    private String name;
}

@PostMapping("/api/products")
public Product create(@Validated(CreateGroup.class) @RequestBody ProductRequest req) { }

@PutMapping("/api/products/{id}")
public Product update(@Validated(UpdateGroup.class) @RequestBody ProductRequest req) { }
```

**Validating configuration:** Apply `@Validated` to `@ConfigurationProperties` classes. The service will not start if config is invalid:

```java
@ConfigurationProperties(prefix = "shopkart.inventory")
@Validated
public class InventoryProperties {
    @NotNull
    @DurationMin(seconds = 1)
    @DurationMax(minutes = 5)
    private Duration reservationTtl;
    
    @Min(1) @Max(1000)
    private int maxReservationsPerUser;
}
```

### Concurrency: virtual threads, executor sizing, and the hard rule about bounded pools

**Platform threads vs virtual threads:**

**Plain English:** Platform threads are the traditional Java threads backed by OS threads. They are expensive (1–2 MB stack each, kernel scheduling). Virtual threads are lightweight threads managed by the JVM. You can have millions. They are cheap (a few KB each).

**Analogy:** Platform threads are like owning one taxi per passenger — expensive and you run out fast. Virtual threads are like Uber's dynamic dispatch — thousands of trips with far fewer actual drivers, because most trips spend most of their time waiting (passenger getting in, traffic lights, destination).

**In the real world:** A typical Spring Boot service on platform threads uses a Tomcat thread pool of 200. That pool exhausts when 200 requests are in-flight, even if 190 of them are blocked waiting for a database query. With virtual threads, the same service can handle 10,000 concurrent requests because blocked virtual threads yield the carrier thread.

**Mechanics:** Enable virtual threads:

```yaml
spring:
  threads:
    virtual:
      enabled: true
```

Spring Boot uses virtual threads for Tomcat request handlers, `@Async` methods, and scheduled tasks. Your code does not change. Blocking IO (JDBC, RestClient, `Thread.sleep()`) works as-is. The JVM schedules virtual threads onto a small pool of carrier threads (usually equal to CPU core count).

**What virtual threads fix:** Blocking IO scalability. If your service spends most of its time waiting on database queries, HTTP calls, or Kafka polls, virtual threads dramatically increase throughput with no code change.

**What virtual threads do NOT fix:**

- **CPU-bound work:** If a thread is computing (parsing JSON, hashing passwords, compressing data), it occupies a carrier thread. Virtual threads buy you nothing.
- **Pooled resource limits:** If your database connection pool has 20 connections, only 20 queries can run concurrently, whether on platform or virtual threads.
- **Pinning:** On JDK 24+, synchronized monitor pinning is removed (JEP 491), but native/JNI frames still pin. Check with the `jdk.VirtualThreadPinned` JFR event (default 20 ms threshold). If you see pinning, replace `synchronized` with `ReentrantLock` or refactor.

**What breaks:** Libraries that rely on thread-local state (MDC, security context) can misbehave if thread locals are not explicitly propagated. Spring's `TaskDecorator` handles this for `@Async`. For custom executors, propagate context manually.

**Thread pool sizing formulas:**

- **CPU-bound:** `threads = cores` or `cores + 1`. More threads = more context switching, slower.
- **IO-bound (platform threads):** `threads = cores × (1 + wait_time / compute_time)`. If a task waits 90% of the time and computes 10%, `wait/compute = 9`, so `threads = cores × 10`. On a 4-core machine, that is 40 threads.
- **Virtual threads:** Sizing becomes irrelevant for IO-bound work. Use an unbounded executor (`Executors.newVirtualThreadPerTaskExecutor()`), but still bound the **work queue** and have a rejection policy to prevent runaway memory growth.

**The hard rule:** Every executor must be bounded with a named rejection policy. An unbounded queue + unbounded threads = OutOfMemoryError when load spikes.

```java
@Configuration
public class ExecutorConfig {
    
    @Bean(name = "orderProcessingExecutor")
    public ThreadPoolTaskExecutor orderProcessingExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(50);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("order-processing-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.setAwaitTerminationSeconds(20);
        executor.setWaitForTasksToCompleteOnShutdown(true);
        executor.initialize();
        return executor;
    }
}

@Service
public class OrderService {
    
    @Async("orderProcessingExecutor")
    public CompletableFuture<OrderResult> processOrderAsync(OrderRequest req) {
        // Runs on the named executor
        return CompletableFuture.completedFuture(processOrder(req));
    }
}
```

**Rejection policies:**

- `AbortPolicy` (default) — throw `RejectedExecutionException`
- `CallerRunsPolicy` — caller thread runs the task (backpressure)
- `DiscardPolicy` — silently drop the task (data loss; rarely right)
- `DiscardOldestPolicy` — drop oldest queued task (rarely right)

For a web service, `CallerRunsPolicy` is usually correct: when the pool is full, the request thread runs the task, which slows down incoming requests (backpressure) instead of failing them.

**TaskDecorator for MDC/trace context propagation:**

```java
@Configuration
public class ExecutorConfig {
    
    @Bean
    public ThreadPoolTaskExecutor asyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(50);
        executor.setQueueCapacity(100);
        executor.setTaskDecorator(new MdcTaskDecorator());
        executor.initialize();
        return executor;
    }
}

public class MdcTaskDecorator implements TaskDecorator {
    @Override
    public Runnable decorate(Runnable runnable) {
        Map<String, String> contextMap = MDC.getCopyOfContextMap();
        return () -> {
            try {
                if (contextMap != null) {
                    MDC.setContextMap(contextMap);
                }
                runnable.run();
            } finally {
                MDC.clear();
            }
        };
    }
}
```

Micrometer Observation handles trace propagation automatically for `@Async` methods when virtual threads are enabled.

### Scheduling: the 6-replica @Scheduled problem and ShedLock

**The problem:** You have a method:

```java
@Scheduled(cron = "0 0 * * * *") // Every hour
public void generateDailyReport() {
    // Expensive report generation
}
```

You deploy 6 replicas. The method runs 6 times per hour, one per replica. If the task is idempotent (e.g., writing to the same S3 key), you waste resources. If it is not idempotent (e.g., sending emails, incrementing counters), you have a bug.

**Solutions:**

1. **ShedLock:** Distributed lock around the scheduled task. Only one replica runs it.

```java
@Configuration
@EnableScheduling
@EnableSchedulerLock(defaultLockAtMostFor = "10m")
public class SchedulerConfig {
    @Bean
    public LockProvider lockProvider(DataSource dataSource) {
        return new JdbcTemplateLockProvider(dataSource);
    }
}

@Service
public class ReportService {
    
    @Scheduled(cron = "0 0 * * * *")
    @SchedulerLock(name = "generateDailyReport", lockAtMostFor = "9m", lockAtLeastFor = "1m")
    public void generateDailyReport() {
        // Only one replica executes this at a time
    }
}
```

ShedLock acquires a database row lock (or Redis lock) before running the task. Other replicas skip the task if the lock is held.

2. **Leader election:** Use Kubernetes leader election (via a Lease resource). Only the leader runs scheduled tasks. More complex; worth it if you have many scheduled tasks.

3. **External scheduler:** Use a cron job, AWS EventBridge, or a dedicated scheduler service. The scheduled task calls your service's HTTP endpoint. Only one invocation per schedule.

4. **Use a queue:** Instead of `@Scheduled`, push a message to Kafka or SQS on a schedule. A consumer processes it once. Decouples scheduling from processing.

**When to use each:**

- **ShedLock** — simple, a few scheduled tasks, already have JDBC/Redis.
- **Leader election** — many scheduled tasks, want one replica to run all of them.
- **External scheduler** — scheduling logic complex, or you want scheduling separate from the service.
- **Queue** — need retries, DLQ, at-least-once guarantees, or scheduling at scale (millions of tasks).

### Caching: @Cacheable, Caffeine, and the side-effect warning

**Spring Cache abstraction:**

```java
@Configuration
@EnableCaching
public class CacheConfig {
    
    @Bean
    public CacheManager cacheManager() {
        return new CaffeineCacheManager(); // Local in-memory cache
    }
    
    @Bean
    public Caffeine<Object, Object> caffeineConfig() {
        return Caffeine.newBuilder()
            .maximumSize(10_000)
            .expireAfterWrite(5, TimeUnit.MINUTES)
            .recordStats();
    }
}

@Service
public class ProductService {
    
    @Cacheable(value = "products", key = "#productId")
    public Product getProduct(String productId) {
        // Cache miss: query database
        return productRepository.findById(productId)
            .orElseThrow(() -> new ProductNotFoundException(productId));
    }
    
    @CachePut(value = "products", key = "#product.id")
    public Product updateProduct(Product product) {
        // Update database and cache
        return productRepository.save(product);
    }
    
    @CacheEvict(value = "products", key = "#productId")
    public void deleteProduct(String productId) {
        productRepository.deleteById(productId);
    }
}
```

**Key design:** Default key is method arguments. Customize with SpEL: `key = "#user.id + '-' + #region"`. Keys must be stable and collision-free.

**TTL vs refresh-ahead:** `expireAfterWrite` evicts entries after a fixed time. **Refresh-ahead** (Caffeine's `refreshAfterWrite`) reloads the value in the background before expiration, keeping the cache hot. Use refresh-ahead for high-traffic keys where a cache miss would cause a latency spike.

**Null caching:** By default, Spring does not cache null. If a product does not exist, every request queries the database. Enable null caching:

```java
@Cacheable(value = "products", key = "#productId", unless = "#result == null")
public Optional<Product> getProduct(String productId) {
    return productRepository.findById(productId);
}
```

Or cache a sentinel like `Optional.empty()`.

**The warning:** `@Cacheable` on a method with side effects is a bug. This is wrong:

```java
@Cacheable("orderTotals")
public BigDecimal calculateOrderTotal(String orderId) {
    Order order = orderRepository.findById(orderId).orElseThrow();
    auditLog.log("Calculated total for " + orderId); // Side effect
    return order.getTotal();
}
```

On cache hit, the audit log does not run. Only cache pure functions: same inputs → same output, no side effects.

**Distributed caching:** Caffeine is local per replica. For shared state, use Redis with Spring Data Redis:

```java
@Bean
public CacheManager cacheManager(RedisConnectionFactory factory) {
    RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
        .entryTtl(Duration.ofMinutes(5))
        .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(
            new GenericJackson2JsonRedisSerializer()
        ));
    
    return RedisCacheManager.builder(factory)
        .cacheDefaults(config)
        .build();
}
```

**When NOT to cache:** Do not cache if the cost of the cache lookup + deserialization exceeds the cost of the operation. Do not cache rapidly changing data; you will spend more time invalidating than serving. Do not cache what a database index can handle.

### HTTP clients: RestClient, timeouts, and connection pool sizing

**RestClient (synchronous, recommended for blocking IO):**

```java
@Configuration
public class HttpClientConfig {
    
    @Bean
    public RestClient paymentServiceClient(RestClient.Builder builder) {
        return builder
            .baseUrl("https://payment-service.shopkart.internal")
            .defaultHeader(HttpHeaders.USER_AGENT, "order-service/1.0")
            .requestInterceptor((request, body, execution) -> {
                // Add correlation ID, auth token, etc.
                return execution.execute(request, body);
            })
            .build();
    }
}

@Service
public class PaymentService {
    private final RestClient paymentClient;
    
    public PaymentResponse processPayment(PaymentRequest req) {
        return paymentClient.post()
            .uri("/api/payments")
            .contentType(MediaType.APPLICATION_JSON)
            .body(req)
            .retrieve()
            .body(PaymentResponse.class);
    }
}
```

**The four timeouts:**

1. **Connect timeout:** How long to wait for a TCP connection. Default is infinite. Set to ~2–5 seconds.
2. **Read timeout (response timeout):** How long to wait for a response after sending the request. Default is infinite. Set based on the service's p99 latency + margin.
3. **Write timeout:** How long to wait to send the request body. Rarely set; matters for large uploads.
4. **Total timeout:** Cap the entire request (connect + write + read). Use this for hard deadlines.

**Configure with `ClientHttpRequestFactory`:**

```java
@Bean
public RestClient paymentServiceClient(RestClient.Builder builder) {
    HttpComponentsClientHttpRequestFactory factory = new HttpComponentsClientHttpRequestFactory();
    factory.setConnectTimeout(Duration.ofSeconds(3));
    factory.setConnectionRequestTimeout(Duration.ofSeconds(1)); // Time to get connection from pool
    
    return builder
        .baseUrl("https://payment-service.shopkart.internal")
        .requestFactory(factory)
        .build();
}
```

Spring Boot 4.x auto-configures a connection pool for `RestClient`. The default pool size is typically `maxConnTotal=200, maxConnPerRoute=20`. Tune this per dependency:

**Connection pool sizing formula:**

```
pool_size = (peak_requests_per_second × response_time_seconds) + buffer
```

If you make 100 req/s to the payment service and the p99 latency is 200 ms, you need `100 × 0.2 = 20` connections. Add a buffer (50%) → 30 connections.

**Per-dependency client beans:**

```java
@Bean
public RestClient paymentClient() {
    return RestClient.builder()
        .baseUrl("https://payment.shopkart.internal")
        .defaultHeader("X-Client-Id", "order-service")
        .build();
}

@Bean
public RestClient inventoryClient() {
    return RestClient.builder()
        .baseUrl("https://inventory.shopkart.internal")
        .defaultHeader("X-Client-Id", "order-service")
        .build();
}
```

Do not share a single `RestClient` across all dependencies. Each dependency has different latency characteristics and SLAs. Pool exhaustion in one should not affect another.

**WebClient (reactive, for truly async IO):**

```java
@Bean
public WebClient inventoryClient(WebClient.Builder builder) {
    return builder
        .baseUrl("https://inventory.shopkart.internal")
        .filter((request, next) -> {
            // Add correlation ID, metrics, retries
            return next.exchange(request);
        })
        .build();
}

public Mono<InventoryResponse> checkInventory(String productId) {
    return inventoryClient.get()
        .uri("/api/inventory/{productId}", productId)
        .retrieve()
        .bodyToMono(InventoryResponse.class)
        .timeout(Duration.ofMillis(500));
}
```

Use `WebClient` if you are building a reactive service (`spring-boot-starter-webflux`). For traditional blocking services, `RestClient` is simpler.

**Request/response logging with redaction:**

```java
public class RedactingLoggingInterceptor implements ClientHttpRequestInterceptor {
    private static final Logger log = LoggerFactory.getLogger(RedactingLoggingInterceptor.class);
    
    @Override
    public ClientHttpResponse intercept(HttpRequest request, byte[] body, 
                                        ClientHttpRequestExecution execution) throws IOException {
        logRequest(request, body);
        ClientHttpResponse response = execution.execute(request, body);
        logResponse(response);
        return response;
    }
    
    private void logRequest(HttpRequest request, byte[] body) {
        String bodyStr = new String(body, StandardCharsets.UTF_8);
        // Redact sensitive fields
        bodyStr = bodyStr.replaceAll("\"password\":\"[^\"]+\"", "\"password\":\"***\"");
        log.info("HTTP Request: {} {} - Body: {}", 
            request.getMethod(), request.getURI(), bodyStr);
    }
    
    private void logResponse(ClientHttpResponse response) throws IOException {
        log.info("HTTP Response: {} - Status: {}", 
            response.getStatusCode(), response.getStatusText());
    }
}
```

### Persistence basics that bite

**HikariCP pool sizing:**

HikariCP is the default connection pool in Spring Boot. The pool size formula:

```
connections = ((core_count × 2) + effective_spindle_count)
```

For cloud databases with no local spindles, `effective_spindle_count = 1`. On a 4-core machine, that is `(4 × 2) + 1 = 9` connections. **A bigger pool is usually slower**, because more threads contend for locks in the database. Start small; increase only if CPU is underutilized.

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 10
      minimum-idle: 5
      connection-timeout: 3000 # ms
      idle-timeout: 600000
      max-lifetime: 1800000
      leak-detection-threshold: 60000 # Log if connection not returned in 60s
```

**Open-session-in-view:** **Turn it OFF.**

```yaml
spring:
  jpa:
    open-in-view: false
```

OSIV keeps a Hibernate session open for the entire HTTP request. It makes lazy-loading "just work" in controllers and templates, but it holds a database connection for the entire request duration, including template rendering. This exhausts the pool under load. Disable it; fetch associations eagerly or use DTO projections.

**LazyInitializationException:** With OSIV off, accessing a lazy association outside a transaction throws `LazyInitializationException`. Fix it by:

1. **Eager fetching where needed:** `@EntityGraph` or JPQL `JOIN FETCH`.
2. **DTO projections:** Query directly into a DTO with only the fields you need.

```java
@Query("""
    SELECT new com.shopkart.order.OrderSummary(
        o.id, o.customerId, o.total, o.status
    )
    FROM Order o
    WHERE o.customerId = :customerId
    """)
List<OrderSummary> findOrderSummaries(@Param("customerId") String customerId);
```

**Transaction boundaries at the service layer:**

```java
@Service
@Transactional // All public methods transactional by default
public class OrderService {
    
    @Transactional(readOnly = true) // Optimization for read-only queries
    public Order getOrder(String orderId) {
        return orderRepository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));
    }
    
    public Order createOrder(OrderRequest req) {
        // Transaction boundary here; commits at method exit
        Order order = new Order(req.customerId(), req.items());
        orderRepository.save(order);
        eventPublisher.publish(new OrderCreatedEvent(order.id()));
        return order;
    }
}
```

Do not annotate repositories with `@Transactional`; Spring Data already handles it. Do not annotate controllers; transactions belong at the service layer where business logic lives.

### Packaging, startup, and container memory flags

**Layered JARs:** Spring Boot's layered JAR format splits the JAR into layers: dependencies, Spring Boot loader, snapshot dependencies, application classes. In a Dockerfile, copy each layer separately:

```dockerfile
FROM eclipse-temurin:25-jre-alpine AS builder
WORKDIR /app
COPY target/order-service-1.0.0.jar app.jar
RUN java -Djarmode=tools -jar app.jar extract --layers --destination extracted

FROM eclipse-temurin:25-jre-alpine
WORKDIR /app
COPY --from=builder /app/extracted/dependencies/ ./
COPY --from=builder /app/extracted/spring-boot-loader/ ./
COPY --from=builder /app/extracted/snapshot-dependencies/ ./
COPY --from=builder /app/extracted/application/ ./

ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```

Dependency layers change infrequently; Docker caches them. Application layer changes frequently; only it rebuilds. This cuts image build time from 2 minutes to 10 seconds for most changes.

**Class Data Sharing (CDS) and AOT:**

JDK 25 includes Application CDS and the Leyden AOT cache. Spring Boot supports generating a CDS archive:

```bash
java -XX:ArchiveClassesAtExit=app.jsa -jar order-service.jar
```

Run the app, then exit. The JVM writes `app.jsa` with loaded classes. Use it:

```bash
java -XX:SharedArchiveFile=app.jsa -jar order-service.jar
```

Reported speedup: 10–30% faster startup, 5–10% lower memory for the class metadata. Worth it for serverless or autoscaling workloads where pods start frequently.

**GraalVM native image trade-offs:**

- **Pros:** 50–100 ms startup (vs 2–5 seconds JVM), 50–70% lower RSS, standalone binary.
- **Cons:** 10–15 minute builds, reflection/proxy/JNI require configuration, no JIT (peak throughput ~10–20% lower), and some libraries do not support native image.

**When to use native image:**

- Serverless (AWS Lambda, GCP Cloud Run) where cold-start latency matters
- CLI tools
- Minimal containers (scratch/distroless base images)

**When NOT to use native image:**

- Long-running services where JIT throughput matters
- Heavy use of reflection or dynamic proxies (Spring Data JPA has full support now; older libraries may not)
- Frequent redeployments (build time matters)

**Container memory flags:**

```dockerfile
ENV JAVA_OPTS="-XX:MaxRAMPercentage=75.0 -XX:InitialRAMPercentage=50.0"
```

`MaxRAMPercentage` sets the heap size as a percentage of container memory. If your pod has 1 GB, `-XX:MaxRAMPercentage=75` → ~750 MB heap. **Do not use `-Xmx` in containers**; it is a fixed value that does not adapt to the container limit. The JVM also needs memory for metaspace, code cache, thread stacks, direct buffers, and native libraries. Reserve 25–30% for non-heap.

### Spring Cloud in 2026: alive, dead, and legacy

**Alive and recommended:**

- **Spring Cloud Gateway:** API gateway (reactive). Use it for edge routing, rate limiting, auth. A servlet variant (`spring-cloud-gateway-mvc`) exists for blocking stacks.
- **Spring Cloud Config:** Centralized config. Evaluate against Kubernetes ConfigMaps + External Secrets; Config Server is more complex but handles versioned config and encryption well.
- **Spring Cloud LoadBalancer:** Client-side load balancing. Use for service-to-service calls within Kubernetes.
- **Spring Cloud Stream:** Event-driven microservices with Kafka/RabbitMQ binders. Simplifies Kafka integration.
- **Spring Cloud OpenFeign:** Declarative HTTP client. Excellent for service-to-service APIs; integrates with LoadBalancer and Circuit Breaker.
- **Spring Cloud Circuit Breaker / Resilience4j:** Resilience patterns (circuit breaker, retry, rate limiter, bulkhead). Resilience4j is the implementation.
- **Spring Cloud Contract:** Consumer-driven contract testing. Use it for verifying API compatibility.

**Dead (do not use in greenfield):**

- **Spring Cloud Sleuth:** DEAD as of Spring Cloud 2022.0. Removed for Spring Boot 3+. The OTel bridge was archived in Dec 2025. **Replacement:** Micrometer Observation API + Micrometer Tracing with OTLP export. W3C `traceparent` is the default propagation format.
- **Hystrix:** Netflix stopped active development in 2018. **Replacement:** Resilience4j.
- **Ribbon:** Load balancer, no longer maintained. **Replacement:** Spring Cloud LoadBalancer.
- **Zuul 1:** Edge proxy, superseded. **Replacement:** Spring Cloud Gateway.

**Legacy but still shipping (use in brownfield, avoid in greenfield):**

- **Eureka:** Service registry. Still maintained and in Spring Cloud releases, but for greenfield Kubernetes workloads, use Kubernetes Services and DNS. Eureka matters for brownfield systems that pre-date Kubernetes or run on VMs.

**Migration order for a legacy Spring Cloud Netflix estate:**

1. **Ribbon → LoadBalancer:** Unblocks the Spring Boot 3.x upgrade.
2. **Hystrix → Resilience4j:** Separate concern; can run in parallel with Ribbon migration.
3. **Zuul → Gateway:** Edge concern; less urgent than internal service-to-service.
4. **Sleuth → Micrometer Tracing:** Required for Spring Boot 3+. Cannot delay.

## Production patterns

### Pattern: Service template (golden path skeleton)

**What:** A standardized project structure and dependency set for new services. The team's "proven path" embedded in a template repo or Maven archetype.

**When to use:** Every new service. Saves 2–4 hours of setup per service and ensures consistency (logging format, health checks, observability, error handling all identical).

**When NOT:** Specialized workloads (data pipelines, batch jobs, ML inference) where the template does not fit. Customize or skip.

**Structure:**

```
order-service/
├── src/main/java/com/shopkart/order/
│   ├── OrderServiceApplication.java
│   ├── config/
│   │   ├── SecurityConfig.java
│   │   ├── ObservabilityConfig.java
│   │   └── ExecutorConfig.java
│   ├── controller/
│   ├── service/
│   ├── repository/
│   ├── domain/
│   └── exception/
├── src/main/resources/
│   ├── application.yaml
│   ├── application-dev.yaml
│   ├── application-prod.yaml
│   └── db/migration/ (Flyway)
├── src/test/
├── k8s/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── configmap.yaml
├── Dockerfile
├── pom.xml
└── README.md
```

**Failure mode:** Template becomes a 50-file monstrosity that teams ignore. Keep it minimal: the 20% that prevents the 80% of mistakes.

### Pattern: Platform starter (done RIGHT)

**What:** A shared library that auto-configures cross-cutting concerns: correlation IDs, structured logging, auth, metrics, tracing, resilience defaults.

**When to use:** When you have more than 5 services and you are tired of copying the same logging/tracing config.

**When NOT:** Do not build a "common" JAR that contains domain logic, utilities, or business rules. That is the "shared library anti-pattern" that couples all services.

**How to do it right:**

- **Thin:** Only cross-cutting infrastructure.
- **Versioned:** Semantic versioning. Services choose when to upgrade.
- **Opt-in:** Auto-configuration with conditions. Services disable parts they do not need.
- **No domain logic:** Zero business rules, zero domain entities, zero shared DTOs.

**Example:**

```java
// shopkart-platform-observability-starter
@AutoConfiguration
@ConditionalOnClass(MeterRegistry.class)
public class ObservabilityAutoConfiguration {
    
    @Bean
    @ConditionalOnMissingBean
    public CorrelationIdFilter correlationIdFilter(Tracer tracer) {
        return new CorrelationIdFilter(tracer);
    }
    
    @Bean
    public MeterRegistryCustomizer<MeterRegistry> metricsCommonTags(
            @Value("${spring.application.name}") String appName) {
        return registry -> registry.config()
            .commonTags("application", appName, "environment", resolveEnvironment());
    }
}
```

Services add the dependency; everything auto-wires.

**Failure mode:** Platform team adds a mandatory dependency on a specific Kafka version, or a specific auth library. Now every service is stuck on that version. **Fix:** Keep dependencies `<optional>true</optional>`; provide auto-configuration that activates only if the dependency is present.

### Pattern: Health-check design

**What:** Separate liveness and readiness probes. Liveness checks if the service process is alive. Readiness checks if it is ready to serve traffic.

**Liveness:**

```yaml
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 9090
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 3
```

**Readiness:**

```yaml
readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 9090
  initialDelaySeconds: 10
  periodSeconds: 5
  failureThreshold: 2
```

**What breaks:** Checking the database in liveness. Database hiccup → Kubernetes kills all replicas → total outage. Database checks belong in readiness.

### Pattern: Config validation on startup with fail-fast

**What:** Validate configuration on startup. If invalid, crash before the readiness probe succeeds.

```java
@Component
@Validated
public class StartupConfigValidator {
    
    private final OrderProperties properties;
    
    @PostConstruct
    public void validate() {
        if (properties.maxRetries() < 0) {
            throw new IllegalStateException("maxRetries must be non-negative");
        }
        if (properties.processingTimeout().isZero() || properties.processingTimeout().isNegative()) {
            throw new IllegalStateException("processingTimeout must be positive");
        }
        // Verify external URLs are reachable
        testConnection(properties.paymentServiceUrl());
    }
    
    private void testConnection(String url) {
        try {
            RestClient.create().get().uri(url + "/actuator/health/liveness")
                .retrieve().toBodilessEntity();
        } catch (Exception e) {
            throw new IllegalStateException("Cannot reach " + url, e);
        }
    }
}
```

**When NOT:** Do not test every dependency on startup. If Kafka is down, the service can still start and report not-ready. Only validate config that must be correct for the service to function at all.

**What breaks:** Silent failures. The service starts, passes health checks, and then fails every request because `paymentServiceUrl` is wrong. Fail-fast surfaces the issue in deployment, not production.

### Pattern: Correlation-ID filter

**What:** Generate or propagate a correlation ID for every request. Log it, trace it, return it in error responses.

```java
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class CorrelationIdFilter implements Filter {
    private static final String CORRELATION_ID_HEADER = "X-Correlation-ID";
    private static final String MDC_KEY = "correlationId";
    
    private final Tracer tracer;
    
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        HttpServletResponse httpResponse = (HttpServletResponse) response;
        
        String correlationId = httpRequest.getHeader(CORRELATION_ID_HEADER);
        if (correlationId == null || correlationId.isBlank()) {
            correlationId = tracer.currentSpan() != null 
                ? tracer.currentSpan().context().traceId() 
                : UUID.randomUUID().toString();
        }
        
        MDC.put(MDC_KEY, correlationId);
        httpResponse.setHeader(CORRELATION_ID_HEADER, correlationId);
        
        try {
            chain.doFilter(request, response);
        } finally {
            MDC.remove(MDC_KEY);
        }
    }
}
```

Log format includes `correlationId`. Errors return it. Users report "order failed, correlation ID abc123". You grep logs for `abc123`, see the entire request flow.

### Pattern: Outbound client factory with timeouts + retry + metrics

**What:** A factory that creates `RestClient` or `WebClient` instances with baked-in timeouts, retries, and metrics.

```java
@Component
public class HttpClientFactory {
    
    private final MeterRegistry meterRegistry;
    
    public RestClient createClient(String serviceName, String baseUrl, Duration timeout) {
        HttpComponentsClientHttpRequestFactory factory = new HttpComponentsClientHttpRequestFactory();
        factory.setConnectTimeout(Duration.ofSeconds(2));
        factory.setConnectionRequestTimeout(Duration.ofSeconds(1));
        
        return RestClient.builder()
            .baseUrl(baseUrl)
            .requestFactory(factory)
            .requestInterceptor((request, body, execution) -> {
                Timer.Sample sample = Timer.start(meterRegistry);
                try {
                    ClientHttpResponse response = execution.execute(request, body);
                    sample.stop(Timer.builder("http.client.requests")
                        .tag("service", serviceName)
                        .tag("status", String.valueOf(response.getStatusCode().value()))
                        .register(meterRegistry));
                    return response;
                } catch (Exception e) {
                    sample.stop(Timer.builder("http.client.requests")
                        .tag("service", serviceName)
                        .tag("status", "error")
                        .register(meterRegistry));
                    throw e;
                }
            })
            .build();
    }
}

@Configuration
public class ClientConfig {
    
    @Bean
    public RestClient paymentClient(HttpClientFactory factory) {
        return factory.createClient("payment-service", 
            "https://payment.shopkart.internal", Duration.ofMillis(500));
    }
}
```

Every client gets timeout, metrics, correlation ID propagation, and retries for free.

### Pattern: Feature flag integration point

**What:** A central abstraction for feature flags. Services query it; the implementation swaps between LaunchDarkly, config files, or environment variables.

```java
public interface FeatureFlags {
    boolean isEnabled(String flagName);
    boolean isEnabled(String flagName, String userId);
}

@Service
public class ConfigBasedFeatureFlags implements FeatureFlags {
    private final Map<String, Boolean> flags;
    
    public ConfigBasedFeatureFlags(@Value("#{${feature.flags}}") Map<String, Boolean> flags) {
        this.flags = flags;
    }
    
    @Override
    public boolean isEnabled(String flagName) {
        return flags.getOrDefault(flagName, false);
    }
    
    @Override
    public boolean isEnabled(String flagName, String userId) {
        return isEnabled(flagName); // No user targeting in simple implementation
    }
}

@Service
public class OrderService {
    private final FeatureFlags featureFlags;
    
    public OrderResponse createOrder(OrderRequest req) {
        if (featureFlags.isEnabled("new-pricing-engine")) {
            return createOrderWithNewPricing(req);
        } else {
            return createOrderWithLegacyPricing(req);
        }
    }
}
```

**When NOT:** Do not use feature flags for configuration (timeouts, URLs). Use them for behavioral switches: new algorithm, new UI, rollout, A/B test.

### Pattern: Structured logging setup

**What:** Logback configured to emit JSON logs with correlation ID, trace ID, service name, timestamp, level, logger, thread, message, exception.

**logback-spring.xml:**

```xml
<configuration>
    <springProperty scope="context" name="appName" source="spring.application.name"/>
    
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <includeMdc>true</includeMdc>
            <customFields>{"application":"${appName}"}</customFields>
            <fieldNames>
                <timestamp>timestamp</timestamp>
                <message>message</message>
                <logger>logger</logger>
                <thread>thread</thread>
                <level>level</level>
                <levelValue>[ignore]</levelValue>
            </fieldNames>
        </encoder>
    </appender>
    
    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
    </root>
</configuration>
```

**What breaks:** Logging secrets. Add a redacting converter for sensitive fields. Never log raw request bodies that might contain passwords or tokens.

## How big tech does it

**Large financial institutions** (Goldman Sachs, JPMorgan, Capital One, Fidelity) run enormous Spring Boot estates. These organizations standardize heavily:

- **Platform BOM:** A central team publishes a versioned BOM that pins Spring Boot, Spring Cloud, every approved library, and internal frameworks. Application teams import this BOM and get zero choice on versions. This is how they patched Log4Shell across 1,500 services in 48 hours. The platform BOM updates monthly; security patches ship within days.

- **Golden-path service template:** A Maven archetype or template repository with pre-configured logging (Splunk/ELK integration), tracing (Dynatrace or Jaeger), auth (OAuth2 resource server wired to the corporate IdP), health checks, graceful shutdown, Kubernetes manifests, Jenkinsfile. New services start from the template and deviate only with explicit approval.

- **Production readiness review:** A checklist or automated gate before a service can deploy to production. The checklist covers: config validation, timeouts on all outbound calls, circuit breakers on dependencies, health checks, metrics exported, logs structured, secrets in vault, no unbounded executors, database migrations automated, rollback plan documented. Some shops automate parts of this with static analysis (scan for `@Async` without a named executor, scan for `RestClient` without a timeout).

- **Service mesh or sidecar standardization:** Many run Istio or Linkerd. Application code does not handle retries, timeouts, or mTLS; the mesh does. Spring services are simpler but the platform complexity moves to the mesh layer.

**Netflix** is historically tied to Spring Cloud. They open-sourced Eureka, Ribbon, Hystrix, and Zuul because they solved Netflix's problems in 2012–2016. But Netflix stopped active development on most of these by 2018. They moved to a different architecture: GraphQL federation at the edge (replaced Zuul), gRPC for service-to-service (replaced REST/Ribbon), and custom resilience libraries (replaced Hystrix). Netflix still runs Spring Boot services internally, but the Spring Cloud Netflix stack is legacy even at Netflix. The lesson: **a company's open-source contribution does not mean they still use it the same way**.

**Alibaba** forked Spring Cloud to create **Spring Cloud Alibaba**, which replaces Netflix components with Alibaba's internal tools: **Nacos** (service discovery + config, replaces Eureka + Config Server), **Sentinel** (rate limiting + circuit breaking, replaces Hystrix), **Seata** (distributed transactions). This ecosystem exists because Alibaba's scale (Singles' Day traffic spikes to tens of millions of requests per second) and deployment model (mostly on-premise data centers in China, not AWS) created different constraints than Western companies. Spring Cloud Alibaba is widely used in China but rare elsewhere. Mention it in interviews only if the company operates in China or you see Nacos/Sentinel in the job description.

**Zalando** (European e-commerce, ~50 million customers) publicly documented their **RESTful API and Event guidelines** (https://opensource.zalando.com/restful-api-guidelines/). These are enforced platform standards: services must version APIs with media types, use RFC 9457 ProblemDetail for errors, include correlation IDs, expose Prometheus metrics, and emit structured logs. Zalando built linters that fail CI if a service violates the guidelines. This is the model at most platform-oriented companies: standards are not suggestions; they are automated gates.

**Google and Meta do not run Spring Boot** in production for user-facing services. Google uses an internal RPC framework (Stubby, the ancestor of gRPC), Borg (the ancestor of Kubernetes), and languages like C++, Go, and Java with internal frameworks (not Spring). Meta uses Hack (PHP variant), C++, Rust, and Python, with Thrift for RPC. Both companies have built ecosystems that predate the Spring Boot / microservices wave and have no reason to adopt it. **Why this matters for interviews:** If you interview at Google or Meta for backend roles, focus on distributed systems fundamentals (consensus, CAP, replication, sharding, RPC, observability) and less on Spring-specific knowledge. They will ask "how would you design X" (rate limiter, notification system, feed ranking) and care about systems thinking, not whether you know `@Transactional` propagation levels. If you interview at banks, fintechs (Stripe, Robinhood, Coinbase), SaaS companies (Atlassian, Shopify, Confluent), or e-commerce (Amazon uses a lot of Java), Spring Boot is in the stack and detailed knowledge is valuable.

**The universal pattern** at every scaled organization with Spring Boot:

1. **A golden-path service template** that embeds the platform's opinions.
2. **A thin platform starter library** that auto-configures cross-cutting concerns (tracing, metrics, auth, logging format, error handling).
3. **A production readiness review** (manual or automated) that checks for unbounded resources, missing health checks, and failure to follow standards.

Services that follow the golden path deploy in 20 minutes. Services that deviate spend 2 weeks in review and get pushback. The platform team's job is to make the right thing the easy thing.

## Best-practice checklist

Use this checklist before deploying a Spring Boot service to production. Each item is a real outage prevented.

### Contract

- [ ] Every endpoint documented with OpenAPI or Spring REST Docs
- [ ] Errors return RFC 9457 `ProblemDetail` with `errorCode` and `retryable` fields
- [ ] No stack traces, SQL, or internal class names in error responses
- [ ] Input validation on all `@RequestBody` and `@RequestParam`
- [ ] Validation errors return 400 with field-level detail
- [ ] API versioned (URL path, media type, or header)
- [ ] Breaking changes follow deprecation policy (mark deprecated → wait N weeks → remove)

### Configuration

- [ ] All config externalized (no hardcoded URLs, credentials, timeouts)
- [ ] `@ConfigurationProperties` used for structured config
- [ ] `@Validated` on config classes; service fails on startup if config invalid
- [ ] Secrets loaded from Kubernetes Secrets or vault, never in `application.yaml`
- [ ] Config defaults are safe (no timeout = infinity, no pool size = 200 is wrong)
- [ ] Per-environment config in `application-{profile}.yaml`
- [ ] `spring.profiles.active` set via environment variable, not baked into JAR

### Resilience

- [ ] Every outbound HTTP call has a connect timeout (2–5 seconds)
- [ ] Every outbound HTTP call has a read timeout (based on dependency's p99 + margin)
- [ ] Circuit breaker on flaky dependencies (Resilience4j `@CircuitBreaker`)
- [ ] Retries with exponential backoff on idempotent operations only
- [ ] Bulkhead (separate thread pool or Resilience4j bulkhead) for each critical dependency
- [ ] Fallback behavior for degraded mode (e.g., serve stale cache if dependency is down)
- [ ] Rate limiting on public endpoints (Resilience4j `@RateLimiter` or API Gateway)
- [ ] Database connection pool sized correctly (formula: `(cores × 2) + 1`, not 200)
- [ ] All executors bounded with a rejection policy (no unbounded queues)
- [ ] Kafka consumers have `max.poll.records` and processing timeout tuned to message size

### Lifecycle

- [ ] Graceful shutdown enabled (`server.shutdown=graceful`, `spring.lifecycle.timeout-per-shutdown-phase=30s`)
- [ ] Kubernetes `preStop` hook with 5-second sleep to allow endpoint removal propagation
- [ ] Liveness probe checks only service process health, never dependencies
- [ ] Readiness probe checks dependencies (database, Kafka, downstream services)
- [ ] Health checks on separate management port (9090), not exposed to public internet
- [ ] Startup time under 30 seconds (measured from pod creation to readiness)
- [ ] Service handles SIGTERM and finishes in-flight requests before exit

### Observability

- [ ] Structured JSON logs with correlation ID, trace ID, service name, level, timestamp
- [ ] Logs written to stdout (not files); platform collects them
- [ ] No secrets, PII, or raw request bodies in logs
- [ ] `/actuator/prometheus` endpoint exposes metrics
- [ ] Custom business metrics instrumented (`MeterRegistry.counter`, `Timer`)
- [ ] Distributed tracing configured (Micrometer Tracing + OTLP export to Jaeger/Tempo)
- [ ] Every outbound HTTP call and Kafka message tagged with trace context
- [ ] `/actuator/info` exposes build version, git commit, timestamp
- [ ] Alerts defined for error rate, latency p99, and resource exhaustion (CPU, memory, connections)

### Security

- [ ] Actuator endpoints secured (management port internal-only or auth-protected)
- [ ] No `/actuator/env`, `/actuator/heapdump` exposed to the internet
- [ ] Authentication on all endpoints except health checks and public APIs
- [ ] Authorization enforced (roles, scopes, or policies)
- [ ] Secrets rotated automatically (External Secrets Operator, AWS Secrets Manager rotation)
- [ ] HTTPS only; HTTP redirects to HTTPS or rejected
- [ ] CORS policy restrictive (allow-list origins, not `*`)
- [ ] Input sanitized to prevent injection (SQL, XSS, command injection)
- [ ] Dependencies scanned for CVEs (Dependabot, Snyk, or Trivy in CI)

### Performance

- [ ] Virtual threads enabled if service is IO-bound (`spring.threads.virtual.enabled=true`)
- [ ] Hibernate `open-in-view` disabled (`spring.jpa.open-in-view=false`)
- [ ] Database queries use indexes on filter/join columns
- [ ] N+1 queries eliminated (use `@EntityGraph` or DTO projections)
- [ ] Caching applied to expensive, frequently-read, rarely-changing data
- [ ] Cache TTL short enough to prevent stale data issues
- [ ] No blocking calls in virtual threads without a bounded pool or timeout
- [ ] JVM heap sized to 70–80% of container memory (`-XX:MaxRAMPercentage=75`)
- [ ] GC tuned for workload (G1 for general, ZGC for low pause, Parallel for batch throughput)

### Operations

- [ ] Container image uses layered JARs for fast rebuilds
- [ ] Image built with a non-root user
- [ ] Image scanned for vulnerabilities (Trivy, Grype)
- [ ] Kubernetes resource requests and limits set (CPU, memory)
- [ ] Horizontal pod autoscaler configured based on CPU or custom metrics
- [ ] Database migrations automated (Flyway or Liquibase)
- [ ] Migrations tested in non-prod before prod deploy
- [ ] Rollback plan documented and tested
- [ ] No `latest` tags in production; semantic version tags only

## Anti-patterns and war stories

### Anti-patterns

**Field injection:**

```java
// WRONG
@Service
public class OrderService {
    @Autowired
    private OrderRepository orderRepository;
}
```

Why wrong: Harder to test (need reflection or Spring context), hides dependencies (no compile error if you forget `@Autowired`), no immutability. Use constructor injection:

```java
// RIGHT
@Service
public class OrderService {
    private final OrderRepository orderRepository;
    
    public OrderService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }
}
```

Spring auto-wires single-constructor classes. Dependencies are explicit, final, and testable.

**The god "common" library:**

A JAR named `shopkart-common` that contains domain entities, DTOs, utility classes, business logic, database schemas, and 40 other things. Every service depends on it. One team changes a shared DTO; 20 services break. The library becomes a coupling nightmare. **Fix:** Delete it. Duplicate DTOs across services. Share only genuinely cross-cutting infrastructure (observability, auth clients, error codes) in a thin platform starter. Business logic and domain models stay in the service that owns them.

**@Transactional on controllers:**

```java
@RestController
@Transactional // WRONG
public class OrderController {
    public Order createOrder(@RequestBody OrderRequest req) { }
}
```

Transactions span the entire HTTP request, including response serialization. A slow Jackson serialization holds a database connection open. Put `@Transactional` on service methods where the actual database work happens.

**Catching Exception and returning 200:**

```java
@PostMapping("/api/orders")
public ResponseEntity<OrderResponse> createOrder(@RequestBody OrderRequest req) {
    try {
        return ResponseEntity.ok(orderService.createOrder(req));
    } catch (Exception e) {
        log.error("Order creation failed", e);
        return ResponseEntity.ok(new OrderResponse("failed")); // WRONG: 200 for failure
    }
}
```

The client thinks the order succeeded. The monitoring thinks the endpoint is healthy. The error rate is zero. Failures are silent. **Fix:** Let exceptions propagate to `@RestControllerAdvice`. Return 4xx/5xx with `ProblemDetail`.

**Unbounded @Async executor:**

```java
@Async // Uses default executor: unbounded queue + unbounded threads
public void sendNotification(String userId) { }
```

A spike in notifications creates 10,000 threads, exhausts memory, crashes the service. **Fix:** Define a named executor with bounded pool and queue. Use `@Async("notificationExecutor")`.

**Blocking calls inside WebFlux:**

```java
@RestController
public class ProductController {
    private final ProductRepository repository; // Blocking JDBC
    
    public Mono<Product> getProduct(String id) {
        return Mono.fromCallable(() -> repository.findById(id).orElseThrow())
            .subscribeOn(Schedulers.boundedElastic()); // WRONG: masks the problem
    }
}
```

Blocking on the event loop defeats the point of reactive. Either go fully reactive (R2DBC, reactive Kafka) or use `spring-boot-starter-web` (blocking stack) with virtual threads. Do not mix.

**Actuator exposed to the internet:**

```yaml
management:
  endpoints:
    web:
      exposure:
        include: "*"
```

`/actuator/env` leaks configuration. `/actuator/heapdump` leaks memory contents (secrets, PII, session tokens). `/actuator/shutdown` lets an attacker kill the service. **Fix:** Management port internal-only, or expose only `health`, `info`, `prometheus` and secure the rest.

**Dynamic config refresh applied fleet-wide with no canary:**

A team pushes a new property to Spring Cloud Config Server. All 200 service instances poll the server within 60 seconds and apply the change. The property is wrong (timeout set to `-1`). All 200 instances start failing. **Fix:** Config changes deploy like code: test in dev, stage to one canary replica, observe, then roll out.

**`latest` dependency versions:**

```xml
<dependency>
    <groupId>com.shopkart.platform</groupId>
    <artifactId>shopkart-observability-starter</artifactId>
    <version>LATEST</version> <!-- WRONG -->
</dependency>
```

The next build pulls a version no one tested. Transitive dependency conflicts. Incompatible API change. Builds are non-reproducible. **Fix:** Pin versions in a BOM. Update them intentionally.

**Business logic in the gateway:**

```java
// In Spring Cloud Gateway
public class OrderRoutingFilter implements GlobalFilter {
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // Check if user has permission to create orders
        // Apply pricing rules
        // Validate inventory
        // ...
    }
}
```

The gateway is now a monolith. It knows about orders, pricing, inventory. **Fix:** The gateway routes and handles cross-cutting concerns (auth, rate limiting, CORS). Business logic lives in domain services.

**Using Eureka on Kubernetes for new systems:**

Kubernetes already has a service registry: Services and DNS. Adding Eureka adds operational complexity, another failure mode, and no benefit. **Fix:** Use Kubernetes Services for discovery. Eureka is for brownfield VM-based systems that predate Kubernetes.

### War story 1: Thread pool exhaustion cascades to healthy endpoints

**Setup:** The `order` service has dependencies: `payment` (p99 latency 50 ms), `inventory` (p99 200 ms), `notification` (p99 1 second). All three share the Tomcat thread pool (200 threads).

**Incident:** The `notification` service has a database deadlock. Queries that normally take 50 ms now take 30 seconds and time out. Notification calls start exhausting the thread pool. After 200 concurrent notification requests, the thread pool is full. **New order creation requests** (which do not even call notification) queue up because the pool is exhausted. The entire `order` service is down, even though only the notification dependency failed.

**Detection:** Request latency spikes across all endpoints. Thread dump shows 200 threads blocked in `NotificationServiceClient`. Prometheus metrics show `http_server_requests_active_count` at the pool limit.

**Diagnosis:** No bulkhead. A slow dependency consumed all threads. Other endpoints could not get a thread.

**Fix:**

1. Immediate: Restart the service to clear stuck threads. Fix the notification service's database deadlock.
2. Durable: Separate thread pool for notification calls. Timeout on notification client (5 seconds). Circuit breaker: after 5 consecutive failures, open the circuit and skip notifications for 30 seconds. Fallback: log "notification skipped" and succeed the order creation.

**Code:**

```java
@Configuration
public class ExecutorConfig {
    @Bean(name = "notificationExecutor")
    public ThreadPoolTaskExecutor notificationExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(50);
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.AbortPolicy());
        executor.initialize();
        return executor;
    }
}

@Service
public class NotificationService {
    
    @Async("notificationExecutor")
    @CircuitBreaker(name = "notification", fallbackMethod = "fallbackNotification")
    public CompletableFuture<Void> sendOrderNotification(String orderId) {
        notificationClient.send(orderId);
        return CompletableFuture.completedFuture(null);
    }
    
    public CompletableFuture<Void> fallbackNotification(String orderId, Exception e) {
        log.warn("Notification skipped for order {}: {}", orderId, e.getMessage());
        return CompletableFuture.completedFuture(null);
    }
}
```

Now notification failures do not bring down the service. The notification pool exhausts; other work continues.

**Lesson:** Isolate failure domains. Bulkheads (separate pools) + timeouts + circuit breakers prevent one dependency from cascading failure to the whole service.

### War story 2: Config refresh pushed a bad property fleet-wide in 30 seconds

**Setup:** Services use Spring Cloud Config Server. Config refresh is enabled with `@RefreshScope`. Config changes propagate automatically via polling (every 30 seconds) or a webhook.

**Incident:** A developer pushes a change to `application.yaml` in the config repo: `spring.datasource.hikari.maximum-pool-size: 0` (typo; meant `10`). The Config Server picks it up. Within 30 seconds, all 150 service instances poll the server, apply the refresh, and set the connection pool size to 0. Every database query fails with "no connections available". All 150 services are down.

**Detection:** Error rate spikes to 100% across the fleet. Logs show `SQLTransientConnectionException: Connection is not available`. Grafana shows connection pool metrics drop to zero.

**Diagnosis:** Config refresh applied a bad value fleet-wide with no validation or canary.

**Fix:**

1. Immediate: Revert the config change in the repo. Services re-poll and get the old value. Service health restores in 60 seconds.
2. Durable: Disable automatic refresh (`management.endpoints.web.exposure.exclude=refresh`). Config changes now require a manual `/actuator/refresh` call per instance. Deploy config changes like code: test in dev, deploy to one canary replica, observe for 10 minutes, then roll out.

**Code:**

```java
@Component
@Validated
public class ConfigValidator {
    
    @EventListener(EnvironmentChangeEvent.class)
    public void onConfigChange(EnvironmentChangeEvent event) {
        // Validate changed properties
        for (String key : event.getKeys()) {
            if (key.equals("spring.datasource.hikari.maximum-pool-size")) {
                Integer poolSize = environment.getProperty(key, Integer.class);
                if (poolSize == null || poolSize <= 0) {
                    throw new IllegalStateException("Invalid pool size: " + poolSize);
                }
            }
        }
    }
}
```

**Lesson:** Treat config as code. Test it. Canary it. Validate it. Automatic fleet-wide rollouts are dangerous. Dynamic config is useful for feature flags and tuning; structural changes (pool sizes, URLs, timeouts) deploy with the service.

### War story 3: OOMKill because container limit did not match JVM heap

**Setup:** Kubernetes pod has `resources.limits.memory: 1Gi`. The JVM starts with default heap settings (no `-Xmx`). On the host (a 64 GB machine), the JVM defaults to a heap of ~16 GB (25% of host RAM). The container has only 1 GB.

**Incident:** The service starts, runs for 10 minutes, and OOMKills when the heap grows past 1 GB. Kubernetes restarts it. It OOMKills again. The pod is CrashLoopBackOff.

**Detection:** Pod restart loop. `kubectl describe pod` shows `OOMKilled`. No logs because the process is killed mid-flight.

**Diagnosis:** The JVM heap exceeded the container limit. The kernel OOM killer stepped in.

**Fix:**

1. Immediate: Set `-XX:MaxRAMPercentage=75.0` so the JVM uses 75% of the container's 1 GB → 750 MB heap. Reserve 250 MB for non-heap (metaspace, code cache, threads, direct buffers).

```dockerfile
ENV JAVA_OPTS="-XX:MaxRAMPercentage=75.0 -XX:InitialRAMPercentage=50.0"
ENTRYPOINT ["java", "-jar", "app.jar"]
```

2. Durable: Standardize JVM flags across all services. Add a readiness check that the JVM has enough memory headroom. Monitor container memory usage and heap usage separately; alert when the gap is too small.

**Lesson:** The JVM does not know about container limits unless you tell it. Always set `-XX:MaxRAMPercentage` in containers. Reserve 25–30% of container memory for non-heap. Never rely on JVM defaults in containerized environments.

## Projects for this phase

**SPEC ONLY. These are project *specifications*, not implementations. Each defines the goal, architecture sketch, acceptance criteria, and stretch goals. Code lives elsewhere.**

### Small projects

1. **Production-ready service template**
   - **Goal:** A Maven archetype or GitHub template repository for new Spring Boot services.
   - **Scope:** Pre-configured logging (structured JSON with correlation ID), health checks (liveness + readiness), actuator on management port, graceful shutdown, dockerfile with layered JAR, Kubernetes manifests (deployment, service, configmap), basic CI pipeline (build, test, image, deploy).
   - **Acceptance criteria:** A developer runs `mvn archetype:generate` or clicks "use this template", fills in service name and package, and has a deployable service in 5 minutes. The service passes readiness, logs to stdout in JSON, exports Prometheus metrics, and survives a rolling restart with zero dropped requests.
   - **Stretch goals:** Pre-configured Resilience4j circuit breaker, Micrometer Tracing OTLP export, ArchUnit test that enforces package structure.
   - **Time box:** 6 hours.

2. **Custom Spring Boot starter for cross-cutting concerns**
   - **Goal:** A thin library that auto-configures correlation ID filter, error handling (ProblemDetail), metrics tags (service name, environment), and trace propagation.
   - **Scope:** One auto-configuration class with `@ConditionalOnClass`, `@ConditionalOnMissingBean`. A `CorrelationIdFilter`, a `@RestControllerAdvice` exception handler, a `MeterRegistryCustomizer`. Packaged as a JAR with `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`.
   - **Acceptance criteria:** A service adds the dependency, and correlation IDs appear in logs, error responses return RFC 9457 ProblemDetail, and metrics have `application` and `environment` tags. The service does nothing except add the JAR.
   - **Stretch goals:** Config properties to disable parts (`shopkart.platform.tracing.enabled=false`). Integration test that verifies auto-configuration activates only when dependencies are present.
   - **Time box:** 8 hours.

3. **Actuator health aggregator for downstream dependencies**
   - **Goal:** A custom `/actuator/health/dependencies` endpoint that checks multiple downstream services and returns their collective health.
   - **Scope:** A `HealthIndicator` that queries multiple `RestClient` instances concurrently (using virtual threads or `CompletableFuture`), aggregates results, and reports which dependencies are down.
   - **Acceptance criteria:** Endpoint returns 200 if all dependencies are up, 503 if any is down, with detail for each dependency (name, status, latency).
   - **Stretch goals:** Weighted health (e.g., payment service down = 503, cache down = 200 with warning). Circuit breaker: if a dependency is down, cache the result for 10 seconds to avoid hammering it.
   - **Time box:** 6 hours.

4. **Virtual threads vs thread pool benchmark harness**
   - **Goal:** A Spring Boot service with two endpoints: one using platform threads, one using virtual threads. Both call a slow dependency (simulated with `Thread.sleep`). Benchmark throughput and latency under load.
   - **Scope:** Two `RestClient` configurations: one with platform thread executor, one with virtual threads. A load test (Gatling or k6) that hits both endpoints with 1,000 concurrent requests. Measure p50, p99 latency and requests/second.
   - **Acceptance criteria:** Virtual thread endpoint handles 10x more concurrent requests with the same latency. Report includes graphs and a written analysis of when virtual threads help.
   - **Stretch goals:** Benchmark CPU-bound work (JSON parsing, hashing) and show virtual threads buy nothing. Pinning detection with JFR events.
   - **Time box:** 8 hours.

### Large project

**ShopKart platform baseline**

- **Goal:** A production-ready platform for the ShopKart services: a service template, a platform starter, and an automated production readiness checker.
- **Scope:**
  - **Template:** Maven archetype with pre-configured logging, tracing, health checks, Kubernetes manifests, Dockerfile, CI pipeline.
  - **Starter:** `shopkart-platform-observability-starter` with correlation ID, ProblemDetail error handling, metrics, trace propagation.
  - **Readiness checker:** A CLI tool or CI step that scans a service codebase for violations: unbounded executors, missing timeouts, no health checks, actuator exposed, dependencies without circuit breakers. Fails the build if violations found.
- **Architecture sketch:**
  - `shopkart-archetype/`: Maven archetype
  - `shopkart-platform-observability-starter/`: Auto-configuration JAR
  - `shopkart-readiness-checker/`: Static analysis tool (ArchUnit or custom AST parser)
- **Acceptance criteria:**
  - A developer generates a new service from the archetype, adds one `@RestController`, and deploys it. The service logs JSON, exports metrics, passes liveness/readiness, and survives a rolling restart.
  - The readiness checker scans an existing service and catches 5 common violations (e.g., no timeout on `RestClient`, unbounded `@Async`).
  - Documentation: a README with the checklist, a migration guide for legacy services, and a runbook for operating a service built on the platform.
- **Stretch goals:**
  - Automated dependency updates: a bot that opens PRs when the platform BOM updates.
  - Pre-built dashboards (Grafana) and alerts (Prometheus) for services following the template.
  - A smoke test generator: given an OpenAPI spec, generate a test that hits every endpoint.
- **Time box:** 20–30 hours.

**See:** `projects/small-projects.md`, `projects/large-projects.md`, and `projects/project-rubric.md` for detailed rubrics and evaluation criteria.

## Interview drilldown

### 1. How does Spring Boot auto-configuration work?

**Strong answer:**

Auto-configuration classes are listed in `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`. At startup, Spring Boot loads these classes and evaluates their `@Conditional*` annotations (`@ConditionalOnClass`, `@ConditionalOnMissingBean`, `@ConditionalOnProperty`). If conditions match, Spring registers the beans. Auto-configurations can declare ordering constraints (`@AutoConfiguration(before/after)`). You can debug with `--debug` flag to see the condition evaluation report.

**Follow-up:** What happens if your custom bean and an auto-configured bean conflict?

**Strong follow-up answer:** If you define a bean of the same type, auto-configuration typically has `@ConditionalOnMissingBean`, so your bean wins. If that does not work, use `@AutoConfiguration(before = SomeAutoConfig.class)` to run your config first, or exclude the auto-config with `@SpringBootApplication(exclude = {...})`.

**Weak answer:** "Spring Boot automatically sets up beans based on the classpath." (No detail on how, no mention of conditions or ordering.)

### 2. What happens when Spring Boot starts?

**Strong answer:**

1. `SpringApplication.run()` starts.
2. Application context is created (servlet or reactive).
3. Environment is prepared: load `application.yaml`, apply profiles, merge property sources.
4. Auto-configuration classes are loaded and evaluated.
5. User `@Configuration` classes are processed.
6. Beans are instantiated in dependency order.
7. `@PostConstruct` and `InitializingBean.afterPropertiesSet()` run.
8. `ApplicationRunner` and `CommandLineRunner` beans execute.
9. Embedded server starts (Tomcat/Netty).
10. Application is ready; readiness state becomes true.

**Follow-up:** When does `@PostConstruct` run relative to dependency injection?

**Strong follow-up answer:** After dependency injection completes. The order is: constructor → dependencies injected → `@PostConstruct` → `afterPropertiesSet()`.

**Weak answer:** "It loads the configuration files and starts the server." (No lifecycle detail, no mention of bean instantiation order.)

### 3. Why does `@Transactional` self-invocation not work, and how do you fix it?

**Strong answer:**

Spring wraps `@Transactional` beans in a proxy. The proxy intercepts calls from outside the bean and starts a transaction. Self-invocation (one method calling another method on `this`) bypasses the proxy, so no transaction starts. **Fixes:** (1) Extract the transactional method to another service and inject it. (2) Self-inject the bean (`@Autowired OrderService self`) and call `self.method()`, which goes through the proxy. (3) Use `AopContext.currentProxy()` (invasive, discouraged).

**Follow-up:** What is the default propagation level?

**Strong follow-up answer:** `REQUIRED`. If a transaction exists, the method joins it. If not, it starts a new one.

**Weak answer:** "You need to call it from another class." (Correct workaround but no explanation of why.)

### 4. Virtual threads vs WebFlux: when would you pick which?

**Strong answer:**

**Virtual threads:** Blocking IO-heavy services (JDBC, blocking HTTP clients, Kafka with blocking consumers). Enable with `spring.threads.virtual.enabled=true`. No code change. Handles 10,000+ concurrent requests where each blocks on IO. Simpler to write and debug than reactive. **Do NOT use for CPU-bound work**; virtual threads buy nothing there.

**WebFlux:** When the entire stack is reactive: R2DBC, reactive Kafka, reactive downstream clients. Good for systems that need backpressure (client pushes too fast; server slows it down). More complex: backpressure, error handling, debugging stack traces are harder.

**Decision:** For a typical Spring Boot CRUD service with JDBC and REST calls, use virtual threads (simpler). For a streaming data pipeline or a service with 100,000+ concurrent WebSocket connections, use WebFlux.

**Follow-up:** Can you mix blocking and reactive code?

**Strong follow-up answer:** Technically yes with `Mono.fromCallable(() -> blockingCall()).subscribeOn(Schedulers.boundedElastic())`, but it defeats the point. If you need blocking calls, stay on the blocking stack with virtual threads. Mixing creates complexity with no upside.

**Weak answer:** "Virtual threads are faster." (No; reactive can have lower overhead. The point is simplicity for blocking IO.)

### 5. How do you size a thread pool and a connection pool?

**Strong answer:**

**Thread pool (platform threads, IO-bound):** `threads = cores × (1 + wait_time / compute_time)`. If a task waits 90% and computes 10%, `wait/compute = 9`, so `threads = cores × 10`. For CPU-bound work, `threads = cores`.

**Virtual threads:** Sizing is irrelevant for IO-bound work. Use an unbounded executor, but bound the **work queue** or limit inbound requests to prevent runaway memory.

**Connection pool:** `pool_size = (peak_requests_per_second × response_time_seconds) + buffer`. Example: 100 req/s, 200 ms p99 latency → `100 × 0.2 = 20` connections, plus 50% buffer = 30 connections. **Bigger is not better**; more connections = more contention in the database.

**Follow-up:** What is the default HikariCP pool size formula?

**Strong follow-up answer:** `(cores × 2) + effective_spindle_count`. For cloud databases with no local disk, `effective_spindle_count = 1`. On a 4-core machine, that is 9 connections. Start there; increase only if CPU is underutilized.

**Weak answer:** "200 threads, 20 connections." (No reasoning, no adaptation to workload.)

### 6. Liveness vs readiness: what is the difference?

**Strong answer:**

**Liveness:** Is the process alive? If not, Kubernetes kills it and starts a new one. Must be cheap; never check dependencies. Checking the database in liveness means a database hiccup kills all replicas → total outage.

**Readiness:** Is the service ready to serve traffic? Checks dependencies (database, Kafka, downstream services). If not ready, Kubernetes removes the pod from the load balancer but does not kill it. When dependencies recover, the pod becomes ready again.

**Follow-up:** What happens if liveness fails?

**Strong follow-up answer:** Kubernetes sends SIGKILL after `failureThreshold` failures. The pod restarts. In-flight requests are lost unless graceful shutdown completed.

**Weak answer:** "Liveness checks if the service is running, readiness checks if it is healthy." (Vague; no mention of dependencies or the consequence of liveness failure.)

### 7. How do you achieve zero-downtime deploys for a Spring service?

**Strong answer:**

1. **Graceful shutdown:** `server.shutdown=graceful`, `spring.lifecycle.timeout-per-shutdown-phase=30s`. On SIGTERM, the server stops accepting new requests, waits for active requests to finish.
2. **Kubernetes preStop hook:** Sleep 5 seconds before SIGTERM to let endpoint removal propagate to kube-proxy.
3. **Rolling update strategy:** `maxUnavailable=0`, `maxSurge=1`. Kubernetes starts a new pod, waits for it to be ready, then kills an old pod.
4. **Readiness probe:** New pod does not receive traffic until it passes readiness.
5. **Database migrations:** Run migrations before deploying the new code. Migrations must be backward-compatible (add columns, do not drop).

**Follow-up:** What if a migration is not backward-compatible?

**Strong follow-up answer:** Deploy in two phases. Phase 1: new code reads both old and new columns. Phase 2: drop the old column after all pods run the new code.

**Weak answer:** "Use rolling updates." (No detail on graceful shutdown or preStop hook, which are critical.)

### 8. How do you externalize config in Kubernetes?

**Strong answer:**

- **ConfigMaps** for non-sensitive config (URLs, feature flags, tuning parameters).
- **Secrets** for sensitive data (database passwords, API keys).
- **External Secrets Operator** to sync secrets from a vault (AWS Secrets Manager, HashiCorp Vault) into Kubernetes Secrets.
- **Config trees:** Mount secrets as files; Spring reads them with `spring.config.import=configtree:/etc/secrets/`.
- **Environment variables:** `env` or `envFrom` in the pod spec to inject ConfigMap/Secret values.

Avoid embedding config in the JAR. Use `application-{profile}.yaml` for defaults, override with environment-specific ConfigMaps.

**Follow-up:** How do you rotate secrets without downtime?

**Strong follow-up answer:** Use External Secrets Operator with auto-rotation. Update the secret in the vault; the operator syncs it to Kubernetes. The service reloads the secret (if using config trees and a file watcher) or restarts on the next deploy. For database passwords, use connection pool refresh on secret change.

**Weak answer:** "Use ConfigMaps and Secrets." (No detail on how to inject them or rotation strategy.)

### 9. What does `@Async` actually do?

**Strong answer:**

`@Async` runs the method on a separate thread from an executor. Spring wraps the bean in a proxy that intercepts `@Async` methods, submits them to the executor, and returns a `Future` or `CompletableFuture` immediately. The default executor is unbounded (dangerous). You should define a named executor with bounded pool and queue: `@Async("myExecutor")`.

**Follow-up:** What happens if the executor queue is full?

**Strong follow-up answer:** The rejection policy executes. `AbortPolicy` throws `RejectedExecutionException`. `CallerRunsPolicy` runs the task on the caller thread (backpressure). `DiscardPolicy` silently drops the task (data loss).

**Weak answer:** "It makes the method asynchronous." (No detail on the executor or how it works.)

### 10. How do you propagate MDC and trace context to async code?

**Strong answer:**

Use a `TaskDecorator` on the executor to copy MDC before running the task and clear it after:

```java
public class MdcTaskDecorator implements TaskDecorator {
    public Runnable decorate(Runnable runnable) {
        Map<String, String> contextMap = MDC.getCopyOfContextMap();
        return () -> {
            try {
                if (contextMap != null) MDC.setContextMap(contextMap);
                runnable.run();
            } finally {
                MDC.clear();
            }
        };
    }
}
```

For trace context, Micrometer Observation auto-propagates it for `@Async` methods when virtual threads are enabled. For manual `CompletableFuture` chains, wrap the Runnable with `ContextSnapshot.captureAll()` from Reactor Context.

**Follow-up:** What if you are using virtual threads?

**Strong follow-up answer:** Virtual threads and scoped values (JDK 25+) make context propagation cleaner. MDC still works, but scoped values are the future. Micrometer Tracing handles trace context propagation automatically with virtual threads.

**Weak answer:** "Use ThreadLocal." (No; ThreadLocal does not propagate automatically. You need TaskDecorator or context capture.)

### 11. GraalVM native image: worth it?

**Strong answer:**

**Pros:** 50–100 ms startup (vs 2–5 seconds JVM), 50–70% lower memory, standalone binary. Great for serverless (Lambda, Cloud Run), CLI tools, minimal containers.

**Cons:** 10–15 minute builds, reflection/proxy/JNI need configuration, no JIT (peak throughput ~10–20% lower), some libraries lack native support.

**Decision:** Use native for serverless or autoscaling workloads where cold-start latency matters and you can afford the build time. For long-running services where throughput matters, stick with the JVM and use the Leyden AOT cache for a smaller startup win with no build time hit.

**Follow-up:** What is the Leyden AOT cache?

**Strong follow-up answer:** A training run generates a cache of loaded/linked classes and profiles. The JVM reuses this cache on subsequent runs, improving startup by ~20–30% without leaving HotSpot. Shipping in JDK 24–26. Simpler than native image, no code changes.

**Weak answer:** "Native image is faster." (Startup yes, peak throughput no. Oversimplified.)

### 12. How do you prevent a slow dependency from killing your service?

**Strong answer:**

1. **Separate thread pool (bulkhead):** The slow dependency gets its own executor. It exhausts; other work continues.
2. **Timeout:** Every outbound call has a connect timeout (2–5 seconds) and read timeout (based on p99 + margin).
3. **Circuit breaker:** After N consecutive failures, stop calling the dependency for a period. Return a fallback or fail fast.
4. **Retry with backoff:** Retry idempotent operations, but with exponential backoff and a max retry limit. Do not hammer a failing service.
5. **Fallback:** Serve stale cache, degrade gracefully, or fail the request with a clear error (503 "payment service unavailable, retry later").

**Follow-up:** What metrics would you alert on?

**Strong follow-up answer:** Circuit breaker state (alert when open), dependency latency p99 (alert when exceeds SLA), error rate on dependency calls, thread pool exhaustion (`threads_active / threads_max > 0.8`).

**Weak answer:** "Use a timeout." (Partial; does not prevent thread exhaustion or cascading failures.)

## Level signals: Senior / Staff / Principal

### Senior Engineer (2–5 years)

**You know:** How to build a Spring Boot service that works. You configure health checks, write `@RestController` endpoints, use `@Transactional`, externalize config, write unit tests, deploy to Kubernetes.

**You debug:** You can read a stack trace, use `--debug` to see auto-configuration, check actuator endpoints, and fix most issues in under 30 minutes.

**You ask:** "How should I structure this?" "What is the best practice for caching here?" "Should I use WebFlux or blocking?"

**Interviews test:** Can you write a REST API? Do you know `@Transactional` propagation? Can you configure a connection pool? Do you understand health checks?

**Production blind spots:** You might not think about thread pool exhaustion, graceful shutdown, or circuit breakers until an outage forces you to. You have not operated a service through a Black Friday traffic spike or a dependency outage.

### Staff Engineer (5–10 years)

**You design:** You design the service template, the platform starter, the production readiness checklist. You decide the observability stack, the resilience patterns, the config management strategy. You write ADRs and convince the team.

**You prevent:** You see the outage before it happens. You spot unbounded executors in code review, add circuit breakers on flaky dependencies, tune pool sizes based on load tests, and build dashboards that surface problems before they page.

**You teach:** Junior engineers ask you "why does my `@Cacheable` not work" and you explain the proxy trap. You run workshops on graceful shutdown and write runbooks.

**You operate:** You have debugged thread dumps, heap dumps, and JFR profiles in production. You have tuned GC, diagnosed memory leaks, and fixed a cascading failure at 3 a.m.

**Interviews test:** Design a resilient order service. How do you size pools? What happens during a database failover? How do you achieve zero-downtime deploys? Walk through a production incident you debugged. What metrics do you alert on?

**You ship:** You own the full lifecycle. You are paged when it breaks. You write the postmortem. You prevent the same failure from happening again.

### Principal Engineer (10+ years)

**You set direction:** You decide whether the company adopts virtual threads, GraalVM native, Kubernetes Gateway API, or stays on Istio. You publish the tech radar. You drive platform evolution.

**You scale the team:** You design systems that 50 engineers can contribute to without stepping on each other. You build the platform that makes the right thing the easy thing. You write the ADRs that teams reference for 3 years.

**You influence the industry:** You speak at conferences, write blog posts that get cited, contribute to Spring, Kubernetes, or Istio, or publish papers on distributed systems.

**You see the forest:** You trace a performance issue to a database index, a JVM GC pause, a network misconfiguration, a load balancer timeout, and an application-level retry loop, all interacting. You debug cross-system failures that span 5 teams.

**Interviews test:** Design the architecture for a new product line. How do you build a platform for 100 microservices? What are the biggest risks in this design? What is your process for migrating from monolith to microservices? Explain a hard technical decision you made and the trade-offs.

**You are called in:** When the outage is multi-team, cross-region, or involves fundamental design flaws, you are the one who gets the war room on the line and drives it to resolution.

## Exit criteria

You are ready for Phase 3 when you can check every box:

- [ ] I can explain how Spring Boot auto-configuration works and debug a missing bean in under 10 minutes.
- [ ] I know the bean lifecycle: constructor → injection → `@PostConstruct` → `afterPropertiesSet()`.
- [ ] I understand the proxy trap and can fix `@Transactional`, `@Async`, and `@Cacheable` self-invocation issues.
- [ ] I have configured separate liveness and readiness probes and know what each checks.
- [ ] I have implemented graceful shutdown and a Kubernetes preStop hook for zero-downtime deploys.
- [ ] I can write an RFC 9457 `ProblemDetail` error response with `errorCode` and `retryable` fields.
- [ ] I validate configuration on startup with `@Validated` and `@PostConstruct` checks.
- [ ] I externalize all config and secrets; no hardcoded values.
- [ ] I know when to use virtual threads (IO-bound, blocking stack) vs WebFlux (reactive, backpressure).
- [ ] I have sized a thread pool and connection pool based on workload characteristics.
- [ ] I define bounded executors for `@Async` with named rejection policies.
- [ ] I propagate MDC and trace context to async code with a `TaskDecorator`.
- [ ] I have written a custom `HealthIndicator` for a downstream dependency.
- [ ] I configure timeouts (connect, read) on every outbound HTTP call.
- [ ] I use circuit breakers on flaky dependencies and fallbacks for degraded mode.
- [ ] I know the difference between Caffeine (local cache) and Redis (distributed cache) and when to use each.
- [ ] I have disabled Hibernate `open-in-view` and fixed `LazyInitializationException` with DTO projections.
- [ ] I put `@Transactional` at the service layer, not controllers or repositories.
- [ ] I understand HikariCP pool sizing (`(cores × 2) + 1`) and tune it based on database behavior.
- [ ] I have configured structured JSON logging with correlation IDs and trace IDs.
- [ ] I expose actuator on a separate management port and secure it.
- [ ] I know the difference between Spring Cloud components that are alive (Gateway, LoadBalancer, Config), dead (Sleuth, Hystrix, Ribbon, Zuul), and legacy (Eureka).
- [ ] I have built a platform starter with auto-configuration and `@ConditionalOnClass` conditions.
- [ ] I can write a layered Dockerfile that rebuilds in 10 seconds for code changes.
- [ ] I set JVM memory flags for containers (`-XX:MaxRAMPercentage=75`).
- [ ] I have debugged a production issue using thread dumps, heap dumps, or JFR profiles.
- [ ] I follow the production readiness checklist before deploying a service.
- [ ] I can explain the trade-offs between JVM, GraalVM native, and the Leyden AOT cache.
- [ ] I have implemented distributed locking (ShedLock) for scheduled tasks in a multi-replica deployment.
- [ ] I know 5 anti-patterns (field injection, god library, `@Transactional` on controllers, unbounded executors, actuator exposed).

## Resources

**Official Spring documentation:**

- Spring Boot Reference Documentation (https://docs.spring.io/spring-boot/reference/) — The definitive source. Read the sections on Actuator, Externalized Configuration, and Production-Ready Features.
- Spring Framework Reference (https://docs.spring.io/spring-framework/reference/) — Core container, AOP, transactions, data access.
- Spring Cloud Reference (https://spring.io/projects/spring-cloud) — Each project (Gateway, LoadBalancer, Config, Stream) has its own reference guide.
- Micrometer Documentation (https://micrometer.io/docs) — Metrics, tracing, observation API.

**Books:**

- **"Cloud Native Spring in Action" by Thomas Vitale** (Manning, 2023) — Modern Spring Boot, Kubernetes, observability, cloud-native patterns. Covers Spring Boot 3+, virtual threads, and the Micrometer Observation API.
- **"Spring Boot: Up and Running" by Mark Heckler** (O'Reilly, 2021) — Practical guide to building production services. Covers auto-configuration, actuator, and deployment.
- **"Spring in Action" (6th edition) by Craig Walls** (Manning, 2022) — Comprehensive Spring reference. Covers Spring 6 and Jakarta EE 10.

**Blogs and experts:**

- **Spring Blog** (https://spring.io/blog) — Official announcements, release notes, and tutorials.
- **Baeldung** (https://www.baeldung.com/) — Hundreds of Spring tutorials. High quality, practical examples.
- **Josh Long** (https://joshlong.com/) — Spring Developer Advocate. Conference talks, screencasts, blog posts.
- **Vlad Mihalcea** (https://vladmihalcea.com/) — Hibernate and JPA expert. Deep dives on persistence, connection pooling, and performance.
- **Marco Behler** (https://www.marcobehler.com/) — Spring, JVM, and Java tutorials. Explains how things work under the hood.
- **InfoQ** and **DZone** — Industry articles on Spring, microservices, and architecture.

**Release notes and migration guides:**

- Spring Boot 4.0 Release Notes — Breaking changes from Spring Boot 3.x, new features, deprecations.
- Spring Boot 3.0 Migration Guide — The Jakarta EE namespace migration (`javax.*` → `jakarta.*`).
- Spring Cloud 2025.1 Release Notes — What is new in the current release train.

**Conference talks:**

- SpringOne (annual conference; talks archived on YouTube) — Production patterns, case studies, new features.
- Devoxx, QCon, GOTO — Microservices, resilience, Kubernetes, observability talks often feature Spring.

**OpenTelemetry and observability:**

- OpenTelemetry Java Documentation (https://opentelemetry.io/docs/instrumentation/java/) — Auto-instrumentation, manual spans, context propagation.
- Prometheus best practices (https://prometheus.io/docs/practices/) — Metric naming, cardinality, alerting.

**Testing and production readiness:**

- "Release It!" by Michael T. Nygard — Stability patterns, anti-patterns, war stories. Timeless.
- "Accelerate" by Nicole Forsgren, Jez Humble, Gene Kim — Metrics for high-performing teams.
- ArchUnit (https://www.archunit.org/) — Architecture testing. Enforce package structure, dependency rules, naming conventions.

**Kubernetes and cloud-native:**

- "Kubernetes Patterns" by Bilgin Ibryam and Roland Huß (O'Reilly, 2019) — Health probes, graceful shutdown, init containers, sidecars.
- "The Twelve-Factor App" (https://12factor.net/) — Foundational principles for cloud-native services.

**Internal platforms and SRE:**

- Google's SRE books (https://sre.google/books/) — SLIs, SLOs, error budgets, incident response.
- "Team Topologies" by Matthew Skelton and Manuel Pais — How to structure platform teams and stream-aligned teams.
