# architecture


## CLOB - Character Large Object - not storing in the sql table, but store a pointer to a separate location on the disk

If you're storing your large text data as a **CLOB** and plan to **process, clean, and present it in a dashboard**, here’s a breakdown of what to expect in terms of **query performance** and practical implications:

---

### ⚡ Is CLOB fast for querying?

**Short answer**:
❌ **No, CLOBs are not fast for traditional querying** like filtering, searching, or joining — especially not at the scale of 10,000+ lines per row.

CLOBs are designed for **storing large text** efficiently, not for fast parsing or searching in SQL. Access is typically **sequential and stream-based**, not optimized for in-place filtering or transformation.

---

### 🔍 Performance Limitations

| Operation                    | Performance with CLOB                                  |
| ---------------------------- | ------------------------------------------------------ |
| Simple retrieval (`SELECT`)  | 🟡 OK, but slower than `TEXT`/`VARCHAR` for large rows |
| Search (`LIKE`, regex)       | 🔴 Slow or unsupported without full-text indexing      |
| Joins/Filters on content     | 🔴 Very inefficient or not allowed directly            |
| Aggregations on content      | 🔴 Not practical inside SQL                            |
| Application-level processing | 🟡 Reasonable if streamed and cached properly          |

---

### 🧠 What’s the typical workflow with CLOB in your case?

You want to:

1. **Store** large text data (10,000+ lines).
2. **Clean/process** it (e.g., remove noise, format).
3. **Present results** on a dashboard.

In this case, a better pattern would be:

---

### ✅ Recommended Architecture

1. **Store raw text as a CLOB or large TEXT column** (depending on your DB).
2. Use a **background process** (Python, Node.js, Java) to:

   * Read and parse the CLOB.
   * Extract structured data (e.g., error codes, summaries, metrics).
   * Store **cleaned data in structured tables** (`rows`, `numbers`, `JSON`, etc.).
3. Build your **dashboard** from the **processed structured data**, not directly from the CLOB.

This avoids the bottleneck of querying CLOBs directly, and lets you build fast, indexed dashboards.

---

### 🚫 Example of what *not* to do:

```sql
SELECT * FROM logs WHERE log_clob LIKE '%timeout%';
-- very slow or not supported
```

---

### ✅ Example of a better setup:

| Table: raw\_logs     | Table: extracted\_metrics |
| -------------------- | ------------------------- |
| `id`                 | `log_id`                  |
| `log_content (CLOB)` | `error_count`             |
| `created_at`         | `timeout_occurred (bool)` |
| ...                  | `processed_at`            |

Then your dashboard reads from `extracted_metrics`, which is fast.

---


- not be confused with Blob - binary large object, which stores file jpg, pdf as binary, different from text/clob
A **CLOB** (Character Large Object) is a **SQL data type** designed to store **very large amounts of text data**—often much larger than a standard `VARCHAR` or `TEXT` field can handle.

---

### 🔍 What is a CLOB?

* **CLOB** stands for **Character Large Object**.
* It's used for **storing large plain text**, such as:

  * Articles
  * Code dumps
  * Log files
  * Documents
* Typically supports **gigabytes** of data, depending on the database.
* Stored in a way that's optimized for **sequential reading**, not frequent random access.

---

### 💡 Key Characteristics

| Feature           | Value                                                                    |
| ----------------- | ------------------------------------------------------------------------ |
| Data type         | Stores **character data** (e.g., UTF-8, UTF-16)                          |
| Max size          | Often **up to 2–4 GB**, varies by DB engine                              |
| SQL operations    | Can be read/written, but often **not searchable with `LIKE`** by default |
| Access method     | Often accessed as a **stream** in code, not fully loaded into memory     |
| Use case vs. BLOB | Like BLOB, but for **text**, not binary                                  |

---

### 🏗 Examples by Database

| Database   | Large Text Type    | Notes                                              |
| ---------- | ------------------ | -------------------------------------------------- |
| Oracle     | `CLOB`             | Native support; also supports `NCLOB` for Unicode  |
| PostgreSQL | `TEXT` (no `CLOB`) | `TEXT` is unlimited, used instead of `CLOB`        |
| MySQL      | `LONGTEXT`         | No `CLOB` keyword, but `LONGTEXT` behaves like one |
| SQL Server | `VARCHAR(MAX)`     | Modern replacement for `TEXT` and `CLOB`           |
| SQLite     | `TEXT`             | Flexible typing; no distinct `CLOB` type needed    |

---

### 🧠 When to Use CLOB

Use a `CLOB` (or its equivalent) when:

* You need to store **very large amounts of text**.
* You don't need **fast substring search** inside SQL (or you're okay with workarounds).
* You're primarily reading/writing the text **via application code**, not inline SQL.

---


### 🔍 **How a CLOB works under the hood**

A `CLOB` (Character Large Object) is a way for a database to store **very large text data** separately from the main table structure. Here's how it generally works:

---

### 🧱 1. **Storage Mechanism**

* **Text Format**: A CLOB stores **text, not binary**. It’s encoded in a character set (e.g., UTF-8, UTF-16), depending on your DB and column settings.
* **Out-of-Row Storage**: Most databases **do not store the full CLOB content in the table row**—only a pointer or reference. The actual content is kept in a separate location on disk (like a LOB segment or file).
* **Streaming**: CLOBs are often **streamed in and out** rather than read fully into memory. This avoids loading multi-MB or GB text into RAM at once.

---

### 🔄 2. **Reading/Writing Behavior**

* **In SQL**: You can write to a CLOB using `INSERT`/`UPDATE` or dedicated streaming APIs.
* **In Applications**: Languages like Java or Python provide stream-based interfaces (e.g., `Clob.setCharacterStream()` or `read()`) to avoid memory overuse.
* **Searching**: Searching a CLOB with `LIKE` or full-text search usually requires special indexing (or isn’t supported at all without it).

---

### 🗜️ 3. **Compression**

* **CLOBs are not always compressed by default.**
* It depends on the **database and configuration**:

| Database   | Compression Support for CLOB                                          |
| ---------- | --------------------------------------------------------------------- |
| Oracle     | ✅ Yes, with `SecureFiles` LOBs and compression enabled                |
| PostgreSQL | ✅ Yes, if using TOAST compression for large `TEXT` fields             |
| MySQL      | ❌ Not by default for `TEXT`, `LONGTEXT` (no built-in LOB compression) |
| SQL Server | ✅ Yes, if using `FILESTREAM` or `COMPRESS()` manually                 |
| SQLite     | ❌ No built-in compression (you must compress manually before insert)  |

* **Custom compression**: In some cases, applications compress the text (e.g., using `gzip`) and store it in a `BLOB`, but that loses searchability and readability.

---

### ⚠️ Considerations

* **Searchability**: CLOBs can’t always be searched or indexed efficiently unless your DB offers special full-text indexing.
* **Performance**: Good for write-once, read-occasionally cases. Not great for frequent edits or updates.
* **Encoding**: Since it stores text, encoding consistency (UTF-8 etc.) is crucial. Misaligned encodings can corrupt reads.

---

### ✅ Summary

* A **CLOB stores very large text**, often in a separate area on disk.
* It uses **character encoding**, not raw binary.
* **Compression** depends on DB engine and settings (sometimes manual).
* Best used when you need to store **large text blobs**, but **don't need frequent text processing** in SQL.



## OCI resource principal
What is resource principal (RP)
- RP could be db, virtual machine, functions, OKE container
- Policy permission: Allow my DB with a specific id in the compartment to access my key
- How to identify? Use Resource Principal Session Token (RPST) comes into play: a short-lived token that asserts the identity of a specific resource type, such as 'I am a resource of type container with ID ocid1.container.111111111111111.' With RPST, resources can securely make API calls under their own identity

https://blogs.oracle.com/cloud-infrastructure/post/behind-the-scenes-iam-oci-resource-principals


Good question — in **microservices**, **data loss** isn't common per se, but it's definitely a **risk**, especially during **inter-service communication** or **synchronization**, depending on how things are designed.

Here are some **common causes of data loss or inconsistency** between components:

---

### 🔄 **1. Sync Failures (Async Communication)**
- If you use message brokers like Kafka, RabbitMQ, or SNS/SQS for async communication and:
  - A message gets published, but the consumer crashes or fails to process it
  - No proper **acknowledgement** / **retry** strategy is in place
  - Messages are not **durable/persisted**
  
💡 **Solution**: Use **durable queues**, **retry logic**, and **dead-letter queues (DLQs)** to capture failed events.

---

### ⚡ **2. Synchronous API Failures**
- Service A calls Service B via REST/gRPC, but:
  - Service B is down
  - There's a timeout
  - There’s a network partition

If Service A proceeds thinking the call succeeded, you get **partial updates** → classic **data inconsistency**.

💡 **Solution**: Use patterns like **circuit breakers**, **retries with backoff**, or **sagas** for better resilience.

---

### ⚙️ **3. Lack of Idempotency**
- If retries are not idempotent (e.g. inserting a row again instead of updating), you can get data corruption or duplication.

💡 **Solution**: Ensure **idempotent operations** (same request → same result, no side effects).

---

### 📦 **4. No Distributed Transaction Handling**
- Microservices typically avoid distributed transactions (2PC) because of complexity and tight coupling.
- Without a strategy like **Saga pattern**, one service might commit a change, but the others don’t → inconsistent data state.

💡 **Solution**: Implement **saga orchestration** or **choreography** to handle failures across multiple services in a business process.

---

### 📜 **5. Versioning or Schema Drift**
- If one service updates its schema but another isn't updated, you might silently drop fields or misinterpret data.

💡 **Solution**: Use **backward/forward-compatible schemas** (e.g. Avro, Protobuf) and enforce **versioning discipline**.

---

### ✅ TL;DR:
- **Data loss isn’t expected**, but can happen due to **network issues**, **bad error handling**, or **missing retry/durability mechanisms**.
- Prevent it using: **durable messaging**, **idempotency**, **saga patterns**, and **robust error handling**.

---


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

## Queue and channel
- queue is queue of message to be processed
- channels are paths for task/message to be filtered and delegate

# what is cloud stream
- like kafka - store vent logs in partition
- services can replay and consume independantly
- highly throughput
- event driven architecture, large scale ingestion

# what is cloud topic (notification)
- it is a pub/sub model
- push msg to
- email
- sms
- https endpoint
- functions
- Useful for sending alerts, broadcasting updates, triggering actions.

Great follow-up! Let's break it down simply:

---

## 🧠 What is **Kafka**?

**Apache Kafka** is an **open-source distributed event streaming platform**.

### In simple terms:
It’s like a **giant real-time logbook** where systems can:
- ✍️ **Write** events (producers)
- 📖 **Read** events (consumers)
- 📂 Keep events for a while (retention)

It's used to **decouple** services, handle **massive volumes of data**, and enable **real-time data pipelines**.

---

## 📦 Kafka Core Concepts (at a glance)

| Concept | Description |
|--------|-------------|
| **Producer** | App that writes (publishes) messages |
| **Consumer** | App that reads (subscribes to) messages |
| **Topic** | Named channel of messages (like a stream or folder) |
| **Partition** | A topic can be split into partitions for parallelism |
| **Offset** | A message's position in a partition |
| **Broker** | A Kafka server that stores and serves messages |
| **Consumer Group** | A group of consumers that work together to process messages |

---

## 📈 Why Do People Use Kafka?

| Reason | Benefit |
|--------|---------|
| **High throughput** | Can handle millions of events/sec |
| **Real-time** | Low-latency message processing |
| **Durable** | Messages are stored safely (disk-backed) |
| **Replayable** | Consumers can re-read old messages |
| **Scalable** | Add more brokers, partitions, consumers |
| **Decoupled systems** | Producers and consumers don’t need to know about each other |

---

## 🧪 Example: E-commerce App

1. **User places order** → event sent to `orders` topic
2. **Inventory service** consumes `orders` topic → updates stock
3. **Billing service** consumes `orders` topic → processes payment
4. **Email service** consumes `orders` topic → sends confirmation

They all subscribe to the same topic, but work independently.

---

## ☁️ Kafka in the Cloud

Instead of managing Kafka yourself (which can be complex), cloud platforms offer **managed Kafka-like services**:

| Cloud | Kafka Equivalent |
|-------|------------------|
| **Oracle Cloud (OCI)** | **OCI Streaming (Kafka-compatible)** |
| **AWS** | Amazon MSK |
| **Azure** | Azure Event Hubs |
| **Google Cloud** | Confluent Cloud / PubSub |

---

## 🧠 TL;DR:

> Kafka is a **high-performance, durable, event streaming system** used to connect services and process data in real time.



  
