# System Design: Social Media & Discovery Architectures

This document covers architectures for Twitter (Feeds), Instagram (Media), and Tinder (Location/Discovery).

---

## 1. Twitter / X (Timeline & Feeds)
**Scale**: 400M MAU, Read-Heavy (Reading tweets >> Writing tweets).
**Core Problem**: The "Justin Bieber" Issue (Fanout).

### 1.1 The Tech Stack
*   **Language**: Scala (Logic), Java.
*   **Cache**: **Redis Clusters** (The backbone of Twitter).
*   **Database**: **Manhattan** (Custom distributed KV store), MySQL (User data), Snowflake (IDs).
*   **Search**: **Earlybird** (Built on Lucene).

### 1.2 Architecture: Feed Generation
1.  **Pull Model (Fanout-on-Read)** - *For Celebrities*:
    *   When User A follows Musk (celebrity), Musk's tweets are pulled from the DB at the moment A loads their feed.
2.  **Push Model (Fanout-on-Write)** - *For Normal Users*:
    *   When User B (normal) tweets, the system finds all 500 followers.
    *   It immediately inserts the Tweet ID into 500 **Redis Lists** (one for each follower).
    *   **Pros**: Reading the feed is O(1). Fast.
    *   **Cons**: Writing is slow if user has 10M followers.
3.  **Hybrid Approach**:
    *   Normal tweets are Pushed. Celebrity tweets are Pulled. The feed merges them at runtime.

### 1.3 Snowflake IDs
Twitter needs unique Sortable IDs across 1,000 servers.
**Solution**: 64-bit Integer.
*   1 bit: Sign.
*   41 bits: Timestamp (Milliseconds).
*   10 bits: Machine ID (Data Center + Node).
*   12 bits: Sequence Number (0-4095 per millisecond).

---

## 2. Instagram (Image Heavy Social)
**Scale**: 2B Users, Storage Intensive.
**Core Problem**: Efficiently storing and serving billions of photos with low latency.

### 2.1 The Tech Stack
*   **Backend**: Python (Django). Huge monolith.
*   **Database**: **PostgreSQL** (Sharded by User ID).
*   **Storage**: AWS S3.
*   **Cache**: Memcached (Billions of keys).
*   **GraphDB**: **TAO** (Facebook's custom graph store for "Who follows who").

### 2.2 Architecture: Sharding Photos
1.  **Logical Sharding**:
    *   Instagram runs thousands of Logical Shards mapped to Physical Postgres Servers.
2.  **Image Upload**:
    *   User uploads photo -> S3.
    *   S3 returns URL (`http://s3.../img_123.jpg`).
    *   App saves metadata (User ID, Lat/Lon, S3_URL) to Postgres.
3.  **Feed Construction**:
    *   Similar to Twitter (Push Model).
    *   Uses **Cassandra** for the "Activity Feed" (Likes/Comments) due to high write throughput.

---

## 3. Tinder (Location & Matchmaking)
**Scale**: 75M MAU, Geospatial Intensive.
**Core Problem**: Finding people "Near Me" efficiently.

### 3.1 The Tech Stack
*   **Backend**: Node.js / Java.
*   **Database**: DynamoDB (User Profiles).
*   **Geospatial**: **Elasticsearch** or **Redis (Geo)**.
*   **Gateway**: WebSocket (for instant Match notifications).

### 3.2 Architecture: Geospatial Indexing
1.  **Geohashing**:
    *   The world is divided into a grid. Strings represent rectangles.
    *   `dp3nb` represents a neighborhood in Toronto. `dp3` represents Ontario.
    *   **Key Idea**: Users sharing the same prefix are close to each other.
2.  **Query**:
    *   "Find users in `dp3nb`".
    *   Redis Command: `GEORADIUS user_locations -79.3 43.6 5 km`.
3.  **Sharding**:
    *   Shard 1 handles "North America". Shard 2 handles "Europe".
    *   Prevents one DB from handling global traffic.

### 3.3 The Recommendation Engine
*   When App opens, it pre-fetches 40 potential matches.
*   It filters out:
    *   Already Swiped (Bloom Filter).
    *   Outside Filters (Age/Distance).
*   Stores this "Deck" in Redis.
