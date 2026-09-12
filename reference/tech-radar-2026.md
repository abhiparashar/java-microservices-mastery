# Tech Radar - Late 2026

**Snapshot date:** 2026-09-12. Re-verify quarterly; specific versions, project status, and vendor offerings shift faster than architecture principles.

## How to read the rings

- **Adopt:** Default choice for new production work. Battle-tested, widely supported, stable APIs, clear upgrade path. You need a specific reason NOT to use it.
- **Trial:** Production-ready but newer, narrower domain, or maturing rapidly. Use it on a real project with intent; gather evidence. Not for your most critical path on day one.
- **Assess:** Learn it, watch it, build a prototype. Not ready for production load or the ecosystem/maturity isn't there yet. Re-evaluate in 3–6 months.
- **Hold:** Do not start new work with it. Either deprecated, architecturally obsolete, or superseded by a better alternative. Existing usage is maintenance mode.

Items in **Hold** often carry migration guidance because you will meet them in legacy code.

---

## Language and runtime

### Adopt

**Java 25 LTS** - Current LTS (Sept 2025), supported through ~2028; virtual threads stable, structured concurrency and scoped values finalized, monitor pinning removed in JDK 24.

**Java 21 LTS** - Extremely common in enterprise; virtual threads introduced, good runtime for migrating from Java 17 without bleeding-edge risk.

**Virtual threads (`spring.threads.virtual.enabled=true`)** - JDK 21+ runtime feature; JDK 24 removed synchronized monitor pinning, leaving only native/JNI pinning. Observe with `jdk.VirtualThreadPinned` JFR event (20 ms threshold); Spring Boot 3.2+ has config flag for platform executor.

**G1 GC** - Default collector; balanced throughput and pause times for most workloads under 32 GB heap.

**Generational ZGC** - Low-pause collector for heaps beyond 32 GB; sub-millisecond pauses at 100+ GB heap with minimal tuning.

**Leyden AOT cache** - Incremental delivery in JDK 24–26; training run caches loaded/linked classes and profiles, reported tens-of-percent startup improvement without leaving HotSpot. Spring Boot supports it. Easiest "free lunch" optimization.

**GraalVM native image** - Production-ready for Spring apps (GA since Boot 3.0); 50–100 ms startup, several-times lower RSS versus JVM, paid for with 10–15 minute builds and no JIT peak throughput. Run in separate CI job after fast feedback.

### Trial

**Structured concurrency (JEP 480, finalized JDK 25)** - Fan-out/gather pattern for virtual threads; clearer error handling and cancellation than manual ExecutorService wiring. Trial it in new code paths doing parallel calls; retrofit when touching old fan-out logic.

**Scoped values (JEP 481, finalized JDK 25)** - Virtual-thread-safe alternative to ThreadLocal; use for request-scoped context (trace ID, user principal) in virtual-thread apps. Replacing ThreadLocal isn't urgent unless profiling shows allocation pressure.

**Parallel GC** - Throughput-first collector for batch jobs where pause times don't matter; beats G1 by 10–20% on pure number-crunching workloads. Not for request-serving services.

**Shenandoah GC** - Low-pause alternative to ZGC; similar pause profiles, slightly different tuning knobs. Choose ZGC unless your platform already standardized on Shenandoah.

### Assess

**CRaC (Coordinated Restore at Checkpoint)** - Snapshot/restore for sub-100 ms serverless cold starts; strongest for stateless Lambda-style workloads. Snapshots carry security/state caveats (credentials baked in, random seeds frozen). Watch for Spring Boot first-class support maturing.

**Project Valhalla (value types)** - JEP drafts targeting JDK 27+; inline classes to eliminate boxed primitive overhead. Not shipping yet; assess impact on collection-heavy code.

### Hold

**Java 17 LTS** - Legacy-but-alive; end of free support Sept 2029. Migrate to 21 or 25; the virtual threads and pattern-matching wins are material.

**Java 11 LTS** - End of free support Sept 2026. Extended support available but expensive; 17 is the floor for new work.

**Kotlin for backend microservices** - Not bad, but majority of Spring ecosystem, tooling, and hiring pool is Java-first. Choose Kotlin if your team is deeply experienced with it or you need coroutines for a reactive stack. Otherwise Java 25 virtual threads deliver 90% of the concurrency benefit with zero language fragmentation.

---

## Frameworks

### Adopt

**Spring Boot 4.1.x** - Current GA (Aug 2026), pairs with Spring Framework 7.0.x, Jakarta EE 11, and Spring Cloud 2025.1.x. Micrometer Observation API + Tracing baked in; virtual thread support stable. Use the Spring Boot BOM for version alignment.

**Spring Framework 7.0.x** - `jakarta.*` namespace only; servlet, reactive, and virtual thread execution models unified under a single programming model.

### Trial

**Quarkus 3.x** - Strong for Kubernetes-native, GraalVM-first workloads; dev mode with live reload is best-in-class. ArC (CDI) DI instead of Spring DI; rewriting Spring services to Quarkus is a ground-up port. Choose it for greenfield if native startup time is a hard requirement and your team will commit to the Quarkus way.

**Micronaut 4.x** - Compile-time DI, GraalVM-native, low memory. Smaller ecosystem than Spring; sensible choice for resource-constrained environments (IoT edge, tightly budgeted Lambda functions). Not worth switching from Spring unless you hit a wall on memory/startup.

**Spring Modulith** - Structure modular monoliths inside a single deployable; named modules, event publication, architecture tests, documentation generation. Trial it for new projects where you want logical service boundaries without network hops. Easier to extract a module to a separate service later than to merge over-split microservices.

### Assess

**Helidon 4.x** - Oracle's MicroProfile + reactive framework; Níma (virtual-thread-based server) competes with Spring Boot on throughput. Niche adoption; assess if you are already deep in Oracle middleware.

**Vert.x 5.x** - Reactive polyglot toolkit; excellent for event-driven, high-throughput gateways. Harder programming model than virtual threads; async/callback soup unless you use Kotlin coroutines. Assess for specialized proxies/gateways, not general CRUD services.

### Hold

**Spring Boot 3.5.x and earlier** - Wind-down mode. Spring Boot 4.0 shipped June 2026; 3.x is the upgrade path, not the target. If still on 3.x, plan the 4.x migration for Q4 2026 or Q1 2027.

**Spring Boot 2.x** - End of commercial support Nov 2025. If still on 2.x in late 2026, you are accumulating security debt. Migrate to 4.x via 3.x.

**JavaEE 8 / javax.* namespace** - Jakarta EE 9 moved to `jakarta.*` in 2020. If you are still importing `javax.servlet`, you are on unsupported libraries. Migrate to Jakarta EE 11.

---

## Spring ecosystem specifics

### Adopt

**Spring Cloud Gateway** - Reactive API gateway; non-blocking, filter chains, WebFlux-based. Current replacement for Zuul 1. Also ships a servlet/MVC variant if you cannot run reactive.

**Spring Cloud LoadBalancer** - Client-side load balancing; replaced Ribbon. Pluggable, works with Kubernetes Services or Eureka. On k8s, DNS round-robin often suffices for east-west traffic; use LoadBalancer when you need retries, circuit breaking, or weighted routing.

**Resilience4j** - Circuit breaker, rate limiter, retry, bulkhead, time limiter. Annotation-driven or programmatic. Replaced Hystrix. Use with Micrometer for metrics; integrates cleanly with Spring Boot 4.x.

**Spring Cloud Config** - Centralized external configuration backed by Git, Vault, JDBC. Still current; on Kubernetes weigh it against ConfigMaps + External Secrets Operator, which are simpler for static config. Config Server wins for dynamic property refresh and audit trails.

**Micrometer Observation API + Micrometer Tracing** - Spring Boot 4.x built-in tracing; replaces Spring Cloud Sleuth. Use `io.micrometer:micrometer-tracing-bridge-otel` to export OTLP. W3C `traceparent` is the default propagation format; B3 is legacy.

**OpenFeign** - Declarative HTTP client; mature, widely used, integrates with LoadBalancer and Resilience4j. Still the pragmatic choice for most service-to-service calls.

**Spring Data JPA (Hibernate 7.x ORM)** - ORM for relational writes and simple queries; use jOOQ for complex reads. Hibernate 7.0 (Jakarta EE 11, Java 17+) is the current major version.

**Spring Authorization Server** - OAuth 2.1 / OIDC authorization server; Spring's official replacement for the deprecated Spring Security OAuth project. Production-ready as of 1.0 (Nov 2022). Use it for first-party token issuance; Keycloak or cloud IAM for federation/multi-tenant.

### Trial

**RestClient / HTTP interfaces (Spring Framework 7.x)** - Declarative HTTP clients using Java interfaces and `@HttpExchange`; synchronous alternative to WebClient. Simpler than OpenFeign for new code; trial it on greenfield services, keep Feign where it already works.

**Spring Cloud Stream** - Abstraction over messaging (Kafka, RabbitMQ, Pulsar); function-based programming model. Trial it if you want to swap message brokers without rewriting application code. Overkill if you are Kafka-only; use Spring Kafka directly for less indirection.

**Spring Modulith** - Already listed under frameworks; belongs here too. Architectural modules, named boundaries, event publication, documentation. Trial for modular monoliths or gradual extraction.

### Assess

**RSocket with Spring** - Reactive, multiplexed, backpressure-aware RPC. Excellent protocol; narrow adoption. Assess for internal high-throughput services where HTTP/2 overhead matters.

**Spring GraphQL** - GraphQL integration using GraphQL Java. Assess if your frontend team wants GraphQL federation. Otherwise REST + OpenAPI is simpler.

### Hold

**Spring Cloud Sleuth** - Deprecated from Spring Cloud 2022.0, removed for Spring Boot 3.x; OTel bridge repo archived Dec 2025. Replace with Micrometer Observation + Micrometer Tracing. Migration: remove `spring-cloud-sleuth-zipkin`, add `micrometer-tracing-bridge-otel` and `opentelemetry-exporter-otlp`.

**Hystrix** - Netflix stopped active development 2018. Replace with Resilience4j. Migration: swap annotations (`@HystrixCommand` → `@CircuitBreaker`), rewrite config (Hystrix used Archaius or hardcoded properties; Resilience4j uses `application.yaml`).

**Ribbon** - Deprecated; removed from Spring Cloud 2022.0+. Replace with Spring Cloud LoadBalancer. Migration: Ribbon auto-wires via Eureka/static lists; LoadBalancer requires explicit `@LoadBalanced RestTemplate` or ReactorLoadBalancerExchangeFilterFunction for WebClient.

**Zuul 1** - Netflix's servlet-based gateway; blocking I/O. Replace with Spring Cloud Gateway. Migration: Zuul filters → Gateway filters (reactive); configuration structure changed. Zuul 2 (non-blocking) exists but Netflix never contributed it to Spring Cloud.

**Eureka for greenfield Kubernetes workloads** - Still maintained, still shipping in Spring Cloud, still valid for brownfield AWS/on-prem deployments. On Kubernetes, use Kubernetes Services (DNS) for service discovery; no external registry needed. Eureka adds operational burden (HA, persistence, health checks) for zero Kubernetes-native benefit.

**Spring Cloud Netflix Archaius** - Hystrix config dependency; obsolete with Hystrix. Use Spring Cloud Config or environment variables.

---

## Communication

### Adopt

**REST + OpenAPI 3.1** - HTTP/JSON APIs documented with OpenAPI; `springdoc-openapi` generates specs from annotations. Still the default for public APIs and service-to-service unless you have a specific reason for gRPC.

**gRPC (Protocol Buffers)** - Binary, HTTP/2, strongly typed, bidirectional streaming. Use for high-throughput internal east-west traffic (microservice mesh). Overkill for CRUD APIs; strong for batch sync, event streams, real-time data pipelines. Spring Boot: `grpc-spring-boot-starter` or native `grpc-java`.

**Server-Sent Events (SSE)** - One-way server push over HTTP; simple, text-based, auto-reconnect in browsers. Use for real-time notifications, live dashboards. Simpler than WebSocket when you don't need client-to-server messages.

**OpenAPI code generation (client + server stubs)** - `openapi-generator-maven-plugin` or Gradle equivalent; contract-first development. Generate Java clients from partner APIs; generate server stubs to enforce contract. Keeps API and implementation in sync.

### Trial

**GraphQL (federation)** - Query language for aggregating multiple backend services into a unified graph. Trial it if frontend teams already use GraphQL and you are federating 3+ services. Apollo Federation or Netflix DGS. Not simpler than REST for single-service APIs; complexity is in the gateway and schema stitching.

**AsyncAPI** - OpenAPI equivalent for event-driven APIs (Kafka, AMQP, MQTT). Trial for documenting message schemas and pub/sub contracts. Tooling less mature than OpenAPI but improving.

**WebSocket** - Bidirectional, persistent TCP connection over HTTP; use for chat, collaborative editing, gaming. Trial when SSE is insufficient (client needs to push). Harder to load-balance (sticky sessions) and scale than stateless HTTP.

### Assess

**RSocket** - Mentioned under Spring ecosystem; protocol-level backpressure, multiplexing, resumption. Assess for internal RPC where latency and throughput matter. Narrow adoption; gRPC has larger ecosystem.

**gRPC-Web** - gRPC over HTTP/1.1 for browser clients; requires Envoy proxy. Assess if you want to expose gRPC services to frontends. Most web apps stick with REST or GraphQL.

### Hold

**SOAP / XML-RPC** - Legacy enterprise integration. If still consuming SOAP services, generate clients with `wsdl2java`; do not expose new SOAP endpoints. Migrate to REST.

**Thrift / Avro RPC** - Succeeded by gRPC in most contexts. Avro as a serialization format (not RPC) is still valid (see Kafka schema registry).

---

## Data

### Adopt

**PostgreSQL 18.x** - Current version; PostGIS for geospatial, JSONB for semi-structured, full-text search, partitioning, replication slots for CDC. Default relational choice for transactional workloads.

**Flyway** - Schema migration as code; SQL or Java-based migrations, versioned, repeatable, checksum validation. Simple, works everywhere. Use it.

**Liquibase** - Schema migration alternative; XML/YAML/JSON/SQL; richer diff tooling, better for DBAs who want declarative schemas. Choose Flyway unless your DBA team is already Liquibase-native.

**jOOQ** - Type-safe SQL DSL; generates Java from database schema, compiles queries at build time. Use for complex queries (joins, window functions, CTEs) where JPA is painful. Pair with Spring Data JPA: JPA for writes, jOOQ for reads.

**Hibernate 7.x** - JPA 3.2 implementation, Jakarta EE 11 baseline. Spring Data JPA uses it by default. Good for simple CRUD; use jOOQ or native queries for analytics.

**Redis 8.x / Valkey 9.x** - In-memory cache/data structure store. Redis 8 is tri-licensed (RSALv2/SSPLv1/AGPLv3 since May 2025); Valkey 9 is the BSD-licensed Linux Foundation fork from Redis 7.2.4. AWS ElastiCache, GCP Memorystore, Azure Cache all support Valkey now. Application-level behavior identical; pick based on license preference and managed-service availability. Use for session storage, rate limiting, leaderboards, pub/sub.

**DynamoDB** - AWS-native key-value/document store; single-digit-millisecond latency, infinite scale, pay-per-request or provisioned capacity. Use for high-scale, single-table-design workloads where you can model access patterns up front. Not a relational replacement.

**Spring Data JPA / Spring Data JDBC** - JPA for ORM; JDBC for simpler mapping without lazy-loading/dirty-checking magic. Use JDBC for read-heavy or event-sourced aggregates.

### Trial

**R2DBC (Reactive Relational Database Connectivity)** - Reactive/non-blocking database access for PostgreSQL, MySQL, H2. Trial with Spring Data R2DBC for reactive/WebFlux apps. JVM ecosystem still JDBC-first; R2DBC is narrower adoption. Virtual threads make blocking JDBC less of a bottleneck.

**CockroachDB / YugabyteDB (distributed SQL)** - PostgreSQL-compatible distributed SQL; multi-region, serializable isolation, horizontal scale. Trial for geo-distributed writes or when single-region Postgres hits write scaling limits. More expensive (licensing, operations) than Postgres+read replicas.

**MongoDB** - Document store; flexible schema, horizontal scale, rich query language. Use for semi-structured data with unpredictable schema evolution (CMS, catalogs, IoT). Do not use as a relational substitute or for financial transactions (unless you really understand its consistency model).

**ScyllaDB** - Drop-in Cassandra replacement, rewritten in C++; 10x throughput at lower latency per ScyllaDB's benchmarks. Trial if you already know Cassandra; otherwise assess Cassandra itself first.

**ClickHouse** - Columnar OLAP database; sub-second analytics on billions of rows. Trial for time-series, logs, metrics, event analytics. Pairs well with PostgreSQL via materialized views or Kafka CDC.

**Apache Iceberg** - Table format for analytics on S3/GCS/ADLS; ACID transactions, schema evolution, time travel, partition pruning. Trial for offloading PostgreSQL OLTP data to a data lake for analytics. Integrates with Flink, Spark, Trino.

### Assess

**Cassandra** - Wide-column store; tunable consistency, masterless replication. Assess for write-heavy workloads requiring multi-datacenter replication. Steep operational learning curve; ScyllaDB may be easier.

**Neo4j / Amazon Neptune (graph databases)** - Purpose-built for graph queries (social networks, fraud detection, recommendation). Assess for domains with complex many-to-many relationships. Niche; most workloads survive on relational + JSON.

**Elasticsearch / OpenSearch** - Full-text search, logs, observability. OpenSearch is the AWS-maintained fork after Elastic's license change (2021). Assess for search-first use cases; ClickHouse increasingly competitive for log analytics at lower cost.

### Hold

**H2 in-memory database as a Postgres substitute in tests** - Dialect incompatibilities cause false positives and false negatives. Use Testcontainers with real Postgres in tests. H2 acceptable for throwaway prototypes.

**MySQL 5.x** - End of life. Migrate to MySQL 8.x or PostgreSQL.

**Oracle for greenfield projects** - Licensing cost, vendor lock-in. PostgreSQL covers 95% of use cases. Oracle only for brownfield where migration cost exceeds license cost, or specific Oracle-only features (RAC, Exadata).

---

## Messaging and streaming

### Adopt

**Kafka 4.3.x (KRaft mode)** - Distributed log; KRaft is mandatory in Kafka 4.x; ZooKeeper was removed in 4.0 (Feb 2026). Metadata lives in `__cluster_metadata` topic. Use for event streaming, CDC, inter-service async communication. Consumer groups for competing consumers; partitions for parallelism.

**Spring Kafka** - Annotation-driven Kafka producer/consumer; integrates with Spring Boot autoconfiguration, retry, error handling, metrics. Use it unless you need Kafka Streams or Flink.

**Kafka Streams** - JVM stream-processing library; stateful aggregations, joins, windowing. Runs inside your app, not a separate cluster. Use for lightweight ETL, materialized views, real-time aggregates. Heavier processing → Flink.

**RabbitMQ (quorum queues)** - Message broker; quorum queues (Raft-based replication) replaced classic mirrored queues in RabbitMQ 3.8+. Use for task queues, RPC, priority queues. Simpler operational model than Kafka; lower throughput.

**Debezium** - CDC via Kafka Connect; captures database transaction logs (PostgreSQL, MySQL, MongoDB, etc.), emits change events to Kafka. Use for event sourcing, cache invalidation, search index sync. Near-zero application code; database plugin + Kafka Connect.

**Confluent Schema Registry / Apicurio Registry** - Avro/Protobuf/JSON Schema registry for Kafka; schema evolution, compatibility checks. Use it to prevent deserialization breakage. Apicurio is open-source Red Hat alternative.

### Trial

**Kafka share groups (KIP-932)** - Queue-like semantics on Kafka (round-robin, no partition assignment). Arriving in Kafka; do not assert GA. Trial when it lands for simpler consumer scaling without partition rebalancing.

**Apache Pulsar** - Multi-tenant, geo-replicated pub/sub; tiered storage, message TTL, native multi-tenancy. Trial if you need infinite retention (S3 offload) or complex tenant isolation. Operationally heavier than Kafka.

**Amazon SQS / SNS** - Managed queue/pub-sub on AWS. Trial for AWS-native workloads where operational simplicity beats Kafka's throughput and ordering guarantees. No ordering across messages unless using FIFO queues (lower throughput).

**Google Pub/Sub / Azure Service Bus** - Cloud-managed equivalents; trial for GCP/Azure workloads. Similar tradeoffs to SQS/SNS.

### Assess

**Apache Flink** - Stateful stream processing; event time, watermarks, exactly-once, complex event processing. Assess for heavy transformations, joins across streams, windowed aggregates at scale. Operationally heavier than Kafka Streams; needs separate Flink cluster.

**Redpanda** - Kafka-compatible broker, rewritten in C++; claims lower latency and simpler operations than Kafka. Assess for greenfield if you are committed to managed Redpanda Cloud. On-prem, Kafka 4.x KRaft is mature.

**NATS / NATS JetStream** - Lightweight pub/sub, CNCF project; strong for IoT, edge, low-latency messaging. Assess for non-Java polyglot environments. Narrower JVM ecosystem than Kafka/RabbitMQ.

### Hold

**Kafka with ZooKeeper** - ZooKeeper was removed in Kafka 4.0 (Feb 2026). Migrate to KRaft. Migration: `kafka-storage format`, rolling restart, metadata migrates to `__cluster_metadata`. ZooKeeper-based clusters still run on Kafka 3.x; plan migration to 4.x in 2026–2027.

**RabbitMQ classic mirrored queues** - Deprecated in RabbitMQ 3.8+; replaced by quorum queues (Raft). Migrate to quorum queues for HA.

**ActiveMQ Classic** - Replaced by ActiveMQ Artemis. Migrate to Artemis or RabbitMQ.

---

## Platform

### Adopt

**Kubernetes 1.35–1.37** - Container orchestration; 1.37 is current (Aug 2026), supported window is roughly N-2 (14 months, ~15-week release cadence). Use managed Kubernetes (EKS, GKE, AKS) unless you have a dedicated platform team.

**Gateway API v1.6.x** - Next-gen ingress; GatewayClass, Gateway, HTTPRoute are v1/GA. Richer routing, traffic splitting, multi-tenant delegation. Forward path for L7 routing; Ingress is legacy-but-everywhere. New clusters: Gateway API. Existing Ingress: migrate when touching routing config.

**Karpenter** - Just-in-time node provisioning for EKS; autoscales nodes based on pod requirements, bin-packing, spot instances. Faster, cheaper than Cluster Autoscaler. Use it on AWS.

**Istio (ambient mode)** - Sidecarless service mesh; GA since Istio 1.24 (Nov 2024), current ~1.26/1.27. Ambient uses ztunnel (L4) + waypoint proxies (L7). Simpler upgrades, lower CPU/memory than sidecars. Trial sidecar mode only for brownfield Istio estates; greenfield should start ambient. Multicluster ambient still maturing.

**Cilium (eBPF data plane)** - CNI + service mesh; eBPF-based (Linux 5.8+), sidecarless, network policy, observability. Strong for NetworkPolicy enforcement, eBPF-accelerated networking. Use as CNI; assess as a service mesh alternative to Istio.

**Crossplane** - Kubernetes-native infrastructure provisioning; GitOps-driven, manage cloud resources via CRDs. Use for platform teams building internal PaaS. Heavier than Terraform for one-off infra; wins for self-service developer platforms.

**AWS Lambda with SnapStart** - Serverless functions; SnapStart (Java 11+) takes a snapshot after initialization, restores it for subsequent cold starts (~10x faster). Use for event-driven workloads, low-frequency APIs. GraalVM native still beats SnapStart for absolute cold start time but loses JIT peak throughput.

### Trial

**Linkerd 2.20 (native sidecars)** - Lightweight service mesh; native sidecars (no init container) are default as of 2.20 (June 2026). Simpler than Istio, less feature-rich. Trial for smaller clusters where Istio is overkill.

**Google Cloud Run / Azure Container Apps** - Serverless containers; scale-to-zero, pay-per-request, auto-TLS. Trial for stateless HTTP services with variable load. Easier than Lambda for existing containerized apps.

**Backstage** - Developer portal; CNCF project (Spotify origin). Catalog, scaffolding, docs, TechDocs, plugin ecosystem. Trial for platform teams building golden paths. Requires investment to customize.

**ArgoCD Image Updater / Renovate for image tags** - Auto-update container image tags in GitOps repos. Trial to close the GitOps loop; otherwise manual tag bumps accumulate drift.

### Assess

**Knative Serving** - Serverless on Kubernetes; scale-to-zero, revision-based deploys. Assess if Cloud Run / Azure Container Apps don't fit. Operationally heavier than managed serverless.

**Virtual Kubelet** - Kubernetes API → serverless backends (AWS Fargate, Azure Container Instances). Assess for burst capacity or avoiding node management.

**Dapr** - Distributed application runtime; building blocks for state, pub/sub, bindings, secrets. Assess for polyglot microservices (non-Java). Overlap with Spring Cloud, less mature in JVM world.

### Hold

**Ingress (v1) as the forward path** - Not deprecated but legacy. Gateway API is the successor. New routing logic: use Gateway API. Existing Ingress: migrate when touching routing config, no urgency.

**Docker Swarm** - Deprecated. Migrate to Kubernetes.

**Sidecar-only service mesh assumptions** - Istio ambient, Cilium eBPF, and Linkerd native sidecars are the direction. Sidecar-per-pod is operational overhead (CPU, memory, upgrade blast radius). New meshes: ambient/eBPF. Existing sidecar meshes: plan migration when Istio ambient multicluster stabilizes or when you hit upgrade pain.

---

## Delivery

### Adopt

**Trunk-based development** - Short-lived feature branches (<1 day), frequent merges to main, feature flags for incomplete work. Pairs with high test coverage and fast CI. Reduces merge conflicts, enables continuous delivery.

**GitHub Actions** - CI/CD in GitHub; YAML workflows, matrix builds, reusable actions. Use it if you are on GitHub. GitLab CI for GitLab; Jenkins only for complex brownfield pipelines that are working.

**Argo CD** - GitOps for Kubernetes; declarative, Git as source of truth, automated sync. Use for multi-cluster, multi-tenant deployments. Pairs with Argo Rollouts for progressive delivery.

**Flux CD** - GitOps alternative to Argo; CNCF graduated, Git → Kubernetes reconciliation. Lighter than Argo CD for single-cluster use cases. Choose Argo for UI/multi-tenancy, Flux for simplicity.

**Argo Rollouts** - Progressive delivery for Kubernetes; canary, blue-green, A/B testing. Integrates with Istio, Gateway API, Nginx for traffic shaping. Use it for controlled rollouts with automated rollback.

**Terraform / OpenTofu** - Infrastructure as code; HCL DSL, state management, provider ecosystem. OpenTofu is the Linux Foundation fork (Aug 2023) after HashiCorp's license change to BSL. AWS CDK for AWS-only; Terraform/OpenTofu for multi-cloud. Use Terragrunt for DRY multi-environment config.

**Renovate** - Dependency update automation; creates PRs for version bumps, configurable schedules, auto-merge rules. Use it to avoid accumulating dependency debt. GitHub-native Dependabot is simpler but less flexible.

**OpenFeature** - Feature flag abstraction; vendor-agnostic SDK, supports LaunchDarkly, Split, Unleash, env vars. Use for decoupling feature flags from vendor APIs.

### Trial

**Flagger** - Progressive delivery operator; canary analysis with Prometheus, Istio/Gateway API integration, automated rollback. Trial as an alternative to Argo Rollouts; lighter, narrower scope.

**Pulumi** - Infrastructure as code in real programming languages (Java, TypeScript, Python, Go). Trial if your team hates HCL and wants IDE autocomplete / unit tests for infrastructure. Heavier learning curve than Terraform for ops teams.

**Dagger** - Programmable CI/CD pipelines as code (Go, Python, TypeScript); portable across CI vendors. Trial for complex pipelines with lots of custom logic. Overkill for simple workflows.

**Gradle build cache (remote)** - Shared build cache across CI agents; avoids recompiling unchanged modules. Trial for large multi-module Gradle builds where CI time is a bottleneck.

**Bazel** - Google's build tool; hermetic, incremental, polyglot. Trial for monorepos with 50+ modules and multiple languages. Steep learning curve; use Gradle unless you hit its scaling limits.

### Assess

**Gradle affected builds (built-in or via plugins)** - Only build changed modules and their dependents. Assess for large repos where full builds take >10 minutes. Bazel has stronger guarantees; Gradle plugins (e.g., `gradle-modules-plugin`) are lighter.

**Tekton Pipelines** - Kubernetes-native CI/CD; CNCF project. Assess if you want CI/CD as k8s resources. More complex than GitHub Actions for typical pipelines.

### Hold

**Jenkins for new installations** - Rich plugin ecosystem but groovy-based, stateful, UI-driven config, hard to reproduce. Use for brownfield pipelines that are working; do not start new Jenkins instances. Migrate to GitHub Actions, GitLab CI, or Argo Workflows.

**GitOps without secrets management** - Pushing secrets into Git (even encrypted) is fragile. Use External Secrets Operator (fetches from Vault, AWS Secrets Manager, etc.) or Sealed Secrets (encrypt with a cluster public key).

---

## Observability

### Adopt

**OpenTelemetry (traces, metrics, logs)** - Vendor-neutral observability; CNCF graduated. Java instrumentation ~2.30 (mid-2026), monthly releases. W3C `traceparent` is the default trace propagation. Use OTel SDK (manual) or OTel Java agent (auto-instrumentation). Export OTLP to any backend (Tempo, Jaeger, Prometheus, Loki, commercial vendors).

**Micrometer** - Metrics abstraction; Spring Boot's metrics API, exports to Prometheus, Datadog, New Relic, etc. Use `@Timed`, `Counter`, `Gauge`, `DistributionSummary`. Pairs with Micrometer Observation API for unified traces/metrics.

**Prometheus + native histograms** - Time-series metrics; pull model, PromQL, service discovery. Native histograms (GA in Prometheus 2.40+) replace classic histogram buckets with dynamic, high-resolution histograms. Use for latency percentiles without pre-bucketing.

**Grafana + Loki + Tempo** - Observability stack; Grafana for dashboards, Loki for logs (indexed by labels, not full-text), Tempo for traces. Use for cost-effective observability. Pairs with Prometheus. Grafana Alloy (formerly Grafana Agent) for collection.

**Continuous profiling** - Always-on CPU/memory profiling in production. Pyroscope (CNCF), Grafana Phlare (now merged into Pyroscope), Datadog Profiling. Use to diagnose latency spikes, memory leaks, GC pauses without reproducing locally. Attach profiling to traces for code-level hotspots.

**Tail sampling** - Sample traces after completion based on latency, errors, user ID. Keeps all interesting traces, drops boring ones. Use OpenTelemetry Collector tail sampling or vendor-native (Honeycomb, Lightstep). Head sampling (sample at ingestion) misses rare errors.

### Trial

**ClickHouse for observability** - Columnar OLAP; used by Grafana Cloud, Sentry, Cloudflare for logs/traces/metrics. Trial for self-hosted observability at scale; cheaper than Elasticsearch, faster for aggregations. Requires SQL comfort.

**Grafana Beyla (eBPF auto-instrumentation)** - Auto-instrument binaries without code changes or agents; eBPF-based HTTP/gRPC tracing. Trial for third-party apps you can't modify. Limited to protocol-level spans; no business-logic spans.

**OpenTelemetry profiling signal** - Still alpha/experimental as of late 2026; do not use in production. Assess for standardizing profiling data alongside traces/metrics once it stabilizes.

### Assess

**Vector (observability pipeline)** - Rust-based log/metric router; buffers, transforms, routes to multiple sinks. Assess for complex observability pipelines (e.g., multi-cloud, multi-backend). Simpler needs: Grafana Alloy or OTel Collector.

**Elastic (ELK stack)** - Elasticsearch, Logstash, Kibana. Assess for brownfield estates already using it. Greenfield: ClickHouse or Loki are cheaper and simpler for logs. Elastic's SSPL license (Mar 2021) complicates managed-service offerings.

### Hold

**Bespoke per-vendor instrumentation** - Using DatadogTracer, New Relic agent APIs, Dynatrace OneAgent SDK directly in code. Vendor lock-in. Replace with OpenTelemetry; vendors support OTLP ingestion.

**Spring Cloud Sleuth** - Covered under Spring ecosystem. Deprecated. Use Micrometer Tracing + OTel.

**Jaeger UI as the primary trace backend** - Jaeger is a UI + storage; Tempo is storage + Jaeger UI compatibility. Use Tempo for long-term trace retention; Jaeger for quick local dev.

---

## Security

### Adopt

**OAuth 2.1 (RFC 9068 JWT profile, PKCE mandatory)** - Consolidated OAuth 2.0 best practices; PKCE mandatory for all clients, bearer tokens, refresh token rotation. Use for API authorization. Avoid rolling your own OAuth server; use Spring Authorization Server or Keycloak.

**Spring Authorization Server** - OAuth 2.1 / OIDC authorization server; Spring's official replacement for deprecated Spring Security OAuth. Production-ready (1.0 Nov 2022). Use for first-party token issuance. Keycloak for federation/multi-tenant/SAML.

**SPIFFE / SPIRE** - Workload identity; X.509 SVIDs, automatic rotation, zero-trust. Use for service-to-service mTLS in Kubernetes (Istio uses SPIRE under the hood). SPIFFE spec; SPIRE is the reference implementation.

**Open Policy Agent (OPA) / Cedar** - Policy-as-code; declarative policies for authorization. OPA uses Rego (JSON-based), Cedar is AWS's policy language (DynamoDB, Verified Permissions). Use for complex RBAC/ABAC where Spring Security expressions are insufficient. OPA has wider adoption; Cedar is simpler for AWS-native.

**OpenFGA / Zanzibar-style ReBAC (Relationship-Based Access Control)** - Authorization graph; "user X can view document Y because user X is in group Z which has viewer role on Y". Use for Google Docs-style fine-grained sharing. OpenFGA is CNCF sandbox; Ory Keto, SpiceDB are alternatives.

**HashiCorp Vault / AWS Secrets Manager / Azure Key Vault / GCP Secret Manager** - Secret storage, rotation, dynamic secrets (database credentials, cloud IAM). Use Vault for on-prem or multi-cloud; cloud-native secret managers for single-cloud. Integrate with External Secrets Operator for Kubernetes.

**Sigstore / SLSA** - Software supply chain security. Sigstore: keyless signing (Cosign), transparency log (Rekor). SLSA: framework for supply chain levels (build provenance, signed artifacts). Use `cosign sign` for container images; SLSA level 3 for critical services.

**SBOM (Software Bill of Materials)** - List of dependencies; CycloneDX or SPDX format. Generate with `mvn cyclonedx:makeBom` or Syft. Required for compliance (EU Cyber Resilience Act, U.S. EO 14028). Scan SBOMs with Grype or Trivy for CVEs.

### Trial

**Keycloak** - Open-source identity and access management; OIDC/SAML, federation, multi-tenant, user management UI. Trial for brownfield where you need federation (LDAP, SAML IdP). Heavier than Spring Authorization Server; wins for multi-realm/multi-tenant.

**cert-manager** - Kubernetes certificate management; auto-renew TLS certs from Let's Encrypt, Vault, or private CA. Trial for automated TLS; pairs with Gateway API.

**Falco** - Runtime security for Kubernetes; eBPF-based, detects anomalous syscalls, network activity. Trial for runtime threat detection; requires tuning to avoid false positives.

### Assess

**Istio AuthorizationPolicy** - Service mesh-native RBAC; L4/L7 policies, deny-by-default. Assess if you already use Istio. Otherwise OPA or Cedar is more portable.

**Teleport** - Access proxy for SSH, Kubernetes, databases; zero-trust, session recording, MFA. Assess for platform teams managing multi-cluster/multi-cloud access. AWS SSM Session Manager is lighter for AWS-only.

### Hold

**OAuth 2.0 implicit flow** - Removed in OAuth 2.1; PKCE authorization code flow is the replacement. Migrate SPAs to PKCE.

**JSON Web Tokens (JWT) without expiration or rotation** - Long-lived JWTs are bearer tokens; if stolen, attacker has access until expiry. Use short-lived access tokens (15 min), refresh tokens with rotation.

**Storing secrets in Git (even encrypted)** - Sealed Secrets and SOPS encrypt secrets but key management is still hard. Use External Secrets Operator + Vault/cloud secret manager.

---

## Testing

### Adopt

**Testcontainers** - Spin up real Docker containers (Postgres, Kafka, Redis, etc.) in JUnit tests. Use for integration tests; no H2 dialect incompatibilities, no mocks for infrastructure. Pair with `@ServiceConnection` in Spring Boot 3.1+ for zero-config test containers.

**JUnit 5 / AssertJ / Mockito** - Standard Java test stack. JUnit 5 for parameterized tests, `@Nested`, lifecycle hooks. AssertJ for fluent assertions. Mockito for mocks (sparingly; prefer real collaborators or test doubles).

**Pact / Spring Cloud Contract** - Consumer-driven contract testing. Pact for polyglot (has JVM, Node, Go, etc. clients); Spring Cloud Contract for Spring-to-Spring. Use to verify API contracts between services without E2E tests.

**WireMock** - HTTP mock server; record/playback, stubbing, fault injection. Use for testing HTTP clients, third-party API integration. WireMock Cloud for managed mocks; self-hosted for free.

**k6** - Load testing; Grafana project, JavaScript DSL, runs in containers, exports metrics to Prometheus. Use for spike tests, soak tests, stress tests. Simpler than JMeter; integrates with Grafana dashboards.

### Trial

**Gatling** - Load testing; Scala DSL (Java DSL available), detailed HTML reports. Trial if your team prefers Gatling's DSL over k6's JavaScript. Both are excellent.

**jqwik (property-based testing)** - Generate random test inputs, find edge cases automatically. Trial for testing parsers, validators, algorithms. Heavier than example-based tests; use for genuinely complex invariants.

**PIT (mutation testing)** - Mutate code (flip `==` to `!=`, remove `if` branches), check if tests fail. Use to measure test quality. Slow; run on CI for critical modules, not full suite every commit.

**Playwright (Java)** - Browser automation; alternative to Selenium. Trial for UI testing if you must test browser flows. Prefer API tests over browser tests.

### Assess

**Testcontainers Cloud** - Remote Docker daemon for Testcontainers; faster on macOS, avoids Docker-in-Docker. Assess for teams hitting local Docker performance issues. Costs money; local Testcontainers is free.

**Karate** - API testing DSL; Cucumber-style given-when-then, built-in JSON/XML assertions. Assess for QA teams who prefer BDD syntax. Developers usually prefer JUnit + RestAssured.

### Hold

**H2 in-memory database as Postgres substitute** - Covered under Data. Use Testcontainers with real Postgres. H2 dialect incompatibilities cause false positives (test passes, production fails) and false negatives (test fails, production works).

**Giant E2E test suites as release gates** - Slow, flaky, high maintenance. Use contract tests (Pact) and smoke tests instead. E2E tests should be a thin layer; test most logic with unit/integration tests.

**JUnit 4** - Deprecated. Migrate to JUnit 5 (`junit-jupiter`). JUnit Vintage runs JUnit 4 tests in JUnit 5 runner; use for gradual migration.

---

## The dead list

Technologies you will meet in legacy code. Know them to migrate away.

| What | Why it died | Replacement | Migration order |
|------|-------------|-------------|-----------------|
| **Spring Cloud Sleuth** | Deprecated Spring Cloud 2022.0, removed for Boot 3+, OTel bridge archived Dec 2025. | Micrometer Observation + Micrometer Tracing (`io.micrometer:micrometer-tracing-bridge-otel`). | Remove `spring-cloud-sleuth-zipkin`, add Micrometer Tracing, configure OTLP exporter. W3C `traceparent` is default; B3 is legacy. |
| **Hystrix** | Netflix ended active development 2018; no cloud-native support. | Resilience4j. | Replace `@HystrixCommand` → `@CircuitBreaker`, rewrite Archaius config to `application.yaml`, remove `hystrix-core` dependency. |
| **Ribbon** | Deprecated, removed Spring Cloud 2022.0+. Blocking I/O, Eureka-coupled. | Spring Cloud LoadBalancer. | Upgrade blocks on Ribbon removal. Replace `@LoadBalanced RestTemplate` config, swap Eureka `DiscoveryClient` integration, test client-side retries. |
| **Zuul 1** | Servlet-based, blocking. Netflix never upstreamed Zuul 2 to Spring Cloud. | Spring Cloud Gateway (reactive) or MVC-based Gateway variant. | Rewrite filters (Zuul filters → Gateway filters), change config schema, test routing/rate limiting. Zuul 2 exists but Spring never adopted it. |
| **Eureka for greenfield k8s** | Not dead but obsolete on Kubernetes. Adds operational burden (HA, persistence) for zero benefit over k8s DNS. | Kubernetes Services (DNS). | Remove Eureka client/server dependencies, change service URLs to `http://service-name.namespace.svc.cluster.local`, remove `@EnableEurekaClient`. Eureka still valid for brownfield AWS/on-prem. |
| **`javax.*` namespace** | Jakarta EE 9+ moved to `jakarta.*` in 2020. `javax.*` is Java EE 8 (EOL). | `jakarta.*` (Servlet, JPA, Validation, etc.). | Upgrade Spring Boot to 3.x+, change imports `javax.servlet` → `jakarta.servlet`. Libraries using `javax.*` must upgrade or use `javax→jakarta` transformer. |
| **ZooKeeper for Kafka** | Removed Kafka 4.0 (Feb 2026). Metadata moved to `__cluster_metadata` topic (KRaft). | Kafka KRaft mode. | Run `kafka-storage format --cluster-id UUID --config server.properties`, rolling restart, metadata migrates. ZooKeeper-based Kafka runs on 3.x; upgrade to 4.x. |
| **Ingress (legacy, not dead)** | Gateway API supersedes it. Ingress is limited (no traffic splitting, delegation). | Gateway API (GatewayClass, Gateway, HTTPRoute). | Migrate routing rules to HTTPRoute, replace Ingress with Gateway, test traffic shaping. No urgency; Ingress still supported. |
| **Sidecar-only service mesh** | Operational overhead: CPU, memory, upgrade blast radius. Ambient/eBPF is cheaper. | Istio ambient mode, Cilium eBPF, Linkerd native sidecars. | Istio: enable ambient on namespace, remove sidecar injection labels, test ztunnel + waypoint. Multicluster ambient maturing; wait if needed. |
| **Docker Swarm** | Kubernetes won. Docker deprecated Swarm mode for new features. | Kubernetes. | Rebuild apps as Helm charts, migrate volumes/secrets, deploy to k8s. No in-place migration path. |
| **ActiveMQ Classic** | Replaced by ActiveMQ Artemis (JMS 2.0, faster). | ActiveMQ Artemis or RabbitMQ. | Artemis is wire-compatible (AMQP, MQTT, STOMP); change broker URL, test message ordering/persistence. |
| **RabbitMQ classic mirrored queues** | Replaced by quorum queues (Raft-based replication) in 3.8+. | Quorum queues. | Declare new quorum queues, migrate producers/consumers, delete classic queues, test failover. |
| **Spring Cloud Config without encryption** | Storing plain secrets in Git is a CVE. | External Secrets Operator (Vault, AWS Secrets Manager) or Sealed Secrets. | Migrate secrets to Vault, configure External Secrets Operator, reference k8s Secrets in config, delete secrets from Git history. |
| **Jenkins for new installs** | Stateful, UI-driven, groovy-based. Hard to reproduce pipelines. | GitHub Actions, GitLab CI, Argo Workflows. | Rewrite Jenkinsfiles as YAML workflows, migrate secrets to GitHub Secrets/Vault, test CI/CD. Keep existing Jenkins if it works. |

---

## The watch list for 2027

Technologies to assess in 3–6 months; one concrete signal moves them to Trial.

| Technology | Current status | Signal to move to Trial |
|------------|----------------|------------------------|
| **Kafka share groups (KIP-932)** | Alpha/early in Kafka 4.x; queue-like round-robin semantics, no partition assignment. | GA release with production use cases documented; Spring Kafka support. Simplifies consumer scaling for queue-style workloads. |
| **OpenTelemetry profiling signal** | Alpha/experimental as of late 2026; not production-ready. | GA specification, vendor support (Grafana, Datadog, etc.), OTel Java SDK stable API. Standardizes continuous profiling alongside traces/metrics. |
| **Istio ambient multicluster** | Single-cluster ambient is GA (1.24+); multicluster ambient maturing. | Documented production deployments across 3+ clusters, trust-bundle federation stable. Removes last blocker for ambient adoption in multi-region estates. |
| **Virtual threads in reactive libraries (WebFlux, R2DBC)** | Spring WebFlux is still reactive (Reactor); virtual threads are for blocking code. Project Loom adaptors experimental. | Spring Framework integrates virtual threads with reactive stacks, or mainstream adoption of structured concurrency over reactive. Simplifies programming model. |
| **GraalVM native image startup <10 ms** | Current ~50–100 ms with PGO; faster than JVM but not instant. | Sub-10 ms cold start in production use cases; closes gap with Go/Rust for serverless. AWS Lambda SnapStart competitor. |
| **CRaC (Checkpoint/Restore)** | JEP 439 (Java 21+), Spring Boot experimental support. Snapshots carry security/state caveats. | First-class Spring Boot support, documented security model for credential rotation, production success stories. Sub-100 ms serverless without GraalVM tradeoffs. |
| **WebAssembly for microservice plugins** | Strong at edge (Envoy filters, Cloudflare Workers); not mainstream for JVM microservices. | Java-to-Wasm toolchain production-ready, Spring Boot plugin model using Wasm, performance parity with native JVM. Sandboxed extensions without separate processes. |
| **Cedar policy language (AWS Verified Permissions)** | GA for AWS services; narrower adoption than OPA. | Multi-cloud support, Spring Security integration, migration tooling from OPA Rego. Simpler policy syntax than Rego. |

---

## How to keep this radar current yourself

Technology moves; this snapshot expires in 6–9 months. A quarterly 90-minute review ritual keeps you ahead.

**Quarterly review (every 3 months):**

1. **Check release notes (30 minutes):** Spring Boot, Spring Cloud, Kubernetes, Kafka, Istio, OpenTelemetry, PostgreSQL. Scan for GA releases, deprecations, security advisories. Update version numbers in your radar.
2. **Survey your production estate (20 minutes):** What libraries/platforms are you running? Compare to radar. Identify drift (Hold items still in use, Adopt items not yet adopted). Create upgrade tickets.
3. **Read engineering blogs (20 minutes):** Netflix, Uber, Spotify, Amazon, Meta, Google, LinkedIn, Stripe, Airbnb. Filter for "how we built/migrated/scaled X". Look for patterns (e.g., multiple companies moving from distributed to modular monolith → update radar trend).
4. **Scan CNCF landscape updates (10 minutes):** New graduated/incubating projects, archived projects. CNCF graduation = maturity signal; archival = death signal.
5. **Reddit/Hacker News search (10 minutes):** Search "microservices", "Kafka", "Kubernetes", "Spring Boot", filter by top posts last 3 months. Identify sentiment shifts (e.g., backlash, new best practices).

**Sources to follow:**

- **Release notes:** Spring Boot, Spring Cloud, Kubernetes, Istio, Kafka, OpenTelemetry, PostgreSQL (official changelogs).
- **Engineering blogs:** Netflix Tech Blog, Uber Engineering, Spotify Engineering, AWS Architecture Blog, Google Cloud Blog, LinkedIn Engineering, Stripe Engineering, Airbnb Engineering.
- **Aggregators:** CNCF blog, InfoQ (Java/microservices tags), Thoughtworks Technology Radar (bi-annual).
- **Podcasts (monthly):** Software Engineering Daily, The Changelog, Kubernetes Podcast.
- **Books (annual):** Re-read Architecture Patterns with Python, Building Microservices (O'Reilly), Designing Data-Intensive Applications. New editions signal industry shifts.

**Annual deep review (December, 2 hours):**

Compare this radar to Thoughtworks Technology Radar, CNCF Annual Report, Stack Overflow Developer Survey, JetBrains State of Developer Ecosystem. Identify major divergences. Update radar, justify changes.

**Your radar is live.** Treat it as a living document in your team wiki. Version it in Git. Link adoption decisions to radar entries in ADRs (Architecture Decision Records). Radar drift from production reality = technical debt inventory.
