# gRPC — Complete Notes

> Advanced reference covering gRPC architecture, Protocol Buffers, service types, Java implementation, interceptors, error handling, and production patterns.

---

## Table of Contents

- [What is gRPC?](#what-is-grpc)
- [Protocol Buffers (Protobuf)](#protocol-buffers-protobuf)
  - [Syntax & Field Types](#syntax--field-types)
  - [Messages & Enums](#messages--enums)
  - [Nested, Oneof, Map](#nested-oneof-map)
  - [Well-Known Types](#well-known-types)
  - [Reserved Fields & Backward Compatibility](#reserved-fields--backward-compatibility)
- [gRPC Service Types](#grpc-service-types)
  - [Unary RPC](#unary-rpc)
  - [Server Streaming RPC](#server-streaming-rpc)
  - [Client Streaming RPC](#client-streaming-rpc)
  - [Bidirectional Streaming RPC](#bidirectional-streaming-rpc)
- [Java Setup](#java-setup)
- [Server Implementation](#server-implementation)
- [Client Implementation](#client-implementation)
  - [Blocking Stub](#blocking-stub)
  - [Async Stub](#async-stub)
  - [Future Stub](#future-stub)
- [Channels & Connection Management](#channels--connection-management)
- [Interceptors](#interceptors)
  - [Server Interceptor](#server-interceptor)
  - [Client Interceptor](#client-interceptor)
- [Metadata (Headers)](#metadata-headers)
- [Error Handling](#error-handling)
  - [Status Codes](#status-codes)
  - [Rich Error Model](#rich-error-model)
- [Deadlines & Timeouts](#deadlines--timeouts)
- [Authentication & TLS](#authentication--tls)
- [Load Balancing](#load-balancing)
- [Health Checking](#health-checking)
- [Reflection](#reflection)
- [gRPC vs REST](#grpc-vs-rest)
- [gRPC with Spring Boot](#grpc-with-spring-boot)
- [Testing](#testing)
- [Performance Tuning](#performance-tuning)
- [Common Pitfalls](#common-pitfalls)
- [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)

---

## What is gRPC?

gRPC (Google Remote Procedure Call) is a high-performance, open-source RPC framework built on:
- **HTTP/2** — multiplexing, header compression, bidirectional streaming
- **Protocol Buffers** — binary serialization (default), strongly typed, ~3–10× smaller than JSON
- **Code generation** — `.proto` files generate client/server stubs in 10+ languages

```
Client App                          Server App
┌──────────┐   HTTP/2 + Protobuf   ┌──────────┐
│  Stub    │ ────────────────────► │ Service  │
│ (generated) ◄──────────────────  │ (impl)   │
└──────────┘                       └──────────┘
```

**Why gRPC over REST:**
- Binary protocol — smaller payloads, faster serialization
- Strongly typed contracts via `.proto` — no schema drift
- Bidirectional streaming — impossible with REST
- Auto-generated clients in any language
- Built-in deadline propagation, cancellation, load balancing

---

## Protocol Buffers (Protobuf)

### Syntax & Field Types

```protobuf
syntax = "proto3";

package com.example.user;

option java_package = "com.example.user.proto";
option java_multiple_files = true;         // one class per message
option java_outer_classname = "UserProto"; // wrapper class name
```

**Scalar types:**

| Proto type | Java type | Notes |
|---|---|---|
| `double` | `double` | 64-bit float |
| `float` | `float` | 32-bit float |
| `int32` | `int` | Inefficient for negatives — use sint32 |
| `int64` | `long` | Inefficient for negatives — use sint64 |
| `uint32` | `int` | Unsigned (Java has no unsigned) |
| `uint64` | `long` | Unsigned |
| `sint32` | `int` | Signed, efficient for negatives (zigzag) |
| `sint64` | `long` | Signed, efficient for negatives |
| `fixed32` | `int` | Always 4 bytes — better for values > 2²⁸ |
| `fixed64` | `long` | Always 8 bytes |
| `bool` | `boolean` | |
| `string` | `String` | UTF-8 |
| `bytes` | `ByteString` | Arbitrary byte sequence |

---

### Messages & Enums

```protobuf
syntax = "proto3";

package com.example.order;

option java_package = "com.example.order.proto";
option java_multiple_files = true;

import "google/protobuf/timestamp.proto";
import "google/protobuf/wrappers.proto";

enum OrderStatus {
  ORDER_STATUS_UNSPECIFIED = 0; // always define 0 as default/unknown
  ORDER_STATUS_PENDING     = 1;
  ORDER_STATUS_CONFIRMED   = 2;
  ORDER_STATUS_SHIPPED     = 3;
  ORDER_STATUS_DELIVERED   = 4;
  ORDER_STATUS_CANCELLED   = 5;
}

message Address {
  string street  = 1;
  string city    = 2;
  string country = 3;
  string pin     = 4;
}

message OrderItem {
  string product_id = 1;
  string name       = 2;
  int32  quantity   = 3;
  double unit_price = 4;
}

message Order {
  string                    order_id        = 1;
  string                    user_id         = 2;
  repeated OrderItem        items           = 3; // list
  OrderStatus               status          = 4;
  Address                   shipping_address= 5;
  google.protobuf.Timestamp created_at      = 6;
  google.protobuf.Timestamp updated_at      = 7;
  google.protobuf.DoubleValue discount      = 8; // nullable double
}
```

> **Field numbers:**
> - 1–15 use 1 byte in encoding — use for frequently populated fields
> - 16–2047 use 2 bytes — use for less common fields
> - Never reuse field numbers — causes silent data corruption

---

### Nested, Oneof, Map

```protobuf
message PaymentRequest {
  string order_id = 1;
  double amount   = 2;

  // oneof — exactly one of these fields can be set at a time
  oneof payment_method {
    CreditCard  credit_card = 3;
    UpiPayment  upi         = 4;
    NetBanking  net_banking = 5;
  }

  // map — key-value pairs
  map<string, string> metadata = 6;

  // nested message
  message CreditCard {
    string number   = 1;
    string expiry   = 2;
    string cvv      = 3;
  }

  message UpiPayment {
    string upi_id = 1;
  }

  message NetBanking {
    string bank_code   = 1;
    string account_no  = 2;
  }
}
```

---

### Well-Known Types

```protobuf
import "google/protobuf/timestamp.proto";   // Timestamp
import "google/protobuf/duration.proto";    // Duration
import "google/protobuf/wrappers.proto";    // nullable scalars
import "google/protobuf/empty.proto";       // Empty (void)
import "google/protobuf/any.proto";         // Any type
import "google/protobuf/struct.proto";      // dynamic JSON-like struct
import "google/protobuf/field_mask.proto";  // partial update masks
```

```protobuf
message UpdateUserRequest {
  string                    user_id    = 1;
  string                    name       = 2;
  google.protobuf.StringValue bio       = 3;  // nullable String
  google.protobuf.Timestamp  updated_at = 4;
  google.protobuf.FieldMask  update_mask= 5;  // which fields to update
}

// Empty response (like void)
rpc DeleteUser(DeleteUserRequest) returns (google.protobuf.Empty);
```

**Java usage of Timestamp:**

```java
// Proto → Java
Timestamp ts = order.getCreatedAt();
Instant instant = Instant.ofEpochSecond(ts.getSeconds(), ts.getNanos());

// Java → Proto
Instant now = Instant.now();
Timestamp ts = Timestamp.newBuilder()
    .setSeconds(now.getEpochSecond())
    .setNanos(now.getNano())
    .build();
```

---

### Reserved Fields & Backward Compatibility

```protobuf
message User {
  reserved 2, 15, 9 to 11;          // reserve field numbers — can't be reused
  reserved "old_name", "legacy_id"; // reserve field names

  string id    = 1;
  string email = 3;
  // field 2 was "phone" — deleted but reserved to prevent reuse
}
```

**Backward compatibility rules:**
- ✅ Adding new fields (existing clients ignore unknown fields)
- ✅ Removing a field (add to `reserved` list)
- ✅ Renaming a field (field number is what matters in wire format)
- ❌ Changing a field's type
- ❌ Changing a field's number
- ❌ Changing enum values' numbers

---

## gRPC Service Types

### Service Definition (`.proto`)

```protobuf
syntax = "proto3";

package com.example.order;

import "google/protobuf/empty.proto";

service OrderService {
  // Unary
  rpc GetOrder       (GetOrderRequest)    returns (Order);
  rpc CreateOrder    (CreateOrderRequest) returns (Order);

  // Server streaming
  rpc ListOrders     (ListOrdersRequest)  returns (stream Order);

  // Client streaming
  rpc BulkCreateOrders (stream CreateOrderRequest) returns (BulkCreateResponse);

  // Bidirectional streaming
  rpc TrackOrders    (stream TrackRequest) returns (stream OrderStatusUpdate);
}
```

---

### Unary RPC

Standard request-response. One request, one response.

```
Client                    Server
  │── GetOrder(req) ──────►│
  │◄── Order(res) ─────────│
```

```java
// Server
@Override
public void getOrder(GetOrderRequest request, StreamObserver<Order> responseObserver) {
    try {
        Order order = orderService.findById(request.getOrderId());
        responseObserver.onNext(order);     // send response
        responseObserver.onCompleted();     // signal done
    } catch (OrderNotFoundException e) {
        responseObserver.onError(
            Status.NOT_FOUND
                .withDescription("Order not found: " + request.getOrderId())
                .asRuntimeException()
        );
    }
}

// Client (blocking stub)
Order order = stub.getOrder(
    GetOrderRequest.newBuilder().setOrderId("ORD-123").build()
);
```

---

### Server Streaming RPC

One request, stream of responses. Server sends multiple messages.

```
Client                    Server
  │── ListOrders(req) ────►│
  │◄── Order(1) ───────────│
  │◄── Order(2) ───────────│
  │◄── Order(3) ───────────│
  │◄── onCompleted() ──────│
```

```java
// Server
@Override
public void listOrders(ListOrdersRequest request,
                       StreamObserver<Order> responseObserver) {
    try {
        orderService.streamByUserId(request.getUserId())
            .forEach(order -> {
                responseObserver.onNext(order); // send each order
                // check cancellation for long streams
                if (Context.current().isCancelled()) return;
            });
        responseObserver.onCompleted();
    } catch (Exception e) {
        responseObserver.onError(Status.INTERNAL.withCause(e).asRuntimeException());
    }
}

// Client (async stub) — receives stream
stub.listOrders(request, new StreamObserver<Order>() {
    public void onNext(Order order)    { process(order); }
    public void onError(Throwable t)   { log.error("Stream error", t); }
    public void onCompleted()          { log.info("Stream complete"); }
});
```

---

### Client Streaming RPC

Stream of requests, one response. Client sends multiple messages, server responds once.

```
Client                    Server
  │── CreateOrder(1) ─────►│
  │── CreateOrder(2) ─────►│
  │── CreateOrder(3) ─────►│
  │── onCompleted() ───────►│
  │◄── BulkCreateResponse ─│
```

```java
// Server
@Override
public StreamObserver<CreateOrderRequest> bulkCreateOrders(
        StreamObserver<BulkCreateResponse> responseObserver) {

    List<Order> created = new ArrayList<>();

    return new StreamObserver<CreateOrderRequest>() {
        public void onNext(CreateOrderRequest req) {
            created.add(orderService.create(req)); // accumulate
        }

        public void onError(Throwable t) {
            log.error("Client stream error", t);
        }

        public void onCompleted() {
            responseObserver.onNext(BulkCreateResponse.newBuilder()
                .setCreatedCount(created.size())
                .build());
            responseObserver.onCompleted();
        }
    };
}

// Client (async stub)
StreamObserver<BulkCreateResponse> responseObserver = new StreamObserver<>() {
    public void onNext(BulkCreateResponse res) { log.info("Created: {}", res.getCreatedCount()); }
    public void onError(Throwable t)           { log.error("Error", t); }
    public void onCompleted()                  { log.info("Done"); }
};

StreamObserver<CreateOrderRequest> requestObserver =
    asyncStub.bulkCreateOrders(responseObserver);

orders.forEach(o -> requestObserver.onNext(buildRequest(o)));
requestObserver.onCompleted(); // signal end of client stream
```

---

### Bidirectional Streaming RPC

Both client and server send streams independently. Full duplex.

```
Client                    Server
  │── TrackRequest(A) ────►│
  │◄── StatusUpdate(A) ────│
  │── TrackRequest(B) ────►│
  │◄── StatusUpdate(B) ────│
  │◄── StatusUpdate(A) ────│  (server can push anytime)
```

```java
// Server
@Override
public StreamObserver<TrackRequest> trackOrders(
        StreamObserver<OrderStatusUpdate> responseObserver) {

    return new StreamObserver<TrackRequest>() {
        public void onNext(TrackRequest req) {
            // Subscribe to order updates and push to client
            orderTracker.subscribe(req.getOrderId(), status ->
                responseObserver.onNext(OrderStatusUpdate.newBuilder()
                    .setOrderId(req.getOrderId())
                    .setStatus(status)
                    .build())
            );
        }

        public void onError(Throwable t)  { orderTracker.unsubscribeAll(); }
        public void onCompleted()         { responseObserver.onCompleted(); }
    };
}
```

---

## Java Setup

```xml
<!-- pom.xml -->
<properties>
    <grpc.version>1.62.2</grpc.version>
    <protobuf.version>3.25.3</protobuf.version>
</properties>

<dependencies>
    <dependency>
        <groupId>io.grpc</groupId>
        <artifactId>grpc-netty-shaded</artifactId>
        <version>${grpc.version}</version>
    </dependency>
    <dependency>
        <groupId>io.grpc</groupId>
        <artifactId>grpc-protobuf</artifactId>
        <version>${grpc.version}</version>
    </dependency>
    <dependency>
        <groupId>io.grpc</groupId>
        <artifactId>grpc-stub</artifactId>
        <version>${grpc.version}</version>
    </dependency>
    <dependency>
        <groupId>io.grpc</groupId>
        <artifactId>grpc-services</artifactId>  <!-- health check, reflection -->
        <version>${grpc.version}</version>
    </dependency>
</dependencies>

<build>
    <extensions>
        <extension>
            <groupId>kr.motd.maven</groupId>
            <artifactId>os-maven-plugin</artifactId>
            <version>1.7.1</version>
        </extension>
    </extensions>
    <plugins>
        <plugin>
            <groupId>org.xolstice.maven.plugins</groupId>
            <artifactId>protobuf-maven-plugin</artifactId>
            <version>0.6.1</version>
            <configuration>
                <protocArtifact>com.google.protobuf:protoc:${protobuf.version}:exe:${os.detected.classifier}</protocArtifact>
                <pluginId>grpc-java</pluginId>
                <pluginArtifact>io.grpc:protoc-gen-grpc-java:${grpc.version}:exe:${os.detected.classifier}</pluginArtifact>
            </configuration>
            <executions>
                <execution>
                    <goals>
                        <goal>compile</goal>
                        <goal>compile-custom</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

Proto files go in `src/main/proto/`. Generated code lands in `target/generated-sources/`.

---

## Server Implementation

```java
public class OrderGrpcService extends OrderServiceGrpc.OrderServiceImplBase {

    private final OrderRepository orderRepository;

    public OrderGrpcService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    @Override
    public void getOrder(GetOrderRequest request,
                         StreamObserver<Order> responseObserver) {
        try {
            Order order = orderRepository.findById(request.getOrderId())
                .orElseThrow(() -> new OrderNotFoundException(request.getOrderId()));

            responseObserver.onNext(toProto(order));
            responseObserver.onCompleted();
        } catch (OrderNotFoundException e) {
            responseObserver.onError(
                Status.NOT_FOUND
                    .withDescription(e.getMessage())
                    .asRuntimeException()
            );
        } catch (Exception e) {
            responseObserver.onError(
                Status.INTERNAL
                    .withDescription("Internal server error")
                    .withCause(e)
                    .asRuntimeException()
            );
        }
    }
}

// Starting the server
public class GrpcServer {
    public static void main(String[] args) throws Exception {
        Server server = ServerBuilder.forPort(9090)
            .addService(new OrderGrpcService(orderRepository))
            .addService(new UserGrpcService(userRepository))
            .addService(ProtoReflectionService.newInstance())   // reflection
            .addService(HealthStatusManager.getHealthService()) // health check
            .intercept(new LoggingInterceptor())
            .intercept(new AuthInterceptor())
            .maxInboundMessageSize(10 * 1024 * 1024)  // 10MB
            .build()
            .start();

        Runtime.getRuntime().addShutdownHook(new Thread(() -> {
            server.shutdown();
            try { server.awaitTermination(30, TimeUnit.SECONDS); }
            catch (InterruptedException e) { Thread.currentThread().interrupt(); }
        }));

        server.awaitTermination(); // block main thread
    }
}
```

---

## Client Implementation

### Blocking Stub

Simplest — blocks the calling thread until response received.

```java
ManagedChannel channel = ManagedChannelBuilder
    .forAddress("localhost", 9090)
    .usePlaintext() // no TLS — dev only
    .build();

OrderServiceGrpc.OrderServiceBlockingStub stub =
    OrderServiceGrpc.newBlockingStub(channel)
        .withDeadlineAfter(5, TimeUnit.SECONDS); // always set deadline

try {
    Order order = stub.getOrder(
        GetOrderRequest.newBuilder().setOrderId("ORD-123").build()
    );
    System.out.println(order);

    // Server streaming — returns Iterator
    Iterator<Order> orders = stub.listOrders(
        ListOrdersRequest.newBuilder().setUserId("USR-1").build()
    );
    while (orders.hasNext()) {
        process(orders.next());
    }
} catch (StatusRuntimeException e) {
    Status status = e.getStatus();
    if (status.getCode() == Status.Code.NOT_FOUND) {
        // handle not found
    }
} finally {
    channel.shutdown().awaitTermination(5, TimeUnit.SECONDS);
}
```

---

### Async Stub

Non-blocking — uses `StreamObserver` callbacks.

```java
OrderServiceGrpc.OrderServiceStub asyncStub =
    OrderServiceGrpc.newStub(channel)
        .withDeadlineAfter(10, TimeUnit.SECONDS);

asyncStub.getOrder(
    GetOrderRequest.newBuilder().setOrderId("ORD-123").build(),
    new StreamObserver<Order>() {
        public void onNext(Order order)  { process(order); }
        public void onError(Throwable t) { handleError(t); }
        public void onCompleted()        { log.info("done"); }
    }
);
```

---

### Future Stub

Returns `ListenableFuture` — integrates with Guava futures / `CompletableFuture`.

```java
OrderServiceGrpc.OrderServiceFutureStub futureStub =
    OrderServiceGrpc.newFutureStub(channel);

ListenableFuture<Order> future = futureStub.getOrder(
    GetOrderRequest.newBuilder().setOrderId("ORD-123").build()
);

// Convert to CompletableFuture
CompletableFuture<Order> cf = new CompletableFuture<>();
Futures.addCallback(future, new FutureCallback<Order>() {
    public void onSuccess(Order result)   { cf.complete(result); }
    public void onFailure(Throwable t)    { cf.completeExceptionally(t); }
}, executor);
```

---

## Channels & Connection Management

```java
ManagedChannel channel = ManagedChannelBuilder
    .forAddress("order-service", 9090)
    .useTransportSecurity()                     // TLS
    .keepAliveTime(30, TimeUnit.SECONDS)        // ping every 30s
    .keepAliveTimeout(10, TimeUnit.SECONDS)     // wait 10s for pong
    .keepAliveWithoutCalls(true)                // ping even when idle
    .maxInboundMessageSize(4 * 1024 * 1024)    // 4MB response limit
    .maxRetryAttempts(3)                        // retry on transient failures
    .enableRetry()
    .build();

// Channel is thread-safe — reuse across threads, create once
// Stub is lightweight — can create per-call with deadline

// Graceful shutdown
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    try {
        channel.shutdown().awaitTermination(10, TimeUnit.SECONDS);
    } catch (InterruptedException e) {
        channel.shutdownNow();
    }
}));
```

**Channel reuse:** Create one `ManagedChannel` per server, share across the application. Channels are expensive to create (TCP + TLS handshake). Stubs are cheap wrappers — create per-call or per-request.

---

## Interceptors

### Server Interceptor

Runs for every incoming RPC on the server side.

```java
public class LoggingServerInterceptor implements ServerInterceptor {

    private static final Logger log = LoggerFactory.getLogger(LoggingServerInterceptor.class);

    @Override
    public <ReqT, RespT> ServerCall.Listener<ReqT> interceptCall(
            ServerCall<ReqT, RespT> call,
            Metadata headers,
            ServerCallHandler<ReqT, RespT> next) {

        String method = call.getMethodDescriptor().getFullMethodName();
        long start = System.currentTimeMillis();

        log.info("gRPC call: {}", method);

        ServerCall<ReqT, RespT> wrappedCall = new ForwardingServerCall
                .SimpleForwardingServerCall<>(call) {
            @Override
            public void close(Status status, Metadata trailers) {
                log.info("{} completed in {}ms status={}",
                    method, System.currentTimeMillis() - start, status.getCode());
                super.close(status, trailers);
            }
        };

        return next.startCall(wrappedCall, headers);
    }
}

// Auth interceptor
public class AuthServerInterceptor implements ServerInterceptor {

    static final Context.Key<String> USER_ID_KEY = Context.key("userId");

    private static final Metadata.Key<String> AUTH_TOKEN_KEY =
        Metadata.Key.of("authorization", Metadata.ASCII_STRING_MARSHALLER);

    @Override
    public <ReqT, RespT> ServerCall.Listener<ReqT> interceptCall(
            ServerCall<ReqT, RespT> call,
            Metadata headers,
            ServerCallHandler<ReqT, RespT> next) {

        String token = headers.get(AUTH_TOKEN_KEY);
        if (token == null || !tokenService.isValid(token)) {
            call.close(Status.UNAUTHENTICATED.withDescription("Invalid token"), new Metadata());
            return new ServerCall.Listener<>() {}; // no-op listener
        }

        String userId = tokenService.extractUserId(token);
        Context ctx = Context.current().withValue(USER_ID_KEY, userId);
        return Contexts.interceptCall(ctx, call, headers, next);
    }
}

// Access in service
String userId = AuthServerInterceptor.USER_ID_KEY.get(); // from Context
```

---

### Client Interceptor

Runs for every outgoing RPC on the client side.

```java
public class HeaderClientInterceptor implements ClientInterceptor {

    private static final Metadata.Key<String> CORRELATION_ID_KEY =
        Metadata.Key.of("x-correlation-id", Metadata.ASCII_STRING_MARSHALLER);

    @Override
    public <ReqT, RespT> ClientCall<ReqT, RespT> interceptCall(
            MethodDescriptor<ReqT, RespT> method,
            CallOptions callOptions,
            Channel next) {

        return new ForwardingClientCall.SimpleForwardingClientCall<>(
                next.newCall(method, callOptions)) {

            @Override
            public void start(Listener<RespT> responseListener, Metadata headers) {
                headers.put(CORRELATION_ID_KEY, UUID.randomUUID().toString());
                headers.put(AUTH_TOKEN_KEY, tokenProvider.getToken());
                super.start(responseListener, headers);
            }
        };
    }
}

// Attach to channel
ManagedChannel channel = ManagedChannelBuilder
    .forAddress("localhost", 9090)
    .intercept(new HeaderClientInterceptor())
    .build();
```

---

## Metadata (Headers)

```java
// Binary header — use -bin suffix
Metadata.Key<byte[]> BINARY_KEY =
    Metadata.Key.of("data-bin", Metadata.BINARY_BYTE_MARSHALLER);

// ASCII header
Metadata.Key<String> TENANT_KEY =
    Metadata.Key.of("x-tenant-id", Metadata.ASCII_STRING_MARSHALLER);

// Sending from client
Metadata headers = new Metadata();
headers.put(TENANT_KEY, "tenant-abc");

stub = MetadataUtils.attachHeaders(stub, headers);
// or per-call:
stub.withInterceptors(MetadataUtils.newAttachHeadersInterceptor(headers));

// Reading on server (in interceptor or service via Context)
String tenantId = headers.get(TENANT_KEY);

// Sending trailing metadata from server (response trailers)
Metadata trailers = new Metadata();
trailers.put(RATE_LIMIT_REMAINING_KEY, "99");
responseObserver.onError(
    Status.RESOURCE_EXHAUSTED
        .withDescription("Rate limit exceeded")
        .asRuntimeException(trailers)
);
```

---

## Error Handling

### Status Codes

```java
// Server — send errors
responseObserver.onError(Status.NOT_FOUND
    .withDescription("Order ORD-123 not found")
    .withCause(originalException)       // not sent to client — for server logging only
    .asRuntimeException());

// Client — handle errors
try {
    Order order = stub.getOrder(request);
} catch (StatusRuntimeException e) {
    switch (e.getStatus().getCode()) {
        case NOT_FOUND         -> handleNotFound();
        case INVALID_ARGUMENT  -> handleBadRequest(e.getStatus().getDescription());
        case UNAUTHENTICATED   -> refreshTokenAndRetry();
        case UNAVAILABLE       -> retryWithBackoff();
        case DEADLINE_EXCEEDED -> handleTimeout();
        default                -> handleUnexpected(e);
    }
}
```

**Standard Status Codes:**

| Code | HTTP equiv | Use case |
|---|---|---|
| `OK` | 200 | Success |
| `CANCELLED` | 499 | Client cancelled the request |
| `INVALID_ARGUMENT` | 400 | Bad request parameters |
| `NOT_FOUND` | 404 | Resource not found |
| `ALREADY_EXISTS` | 409 | Resource already exists |
| `PERMISSION_DENIED` | 403 | Authenticated but not authorized |
| `UNAUTHENTICATED` | 401 | Missing or invalid credentials |
| `RESOURCE_EXHAUSTED` | 429 | Rate limit or quota exceeded |
| `FAILED_PRECONDITION` | 400 | System not in required state |
| `ABORTED` | 409 | Concurrency conflict (retry the op) |
| `DEADLINE_EXCEEDED` | 504 | Deadline expired before completion |
| `UNIMPLEMENTED` | 501 | Method not implemented |
| `INTERNAL` | 500 | Internal server error |
| `UNAVAILABLE` | 503 | Service down, client should retry |
| `DATA_LOSS` | 500 | Unrecoverable data loss |

**Retryable codes:** `UNAVAILABLE`, `RESOURCE_EXHAUSTED` (sometimes), `DEADLINE_EXCEEDED` (if idempotent).

---

### Rich Error Model

For richer errors than a status code + description, use `google.rpc.Status` with `details`.

```xml
<dependency>
    <groupId>com.google.api.grpc</groupId>
    <artifactId>grpc-google-common-protos</artifactId>
    <version>2.37.1</version>
</dependency>
```

```java
// Server — send rich error
import com.google.rpc.BadRequest;
import io.grpc.protobuf.StatusProto;

BadRequest badRequest = BadRequest.newBuilder()
    .addFieldViolations(BadRequest.FieldViolation.newBuilder()
        .setField("quantity")
        .setDescription("must be greater than 0")
        .build())
    .addFieldViolations(BadRequest.FieldViolation.newBuilder()
        .setField("product_id")
        .setDescription("product not found")
        .build())
    .build();

com.google.rpc.Status status = com.google.rpc.Status.newBuilder()
    .setCode(com.google.rpc.Code.INVALID_ARGUMENT_VALUE)
    .setMessage("Validation failed")
    .addDetails(Any.pack(badRequest))
    .build();

responseObserver.onError(StatusProto.toStatusRuntimeException(status));

// Client — extract rich error
StatusRuntimeException e = ...; // caught
com.google.rpc.Status status = StatusProto.fromThrowable(e);
if (status != null) {
    for (Any detail : status.getDetailsList()) {
        if (detail.is(BadRequest.class)) {
            BadRequest br = detail.unpack(BadRequest.class);
            br.getFieldViolationsList().forEach(v ->
                log.error("Field: {} Error: {}", v.getField(), v.getDescription()));
        }
    }
}
```

---

## Deadlines & Timeouts

```java
// Always set deadlines on client calls — no default deadline in gRPC!

// Per-stub deadline
OrderServiceGrpc.OrderServiceBlockingStub stub =
    OrderServiceGrpc.newBlockingStub(channel)
        .withDeadlineAfter(5, TimeUnit.SECONDS);

// Per-call deadline
stub.withDeadline(Deadline.after(3, TimeUnit.SECONDS)).getOrder(request);

// Absolute deadline (propagate from upstream)
Deadline deadline = Context.current().getDeadline();
if (deadline != null) {
    downstreamStub = downstreamStub.withDeadline(deadline); // propagate!
}

// Server — check if client cancelled
@Override
public void longRunningOperation(Request req, StreamObserver<Response> obs) {
    for (Item item : getLargeDataset()) {
        if (Context.current().isCancelled()) {
            obs.onError(Status.CANCELLED
                .withDescription("Client cancelled")
                .asRuntimeException());
            return;
        }
        obs.onNext(process(item));
    }
    obs.onCompleted();
}
```

> **Deadline vs Timeout:** A `Deadline` is an absolute point in time. A `Timeout` is a duration. gRPC uses `Deadline` internally — set once at the top of the call chain and propagate down to avoid total deadline exceeding the intent.

---

## Authentication & TLS

```java
// Server — TLS
Server server = NettyServerBuilder.forPort(443)
    .sslContext(GrpcSslContexts.forServer(certChainFile, privateKeyFile).build())
    .addService(new OrderGrpcService())
    .build();

// Client — TLS with custom cert
ManagedChannel channel = NettyChannelBuilder
    .forAddress("order-service", 443)
    .sslContext(GrpcSslContexts.forClient()
        .trustManager(caCertFile)
        .build())
    .build();

// Mutual TLS (mTLS)
ManagedChannel mtlsChannel = NettyChannelBuilder
    .forAddress("order-service", 443)
    .sslContext(GrpcSslContexts.forClient()
        .trustManager(caCertFile)
        .keyManager(clientCertFile, clientKeyFile)
        .build())
    .build();

// JWT / Token authentication via metadata
Metadata headers = new Metadata();
headers.put(
    Metadata.Key.of("authorization", Metadata.ASCII_STRING_MARSHALLER),
    "Bearer " + jwtToken
);
OrderServiceGrpc.OrderServiceBlockingStub stub =
    OrderServiceGrpc.newBlockingStub(channel);
stub = stub.withInterceptors(MetadataUtils.newAttachHeadersInterceptor(headers));
```

---

## Load Balancing

```java
// Client-side load balancing (round-robin across resolved addresses)
ManagedChannel channel = ManagedChannelBuilder
    .forTarget("dns:///order-service:9090")    // DNS resolves multiple IPs
    .defaultLoadBalancingPolicy("round_robin")  // or "pick_first" (default)
    .build();

// With service discovery (e.g. Kubernetes headless service)
// DNS returns multiple pod IPs → round_robin distributes across them

// Custom name resolver for Consul/Eureka
ManagedChannel channel = ManagedChannelBuilder
    .forTarget("consul:///order-service")
    .nameResolverFactory(new ConsulNameResolverProvider())
    .defaultLoadBalancingPolicy("round_robin")
    .build();
```

---

## Health Checking

```java
// Server — register health service
HealthStatusManager healthManager = new HealthStatusManager();

Server server = ServerBuilder.forPort(9090)
    .addService(healthManager.getHealthService())
    .build();

// Set service health
healthManager.setStatus("com.example.OrderService",
    HealthCheckResponse.ServingStatus.SERVING);
healthManager.setStatus("com.example.OrderService",
    HealthCheckResponse.ServingStatus.NOT_SERVING);
healthManager.clearStatus("com.example.OrderService");

// Client — check health
HealthGrpc.HealthBlockingStub healthStub = HealthGrpc.newBlockingStub(channel);
HealthCheckResponse response = healthStub.check(
    HealthCheckRequest.newBuilder()
        .setService("com.example.OrderService")
        .build()
);
boolean healthy = response.getStatus() == HealthCheckResponse.ServingStatus.SERVING;

// Watch (streaming health updates)
HealthGrpc.HealthStub asyncHealthStub = HealthGrpc.newStub(channel);
asyncHealthStub.watch(request, new StreamObserver<HealthCheckResponse>() {
    public void onNext(HealthCheckResponse r) { updateLoadBalancer(r.getStatus()); }
    public void onError(Throwable t)          { markUnhealthy(); }
    public void onCompleted()                 {}
});
```

---

## Reflection

Allows tools like `grpcurl` and Postman to discover services at runtime.

```java
Server server = ServerBuilder.forPort(9090)
    .addService(ProtoReflectionService.newInstance()) // add reflection
    .build();
```

```bash
# List all services
grpcurl -plaintext localhost:9090 list

# Describe a service
grpcurl -plaintext localhost:9090 describe com.example.order.OrderService

# Call a method
grpcurl -plaintext -d '{"order_id": "ORD-123"}' \
  localhost:9090 com.example.order.OrderService/GetOrder
```

---

## gRPC vs REST

| Aspect | gRPC | REST/HTTP |
|---|---|---|
| Protocol | HTTP/2 | HTTP/1.1 or HTTP/2 |
| Payload | Protobuf (binary) | JSON (text) |
| Schema | Strongly typed `.proto` | OpenAPI / ad hoc |
| Code gen | First-class | Requires tooling (OpenAPI gen) |
| Streaming | Bidirectional built-in | SSE / WebSocket workarounds |
| Browser support | Limited (grpc-web needed) | Native |
| Human readable | No (binary) | Yes (JSON) |
| Performance | ~3–10× faster | Baseline |
| Error model | Status codes + rich details | HTTP status codes |
| Versioning | Field numbers + reserved | URL versioning / headers |
| Best for | Internal service-to-service | Public APIs, browsers |

---

## gRPC with Spring Boot

```xml
<dependency>
    <groupId>net.devh</groupId>
    <artifactId>grpc-server-spring-boot-starter</artifactId>
    <version>3.1.0.RELEASE</version>
</dependency>
<dependency>
    <groupId>net.devh</groupId>
    <artifactId>grpc-client-spring-boot-starter</artifactId>
    <version>3.1.0.RELEASE</version>
</dependency>
```

```yaml
# application.yml
grpc:
  server:
    port: 9090
    security:
      enabled: false  # set true for TLS
  client:
    order-service:
      address: static://order-service:9090
      negotiation-type: plaintext
```

```java
// Server service — just annotate
@GrpcService
public class OrderGrpcService extends OrderServiceGrpc.OrderServiceImplBase {

    @Autowired
    private OrderRepository orderRepository;

    @Override
    public void getOrder(GetOrderRequest request,
                         StreamObserver<Order> responseObserver) {
        // Spring beans available here
    }
}

// Client — inject stub
@Service
public class OrderClientService {

    @GrpcClient("order-service")  // matches grpc.client.order-service in yml
    private OrderServiceGrpc.OrderServiceBlockingStub orderStub;

    public Order getOrder(String orderId) {
        return orderStub
            .withDeadlineAfter(5, TimeUnit.SECONDS)
            .getOrder(GetOrderRequest.newBuilder().setOrderId(orderId).build());
    }
}
```

---

## Testing

```java
// Unit test — in-process channel (no network)
@ExtendWith(MockitoExtension.class)
class OrderGrpcServiceTest {

    private OrderGrpcService service;
    private ManagedChannel channel;
    private OrderServiceGrpc.OrderServiceBlockingStub stub;

    @BeforeEach
    void setUp() throws Exception {
        String serverName = InProcessServerBuilder.generateName();

        InProcessServerBuilder
            .forName(serverName)
            .directExecutor()
            .addService(new OrderGrpcService(mockOrderRepo))
            .build()
            .start();

        channel = InProcessChannelBuilder
            .forName(serverName)
            .directExecutor()
            .build();

        stub = OrderServiceGrpc.newBlockingStub(channel);
    }

    @AfterEach
    void tearDown() throws InterruptedException {
        channel.shutdown().awaitTermination(5, TimeUnit.SECONDS);
    }

    @Test
    void getOrder_shouldReturnOrder() {
        when(mockOrderRepo.findById("ORD-123")).thenReturn(Optional.of(sampleOrder));

        Order response = stub.getOrder(
            GetOrderRequest.newBuilder().setOrderId("ORD-123").build()
        );

        assertThat(response.getOrderId()).isEqualTo("ORD-123");
    }

    @Test
    void getOrder_shouldReturnNotFound() {
        when(mockOrderRepo.findById("UNKNOWN")).thenReturn(Optional.empty());

        StatusRuntimeException ex = assertThrows(StatusRuntimeException.class, () ->
            stub.getOrder(GetOrderRequest.newBuilder().setOrderId("UNKNOWN").build())
        );

        assertThat(ex.getStatus().getCode()).isEqualTo(Status.Code.NOT_FOUND);
    }
}
```

---

## Performance Tuning

```java
// Server — thread pool sizing
Server server = NettyServerBuilder.forPort(9090)
    .executor(Executors.newFixedThreadPool(
        Runtime.getRuntime().availableProcessors() * 2)) // I/O + compute
    .maxConnectionIdle(30, TimeUnit.MINUTES)
    .maxConnectionAge(2, TimeUnit.HOURS)
    .maxConnectionAgeGrace(30, TimeUnit.SECONDS)
    .permitKeepAliveTime(5, TimeUnit.SECONDS)   // minimum keep-alive interval from client
    .permitKeepAliveWithoutCalls(true)
    .maxInboundMessageSize(16 * 1024 * 1024)   // 16MB
    .build();

// Client — connection pooling (multiple subchannels)
ManagedChannel channel = ManagedChannelBuilder
    .forAddress("order-service", 9090)
    .keepAliveTime(30, TimeUnit.SECONDS)
    .keepAliveTimeout(10, TimeUnit.SECONDS)
    .keepAliveWithoutCalls(true)
    .idleTimeout(5, TimeUnit.MINUTES)
    .build();

// Large message compression
stub = stub.withCompression("gzip"); // per-call compression

// Server — enable compression
ServerBuilder.forPort(9090)
    .compressorRegistry(CompressorRegistry.getDefaultInstance())
    .decompressorRegistry(DecompressorRegistry.getDefaultInstance())
```

**Protobuf tips for performance:**
- Use field numbers 1–15 for frequent fields (1-byte tag)
- Prefer `sint32/sint64` for negative numbers (zigzag encoding)
- Use `bytes` + manual encoding for high-frequency binary data
- Avoid deeply nested messages in hot paths
- Reuse `MessageLite.Builder` objects when possible (avoid GC pressure)

---

## Common Pitfalls

| Pitfall | Problem | Fix |
|---|---|---|
| No deadline set | Client blocks forever | Always `.withDeadlineAfter()` |
| Leaking `StreamObserver` | Memory leak on error paths | Always call `onError` or `onCompleted`, never both |
| Calling `onNext` after `onCompleted` | `IllegalStateException` | Guard with a `completed` flag |
| Not propagating deadline | Downstream ignores parent deadline | Propagate `Context.current().getDeadline()` |
| EAGER channel creation per request | Connection overhead | Create channel once, share |
| Swallowing `StatusRuntimeException` | Silent failures | Always log + handle status code |
| Unbounded server streaming without backpressure | OOM on client | Check `Context.current().isCancelled()` in loops |
| Using ORDINAL enum in proto | Breaks on reorder | Always use field numbers, not values |
| Not adding `reserved` on field removal | Future reuse causes corruption | Always `reserved` deleted field numbers |
| Missing `option java_multiple_files = true` | All messages in one file | Add to all `.proto` files |

---

## Quick Reference Cheat Sheet

### Service type selection

```
Single request, single response    → Unary
Single request, stream of data     → Server Streaming
Stream of data, single response    → Client Streaming
Stream both ways, full duplex      → Bidirectional Streaming
```

### Stub type selection

```
Simple synchronous call            → BlockingStub
Async with callbacks               → AsyncStub (StreamObserver)
Integration with Future pipelines  → FutureStub (ListenableFuture)
```

### Error code selection

```
Bad input from client              → INVALID_ARGUMENT
Resource not found                 → NOT_FOUND
Already exists                     → ALREADY_EXISTS
No credentials                     → UNAUTHENTICATED
No permission                      → PERMISSION_DENIED
Rate limit hit                     → RESOURCE_EXHAUSTED
Retry this op                      → UNAVAILABLE or ABORTED
Time ran out                       → DEADLINE_EXCEEDED
Bug in server                      → INTERNAL
```

### Proto field type selection

```
Counter, age, quantity             → int32
Large ID, timestamp epoch          → int64
Possibly negative number           → sint32 / sint64
Price, percentage                  → double
Flag                               → bool
Text                               → string
Binary blob                        → bytes
List of things                     → repeated T
Nullable scalar                    → google.protobuf.XxxValue (wrapper type)
Timestamp                          → google.protobuf.Timestamp
Void return                        → google.protobuf.Empty
```

---

*References: gRPC official docs (grpc.io) | Protocol Buffers docs (protobuf.dev) | gRPC Java GitHub (github.com/grpc/grpc-java)*
