# Enterprise API Design: Rate Limiting & DDoS Protection

## 1. The Problem: Uncontrolled Traffic & DDoS
In production, APIs face sudden spikes (flash sales) or malicious attacks (DDoS). Without protection, a single client can exhaust thread pools, database connections, or memory, crashing the system for everyone ("Noisy Neighbor" problem).

**Scenario**: You have an API limited to 5,000 requests/second (RPS) capacity. An attacker sends 50,000 RPS.

## 2. Solutions & Algorithms

### 2.1 Fixed Window Counter
*   **Concept**: A counter resets every minute.
*   **Pros**: Simple, low memory.
*   **Cons**: **The Boundary Issue**. If a user sends 5k requests at 11:59:59 and 5k at 12:00:01, they sent 10k requests in 2 seconds, potentially crashing the DB.
*   **Verdict**: Avoid for high-precision needs.

### 2.2 Token Bucket (Recommended for Bursts)
*   **Concept**: A bucket holds `N` tokens. Refills at rate `R` per second. Each request consumes 1 token. If empty, reject.
*   **Pros**: Allows short "bursts" of traffic (up to bucket size) while enforcing an average rate data.
*   **Implementation**: `Bucket4j`.

### 2.3 Leaky Bucket (Recommended for Smoothing)
*   **Concept**: Requests enter a queue (bucket) and are processed at a constant rate (drip). If queue is full, discard.
*   **Pros**: Smooths out traffic spikes (good for writing to a weak DB).
*   **Cons**: No bursts allowed.

### 2.4 Sliding Window Log (High Precision)
*   **Concept**: Store timestamp of every request. Count logs in the last `T` seconds.
*   **Pros**: Perfect accuracy.
*   **Cons**: High memory usage (storing millions of timestamps).

### 2.5 Sliding Window Counter (Best Balance)
*   **Concept**: Hybrid of Fixed Window + Rolling calculation.
*   **Formula**: `Requests = (Requests in Current Window) + (Requests in Prev Window * Overlap %)`.
*   **Pros**: 99% accuracy, low memory (Redis/Hazelcast).

---

## 3. Implementation Strategies

### Strategy A: Local Rate Limiting (Guava/Bucket4j)
**Use Case**: Monoliths or protecting a specific instance from CPU exhaustion.
**Limitation**: In a microservices cluster with 10 instances, the limit isn't shared. 10 instances x 100 RPS = 1000 RPS global.

### Strategy B: Distributed Rate Limiting (Redis + Lua)
**Use Case**: Microservices requiring a strict **Global** limit (e.g., "User X can only send 100 emails/day across all servers").
**Mechanism**:
1.  Use Redis to store counters.
2.  Use **Lua Scripts** to ensure atomicity (Check + Decrement happens in one step).

---

## 4. Production Code Implementation (Java + Spring Boot + Bucket4j + Redis)

### 4.1 Dependency (Maven)
```xml
<dependency>
    <groupId>com.giffing.bucket4j.spring.boot.starter</groupId>
    <artifactId>bucket4j-spring-boot-starter</artifactId>
    <version>0.8.1</version>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

### 4.2 Configuration (application.yml)
We define a rule: 5 access tokens per minute for the "pricing" API.

```yaml
bucket4j:
  enabled: true
  filters:
    - cache-name: buckets
      url: /api/v1/pricing.*
      strategy: first
      rate-limits:
        - bandwidths:
            - capacity: 5
              time: 1
              unit: minutes
```

### 4.3 Custom Filter Implementation (The Expert Way)
Sometimes annotation-based config isn't enough. You need dynamic limits (e.g., Premium users get 100 RPS, Free get 10 RPS).

```java
@Component
public class RateLimitFilter implements Filter {

    private final Map<String, Bucket> cache = new ConcurrentHashMap<>();

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) 
            throws IOException, ServletException {
        
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        String apiKey = httpRequest.getHeader("X-API-KEY");

        if (apiKey == null || apiKey.isEmpty()) {
            ((HttpServletResponse) response).sendError(401, "Missing API Key");
            return;
        }

        Bucket bucket = resolveBucket(apiKey);

        // Try to consume a token
        ConsumptionProbe probe = bucket.tryConsumeAndReturnRemaining(1);

        if (probe.isConsumed()) {
            // Add headers for client visibility
            HttpServletResponse httpResponse = (HttpServletResponse) response;
            httpResponse.setHeader("X-Rate-Limit-Remaining", String.valueOf(probe.getRemainingTokens()));
            chain.doFilter(request, response);
        } else {
            // 429 Too Many Requests
            HttpServletResponse httpResponse = (HttpServletResponse) response;
            httpResponse.setStatus(429);
            httpResponse.setHeader("Retry-After", String.valueOf(probe.getNanosToWaitForRefill() / 1_000_000_000));
            httpResponse.getWriter().write("Too Many Requests");
        }
    }

    private Bucket resolveBucket(String apiKey) {
        return cache.computeIfAbsent(apiKey, this::createNewBucket);
    }

    private Bucket createNewBucket(String apiKey) {
        // Dynamic Limit: Check DB if user is PERMIUM or FREE
        long capacity = isPremium(apiKey) ? 1000 : 20; 
        
        return Bucket.builder()
                .addLimit(Bandwidth.classic(capacity, Refill.greedy(capacity, Duration.ofMinutes(1))))
                .build();
    }
    
    private boolean isPremium(String key) {
        // Mock DB call
        return key.startsWith("prem_");
    }
}
```

### 4.4 Distributed Implementation (Redis)
For a cluster, using an in-memory `ConcurrentHashMap` (as above) is wrong. You must switch the `Bucket` storage to Redis.

```java
@Configuration
public class RedisBucketConfig {

    @Bean
    public ProxyManager<String> lettuceProxyManager(RedisClient redisClient) {
        // Use Lettuce or Jedis client
        return LettuceBasedProxyManager.builderFor(redisClient)
            .withExpirationStrategy(ExpirationAfterWriteStrategy.basedOnTimeForRefillingBucketUpToMax(Duration.ofMinutes(10)))
            .build();
    }
}
```

---

## 5. Defense in Depth (Network Layer)
Application-level rate limiting (Java) is for **logic** (pricing tiers). For massive **DDoS** (volumetric attacks), Java is too slow. You must block traffic *before* it hits the JVM.

1.  **Cloudflare / AWS WAF**: Block malicious IPs, Geo-blocking.
2.  **API Gateway (Kong / Nginx)**: Hard limits per IP.
3.  **Kubernetes Ingress**: Nginx Ingress Rate Limiting annotations.

use `nginx.ingress.kubernetes.io/limit-rps: "5"` for defense-in-depth.
