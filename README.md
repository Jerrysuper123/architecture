# architecture

### 🔄 **Current Approach: Using SQL/NoSQL Tables to Communicate Between Microservices**

#### ✅ What's Happening
You're currently:
- Creating **custom database tables**.
- One service writes data to a table.
- Another service **polls** the table to read/process the data.
- Doing this for each new integration or feature.

#### ❌ Why It’s Painful
- **Not scalable**: Every new feature or integration = new table/schema changes.
- **High latency**: Polling DBs introduces unnecessary load and delays.
- **Complex maintenance**: Managing retries, failure states, and cleanup manually.
- **Tight coupling**: Services know too much about each other’s data structures.

---

### ✅ **Proposed Improvement: Using Cloud Queue for Inter-Service Communication**

Cloud **Queue** is part of the **Cloud Messaging Service**, designed for **asynchronous, decoupled communication** between microservices.

---

### 🔁 What Cloud Queue Brings to the Table

| Feature | Benefit |
|--------|--------|
| **Message-based** | Services send/receive discrete messages instead of managing tables. |
| **Decoupled** | Producers and consumers don’t need to know each other’s structure. |
| **Scalable** | Add new queues instead of tables; queues are managed resources. |
| **Durable & Reliable** | Messages are persisted with retry and DLQ (dead letter queue) support. |
| **No polling delays** | Supports long-polling and near real-time delivery. |
| **Security & IAM** | Integrated with Cloud IAM policies for secure access control. |

---

### 🧠 Typical Flow Using Cloud Queue

1. **Service A (Producer)** publishes a message to an Cloud Queue.
2. **Service B (Consumer)** reads messages from the queue asynchronously or via polling.
3. Queue ensures **at-least-once delivery**, optionally FIFO.
4. Messages are **acknowledged**, retried, or moved to **DLQ** if not processed.

---

### 🔄 SQL Table vs Cloud Queue: A Quick Visual

```
[Service A] --> INSERT INTO table --> [Service B polls table] (delay, overhead)

vs

[Service A] --> PUT Message in Cloud Queue --> [Service B gets message] (event-driven)
```

---
