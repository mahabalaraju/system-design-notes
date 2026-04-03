# Kafka — Complete Reference Guide
> Learned through building AgriConnect Agri-Tech Microservices Platform

---

## What is Kafka?

Apache Kafka is an open-source, distributed event streaming platform used for high-performance data pipelines, streaming analytics, and data integration.

**Event streaming** is the practice of:
- Capturing data in real-time from event sources like databases, sensors, mobile devices, cloud services, and software applications in the form of streams of events
- Storing these event streams durably for later retrieval
- Manipulating, processing, and reacting to the event streams in real-time as well as retrospectively

---

## Real-world use cases

| Industry | Use Case |
|----------|----------|
| Banking | Process payments and financial transactions in real-time |
| Logistics | Track and monitor cars, trucks, fleets, and shipments in real-time |
| IoT / Manufacturing | Capture and analyze sensor data from IoT devices continuously |
| E-commerce | Order processing, inventory updates, and delivery tracking |
| AgriConnect | Farmer registration, crop events, mandi price updates in real-time |

---

## Core concepts and terminology

### Topic
A topic is a category or feed name to which records are published. Think of it like a table in a database or a folder in a filesystem.

```
farmer.registered     ← all farmer registration events
crop.sowed            ← all crop sowing events
crop.harvested        ← all crop harvest events
crop.distress         ← all crop distress alerts
price.updated         ← all mandi price update events
```

### Producer
A producer is an application that publishes (writes) events to a Kafka topic.

```java
// In AgriConnect — farmer-service publishes this event
kafkaTemplate.send("farmer.registered", farmerId, farmerRegisteredEvent);
```

### Consumer
A consumer is an application that subscribes to (reads) events from a Kafka topic.

```java
// In AgriConnect — alert-service consumes this event
@KafkaListener(topics = "farmer.registered", groupId = "alert-service-group")
public void consumeFarmerRegistered(ConsumerRecord<String, FarmerRegisteredEvent> record) {
    // send welcome SMS to farmer
}
```

### Consumer Group
A consumer group is a group of consumers that work together to consume a topic. Each message is delivered to only one consumer within the group.

```
farmer.registered topic
        |
        ├──► alert-service-group   (one consumer gets the message)
        |         └── sends welcome SMS
        |
        └──► crop-service-group    (one consumer gets the message)
                  └── creates default crop profile
```

Key points:
- Different consumer groups each receive ALL messages independently
- Within the same group, messages are distributed across consumers
- This is how AgriConnect delivers the same event to both alert-service AND crop-service

### Partition
A topic is divided into partitions. Partitions allow Kafka to scale horizontally.

```
farmer.registered topic — 3 partitions
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│ Partition 0 │  │ Partition 1 │  │ Partition 2 │
│  Msg 0,3,6  │  │  Msg 1,4,7  │  │  Msg 2,5,8  │
└─────────────┘  └─────────────┘  └─────────────┘
```

In AgriConnect we use `farmerId` as the message key — all events for the same farmer always go to the same partition, ensuring ordering.

### Offset
The offset is a unique identifier for each record within a partition. It is a sequential number that Kafka assigns to each message.

```
Partition 0: [offset 0] [offset 1] [offset 2] [offset 3] ...
```

When a consumer acknowledges a message (`acknowledgment.acknowledge()`), Kafka advances the offset for that consumer group.

### Broker
A Kafka broker is a server that stores data and serves clients. A Kafka cluster consists of multiple brokers.

### Zookeeper
Zookeeper manages and coordinates Kafka brokers. It keeps track of which brokers are part of the Kafka cluster.

> Note: Newer versions of Kafka (KRaft mode) are replacing Zookeeper, but it is still widely used.

---

## Replication factor

When one Kafka broker (leader) goes down, other brokers take over to ensure no data loss.

```
Topic: farmer.registered — Replication Factor: 3

Broker 1 (Leader)    Broker 2 (Replica)    Broker 3 (Replica)
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  Partition 0 │────►│  Partition 0 │────►│  Partition 0 │
│  (Primary)   │     │  (Copy)      │     │  (Copy)      │
└──────────────┘     └──────────────┘     └──────────────┘
       ↓ fails
Broker 2 automatically becomes Leader — zero data loss
```

In AgriConnect docker-compose we use `replicas(1)` because it's a single-node local setup. In production always use `replicas(3)`.

---

## Retention policy

Kafka retention policy defines after how long a Kafka record is deleted.

```properties
# Keep messages for 7 days (default)
log.retention.hours=168

# Keep messages up to 1GB per partition
log.retention.bytes=1073741824

# Keep messages forever (set to -1)
log.retention.hours=-1
```

Unlike a queue (where message is deleted after consumption), Kafka retains messages even after consumption. This means:
- A new consumer can read historical messages
- A failed consumer can replay messages it missed
- Multiple consumer groups read the same messages independently

---

## Kafka Connect

Kafka Connect is a tool for scalably and reliably streaming data between Apache Kafka and other systems. It makes it simple to quickly define connectors that move large collections of data into and out of Kafka.

```
External Systems                    Kafka                    External Systems
┌─────────────┐                ┌──────────┐                ┌─────────────┐
│   MySQL DB  │──► Source   ──►│  Topic   │──► Sink     ──►│ Elasticsearch│
│   REST API  │    Connector   │          │    Connector    │  S3 Bucket  │
│   File CSV  │                └──────────┘                │  PostgreSQL │
└─────────────┘                                            └─────────────┘
```

Use cases:
- Sync MySQL database changes to Kafka automatically (Debezium connector)
- Stream Kafka events into Elasticsearch for search
- Archive Kafka events to S3 for long-term storage

---

## Kafka in AgriConnect

### Event flow

```
farmer-service
    │
    │ farmer.registered
    ├──────────────────► alert-service-group   → welcome SMS to farmer
    └──────────────────► crop-service-group    → creates default crop profile

crop-service
    │
    ├── crop.sowed      ──────────────────────► alert-service-group   → sowing notification
    ├── crop.harvested  ──────────────────────► market-service-group  → creates harvest record
    └── crop.distress   ──────────────────────► alert-service-group   → urgent alert

market-service
    │
    └── price.updated   ──────────────────────► alert-service-group   → price notification
```

### Topics created in AgriConnect

| Topic | Producer | Consumers | Partitions |
|-------|----------|-----------|------------|
| farmer.registered | farmer-service | alert-service, crop-service | 3 |
| crop.sowed | crop-service | alert-service | 3 |
| crop.harvested | crop-service | market-service | 3 |
| crop.distress | crop-service | alert-service | 3 |
| price.updated | market-service | alert-service | 3 |

### Dead Letter Topics (DLT)

When a consumer fails to process a message after all retries, the message goes to a Dead Letter Topic.

```
farmer.registered      → farmer.registered.DLT      (failed messages)
crop.harvested         → crop.harvested.DLT          (failed messages)
```

In AgriConnect we configured this using `@RetryableTopic`:
```java
@RetryableTopic(
    attempts = "3",
    backoff = @Backoff(delay = 1000, multiplier = 2),
    dltTopicSuffix = ".DLT"
)
```

This means: retry 3 times with delays of 1s → 2s → 4s. If all fail, send to DLT.

---

## Producer configuration in AgriConnect

```java
// KafkaConfig.java — producer settings
config.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);  // No duplicate messages
config.put(ProducerConfig.RETRIES_CONFIG, 3);                // Retry 3 times on failure
config.put(ProducerConfig.ACKS_CONFIG, "all");               // Wait for all replicas
```

| Config | Value | Meaning |
|--------|-------|---------|
| `ENABLE_IDEMPOTENCE` | true | Even if producer retries, message written exactly once |
| `RETRIES` | 3 | Retry sending up to 3 times if network fails |
| `ACKS` | all | Wait for all replica brokers to confirm before success |

---

## Consumer configuration in AgriConnect

```properties
# application.properties — consumer settings
spring.kafka.consumer.group-id=alert-service-group
spring.kafka.consumer.auto-offset-reset=earliest
spring.kafka.listener.ack-mode=manual
```

| Config | Value | Meaning |
|--------|-------|---------|
| `group-id` | alert-service-group | This consumer belongs to this group |
| `auto-offset-reset` | earliest | If no offset saved, read from beginning |
| `ack-mode` | manual | Consumer manually calls acknowledge() after processing |

### Manual acknowledgment pattern

```java
// In AgriConnect consumers
try {
    // process the event
    notificationService.sendWelcomeNotification(event);

    // tell Kafka: message processed successfully, move to next
    acknowledgment.acknowledge();

} catch (Exception e) {
    // do NOT acknowledge — Kafka will retry
    throw e;
}
```

---

## Serialization and deserialization

Kafka sends messages as bytes. We use JSON serialization in AgriConnect.

**Producer side (serialization — Java object → JSON bytes):**
```properties
spring.kafka.producer.value-serializer=org.springframework.kafka.support.serializer.JsonSerializer
```

**Consumer side (deserialization — JSON bytes → Java object):**
```properties
spring.kafka.consumer.value-deserializer=org.springframework.kafka.support.serializer.JsonDeserializer
spring.kafka.consumer.properties.spring.json.trusted.packages=*
spring.kafka.consumer.properties.spring.json.value.default.type=com.agriconnect.alertservice.dto.FarmerRegisteredEvent
```

---

## Message key importance

In AgriConnect we always use `farmerId` as the Kafka message key:

```java
kafkaTemplate.send(topic, farmerId, event);
//                         ↑
//                   message key
```

Why? Because Kafka guarantees that all messages with the same key go to the same partition. This means:
- All events for farmer "ABC" always go to Partition 1
- Events for farmer "ABC" are always processed in order (registered → sowed → harvested)
- No out-of-order processing for the same farmer

---

## Kafka vs traditional messaging (RabbitMQ / ActiveMQ)

| Feature | Kafka | RabbitMQ |
|---------|-------|----------|
| Message retention | Keeps messages (configurable) | Deletes after consumption |
| Consumer groups | Multiple groups, each gets all messages | One consumer gets each message |
| Ordering | Guaranteed within partition | Not guaranteed |
| Throughput | Millions of messages/second | Thousands/second |
| Replay | Yes — read historical messages | No |
| Use case | Event streaming, audit log | Task queue, RPC |

---

## What to say in interviews

> "In AgriConnect we used Apache Kafka as the messaging backbone connecting five microservices. When a farmer registers, farmer-service publishes a `farmer.registered` event. Both alert-service and crop-service consume this event independently through separate consumer groups — alert-service sends a welcome SMS while crop-service creates a default crop profile. We used `farmerId` as the message key to guarantee ordering within partitions, configured idempotent producers to prevent duplicate events, and implemented `@RetryableTopic` with exponential backoff and dead letter topics for production-grade error handling."

---

## Kafka CLI commands (useful for debugging)

```bash
# List all topics
kafka-topics.sh --list --bootstrap-server localhost:9092

# Create a topic
kafka-topics.sh --create --topic farmer.registered --partitions 3 --replication-factor 1 --bootstrap-server localhost:9092

# Describe a topic
kafka-topics.sh --describe --topic farmer.registered --bootstrap-server localhost:9092

# Consume messages from beginning
kafka-console-consumer.sh --topic farmer.registered --from-beginning --bootstrap-server localhost:9092

# Check consumer group offsets
kafka-consumer-groups.sh --describe --group alert-service-group --bootstrap-server localhost:9092
```

---

## Kafka UI (used in AgriConnect)

We added Kafka UI to docker-compose for visual monitoring:

```yaml
kafka-ui:
  image: provectuslabs/kafka-ui:latest
  ports:
    - "8090:8080"
  environment:
    KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:9092
```

Access at: `http://localhost:8090`

From Kafka UI you can:
- See all topics and their partitions
- Browse messages in each topic
- Monitor consumer group lag
- Inspect Dead Letter Topics for failed messages

---

*Document maintained by H Mahabalaraju — AgriConnect Project*
*Last updated: 2026*
