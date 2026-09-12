# Phase 3 - Communication, APIs and Contracts

> **Weeks:** 18-23 | **Prerequisites:** Phase 2 (Spring Boot production core) | **Time budget:** 40-50 hrs
> **You finish this phase able to:**
> - Design REST APIs that evolve without breaking consumers
> - Choose sync vs async communication with a defensible rationale
> - Implement idempotent mutations that survive retries and network replays
> - Write gRPC services with wire-compatible protobuf evolution
> - Detect and fix N+1 queries in GraphQL resolvers
> - Build contract-first workflows with OpenAPI and breaking-change detection in CI

## Why this phase exists

The API is the only part of your service other teams can see. It is a public promise that outlives your implementation, your framework version, and usually your tenure. A bad schema design locks in technical debt; a version-breaking change forces a coordinated deployment across teams; an API that does not signal retryability causes retry storms. Inter-service communication failures — timeouts, retries, backpressure, deserialization breaks — are the top cause of production incidents in distributed systems. This phase teaches you to design APIs that do not break, calls that do not amplify failures, and contracts that make breaking changes detectable before deploy.

## Mental model

Every inter-service interaction is a choice on three axes:

1. **Coupling**: who must know about whom? Request-reply couples caller to callee's uptime and response time. Events decouple: producer does not know consumers exist.
2. **Timing**: must both services be up simultaneously? Synchronous calls require it; asynchronous messaging does not.
3. **Consistency**: when does the caller learn the truth? Sync returns the answer immediately; async returns "accepted" and the truth arrives later, possibly never.

Pick the axis you need, then the protocol — never the reverse. Do not pick gRPC because it is "modern" or Kafka because it is "scalable." Pick based on the access pattern: if the user is waiting for an answer, you need synchronous request-reply. If you are propagating state changes for eventual processing, you need asynchronous messaging. The default is simpler than most teams assume: **synchronous for queries, asynchronous for state propagation**.

## Core concepts

### Synchronous vs asynchronous decision framework

| Criterion | Sync (REST/gRPC) | Async (messaging/events) |
|-----------|------------------|--------------------------|
| **User blocking on this call?** | Yes → sync | No → async |
| **Need an answer *now*?** | Yes → sync | No → async |
| **Caller can tolerate stale data?** | No → sync | Yes → async |
| **Failure isolation required?** | No; caller fails if callee down | Yes; producer continues if consumer down |
| **Fan-out to N consumers?** | Inefficient; N calls | Natural; pub-sub |
| **Ordering guarantees needed?** | Manual sequencing required | Native in Kafka partitions / message groups |
| **Request idempotency?** | Must implement explicitly | Often built-in (at-least-once + idempotent consumer) |
| **Operational complexity** | Lower; HTTP is debuggable | Higher; need broker, dead-letter queues, lag monitoring |
| **Latency** | Single round-trip (ms) | Eventual (seconds to minutes) |
| **Backpressure** | Natural (slow consumer → slow response) | Must design (consumer lag → alert) |

**Default recommendation**: Use synchronous calls for read-your-writes scenarios (user submits order → needs order confirmation), command-query paths where the user is waiting (search, cart totals, real-time inventory check). Use asynchronous messaging for state broadcasts (order created → notify shipping, update analytics, send email), cross-team integrations where you cannot afford downtime coupling, and any scenario where "eventual consistency in 5 seconds" is acceptable.

**Anti-pattern**: Using async messaging for user-blocking operations because "it scales better." The user still waits; you have just added broker latency and failure modes. Netflix's API gateway calls dozens of backend services synchronously in parallel with strict timeouts — this is correct for their use case.

### REST done properly

REST is not "JSON over HTTP." It is resource-oriented design with HTTP semantics used correctly.

#### Resource modeling

Model nouns, not verbs. `POST /orders` not `POST /createOrder`. Collections (`/orders`) vs items (`/orders/{id}`) vs sub-resources (`/orders/{id}/items`). Nested resources imply ownership: `/users/{userId}/addresses` means addresses belong to a user. Flat resources avoid deep hierarchies: `/addresses?userId={id}` is often simpler.

**ShopKart example**:
- `GET /catalog/products` — list products
- `GET /catalog/products/{sku}` — product detail
- `POST /cart/items` — add to cart (item contains `productSku`, `quantity`)
- `GET /cart` — current cart state
- `POST /orders` — create order
- `GET /orders/{orderId}` — order detail
- `POST /orders/{orderId}/cancel` — sub-resource action (exception to verb rule when the action is not CRUD)

#### HTTP method semantics

| Method | Safe? | Idempotent? | Use case |
|--------|-------|-------------|----------|
| **GET** | Yes | Yes | Retrieve resource; no side effects; cacheable |
| **HEAD** | Yes | Yes | Metadata only (headers, no body) |
| **OPTIONS** | Yes | Yes | Discover allowed methods (CORS preflight) |
| **POST** | No | **No** | Create resource (server assigns ID), non-idempotent operations |
| **PUT** | No | Yes | Replace entire resource (client provides ID), upsert |
| **PATCH** | No | Yes (if designed carefully) | Partial update; idempotent if using absolute values, not deltas |
| **DELETE** | No | Yes | Remove resource; second DELETE → 404, still idempotent |

**Safe** means no side effects visible to the client; you can call it repeatedly without changing server state. **Idempotent** means calling N times has the same effect as calling once. POST is the only common non-idempotent method; this is why retry logic treats POST specially.

**Critical nuance**: PUT is idempotent if you send the full resource representation every time. `PUT /cart {"items": [...]}` replacing the entire cart is idempotent. `PUT /cart {"add": {"sku": "X"}}` is not — it is a delta, and calling it twice adds the item twice. Use PATCH for deltas, with explicit replace/add/remove semantics (JSON Patch RFC 6902 or JSON Merge Patch RFC 7386).

#### Status code discipline

Use the right code; do not return `200 OK` with `{"error": "..."}` in the body.

| Code | Meaning | When to use |
|------|---------|-------------|
| **200 OK** | Success with body | GET, PUT, PATCH success |
| **201 Created** | Resource created | POST success; include `Location` header with new resource URI |
| **202 Accepted** | Async operation started | Request valid, processing in background; return status URL |
| **204 No Content** | Success, no body to return | DELETE success, PUT with no response payload |
| **304 Not Modified** | Conditional GET; resource unchanged | ETag matches; saves bandwidth |
| **400 Bad Request** | Malformed request | Invalid JSON, missing required field, type mismatch |
| **401 Unauthorized** | Authentication required | Missing/invalid token |
| **403 Forbidden** | Authenticated but insufficient permissions | User lacks role for this operation |
| **404 Not Found** | Resource does not exist | Unknown ID |
| **409 Conflict** | Cannot complete due to current state | Delete order that already shipped, double-booking |
| **412 Precondition Failed** | `If-Match` ETag mismatch | Optimistic concurrency failure |
| **422 Unprocessable Entity** | Valid syntax, invalid semantics | Quantity negative, date in past, business rule violation |
| **428 Precondition Required** | Server requires `If-Match` but none sent | Force clients to use optimistic concurrency |
| **429 Too Many Requests** | Rate limit exceeded | Include `Retry-After` header (seconds or HTTP date) |
| **500 Internal Server Error** | Unhandled server failure | Bug, uncaught exception |
| **503 Service Unavailable** | Temporary overload or maintenance | Include `Retry-After`; circuit breaker open |

**Security leak**: `403 Forbidden` vs `404 Not Found`. If `/orders/999` returns 404 when the order does not exist but 403 when it exists and belongs to another user, you have leaked the existence of order 999. Consistent rule: return 404 for both cases if the resource is access-controlled. Stripe does this: inaccessible charge IDs return 404, not 403.

#### Conditional requests and optimistic concurrency

**ETag** (entity tag) is a hash or version of the resource representation. The server returns it in the `ETag` header; the client sends it back in `If-Match` (for writes) or `If-None-Match` (for reads).

```http
GET /cart/123
HTTP/1.1 200 OK
ETag: "v42"
Content-Type: application/json

{"items": [...], "total": 99.99}
```

Client updates cart:

```http
PUT /cart/123
If-Match: "v42"
Content-Type: application/json

{"items": [...], "total": 109.99}
```

If another client modified the cart in the meantime (ETag now `"v43"`):

```http
HTTP/1.1 412 Precondition Failed
ETag: "v43"
```

The client refetches, merges changes, and retries. This is **optimistic concurrency over HTTP** — no locks, no distributed transactions, just compare-and-swap semantics.

**Implementation**: ETag can be a database version column, a content hash (MD5 of JSON), or a timestamp. Version column is cheapest. Return `428 Precondition Required` if the resource requires `If-Match` but the client omitted it.

**Conditional GET**: Client sends `If-None-Match: "v42"` with a GET. If the resource has not changed, server returns `304 Not Modified` with no body, saving bandwidth.

#### HATEOAS and hypermedia

HATEOAS (Hypermedia as the Engine of Application State) means embedding links in responses so clients can discover actions without hardcoding URIs.

```json
{
  "orderId": "12345",
  "status": "pending",
  "total": 99.99,
  "_links": {
    "self": {"href": "/orders/12345"},
    "cancel": {"href": "/orders/12345/cancel", "method": "POST"},
    "payment": {"href": "/orders/12345/payment"}
  }
}
```

Once the order ships, the `cancel` link disappears; the client disables the cancel button. The client never hardcodes `/orders/{id}/cancel`.

**Verdict**: HATEOAS is elegant in theory; in practice it is heavyweight. Public APIs (GitHub, Stripe) use it successfully. Internal microservices usually do not — the coupling is already tight (same org, same deployment cadence), and hardcoding URIs is simpler. Use HATEOAS if you have clients you do not control (public API, mobile apps with long release cycles) and the API surface is large enough that discovery is a real problem. Skip it for internal service-to-service calls.

### Pagination, filtering, and sorting

#### Offset vs cursor-based pagination

**Offset pagination**: `GET /products?limit=20&offset=40` returns items 41–60.

```sql
SELECT * FROM products ORDER BY id LIMIT 20 OFFSET 40;
```

**Problem**: `OFFSET 100000` means the database scans and discards 100,000 rows before returning the next 20. At scale, this is O(n) in the offset value. Page 5000 of search results is slow and expensive.

**Additional problem**: inconsistent results during pagination. User requests page 1; 20 new products are inserted; user requests page 2 and sees duplicates or skips items because the offset no longer aligns with the dataset.

**Cursor (keyset) pagination**: `GET /products?limit=20&after=prod_xyz789` returns the next 20 items after the cursor.

```sql
SELECT * FROM products WHERE id > 'prod_xyz789' ORDER BY id LIMIT 20;
```

The cursor encodes the last-seen value (often base64-encoded JSON: `{"id": "prod_xyz789", "createdAt": "2026-09-12T10:30:00Z"}`). The query uses a WHERE clause, not OFFSET — this is O(log n) with an index.

**Cursor encoding example**:

```java
record ProductCursor(String id, Instant createdAt) {
    String encode() {
        String json = """
            {"id":"%s","createdAt":"%s"}
            """.formatted(id, createdAt);
        return Base64.getUrlEncoder().encodeToString(json.getBytes(UTF_8));
    }
    
    static ProductCursor decode(String cursor) {
        String json = new String(Base64.getUrlDecoder().decode(cursor), UTF_8);
        // parse JSON; return record
    }
}
```

**Requirement**: The sort key must be unique (or unique + tie-breaker). If sorting by `createdAt`, include `id` as secondary sort: `ORDER BY createdAt DESC, id ASC`. Otherwise, items with identical timestamps can appear on multiple pages.

**Trade-off**: Cursor pagination prevents random access (no "jump to page 50"). Use offset pagination for small datasets or admin UIs where users need page numbers. Use cursor pagination for infinite scroll, APIs, and any dataset over ~10k items.

#### Filtering and sorting

Support common query patterns: `GET /products?category=electronics&minPrice=100&maxPrice=500&sort=-price,name` (sort descending by price, ascending by name).

**Security**: User-controlled `sort` parameters can trigger expensive queries or expose internal schema. Whitelist allowed sort fields. Never pass user input directly into `ORDER BY`.

```java
private static final Set<String> ALLOWED_SORT_FIELDS = Set.of("price", "name", "createdAt");

String sanitizeSortField(String field) {
    String normalized = field.startsWith("-") ? field.substring(1) : field;
    if (!ALLOWED_SORT_FIELDS.contains(normalized)) {
        throw new BadRequestException("Invalid sort field: " + field);
    }
    return field;
}
```

#### Sparse fieldsets

Let clients request only needed fields: `GET /orders/12345?fields=id,status,total` returns a smaller payload. Useful for mobile clients on slow networks or when fetching thousands of records.

GraphQL does this natively; in REST you build it manually. Spring Data JPA projections or custom DTOs handle this at the persistence layer.

### API versioning

The real rule is **never break the contract**. Versioning is what you do when you must break it anyway.

#### Expand-and-contract (Parallel Change)

The safest versioning strategy is no versioning: evolve the schema additively.

1. **Expand**: Add new field `customerEmail`; keep old field `email` populated with the same value.
2. **Migrate consumers**: Update clients to use `customerEmail`.
3. **Contract**: Remove `email` once all consumers migrated.

This works for adding fields, adding optional parameters, widening types (string → string or null). It does not work for removing required fields, changing semantics, or narrowing types.

**Additive-only evolution rules**:
- New fields must be optional (nullable, default value).
- Never remove or rename a field; deprecate and add a new one.
- Never change field types in incompatible ways (string → int).
- Never repurpose a field (changing its meaning).
- Enum values: can add, cannot remove.

If every change is additive, you never need versioning.

#### When you must version

You need versioning when you cannot avoid breaking changes: removing a field consumers depend on, changing response structure, altering semantics (price now includes tax when it previously did not).

| Strategy | Format | Pros | Cons |
|----------|--------|------|------|
| **URI versioning** | `/v1/orders`, `/v2/orders` | Simple, visible, cacheable | Couples version to resource; URL changes |
| **Header versioning** | `Accept: application/vnd.shopkart.v2+json` | Clean URIs | Harder to test (cannot paste URL into browser), cache key complexity |
| **Query param** | `/orders?version=2` | Simple | Not RESTful (resource identity changes with version) |
| **Media type** | `Content-Type: application/vnd.order.v2+json` | Proper content negotiation | Heavyweight, rarely used |

**Recommendation**: URI versioning (`/v1/`, `/v2/`) for public APIs — it is the easiest for external developers to understand and test. Header versioning for internal APIs if you want stable URIs. Never use both.

#### Date-based versioning (Stripe model)

Stripe versions its API by date: `Stripe-Version: 2024-11-20`. The client specifies the API version it was built against; Stripe internally translates between versions.

```http
POST /v1/charges
Stripe-Version: 2024-11-20
```

This decouples deployment from API changes. Stripe can ship a new field on 2026-01-15; clients on `2024-11-20` do not see it. Stripe maintains adapters that convert between representations.

**When to use**: Public SaaS APIs with many external clients on different upgrade schedules. Requires significant engineering investment (version adapters, compatibility tests). Overkill for internal microservices.

#### Deprecation and retirement

Versioning is not complete until you retire the old version.

1. **Announce deprecation**: Add `Sunset` header (RFC 8594) with retirement date: `Sunset: Sat, 01 Mar 2025 00:00:00 GMT`. Add `Deprecation` header: `Deprecation: true` or `Deprecation: Sat, 01 Sep 2024 00:00:00 GMT`.
2. **Track usage**: Log `User-Agent` or API key; identify active consumers of the deprecated version.
3. **Contact consumers**: Email, Slack, dashboard notification. Give 6–12 months for public APIs, 3–6 months internal.
4. **Provide migration guide**: What changed, how to upgrade, example diff.
5. **Warn in responses**: Return `Warning` header (RFC 7234): `Warning: 299 - "API version v1 is deprecated; migrate to v2 by 2025-03-01"`.
6. **Retire**: Return `410 Gone` for all requests to the old version. Keep the endpoint alive (do not 404) so consumers get a clear error.

**Consumer inventory**: Maintain a list of all consumers (services, teams, external clients) for each version. Without this, you cannot retire anything.

### Errors as a contract

HTTP status codes tell the client *what* went wrong. The response body tells them *why* and *what to do*.

#### RFC 9457 Problem Details for HTTP APIs

Standard error format: `Content-Type: application/problem+json`.

```json
{
  "type": "https://docs.shopkart.com/errors/insufficient-inventory",
  "title": "Insufficient Inventory",
  "status": 409,
  "detail": "Product SKU-12345 has only 2 units available; requested 5.",
  "instance": "/orders/67890",
  "sku": "SKU-12345",
  "available": 2,
  "requested": 5
}
```

- **`type`**: URI identifying the error type (documentation link).
- **`title`**: Human-readable summary.
- **`status`**: HTTP status code (redundant with HTTP response, but included for consistency).
- **`detail`**: Human-readable explanation specific to this occurrence.
- **`instance`**: URI of the specific request (e.g., order ID).
- **Extension fields**: `sku`, `available`, `requested` — machine-readable context.

**Machine-readable error codes**: Clients need to know if they should retry. Include a `retryable` boolean or a `code` field:

```json
{
  "type": "https://docs.shopkart.com/errors/timeout",
  "status": 504,
  "code": "UPSTREAM_TIMEOUT",
  "retryable": true,
  "detail": "Inventory service did not respond within 2s."
}
```

Standard codes: `VALIDATION_ERROR` (400, not retryable), `UNAUTHORIZED` (401, not retryable), `RATE_LIMITED` (429, retryable after delay), `CONFLICT` (409, not retryable), `UPSTREAM_TIMEOUT` (504, retryable), `INTERNAL_ERROR` (500, retryable with backoff).

**Partial failure responses**: For batch operations, include per-item results:

```json
{
  "status": 207,
  "results": [
    {"index": 0, "status": 201, "id": "item-001"},
    {"index": 1, "status": 409, "error": {"code": "DUPLICATE", "detail": "Item already exists"}},
    {"index": 2, "status": 201, "id": "item-003"}
  ]
}
```

HTTP `207 Multi-Status` is underused but perfect here.

**Anti-pattern**: `200 OK` with `{"success": false, "error": "..."}`. The HTTP status code is the contract. Monitoring tools, proxies, and client libraries all check status codes; hiding errors in a 200 response breaks observability.

### Idempotency for mutating APIs

**Plain English:** Idempotency means you can safely retry the same request multiple times and it will have the same effect as calling it once.

**Analogy:** Light switches are idempotent. Flipping the switch to "on" multiple times leaves the light on; flipping to "off" multiple times leaves it off. The final state depends only on the last command, not how many times you repeated it. A button that increments a counter is *not* idempotent — pressing it ten times increments by ten.

**In the real world:** When you submit a payment on Stripe or Amazon and see a spinner, you do not know if the request reached the server. You might hit "pay" again. The system must detect the duplicate and return the original result, not charge you twice. The client sends an `Idempotency-Key` header; the server stores the result keyed by that ID and replays it on retries.

**Mechanics:** The client generates a unique key (UUID) and sends it with every request:

```http
POST /orders
Idempotency-Key: a1b2c3d4-5678-90ab-cdef-1234567890ab
Content-Type: application/json

{"items": [...], "total": 99.99}
```

Server-side pseudocode:

1. Check if an idempotency record exists for this key.
2. If it exists and status is `complete`, return the stored response (replay).
3. If it exists and status is `in_progress`, return `409 Conflict` (concurrent duplicate).
4. If it does not exist, insert a record with status `in_progress` and a fingerprint of the request body.
5. Execute the business logic (create order).
6. Update the record to `complete`, store the response payload, set TTL (e.g., 24 hours).
7. Return the response.

**DDL for idempotency table**:

```sql
CREATE TABLE idempotency_records (
    idempotency_key VARCHAR(255) PRIMARY KEY,
    request_fingerprint VARCHAR(64) NOT NULL,  -- hash of request body
    status VARCHAR(20) NOT NULL,               -- in_progress | complete
    response_status INT,
    response_body TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP NOT NULL
);

CREATE INDEX idx_expires_at ON idempotency_records(expires_at);
```

**Concurrent duplicate handling**: The unique constraint on `idempotency_key` prevents two concurrent requests from both succeeding. One insert wins; the other fails with a constraint violation. The loser retries the read and finds status `in_progress`, returns `409 Conflict` with `Retry-After: 1`.

**Request fingerprint**: Hashing the request body ensures replay only if the request is identical. If the client sends the same `Idempotency-Key` but different payload (bug), the server detects it and returns `422 Unprocessable Entity: Idempotency key reused with different request`.

**TTL**: Idempotency records expire after 24 hours (or 7 days for payment systems). After expiry, the same key can be reused. Cleanup: background job deletes rows where `expires_at < NOW()`.

**What breaks:** Client retries with a different `Idempotency-Key` → you execute the operation twice (charge customer twice, create duplicate order). Client omits `Idempotency-Key` → every retry is a new operation. Server crashes between executing business logic and storing the response → on retry, the business logic runs again. Fix: make the business logic itself idempotent (upsert with unique constraint, or store the idempotency record in the same transaction as the business state).

**Java implementation sketch**:

```java
@RestController
class OrderController {
    private final IdempotencyService idempotency;
    private final OrderService orderService;
    
    @PostMapping("/orders")
    ResponseEntity<OrderResponse> createOrder(
        @RequestHeader("Idempotency-Key") String idempotencyKey,
        @RequestBody OrderRequest request
    ) {
        String fingerprint = computeFingerprint(request);
        
        IdempotencyRecord record = idempotency.findOrCreate(idempotencyKey, fingerprint);
        
        return switch (record.status()) {
            case COMPLETE -> ResponseEntity
                .status(record.responseStatus())
                .body(record.responseBody());
            case IN_PROGRESS -> ResponseEntity
                .status(409)
                .header("Retry-After", "1")
                .build();
            case NEW -> {
                OrderResponse response = orderService.createOrder(request);
                idempotency.recordSuccess(idempotencyKey, 201, response);
                yield ResponseEntity.status(201).body(response);
            }
        };
    }
    
    private String computeFingerprint(OrderRequest request) {
        return DigestUtils.sha256Hex(objectMapper.writeValueAsString(request));
    }
}
```

**When to require idempotency keys**: For any non-idempotent operation (POST that creates a resource, payment, state-changing action). GET/PUT/DELETE are idempotent by design (if implemented correctly). If you want to enforce it, return `428 Precondition Required` when `Idempotency-Key` is missing.

### Contract-first workflow

**Contract-first** means the OpenAPI spec is the source of truth; code is generated from it. The alternative is **code-first** (annotate controllers, generate spec from code). Contract-first wins for teams larger than one person because it forces API design to be a deliberate, reviewable step before implementation.

#### OpenAPI spec as source of truth

```yaml
openapi: 3.1.0
info:
  title: ShopKart Order API
  version: 2.0.0

paths:
  /orders:
    post:
      summary: Create order
      operationId: createOrder
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/OrderRequest'
      responses:
        '201':
          description: Order created
          headers:
            Location:
              schema:
                type: string
                format: uri
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/OrderResponse'
        '409':
          description: Conflict
          content:
            application/problem+json:
              schema:
                $ref: '#/components/schemas/ProblemDetail'

components:
  schemas:
    OrderRequest:
      type: object
      required: [items]
      properties:
        items:
          type: array
          items:
            $ref: '#/components/schemas/OrderItem'
    OrderItem:
      type: object
      required: [sku, quantity]
      properties:
        sku:
          type: string
        quantity:
          type: integer
          minimum: 1
```

Store this in `src/main/resources/openapi/order-api.yaml`. Generate server stubs and client code in the build.

**Maven plugin** (Spring Boot 4.x + OpenAPI Generator):

```xml
<plugin>
    <groupId>org.openapitools</groupId>
    <artifactId>openapi-generator-maven-plugin</artifactId>
    <version>7.12.0</version>
    <executions>
        <execution>
            <goals><goal>generate</goal></goals>
            <configuration>
                <inputSpec>${project.basedir}/src/main/resources/openapi/order-api.yaml</inputSpec>
                <generatorName>spring</generatorName>
                <apiPackage>com.shopkart.order.api</apiPackage>
                <modelPackage>com.shopkart.order.model</modelPackage>
                <configOptions>
                    <useJakartaEe>true</useJakartaEe>
                    <interfaceOnly>true</interfaceOnly>
                    <skipDefaultInterface>true</skipDefaultInterface>
                    <useTags>true</useTags>
                </configOptions>
            </configuration>
        </execution>
    </executions>
</plugin>
```

This generates `OrderApi` interface; your controller implements it.

#### Spec linting and style guides

**Spectral** is a linter for OpenAPI specs. Define rules in `.spectral.yaml`:

```yaml
extends: [[spectral:oas, all]]
rules:
  operation-operationId: error
  operation-summary: error
  operation-tags: error
  paths-kebab-case: error
  oas3-valid-media-example: error
  no-$ref-siblings: error
  # Custom: all paths must be versioned
  path-versioning:
    message: Paths must start with /v{number}/
    given: $.paths.*~
    then:
      function: pattern
      functionOptions:
        match: ^\/v\d+\/
```

Run in CI: `spectral lint src/main/resources/openapi/*.yaml`. Fail the build on errors.

**Style guides**:
- **Zalando RESTful API Guidelines**: opinionated, comprehensive, public.
- **Google API Improvement Proposals (AIP)**: resource-oriented design, field masks, long-running operations.
- **Microsoft REST API Guidelines**: similar to Zalando, less opinionated.

Pick one; enforce it with Spectral rules.

#### Breaking change detection

Use **openapi-diff** or **oasdiff** to compare spec versions:

```bash
oasdiff breaking openapi/v1.yaml openapi/v2.yaml
```

Output:

```
error: removed required property 'email' from schema OrderRequest
error: changed response status from 200 to 201 for POST /orders
```

Run this in CI on pull requests that modify the spec. Breaking changes require a version bump and approval.

#### Publishing to a developer portal

Internal developer portals (Backstage, Swagger UI, Redoc) consume OpenAPI specs. Publish your spec to a catalog:

```bash
curl -X POST https://catalog.internal/api/specs \
  -H "Content-Type: application/yaml" \
  --data-binary @openapi/order-api.yaml
```

Developers discover APIs, test them interactively, and generate client SDKs.

### gRPC

gRPC is HTTP/2-based RPC with Protobuf serialization. It wins for internal, high-QPS, strongly-typed service-to-service calls. It loses for public APIs (no browser support without grpc-web), debugging (binary payloads), and human-readable logs.

#### Protobuf schema

```protobuf
syntax = "proto3";

package shopkart.order.v1;

option java_package = "com.shopkart.order.v1";
option java_multiple_files = true;

service OrderService {
  rpc CreateOrder(CreateOrderRequest) returns (Order);
  rpc GetOrder(GetOrderRequest) returns (Order);
  rpc ListOrders(ListOrdersRequest) returns (stream Order);  // server streaming
  rpc TrackOrderUpdates(TrackOrderRequest) returns (stream OrderUpdate);  // server streaming
}

message CreateOrderRequest {
  repeated OrderItem items = 1;
  string idempotency_key = 2;
}

message Order {
  string order_id = 1;
  OrderStatus status = 2;
  repeated OrderItem items = 3;
  double total = 4;
}

enum OrderStatus {
  ORDER_STATUS_UNSPECIFIED = 0;  // required default
  ORDER_STATUS_PENDING = 1;
  ORDER_STATUS_CONFIRMED = 2;
  ORDER_STATUS_SHIPPED = 3;
}

message OrderItem {
  string sku = 1;
  int32 quantity = 2;
}
```

**Field numbers** are the wire-format identifier. Never reuse a field number. When you remove a field, mark it `reserved`:

```protobuf
message Order {
  reserved 5;  // old field 'discount' removed
  reserved "discount";
  string order_id = 1;
  // ...
}
```

#### Wire compatibility rules

- **Adding fields**: Always safe if the field is optional (proto3 default).
- **Removing fields**: Safe if you mark it `reserved`. Clients with old schemas ignore unknown fields.
- **Renaming fields**: Safe; only field numbers matter on the wire. Field names are for code generation.
- **Changing field numbers**: Breaks everything. Never do this.
- **Changing field types**: Mostly breaks. `int32 ↔ int64` works. `string → bytes` works. Most others do not.
- **Enum values**: Can add new values. Cannot remove or renumber. Default value (0) must exist.

#### The four RPC types

1. **Unary**: Single request, single response (like REST). `rpc CreateOrder(CreateOrderRequest) returns (Order);`
2. **Server streaming**: Single request, stream of responses. `rpc ListOrders(...) returns (stream Order);` — server sends multiple Order messages.
3. **Client streaming**: Stream of requests, single response. `rpc UploadItems(stream OrderItem) returns (UploadSummary);`
4. **Bidirectional streaming**: Stream in both directions. `rpc Chat(stream Message) returns (stream Message);`

#### Deadlines and cancellation

**Deadlines propagate** in gRPC. If the client sets a 5-second deadline and makes a nested call to another service, the deadline shrinks: service A → service B (4.9s left) → service C (4.8s left). When the deadline expires, all pending calls are canceled.

```java
ManagedChannel channel = ManagedChannelBuilder.forAddress("localhost", 8080)
    .usePlaintext()
    .build();

OrderServiceGrpc.OrderServiceBlockingStub stub = OrderServiceGrpc.newBlockingStub(channel)
    .withDeadlineAfter(5, TimeUnit.SECONDS);

try {
    Order order = stub.createOrder(request);
} catch (StatusRuntimeException e) {
    if (e.getStatus().getCode() == Status.Code.DEADLINE_EXCEEDED) {
        // timeout
    }
}
```

HTTP timeouts do not propagate by default; you must manually decrement the timeout and pass it in a header. gRPC does this automatically.

**Cancellation**: When a client cancels a request (user navigates away, timeout), the server receives a cancellation signal. The server should stop processing and clean up.

#### Status codes and error handling

gRPC status codes map roughly to HTTP but are more granular:

| gRPC Code | HTTP equiv | Meaning |
|-----------|------------|---------|
| OK | 200 | Success |
| CANCELLED | 499 | Client canceled |
| INVALID_ARGUMENT | 400 | Bad request |
| DEADLINE_EXCEEDED | 504 | Timeout |
| NOT_FOUND | 404 | Resource not found |
| ALREADY_EXISTS | 409 | Conflict |
| PERMISSION_DENIED | 403 | Forbidden |
| UNAUTHENTICATED | 401 | Auth required |
| RESOURCE_EXHAUSTED | 429 | Rate limit / quota |
| FAILED_PRECONDITION | 412 | Precondition failed |
| UNAVAILABLE | 503 | Service down |
| INTERNAL | 500 | Server error |

**Rich error model**: Attach structured metadata to errors using `google.rpc.Status`:

```java
io.grpc.Status status = io.grpc.Status.INVALID_ARGUMENT
    .withDescription("Quantity must be positive");
StatusRuntimeException exception = status.asRuntimeException();
Metadata metadata = new Metadata();
metadata.put(Metadata.Key.of("sku", Metadata.ASCII_STRING_MARSHALLER), "SKU-999");
throw exception;
```

#### Load balancing

gRPC is HTTP/2, which reuses a single TCP connection. A naive Kubernetes Service (ClusterIP) load-balances at connection open, then all requests on that connection go to the same pod. If you have 10 pods and 1 client, 9 pods sit idle.

**Solutions**:
1. **Headless Service + client-side load balancing**: Set `clusterIP: None`. gRPC client resolves all pod IPs via DNS and round-robins requests.
2. **Service mesh (Istio/Linkerd)**: Sidecar proxy terminates HTTP/2 and load-balances at L7.
3. **Proxyless gRPC (xDS)**: gRPC client speaks xDS protocol to a control plane (Istio, Envoy), gets endpoint list, does client-side LB without a sidecar.

For internal gRPC in Kubernetes, use headless service + client-side LB or a mesh. Do not use a ClusterIP service without understanding the connection reuse issue.

#### When gRPC wins

- **Internal services**: Type safety, performance, streaming.
- **High QPS**: Binary serialization is 3–10× smaller and faster than JSON.
- **Polyglot**: Protobuf generates clients for 20+ languages.
- **Streaming**: Server push, bidirectional streams (chat, live updates, log tailing).

#### When gRPC loses

- **Public APIs**: Browsers need grpc-web (proxy required); REST is simpler.
- **Debugging**: Binary payloads. `grpcurl` helps, but HTTP + JSON is easier.
- **Logs**: Protobuf in logs is unreadable. REST logs JSON.
- **Firewalls/proxies**: HTTP/2 multiplexing can confuse legacy middleboxes.

**Default recommendation**: Use gRPC for internal service-to-service. Use REST for public APIs, browser clients, and anything external developers touch.

### GraphQL

GraphQL solves the problem of over-fetching and under-fetching in REST. A mobile client that needs `{userId, username, avatarUrl}` but gets a full user object with 50 fields wastes bandwidth. A client that needs a user plus their last 5 orders makes two REST calls; in GraphQL it is one query.

#### Schema and query example

```graphql
type Query {
  order(id: ID!): Order
  orders(userId: ID!, limit: Int): [Order!]!
}

type Order {
  id: ID!
  status: OrderStatus!
  items: [OrderItem!]!
  total: Float!
  user: User!
}

type OrderItem {
  sku: String!
  quantity: Int!
  product: Product!
}

type Product {
  sku: String!
  name: String!
  price: Float!
}
```

Client query:

```graphql
query {
  order(id: "12345") {
    id
    status
    items {
      sku
      quantity
      product {
        name
        price
      }
    }
  }
}
```

The server returns exactly the fields requested, nothing more.

#### The N+1 query problem

**Plain English:** If you fetch 100 orders and each order has a `user` field, a naive GraphQL implementation makes 1 query for orders, then 100 separate queries to fetch each user. That is 101 queries when 2 would suffice.

**Analogy:** You are organizing a party and need to pick up 10 friends. The naive approach is to drive to each friend's house individually (10 trips). The batched approach is to pick up friends who live nearby in one trip (2 trips). The first is N+1; the second is batching.

**In the real world:** Instagram's feed loads posts and each post's author. Without batching, loading 50 posts means 1 query for posts + 50 queries for authors. With batching, it is 1 query for posts + 1 batched query for all 50 authors. Facebook's DataLoader library (ported to every GraphQL ecosystem) solved this in 2016; every production GraphQL server uses it.

**Mechanics:** The **DataLoader** pattern collects all user IDs requested in a single event loop tick, then issues one batched query.

```java
// Spring GraphQL + DataLoader
@Component
class UserDataLoader implements BatchLoader<String, User> {
    private final UserRepository userRepo;
    
    @Override
    public Mono<Map<String, User>> load(Set<String> userIds) {
        return userRepo.findAllById(userIds)
            .collectMap(User::getId);
    }
}

// Resolver
@SchemaMapping(typeName = "Order")
class OrderResolver {
    CompletableFuture<User> user(Order order, DataLoader<String, User> userLoader) {
        return userLoader.load(order.getUserId());
    }
}
```

The framework queues `load(order1.userId)`, `load(order2.userId)`, …, then calls `load(Set.of(id1, id2, ...))` once. The batch query:

```sql
SELECT * FROM users WHERE id IN ('id1', 'id2', ..., 'id100');
```

**What breaks:** Forgetting to use DataLoader. Symptom: slow queries, 1000+ database queries for a single GraphQL request. Check the logs; if you see repeated identical queries, you have N+1. Fix: wrap every resolver that fetches related data in a DataLoader.

#### Production controls

GraphQL is Turing-complete; clients can write arbitrarily expensive queries. Without limits, an attacker (or a careless developer) can bring down your API.

**Query depth limiting**: Prevent deeply nested queries.

```graphql
query {
  order { user { orders { user { orders { user { ... } } } } } }
}
```

Limit depth to 5–10 levels. Spring GraphQL:

```java
@Bean
GraphQlSource graphQlSource() {
    return GraphQlSource.schemaResourceBuilder()
        .schemaResources(...)
        .inspectSchemaMappings(mappings -> {
            mappings.inspector(new MaxQueryDepthInstrumentation(10));
        })
        .build();
}
```

**Query cost analysis**: Assign a cost to each field (e.g., fetching a list costs 10 × limit). Reject queries over a total cost budget.

**Persisted queries**: Clients send a hash of the query, not the query text. Server validates the hash against a pre-approved whitelist. This prevents arbitrary queries and shrinks request size.

```http
POST /graphql
Content-Type: application/json

{"queryId": "a1b2c3d4", "variables": {"orderId": "12345"}}
```

Server looks up `a1b2c3d4` in a registry, executes the stored query.

#### Federation for multi-team schemas

**Apollo Federation** lets multiple teams own parts of the schema. The `order` service owns `Order`; the `user` service owns `User`. A gateway composes them into one graph.

```graphql
# order-service
type Order @key(fields: "id") {
  id: ID!
  userId: ID!
  status: OrderStatus!
}

extend type User @key(fields: "id") {
  id: ID! @external
  orders: [Order!]!
}
```

The gateway fetches `User` from user-service, then fetches `orders` from order-service. The client sees one unified schema.

**Trade-off**: Federation adds complexity (gateway, schema stitching, coordination). Use it only if you have multiple teams owning independent domains and you need a unified API for web/mobile clients.

#### Caching difficulty

REST resources have URLs; caching is URL-based. GraphQL queries are POST requests with a query in the body; every query is unique. Standard HTTP caches do not help.

**Solutions**:
- **Persisted queries**: The query hash becomes a cache key.
- **Automatic persisted queries (APQ)**: Client sends query hash; if the server has not seen it, client sends full query, server caches it.
- **Field-level caching**: Cache individual fields (product, user) with a TTL; dedupe fetches within a request.
- **Client-side normalized cache** (Apollo Client): Cache entities by ID, dedupe across queries.

Caching GraphQL is harder than caching REST. Budget for it.

#### When NOT to use GraphQL

**Do not use GraphQL for internal service-to-service communication.** It is designed for client-server, where the client is a mobile app or web frontend with unpredictable query patterns. Between two backend services, you know exactly what you need; a typed RPC (gRPC) or a purpose-built REST endpoint is simpler, faster, and easier to version.

**Anti-pattern**: Using GraphQL as a universal API gateway for all microservices. You end up with a single team (the gateway team) bottlenecking every schema change, or you use federation and pay the complexity tax. GraphQL is best for consolidating multiple backends for a specific client need (mobile BFF, public API), not as internal infrastructure.

### Serialization formats

| Format | Size (relative) | Speed | Schema | Human-readable | Ecosystem |
|--------|-----------------|-------|--------|----------------|-----------|
| **JSON** | 1× (baseline) | Medium | Optional (JSON Schema) | Yes | Universal |
| **Protobuf** | 0.3× | Fast | Required (`.proto`) | No | gRPC, Google |
| **Avro** | 0.4× | Fast | Required (JSON schema) | No | Kafka, Hadoop |
| **MessagePack** | 0.7× | Fast | No | No | Redis, RPC |

**Compression**: gzip reduces JSON to ~0.3× original size but costs CPU. Modern HTTP/2 and gRPC compress by default. For Kafka, enable `compression.type=snappy` or `lz4` (low CPU, decent ratio).

**Numbers matter**: A 10 KB JSON payload becomes 3 KB Protobuf. At 100,000 requests/second, that is 700 KB/s vs 300 KB/s — 400 KB/s saved, ~1 TB/month of bandwidth. For a single request, irrelevant. At scale, it matters.

**Payload size discipline**: Do not return 50 fields when the client needs 5. Use sparse fieldsets (REST) or GraphQL. Do not log full request/response bodies in production (PII leak, log volume). Sample at 1% or hash the payload.

### Service discovery

**Kubernetes Services** provide DNS: `http://order-service.production.svc.cluster.local`. The DNS name resolves to a stable ClusterIP; kube-proxy load-balances to healthy pods. This is the default for greenfield Kubernetes deployments.

**Eureka/Consul** (client-side discovery): Services register with a registry; clients query the registry for endpoint IPs and load-balance locally. Eureka is legacy but still shipping in Spring Cloud; use it if you have a brownfield Spring Cloud Netflix estate. For greenfield, Kubernetes DNS is simpler.

**Health-based endpoint removal**: Kubernetes removes endpoints that fail readiness probes. Eureka removes instances that miss heartbeats. The same principle: unhealthy instances disappear from the load balancer pool.

### Gateway vs BFF vs service mesh responsibility matrix

| Concern | API Gateway | BFF (Backend for Frontend) | Service Mesh (Istio/Linkerd) | Calling Service |
|---------|-------------|----------------------------|------------------------------|-----------------|
| **TLS termination** | Yes (public ingress) | No | Yes (mTLS between services) | No |
| **Authentication** | Yes (token validation) | Optional (per-client auth) | No | No |
| **Authorization** | Coarse (API key, rate limit) | No | No | Yes (business logic) |
| **Rate limiting** | Yes (per-client) | Optional | Yes (per-service) | No |
| **Routing** | Yes (path-based, canary) | No | Yes (traffic splitting) | No |
| **Aggregation** | No | Yes (fan-out to N services) | No | Sometimes (internal fan-out) |
| **Transformation** | No (anti-pattern) | Yes (reshape for client) | No | No |
| **Retries** | Simple (no business logic) | Optional | Yes (auto-retry with backoff) | Yes (resilience4j) |
| **mTLS** | No | No | Yes | No (mesh handles it) |
| **Observability** | Yes (access logs, metrics) | Yes | Yes (distributed tracing) | Yes (span creation) |
| **Business logic** | NEVER | Sometimes (client-specific rules) | NEVER | Yes |

**Rule**: Business logic never belongs in the gateway or mesh. A gateway that validates "quantity > 0" has crossed the line into application logic. If it has business rules, it is not a gateway; it is a service.

### Call-site reliability basics

Every synchronous call is a distributed systems problem. The callee might be slow, down, or overloaded. The caller must handle it.

#### Timeouts

**Default timeout**: Derived from the callee's p99.9 latency. If the callee's p99.9 is 200 ms, set client timeout to 500 ms (2.5× headroom). Measure this; do not guess.

Spring WebClient:

```java
WebClient client = WebClient.builder()
    .baseUrl("http://inventory-service")
    .clientConnector(new ReactorClientHttpConnector(
        HttpClient.create()
            .responseTimeout(Duration.ofMillis(500))
    ))
    .build();
```

**Deadline propagation**: In gRPC, deadlines propagate automatically. In HTTP, you must pass a timeout budget in a header (e.g., `X-Timeout-Ms: 450`) and decrement it at each hop. Otherwise, service A → B → C sets a 5-second timeout at every layer, and C gets 5 seconds when it should get 4.5 seconds.

#### Retry budgets and amplification

**Retry amplification math**: If service A calls B with 3 retries, and B calls C with 3 retries, and C calls D with 3 retries, one user request becomes 3 × 3 × 3 = 27 requests to D. A small spike in failures triggers exponential load.

**Retry budget**: Allow retries only if the error rate is below 10%. If 20% of requests are failing, retries make it worse (you are already overloaded). Track `(retries / total_requests)`; when it exceeds the budget, stop retrying.

**Idempotency and retries**: Only retry safe methods (GET) or idempotent methods (PUT, DELETE) by default. POST is not idempotent; retry only if the caller sends an `Idempotency-Key`.

## Production patterns

### API gateway

**What**: Single entry point for all external traffic. TLS termination, authentication, rate limiting, routing to backend services.

**When to use**: Always, for public APIs. Isolates internal services from the internet.

**When NOT**: For internal service-to-service calls. A gateway between microservices adds latency and a SPOF.

**Failure modes**: Gateway becomes a bottleneck. Circuit breakers must protect the gateway from slow backends. Misconfigured routing sends traffic to the wrong service.

**Spring Cloud Gateway snippet**:

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: order-service
          uri: http://order-service
          predicates:
            - Path=/api/orders/**
          filters:
            - StripPrefix=1  # remove /api
            - name: CircuitBreaker
              args:
                name: orderCB
                fallbackUri: forward:/fallback/orders
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 100
                redis-rate-limiter.burstCapacity: 200
```

### Backend for Frontend (BFF)

**What**: A service per client type (web BFF, mobile BFF, partner API BFF) that aggregates and reshapes data from multiple backend services.

**When to use**: When different clients need different payloads. Mobile needs minimal data; web dashboard needs everything.

**When NOT**: When all clients consume the same API. Over-use creates redundant aggregation logic.

**Failure modes**: BFF becomes a monolith. Shared logic leaks into multiple BFFs (copy-paste). Timeout chains amplify latency.

**Pattern**: Fan-out to N services in parallel, aggregate, return.

```java
@RestController
class MobileBFFController {
    private final WebClient orderClient;
    private final WebClient inventoryClient;
    private final WebClient pricingClient;
    
    @GetMapping("/mobile/product/{sku}")
    Mono<MobileProductView> getProduct(@PathVariable String sku) {
        Mono<Product> product = orderClient.get()
            .uri("/products/{sku}", sku)
            .retrieve()
            .bodyToMono(Product.class);
        
        Mono<Integer> stock = inventoryClient.get()
            .uri("/inventory/{sku}", sku)
            .retrieve()
            .bodyToMono(Integer.class);
        
        Mono<BigDecimal> price = pricingClient.get()
            .uri("/pricing/{sku}", sku)
            .retrieve()
            .bodyToMono(BigDecimal.class);
        
        return Mono.zip(product, stock, price)
            .map(tuple -> new MobileProductView(
                tuple.getT1().name(),
                tuple.getT2(),
                tuple.getT3()
            ))
            .timeout(Duration.ofMillis(800));  // fail fast
    }
}
```

### ETag-based optimistic concurrency

**What**: Client sends `If-Match` with ETag; server returns 412 if version mismatch.

**When to use**: Any resource multiple clients might modify concurrently (cart, order, inventory allocation).

**When NOT**: Read-only resources, append-only logs.

**Failure modes**: Client ignores 412 and retries without refetching → infinite loop or wrong merge.

```java
@PutMapping("/cart/{id}")
ResponseEntity<CartResponse> updateCart(
    @PathVariable String id,
    @RequestHeader(value = "If-Match", required = false) String ifMatch,
    @RequestBody CartRequest request
) {
    Cart current = cartRepo.findById(id)
        .orElseThrow(() -> new NotFoundException("Cart not found"));
    
    String currentEtag = "\"v" + current.getVersion() + "\"";
    
    if (ifMatch != null && !ifMatch.equals(currentEtag)) {
        return ResponseEntity.status(412)
            .eTag(currentEtag)
            .build();
    }
    
    Cart updated = current.withItems(request.items());
    updated = cartRepo.save(updated);  // version column auto-increments
    
    return ResponseEntity.ok()
        .eTag("\"v" + updated.getVersion() + "\"")
        .body(CartResponse.from(updated));
}
```

### Idempotency filter

**What**: Servlet filter that checks `Idempotency-Key` header, deduplicates requests.

**When to use**: All non-idempotent endpoints (POST).

**When NOT**: GET/PUT/DELETE (already idempotent by design).

**Failure modes**: Missing unique constraint on idempotency key → duplicate execution. Forgetting to fingerprint request body → replay with wrong payload.

```java
@Component
class IdempotencyFilter extends OncePerRequestFilter {
    private final IdempotencyService idempotency;
    
    @Override
    protected void doFilterInternal(HttpServletRequest request, 
                                     HttpServletResponse response, 
                                     FilterChain chain) throws IOException, ServletException {
        if (!"POST".equals(request.getMethod())) {
            chain.doFilter(request, response);
            return;
        }
        
        String key = request.getHeader("Idempotency-Key");
        if (key == null) {
            response.setStatus(428);
            response.getWriter().write("Idempotency-Key required");
            return;
        }
        
        CachedBodyHttpServletRequest cachedRequest = new CachedBodyHttpServletRequest(request);
        String fingerprint = computeFingerprint(cachedRequest.getBody());
        
        IdempotencyRecord record = idempotency.findOrCreate(key, fingerprint);
        
        if (record.status() == COMPLETE) {
            response.setStatus(record.responseStatus());
            response.getWriter().write(record.responseBody());
            return;
        }
        
        chain.doFilter(cachedRequest, response);
        // post-filter: capture response, store in idempotency record
    }
}
```

### Async request-reply over HTTP (202 pattern)

**What**: Client POSTs; server returns `202 Accepted` with a status URL. Client polls or receives a webhook.

**When to use**: Long-running operations (export, batch processing) where user cannot wait.

**When NOT**: User-blocking operations (checkout).

**Failure modes**: Client never polls; operation completes but user never learns. Status URL expires.

```java
@PostMapping("/exports")
ResponseEntity<Void> createExport(@RequestBody ExportRequest request) {
    String jobId = UUID.randomUUID().toString();
    exportService.startExportAsync(jobId, request);
    
    return ResponseEntity.accepted()
        .location(URI.create("/exports/" + jobId + "/status"))
        .build();
}

@GetMapping("/exports/{jobId}/status")
ResponseEntity<ExportStatus> getExportStatus(@PathVariable String jobId) {
    ExportStatus status = exportService.getStatus(jobId);
    
    if (status.state() == COMPLETE) {
        return ResponseEntity.ok()
            .header("Link", "</exports/" + jobId + "/download>; rel=\"download\"")
            .body(status);
    } else {
        return ResponseEntity.ok()
            .header("Retry-After", "5")
            .body(status);
    }
}
```

### Webhooks

**What**: Server calls client's HTTP endpoint when an event occurs.

**When to use**: Third-party integrations (payment gateway → your service), async notifications.

**When NOT**: Internal service-to-service (use messaging).

**Failure modes**: Webhook URL is down; retries overwhelm the client. Replay attacks.

**Pattern**: Sign payloads (HMAC), retry with exponential backoff, provide replay endpoint.

```java
@PostMapping("/webhooks/payment")
void handlePaymentWebhook(@RequestBody String payload,
                          @RequestHeader("X-Signature") String signature) {
    String expectedSig = computeHMAC(payload, webhookSecret);
    if (!MessageDigest.isEqual(signature.getBytes(), expectedSig.getBytes())) {
        throw new UnauthorizedException("Invalid signature");
    }
    
    PaymentEvent event = parsePayload(payload);
    
    // Idempotent processing: check event ID
    if (processedEvents.contains(event.id())) {
        return;  // already processed
    }
    
    paymentService.handleEvent(event);
    processedEvents.add(event.id());
}
```

Client must provide a replay endpoint for debugging: `POST /webhooks/replay?eventId=xyz`.

### Batch/bulk endpoints

**What**: `POST /orders/bulk` accepts array of orders, processes in one transaction or background job.

**When to use**: Admin tools, data imports, mobile sync.

**When NOT**: User-facing single-item operations (slower, harder to debug).

**Failure modes**: One item fails; entire batch rolls back. Partial success hidden.

**Pattern**: Return `207 Multi-Status` with per-item results.

```java
@PostMapping("/orders/bulk")
ResponseEntity<BulkOrderResponse> createBulkOrders(@RequestBody List<OrderRequest> requests) {
    List<BulkItemResult> results = requests.stream()
        .map(req -> {
            try {
                Order order = orderService.createOrder(req);
                return new BulkItemResult(201, order.getId(), null);
            } catch (ValidationException e) {
                return new BulkItemResult(422, null, e.getMessage());
            }
        })
        .toList();
    
    return ResponseEntity.status(207)
        .body(new BulkOrderResponse(results));
}

record BulkItemResult(int status, String id, String error) {}
```

## How big tech does it

**Google's API Improvement Proposals (AIPs)** are the most influential API design system outside REST itself. AIPs define resource-oriented design as a discipline: every API models resources (nouns), standard methods (list, get, create, update, delete), and custom methods only when standard methods do not fit. AIP-132 (list) mandates pagination with `page_size` and `page_token`, never offset. AIP-134 (update) requires field masks (`update_mask`) so clients specify exactly which fields to update, preventing the "send entire object on every update" waste. AIP-151 (long-running operations) standardizes the pattern: return an operation ID, poll `/operations/{id}`, get the result when `done: true`. Google Cloud APIs (Compute, BigQuery, Firestore) all follow AIPs; external developers see consistency across 200+ APIs. The lesson: **consistency across a large API surface requires governance as code**. AIPs are published, versioned, and enforced in code review. Zapier's public API evolved similarly after pain from ad-hoc design.

**Stripe's API** is the reference implementation every fintech copies. Date-based versioning (`Stripe-Version: 2024-11-20`) decouples client upgrades from Stripe deployments; Stripe maintains compatibility shims for ~6 years of API versions. Idempotency keys are required for all mutation endpoints; the client SDK auto-generates them by default. Errors follow a rigid structure: `type` (e.g., `card_error`, `invalid_request_error`), `code` (e.g., `card_declined`, `parameter_invalid`), `param` (which field failed), `message` (human-readable). Clients can programmatically handle `card_declined` differently from `insufficient_funds`. Stripe's [API changelog](https://stripe.com/docs/upgrades) shows every breaking change with exact date and migration path; deprecated fields remain functional for years. The shared rule: **design errors as a product, not an afterthought**. Stripe's error response is more carefully designed than some companies' success responses.

**Amazon's internal service contract culture**: Every service exposes a machine-readable contract (originally WSDL, now often Smithy models). A service cannot call another service unless it declares the dependency and the contract in its model. Breaking changes require consumer approval or a migration plan. This shows up in AWS's public APIs: every service has an API model (Smithy or OpenAPI); SDKs are generated from the model. The change process is heavyweight: breaking changes require an ARD (Architecture Review Document), consumer migration timeline, and multi-quarter rollout. The lesson: **contract enforcement prevents the "we only have one consumer" trap**. That one consumer becomes ten consumers; by then the undocumented assumptions are load-bearing. Amazon's culture assumes every internal service will eventually be externalized (it often is: S3 was internal first).

**Netflix's API layer evolution**: Netflix's edge layer has been rebuilt three times. Originally a monolithic API that returned device-specific views (one `/browse` endpoint, server logic chose what to return based on client type). This became unmaintainable at 100+ device types. Falcor (2015) let clients declare data dependencies as a graph; the server stitched data from microservices. Falcor was too Netflix-specific and never reached wide adoption. By 2020, Netflix migrated to **GraphQL federation** for device APIs. Each domain team owns a GraphQL schema slice (catalog, playback, recommendations); the edge gateway composes them. The mobile client sends one query; the gateway fans out to N services. The lesson: **BFF consolidation is real, but the implementation technology matters less than team ownership boundaries**. Falcor failed partly because one team owned the entire runtime; federation succeeded because domain teams own their schema.

**Uber's DOMA (Domain-Oriented Microservices Architecture)** formalizes the concept: a domain (e.g., rider, driver, matching) owns a collection of services behind a gateway. External teams call the domain gateway, not individual services. Inter-domain calls are gRPC; intra-domain calls can be gRPC or local function calls (if a domain is deployed as one binary). Uber reports ~70% of inter-service traffic is gRPC (as of public engineering blog posts 2022–2023); REST exists for public APIs and legacy estate. gRPC wins for Uber because they have hundreds of services, polyglot teams (Go, Java, Node), and strong typing prevents entire classes of integration bugs. The lesson: **type safety at the wire format pays off at scale**. At 10 services, JSON + manual testing works. At 1000 services, Protobuf + generated clients prevent a category of production incidents (type mismatches, field renames, silent nulls).

**Slack's adoption of gRPC internally** (described in public talks 2021–2023): Slack's backend was originally PHP + MySQL, evolved to a polyglot estate (Java, Go, Node). REST + JSON required manual client code; every service had bespoke error handling. Migrating to gRPC gave them generated clients, automatic retries (via gRPC retry policies), and deadline propagation. Slack's edge (the gateway that serves web/mobile) remains HTTP/JSON because browsers and mobile SDKs are first-class citizens; internal services are gRPC. The pattern: **public API is REST for ergonomics; internal API is gRPC for reliability**. Slack's reported win: fewer integration bugs (type mismatches caught at compile time), easier polyglot development (one `.proto` file generates clients for 6 languages).

**Shopify's GraphQL-first public API** (2020 onward) replaced a sprawling REST API with 1000+ endpoints. The GraphQL API launched with federation: each domain (products, orders, customers) owns a subgraph. Partners (third-party app developers) can fetch exactly what they need in one query. The cost: Shopify had to build **query cost analysis** (every field has a cost; queries over the cost budget are rejected), **rate limiting per query complexity**, and **persisted queries** (only approved queries can run). Shopify publicly described the N+1 problem at scale: early partners wrote queries that fetched 50 products, then 50 × inventoryLevels, then 50 × metafields — 1 + 50 + 50 queries, all in one GraphQL request. DataLoader fixed it, but required training every backend engineer. The lesson: **GraphQL ships observability and cost-control complexity to the server**. REST's simplicity (one URL, one query) is a feature; GraphQL's flexibility requires runtime guards.

**Zalando's RESTful API Guidelines** are published and enforced across 200+ teams. Every API must pass automated linting (Spectral rules) in CI. Rules include: use `kebab-case` for paths, `snake_case` for JSON fields (controversial, but consistent), return RFC 9457 Problem Details for errors, use cursor pagination (`cursor` parameter, not `offset`), include `Deprecation` and `Sunset` headers when retiring endpoints. Zalando's guideline evolution: originally mandated HATEOAS; relaxed after internal teams found it too heavyweight for simple CRUD. The lesson: **published guidelines only work if they are enforced in CI and evolve with feedback**. A 100-page PDF no one reads is worse than 20 Spectral rules that fail the build.

**Shared rules across these examples**:
1. **Machine-readable contracts are non-negotiable at scale** (Google AIPs, Amazon Smithy, Uber Protobuf, Shopify GraphQL schema).
2. **Versioning strategy must be chosen early and applied everywhere** (Stripe's date-based, Google's additive-only, Amazon's major version in path).
3. **Errors are part of the API contract** (Stripe's error taxonomy, Google's standard error codes, Zalando's Problem Details).
4. **Idempotency is required, not optional** (Stripe, Amazon, Uber all require idempotency keys for mutations).
5. **Public API and internal API can use different protocols** (Slack/Netflix REST for public, gRPC for internal).
6. **Governance without automation fails** (Zalando's Spectral rules, Google's API reviewers, Amazon's model validation).

**Caveats**:
- **Google's AIPs assume resource-oriented design fits your domain**. Long-running batch operations and RPC-style commands (start simulation, trigger workflow) do not fit naturally; custom methods exist but feel like escapes.
- **Stripe's date-based versioning requires significant engineering investment**. Maintaining 6 years of API versions means compatibility shims, parallel test suites, and internal version-translation layers. Viable for a product company whose API is the product; overkill for internal microservices.
- **Netflix/Shopify's GraphQL success stories come with operational costs they do not always highlight**: query cost analysis, N+1 detection, persisted query whitelisting, schema federation coordination. GraphQL is not "adopt and it just works."
- **Uber's DOMA requires strong organizational boundaries**. A domain gateway makes sense if domains align with team ownership. If every feature needs 3 domains, you have created a coordination problem.

## Best-practice checklist

Contract and schema:
- [ ] **Contract-first workflow**: OpenAPI/Protobuf spec is source of truth; code is generated.
- [ ] **Breaking-change detection in CI**: `oasdiff` or `buf breaking` fails the build on schema violations.
- [ ] **Additive-only evolution**: new fields optional; never remove, rename, or repurpose fields.
- [ ] **Semantic versioning for breaking changes**: major version in URI (`/v2/`) or header; document migration path.
- [ ] **Published API changelog**: every change logged with date, impact, and migration guide.
- [ ] **Field naming consistency**: `snake_case` or `camelCase`, never mixed; apply org-wide.
- [ ] **No abbreviations**: `customerId`, not `custId`; `inventoryStatus`, not `invStat`.

Pagination and query limits:
- [ ] **Every list endpoint is paginated**: no unbounded `GET /products`.
- [ ] **Cursor-based pagination for large datasets**: encode opaque cursor, not offset.
- [ ] **Default page size and max page size**: default 20, max 100; reject `limit=999999`.
- [ ] **Stable sort order**: include tie-breaker (e.g., `ORDER BY createdAt DESC, id ASC`).
- [ ] **Consistent pagination metadata**: `nextCursor`, `hasMore` in response; or IETF RFC 8288 `Link` header.

Mutations and idempotency:
- [ ] **All non-idempotent mutations require `Idempotency-Key`**: return `428 Precondition Required` if missing.
- [ ] **Idempotency key scoped per resource type**: `POST /orders` and `POST /payments` can reuse same key.
- [ ] **Store idempotency records with TTL**: 24 hours for most; 7 days for financial.
- [ ] **Fingerprint request body**: prevent key reuse with different payload.
- [ ] **Concurrent duplicate detection**: unique constraint on key; return `409 Conflict` if in-progress.

Errors and observability:
- [ ] **RFC 9457 Problem Details for errors**: `type`, `title`, `status`, `detail`, `instance`.
- [ ] **Machine-readable error codes**: `VALIDATION_ERROR`, `RATE_LIMITED`, `CONFLICT`.
- [ ] **`retryable` boolean in error response**: clients know whether to retry.
- [ ] **Consistent HTTP status code usage**: `400` vs `422` vs `409` vs `412` correctly.
- [ ] **Never return `200 OK` with error in body**: status code is the contract.
- [ ] **No PII in URLs or logs**: user ID is fine; email/phone/SSN is not.
- [ ] **Request ID in every response**: `X-Request-ID` header; include in logs for tracing.

Timeouts and deadlines:
- [ ] **Every outbound call has a timeout**: derived from callee's p99.9 + headroom.
- [ ] **Deadline propagation**: gRPC automatic; HTTP manual (decrement and pass in header).
- [ ] **gRPC deadline in every client call**: `withDeadlineAfter(duration)`.
- [ ] **Client timeout shorter than server timeout**: prevent zombie requests.

Authentication and authorization:
- [ ] **Every endpoint requires authentication**: no unauthenticated routes except health/metrics.
- [ ] **OAuth 2.0 or API keys for external APIs**: never Basic Auth over HTTP.
- [ ] **Authorization checked in business logic, not gateway**: gateway validates token; service checks permissions.
- [ ] **Return `401` for missing/invalid token, `403` for insufficient permissions**: do not conflate.
- [ ] **Avoid leaking existence via 403 vs 404**: return `404` for access-controlled resources.

Versioning and deprecation:
- [ ] **Deprecation announced 6–12 months ahead**: public APIs 12 months; internal 3–6 months.
- [ ] **`Deprecation` and `Sunset` headers on deprecated endpoints**: RFC 8594.
- [ ] **Consumer inventory maintained**: know who uses each version; contact before retirement.
- [ ] **Migration guide published**: what changed, how to upgrade, example diff.
- [ ] **Retired endpoints return `410 Gone`, not `404`**: clear signal.

Protocol-specific:
- [ ] **gRPC: never renumber protobuf fields**: mark removed fields `reserved`.
- [ ] **gRPC: enums start at 0 and include `UNSPECIFIED`**: required for safe deserialization.
- [ ] **GraphQL: query depth limit (5–10 levels)**: prevent deeply nested attacks.
- [ ] **GraphQL: query cost analysis**: assign cost per field; reject over-budget queries.
- [ ] **GraphQL: DataLoader for every N+1 risk**: batch related fetches.
- [ ] **REST: ETag and `If-Match` for concurrent updates**: prevent lost updates.
- [ ] **REST: `OPTIONS` returns allowed methods**: support CORS preflight.

## Anti-patterns and war stories

### Anti-patterns

**Chatty APIs requiring N calls to render a screen**: Mobile client needs product, inventory, price, reviews. Four separate calls; four round-trips; 200 ms latency becomes 800 ms. **Fix**: Aggregation endpoint (`GET /products/{sku}/view` returns everything) or GraphQL. **Caveat**: Do not over-aggregate; `/products/{sku}/everything` that includes 10 MB of related data is worse. Aggregate what the screen needs, nothing more.

**Unbounded list endpoints**: `GET /products` returns 500,000 products as a 200 MB JSON array. Client times out; server OOMs. **Fix**: Mandatory pagination. **Detection**: Monitor response sizes; alert on >1 MB. **Real incident**: A customer-facing report endpoint returned all transactions (no limit); a user with 2 years of data crashed the server.

**Breaking changes shipped "because only one consumer uses it"**: Team A removes a field from `/orders`; Team B's service breaks. Team A says "you should have told us you depend on that field." **Fix**: Consumer registry; breaking-change detection in CI; contract tests that consumers own. **Reality**: The "one consumer" assumption is always wrong; logging, analytics, and cron jobs you forgot about also consume the API.

**Verbs in paths as a religion vs pragmatism**: REST purists demand `POST /orders/{id}/cancellations` instead of `POST /orders/{id}/cancel`. The resource is "cancellation," not "cancel the order." In practice, `cancel` is clearer. **Rule**: Prefer nouns (`/cancellations`), but do not contort the API to avoid a verb when the verb is the natural name for the operation. `POST /orders/{id}/ship` is fine. **Anti-pattern**: `/orders/{id}/shipments` when shipments are not a first-class resource you can list/get/update independently.

**GraphQL between internal services**: Service A calls Service B with a GraphQL query. This couples A to B's schema; B cannot evolve the schema without coordinating with A. gRPC or a bespoke REST endpoint is simpler. **Reality**: GraphQL is for clients with unpredictable query patterns (mobile app, web UI), not for service-to-service where you know exactly what you need. **Symptom**: GraphQL query hardcoded in Service A; might as well be a typed RPC.

**Gateway holding business logic**: API Gateway has a rule: "if `productCategory == electronics`, add 10% tax." This is business logic. The gateway is infrastructure; business logic belongs in services. **Fix**: Gateway routes, authenticates, rate-limits; services decide tax. **Symptom**: Gateway config is 5000 lines of conditional logic; deploying the gateway requires business approval.

**Shared DTO library across service boundaries**: `common-models.jar` contains `OrderDTO`, shared by `order-service`, `shipping-service`, `billing-service`. One team adds a field; all services must recompile and redeploy. **Fix**: Each service owns its models; translate at the boundary. **Reality**: Shared DTOs are tight coupling disguised as code reuse. **Exception**: Within a single deployment unit (monorepo with 5 modules), a shared model is fine. Across service boundaries, it is coupling.

**Using HTTP 200 for failures**: `{"success": false, "error": "Order not found"}` with `200 OK`. Proxies cache it; monitoring does not see the failure; client libraries that check status codes think it succeeded. **Fix**: Use the correct status code (`404`); put error details in the body.

**Ignoring deadline propagation**: Service A calls B with a 5-second timeout; B calls C with a 5-second timeout; C calls D with a 5-second timeout. One request to A can take 15 seconds (timeouts add). **Fix**: Decrement deadline at each hop. gRPC does this automatically; HTTP requires manual `X-Timeout-Ms` header propagation. **Symptom**: Deep call chains have unbounded total latency.

**Protobuf field renumbering**: Developer removes `optional string email = 2;` and later adds `optional int32 age = 2;`. Old clients send a string; new servers expect an int; deserialization fails silently or crashes. **Fix**: `reserved 2; reserved "email";` when removing a field. **Detection**: `buf breaking` in CI catches this.

### War story 1: The missing deadline that took down the site

**Incident**: A product manager requested a new export feature: "Download all orders as CSV." Engineering built `GET /admin/orders/export`, which queries the database for all orders, formats them as CSV, and returns a 50 MB file. The query takes 30 seconds on average. The team tested it in staging (1000 orders, instant). It shipped.

First week: fine. Second week: a customer exported 500,000 orders. The query took 90 seconds. The load balancer's default timeout was 60 seconds; it killed the connection. The client retried. The retry hit a different backend server; that query also took 90 seconds. The retry queue built up; 10 retries in-flight, each holding a database connection for 90 seconds. The connection pool exhausted. **New orders could not be saved** (no available connections). Revenue-impacting outage.

**Root cause**: No timeout on the export query; no deadline propagation; the retry logic did not check idempotency. The client sent `Idempotency-Key`, but the server did not honor it (the export was read-only, so the team thought idempotency did not apply).

**Fix**: (1) Async export: `POST /admin/exports` returns `202 Accepted` and a job ID; background worker processes it; client polls `/admin/exports/{id}/status` and downloads the result. (2) Timeout on the database query: 10 seconds max; if it times out, return `202` and process async. (3) Idempotency check even for GET: if the same export is requested twice, return the cached result.

**Lesson**: Long-running operations must be async. Timeouts must exist at every layer. Idempotency applies to any operation that allocates server resources.

### War story 2: The "harmless" required field

**Incident**: A protobuf schema for `OrderCreatedEvent` initially had:

```protobuf
message OrderCreatedEvent {
  string order_id = 1;
  string user_id = 2;
  double total = 3;
}
```

A team added a new field for internal analytics:

```protobuf
message OrderCreatedEvent {
  string order_id = 1;
  string user_id = 2;
  double total = 3;
  string warehouse_id = 4;  // newly required
}
```

The change was reviewed and approved. The team deploying the producer service started populating `warehouse_id`. 

**Problem**: Five consumer services were reading `OrderCreatedEvent`. Three were on the latest schema; two were still on the old schema (they had not redeployed in months). The old schema clients deserialized the message; `warehouse_id` was unknown (field 4 did not exist in their `.proto` file). Protobuf's default behavior: ignore unknown fields. So far, fine.

But one of the old clients had validation logic: "if `warehouse_id` is set, check inventory at that warehouse." The code was:

```java
if (!event.getWarehouseId().isEmpty()) {
    checkInventory(event.getWarehouseId());
}
```

In the old schema, `getWarehouseId()` did not exist. The code did not compile after the schema update — except the old client had not updated the schema, so it compiled fine against the old schema. When the new events arrived, Protobuf deserialized them successfully (ignoring the unknown field). But the Java code never saw the field because the generated Java class did not have the getter. The validation logic never ran. **Orders were created without inventory checks.**

**Detection**: An internal audit noticed 3% of orders had no warehouse assigned. Investigation revealed the schema mismatch.

**Root cause**: Adding a field is safe; relying on its presence in old clients is not. The field was added as optional (proto3 default), but the producer immediately started populating it, and a consumer assumed its presence.

**Fix**: (1) Mark the field as genuinely optional; consumers must handle `warehouse_id` being empty. (2) Gradual rollout: add the field, deploy all consumers with the new schema (they ignore the field), then deploy producers populating it, then deploy consumers using it. Three-phase rollout. (3) Schema registry with compatibility checks: Confluent Schema Registry or Buf Schema Registry catches this (you cannot require a field that did not exist before).

**Lesson**: Protobuf field additions are backward-compatible only if consumers tolerate the field being absent. Never assume a newly added field will be populated in all messages. Even optional fields must be treated as nullable.

### War story 3: The unstable sort that corrupted the ledger

**Incident**: A financial SaaS company provided a transaction export API: `GET /transactions?limit=1000&offset=0`. Customers used it to sync transactions into their internal ledger. The query:

```sql
SELECT * FROM transactions ORDER BY created_at LIMIT 1000 OFFSET ?;
```

`created_at` was a timestamp with millisecond precision. Multiple transactions could have the same timestamp (batch imports, high-frequency trading). The `ORDER BY created_at` was non-deterministic: if 10 transactions had `created_at = '2026-09-12 10:00:00.123'`, their order in the result set was arbitrary.

**Problem**: Customer fetched page 1 (offset 0), got transactions 1–1000. Between page 1 and page 2, 50 new transactions were inserted at the beginning (earlier `created_at`). Customer fetched page 2 (offset 1000). The database returned rows 1000–2000 of the new dataset, which included 50 rows that were in the 950–1000 range of page 1. **Duplicates.**

Worse: some rows in the 1000–1050 range of the original dataset were pushed to offset 1050, so page 2 skipped them. **Missing transactions.**

The customer's ledger had duplicate entries and missing entries. The accounting did not balance. The customer filed a support ticket: "Your API is returning duplicate transactions."

**Root cause**: Non-deterministic sort + offset pagination + concurrent inserts.

**Fix**: (1) Add a tie-breaker to the sort: `ORDER BY created_at, id`. `id` is unique, so the sort is deterministic. (2) Migrate to cursor pagination: the cursor encodes `{created_at, id}` of the last row; next page is `WHERE (created_at, id) > (cursor.created_at, cursor.id) ORDER BY created_at, id LIMIT 1000`. This is stable even with concurrent inserts.

**Detection**: Internal testing did not catch it because test data had unique timestamps. Load testing with concurrent writes would have caught it.

**Lesson**: Any sort key that is not unique must include a tie-breaker. Offset pagination is fragile under concurrent modification; cursor pagination is deterministic.

## Projects for this phase

### Small projects

**Idempotency-key middleware** (5–8 hours): Build a Spring Boot starter that provides idempotency for `POST` endpoints. Accepts `Idempotency-Key` header, stores request fingerprint and response in Redis (TTL 24 hours), replays on duplicate. Handles concurrent requests (unique constraint on key, return `409 Conflict`). Deliverable: `@IdempotentEndpoint` annotation; documented usage; unit tests proving duplicate requests return cached response; load test proving no duplicate execution under concurrent retries.

**Cursor pagination library** (6–10 hours): Generic pagination utility for JPA/Spring Data. Encodes cursor as base64 JSON `{sort_field: value, id: value}`. Generates `WHERE (sort_field, id) > (?, ?)` clause. Supports multi-field sort. Returns `PageResult<T>` with `nextCursor`, `hasMore`. Deliverable: library code; example usage in a REST controller; unit tests proving stable sort under concurrent inserts; benchmark proving performance matches hand-written query.

**gRPC service with deadlines and interceptors** (8–12 hours): Implement a gRPC service (`OrderService`) that calls two other gRPC services (`InventoryService`, `PricingService`) in parallel. Enforce deadline propagation (if caller sets 5s, downstream calls get 4.9s). Add server interceptor that logs request ID, user ID, duration. Add client interceptor that propagates request ID in metadata. Deliverable: three services running; client call with 2s deadline; log entries proving deadline propagation; client cancellation scenario (cancel mid-flight, server stops processing).

**OpenAPI lint + breaking-change gate in CI** (6–10 hours): Set up Spectral with custom rules: all paths versioned (`/v1/`, `/v2/`), all operations have `operationId` and `summary`, all errors use Problem Details schema, no `offset` pagination (only cursor). Add `oasdiff breaking` check comparing feature branch spec to `main`. Fail PR build on errors. Deliverable: `.spectral.yaml`, CI workflow, example spec passing lint, example spec failing lint with error message, example breaking change rejected in CI.

**Webhook delivery service with retries and signing** (10–15 hours): Background worker that delivers webhooks to customer-provided URLs. Exponential backoff (1s, 2s, 4s, 8s, 16s, max 5 retries). HMAC signature in `X-Signature` header. Dead-letter queue after max retries. Idempotent delivery (track delivered event IDs). Deliverable: service code; webhook receiver endpoint (test harness); configuration for webhook URL and secret; log proving retries with backoff; signature verification example; dead-letter queue populated after max retries.

### Large project: ShopKart Edge (API gateway + mobile BFF + web BFF + contract governance)

**Goal**: Build the edge layer for ShopKart that exposes REST APIs for web and mobile clients, aggregates data from backend services, and enforces API contracts.

**Scope** (30–50 hours):

**Services**:
- **API Gateway** (Spring Cloud Gateway): TLS termination, authentication (JWT validation), rate limiting (Redis-based), routing to BFFs and backend services. Routes: `/v1/catalog/**` → `catalog-service`, `/v1/cart/**` → `cart-service`, `/mobile/**` → `mobile-bff`, `/web/**` → `web-bff`.
- **Mobile BFF**: Aggregates product + inventory + pricing in one call (`GET /mobile/product/{sku}`). Returns minimal payload (id, name, price, stockStatus). Parallel fan-out with WebClient; timeout 500 ms.
- **Web BFF**: Aggregates product + reviews + recommendations (`GET /web/product/{sku}`). Returns full payload. Parallel fan-out; timeout 1 second.
- **Backend services** (stubs or real): `catalog-service`, `inventory-service`, `pricing-service`, `review-service`.

**API contract governance**:
- OpenAPI specs for mobile-bff and web-bff stored in `specs/` directory.
- Spectral lint rules: all paths versioned, pagination mandatory for lists, errors use Problem Details.
- CI job runs `oasdiff breaking` against `main` branch spec.
- Breaking change requires version bump and `CHANGELOG.md` entry.

**Resilience**:
- Circuit breaker on gateway routes (Resilience4j).
- Timeouts on all WebClient calls in BFFs.
- Fallback responses: if pricing service is down, return product with `priceUnavailable: true`.

**Observability**:
- Request ID generated at gateway, propagated to BFFs and backend services (via header).
- Micrometer metrics: request count, error rate, latency per route.
- Distributed tracing with Micrometer Tracing + Zipkin.

**Authentication**:
- Gateway validates JWT (HS256 or RS256); extracts `userId` claim; passes in `X-User-ID` header to downstream services.
- BFFs and backend services trust the header (no re-validation).

**Acceptance criteria**:
- Web client can fetch product view (product + reviews + recommendations) in one call; 3 backend services called in parallel; response time <1s at p99.
- Mobile client gets minimal product view; response size <5 KB.
- Invalid JWT returns `401 Unauthorized` at gateway.
- Rate limit exceeded returns `429 Too Many Requests` with `Retry-After` header.
- Catalog service down → circuit breaker opens → returns `503 Service Unavailable`.
- Breaking change to mobile-bff spec fails CI.
- Request ID traces through gateway → BFF → backend services; visible in Zipkin.

**Stretch goals**:
- Idempotency middleware for cart mutations (`POST /cart/items`).
- Cursor pagination for order history (`GET /mobile/orders?cursor=...`).
- GraphQL gateway as alternative to REST BFFs; compare complexity and performance.
- Canary routing: 5% of traffic to `mobile-bff:v2`, 95% to `v1`.

**Time box**: 40 hours (20 hours core implementation; 10 hours resilience and observability; 10 hours contract governance and testing).

## Interview drilldown

**Question 1: When would you choose REST vs gRPC vs GraphQL for these three scenarios: (a) a public API for third-party developers, (b) internal service-to-service communication in a microservices estate, (c) a mobile app calling a backend?**

**Strong answer**: (a) REST for public APIs. Third-party developers need browser compatibility, easy testing (Postman, curl), human-readable responses, and extensive documentation. GraphQL is viable if the API surface is large and clients need flexible querying (Shopify, GitHub), but it requires query cost controls and a learning curve. gRPC is not practical (no browser support without grpc-web proxy). (b) gRPC for internal service-to-service. Type safety (Protobuf), performance (binary serialization), automatic deadline propagation, and generated clients in every language. REST is acceptable for low-QPS CRUD services or when debugging ease outweighs performance. GraphQL does not make sense here — you know exactly what you need, so a typed RPC is simpler. (c) Mobile app → BFF → backend: REST or GraphQL at the BFF layer (client-facing), gRPC for BFF → backend. Mobile benefits from REST's simplicity or GraphQL's flexibility (reduce over-fetching). Internal calls benefit from gRPC's performance. The choice depends on query flexibility: if every screen needs a different payload shape, GraphQL wins; if screens align with specific endpoints, REST wins.

**Follow-up: What if your public API users complain about bandwidth (they are on slow mobile networks)?**
- **Strong answer**: Negotiate compression (gzip, Brotli) and sparse fieldsets (`?fields=id,name,price`). If that is insufficient, consider GraphQL so clients request only needed fields. Alternative: provide a "lite" REST endpoint per high-traffic use case (`/products/{sku}/mobile-view`).

**Weak answer signals**: "gRPC is faster so use it for everything" (ignores browser compatibility and debugging). "REST is old, GraphQL is the future" (ignores GraphQL's operational cost). "Use whatever the team knows" (no engineering rationale).

---

**Question 2: Design an idempotent payment API. What fields go in the request? How do you handle retries? What happens if the payment gateway times out?**

**Strong answer**: 

**Request**:
```json
POST /payments
Idempotency-Key: <uuid>
{
  "orderId": "order-123",
  "amount": 99.99,
  "currency": "USD",
  "paymentMethod": "card",
  "cardToken": "tok_xyz"
}
```

**Idempotency**: Server stores `{idempotencyKey, requestFingerprint, status, response}` in database with unique constraint on key. On retry, if `status == complete`, replay cached response (200 + original payment ID). If `status == in_progress`, return 409 Conflict (concurrent retry). If not found, insert record, call payment gateway, update status to complete, return response.

**Payment gateway timeout**: If the gateway times out (no response in 10s), the server does not know if the payment succeeded. Return `202 Accepted` with a status URL (`/payments/{id}/status`). Background job polls the gateway for the payment result; updates the status; webhooks can notify when the result is known. On retry, if status is still `pending`, return the same `202` and status URL. If the retry happens after the background job learned the result, return the final result (200 + payment ID or 402 Payment Failed).

**Concurrency**: Unique constraint on idempotency key prevents double-charging. Database constraint fails; one request wins, the other retries and sees `in_progress`.

**TTL**: Idempotency record expires after 7 days (financial data; longer than the typical 24 hours). After expiry, the same key can be reused (new transaction).

**Follow-up: What if the payment succeeded but the server crashed before storing the result? The client retries; do you charge twice?**
- **Strong answer**: The payment gateway must deduplicate on its own idempotency key (most gateways support this — Stripe, PayPal, Adyen all do). Send the same idempotency key to the gateway on retry; the gateway returns the original payment result. Alternatively, query the gateway by `orderId` before retrying to check if a payment already exists.

**Weak answer signals**: "Store payment status in Redis" (Redis can lose data; financial transactions need durable storage). "Let the client handle retries" (client cannot know if server received the request). "If timeout, just fail the payment" (user loses money or sees an error for a successful payment).

---

**Question 3: How do you version a REST API without breaking existing clients? Walk me through adding a required field to a request.**

**Strong answer**: You cannot add a required field to an existing endpoint without breaking clients; that is a breaking change by definition. Options:

1. **Additive evolution**: Add the field as optional (`nullable` or with a default). Old clients omit it; server uses a default value or treats it as null. New clients send it. Eventually, after all clients migrate, make it required in v2 of the API. This requires a major version bump.

2. **New endpoint**: Create `/v2/orders` with the new required field; keep `/v1/orders` unchanged. Old clients use v1; new clients use v2. Deprecate v1 after a migration period (6–12 months). Both endpoints can call the same business logic internally; the v1 adapter supplies a default value for the new field.

3. **Content negotiation**: Use `Accept: application/vnd.shopkart.v2+json` to request the v2 response format. This keeps the URL stable but requires clients to change the header. Less common than URI versioning.

**Migration process**: (1) Announce v2 with the new required field. (2) Publish migration guide showing the diff. (3) Add `Deprecation: true` and `Sunset: Sat, 01 Jun 2025 00:00:00 GMT` headers to v1 responses. (4) Monitor v1 usage (track `User-Agent` or API key). (5) Contact active users 3 months before sunset. (6) Retire v1 on the sunset date (return `410 Gone`).

**Follow-up: What if you have 100 clients and cannot contact them all (public API)?**
- **Strong answer**: Extend the deprecation period to 12–18 months. Log every v1 request with API key; send automated emails. Provide a self-service migration dashboard showing which endpoints they call and which are deprecated. After sunset, return `410 Gone` with a link to the migration guide. Some clients will break; that is the cost of a public API. The alternative is maintaining v1 forever, which is unsustainable.

**Weak answer signals**: "Just add the field and make it required; clients will update" (no, they will break). "Use feature flags to toggle the field" (feature flags are for deployment, not API versioning). "Make the field optional but fail the request if it is missing" (that is still a breaking change, just delayed to runtime).

---

**Question 4: How do you paginate 100 million rows efficiently?**

**Strong answer**: Cursor-based pagination, not offset. Offset pagination (`LIMIT 1000 OFFSET 50000000`) scans and discards 50 million rows; it is O(n) in the offset. Cursor pagination uses a WHERE clause: `WHERE id > last_seen_id ORDER BY id LIMIT 1000`, which is O(log n) with an index.

**Cursor design**: Encode the last-seen sort key as a base64 opaque token. If sorting by `created_at, id`:
```sql
WHERE (created_at, id) > (cursor.created_at, cursor.id)
ORDER BY created_at DESC, id ASC
LIMIT 1000;
```

**Index**: `CREATE INDEX idx_pagination ON table(created_at DESC, id ASC)`. Without the index, the query still scans the table.

**Client side**: Return `nextCursor` in the response. Client sends it in the next request: `GET /items?cursor=eyJjcmVhdGVkQXQiOiIy...`

**Edge cases**: (1) Cursor must encode all sort fields; otherwise ties are non-deterministic. (2) If a row is deleted after the client fetched it, the cursor still works (you just skip past that row). (3) If new rows are inserted before the cursor, the client sees them on the next page (acceptable for most use cases; if not, use snapshot isolation or timestamp-based filtering).

**Seek method (alternative)**: Some ORMs call this "seek" or "keyset" pagination. Hibernate, jOOQ, and Spring Data JPA all support it.

**Follow-up: What if the user needs to jump to page 50?**
- **Strong answer**: Cursor pagination does not support random access. If you need page numbers (admin UIs, search result pages), use offset pagination but add a max offset (e.g., `OFFSET 10000` max). For datasets >10k rows, hide the page numbers and use infinite scroll (cursor-based). Google search does this: you cannot jump to page 100; you paginate forward.

**Weak answer signals**: "Use `OFFSET`; it is fine" (no at 100M rows). "Load all rows into memory and paginate in-app" (OOM). "Use a cache" (caching does not fix slow queries; it hides them).

---

**Question 5: How does gRPC handle load balancing in Kubernetes? Why does a standard Service not work well?**

**Strong answer**: gRPC uses HTTP/2, which multiplexes multiple requests over a single TCP connection. Kubernetes Service (ClusterIP) load-balances at connection establishment (L4, TCP SYN). Once a client opens a connection to pod A, all subsequent requests on that connection go to pod A. If you have 10 pods and 1 client, 9 pods are idle.

**Solutions**:

1. **Headless Service + client-side load balancing**: Set `clusterIP: None`. The DNS query returns all pod IPs. The gRPC client (using `grpc-go`, `grpc-java`, etc.) resolves all IPs and load-balances requests across them (usually round-robin). The client opens one connection per pod and distributes requests.

2. **Service mesh (Istio, Linkerd)**: The sidecar proxy intercepts gRPC traffic, terminates the HTTP/2 connection, and load-balances at L7 (per-request). The client thinks it is talking to one endpoint; the proxy fans out to pods.

3. **Proxyless gRPC (xDS)**: The gRPC client speaks the xDS protocol to a control plane (e.g., Istio's istiod) and gets the list of pod IPs. The client does client-side load balancing without a sidecar. This is experimental in most languages but GA in `grpc-go` and `grpc-java`.

**Recommendation**: For internal gRPC in Kubernetes, use headless service + client-side LB (simplest) or a service mesh if you already have one. Do not use a standard ClusterIP service without understanding the connection-reuse issue.

**Follow-up: What if the client is outside Kubernetes (external traffic)?**
- **Strong answer**: Use an L7 load balancer that understands HTTP/2 (Envoy, NGINX with `grpc_pass`, or a cloud LB like AWS ALB with gRPC support). The LB terminates HTTP/2 connections and load-balances per-request to backend pods.

**Weak answer signals**: "Use a regular Service; it just works" (no, it does not for gRPC). "Open multiple connections from the client" (wasteful; client-side LB is cleaner). "Avoid gRPC in Kubernetes" (gRPC is common in Kubernetes; you just need the right LB).

---

**Question 6: What is the N+1 query problem in GraphQL, and how do you detect and fix it?**

**Strong answer**: 

**Problem**: A GraphQL query fetches 100 orders, and each order has a `user` field. A naive resolver fetches the user for each order:
```javascript
orders.forEach(order => {
  order.user = fetchUser(order.userId);  // 100 queries
});
```
That is 1 query for orders + 100 queries for users = 101 queries.

**Detection**: Enable database query logging; see repeated `SELECT * FROM users WHERE id = ?` with different IDs. Or use a GraphQL query analyzer that counts database queries per request. If a query fetches N items and makes N+1 database queries, you have N+1.

**Fix: DataLoader**. A DataLoader batches requests within a single event loop tick:
```javascript
// Resolver
user(order) {
  return userLoader.load(order.userId);  // queues the load
}

// DataLoader
const userLoader = new DataLoader(async (userIds) => {
  const users = await db.query('SELECT * FROM users WHERE id IN (?)', [userIds]);
  return userIds.map(id => users.find(u => u.id === id));
});
```
The loader collects all `userIds`, issues one query, and returns the results.

**In Java (Spring GraphQL)**:
```java
@SchemaMapping(typeName = "Order")
class OrderResolver {
    CompletableFuture<User> user(Order order, DataLoader<String, User> userLoader) {
        return userLoader.load(order.getUserId());
    }
}

@Component
class UserDataLoader implements BatchLoader<String, User> {
    @Override
    public Mono<Map<String, User>> load(Set<String> userIds) {
        return userRepo.findAllById(userIds).collectMap(User::getId);
    }
}
```

**Follow-up: What if users can belong to multiple organizations, and you need to fetch organizations for each user? Is that N+1 again?**
- **Strong answer**: Yes, and you fix it the same way: a `BatchLoader` for organizations. The pattern composes: DataLoader for users, then DataLoader for organizations. Total queries: 1 (orders) + 1 (users) + 1 (organizations) = 3, regardless of result size.

**Weak answer signals**: "Cache the user queries" (does not fix the underlying issue; you still make 100 queries on cache miss). "Use JOINs in the database" (GraphQL resolvers are decoupled; you cannot always use JOINs). "Fetch everything in one query" (breaks GraphQL's resolver model).

---

**Question 7: A client needs to know if an order shipped. The client waits for an answer. Synchronous or asynchronous communication? Justify.**

**Strong answer**: Synchronous. The client is blocking on the answer ("did it ship?"), so the response must be immediate. A synchronous request-reply (REST or gRPC) fits: `GET /orders/{id}` returns `{"status": "shipped"}` or `{"status": "pending"}`. The client gets the answer in one round-trip (10–100 ms).

Asynchronous messaging (Kafka, SQS) does not fit here because the client must poll or wait for a callback, adding latency. Async is for state propagation ("notify me when it ships"), not for queries ("has it shipped yet?").

**Caveat**: If the shipping status is expensive to compute (query a third-party carrier API), and the user can tolerate stale data, cache the status and return it synchronously from the cache. Refresh the cache asynchronously in the background.

**Follow-up: What if the order service needs to notify the shipping service, analytics service, and email service when an order is created?**
- **Strong answer**: Asynchronous. Publish an `OrderCreated` event to Kafka or a message queue. The three consumers subscribe and process independently. The order service does not wait for them (fan-out without blocking). If the email service is down, the order still succeeds.

**Weak answer signals**: "Always use async for scalability" (ignores user-blocking scenarios). "Use sync for everything because it is simpler" (ignores failure isolation). "Use async so the client does not wait" (the client still waits if you return 202 and poll).

---

**Question 8: Design an error response for a REST API. What fields do you include? How do you make it machine-readable?**

**Strong answer**: Use RFC 9457 Problem Details for HTTP APIs.

```json
{
  "type": "https://docs.shopkart.com/errors/insufficient-inventory",
  "title": "Insufficient Inventory",
  "status": 409,
  "detail": "Product SKU-12345 has only 2 units available; requested 5.",
  "instance": "/orders/67890",
  "code": "INSUFFICIENT_INVENTORY",
  "retryable": false,
  "sku": "SKU-12345",
  "available": 2,
  "requested": 5
}
```

**Fields**:
- **`type`**: URI identifying the error type (links to docs).
- **`title`**: Short human-readable summary.
- **`status`**: HTTP status code (409).
- **`detail`**: Human-readable explanation specific to this occurrence.
- **`instance`**: URI of the specific request (order ID, transaction ID).
- **`code`**: Machine-readable error code (for programmatic handling).
- **`retryable`**: Boolean; should client retry?
- **Extension fields** (`sku`, `available`, `requested`): Context for debugging or client logic.

**Machine-readable**: Clients switch on `code` or `type`:
```java
if (error.code().equals("INSUFFICIENT_INVENTORY")) {
    // show "out of stock" UI
} else if (error.code().equals("PAYMENT_DECLINED")) {
    // prompt for different payment method
}
```

**Standard codes**: `VALIDATION_ERROR` (400), `UNAUTHORIZED` (401), `FORBIDDEN` (403), `NOT_FOUND` (404), `CONFLICT` (409), `RATE_LIMITED` (429), `INTERNAL_ERROR` (500).

**Follow-up: What if multiple fields fail validation? How do you return all errors?**
- **Strong answer**: Include an `errors` array:
```json
{
  "type": "https://docs.shopkart.com/errors/validation",
  "status": 400,
  "errors": [
    {"field": "email", "code": "INVALID_FORMAT", "message": "Email is not valid"},
    {"field": "quantity", "code": "OUT_OF_RANGE", "message": "Quantity must be between 1 and 100"}
  ]
}
```

**Weak answer signals**: `{"error": "Something went wrong"}` (not specific). `{"success": false, "message": "..."}` with 200 OK (wrong status code). Including stack traces in production (security leak).

---

**Question 9: What is a deadline, and why does it matter in distributed systems?**

**Strong answer**: A deadline is the absolute time by which a request must complete. It propagates through a call chain: if Service A calls Service B with a 5-second deadline, and B calls C, C should get a 4.9-second deadline (accounting for A→B latency). When the deadline expires, all pending operations are canceled, freeing resources.

**Why it matters**: Without deadlines, slow requests accumulate. Service C takes 30 seconds; B waits 30 seconds; A waits 30 seconds. If A's timeout is 10 seconds, A gives up but B and C keep working (wasted CPU). With deadlines, when A's deadline expires, the cancellation propagates to B and C; they stop processing immediately.

**gRPC**: Deadlines are built-in. Client sets `withDeadlineAfter(5, SECONDS)`; gRPC propagates it in the `grpc-timeout` header. Server checks the deadline; if expired, returns `DEADLINE_EXCEEDED`.

**HTTP**: Manual. Client sends `X-Timeout-Ms: 5000`. Each service decrements it (A→B: `X-Timeout-Ms: 4900`) and enforces the timeout locally. No standard; each org invents its own header.

**Symptom of missing deadlines**: Deep call chains have unbounded latency. A→B→C→D; each layer has a 10s timeout; total latency can be 40s. With deadlines, total latency is capped at A's deadline.

**Follow-up: What happens if a service ignores the deadline and keeps processing?**
- **Strong answer**: The client cancels and moves on; the server wastes resources. In gRPC, the server receives a cancellation signal and should check `Context.isCancelled()` in long-running loops. In HTTP, the connection closes; the server should check if the client disconnected. If the server ignores cancellation, it processes a request no one cares about.

**Weak answer signals**: "Deadline is the same as timeout" (timeout is per-hop; deadline is end-to-end). "Set a long timeout so requests do not fail" (defeats the purpose; a 60s deadline means failed requests hold resources for 60s). "Just retry if it times out" (without deadlines, retries make the problem worse).

---

**Question 10: How do you deprecate an API endpoint that 40 internal teams use?**

**Strong answer**:

1. **Announce**: Email all teams; post in internal docs. Explain why (e.g., migrating to v2 with better performance, breaking change required). Provide a migration guide: what changed, how to update, example code diff.

2. **Add deprecation headers**: `Deprecation: true` and `Sunset: Sat, 01 Jun 2025 00:00:00 GMT` (RFC 8594). Clients see it in response headers.

3. **Track usage**: Log every request to the deprecated endpoint; capture team ID or service name. Build a dashboard showing which teams are still using it.

4. **Contact teams individually**: If a team has not migrated 3 months before sunset, ping them. Offer to pair-program the migration if needed.

5. **Provide a compatibility shim**: If possible, route old endpoint to new endpoint with a translation layer. This buys time for slow-moving teams.

6. **Enforce**: On the sunset date, return `410 Gone` for all requests to the old endpoint. Include a link to the migration guide in the response body.

7. **Monitor**: Check error dashboards for 410s. If a critical team missed the deadline and is broken, provide a temporary workaround (re-enable the endpoint for that team only) while they migrate.

**Timeline**: 6 months for internal teams; 12 months for public APIs.

**Follow-up: What if a critical team refuses to migrate (e.g., they are swamped with other work)?**
- **Strong answer**: Escalate to management. Deprecation is a contract; you cannot maintain two versions forever. If they cannot migrate, you maintain the old endpoint indefinitely (technical debt) or the new feature is blocked. Force the decision up the chain. Alternatively, offer to do the migration for them (pair with their team, submit the PR).

**Weak answer signals**: "Just delete the endpoint; they will figure it out" (breaks production). "Support both forever" (unsustainable). "Add a feature flag" (feature flags are for deployment, not long-term versioning).

---

**Question 11: You have a public REST API. A client sends the same `POST` request 5 times because they think it failed (network blip). What happens?**

**Strong answer**: If the endpoint requires an `Idempotency-Key` header, the server executes the operation once and replays the response for retries 2–5. The client sees 5 identical successful responses; the server only charged the card (or created the order) once.

If the endpoint does not require an idempotency key, the server executes the operation 5 times. The client is charged 5 times, or 5 duplicate orders are created. This is a bug in the API design.

**Best practice**: All non-idempotent operations (POST) should require `Idempotency-Key`. If the client omits it, return `428 Precondition Required`.

**Client responsibility**: The client must generate a unique key per logical operation. If the client re-uses the same key for two different operations (two different orders), the server detects it (request fingerprint mismatch) and returns `422 Unprocessable Entity: Idempotency key reused`.

**Follow-up: What if the client sends 5 concurrent requests with the same idempotency key?**
- **Strong answer**: The database has a unique constraint on `idempotency_key`. The first request inserts the record; the other 4 fail the constraint and retry. They see `status == in_progress` and return `409 Conflict` with `Retry-After: 1`. The client waits 1 second and retries; by then, the first request completed, and the retry gets the cached response.

**Weak answer signals**: "The client should not retry" (networks fail; retries are necessary). "Let the client deduplicate" (client cannot know if the server processed the request). "Store idempotency keys in memory" (lost on restart; financial operations need durable storage).

---

**Question 12: Walk me through designing a paginated API for a ledger (financial transactions). What constraints do you add?**

**Strong answer**:

**Requirements**:
- Ledger must be consistent: no duplicates, no skipped rows, deterministic order.
- Pagination must be stable under concurrent inserts.
- Cursor-based pagination (offset is unreliable for large datasets).

**Design**:
```http
GET /transactions?cursor=<base64>&limit=100
```

**Cursor encodes**: `{timestamp, transaction_id}`. Both are indexed.

**Query**:
```sql
SELECT * FROM transactions
WHERE (timestamp, transaction_id) > (?, ?)
ORDER BY timestamp ASC, transaction_id ASC
LIMIT 100;
```

**Index**: `CREATE INDEX idx_transactions_pagination ON transactions(timestamp ASC, transaction_id ASC);`

**Constraints**:
1. **Deterministic sort**: `ORDER BY timestamp, transaction_id`. `transaction_id` is unique; this guarantees no ties.
2. **Immutable sort key**: Once a transaction is written, its `timestamp` and `transaction_id` never change. Mutable sort keys (e.g., `updated_at`) break cursor pagination (a row can move forward/backward between pages).
3. **No deletions**: Ledgers are append-only. If you must "delete," mark as `deleted=true` and filter in the query. Actual deletion breaks cursors (next page might reference a deleted row).
4. **Cursor validation**: Decode and validate the cursor. If it is malformed or references a future timestamp (client tampering), return `400 Bad Request`.
5. **Max page size**: `limit` defaults to 100, max 1000. Reject `limit=999999`.

**Response**:
```json
{
  "transactions": [...],
  "nextCursor": "eyJ0aW1lc3RhbXAiOiIy...",
  "hasMore": true
}
```

**Edge case**: If a transaction is inserted between page 1 and page 2, it appears on page 2 (acceptable for most ledgers). If the client needs a snapshot view (page 1 and page 2 must reflect the same dataset), add a `snapshot_timestamp` parameter: `WHERE timestamp <= snapshot_timestamp`.

**Follow-up: What if the client wants to paginate backward (previous page)?**
- **Strong answer**: Reverse the inequality: `WHERE (timestamp, transaction_id) < (cursor)` and `ORDER BY timestamp DESC, transaction_id DESC`. The cursor encodes the first item of the current page; the query fetches items before it.

**Weak answer signals**: "Use `OFFSET`" (does not scale). "Return all transactions; let the client paginate" (financial ledgers can have millions of rows). "Just paginate by `id`" (if `id` is auto-increment, this works, but `timestamp` is a business requirement for ledgers).

## Level signals: Senior / Staff / Principal

### Senior Engineer

**API design**: Designs REST endpoints following conventions (plural nouns, HTTP methods, status codes). Implements pagination and filtering. Handles errors with appropriate status codes and readable messages. Writes OpenAPI specs for documentation.

**Contracts**: Adds fields additively; avoids breaking changes. Knows when a change is breaking (removing field, changing type, adding required field).

**Idempotency**: Implements `Idempotency-Key` support for critical endpoints (payments, order creation). Understands retries and deduplication.

**Protocols**: Can implement gRPC or GraphQL following tutorials and examples. Fixes N+1 queries with DataLoader after being told it is a problem.

**Interview signals**: Answers "When to use REST vs gRPC" with basic tradeoffs (REST for public, gRPC for internal). Designs a pagination API with offset (does not know cursor is better). Forgets to include error codes or retryability in error responses. Does not mention deadline propagation.

### Staff Engineer

**API design**: Designs evolvable APIs: additive evolution, versioning strategy (URI or header), deprecation plan with sunset dates. Uses RFCs (9457, 8594) correctly. Designs batch endpoints with partial success (`207 Multi-Status`).

**Contracts**: Enforces contract-first workflow; OpenAPI or Protobuf is source of truth. Sets up CI gates for breaking changes (Spectral, `buf breaking`). Maintains a consumer registry; coordinates migrations.

**Idempotency**: Implements idempotency infrastructure (shared library, database schema, fingerprinting). Handles edge cases: concurrent duplicates, request fingerprint mismatches, TTL cleanup.

**Protocols**: Chooses protocol based on constraints (sync vs async, fan-out, failure isolation). Implements gRPC with deadlines, interceptors, and client-side load balancing (headless service in Kubernetes). Debugs and fixes N+1 in GraphQL proactively.

**Resilience**: Adds timeouts, circuit breakers, and retry budgets. Understands retry amplification (exponential load from nested retries).

**Interview signals**: Designs cursor pagination without prompting. Explains deadline propagation and why it matters. Critiques a chatty API and proposes BFF or GraphQL. Describes idempotency with concurrent duplicate handling. Identifies protobuf field renumbering as a breaking change.

### Principal Engineer

**API design**: Defines organization-wide API standards (REST guidelines, gRPC conventions). Writes ADRs for protocol choices (why GraphQL for public API, gRPC for internal). Designs API gateways and BFFs with clear responsibility boundaries. Publishes API design reviews as teaching artifacts.

**Contracts**: Builds contract governance tooling: schema registry, automated compatibility checks, consumer-driven contract tests. Tracks API consumers across the org; enforces migration deadlines. Deprecates APIs org-wide (40+ teams).

**Protocols**: Migrates large estates from one protocol to another (Ribbon → Spring Cloud LoadBalancer, REST → gRPC). Debugs production incidents involving protocol mismatches, deadline propagation failures, or gRPC load balancing.

**Idempotency and reliability**: Designs distributed idempotency (idempotency keys that span multiple services). Handles cross-service retries and saga rollback. Defines retry policies org-wide (retry budgets, jitter, exponential backoff).

**Industry awareness**: References real examples (Stripe's versioning, Google's AIPs, Uber's DOMA, Netflix's GraphQL federation). Explains when practices do not apply (Stripe's date-based versioning is overkill for internal APIs).

**Teaching**: Writes runbooks, playbooks, and training material. Reviews others' API designs; spots anti-patterns (chatty APIs, missing deadlines, unbounded lists). Mentors Staff engineers on contract evolution.

**Interview signals**: Proposes a versioning strategy with organizational buy-in plan. Explains retry amplification with math (3 retries × 3 hops = 27× load). Designs a multi-team API gateway with ownership boundaries. Describes how Shopify's GraphQL cost analysis works and why it is necessary. Critiques GraphQL for internal service-to-service communication with specific reasons.

## Exit criteria

- [ ] I can design a REST API with correct HTTP methods, status codes, and pagination (cursor-based for large datasets).
- [ ] I can write an OpenAPI spec and generate server stubs or client SDKs from it.
- [ ] I can implement idempotent POST endpoints with `Idempotency-Key` and handle concurrent duplicates.
- [ ] I can evolve an API additively and identify breaking changes (removing field, adding required field, changing type).
- [ ] I can version a public API (URI or header versioning) and deprecate old versions with sunset headers.
- [ ] I can implement a BFF that aggregates data from 3+ backend services in parallel with timeouts.
- [ ] I can write a gRPC service with Protobuf schemas, deadlines, and interceptors.
- [ ] I understand gRPC load balancing in Kubernetes (headless service or service mesh required).
- [ ] I can implement GraphQL resolvers with DataLoader to prevent N+1 queries.
- [ ] I can set up contract tests or breaking-change detection in CI (Spectral, `oasdiff`, `buf breaking`).
- [ ] I can design errors with RFC 9457 Problem Details: `type`, `status`, `detail`, `code`, `retryable`.
- [ ] I can explain when to use REST vs gRPC vs GraphQL with technical justification (not just preference).
- [ ] I understand deadline propagation and why it matters in deep call chains.
- [ ] I can implement retry logic with exponential backoff, jitter, and retry budgets.
- [ ] I have built and verified one small project from this phase (idempotency middleware, cursor pagination, gRPC service with deadlines, or OpenAPI CI gate).

## Resources

**Books**:
- **"API Design Patterns"** by JJ Geewax (Google) — resource-oriented design, pagination, long-running operations, field masks. Heavily influenced by Google's AIPs.
- **"RESTful Web APIs"** by Leonard Richardson, Mike Amundsen — REST fundamentals, HATEOAS, hypermedia, content negotiation.
- **"Designing Data-Intensive Applications"** by Martin Kleppmann — Chapter 4 (Encoding and Evolution) covers schema evolution, Protobuf, Avro, compatibility.

**Specs and RFCs**:
- **RFC 9110** (HTTP Semantics) — defines HTTP methods, status codes, headers.
- **RFC 9457** (Problem Details for HTTP APIs) — standard error format.
- **RFC 8594** (Sunset Header) — deprecation signaling.

**Official docs**:
- **OpenAPI Specification** (https://spec.openapis.org/oas/latest.html) — OpenAPI 3.1.
- **Google API Improvement Proposals (AIPs)** (https://google.aip.dev/) — resource-oriented design, standard methods, field masks, long-running operations.
- **gRPC documentation** (https://grpc.io/docs/) — guides, best practices, language-specific tutorials.
- **GraphQL specification** (https://spec.graphql.org/) — official spec; federation docs at Apollo.
- **Buf Schema Registry docs** (https://buf.build/docs) — Protobuf linting, breaking-change detection, schema registry.

**API design guidelines (reference implementations)**:
- **Stripe API documentation** (https://stripe.com/docs/api) — idempotency, versioning, error design.
- **Zalando RESTful API Guidelines** (https://opensource.zalando.com/restful-api-guidelines/) — comprehensive, opinionated, public.
- **Microsoft REST API Guidelines** (https://github.com/microsoft/api-guidelines) — resource modeling, pagination, error handling.
- **Kubernetes API Conventions** (https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md) — versioning, compatibility, list pagination.

**Tools**:
- **Spectral** (https://stoplight.io/open-source/spectral) — OpenAPI linting.
- **oasdiff** (https://github.com/Tufin/oasdiff) — breaking-change detection for OpenAPI.
- **Postman** / **Insomnia** — API testing, mocking, documentation.
- **grpcurl** (https://github.com/fullstorydev/grpcurl) — curl for gRPC.
- **Buf CLI** (https://buf.build/) — Protobuf linting, breaking-change detection.

**Engineering blogs**:
- **Stripe Engineering Blog** — API versioning, idempotency, webhook reliability.
- **Netflix Tech Blog** — Falcor, GraphQL federation, API gateway evolution.
- **Uber Engineering Blog** — DOMA, gRPC adoption, domain-oriented architecture.
- **Shopify Engineering Blog** — GraphQL at scale, query cost analysis, N+1 problem.
- **Slack Engineering Blog** — gRPC migration, retries, deadline propagation.

**Practice**:
- Reverse-engineer public APIs (Stripe, GitHub, Twilio) — read their docs, note patterns (idempotency keys, error structure, pagination).
- Implement a simple API gateway (Spring Cloud Gateway or Envoy) and route to 2–3 backend services; add circuit breakers and timeouts.
- Write a GraphQL resolver for a 1-to-N relationship (order → items); verify DataLoader batches queries.
- Migrate a REST endpoint to gRPC; measure serialization size and latency difference.
