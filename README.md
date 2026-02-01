# Enterprise Java Design Patterns & Solutions

This repository contains expert-level documentation and implementation patterns for solving critical production issues in Enterprise Java applications.

## 📚 Contents

### 1. [API Rate Limiting & DDoS Protection](./01_API_RATE_LIMITING_AND_DDOS.md)
*   **Problem**: Handling traffic spikes, flash sales, and malicious attacks.
*   **Solutions**:
    *   Token Bucket vs Sliding Window algorithms.
    *   Distributed Rate Limiting with **Redis + Lua**.
    *   Spring Boot + **Bucket4j** implementation.
    *   Defense in Depth (WAF -> Gateway -> App).

### 2. [Distributed Transactions & Resilience](./02_DISTRIBUTED_TRANSACTIONS_AND_RESILIENCE.md)
*   **Problem**: Data consistency across microservices and network failures.
*   **Solutions**:
    *   **Saga Pattern** (Orchestration vs Choreography).
    *   **Idempotency** implementation (Header-based).
    *   **Resilience4j**: Circuit Breakers, Bulkheads, Retro with Exponential Backoff.
    *   Why 2PC/XA is bad for microservices.

### 3. [High-Performance Concurrency & DB](./03_HIGH_PERFORMANCE_CONCURRENCY_AND_DB.md)
*   **Problem**: Thread starvation, race conditions (double booking), and slow queries.
*   **Solutions**:
    *   **Virtual Threads** (Java 21) vs Async (`CompletableFuture`).
    *   **Distributed Locking** with Redisson (Redis).
    *   **Optimistic Locking** (JPA `@Version`).
    *   Solving the **N+1 Problem** and tuning **HikariCP**.

## 🛠 Tech Stack
*   **Language**: Java 17 / 21
*   **Framework**: Spring Boot 3.x
*   **Database**: PostgreSQL, Redis
*   **Libraries**: Bucket4j, Resilience4j, Redisson
