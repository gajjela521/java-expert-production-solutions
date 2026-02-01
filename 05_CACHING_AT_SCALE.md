# Caching Strategies at Scale (Redis/Memcached)

## 1. The Issue: "The Cache is Not Just a Map"
Junior developers treat Redis like a magic `HashMap`. In production (Amazon/Netflix scale), simple caching causes catastrophic failures.

## 2. The Big 3 Cache Problems

### 2.1 Cache Penetration (Data Missing)
**Issue**: A attacker queries `getProfile(id=-1)`. This ID definitely doesn't exist.
*   Cache: Miss.
*   DB: Miss.
*   **Result**: 100k requests bypass cache and hit DB directly, crashing it.
**Solution**: **Bloom Filters**.
Check if ID *might* exist before hitting Redis/DB. Or cache the "null" result with a short TTL (e.g., 5 min).

### 2.2 Cache Breakdown (Hot Key Expiry)
**Issue**: A "Hot Key" (e.g., "iphone-15-price") is hit 100k/sec. It expires at 12:00:00.
*   12:00:01: 10,000 requests see "Cache Miss" simultaneously.
*   All 10,000 hit the DB to calculating the price.
*   **Result**: Thundering Herd. DB dies.
**Solution**: **Mutex Lock / Logical Expiry**.
Only let *one* thread rebuild the cache.
```java
String value = redis.get(key);
if (value == null) {
    if (redis.tryLock(key + "_lock")) {
         value = db.get(key);
         redis.set(key, value);
         redis.unlock();
    } else {
         sleep(50); retry(); // Wait for the other thread
    }
}
```

### 2.3 Cache Avalanche (Mass Expiry)
**Issue**: You restart Redis, or 50% of your keys expire at the exact same time (e.g., default TTL 1 hour).
**Result**: DB gets hit by *everything* at once.
**Solution**: **Randomize TTL**.
Instead of `TTL = 60 min`, use `TTL = 60 min + Random(0-10 min)`. This spreads the re-computation load.

## 3. Caching Patterns

### 3.1 Cache-Aside (Lazy Loading) - Most Common
App checks Cache. If miss, App reads DB, updates Cache.
*   **Pros**: Only caches what is needed.
*   **Cons**: First request is slow. Data can be stale.

### 3.2 Write-Through & Write-Behind
*   **Write-Through**: App updates Cache, Cache updates DB synchronously. (Safe, consistent).
*   **Write-Behind**: App updates Cache, Cache updates DB asynchronously (fast but risk of data loss on crash).

## 4. Multi-Level Caching (L1/L2)
For extreme speed (High-Frequency Trading / AdTech), Network calls to Redis (2ms) are too slow.
**Solution**: **Caffeine (JVM Local Cache) + Redis (Distributed)**.

```java
// Spring Boot Cache Manager
@Bean
public CacheManager cacheManager() {
    // L1: Caffeine (Local RAM) - Fast access
    CaffeineCacheManager caffeine = ...; 
    // L2: Redis - Shared consistency
    RedisCacheManager redis = ...;
    return new CompositeCacheManager(caffeine, redis);
}
```
If L1 misses, check L2. If L2 misses, check DB.
