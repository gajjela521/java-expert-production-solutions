# Enterprise Java Design Patterns & Solutions

This repository contains expert-level documentation and implementation patterns for solving critical production issues in Enterprise Java applications.

## 📚 Contents

### 4. [Handling Large Payloads & Data Transfer](./04_LARGE_PAYLOAD_HANDLING.md)
*   **Problem**: "Mega-Object" failure, OOM, and Network Timeouts transferring massive lists/files.
*   **Solutions**:
    *   **Pagination & Batching**: Splitting 1M records into chunks of 1k.
    *   **Claim Check Pattern**: Using Blob Storage (S3) for payload and messaging for reference.
    *   **Streaming API**: Jackson/gRPC streaming for zero-memory overhead.

### 5. [Caching Strategies at Scale](./05_CACHING_AT_SCALE.md)
*   **Problem**: Cache Penetration, Breakdown (Thundering Herd), and Avalanche.
*   **Solutions**:
    *   Bloom Filters, Mutex Locks (Redisson), and Randomized TTL.
    *   **L1/L2 Caching**: Caffeine (Local) + Redis (Distributed) architecture.

### 6. [Database Scaling: Sharding & Partitioning](./06_DATABASE_SCALING_AND_SHARDING.md)
*   **Problem**: Vertical scaling limits and Replication Lag.
*   **Solutions**:
    *   **Sharding**: Horizontal splitting (Range vs Hash based).
    *   **Replication Lag**: Sticky sessions and "Read-your-own-writes".
    *   **Partitioning**: Postgres table partitioning by time.

### 7. [Async Processing & Messaging](./07_ASYNC_PROCESSING_AND_MESSAGING.md)
*   **Problem**: Coupled microservices and cascading failures.
*   **Solutions**:
    *   **Backpressure**: Queue-based load leveling.
    *   **Reliability**: Dead Letter Queues (DLQ) and Retry policies.
    *   **Transactional Outbox**: Solving the Dual-Write problem consistency.

### 8. [Memory Management & GC Tuning](./08_MEMORY_MANAGEMENT_GC_TUNING.md)
*   **Problem**: `OutOfMemoryError` and long GC pauses ("Stop-the-world").
*   **Solutions**:
    *   **Leak Analysis**: Heap Dump interpretation (MAT).
    *   **GC Selection**: G1 vs ZGC (Low Latency).
    *   **Off-Heap**: Using Direct `ByteBuffers` to bypass GC.

### 9. [Security Best Practices](./09_SECURITY_BEST_PRACTICES.md)
*   **Problem**: Injection, Broken Access Control (IDOR), and Credential Leaks.
*   **Solutions**:
    *   **AuthN/AuthZ**: JWT best practices (RS256) and Pre-authorize checks.
    *   **Secrets**: Vault/Env injection vs Git storage.
    *   **OWASP Top 10**: Practical fixes for Java apps.

### 10. [Observability & Monitoring](./10_OBSERVABILITY_AND_MONITORING.md)
*   **Problem**: "System is slow" but nobody knows where.
*   **Solutions**:
    *   **Three Pillars**: Logging (ELK), Metrics (Prometheus), Tracing (OpenTelemetry).
    *   **Spring Boot Actuator**: Health checks (Liveness/Readiness).
    *   **Alerting**: RED Method and Golden Signals.

## 🛠 Tech Stack
*   **Language**: Java 17 / 21
*   **Framework**: Spring Boot 3.x
*   **Infrastructure**: Kubernetes, Docker, Kafka, Redis, Postgres
*   **Observability**: Prometheus, Grafana, Zipkin
