### How will you design logging in microservices using Kafka?

# Designing Logging in Microservices Using Kafka
Kafka is a strong choice for **centralized, scalable, and decoupled logging** in microservice architectures.

## 1. Goals of Kafka-based Logging
- Centralized logging across microservices
- Asynchronous, non-blocking log ingestion
- High throughput and durability
- Fan-out logs to multiple consumers
- Decoupling log producers from log processors

## 2. High-Level Architecture
```text
+-------------------+
| Microservice A   |
| (Logger + App)   |
+---------+---------+
          |
          v
+-------------------+
| Kafka Producer   |
+-------------------+
          |
          v
+===================+
|     Kafka Topic   |
|   (logs-topic)    |
+===================+
     |        |
     v        v
+---------+ +----------------+
| ELK     | | Alerting / SIEM|
| Consumer| | Consumer       |
+---------+ +----------------+
```

## 3. Logging Strategy

### 3.1 Log Levels
Use standard log levels consistently:
- `ERROR` – system failures, unhandled exceptions
- `WARN` – unexpected but recoverable situations
- `INFO` – business events, lifecycle events
- `DEBUG` – detailed execution flow (disable in production)
- `TRACE` – very fine-grained logs (rarely enabled)

⚠️ Avoid sending `DEBUG` and `TRACE` logs to Kafka in production environments.

## 4. Log Format (Structured Logging)

Always use **structured logging (JSON)** instead of plain text.

### Example Log Event

```json
{
  "timestamp": "2025-01-12T10:15:30.123Z",
  "service": "order-service",
  "environment": "prod",
  "level": "ERROR",
  "message": "Failed to create order",
  "traceId": "af34c12e98",
  "spanId": "bc921",
  "userId": "12345",
  "httpMethod": "POST",
  "endpoint": "/orders",
  "statusCode": 500,
  "exception": {
    "type": "NullPointerException",
    "message": "order is null",
    "stackTrace": "..."
  }
}
```

### Mandatory Fields

The following fields must be present in every log entry:
- service
- environment
- timestamp
- level
- traceId

## 5. Producing Logs to Kafka

Logs should be sent to Kafka asynchronously to avoid blocking application threads.

### Option A: Application → Kafka (Direct)

Each microservice publishes logs directly to Kafka using a Kafka producer.

**Advantages**
- Simple architecture
- Full control over log format and schema

**Disadvantages**
- Application becomes coupled to Kafka
- Kafka outages may impact logging
  
👉 Use async producer + buffering.

### Option B: Sidecar / Agent Pattern (Recommended)

```text
Application → stdout → Fluent Bit / Logstash → Kafka
```

**Advantages**
- Decouples application from Kafka
- Easier to switch log destinations
- Kubernetes-native and widely adopted

## 6. Kafka Topic Design

### 6.1 Topic Naming
```
logs.<environment>.<domain>

logs.prod.application
logs.prod.security
logs.dev.application
```

### 6.2 Partition Strategy
- Use high partition count (e.g., 12–48)
- Key by:
  - service-name OR
  - traceId (keeps logs of a request ordered)
 
### 6.3 Retention Policy
Kafka is not long-term storage.

Typical:
- 3–7 days retention
- Size-based retention preferred

## 7. Reliability & Performance
### Producer Configuration (Critical)
```
acks=1
retries=3
linger.ms=5
batch.size=65536
compression.type=lz4
```
💡 Never use `acks=all` for logs (too slow).

### Failure Handling
- If Kafka is unavailable:
  - Drop logs (acceptable)
  - OR write to local file as fallback
- Logging must never crash your service

## 8. Consumers (Downstream Processing)
### Common Consumers

#### 1. ELK / OpenSearch
 - Index logs for search

#### 2. Alerting Service
- Detect ERROR spikes

#### 3. Audit / Compliance
- Immutable storage (S3, HDFS)

#### 4. ML / Analytics
- Anomaly detection

Kafka allows multiple consumers without duplication.

## 9. Correlation with Tracing & Metrics
### Combine:
- Logs → Kafka
- Traces → Jaeger / Tempo
- Metrics → Prometheus

### Ensure:
- traceId is injected into logs automatically
- Use OpenTelemetry

## 10. Security Considerations
- Mask PII (emails, passwords, tokens)
- Enable Kafka:
  - TLS
  - SASL authentication
- Restrict topic access (ACLs)

🚫 Never log secrets (JWTs, API keys)

## 11. Common Mistakes to Avoid
❌ Logging synchronously

❌ Logging everything to Kafka

❌ Plain text logs

❌ No traceId

❌ Using Kafka as long-term storage

❌ Letting logging failures crash the app

---

### How will you handle failures when saving logs to the database to ensure that no logs are lost?

Durability is guaranteed by kafka
- Kafka topics are persistent and replicated.
- When microservices produce logs:
  - Kafka stores them on disk.
  - Multiple replicas ensure fault tolerance.
- Even if the DB is down, logs remain in Kafka.
- Consumers can read from Kafka once the DB is back online.

✅ This decouples the logging pipeline from database availability.

#### Consumer Handling of Database Downtime
When the database goes down:
1. Consumers detect DB write failures.
2. Consumers can:
  - Retry automatically (with exponential backoff or batch retries).
  - Keep the offset uncommitted in Kafka until the write succeeds.
3. Because the consumer doesn’t commit the Kafka offset until the database write succeeds:
  - Kafka retains the messages.
  - Messages are not lost.
4. Optionally:
- Use a dead-letter topic for logs that fail permanently after several retries.
- Can also use local disk buffer if needed.

#### Idempotent Writes
To handle retries safely, writes to the DB should be idempotent:
- Include a unique log ID (UUID or traceId + timestamp) in each log record.
- This ensures that retrying a message does not create duplicates.

#### Complete Flow During DB Downtime
```text
[Microservice A] ----+
[Microservice B] ----+--> [Kafka Topic] ---> [Kafka Consumer] ---> [Database]
                     |
                     | (DB is down)
                     v
             Kafka retains logs safely
```
- Logs stay in Kafka until the consumer successfully writes to the database.
- DB downtime does not affect microservices.
- Once the DB is back online, consumers continue processing from the last committed offset.

---

### How will you manually commit offsets, and what kinds of challenges might you face with this approach?

## 1. What is Manual Offset Commit?

- In Kafka, an offset represents the position of a consumer in a partition.
- Automatic commit: Kafka periodically commits offsets (default enable.auto.commit=true).
- Manual commit: The consumer decides when to commit an offset (enable.auto.commit=false).

#### Why manual commit?
- It gives precise control over when a message is considered “processed.”
- For logging pipelines, you usually want to commit the offset only after a log is successfully written to the database.

## 2. Challenges with Manual Commit
### 2.1 Duplicate Processing
- If a consumer crashes after processing a message but before committing the offset, Kafka will re-deliver the same message when the consumer restarts.
- **Solution:** Make DB writes idempotent (e.g., include a unique log ID).

### 2.2 Message Loss
- If you commit an offset before processing, messages may be lost during failures.
- **Rule of thumb:** Commit after success.

### 2.3 Performance Overhead
- Committing offsets too frequently can:
  - Increase network chatter with Kafka
  -Affect throughput
- **Solution:** Commit offsets in batches, e.g., after processing N messages or after T milliseconds.

#### 2.4 Complex Error Handling
- If DB fails for a batch:
  - You need to retry without committing offset
  - May require dead-letter topics for messages that cannot be processed after multiple retries.

### 2.5 Consumer Group Rebalance
- Manual commits may interact poorly with consumer group rebalances:
  - Partitions can be reassigned before offsets are committed
  - Messages may be processed twice
- **Solution:** Handle ConsumerRebalanceListener to commit offsets before losing partitions.

### 3. Best Practices
- **Commit after processing** → ensures no logs are lost.
- **Use idempotent writes** → handles retries without duplicates.
- **Batch commits** → commit after N messages or T seconds.
- **Handle rebalance events** → commit offsets in onPartitionsRevoked.

Monitor consumer lag → ensure no backlog builds up during DB downtime.
---

### Difference between MongoDB and MySQL

### One usecase where you will prefer MongoDB and why, answer in terms of data model
