# Memory Management & JVM Garbage Collection Tuning

## 1. The Fear of OOM (Out Of Memory)
`java.lang.OutOfMemoryError` is the nightmare of Java production.

### 1.1 Types of OOM
1.  **Java Heap Space**: You are creating objects faster than GC can kill them. (Memory Leak or Load Spike).
2.  **Metaspace**: Too many classes loaded (Common in dynamic class generation frameworks like Spring/Hibernate if misconfigured).
3.  **Direct Buffer / Native Memory**: Netty/NIO off-heap memory exhaustion.

## 2. Analyzing a Memory Leak
**Symptoms**: Memory usage creates a "Sawtooth" pattern that keeps rising until it hits 100% and crashes.

**Tooling SOP**:
1.  **Heap Dump**: Capture the RAM state on crash.
    `java -XX:+HeapDumpOnOutOfMemoryError`
2.  **Analyzer**: Open `.hprof` file in **Eclipse Memory Analyzer (MAT)** or **VisualVM**.
3.  **Dominator Tree**: Find which object retains the most memory.
    *   *Common Culprit 1*: Static `Map` or `List` that grows forever (Caching without eviction).
    *   *Common Culprit 2*: Unclosed IO Streams / Connections.

## 3. Garbage Collection (GC) Tuning
**Goal**: Reduce **Stop-The-World (STW)** pauses. When GC runs, the app freezes.

### 3.1 Choosing the Right Collector
1.  **Serial GC**: Single-threaded. For small CLI apps.
2.  **Select Parallel GC**: Throughput focus. Good for Batch Jobs. (Pauses ok, just finish work fast).
3.  **G1 GC (Default Java 9+)**: Balanced. Splits Heap into regions. Good for typical microservices (<32GB Heap).
4.  **ZGC / Shenandoah (Java 17/21)**: **Low Latency**. Pauses < 1ms even on Terabyte Heaps.
    *   *Expert Move*: Upgrade to Java 21 and use ZGC (`-XX:+UseZGC`) for latency-sensitive APIs.

### 3.2 Basic Tuning Flags (Docker Friendly)
In containers, Java used to think it had the whole Host RAM.
**Java 10+ Container Awareness**:
`-XX:MaxRAMPercentage=75.0`: Use 75% of Docker limit for Heap, leave 25% for Metaspace/Stack/Native.

## 4. Off-Heap Memory (Expert Optimization)
For High-Frequency applications, Java Objects have overhead (Header = 12-16 bytes).
Storing 1 billion `Long`s = 24GB.
**Solution**: `ByteBuffer.allocateDirect()`. Stores raw bytes outside Heap. ZERO GC impact. Used by Cassandra, Kafka, Lucene.
