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

## database are not meant for polling or constant status update
Due to downsides:
- there is a delay between when data is written. and when it's read. If polling every 10 second, we are missing the real data in 10 second, that is latency
- put extra load on the database, not designed like message broker, slow down the overall system as more services and checks are added
- does not scale as app grows

A better way is to use event-driven messaging
- it is like switching from refreshing your inbox constantly to just getting notifications when email arrives.

  Great! Here's what that line means, broken down clearly:

---

### 📦 "Messages are persisted with retry and DLQ (dead letter queue) support" means:

#### ✅ **1. Messages are persisted**
- Messages sent to the cloud Queue are **stored reliably**.
- If the consumer service isn't available right away, the message is **not lost**.
- It stays in the queue until:
  - It's successfully processed.
  - It expires (based on retention period).
  - Or it's moved to the DLQ (more on that below).

Think of it like a **mailbox that keeps the letter safe** until someone picks it up.

---

#### 🔁 **2. Retry support**
- If a consumer **fails to process a message** (e.g. service crash, error), cloud can:
  - **Retry delivery** automatically.
  - You can configure the number of retries.
- This improves reliability — messages don’t just disappear if something goes wrong once.

Example:  
Your service tries to process a message but hits a network error → cloud can **retry a few times** before giving up.

---

#### 🪦 **3. DLQ = Dead Letter Queue**
- If a message **fails too many times** (after all retry attempts), it is moved to a **Dead Letter Queue**.
- This is like a **quarantine zone** for "bad" messages.
- You can inspect the DLQ later to:
  - Debug the issue.
  - Fix or reprocess failed messages.
  - Avoid data loss.

---

### 🧠 Why this matters:
Without this, you'd have to build your own logic to:
- Retry on failure.
- Track failed messages.
- Avoid losing important data.

Cloud Queue gives you all this **out of the box** — just set it up in the config.


  
