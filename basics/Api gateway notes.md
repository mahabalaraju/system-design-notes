# API Gateway

## What is an API Gateway?

An **API Gateway** is a server that acts as the **single entry point** for all client requests to your backend microservices.

In simple words:

> An API Gateway receives all incoming API calls, routes them to the appropriate microservice, and then returns the response back to the client.

It acts like a **front door / traffic controller / receptionist** for your entire microservices ecosystem.

## Simple Flow

```text
Client (Browser / Mobile / React / Angular)
                    ↓
           [ API Gateway ]         (Port 8080 in AgriConnect)
         ↙         ↓         ↘
farmer-service  crop-service  market-service
  (Port 8081)   (Port 8082)   (Port 8083)
```

**Real-world example** – When you open Amazon, you interact with one URL. Behind the scenes, an API Gateway routes your request to different microservices: product-service, cart-service, pricing-service, user-service — all hidden from the client.

---

## Why Do We Need an API Gateway?

**Without API Gateway:**
```text
Client → calls farmer-service  directly on :8081
Client → calls crop-service    directly on :8082
Client → calls market-service  directly on :8083
```

Problems:
- Client has to know all internal ports and IPs
- No central place for authentication, logging, or rate limiting
- CORS issues when calling from a browser frontend
- No security — internal services are exposed to the internet

**With API Gateway:**
```text
Client → calls only :8080 (gateway)
Gateway handles routing, auth, logging, CORS, rate limiting
```

---

## Core Responsibilities of an API Gateway

### 1. Request Routing
Routes requests to the correct microservice based on URL path.

```text
GET /api/v1/farmers/**  → farmer-service (Port 8081)
GET /api/v1/crops/**    → crop-service   (Port 8082)
GET /api/v1/market/**   → market-service (Port 8083)
```

> This is exactly what is configured in AgriConnect's `application.properties`.

---

### 2. Authentication & Authorization
The Gateway can validate JWT tokens or API keys **before** forwarding the request to any microservice.

```text
Client sends request → Gateway checks JWT → Valid? → Forward to service
                                          → Invalid? → Return 401 Unauthorized
```

This means individual microservices don't need to implement auth logic — the gateway handles it centrally.

---

### 3. Logging & Observability
The Gateway is the perfect place to log every request and response — since every call passes through it.

**AgriConnect's `LoggingFilter` does exactly this:**

```java
@Component
@Slf4j
public class LoggingFilter implements GlobalFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange,
            GatewayFilterChain chain) {

        ServerHttpRequest request = exchange.getRequest();
        long startTime = System.currentTimeMillis();

        log.info("Incoming Request | Method: {} | Path: {} | Time: {}",
                request.getMethod(),
                request.getPath(),
                LocalDateTime.now());

        return chain.filter(exchange).then(
                Mono.fromRunnable(() -> {
                    ServerHttpResponse response = exchange.getResponse();
                    long duration = System.currentTimeMillis() - startTime;

                    log.info("Outgoing Response | Path: {} | Status: {} | Duration: {}ms",
                            request.getPath(),
                            response.getStatusCode(),
                            duration);
                })
        );
    }

    @Override
    public int getOrder() {
        return -1; // Executes FIRST — before all other filters
    }
}
```

**What this logs:**
- HTTP Method (GET, POST, PUT, DELETE)
- Request Path
- Response Status Code (200, 404, 500)
- Total Duration in milliseconds

---

### 4. CORS (Cross-Origin Resource Sharing)
When a browser-based frontend (React on `localhost:3000`) calls the backend, the browser blocks the request by default unless the server explicitly allows it.

The API Gateway handles CORS centrally so individual microservices don't need to configure it.

**AgriConnect's `CorsConfig` allows:**

```java
corsConfig.setAllowedOrigins(List.of(
    "http://localhost:3000",   // React frontend
    "http://localhost:4200",   // Angular frontend
    "http://localhost:8080"    // Gateway itself
));

corsConfig.setAllowedMethods(Arrays.asList(
    "GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"
));

corsConfig.setAllowedHeaders(Arrays.asList(
    "Authorization", "Content-Type", "X-Requested-With", "Accept", "Origin"
));
```

---

### 5. Rate Limiting
Prevents abuse — e.g., a single user can make maximum 100 requests per minute.

```text
Client spams 1000 requests/sec → Gateway rate limiter → Allows only 100/min → Returns 429 Too Many Requests
```

---

### 6. Load Balancing
If a microservice has multiple instances running (for high availability), the gateway can distribute requests across them.

```text
GET /api/v1/farmers/**  → farmer-service Instance 1 (Port 8081)
                         → farmer-service Instance 2 (Port 8082)
                         → farmer-service Instance 3 (Port 8083)
```

---

### 7. SSL Termination
The gateway can handle HTTPS decryption, so internal microservices communicate over plain HTTP.

```text
Client (HTTPS) → Gateway decrypts → Internal services (HTTP)
```

---

## Spring Cloud Gateway — How It Works

Spring Cloud Gateway is built on top of **Project Reactor** (reactive/non-blocking I/O). This is why:
- It uses `Mono<Void>` instead of `void` in filters
- It uses `ServerHttpRequest` / `ServerHttpResponse` (reactive HTTP types, not Servlet)
- It runs on **Netty** (not Tomcat)

### Core Concepts

| Concept | Description |
|---|---|
| **Route** | A rule that maps a request (by path, header, method) to a destination URI |
| **Predicate** | The condition to match the request (e.g., `Path=/api/v1/farmers/**`) |
| **Filter** | Logic that runs before and/or after the request is forwarded |
| **GlobalFilter** | A filter that applies to ALL routes |
| **GatewayFilter** | A filter that applies to a specific route only |

---

## AgriConnect API Gateway Configuration

### Routing Configuration (`application.properties`)

```properties
spring.application.name=api-gateway
server.port=8080

# farmer-service route
spring.cloud.gateway.routes[0].id=farmer-service
spring.cloud.gateway.routes[0].uri=http://localhost:8081
spring.cloud.gateway.routes[0].predicates[0]=Path=/api/v1/farmers/**

# crop-service route
spring.cloud.gateway.routes[1].id=crop-service
spring.cloud.gateway.routes[1].uri=http://localhost:8082
spring.cloud.gateway.routes[1].predicates[0]=Path=/api/v1/crops/**

# market-service route
spring.cloud.gateway.routes[2].id=market-service
spring.cloud.gateway.routes[2].uri=http://localhost:8083
spring.cloud.gateway.routes[2].predicates[0]=Path=/api/v1/market/**
```

### How a Request Flows Through AgriConnect Gateway

```text
Step 1 → Client sends: GET http://localhost:8080/api/v1/farmers/123

Step 2 → LoggingFilter fires (order = -1):
           Logs: "Incoming Request | GET | /api/v1/farmers/123 | 2024-..."

Step 3 → Gateway checks routes:
           Does path match /api/v1/farmers/** ? → YES

Step 4 → Gateway forwards to:
           GET http://localhost:8081/api/v1/farmers/123

Step 5 → farmer-service processes and returns response

Step 6 → LoggingFilter fires again (response phase):
           Logs: "Outgoing Response | /api/v1/farmers/123 | 200 OK | 45ms"

Step 7 → Response returned to client
```

---

## Types of Filters in Spring Cloud Gateway

### GlobalFilter (applies to ALL routes)
```java
@Component
public class LoggingFilter implements GlobalFilter, Ordered {
    // runs for every single request regardless of path
}
```

### GatewayFilter (applies to SPECIFIC routes only)
Defined inline in route config:
```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: farmer-service
          uri: http://localhost:8081
          predicates:
            - Path=/api/v1/farmers/**
          filters:
            - AddRequestHeader=X-Service-Name, farmer-service
            - CircuitBreaker=name=farmerCB,fallbackUri=/fallback
```

### Common Built-in Filters

| Filter | Purpose |
|---|---|
| `AddRequestHeader` | Adds a header before forwarding |
| `AddResponseHeader` | Adds a header to the response |
| `RewritePath` | Rewrites the URL path |
| `CircuitBreaker` | Integrates with Resilience4j for fault tolerance |
| `RateLimiter` | Limits requests per second using Redis |
| `StripPrefix` | Removes prefix from path before forwarding |

---

## Filter Execution Order

```text
Request comes in
        ↓
Pre-filters (run in ascending order by getOrder())
        ↓
[Service processes request]
        ↓
Post-filters (run in descending order — like a stack)
        ↓
Response goes to client
```

**In AgriConnect:**
- `LoggingFilter` has `getOrder() = -1` — it runs **before** all default filters (negative = higher priority)

---

## Actuator Endpoints in API Gateway

The AgriConnect gateway exposes these management endpoints:

```properties
management.endpoints.web.exposure.include=health,info,gateway
management.endpoint.health.show-details=always
management.endpoint.gateway.enabled=true
```

| Endpoint | URL | Purpose |
|---|---|---|
| Health | `GET /actuator/health` | Check if gateway is UP |
| Info | `GET /actuator/info` | App metadata |
| Routes | `GET /actuator/gateway/routes` | Lists all configured routes |
| Refresh | `POST /actuator/gateway/refresh` | Reload routes without restart |

---

## API Gateway vs Load Balancer

| Feature | API Gateway | Load Balancer |
|---|---|---|
| Works at | Layer 7 (Application) | Layer 4 or 7 |
| Routing logic | URL path, headers, methods | IP, port, connection count |
| Authentication | Yes — can validate JWT | No |
| Logging | Yes — per-request logs | No |
| Rate limiting | Yes | No |
| CORS | Yes | No |
| Protocol translation | HTTP to gRPC, WebSocket, etc. | No |
| Distributes traffic | Can (client-side or via Eureka) | Primary purpose |
| Best for | Microservices entry point | Traffic distribution |

> An API Gateway **can** include load balancing (e.g., using Spring Cloud LoadBalancer + Eureka), but its primary role is **routing + cross-cutting concerns**.

---

## Common Tools Used in Production

| Tool | Type | Key Feature |
|---|---|---|
| **Spring Cloud Gateway** | Java/Reactive | Native Spring Boot integration, reactive non-blocking |
| **Kong** | Open Source | Plugin-based, very extensible, Lua scripting |
| **AWS API Gateway** | Cloud Managed | Fully managed, integrates with Lambda and IAM |
| **NGINX** | Software | Lightweight, can act as API Gateway + reverse proxy |
| **Traefik** | Software | Kubernetes-native, auto service discovery |
| **Zuul (Netflix)** | Java/Servlet | Older Netflix OSS, largely replaced by Spring Cloud Gateway |

---

## What to Add Next in AgriConnect

| Feature | How |
|---|---|
| JWT Authentication | Add a `GlobalFilter` that validates `Authorization: Bearer <token>` header |
| Rate Limiting | Add Redis + `RequestRateLimiter` GatewayFilter |
| Circuit Breaker | Add Resilience4j dependency + `CircuitBreaker` filter per route |
| Service Discovery | Replace hardcoded `http://localhost:8081` with `lb://farmer-service` using Eureka |

---

## Quick Summary

An **API Gateway** is the **single entry point** for all microservice traffic. It handles:

- routing requests to the correct service
- centralized authentication and authorization
- logging every request and response
- CORS for browser-based frontends
- rate limiting to prevent abuse
- load balancing across service instances
- SSL termination

In AgriConnect, the `api-gateway` module on Port 8080 routes traffic to `farmer-service`, `crop-service`, and `market-service`, and uses a `GlobalFilter` for logging and a `CorsWebFilter` for CORS — covering the most important production concerns.
