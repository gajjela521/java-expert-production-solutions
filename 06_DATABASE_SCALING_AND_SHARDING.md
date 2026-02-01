# Database Scaling: Sharding, Partitioning & Replication

## 1. The Limit of Vertical Scaling
Initially, you upgrade your DB Server (64GB RAM -> 128GB -> 256GB). Eventually, keeping a 10TB table in one Node becomes impossible (Indexing slows down, backups take days). You must scale **Horizontally**.

## 2. Read Scalability: Master-Slave Replication
**Concept**: 1 Master (Writes), N Slaves (Reads).
**Challenge**: **Replication Lag**.
*   User updates profile (Write to Master).
*   User refreshes page (Read from Slave).
*   Lag is 500ms. Slave gives old data. User panics ("My update failed!").
**Solution**: **Pinning / Sticky Session**.
After a write, force reads from Master for X seconds. Or force the specific user ensuring consistency to read from Master (e.g., "Read-your-own-writes").

## 3. Write Scalability: Sharding (Partitioning)
Split data across multiple physical servers.

### 3.1 Horizontal Sharding Strategies
**Key**: `user_id`.
1.  **Range Based**: IDs 1-1M -> DB1, 1M-2M -> DB2.
    *   *Problem*: Hotspots. If most users are new (high IDs), DB2 dies while DB1 is idle.
2.  **Hash Based**: `shard_id = user_id % 4`.
    *   *Problem*: **Resharding**. If you add DB5, `id % 4` changes to `id % 5`. MILLIONS of rows must move servers. Consistent Hashing reduces this.

### 3.2 Directory Based (Lookup Service)
A separate service keeps a map: `User A -> Shard 1`. Flexible but complicates architecture.

### 3.3 The "Joined Query" Problem
Once sharded, **you cannot do JOINS** across tables on different servers.
**Solution**:
1.  **Denormalization**: Duplicate data (e.g., store `user_name` inside `orders` table so you don't need to join User DB).
2.  **App-Side Joins**: Fetch Users, fetch Orders, combine in Java Stream.

## 4. Partitioning (Postgres/MySQL Native)
Within a single server, split a large table into smaller files.
*   **Time Partitioning**: `orders_2023_01`, `orders_2023_02`.
*   **Benefits**: When deleting old data, you just `DROP TABLE orders_2021`, which is instant (vs `DELETE FROM ...` which locks rows and bloats Wal Logs).

## 5. CAP Theorem Reality
In distributed systems (like Sharded DBs), you choose:
*   **CP (Consistency)**: If a node disconnects, refuse writes. (Banking).
*   **AP (Availability)**: If a node disconnects, accept writes but sync later. (Social Media Likes).
**Note**: Relational DBs usually favor CP. NoSQL (Cassandra/Dynamo) favors AP.
