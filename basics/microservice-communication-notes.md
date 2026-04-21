# Microservice Communication

## The Core Problem

In a monolith, services call each other as simple Java method calls:
```java
// Monolith — everything in the same JVM
OrderService orderService = new OrderService();
orderService.placeOrder(orderId);
```

In microservices, each service is a **separate process** on a **separate machine**. There are no shared method calls. Services must communicate over a **network**.

This introduces new challenges:
- What if the network is down?
- What if the other service is slow?
- What if the other service crashes mid-call?
- How do services find each other?

This is why microservice communication is one of the most important topics in distributed systems.

---

## Two Broad Categories

```text
Microservice Communication
        │
        ├── Synchronous  →  Caller waits for a response
        │                   (REST, gRPC, GraphQL)
        │
        └── Asynchronous →  Caller fires and moves on
                            (Kafka, RabbitMQ, ActiveMQ)
```

---

## 1. Synchronous Communication

The calling service **sends a request and waits** for the response before continuing.

```text
Order-Service  ──── HTTP POST /payment ───→  Payment-Service
               ←─────── 200 OK ────────────
(waits here)
```

### 1A. REST (HTTP/HTTPS)

The most common form. Services communicate using standard HTTP verbs and JSON.

**Tools in Spring Boot:**
- `RestTemplate` (older, blocking)
- `WebClient` (modern, reactive/non-blocking)
- `OpenFeign` (declarative, cleaner syntax)

#### Using RestTemplate
```java
@Service
public class OrderService {

    private final RestTemplate restTemplate;

    public OrderService(RestTemplate restTemplate) {
        this.restTemplate = restTemplate;
    }

    public PaymentResponse processPayment(PaymentRequest request) {
        String paymentServiceUrl = "http://payment-service/api/v1/payment";

        ResponseEntity<PaymentResponse> response = restTemplate.postForEntity(
            paymentServiceUrl,
            request,
            PaymentResponse.class
        );

        return response.getBody();
    }
}
```

#### Using OpenFeign (Declarative — cleanest approach)
```java
// Just declare the interface — Feign generates the implementation
@FeignClient(name = "payment-service", url = "http://localhost:8085")
public interface PaymentClient {

    @PostMapping("/api/v1/payment")
    PaymentResponse processPayment(@RequestBody PaymentRequest request);

    @GetMapping("/api/v1/payment/{id}/status")
    PaymentStatus getStatus(@PathVariable String id);
}

// Usage in OrderService — looks like a local method call
@Service
public class OrderService {

    private final PaymentClient paymentClient;

    public void placeOrder(Order order) {
        PaymentResponse response = paymentClient.processPayment(order.toPaymentRequest());
        // handle response
    }
}
```

#### Using WebClient (Reactive)
```java
@Service
public class OrderService {

    private final WebClient webClient;

    public OrderService(WebClient.Builder builder) {
        this.webClient = builder.baseUrl("http://payment-service").build();
    }

    public Mono<PaymentResponse> processPayment(PaymentRequest request) {
        return webClient.post()
            .uri("/api/v1/payment")
            .bodyValue(request)
            .retrieve()
            .bodyToMono(PaymentResponse.class);
    }
}
```

**When to use REST:**
- You need an immediate response to continue processing
- CRUD operations on a resource
- The dependent service is reliable and fast
- Example: `order-service` calls `inventory-service` to check stock before confirming an order

---

### 1B. gRPC (Google Remote Procedure Call)

A high-performance alternative to REST. Uses **Protocol Buffers (Protobuf)** instead of JSON, and HTTP/2 instead of HTTP/1.1.

```text
REST  → Text-based JSON   → Slower serialization
gRPC  → Binary Protobuf   → 5x–10x faster serialization
```

**How it works:**

1. Define your service contract in a `.proto` file:
```proto
syntax = "proto3";

service PaymentService {
    rpc ProcessPayment (PaymentRequest) returns (PaymentResponse);
}

message PaymentRequest {
    string order_id = 1;
    double amount = 2;
    string currency = 3;
}

message PaymentResponse {
    string transaction_id = 1;
    bool success = 2;
}
```

2. Generate Java stubs from the `.proto` file (via Maven/Gradle plugin)
3. Implement the service using the generated stubs

**When to use gRPC:**
- Internal service-to-service calls that need high throughput
- Real-time streaming (gRPC supports server-streaming, client-streaming, bidirectional streaming)
- Polyglot environments (the same `.proto` generates clients in Java, Go, Python, etc.)
- Example: a real-time analytics pipeline between services

**REST vs gRPC Quick Comparison:**

| Feature | REST | gRPC |
|---|---|---|
| Protocol | HTTP/1.1 | HTTP/2 |
| Data format | JSON (text) | Protobuf (binary) |
| Speed | Moderate | Fast (5x–10x) |
| Browser support | Native | Limited (needs grpc-web) |
| Contract | OpenAPI (optional) | .proto file (required) |
| Streaming | No (polling/WebSocket) | Yes (native) |
| Best for | Public APIs, CRUD | Internal high-throughput |

---

### 1C. GraphQL

A query language for APIs where the **client specifies exactly what data it needs**.

```graphql
# Client sends this query — asks only for name and email
query {
    farmer(id: "F001") {
        name
        email
        landDetails {
            area
            soilType
        }
    }
}
```

**When to use GraphQL:**
- When different clients (mobile, web) need different subsets of data
- Aggregating data from multiple services in one call (via a GraphQL gateway/BFF)
- Example: a mobile app that only needs `name + cropType` while a web dashboard needs the full farmer profile

---

## 2. Asynchronous Communication

The calling service **sends a message and does NOT wait**. It moves on immediately. The receiving service processes the message at its own pace.

```text
farmer-service  ──── farmer.registered event ───→  [ Kafka ]
    (moves on immediately)                                 ↓
                                                   alert-service
                                                   (processes later)
```

> This is exactly how `farmer-service` communicates with `alert-service` in AgriConnect.

---

### 2A. Apache Kafka

A **distributed event streaming platform**. Services publish events to topics, and other services consume those events.

**Key concepts:**

| Concept | Meaning |
|---|---|
| **Producer** | Service that publishes events |
| **Consumer** | Service that reads events |
| **Topic** | Named channel for events (e.g., `farmer.registered`) |
| **Partition** | A topic is split into partitions for parallel processing |
| **Offset** | Position of a message in a partition — Kafka tracks this per consumer group |
| **Consumer Group** | A group of consumers sharing the work of a topic |
| **Broker** | A Kafka server that stores and serves messages |

**Producer (farmer-service publishes event):**
```java
@Service
public class FarmerEventPublisher {

    private final KafkaTemplate<String, FarmerRegisteredEvent> kafkaTemplate;

    public void publishFarmerRegistered(Farmer farmer) {
        FarmerRegisteredEvent event = new FarmerRegisteredEvent(
            farmer.getId(),
            farmer.getName(),
            farmer.getEmail(),
            LocalDateTime.now()
        );

        kafkaTemplate.send("farmer.registered", event);
        log.info("Published farmer.registered event for farmerId: {}", farmer.getId());
    }
}
```

**Consumer (alert-service listens to event):**
```java
@Service
@Slf4j
public class FarmerRegisteredEventConsumer {

    @KafkaListener(
        topics = "farmer.registered",
        groupId = "alert-service-group"
    )
    public void handleFarmerRegistered(FarmerRegisteredEvent event) {
        log.info("Received farmer.registered event for: {}", event.getFarmerName());

        // Send welcome notification
        sendWelcomeNotification(event.getEmail(), event.getFarmerName());
    }
}
```

**Why Kafka is used here (and not REST):**

| Scenario | Why not REST? | Why Kafka? |
|---|---|---|
| farmer-service registers a farmer, alert-service sends welcome email | farmer-service doesn't need to wait for the email to be sent | farmer-service just fires the event and moves on |
| alert-service is down | REST call would fail — error propagates to farmer-service | Kafka stores the event — alert-service picks it up when it comes back up |
| 10,000 registrations spike | REST: alert-service gets overwhelmed | Kafka: events queue up, alert-service processes at its own pace |

---

### 2B. RabbitMQ

A traditional **message broker** based on the AMQP protocol. More focused on message routing than event streaming.

**Key concepts:**

| Concept | Kafka equivalent | Meaning |
|---|---|---|
| **Exchange** | (no direct equivalent) | Receives messages and routes them to queues based on routing rules |
| **Queue** | Partition | Stores messages for a consumer |
| **Binding** | Topic subscription | Links an exchange to a queue with a routing key |
| **Routing Key** | Topic name | Determines which queue gets the message |

**RabbitMQ Producer:**
```java
@Service
public class NotificationProducer {

    private final RabbitTemplate rabbitTemplate;

    public void sendNotification(NotificationMessage message) {
        rabbitTemplate.convertAndSend(
            "notifications.exchange",  // exchange
            "farmer.welcome",          // routing key
            message
        );
    }
}
```

**RabbitMQ Consumer:**
```java
@Service
public class NotificationConsumer {

    @RabbitListener(queues = "farmer.welcome.queue")
    public void handleWelcomeNotification(NotificationMessage message) {
        log.info("Processing notification for: {}", message.getEmail());
        // send email/SMS
    }
}
```

**Kafka vs RabbitMQ:**

| Feature | Kafka | RabbitMQ |
|---|---|---|
| Message storage | Stores messages on disk (persistent log) | Deletes messages after consumption (by default) |
| Replay | Yes — consumers can re-read past events | No — once consumed, gone |
| Throughput | Very high (millions/sec) | High (tens of thousands/sec) |
| Ordering | Guaranteed within a partition | Not guaranteed by default |
| Routing | Topic-based | Flexible (direct, fanout, topic, headers) |
| Best for | Event sourcing, audit logs, stream processing | Task queues, RPC-style messaging, complex routing |
| Used by | LinkedIn, Netflix, Uber | Instagram, Reddit, small-to-mid systems |

---

## 3. Service Discovery

In production, services don't run on hardcoded IPs. Instances scale up and down dynamically. **Service Discovery** lets services find each other automatically.

### The Problem
```text
# Hardcoded — works in dev, breaks in production
spring.cloud.gateway.routes[0].uri=http://localhost:8081

# What if farmer-service moves to a different IP?
# What if there are 5 instances of farmer-service?
```

### The Solution — Service Registry (Eureka)

```text
1. farmer-service starts → registers itself in Eureka ("I'm farmer-service at 192.168.1.10:8081")
2. api-gateway wants to call farmer-service → asks Eureka "where is farmer-service?"
3. Eureka responds with the IP/port
4. api-gateway calls the correct instance
```

**Enabling Eureka Client:**
```java
// In farmer-service main class
@SpringBootApplication
@EnableEurekaClient
public class FarmerServiceApplication { ... }
```

```properties
# application.properties
eureka.client.service-url.defaultZone=http://localhost:8761/eureka
spring.application.name=farmer-service
```

**Using service name instead of hardcoded URL in Gateway:**
```properties
# Before (hardcoded)
spring.cloud.gateway.routes[0].uri=http://localhost:8081

# After (Eureka-based — "lb://" means load-balanced)
spring.cloud.gateway.routes[0].uri=lb://farmer-service
```

---

## 4. Fault Tolerance in Service Communication

When services call each other synchronously, failures cascade. Service A calls Service B which calls Service C — if C fails, B fails, and A fails.

### Circuit Breaker Pattern (Resilience4j)

Like a real electrical circuit breaker — when too many failures occur, the circuit "opens" and stops making calls to the failing service, preventing cascade failures.

```text
States:
CLOSED (normal) → OPEN (failures detected) → HALF-OPEN (testing recovery)
```

```java
@FeignClient(name = "payment-service")
public interface PaymentClient {

    @GetMapping("/api/v1/payment/{id}/status")
    @CircuitBreaker(name = "paymentService", fallbackMethod = "getStatusFallback")
    PaymentStatus getStatus(@PathVariable String id);

    // Fallback executed when circuit is open
    default PaymentStatus getStatusFallback(String id, Exception ex) {
        return PaymentStatus.UNKNOWN;
    }
}
```

**Resilience patterns to know:**

| Pattern | Purpose |
|---|---|
| **Circuit Breaker** | Stop calling a failing service temporarily |
| **Retry** | Automatically retry a failed call N times |
| **Timeout** | Don't wait forever — fail fast after X milliseconds |
| **Bulkhead** | Limit concurrent calls to a service to prevent resource exhaustion |
| **Fallback** | Return a default response when the real call fails |

---

## 5. Synchronous vs Asynchronous — When to Use What?

| Scenario | Use | Reason |
|---|---|---|
| User registration → send welcome email | **Async (Kafka)** | User doesn't need to wait for email |
| Check inventory before confirming order | **Sync (REST/Feign)** | Order depends on stock availability |
| Payment processing | **Sync (REST/gRPC)** | User must know immediately if payment succeeded |
| Generate report and notify via email | **Async (Kafka)** | Report takes time, don't block the user |
| Fetch product details for a product page | **Sync (REST)** | Page can't render without the data |
| Log analytics / audit events | **Async (Kafka)** | Logging shouldn't slow down the main flow |
| Inter-service real-time data sync (high volume) | **gRPC** | High throughput, binary serialization |

---

## How AgriConnect Uses Both

```text
                            ┌─────────────────────────────────────┐
                            │            AgriConnect               │
                            │                                      │
  Client ──HTTP──→ api-gateway (8080)                              │
                       │                                           │
              ┌────────┼────────┐                                  │
              │        │        │                                  │
         [REST/HTTP] [REST/HTTP] [REST/HTTP]    ← Synchronous      │
              │        │        │                                  │
        farmer-svc  crop-svc  market-svc                           │
           (8081)   (8082)    (8083)                               │
              │                                                    │
              │──── farmer.registered event ──→ [Kafka]            │
                                                    │              │
                                           alert-service (8084)   │
                                           ← Asynchronous         │
                            └─────────────────────────────────────┘
```

- **Client → Gateway → Services**: Synchronous REST (client needs a response)
- **farmer-service → alert-service**: Asynchronous Kafka (farmer-service doesn't need to wait for the welcome email)

---

## Quick Summary

| Style | Protocol | Waiting? | Tools | Best For |
|---|---|---|---|---|
| Synchronous | REST (HTTP/JSON) | Yes | RestTemplate, WebClient, Feign | CRUD, immediate response needed |
| Synchronous | gRPC (HTTP2/Protobuf) | Yes | gRPC stubs | High-throughput internal calls |
| Synchronous | GraphQL | Yes | Spring GraphQL | Flexible queries, BFF |
| Asynchronous | Kafka | No | KafkaTemplate, @KafkaListener | Events, audit logs, decoupled workflows |
| Asynchronous | RabbitMQ | No | RabbitTemplate, @RabbitListener | Task queues, complex routing |

The golden rule:
> Use **synchronous** when the caller **needs the result to proceed**.
> Use **asynchronous** when the caller **doesn't care about the result** (or can handle it later).
