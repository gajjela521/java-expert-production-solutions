# Handling Large Payloads & Data Transfer at Scale

## 1. The Production Issue: The "Mega-Object" Failure
**Scenario**: Service A needs to send a list of 100,000 Users to Service B for processing.
**Naive Approach**: Service A fetches all users, builds a massive `List<User>` (500MB+ JSON), and makes a single HTTP POST to Service B.

**Failures Encountered**:
1.  **Network Timeout**: The request takes 20s to serialize/deserialize, triggering Service A's HTTP client timeout (e.g., `ReadTimeout` is 5s).
2.  **OOM (Out of Memory)**: Service B tries to load the entire JSON into RAM to parse it. If 10 requests hit simultaneously, Service B crashes with `OutOfMemoryError`.
3.  **Gateway Rejections**: Nginx/AWS API Gateway has a hard payload limit (e.g., 10MB or 6MB for Lambda), rejecting the request instantly (413 Payload Too Large).

## 2. Solution 1: Pagination & Batching (The User's Strategy)
Instead of sending 1 big object, split the data into manageable "chunks" or "batches" (e.g., 1,000 records per request).

### Implementation Pattern
**Sender (Service A)**:
1.  Count Total Records.
2.  Loop until all data is sent.

```java
public void sendUsersInBatches(List<User> allUsers) {
    int batchSize = 1000;
    for (int i = 0; i < allUsers.size(); i += batchSize) {
        int end = Math.min(allUsers.size(), i + batchSize);
        List<User> batch = allUsers.subList(i, end);
        
        // Retry logic is critical here! If Batch 5 fails, we must know.
        retryTemplate.execute(context -> {
            serviceBClient.processUsers(batch);
            return null;
        });
    }
}
```

**Receiver (Service B)**:
Service B exposes an endpoint that accepts a List. It processes 1,000 records quickly (e.g., 200ms) and returns 200 OK. This keeps connections short and memory usage low.

**Pros**: Simple, robust, works with standard HTTP.
**Cons**: Transactionality is lost. If Batch 1 succeeds but Batch 2 fails, the system is in a partial state. (Requires Idempotency/Saga).

## 3. Solution 2: The "Claim Check" Pattern (Enterprise Standard)
Used when the payload is massive (Images, Video, 1GB CSV).

**Concept**: Don't send the *data* through the internal network (Service Mesh). Store the data in a "Blob Store" (S3/GCS) and send a *reference* (Claim Check).

**Flow**:
1.  **Service A**: Uploads `users_export_123.json` to S3 bucket.
2.  **Service A**: Sends message to Service B (via Kafka or HTTP): `{ "jobId": "123", "s3Url": "s3://bucket/users_export_123.json" }`.
3.  **Service B**: Receives message. Streams the file from S3 line-by-line (Streaming Parsing).

**Why**: S3 is designed for massive throughput; your HTTP microservice is not.

## 4. Solution 3: Streaming (gRPC / HTTP Chunked Transfer)
If you need real-time flow without temporary storage.

**Method**: Use **Jackson Streaming API** or **gRPC Streams**.
Instead of loading the whole list, you serialize one object, write to `OutputStream`, flush, repeat. Service B reads from `InputStream` token by token.

**Java (Receiver)**:
```java
@PostMapping("/stream")
public void ingestUsers(InputStream inputStream) {
    JsonParser parser = jsonFactory.createParser(inputStream);
    while (parser.nextToken() != JsonToken.END_ARRAY) {
        User user = objectMapper.readValue(parser, User.class);
        process(user); // Process 1 by 1. Zero memory overhead.
    }
}
```
