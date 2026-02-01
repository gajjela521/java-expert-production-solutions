# System Design: Data Intensive & Logistics

This document covers architectures for Uber (Logistics), Amazon (Inventory), Dropbox (File), and Google Crawler.

---

## 1. Uber (Real-Time Logistics)
**Scale**: 100M MAU, 14M Trips/Day.
**Core High-Level**: Matching Riders to Drivers within seconds under highly dynamic conditions.

### 1.1 The Tech Stack
*   **Language**: Go (High throughput), Java (Business Logic), Python (ML).
*   **Database**: **Schemaless** (Custom sharded MySQL layer), Cassandra.
*   **Geospatial**: **Google S2 Geometry** (Spherical hierarchy) instead of Geohash.
*   **Protocol**: **Ringpop** (Consistent Hashing / Gossip Protocol).

### 1.2 Architecture: The Dispatch System (DISCO)
1.  **Driver Tracking**:
    *   Driver app sends GPS every 4 sec.
    *   It hits the **Location Service**.
    *   Instead of writing to DB (too slow), it updates an **In-Memory Geo-Index** (QuadTree) in Redis.
2.  **Matching (Demand/Supply)**:
    *   Rider requests car.
    *   System calculates `S2 Cell ID` (e.g., Level 12 ~ 2km radius).
    *   Queries `Supply Service`: "Give me drivers in Cell X".
    *   Runs matching algo (ETA, Rating, Route).
3.  **Consistent Hashing (Ringpop)**:
    *   To scale, Node A handles San Francisco. Node B handles New York.
    *   Ringpop allows nodes to "Gossip" membership. If Node A dies, Node C takes over SF automatically.

---

## 2. Amazon / E-Commerce (High Consistency)
**Core Problem**: Inventory Management. You have 1 item left. 100 people click "Buy". Only 1 must succeed.

### 2.1 The Tech Stack
*   **Backend**: Java / Scala.
*   **Database**: **DynamoDB** (Key-Value), Oracle (Legacy), RDS.
*   **Architecture**: Service-Oriented (SOA). Not just microservices, but thousands of tiny services.

### 2.2 Architecture: Handling "The Click"
1.  **Eventual Consistency (Shopping Cart)**:
    *   Adding to cart is "Always Writeable".
    *   Uses **Vector Clocks** to merge conflicts (e.g., User adds Item A on Mobile, Item B on Desktop).
2.  **Strong Consistency (Checkout)**:
    *   When clicking "Place Order", the system must lock the inventory.
    *   **Pessimistic Locking** (DB Lock) or **Optimistic Locking** (`version_id`).
    *   Usually: Dequeue inventory -> If count < 0 -> Rollback -> "Sorry, out of stock".
3.  **Search**:
    *   Inverted Index (Elasticsearch). Faceted Search (Filter by Color, Size).

---

## 3. Dropbox (File Storage & Sync)
**Core Problem**: Syncing 1TB of data efficiently.

### 3.1 The Tech Stack
*   **Language**: Rust (Core Sync Engine), Python, Go.
*   **Storage**: **Magic Pocket** (Custom Exabyte-scale block storage), AWS S3 (Cold).
*   **Database**: **Edgestore** (Distributed Meta-data store).

### 3.2 Architecture: Chunking & Deduplication
1.  **The Client (Desktop)**:
    *   Splits a 1GB file into **4MB Blocks**.
    *   **Hashes** each block (SHA-256).
    *   Checks with Server: "Do you have hash `abc`?".
    *   If Server says "Yes" (someone else uploaded this movie), Client *skips* upload (Instant Deduplication).
2.  **Metadata vs Data**:
    *   **Metadata DB**: Stores "File path `/homework.pdf` is made of blocks `[hash1, hash2]`".
    *   **Block Store**: Stores the raw bytes of `hash1`.
3.  **Sync**:
    *   Client uses a "Long Poll" to Notification Server.
    *   If a file changes, Client downloads only the *modified blocks* (Delta Sync).

---

## 4. Google Crawler (Web Search)
**Core Problem**: Downloading the entire internet (billions of pages).

### 4.1 Architecture
1.  **URL Frontier** (The Queue):
    *   Prioritizes URLs. (News sites = refresh every 5 min. Old blogs = every month).
    *   Politeness Enforcer: Don't hit `cnn.com` 1000 times/sec. One request per domain every X sec.
2.  **The Fetcher**:
    *   Downloads HTML.
    *   **Custom DNS Resolver**: Standard DNS is too slow. Google caches DNS for millions of domains.
3.  **Bloom Filter**:
    *   "Have I crawled this URL before?"
    *   Uses a massive Bloom Filter (Bit array) to check existence in O(1) RAM.
4.  **BigTable**:
    *   Stores the content map: `Row Key: URL`, `Columns: Content, Metadata, Links`.
