# 11 — Messaging

## M36. Spring Kafka

### T1. Kafka three ways — `MK` `[P→S→B]`
- A plain `KafkaProducer` / `KafkaConsumer` poll loop → `KafkaTemplate` / `@KafkaListener` → Boot's `spring.kafka.*` auto-configuration.
- Domain: `account-service` publishes `AccountStatusChanged`, and `client-service` consumes it.

### T2. Producing — `MK`
- `KafkaTemplate`, `ProducerFactory`.
- Send results: `ListenableFuture` in 2.x → `CompletableFuture` in `[3.x]`.
- Keys and partitioning; JSON serializers and type headers; the idempotent producer; `acks`.

### T3. Consuming — `MK`
- `@KafkaListener`, container factories, concurrency, consumer groups.
- Ack modes and manual acknowledgment; batch listeners.
- `ErrorHandlingDeserializer` for messages that fail to deserialize.

### T4. Listener containers — `MK`
- Single vs concurrent containers; their `SmartLifecycle` behaviour.
- Pause and resume; rebalance listeners.

### T5. Error handling — `MK`
- `DefaultErrorHandler` (2.8+, which replaced `SeekToCurrentErrorHandler`) with backoff.
- `DeadLetterPublishingRecoverer`.
- Non-blocking retries with `@RetryableTopic`; poison-pill messages.

### T6. Transactions and exactly-once — `MK`
- `KafkaTransactionManager`, `transactional.id`, `read_committed` consumers.
- Combining Kafka with a database transaction; the outbox pattern.

### T7. Request-reply — `GTH`
- `ReplyingKafkaTemplate`.

### T8. Kafka Streams — `GTH`
- `@EnableKafkaStreams`.

### T9. Security configuration — `GTH`
- SASL / SSL properties.

### T10. Observability — `MK`
- Micrometer metrics.
- `[3.x]` Observation and tracing support.

### T11. Testing — `MK`
- `@EmbeddedKafka`; a Kafka Testcontainer.
- `[3.x]` `@ServiceConnection`.

### Traps
- An auto-commit misunderstanding causes lost or duplicate messages.
- Infinite retry on a poison pill.
- A listener that does a slow blocking call and gets kicked from the group during a rebalance.

---

## M37. Messaging Beyond Kafka — `GTH`

### T1. Spring Cloud Stream
- The functional model (`Supplier` / `Function` / `Consumer`), binders, `StreamBridge`.

### T2. SQS with Spring Cloud AWS
- Version pairing: 2.4 with Boot 2.7, `[3.x]` 3.x with Boot 3.
- `@SqsListener`, `SqsTemplate`; visibility timeout; DLQ.

### T3. The Spring messaging abstraction
- `Message`, `MessageChannel`; pointers to JMS and RabbitMQ.

### T4. Choosing a messaging approach
