# Asynchronous Processing & Event-Driven Architecture

## 1. The Trap of Synchronous Microservices
If Service A calls B, C, and D synchronously (HTTP), the **latency stacks** ($A = B + C + D$). If D fails, A fails. This is a distributed monolith.

## 2. Decoupling with Message Queues (Kafka/SQS)
**Fire and Forget**.
Service A publishes "OrderCreated" event. Services B, C, D listen and process at their own pace.

### 2.1 The "Backpressure" Pattern
**Scenario**: Marketing sends 1M emails. The "Email Service" can only handle 100/sec.
*   **Http Push**: The sender crashes the receiver (DDoS).
*   **Queue Pull**: The receiver pulls 10 messages, processes them, then pulls 10 more. The Queue acts as a buffer.

## 3. Reliability Patterns in Messaging

### 3.1 Dead Letter Queues (DLQ)
**Issue**: A "Poison Message" (malformed JSON) crashes the consumer. The consumer restarts, reads the crash-message again, and infinite loops.
**Solution**:
1.  Retry X times (e.g., 3).
2.  If still failing, move message to **DLQ (Dead Letter Queue)**.
3.  Alert humans to inspect DLQ manually.

### 3.2 Outbox Pattern (Dual Write Problem)
**Issue**: You save to DB (`INSERT ORDER`) and publish to Kafka (`send(OrderCreated)`).
If Kafka fails after DB commit, the event is lost. Inconsistent state.
**Solution: The Transactional Outbox**.
1.  Save Order AND "Outbox Event" in the **same DB transaction**.
    ```sql
    INSERT INTO orders ...;
    INSERT INTO outbox (topic, payload) VALUES ('orders', '{...}');
    COMMIT;
    ```
2.  A separate process (Debezium / Poller) reads the Outbox table and pushes to Kafka. This guarantees **At-Least-Once** delivery.

## 4. Kafka vs RabbitMQ vs ActiveMQ
| Feature | Kafka | RabbitMQ / ActiveMQ |
| :--- | :--- | :--- |
| **Model** | Log (Stream). Clients track offset. | Queue (Push). Broker tracks state. |
| **Persistence** | Permanent (Retention Policy). | Ephemeral (Deleted on Ack). |
| **Throughput** | Extreme (Millions/sec). | Moderate (Thousands/sec). |
| **Use Case** | Event Sourcing, Analytics, Big Data. | Complex Routing, Task Queues. |

## 5. Idempotent Consumer (Again)
Since queues guarantee "At-Least-Once", you **will** receive duplicates.
Your consumer code must be idempotent (See Chapter 02).
