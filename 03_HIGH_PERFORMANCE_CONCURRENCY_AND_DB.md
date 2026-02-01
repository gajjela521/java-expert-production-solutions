# High-Performance Concurrency & Threading

## 1. The Issue: "Thread Starvation" & "Race Conditions"
In high-throughput Java applications, incorrect threading leads to:
1.  **Thread Starvation**: All threads blocked waiting for I/O (DB/HTTP).
2.  **Race Conditions**: Two threads modifying the same data -> Data Corruption.
3.  **Deadlocks**: Thread A waiting for B, B waiting for A.

## 2. Solutions for I/O Blocking (Async)

### 2.1 The Problem with Synchronous Servlets (Tomcat)
Default Tomcat has ~200 threads. If requests take 2 seconds (slow DB), you handle 100 RPS max. The CPU is idle, waiting for I/O.

### 2.2 Solution: WebFlux (Reactive) or CompletableFuture
Move from "Thread-per-Request" to "Event Loop".

**CompletableFuture Approach (standard Spring MVC)**:
```java
@Async // Run in a separate pool
public CompletableFuture<User> findUser() {
    // This runs in 'taskExecutor', freeing up the Tomcat HTTP thread immediately
    return CompletableFuture.completedFuture(repo.findUser());
}
```

**Virtual Threads (Java 21+)**:
The Game Changer. Use `Executors.newVirtualThreadPerTaskExecutor()`.
*   **Old**: 200 OS threads (Heavy).
*   **New**: 1,000,000 Virtual threads (Lightweight).
*   **Action**: Upgrade to Java 21 and Spring Boot 3.2+. It handles this automatically.

## 3. Distributed Locking (Race Conditions)
**Scenario**: "Double Booking". Two users book the last seat at the exact same millisecond. Validations pass for both.

**Local Lock (`synchronized`)**: Works on 1 server.
**Cluster Lock**: Fails. Need **Distributed Lock**.

### 3.1 Redis Distributed Lock (Redlock / Redisson)
Use `Redisson` for easy locking.

```java
@Autowired
private RedissonClient redisson;

public void bookSeat(String seatId) {
    RLock lock = redisson.getLock("seat_lock:" + seatId);
    
    try {
        // Wait 100ms for lock, auto-release after 10s (prevent deadlock if crash)
        if (lock.tryLock(100, 10000, TimeUnit.MILLISECONDS)) {
            // CRITICAL SECTION
            if (seat.isFree()) {
                seat.book();
            }
        }
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    } finally {
        lock.unlock();
    }
}
```

### 3.2 Optimistic Locking (Database Versioning) -> Check-Then-Act
Instead of heavy locks, assume success and check at commit time.
*   **Add column**: `@Version private Long version;` (JPA/Hibernate)
*   **Flow**:
    1.  User A reads Row (Version 1).
    2.  User B reads Row (Version 1).
    3.  User A Updates -> `UPDATE table SET val=X, ver=2 WHERE id=1 AND ver=1`. (Success, Rows Updated: 1).
    4.  User B Updates -> `UPDATE table SET val=Y, ver=2 WHERE id=1 AND ver=1`. (Fail, Rows Updated: 0).
    5.  User B catches `OptimisticLockException` and retries.

## 4. Database Performance Issues

### 4.1 The N+1 Problem
**Issue**: Hibernate runs 1 query for Parent, then N queries for Children.
*   *Code*: `List<User> users = repo.findAll(); user.getOrders();`
*   *Result*: 1001 queries for 1000 users.
**Solution**: `JOIN FETCH`.
`@Query("SELECT u FROM User u JOIN FETCH u.orders")`

### 4.2 Connection Pool Exhaustion (HikariCP)
**Symptom**: `ConnectionTimeoutException`.
**Expert Tune**:
*   `maximumPoolSize`: rarely needs to be > 10-20 per node. CPU count * 2 is effective limit for Postgres.
*   `connectionTimeout`: Fast fail (e.g., 2000ms). Don't let threads queue for 30s.

```yaml
spring.datasource.hikari:
  maximum-pool-size: 20
  minimum-idle: 10
  connection-timeout: 3000 # 3 seconds
  idle-timeout: 600000
```
