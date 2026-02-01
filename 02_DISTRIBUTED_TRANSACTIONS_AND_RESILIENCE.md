# Distributed Transactions & Resilience in Microservices

## 1. The Issue: "The Network is Unreliable"
In a Monolith, `@Transactional` guarantees ACID. In Microservices, you cannot use ACID across two databases. A failure in Service B after Service A committed leaves data consistent.

**Scenario**:
1.  **Order Service**: Creates Order #123 (Committed).
2.  **Payment Service**: Charges Customer (Failed).
3.  **Result**: Order exists, but not paid. Inconsistent state.

## 2. Solutions

### 2.1 Two-Phase Commit (2PC) / XA
*   **Concept**: A Coordinator tells all DBs to "Prepare", then "Commit".
*   **Verdict**: **Avoid in Microservices**. It locks rows in all DBs, causing massive performance bottlenecks and deadlocks.

### 2.2 The Saga Pattern (Standard)
A sequence of local transactions. If one fails, execute **Compensating Transactions** to undo changes.

#### A. Choreography (Event-Driven)
Services listen to events.
*   *Flow*: Order Created Event -> Payment Service listens -> Payment Failed Event -> Order Service listens -> Cancel Order.
*   *Pros*: Decoupled.
*   *Cons*: Hard to track "who does what" in complex flows (Cyclic dependencies).

#### B. Orchestration (Command-Driven) -> **Recommended for Enterprise**
A central "Orchestrator" (State Machine) tells services what to do.
*   *Flow*: Order Orchestrator calls Payment Service. If fail, Orchestrator calls "Undo Order".
*   *Tools*: Camunda, Netflix Conductor, Uber Cadence, AWS Step Functions.

### 2.3 Implementation: Orchestrated Saga with Spring Boot

**Step 1: Define steps in code (State Machine)**

```java
public class OrderSagaOrchestrator {
    
    public void createOrder(OrderRequest req) {
        // Step 1: Local Transaction
        Order order = orderService.createPendingOrder(req);
        
        try {
            // Step 2: Remote Call (Payment)
            paymentClient.charge(req.getCard(), req.getAmount());
            
            // Step 3: Remote Call (Inventory)
            inventoryClient.reserve(req.getItems());
            
            // Success: Confirm Order
            orderService.confirmOrder(order.getId());
            
        } catch (Exception e) {
            // FAILURE DETECTED: Trigger Compensation
            compensate(order);
        }
    }
    
    private void compensate(Order order) {
        // Undo Payment (Refund)
        paymentClient.refund(order.getId());
        
        // Undo Order (Cancel)
        orderService.failOrder(order.getId());
    }
}
```

> **Expert Note**: The orchestrator *itself* can crash mid-process. You must persist the "State" (e.g., "Step 2 started") in a database so it can resume after restart. This is why tools like **Camunda** are used.

## 3. Idempotency (The Hidden Critical Requirement)
If you retry a network call (due to timeout), you might charge the customer twice. **All writes must be Idempotent.**

**Solution**: Clients send a unique `Idempotency-Key` header (UUID).

```java
@RestController
public class PaymentController {
 
    @PostMapping("/charge")
    public ResponseEntity charge(@RequestHeader("Idempotency-Key") String key, @RequestBody ChargeRequest req) {
        
        // 1. Check Redis/DB if key exists
        if (transactionRepo.existsByKey(key)) {
            return ResponseEntity.ok(transactionRepo.findByKey(key));
        }
        
        // 2. Process Charge
        Transaction tx = stripeService.charge(req);
        
        // 3. Save Result with Key
        transactionRepo.save(key, tx);
        
        return ResponseEntity.ok(tx);
    }
}
```

## 4. Resilience Patterns (Resilience4j)

### 4.1 Circuit Breaker
Stop calling a failing service to give it time to recover.

```java
@CircuitBreaker(name = "inventoryService", fallbackMethod = "fallbackInventory")
public InventoryResponse getStock(String productId) {
    return remoteClient.getStock(productId);
}

// Fallback: Default behavior when down
public InventoryResponse fallbackInventory(String productId, Throwable t) {
    return new InventoryResponse(productId, 0); // Assume out of stock to be safe
}
```

### 4.2 Bulkhead Pattern
Isolate resources. Don't let one slow service exhaust all Tomcat threads.
*   **Concept**: Service A gets 10 threads. Service B gets 10 threads. If Service A hangs, Service B still works.

```yaml
resilience4j.bulkhead:
  instances:
    backendA:
      maxConcurrentCalls: 10
```

### 4.3 Retry with Exponential Backoff
Don't retry immediately (spamming). Wait 1s, then 2s, then 4s.

```yaml
resilience4j.retry:
  instances:
    backendA:
      maxAttempts: 3
      waitDuration: 500ms
      enableExponentialBackoff: true
      exponentialBackoffMultiplier: 2
```
