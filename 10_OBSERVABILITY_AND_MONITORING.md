# Observability & Monitoring: The "Eyes" of Production

## 1. Logging vs Metrics vs Tracing

### 1.1 Logging (The "What")
Detailed events.
*   *Anti-Pattern*: `System.out.println` or logging to a file on a ephemeral container (lost on restart).
*   *Structured Logging*: Log in JSON.
    ```json
    {"timestamp": "...", "level": "ERROR", "traceId": "abc", "msg": "Payment Failed"}
    ```
*   **Tech**: ELK Stack (Elasticsearch, Logstash, Kibana) or Loki.

### 1.2 Metrics (The "Health")
Aggregated numbers. "CPU is 90%", "RPS is 500".
*   **Tech**: Prometheus (Scraper) + Grafana (Dashboard).
*   **RED Method**: Rate (RPS), Errors (%), Duration (Latency).

### 1.3 Distributed Tracing (The "Where")
In microservices, a request hits Gateway -> Auth -> Order -> Payment. If it's slow, *where* is it slow?
*   **Tech**: OpenTelemetry / Zipkin / Jaeger.
*   **Trace Context**: Pass a `trace-id` header in every HTTP/Kafka call.

## 2. Implementing OpenTelemetry in Spring Boot

### 2.1 Dependencies
```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-brave</artifactId>
</dependency>
<dependency>
    <groupId>io.zipkin.reporter2</groupId>
    <artifactId>zipkin-reporter-brave</artifactId>
</dependency>
```

### 2.2 How it works
1.  Spring automatically generates a Trace ID (e.g., `65b9...`)
2.  It injects it into MDC (logging context).
3.  Your logs now allow filtering by `traceId`.
4.  It sends spans to Zipkin server to visualize the waterfall graph.

## 3. Alerting Levels
Don't alert on everything.
1.  **Ticket (P3)**: "Disk is 80% full" (Wake up tomorrow).
2.  **Page (P1)**: "Site is Down" or "Error Rate > 5%". (Wake up NOW).
3.  **Golden Signals**: Latency, Traffic, Errors, Saturation.

## 4. Health Checks
Expose `/actuator/health` (Spring Boot).
*   **Liveness**: "Am I running?" (If no, K8s restarts pod).
*   **Readiness**: "Can I take traffic?" (If no, K8s removes from Load Balancer).
    *   *Check*: can I connect to DB? No? Then I am not ready.
