# 21 — Messaging: Kafka & RabbitMQ with Spring

## 1. Why messaging?
Synchronous REST calls create **temporal coupling** (both services must be up) and latency chains. Messaging enables:
- **Decoupling** — producer doesn't know consumers.
- **Asynchronous processing** — respond fast, process later.
- **Load leveling** — queue absorbs spikes.
- **Resilience** — consumer down? messages wait.
- **Fan-out** — one event, many subscribers (email, analytics, inventory).
- **Event-driven architecture** & event sourcing.

### Messaging models
| Model | Delivery | Example |
|-------|----------|---------|
| **Point-to-point (queue)** | Each message consumed by one consumer | Task/work queue |
| **Publish/subscribe (topic)** | Every subscriber group gets a copy | `OrderPlaced` event to many services |

### Commands vs events
- **Command**: "do this" (`ReserveStock`) — one intended handler.
- **Event**: "this happened" (`OrderPlaced`) — past tense, any number of listeners.
- **Document/state transfer**: full entity snapshot.

### Delivery semantics
| Semantic | Meaning | How |
|----------|---------|-----|
| At-most-once | May lose, never duplicate | Commit offset before processing |
| **At-least-once** (typical) | Never lose, may duplicate | Commit/ack after processing → **consumers must be idempotent** |
| Exactly-once | Effectively once | Kafka transactions (read-process-write within Kafka) or idempotent consumer + dedup table |

**Idempotent consumer**: store processed message ids (unique constraint) or use naturally idempotent operations (upsert, `SET status = 'PAID' WHERE status = 'PENDING'`).

## 2. Apache Kafka

### 2.1 Concepts
```
Producer ──► Topic "orders" ──────────────────────────────────────────►
              ├─ Partition 0: [m0][m1][m2][m3] ...   (leader on broker 1, replicas on 2,3)
              ├─ Partition 1: [m0][m1] ...
              └─ Partition 2: [m0][m1][m2] ...
                                    ▲
Consumer group "inventory-svc": consumer A reads P0,P1; consumer B reads P2
Consumer group "email-svc":     consumer C reads P0,P1,P2   (independent offsets)
```
| Term | Meaning |
|------|---------|
| **Broker** | Kafka server; cluster = many brokers |
| **Topic** | Named stream of records (category) |
| **Partition** | Ordered, append-only log; unit of parallelism & ordering |
| **Offset** | Position of a record in a partition |
| **Record** | key, value, headers, timestamp |
| **Producer** | Writes records; key → partition (hash(key) % partitions) |
| **Consumer group** | Set of consumers sharing work; each partition consumed by exactly one consumer in the group |
| **Replication factor** | Copies of each partition (typically 3) |
| **ISR** | In-sync replicas |
| **Retention** | Kept by time/size regardless of consumption (default 7 days); or **log compaction** keeps last value per key |
| **KRaft** | Kafka's built-in consensus (ZooKeeper removed in Kafka 4.0) |

**Ordering**: guaranteed **only within a partition**. Use the entity id (orderId) as key → all events for an order go to the same partition, in order.
**Scaling**: max useful consumers in a group = number of partitions. Choose partitions with growth in mind (can increase, but key→partition mapping changes).
**Rebalancing**: when consumers join/leave, partitions are reassigned (cooperative sticky assignor minimizes disruption).

### 2.2 Producer reliability settings
| Setting | Recommendation |
|---------|----------------|
| `acks=all` | Wait for all ISR replicas (default since Kafka 3.0) |
| `enable.idempotence=true` | No duplicates from producer retries (default since 3.0) |
| `retries`, `delivery.timeout.ms` | Retry transient errors |
| `min.insync.replicas=2` (topic/broker) | With RF=3 and acks=all → tolerate 1 broker down |
| `linger.ms`, `batch.size`, `compression.type=lz4/zstd` | Throughput tuning |

### 2.3 Spring for Apache Kafka
```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: all
    consumer:
      group-id: inventory-service
      auto-offset-reset: earliest              # where to start if no committed offset
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.ErrorHandlingDeserializer
      properties:
        spring.deserializer.value.delegate.class: org.springframework.kafka.support.serializer.JsonDeserializer
        spring.json.trusted.packages: "com.shop.events"
    listener:
      ack-mode: record                         # Spring commits after the listener returns successfully
```

```java
// Topic creation (dev); in prod topics are usually managed by infra/GitOps
@Bean NewTopic ordersTopic() { return TopicBuilder.name("orders").partitions(6).replicas(3).build(); }

// Event
public record OrderPlacedEvent(String eventId, Long orderId, Long customerId, BigDecimal total, Instant occurredAt) {}

// Producer
@Service
@RequiredArgsConstructor
public class OrderEventPublisher {
    private final KafkaTemplate<String, OrderPlacedEvent> kafka;

    public void publish(OrderPlacedEvent e) {
        kafka.send("orders", e.orderId().toString(), e)          // key = orderId → ordering per order
             .whenComplete((result, ex) -> {
                 if (ex != null) log.error("Publish failed for {}", e.orderId(), ex);
                 else log.debug("Sent to {}-{}@{}", result.getRecordMetadata().topic(),
                         result.getRecordMetadata().partition(), result.getRecordMetadata().offset());
             });
    }
}

// Consumer
@Component
@Slf4j
public class InventoryListener {

    @KafkaListener(topics = "orders", groupId = "inventory-service", concurrency = "3")
    public void onOrderPlaced(OrderPlacedEvent event,
                              @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
                              @Header(KafkaHeaders.OFFSET) long offset) {
        if (processedEvents.existsById(event.eventId())) return;    // idempotency
        inventoryService.reserve(event.orderId());
        processedEvents.save(new ProcessedEvent(event.eventId()));
    }
}
```

### 2.4 Error handling, retries & DLT
```java
@Bean
DefaultErrorHandler errorHandler(KafkaTemplate<Object, Object> template) {
    var recoverer = new DeadLetterPublishingRecoverer(template);           // → topic "orders.DLT"
    var backoff = new ExponentialBackOffWithMaxRetries(3);
    backoff.setInitialInterval(1_000); backoff.setMultiplier(2.0);
    DefaultErrorHandler h = new DefaultErrorHandler(recoverer, backoff);
    h.addNotRetryableExceptions(ValidationException.class, DeserializationException.class);
    return h;
}
```
Non-blocking retries with retry topics:
```java
@RetryableTopic(attempts = "4", backoff = @Backoff(delay = 1000, multiplier = 2), dltTopicSuffix = ".DLT")
@KafkaListener(topics = "payments")
public void onPayment(PaymentEvent e) { ... }
@DltHandler public void dlt(PaymentEvent e) { alerting.notify(e); }
```
- **Poison pill** = a record that always fails (bad format) — `ErrorHandlingDeserializer` + DLT prevents an infinite loop.
- Blocking retries keep partition order but stall it; retry topics keep throughput but break ordering.

### 2.5 Transactions & exactly-once
- Producer transactions (`spring.kafka.producer.transaction-id-prefix`) + `read_committed` consumers → atomic multi-topic writes and consume-transform-produce exactly-once within Kafka.
- DB + Kafka atomicity is **not** solved by Kafka transactions → use the **Transactional Outbox** pattern (write event row in the same DB transaction; Debezium CDC or a poller publishes it).

### 2.6 Kafka ecosystem
- **Kafka Connect** (source/sink connectors: Debezium CDC from Postgres/MySQL, JDBC, S3, Elasticsearch).
- **Kafka Streams** / Spring Cloud Stream Kafka Streams binder — stateful stream processing (joins, aggregations, windows).
- **Schema Registry** (Avro/Protobuf/JSON Schema) — schema evolution with compatibility rules (backward/forward).
- Monitoring: consumer **lag** (Burrow, Kafka UI, Prometheus exporters) — key health metric.

## 3. RabbitMQ (AMQP 0-9-1)

### 3.1 Concepts
```
Producer ──► Exchange ──(binding: routing key / pattern / headers)──► Queue ──► Consumer
```
| Exchange type | Routing |
|---------------|---------|
| **Direct** | Exact routing key match (`order.created`) |
| **Topic** | Pattern: `*` one word, `#` zero or more (`order.*.eu`, `order.#`) |
| **Fanout** | Broadcast to all bound queues, ignores key |
| **Headers** | Match on headers |
| Default ("") | Routes to queue named by routing key |

- Messages are **removed after acknowledgment** (queue semantics, not a log). Replay not supported (except Streams).
- Acks: auto vs manual; `basicNack`/reject with requeue; **prefetch** (QoS) limits unacked messages per consumer.
- **Dead Letter Exchange (DLX)**: rejected/expired/overflowed messages routed elsewhere.
- **TTL**, priority queues, delayed messages (plugin).
- **Quorum queues** (Raft-replicated) for HA — recommended over classic mirrored queues.
- **Publisher confirms** for reliable publishing.

### 3.2 Spring AMQP
```java
@Configuration
class RabbitConfig {
    @Bean TopicExchange ordersExchange() { return new TopicExchange("orders.exchange"); }
    @Bean Queue emailQueue() {
        return QueueBuilder.durable("email.queue").quorum()
            .withArgument("x-dead-letter-exchange", "orders.dlx").build();
    }
    @Bean Binding emailBinding(Queue emailQueue, TopicExchange ordersExchange) {
        return BindingBuilder.bind(emailQueue).to(ordersExchange).with("order.created.#");
    }
    @Bean MessageConverter jsonConverter() { return new Jackson2JsonMessageConverter(); }
}

@Service @RequiredArgsConstructor
class Publisher {
    private final RabbitTemplate rabbit;
    void publish(OrderPlacedEvent e) { rabbit.convertAndSend("orders.exchange", "order.created.in", e); }
}

@Component
class EmailConsumer {
    @RabbitListener(queues = "email.queue", concurrency = "2-5")
    void handle(OrderPlacedEvent e) { mailer.sendConfirmation(e); }   // exception → retry/DLX per config
}
```
```yaml
spring.rabbitmq:
  host: localhost
  listener.simple:
    acknowledge-mode: auto
    prefetch: 10
    retry: { enabled: true, max-attempts: 3, initial-interval: 1s, multiplier: 2 }
    default-requeue-rejected: false     # after retries → DLX instead of infinite requeue
```

## 4. Kafka vs RabbitMQ
| | Kafka | RabbitMQ |
|--|--|--|
| Model | Distributed **log** (pull) | **Message broker** with queues (push) |
| Retention | Kept for retention period; **replayable** | Deleted once acked |
| Ordering | Per partition | Per queue (single consumer) |
| Throughput | Very high (millions/sec) | High (tens of thousands/sec per queue) |
| Routing | Topic/partition by key | Rich routing (exchanges, patterns, headers) |
| Consumers | Groups track their own offsets | Competing consumers on a queue |
| Use cases | Event streaming, event sourcing, CDC, analytics pipelines, log aggregation, high-volume events between microservices | Task queues, RPC, complex routing, per-message TTL/priority, lower volume integration |

Others: **AWS SQS/SNS**, Google Pub/Sub, Azure Service Bus, Apache Pulsar, NATS, ActiveMQ Artemis (JMS).

## 5. Spring Cloud Stream (broker-agnostic)
Write functions; bind them to Kafka/Rabbit by config:
```java
@Bean Function<OrderPlacedEvent, InvoiceCreatedEvent> createInvoice() { return e -> invoiceService.create(e); }
@Bean Consumer<InvoiceCreatedEvent> notifyCustomer() { return e -> mailer.send(e); }
@Bean Supplier<Heartbeat> heartbeat() { return Heartbeat::now; }      // polled every second by default
```
```yaml
spring.cloud:
  function.definition: createInvoice;notifyCustomer
  stream.bindings:
    createInvoice-in-0:  { destination: orders, group: billing }
    createInvoice-out-0: { destination: invoices }
    notifyCustomer-in-0: { destination: invoices, group: notifications }
```
`StreamBridge.send("orders", event)` for imperative sends.

## 6. Event design best practices
- Events are **immutable facts**, past tense, named in domain language.
- Include: `eventId` (UUID), `eventType`, `version`, `occurredAt`, aggregate id, correlation/causation ids, payload.
- Prefer small events + query for details, or full snapshots (ECST) — trade-off of coupling vs chattiness.
- **Schema evolution**: add optional fields only; never rename/remove without versioning; consumers ignore unknown fields.
- Use **CloudEvents** spec for envelope consistency.
- Publish **after** the DB transaction commits — outbox pattern for reliability.
- Monitor consumer lag, DLQ sizes; alert on growth.
- Keep consumers idempotent and handle out-of-order delivery (version numbers).

## 7. Interview Q&A
**Q. How does Kafka guarantee ordering?** Only within a partition; use a key so related messages land in the same partition.

**Q. What is a consumer group?** Consumers sharing a group id split partitions among themselves; each partition goes to one member; different groups each get all messages.

**Q. What happens if you have more consumers than partitions?** Extra consumers are idle.

**Q. At-least-once vs exactly-once?** At-least-once may duplicate (commit after processing) → idempotent consumers. Exactly-once via Kafka transactions for Kafka-to-Kafka flows, or idempotency + dedup for external side effects.

**Q. How do you handle failed messages?** Retries with backoff, then dead-letter topic/queue; alerting and reprocessing tools; skip poison pills.

**Q. Kafka vs RabbitMQ — when to use which?** (Table above.) Kafka for high-throughput replayable event streams; RabbitMQ for flexible routing and work queues.

**Q. How do you publish an event reliably after a DB update?** Transactional outbox + relay/CDC (Debezium); not "save then send" (dual-write problem).

**Q. What is consumer lag?** Difference between latest offset and the group's committed offset — indicates consumers falling behind.

**Q. What is log compaction?** Kafka keeps at least the latest record per key, useful for changelog/state topics.
