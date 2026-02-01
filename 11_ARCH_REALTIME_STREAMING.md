# System Design: Real-Time & Streaming Architectures

This document covers high-level architectural designs for Video Streaming (Netflix), Chat (WhatsApp), and Conferencing (Zoom).

---

## 1. Netflix (Video on Demand)
**Scale**: 250M+ Users, 15% of Global Internet Traffic.
**Core Requirement**: High Availability, Low Latency Streaming (No Buffering).

### 1.1 The Tech Stack
*   **Frontend**: React (TV UI), iOS/Android.
*   **Backend**: Java / Spring Boot Microservices.
*   **Database**: **Cassandra** (Wide Column Store) due to heavy write capability and multi-region replication.
*   **Messaging**: **Kafka** (Keystone pipeline) for analytics/recommendations.
*   **Storage**: AWS S3 (Master Files).
*   **CDN**: **Open Connect** (Custom Hardware embedded in ISPs).

### 1.2 Architecture
1.  **Onboarding (Transcoding)**:
    *   Producer uploads 1 Master File (4K, 50GB).
    *   **Archer** (Workflow Engine) triggers ~1200 jobs.
    *   File is chunked and converted to 50 formats (Bitrates: 480p, 1080p, 4k; Codecs: H.264, H.265; Audio: 5.1, Stereo).
2.  **Distribution (CDN)**:
    *   These chunks are pushed to **Open Connect** appliances at your ISP (e.g., Comcast/Verizon) during off-peak hours (Pre-fetching).
3.  **Playback**:
    *   Client calls `Playback API` (Edge Service).
    *   Service determines user's bandwidth and location.
    *   Returns URL of the *closest* Open Connect box (inside your city).

### 1.3 Key Design Pattern: "Cell-Based Architecture"
To avoid hitting limits, Netflix splits the backend into isolated "Cells" (e.g., one cell handles 100k users). If a cell crashes, only those users are affected.

---

## 2. WhatsApp (Real-Time Chat)
**Scale**: 2B+ Users, 100B Messages/Day.
**Core Requirement**: Real-time delivery, Low Overhead, Connectivity.

### 2.1 The Tech Stack
*   **Language**: **Erlang** (or Elixir). Why? Lightweight processes. A single server can handle **2 Million** concurrent connections.
*   **Protocol**: **XMPP** (Extensible Messaging and Presence Protocol) - Customized and binary-optimized to save bytes.
*   **Database**: **Mnesia** (Erlang's DB) for routing tables. **Cassandra** for message history (though mostly stored on device).
*   **OS**: FreeBSD (Hyper-tuned networking stack).

### 2.2 Architecture
1.  **Connection**:
    *   User holds a persistent TCP connection (Socket) to the Chat Gateway (Erlang).
2.  **Message Flow**:
    *   User A sends msg to User B.
    *   Server looks up User B's connection in **Mnesia** (Ephemeral Map: UserID -> SocketID).
    *   If User B is online: Push directly to socket.
    *   If Offline: Store in **SQLite/Cassandra** until Ack.
3.  **End-to-End Encryption**:
    *   Server *cannot* read messages. It just blindly routes bytes.
    *   Media (Images/Video) is uploaded to an HTTP Server (Blob Store), and only the *thumbnail + URL* is sent over XMPP.

---

## 3. Zoom (Video Conferencing)
**Scale**: 300M+ Participants.
**Core Requirement**: Low Latency (UDP), Multipoint communication.

### 3.1 The Tech Stack
*   **Protocol**: **WebRTC** (Browser), **RTP/UDP** (Real-time Transport Protocol). TCP is avoided because retransmitting lost packets causes "lag". In video, it's better to lose a frame than to pause.
*   **Codec**: H.264 SVC (Scalable Video Coding).
*   **Server**: Multimedia Router (MMR) written in C++/Rust.

### 3.2 Architecture: SFU (Selective Forwarding Unit)
*   **Mesh (Bad)**: Every user sends video to every other user. (N^2 connections). Uses too much bandwidth.
*   **MCU (Old)**: Server runs ffmpeg, mixes all 50 streams into 1 video, sends 1 video to everyone. (CPU intensive on Server).
*   **SFU (Zoom Design)**: 
    *   User A sends 1 stream to Server (High Quality).
    *   Server *intelligently forwards* that packet to Users B, C, D.
    *   **SVC**: User A actually sends 3 layers (Low, Med, High). Server sends "High" to Desktop users, "Low" to Mobile users.

### 3.3 Optimization
*   **Global Data Centers**: Zoom routes traffic through a dedicated private network, avoiding the public internet's congestion ("Dark Fiber").
