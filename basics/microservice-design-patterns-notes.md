# Microservice Design Patterns

## Why Design Patterns Matter in Microservices

In a monolith, problems like shared state, transactions, and configuration are solved by the framework. In microservices, every service is isolated — which creates a completely new set of problems:

- How does a client talk to 50 services?
- How do you roll out distributed transactions?
- What happens when one service brings down everything?
- How do you migrate from a monolith without rewriting everything?

Design patterns are **proven solutions** to these recurring problems. You don't invent them — you recognise the problem and apply the right pattern.

---

## 1. Gateway Pattern

**Problem:** A client needs to call 10 different microservices. It has to know all their ports, handle auth separately for each, and manage CORS for each — impossible to maintain.

**Solution:** Place a single entry point (the API Gateway) in front of all services. All clients talk only to the gateway.

```text
Client
  ↓
[ API Gateway ]  ← auth, logging, CORS, rate limiting live here
  ↙   ↓   ↘
Svc-A Svc-B Svc-C
```

**What the gateway handles centrally:**
- Authentication (validate JWT once, not in every service)
- Routing (path → service mapping)
- Rate limiting
- Logging / tracing
- CORS
- SSL termination

**In Spring Boot:** Spring Cloud Gateway

```properties
spring.cloud.gateway.routes[0].id=farmer-service
spring.cloud.gateway.routes[0].uri=lb://farmer-service
spring.cloud.gateway.routes[0].predicates[0]=Path=/api/v1/farmers/**
```

**Variants:**
- **Backend for Frontend (BFF):** One gateway per client type — a separate gateway for mobile (returns light payloads) and one for web (returns full data). Avoids over/under-fetching.

**When to use:** Always. Every microservices system needs a gateway.

---

## 2. Service Registry Pattern

**Problem:** In production, services don't have fixed IPs. Instances scale up/down dynamically. How does `order-service` find `payment-service`?

**Solution:** Maintain a central **service registry** where every service registers itself on startup and deregisters on shutdown.

```text
1. payment-service starts → registers at Eureka
   "I am payment-service, running at 10.0.0.5:8085"

2. order-service wants to call payment-service
   → asks Eureka: "where is payment-service?"
   → Eureka returns: 10.0.0.5:8085

3. order-service calls 10.0.0.5:8085
```

**Two discovery models:**

| Model | Who does the lookup? | Example |
|---|---|---|
| **Client-side discovery** | The calling service queries the registry and picks an instance | Eureka + Ribbon |
| **Server-side discovery** | The load balancer queries the registry (caller doesn't care) | AWS ALB + Route 53 |

**Spring Boot (Eureka):**

```java
// Eureka Server
@SpringBootApplication
@EnableEurekaServer
public class ServiceRegistryApplication { }
```

```java
// Every microservice
@SpringBootApplication
@EnableEurekaClient
public class FarmerServiceApplication { }
```

```properties
# application.properties — every service
eureka.client.service-url.defaultZone=http://localhost:8761/eureka
spring.application.name=farmer-service
```

```properties
# Gateway — use service name, not hardcoded IP
spring.cloud.gateway.routes[0].uri=lb://farmer-service
```

**Health check integration:** Eureka uses `/actuator/health` to periodically check if a registered instance is alive. If it fails, Eureka removes it from the registry automatically.

**Tools:** Netflix Eureka, Consul, Zookeeper, AWS Cloud Map

**When to use:** Any system where service instances are dynamic (containers, Kubernetes, cloud autoscaling).

---

## 3. Circuit Breaker Pattern

**Problem:** Service A calls Service B. Service B starts responding slowly or failing. Service A's threads pile up waiting. Service A runs out of threads. Service A goes down. The entire system cascades.

**Solution:** Wrap remote calls with a circuit breaker. When failure rate exceeds a threshold, the circuit "opens" — calls fail immediately without hitting the failing service.

```text
States:

CLOSED ──(failures cross threshold)──→ OPEN ──(wait duration)──→ HALF-OPEN
  ↑                                                                    │
  └────────────────(test call succeeds)───────────────────────────────┘
```

| State | Behaviour |
|---|---|
| **CLOSED** | Normal. All calls go through. Failures are counted. |
| **OPEN** | Calls fail immediately (fast fail). Fallback is returned. No calls to the failing service. |
| **HALF-OPEN** | A few test calls are allowed through. If they succeed → CLOSED. If they fail → OPEN again. |

**Spring Boot (Resilience4j):**

```xml
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot3</artifactId>
</dependency>
```

```java
@Service
public class OrderService {

    @CircuitBreaker(name = "paymentService", fallbackMethod = "paymentFallback")
    public PaymentResponse processPayment(PaymentRequest request) {
        return paymentClient.process(request);  // remote call
    }

    // Called when circuit is OPEN
    public PaymentResponse paymentFallback(PaymentRequest request, Exception ex) {
        log.warn("Payment service unavailable, returning pending status");
        return PaymentResponse.pending(request.getOrderId());
    }
}
```

```yaml
# application.yml
resilience4j:
  circuitbreaker:
    instances:
      paymentService:
        failure-rate-threshold: 50          # open if 50% of calls fail
        wait-duration-in-open-state: 10s    # stay OPEN for 10 seconds
        sliding-window-size: 10             # evaluate last 10 calls
        permitted-number-of-calls-in-half-open-state: 3
```

**Tools:** Resilience4j (modern), Netflix Hystrix (deprecated)

**When to use:** Any synchronous inter-service call that you cannot afford to cascade.

---

## 4. SAGA Pattern

**Problem:** A distributed transaction spans multiple services — e.g., placing an order requires `order-service`, `payment-service`, `inventory-service`, and `notification-service` all to succeed. There's no single database, so you can't use a SQL `COMMIT`. What happens if payment succeeds but inventory fails?

**Solution:** Break the transaction into a sequence of local transactions, each publishing an event. If any step fails, compensating transactions undo the previous steps.

```text
Place Order SAGA:

1. order-service        → Creates order (PENDING)     → emits OrderCreated
2. payment-service      → Charges card                → emits PaymentCompleted
3. inventory-service    → Reserves stock              → emits InventoryReserved
4. notification-service → Sends confirmation email    → emits NotificationSent
5. order-service        → Updates order to CONFIRMED
```

**Rollback via compensating transactions:**
```text
If inventory-service FAILS at step 3:

inventory-service → emits InventoryFailed
      ↓
payment-service   → refunds payment (compensating tx)
      ↓
order-service     → marks order as CANCELLED
```

**Two coordination models:**

### Choreography (Event-Driven)
Services react to each other's events. No central coordinator.

```text
order-service ──OrderCreated──→ Kafka
                                    ↓
                           payment-service processes it
                                    ↓
                           ──PaymentCompleted──→ Kafka
                                                     ↓
                                            inventory-service processes it
```

**Pros:** Loose coupling, simple, no single point of failure
**Cons:** Hard to visualise overall flow, debugging is difficult

### Orchestration (Central Coordinator)
A dedicated **SAGA Orchestrator** tells each service what to do next.

```text
SAGA Orchestrator
      │
      ├──→ "payment-service: charge card"
      │          ↓ PaymentCompleted
      ├──→ "inventory-service: reserve stock"
      │          ↓ InventoryFailed
      └──→ "payment-service: refund" (compensating)
```

**Pros:** Clear flow, easier to debug, centralised state
**Cons:** Orchestrator becomes a bottleneck, slightly more coupling

**Tools:** Axon Framework, Eventuate Tram, Temporal, custom Kafka-based orchestrator

**When to use:** Any operation that touches multiple services and must be atomic (order placement, booking, money transfer, user onboarding).

---

## 5. CQRS — Command Query Responsibility Segregation

**Problem:** A single database model serving both reads and writes creates conflicts. Writes need normalised data (no duplication). Reads need denormalised data (joins are expensive). Optimising for one hurts the other.

**Solution:** Separate the **write model** (Command side) from the **read model** (Query side).

```text
         WRITE SIDE                         READ SIDE
         (Commands)                         (Queries)

POST /orders ──→ OrderCommandService   GET /orders/summary ──→ OrderQueryService
                      ↓                                               ↑
               Normalised MySQL                              Denormalised MongoDB / Redis
               (orders, items,                               (pre-joined, ready to serve)
                payments tables)
                      ↓
               Event Published ─────────────────────────────→ Read DB updated
               (OrderCreated)          (async sync)
```

**Implementation sketch:**

```java
// Command side — write to normalised DB, publish event
@Service
public class OrderCommandService {

    public void placeOrder(PlaceOrderCommand command) {
        Order order = Order.from(command);
        orderRepository.save(order);                      // write to MySQL
        eventPublisher.publish(new OrderCreatedEvent(order)); // publish to Kafka
    }
}

// Read side — consume event, update denormalised read store
@KafkaListener(topics = "order.created")
public void on(OrderCreatedEvent event) {
    OrderSummary summary = OrderSummary.from(event);
    orderSummaryRepository.save(summary);                 // write to MongoDB / Redis
}

// Query side — super fast reads from denormalised store
@Service
public class OrderQueryService {
    public OrderSummary getOrderSummary(String orderId) {
        return orderSummaryRepository.findById(orderId);  // no joins, instant
    }
}
```

**CQRS + Event Sourcing:** Often paired together. Instead of storing current state, you store every event that led to that state. You can replay events to rebuild state at any point in time.

**When to use:**
- High read/write ratio (10:1 or more)
- Read and write models have very different shapes
- Audit logs or historical data are important
- Example: banking (every transaction is an event), e-commerce dashboards

**When NOT to use:** Simple CRUD applications — CQRS adds significant complexity.

---

## 6. Bulkhead Pattern

**Problem:** One slow or failing service consumes all the threads in your thread pool. Every other service that shares that pool also stops responding — even if those other services are perfectly healthy.

**Solution:** Isolate resources (threads, connections) per service — just like bulkheads in a ship divide the hull into compartments. If one compartment floods, the ship doesn't sink.

```text
WITHOUT Bulkhead:
───────────────────────────────
Thread Pool [T1 T2 T3 T4 T5 T6 T7 T8 T9 T10]
                ↓
payment-service slow → all 10 threads stuck waiting
→ inventory-service calls also start failing (no threads left)
───────────────────────────────

WITH Bulkhead:
───────────────────────────────
payment-service   pool: [T1 T2 T3]  ← only these 3 are stuck
inventory-service pool: [T4 T5 T6]  ← completely unaffected
notification pool: [T7 T8 T9]       ← completely unaffected
───────────────────────────────
```

**Spring Boot (Resilience4j):**

```java
@Bulkhead(name = "paymentService", type = Bulkhead.Type.THREADPOOL)
public CompletableFuture<PaymentResponse> processPayment(PaymentRequest request) {
    return CompletableFuture.supplyAsync(() -> paymentClient.process(request));
}
```

```yaml
resilience4j:
  bulkhead:
    instances:
      paymentService:
        max-concurrent-calls: 10    # max 10 concurrent calls to payment-service
        max-wait-duration: 500ms    # wait up to 500ms for a slot, then reject
```

**Two types of bulkhead:**

| Type | Mechanism | Use Case |
|---|---|---|
| **Thread Pool Bulkhead** | Separate thread pool per service | Most common — true isolation |
| **Semaphore Bulkhead** | Limit concurrent calls with a semaphore (same thread) | Reactive / non-blocking stacks |

**When to use:** When a single dependency failure should not affect other dependencies. Pair with Circuit Breaker for complete fault isolation.

---

## 7. Sidecar Pattern

**Problem:** Every microservice needs cross-cutting capabilities: logging, metrics, service discovery, TLS, distributed tracing. Implementing these in every service (possibly in different languages) is duplicate work and hard to update consistently.

**Solution:** Deploy a **sidecar** container alongside every service container. The sidecar handles all the infrastructure concerns. The service focuses only on business logic.

```text
┌──────────────────────────────────┐
│           Pod / VM               │
│                                  │
│  ┌─────────────┐  ┌───────────┐  │
│  │   farmer-   │  │  Sidecar  │  │
│  │   service   │◄─►│  (Envoy)  │  │
│  │  (business  │  │           │  │
│  │   logic)    │  │ - TLS     │  │
│  └─────────────┘  │ - Logging │  │
│                   │ - Metrics │  │
│                   │ - Tracing │  │
│                   └───────────┘  │
└──────────────────────────────────┘
```

**Real-world: Istio Service Mesh**

Istio automatically injects an **Envoy proxy** sidecar into every pod in a Kubernetes cluster. This sidecar intercepts all network traffic and provides:

- mTLS (mutual TLS) between services
- Distributed tracing (Jaeger/Zipkin)
- Traffic metrics (Prometheus)
- Load balancing
- Circuit breaking
- Canary deployments

The service code has zero knowledge of any of this.

**Sidecar use cases:**

| Sidecar | Provides |
|---|---|
| Envoy (Istio) | mTLS, tracing, load balancing, circuit breaking |
| Filebeat | Log shipping to Elasticsearch |
| Fluentd | Log aggregation |
| Vault Agent | Secret injection from HashiCorp Vault |
| Prometheus exporter | Expose custom metrics |

**When to use:** Polyglot environments (services in Java, Python, Go — all need logging/tracing); when you want infrastructure concerns managed separately from business logic; Kubernetes environments.

---

## 8. API Composition Pattern

**Problem:** A client needs data that lives in multiple services. For example, an order summary page needs data from `order-service`, `user-service`, `payment-service`, and `shipping-service`. Making 4 separate REST calls from the client is slow, chatty, and leaks internal service structure.

**Solution:** Compose responses from multiple services in one place — either in the API Gateway or in a dedicated **Aggregator Service** — and return a single combined response to the client.

```text
Client
  ↓  GET /api/v1/orders/123/summary
API Gateway / Aggregator
  ├──→ order-service      GET /orders/123          → { orderId, items, total }
  ├──→ user-service       GET /users/456           → { name, email }
  ├──→ payment-service    GET /payments/order/123  → { status, method }
  └──→ shipping-service   GET /shipping/order/123  → { trackingId, ETA }
        ↓ (combine all responses)
  ← { orderId, items, total, customer, payment, shipping }
Client (one response, one call)
```

**Implementation:**

```java
@Service
public class OrderSummaryAggregator {

    public OrderSummaryResponse getOrderSummary(String orderId) {

        // Call all services in parallel using CompletableFuture
        CompletableFuture<OrderDetails> orderFuture =
            CompletableFuture.supplyAsync(() -> orderClient.getOrder(orderId));

        CompletableFuture<PaymentDetails> paymentFuture =
            CompletableFuture.supplyAsync(() -> paymentClient.getPayment(orderId));

        CompletableFuture<ShippingDetails> shippingFuture =
            CompletableFuture.supplyAsync(() -> shippingClient.getShipping(orderId));

        // Wait for all to complete
        CompletableFuture.allOf(orderFuture, paymentFuture, shippingFuture).join();

        return OrderSummaryResponse.of(
            orderFuture.join(),
            paymentFuture.join(),
            shippingFuture.join()
        );
    }
}
```

**API Composition vs GraphQL:**

| Approach | Best For |
|---|---|
| API Composition | Fixed, known aggregation — e.g., order summary page |
| GraphQL | Dynamic, client-driven queries where different clients need different fields |

**When to use:** Any time a client needs data from more than one service. Always aggregate server-side, not client-side.

---

## 9. Event-Driven Architecture (EDA) Pattern

**Problem:** Services calling each other synchronously creates tight coupling. If `order-service` directly calls `notification-service`, `analytics-service`, and `loyalty-service` every time an order is placed — adding a new downstream service means modifying `order-service`.

**Solution:** Services communicate by **publishing events** to an event bus. Any service that cares about an event subscribes to it. The publisher has zero knowledge of its consumers.

```text
WITHOUT EDA:
order-service → calls notification-service (tight coupling)
              → calls analytics-service    (tight coupling)
              → calls loyalty-service      (tight coupling)
→ Adding invoice-service? Modify order-service.

WITH EDA:
order-service → publishes OrderPlaced event → Kafka
                                                  ↓
                              ┌───────────────────┼────────────────────┐
                              ↓                   ↓                    ↓
                    notification-service  analytics-service   loyalty-service
                    (sends confirmation)  (updates dashboard) (awards points)
→ Adding invoice-service? Just subscribe to OrderPlaced. Zero changes to order-service.
```

**Key EDA concepts:**

| Concept | Description |
|---|---|
| **Event** | A fact that happened — immutable, past tense: `OrderPlaced`, `FarmerRegistered`, `PaymentFailed` |
| **Event Producer** | The service that publishes the event |
| **Event Consumer** | The service that reacts to the event |
| **Event Bus / Broker** | Kafka, RabbitMQ — the transport layer |
| **Event Schema** | The structure of the event (use Avro / JSON Schema for contracts) |

**Event design best practices:**
```java
// Good event — contains enough data for consumers to act without extra calls
public class OrderPlacedEvent {
    private String orderId;
    private String customerId;
    private String customerEmail;    // notification-service needs this
    private List<OrderItem> items;   // inventory-service needs this
    private BigDecimal totalAmount;  // loyalty-service needs this
    private LocalDateTime placedAt;
}
```

**Choreography vs Orchestration:** See SAGA Pattern — same concepts apply here.

**When to use:** When adding new consumers should not require modifying the publisher. Audit logging. Decoupled workflows. Fan-out scenarios (one event, many consumers).

---

## 10. Database per Service Pattern

**Problem:** If all microservices share one database, you get tight coupling at the data layer — schema changes in one service break other services, one service can query another's private data, and you can't choose the best database technology per service.

**Solution:** Each microservice owns its own database. No other service can directly query it. Data access is only via the service's API.

```text
✗ WRONG — Shared Database:
farmer-service  ──→  [ Single MySQL DB ]  ←── crop-service
market-service  ──→                       ←── order-service
(all services share tables — tight coupling)

✓ CORRECT — Database per Service:
farmer-service  ──→ [ farmer_db : MySQL ]
crop-service    ──→ [ crop_db   : PostgreSQL ]
market-service  ──→ [ market_db : MongoDB ]
order-service   ──→ [ order_db  : MySQL ]
analytics-svc   ──→ [ analytics : Cassandra ]
```

**Benefits:**
- Each service can choose the best database for its data model (relational, document, time-series, graph)
- Schema changes in one service don't affect others
- Services can be deployed, scaled, and shut down independently
- No cross-service joins — enforces loose coupling

**The challenge — how do you join data across services?**

You can't do `SELECT * FROM farmers JOIN orders ON ...` across databases. Instead:

1. **API Composition** — call both services and join in application code
2. **Event-driven denormalisation** — when `farmer-service` publishes `FarmerRegistered`, `order-service` subscribes and stores a copy of the farmer name it needs locally
3. **CQRS read model** — maintain a denormalised read store fed by events from multiple services

**When to use:** Always in a true microservices architecture. Shared databases are one of the most common mistakes that defeat the purpose of microservices.

---

## 11. Retry Pattern

**Problem:** A network call to another service fails — but not because the service is down. It could be a transient glitch: a brief network hiccup, a GC pause, a momentary overload. Failing immediately and surfacing an error to the user is too aggressive for these cases.

**Solution:** Automatically retry the failed call a configured number of times before giving up.

```text
Attempt 1 → FAIL (network blip)  → wait 1s  →
Attempt 2 → FAIL (service busy)  → wait 2s  →
Attempt 3 → SUCCESS ✓
```

**Exponential Backoff:** Don't retry at fixed intervals — use exponential backoff with jitter to avoid a thundering herd (all retrying at the same time).

```text
Attempt 1 fails → wait 1s
Attempt 2 fails → wait 2s
Attempt 3 fails → wait 4s
Attempt 4 fails → wait 8s + random jitter
```

**Spring Boot (Resilience4j):**

```java
@Retry(name = "paymentService", fallbackMethod = "retryFallback")
public PaymentResponse processPayment(PaymentRequest request) {
    return paymentClient.process(request);
}

public PaymentResponse retryFallback(PaymentRequest request, Exception ex) {
    log.error("All retry attempts exhausted for payment: {}", ex.getMessage());
    return PaymentResponse.failed(request.getOrderId());
}
```

```yaml
resilience4j:
  retry:
    instances:
      paymentService:
        max-attempts: 3
        wait-duration: 1s
        enable-exponential-backoff: true
        exponential-backoff-multiplier: 2
        retry-exceptions:
          - java.net.ConnectException
          - java.util.concurrent.TimeoutException
        ignore-exceptions:
          - com.example.BadRequestException   # don't retry 4xx errors
```

**Critical rule — only retry idempotent operations:**

| Safe to retry | NOT safe to retry |
|---|---|
| GET requests | POST /payments (would double-charge) |
| Idempotent PUT | Any non-idempotent operation |
| Read-only queries | Sends that trigger side effects |

Make POST operations idempotent using an **idempotency key** (a unique request ID the client sends — the server ignores duplicate requests with the same key).

**Retry + Circuit Breaker together:**
- Retry handles **transient** failures (few bad calls)
- Circuit Breaker handles **sustained** failures (service is truly down)
- Use both: retry first, circuit breaker opens if retries keep failing

**When to use:** All synchronous inter-service calls. Always pair with Circuit Breaker and Timeout.

---

## 12. Configuration Externalization Pattern

**Problem:** Hardcoded configuration (database URLs, API keys, feature flags) inside service code or Docker images makes it impossible to change configuration without rebuilding and redeploying. Different environments (dev, staging, prod) need different values.

**Solution:** Store all configuration **outside the service** in a centralised config server or secrets manager. Services pull their config at startup.

```text
✗ WRONG:
application.properties hardcoded inside JAR
→ changing DB password = rebuild image = redeploy

✓ CORRECT:
Config Server / Vault / Kubernetes Secrets
→ Service fetches config on startup
→ changing DB password = update config server = services refresh (no redeploy)
```

**Spring Cloud Config Server:**

```java
// Config Server
@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication { }
```

```properties
# Config Server — points to a Git repo with all config files
spring.cloud.config.server.git.uri=https://github.com/your-org/agriconnect-config
```

```properties
# Each service pulls its config from the config server
spring.config.import=optional:configserver:http://localhost:8888
spring.application.name=farmer-service
spring.profiles.active=dev
```

The config server serves `farmer-service-dev.yml`, `farmer-service-prod.yml`, etc. from Git.

**Config refresh without restart (`@RefreshScope`):**

```java
@RestController
@RefreshScope   // beans are re-created on POST /actuator/refresh
public class FarmerController {

    @Value("${farmer.max-land-size}")
    private int maxLandSize;
}
```

```bash
# Trigger refresh across all instances via Spring Cloud Bus (Kafka/RabbitMQ)
POST /actuator/busrefresh
```

**Sensitive config — Secrets Management:**

Never store passwords in Git. Use:
- **HashiCorp Vault** — production-grade secrets engine
- **Kubernetes Secrets** — base64-encoded (not encrypted by default, needs Sealed Secrets or Vault)
- **AWS Secrets Manager / Parameter Store** — cloud-native

**Configuration hierarchy (Spring Boot resolves in this order):**

```text
1. Command-line arguments      (highest priority)
2. Environment variables
3. Config Server properties
4. application-{profile}.yml
5. application.yml             (lowest priority)
```

**When to use:** Always. No production system should have credentials, URLs, or feature flags baked into the deployed artifact.

---

## 13. Strangler Fig Pattern

**Problem:** You have a large legacy monolith. You can't rewrite it all at once (too risky, too expensive, too slow). But you need to migrate it to microservices.

**Solution:** Named after the strangler fig tree, which wraps around a host tree and slowly replaces it. Incrementally extract pieces of the monolith into new microservices, routing traffic to new services piece by piece, until the monolith is gone.

```text
Phase 1 — Start:
All traffic → Monolith

Phase 2 — Extract Farmer module:
/api/v1/farmers/** → farmer-service (new microservice)
Everything else   → Monolith (still running)

Phase 3 — Extract Crop module:
/api/v1/farmers/** → farmer-service
/api/v1/crops/**   → crop-service
Everything else   → Monolith

Phase N — Complete:
All traffic → Microservices
Monolith decommissioned
```

**How to implement:**

1. **Put a proxy/gateway in front of the monolith** (this is your API Gateway). It intercepts all traffic.
2. **Extract one bounded context** as a new microservice.
3. **Update the gateway routing** to send that path to the new service.
4. **Keep the monolith running** for everything else.
5. Repeat until the monolith has nothing left.

**The key rules:**
- Never break existing contracts (same API shape for the extracted service)
- Use the same database initially if needed — decouple the DB later (Database per Service Pattern)
- Route at the gateway level — clients see zero change

**AgriConnect context:** If your current company has a monolith ITSM system, the Strangler Fig is the safest way to migrate it. Extract one module (e.g., ticket-service) as a microservice first. If it fails, you roll back the gateway route — no downtime.

**When to use:** Legacy modernisation. Never do a big-bang rewrite. Always strangle incrementally.

---

## 14. Leader Election Pattern

**Problem:** In a horizontally scaled system, you have 5 instances of a service running. Some tasks must be performed by **exactly one** instance — e.g., running a scheduled job, being the primary writer in a cluster, processing a specific Kafka partition. Without coordination, all 5 instances might run the same job simultaneously, causing duplicate processing or race conditions.

**Solution:** Among all running instances, elect one as the **leader**. Only the leader performs the exclusive task. All other instances are followers and take over if the leader dies.

```text
Instance 1 (Leader) ← performs the scheduled job
Instance 2 (Follower) ← standby
Instance 3 (Follower) ← standby
Instance 4 (Follower) ← standby

If Instance 1 crashes → new election → Instance 3 becomes Leader
```

**Implementation approaches in Spring Boot:**

### Using ShedLock (Database-based — simplest)

Prevents a scheduled task from running simultaneously in multiple instances by acquiring a database lock.

```xml
<dependency>
    <groupId>net.javacrumbs.shedlock</groupId>
    <artifactId>shedlock-spring</artifactId>
</dependency>
<dependency>
    <groupId>net.javacrumbs.shedlock</groupId>
    <artifactId>shedlock-provider-jdbc-template</artifactId>
</dependency>
```

```java
@Component
public class MarketPriceUpdateJob {

    @Scheduled(cron = "0 0 * * * *")      // every hour
    @SchedulerLock(
        name = "marketPriceUpdateJob",
        lockAtLeastFor = "PT30M",          // hold lock for at least 30 min
        lockAtMostFor = "PT1H"             // release lock after 1 hour (crash safety)
    )
    public void updateMarketPrices() {
        // Only ONE instance runs this, even if 10 are deployed
        log.info("Running market price update on this instance only");
    }
}
```

```sql
-- ShedLock needs this table
CREATE TABLE shedlock (
    name        VARCHAR(64) NOT NULL,
    lock_until  TIMESTAMP   NOT NULL,
    locked_at   TIMESTAMP   NOT NULL,
    locked_by   VARCHAR(255) NOT NULL,
    PRIMARY KEY (name)
);
```

### Using Zookeeper / etcd / Redis (distributed lock — for leader-exclusive operations)

```java
@Service
public class LeaderAwareService {

    // Spring Integration provides LeaderInitiator backed by Zookeeper or Redis
    @EventListener
    public void onLeaderGranted(OnGrantedEvent event) {
        log.info("This instance is now the LEADER — starting exclusive tasks");
        startExclusiveTasks();
    }

    @EventListener
    public void onLeaderRevoked(OnRevokedEvent event) {
        log.info("Leadership revoked — stopping exclusive tasks");
        stopExclusiveTasks();
    }
}
```

**Kafka and Leader Election:** Kafka Consumer Groups already implement a form of leader election internally. The **Group Coordinator** assigns partitions to consumer instances, and each partition is consumed by exactly one instance in the group at a time.

**Tools:** ShedLock (DB-based), Apache Zookeeper, etcd (Kubernetes uses this), Redis (Redisson), Spring Integration Leader Election

**When to use:**
- Scheduled batch jobs in a multi-instance deployment
- Being a "primary" writer in a master-replica setup
- Any task that must run exactly once across a cluster

---

## Pattern Relationships at a Glance

```text
                    ┌─────────────────────────────────────────────────────────────┐
                    │                    Client                                   │
                    └──────────────────────────┬──────────────────────────────────┘
                                               │
                                   ┌───────────▼──────────┐
                                   │   API Gateway         │  ← Gateway Pattern
                                   │   (Routing, Auth,     │
                                   │    CORS, Rate Limit)  │
                                   └───────────┬───────────┘
                          ┌────────────────────┼───────────────────────┐
                          │                    │                       │
               ┌──────────▼──────┐  ┌──────────▼──────┐  ┌────────────▼────────┐
               │  farmer-service │  │  crop-service   │  │  market-service     │
               │  ┌──────────┐   │  │                 │  │                     │
               │  │ Sidecar  │   │  │ (own DB)        │  │ (own DB)            │
               │  │ (Envoy)  │   │  │                 │  │                     │
               │  └──────────┘   │  └─────────────────┘  └─────────────────────┘
               │  (own DB)       │         ↑ Database per Service Pattern
               └────────┬────────┘
                        │
               Circuit Breaker + Bulkhead + Retry
               (protect each inter-service call)
                        │
                   ┌────▼─────┐
                   │  Kafka   │  ← Event-Driven Architecture
                   └────┬─────┘
                        │
              ┌─────────┴──────────┐
              │                    │
        alert-service      analytics-service
```

---

## Quick Reference

| Pattern | Problem Solved | Key Tool (Spring) |
|---|---|---|
| Gateway | Single entry point, cross-cutting concerns | Spring Cloud Gateway |
| Service Registry | Dynamic service discovery | Eureka, Consul |
| Circuit Breaker | Prevent cascade failures | Resilience4j |
| SAGA | Distributed transactions | Axon, Kafka choreography |
| CQRS | Separate read/write models | Custom + Kafka |
| Bulkhead | Isolate thread pools per dependency | Resilience4j |
| Sidecar | Offload infra concerns to a proxy | Envoy / Istio |
| API Composition | Aggregate data from multiple services | CompletableFuture |
| Event-Driven | Decouple publishers from consumers | Apache Kafka |
| Database per Service | Loose coupling at data layer | Service-owned DBs |
| Retry | Handle transient failures | Resilience4j |
| Config Externalization | Environment-specific config without redeploy | Spring Cloud Config |
| Strangler Fig | Incremental monolith migration | API Gateway routing |
| Leader Election | Exactly-once task execution in a cluster | ShedLock, Zookeeper |

---

## The Golden Rules

1. **Use async (Kafka) when the caller doesn't need the result immediately** — decouples services and improves resilience
2. **Every synchronous call needs Circuit Breaker + Retry + Timeout + Bulkhead** — never make a naked HTTP call between services
3. **Never share a database between services** — the moment you do, it's a distributed monolith
4. **Config, secrets, and feature flags never live inside the artifact** — always externalise
5. **Add a gateway before you add your first microservice** — retrofitting it later is painful
6. **Prefer Strangler Fig over big-bang rewrites** — every big-bang rewrite in history has gone over time and budget
