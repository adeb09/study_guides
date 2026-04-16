# Database Design – Senior Engineer Interview Prep Guide

> System architecture discipline, not just schema design. Dense, concrete, no fluff. Real trade-offs, real database names, real consequences.

---

## Table of Contents

1. [Data Modeling Paradigms](#1-data-modeling-paradigms)
2. [Consistency, Availability, and Partition Tolerance](#2-consistency-availability-and-partition-tolerance)
3. [Normalization vs. Denormalization](#3-normalization-vs-denormalization)
4. [Scaling Strategies](#4-scaling-strategies)
5. [Indexing Strategies](#5-indexing-strategies)
6. [OLTP vs. OLAP vs. HTAP](#6-oltp-vs-olap-vs-htap)
7. [Event Sourcing and CQRS](#7-event-sourcing-and-cqrs)
8. [Modern Trends in 2026](#8-modern-trends-in-2026)
9. [Schema Migrations in Production](#9-schema-migrations-in-production)
10. [Interview Drill – 5 System Design Scenarios](#10-interview-drill--5-system-design-scenarios)

---

## 1. Data Modeling Paradigms

### The Core Decision

Your data model is not a storage detail — it is an architectural contract. Choosing the wrong paradigm means you either fight the database constantly at scale, or you re-architect under production traffic. Get this right early.

---

### Relational (PostgreSQL, MySQL, CockroachDB, Spanner)

**Mental model:** Everything is a table. Relationships are expressed as foreign keys and resolved via joins. The schema is the source of truth for structure.

**Where it shines:**
- Multi-entity transactions with strict ACID guarantees (financial ledgers, inventory, reservations)
- Complex ad-hoc queries that you can't fully anticipate at design time
- Strong referential integrity requirements
- Reporting workloads that need aggregation across many dimensions

**Where it becomes a liability at scale:**
- **Join performance degrades** as table cardinality grows into hundreds of millions of rows. A 5-table join on 500M rows with imperfect index coverage will cause pain.
- **Schema migrations are expensive** — adding a column to a 2B-row table requires careful orchestration even with online DDL tools.
- **Horizontal scaling is hard by default.** PostgreSQL doesn't shard natively. CockroachDB and Spanner solve this but introduce distributed transaction overhead (2PC latency).
- **Write throughput ceiling** — a single PostgreSQL primary tops out around 50k–100k writes/sec under ideal conditions. Beyond that you need sharding.

**When to choose it:** Any domain with business logic that spans multiple entities atomically. E-commerce orders (order + line items + inventory + payment). Banking. Healthcare records.

**When to walk away:** You need horizontal write scale at the outset, your data is naturally hierarchical with no need for cross-entity joins, or your access patterns are known and narrow.

---

### Document (MongoDB, Firestore, Couchbase, DynamoDB in document mode)

**Mental model:** The unit of storage is a document (JSON/BSON). Each document is self-describing and self-contained. Relationships are expressed through embedding or references.

**Where it shines:**
- **Read-heavy workloads with a predictable access pattern** — if you always read a user and their profile together, embedding them in one document saves a round trip.
- **Flexible schema evolution** — different documents in the same collection can have different fields. Useful in early-stage products where the schema is still fluid.
- **Content management, catalogs, user profiles** — entities that are naturally self-contained.
- **Developer velocity** — JSON matches application object models directly; less ORM impedance mismatch.

**Where it becomes a liability:**
- **Many-to-many relationships become painful.** If you embed, you duplicate. If you reference, you do multi-round-trip joins in application code.
- **Transactions across documents** were an afterthought — MongoDB added multi-document ACID transactions in 4.0 but they carry meaningful performance overhead.
- **Unbounded arrays inside documents cause hot documents.** A document that keeps growing (e.g., a chat thread embedded in a user doc) will eventually hit size limits and cause write contention.
- **Aggregation pipelines can be complex** and don't have the query optimizer maturity of PostgreSQL.

**The critical anti-pattern:** Using MongoDB "because it's flexible" and then implementing application-level joins everywhere. You've built a relational system on top of a document store and paid extra for it.

**When to choose it:** Product catalogs (each product has wildly different attributes), CMS, mobile apps with user-specific denormalized data, IOT device state.

---

### Wide-Column (Cassandra, HBase, ScyllaDB, DynamoDB)

**Mental model:** Data is organized by a partition key (which routes to a node) and a clustering key (which determines sort order within a partition). Each row can have a different set of columns. Designed from the ground up for horizontal write scale.

**Where it shines:**
- **Extreme write throughput** — Cassandra is routinely benchmarked at 1M+ writes/sec across a cluster. The LSM-tree write path turns random writes into sequential I/O.
- **Time-series-like access patterns** — "give me the last N events for user X" maps perfectly to a partition on user ID and clustering on timestamp.
- **Geographic distribution** — multi-datacenter replication is a first-class feature, not an afterthought.
- **No single point of failure** — masterless architecture (Cassandra) means no primary failover.

**Where it becomes a liability:**
- **No secondary indexes worth using in production.** Global secondary indexes in Cassandra are notoriously slow and operationally risky. If you need to query by something other than the partition key, you either duplicate the data into a second table modeled for that query, or you do a full cluster scan.
- **No joins, no aggregations natively.**
- **Data modeling is query-first, not data-first.** You design your tables around the queries you know you'll run. Add a new query pattern? Add a new table. This leads to significant data duplication.
- **Eventual consistency by default.** Reads and writes can be tuned (QUORUM, ALL, ONE) but you're always making a trade-off between consistency and latency/availability.
- **Operational complexity is high** — compaction strategies, repair jobs, tombstone accumulation, and tuning GC are real operational concerns.

**When to choose it:** Activity feeds, time-series metrics, audit logs, messaging at scale (Discord uses Cassandra for message storage), recommendation stores.

**The killer question to ask yourself:** Can I describe every single query I'll ever run on this data right now? If yes, wide-column can work. If not, you'll regret it.

---

### Graph (Neo4j, Amazon Neptune, TigerGraph, ArangoDB)

**Mental model:** Data is nodes and edges. Relationships are first-class citizens stored explicitly, not computed via joins. Traversals follow edges directly without index lookups.

**Where it shines:**
- **Relationship-heavy queries where the depth of traversal is variable or large.** "Find all friends-of-friends of user X who have also purchased product Y" is O(depth) in a graph but requires multi-way self-joins in SQL that explode in complexity.
- **Fraud detection** — detecting rings of accounts connected through shared devices, IPs, and transactions.
- **Knowledge graphs** — semantic relationships between entities.
- **Recommendation engines** — traversing collaborative filtering graphs.
- **Access control / permissions** — hierarchical role inheritance (e.g., "can user X access resource Y through any inherited permission chain?").

**Where it becomes a liability:**
- **Write scale is difficult.** Most graph databases are not designed for high-throughput writes. Neo4j in particular struggles at ingestion rates that Cassandra would handle trivially.
- **Querying for non-graph patterns is painful.** If you need to do aggregate analytics (e.g., "how many orders were placed this month"), a graph database is the wrong tool.
- **Operational maturity is lower** than Postgres or Cassandra. Fewer engineers know Cypher or Gremlin. Fewer managed services.
- **Supernode problem** — a node with millions of edges (e.g., a celebrity in a social graph) causes traversal performance to degrade catastrophically unless handled explicitly.

**When to choose it:** The access pattern is fundamentally about traversing relationships of unknown or variable depth. Use it alongside a relational or document store, not as a primary store.

---

### Time-Series (InfluxDB, TimescaleDB, Prometheus, QuestDB, TDengine)

**Mental model:** Data is immutable append-only records indexed by time. The primary access patterns are range scans over time, downsampling, and aggregation over time windows.

**Where it shines:**
- **High-frequency metrics ingestion** — infrastructure monitoring, IoT sensor data, financial tick data.
- **Automatic data retention policies** — TTL-based expiry and downsampling (keep raw data for 7 days, hourly aggregates for 90 days, daily aggregates forever).
- **Time-window aggregations** (`avg cpu over last 5 minutes`) that are clunky in SQL but native operations in time-series DBs.
- **Compression ratios are exceptional** — time-series data is highly compressible (delta encoding, Gorilla compression on floats). InfluxDB gets 10–20x compression vs. raw storage.

**Where it becomes a liability:**
- **Updates and deletes are expensive** or not supported — the data model assumes append-only.
- **Non-time-based queries are slow** — if you need "all readings from sensor X where value > threshold regardless of time", you're doing a full scan.
- **Not a general-purpose store** — you'll always need a companion relational or document DB for the metadata (device registry, user accounts, etc.).

**When to choose it:** Any workload that is fundamentally a stream of timestamped measurements. TimescaleDB (Postgres extension) is often the right default — you get time-series optimizations without leaving the SQL ecosystem.

---

### Vector (Pinecone, Weaviate, Qdrant, Milvus, pgvector in PostgreSQL)

**Mental model:** Data is high-dimensional numerical vectors (embeddings). The primary query is Approximate Nearest Neighbor (ANN) search — "find the K vectors most similar to this query vector."

**Where it shines:**
- **Semantic search** — full-text search where you want meaning-matching, not keyword-matching.
- **Recommendation systems** — find items similar to what a user has interacted with.
- **RAG (Retrieval Augmented Generation)** pipelines — the backbone of LLM applications that need to retrieve relevant context from a corpus.
- **Image/audio similarity search.**

**Where it becomes a liability:**
- **ANN means approximate** — you trade recall (finding all true neighbors) for speed. If you need exact results, you need exact KNN which is O(n) and doesn't scale.
- **Metadata filtering + vector search interaction is subtle.** Filtering first and then doing ANN on the filtered set can be dramatically less accurate than doing ANN on the full set. Different databases handle this differently (pre-filter vs. post-filter vs. in-filter indexing).
- **Index build time is significant** — HNSW index construction on 100M vectors takes hours. Hot updates are expensive.
- **Stale embeddings are a silent failure mode** — if your embedding model changes, all your indexed vectors are now in the wrong space.

**2026 context:** pgvector with HNSW indexes in PostgreSQL is viable for <10M vectors. For 100M+, a dedicated vector store (Qdrant, Weaviate, Milvus) is warranted. Most new AI stacks run PostgreSQL + pgvector first, then graduate to a dedicated store when they hit latency or scale limits.

---

### Paradigm Selection Decision Tree

```
Need to traverse variable-depth relationships?
  → Graph (Neo4j, Neptune)

Primary data is timestamped measurements with TTL?
  → Time-series (TimescaleDB, InfluxDB)

Primary query is semantic similarity / ANN?
  → Vector (pgvector, Pinecone, Qdrant)

Extreme write throughput (>100k writes/sec) with known query patterns?
  → Wide-column (Cassandra, ScyllaDB)

Multi-entity ACID transactions with complex ad-hoc queries?
  → Relational (PostgreSQL, CockroachDB)

Self-contained hierarchical data with flexible schema, no cross-doc joins?
  → Document (MongoDB, Firestore)
```

---

## 2. Consistency, Availability, and Partition Tolerance

### What Is a Network Partition?

A **network partition** is when a distributed system's nodes are split into groups that cannot communicate with each other — even though all the nodes themselves are still running and healthy. The failure is in the **network links between nodes**, not the nodes themselves.

**Concrete example:**

You have three database nodes: A (primary), B (replica), C (replica).

```
Normal:     A ←—— network ——→ B ←—— network ——→ C

Partition:  A  |  firewall / broken switch / packet loss  |  B ——→ C
```

Node A is running fine. Nodes B and C are running fine. But A cannot send messages to B or C. B and C can talk to each other, but not to A. The cluster has been partitioned into two islands.

This happens constantly in real systems due to: a flaky network switch, a misconfigured firewall rule, an AWS availability zone connectivity loss, a datacenter fiber cut, or a hypervisor network throttle during maintenance.

**Why it forces a choice:** A write comes in to node A. A updates its data. A read comes in to node B. B hasn't received A's update because the partition is blocking replication. You must pick one of two bad options:

- **Consistency (CP):** B refuses to serve the read until it can confirm it's caught up with A. The system is unavailable during the partition.
- **Availability (AP):** B serves the read with the data it has, which may be stale. The system stays up but returns potentially incorrect data.

**Why "partition tolerance" is not optional:** You cannot build a distributed system that is immune to network failures — networks fail. So the real choice is always CP vs. AP — how does your system *behave* when a partition occurs? "Partition tolerant" doesn't mean it handles partitions gracefully; it means the system doesn't collapse entirely. A CP system still functions — it returns errors rather than stale data. An AP system still functions — it returns potentially stale data rather than errors. The distinction is what each sacrifices during a partition.

---

### CAP Theorem — What It Actually Says

The CAP theorem states: a distributed system can provide at most two of the following three guarantees simultaneously during a **network partition**:

- **Consistency (C):** Every read receives the most recent write or an error. All nodes see the same data at the same time.
- **Availability (A):** Every request receives a response (not necessarily the most recent data).
- **Partition Tolerance (P):** The system continues to operate even when network partitions drop messages between nodes.

**The critical nuance:** Partition tolerance is not optional in any distributed system. Networks partition. Packets drop. Therefore the real choice is: **during a partition, do you choose C or A?**

This means CAP is better understood as **CP vs. AP**, not a three-way choice.

---

### PACELC — The More Useful Model

CAP only talks about behavior during partitions. **PACELC** extends this:

> If there is a **P**artition, choose between **A**vailability and **C**onsistency.
> **E**lse (normal operation), choose between **L**atency and **C**onsistency.

This matters because most of the time there is no partition — and yet you still face a trade-off. Do you wait for all replicas to acknowledge a write (high consistency, higher latency) or do you return immediately after the primary acknowledges (low latency, risk of reading stale data)?

**Examples mapped to PACELC:**

| Database | Partition behavior | Normal behavior |
|---|---|---|
| PostgreSQL (sync replica) | CP — primary blocks until replica confirms | High C, higher L |
| PostgreSQL (async replica) | AP — primary doesn't wait | Low L, lower C on reads from replica |
| Cassandra (QUORUM writes) | Tunable — leans AP by default | Low L by default, higher C with QUORUM |
| DynamoDB (strongly consistent reads) | CP — reads go to primary | Higher L for strongly consistent reads |
| DynamoDB (eventually consistent reads) | AP | Low L, possible stale reads |
| CockroachDB / Spanner | CP — uses consensus (Raft/Paxos) | Higher L due to consensus rounds |
| Redis (standalone) | CP — single node, no partition concern | Very low L |
| Redis Cluster | AP during partition of majority node | Low L |

---

### Real Design Decisions: When to Sacrifice Consistency

**Sacrifice consistency for availability when:**

1. **User-facing reads where slight staleness is acceptable.** A social media timeline showing a post from 500ms ago instead of 100ms ago is fine. You'd use eventually consistent reads from a DynamoDB global table or a read replica.

2. **Shopping cart contents.** Amazon's famous Dynamo paper was motivated exactly by this. The cart being temporarily inconsistent (showing a stale item) is better than the cart being unavailable during checkout. Conflicts are resolved at read time (last-write-wins or merge).

3. **Leaderboard / counter updates.** A leaderboard that's accurate to ±1 second is acceptable. You'd use Redis with async persistence or Cassandra counters.

4. **Analytics dashboards.** Showing yesterday's numbers instead of numbers from 3 minutes ago is fine for most business dashboards.

5. **DNS.** Classic example — DNS records propagate eventually. A new record may not be visible globally for minutes to hours. The system remains available during propagation.

**Maintain consistency (sacrifice availability) when:**

1. **Financial transactions.** A double-spend must be impossible. A bank transfer that deducts from Account A must atomically credit Account B, or neither should happen. You use 2PC or distributed transactions (CockroachDB, Spanner) and accept the latency cost.

2. **Inventory reservation.** If you have 1 seat left on a flight and 200 concurrent users trying to buy it, you need a serializable or at least repeatable-read isolation level. Overselling is a business-critical failure mode.

3. **Auth / permissions.** If a user is revoked, they should not be able to access resources through a stale replica. You force reads to the primary or use a strongly consistent read.

4. **Unique constraint enforcement.** Usernames must be globally unique. Username registration reads/writes must go through a single authoritative source or use a distributed consensus protocol.

---

### Consistency Levels — The Spectrum

Don't think in binary. Modern systems give you a dial:

```
Eventual Consistency
    → Read-your-writes Consistency (you always see your own writes)
        → Monotonic Read Consistency (reads never go backward)
            → Session Consistency (consistent within a session)
                → Causal Consistency (causally related operations are ordered)
                    → Linearizability / Strict Serializability (strongest)
```

**Linearizability** means all operations appear to execute instantaneously at a single point in time, in an order consistent with real time. This is what CockroachDB and Spanner provide. It's expensive — every write requires Raft consensus before it's committed.

**Eventual consistency** means all replicas will converge to the same state eventually, but there's no bound on when. DynamoDB's default read mode, Cassandra with ONE read/write consistency.

**Session consistency** (read-your-writes within a session) is often the right middle ground for web apps — it ensures a user who just updated their profile immediately sees the update, without paying the cost of full linearizability.

---

### Isolation Levels (OLTP-specific)

Within a single database, isolation levels govern what anomalies can occur across concurrent transactions:

| Level | Dirty Read | Non-repeatable Read | Phantom Read | Performance |
|---|---|---|---|---|
| Read Uncommitted | Possible | Possible | Possible | Highest |
| Read Committed | Prevented | Possible | Possible | High |
| Repeatable Read | Prevented | Prevented | Possible | Medium |
| Serializable | Prevented | Prevented | Prevented | Lowest |

**PostgreSQL default:** Read Committed. Most applications never change this and don't think about the anomalies that can result.

**The phantom read problem:** In Repeatable Read, a transaction that queries "all orders over $1000" twice can get different results if another transaction inserted a new qualifying order between the two reads. Serializable prevents this. PostgreSQL uses SSI (Serializable Snapshot Isolation) to implement Serializable efficiently — it detects conflicts rather than locking, which is much better than traditional 2PL.

**Rule of thumb:** Use Serializable for financial operations. Read Committed is fine for most CRUD. Never use Read Uncommitted.

---

## 3. Normalization vs. Denormalization

### The Theory (Briefly)

**Normalization** is the process of organizing a database to reduce redundancy and improve data integrity by following Normal Forms:

- **1NF:** Atomic column values; no repeating groups.
- **2NF:** All non-key columns depend on the entire primary key (relevant when you have composite keys).
- **3NF:** No transitive dependencies — non-key columns don't depend on other non-key columns.
- **BCNF (3.5NF):** Every determinant is a candidate key.

**The promise:** Update anomalies become impossible. You update a customer's address once, not in 50 order records.

**The cost:** Reads require joins. Lots of them.

---

### When Strict Normalization Hurts

**1. High-read, low-write systems with hot join paths.**

Consider an e-commerce product listing page that needs: product name, price, category name, brand name, primary image URL, average rating, review count. In a fully normalized schema, this is 5–6 table joins on every page load. At 10k req/sec, you're issuing 50k–60k queries per second, and your query planner is doing join planning on every request.

The architectural solution: **materialize** this view. Either use a PostgreSQL materialized view (refreshed periodically), or denormalize at write time (maintain a `product_summary` table that gets updated when any constituent changes).

**2. Distributed systems where joins across shards are prohibitive.**

In a sharded database, joining across shards requires scatter-gather — send the query to all shards and merge results. This is expensive and often a performance cliff. The solution is to denormalize so that all data needed for a query lives on the same shard.

Example: In a multi-tenant SaaS app sharded by `tenant_id`, include `tenant_name` and `tenant_plan` in every row that references a tenant, even though it's redundant. You avoid a cross-shard join on every query.

**3. Read replicas with eventual consistency.**

If your replica is slightly behind the primary, a normalized query that reads from multiple tables (some from primary, some from replica) can return inconsistent results. Denormalization onto a single replica-sourced table eliminates this risk.

**4. High-frequency analytical queries on OLTP tables.**

A `COUNT(*)` with a GROUP BY across a normalized schema requires full table scans across multiple tables. A pre-aggregated summary table (denormalized) lets you answer this in microseconds.

---

### When Denormalization Is the Right Architectural Call

**The golden rule:** Denormalize around your read patterns. Know your top-10 most frequent queries and ensure they hit as few tables as possible.

**Practical patterns:**

1. **Embedding frequently co-accessed data.** If you always read `order` with `customer.name` and `customer.email`, add those fields to the `orders` table or a materialized view. Accept that you now have to update more places when a customer changes their name (a rare event).

2. **Pre-aggregating counters.** Instead of `SELECT COUNT(*) FROM likes WHERE post_id = X`, maintain a `like_count` column on the `posts` table. Increment atomically with the like insert (`UPDATE posts SET like_count = like_count + 1 WHERE id = X`). This turns an O(n) scan into an O(1) point read. Caveat: counter updates must be atomic; use Redis if you need very high counter throughput.

3. **Storing computed fields.** If you frequently display `full_name = first_name || ' ' || last_name`, either store it or use a generated column (PostgreSQL supports this). Don't compute it in every query.

4. **Flattening hierarchies.** If you have a deep category tree (`electronics → computers → laptops → gaming laptops`), and you frequently query "all products under electronics", storing the full path array (ltree in PostgreSQL, or an array of ancestor IDs) avoids recursive CTEs on every request.

**The update anomaly bargain:** Every time you denormalize, you're saying "I accept that writes become more complex in exchange for simpler, faster reads." Be explicit about this trade-off and document it. Make sure your application code enforces consistency of denormalized fields transactionally.

---

### The CQRS Connection

Denormalization taken to its logical extreme becomes CQRS: your write model (normalized, OLTP) and your read model (denormalized, optimized for specific views) are separate data stores. When a write happens, an event updates the read model asynchronously. This is covered in depth in Section 7.

---

## 4. Scaling Strategies

### Vertical Scaling

Throw more CPU, RAM, and faster storage at the single node.

**When it's right:**
- You're in early stages and operational simplicity matters more than theoretical scale limits.
- Your workload is CPU-bound on complex queries that benefit from more cores.
- You're using PostgreSQL and haven't hit write throughput limits (~50k–100k writes/sec on NVMe with connection pooling).

**Practical ceiling:** AWS RDS db.r7g.16xlarge gives you 64 vCPUs, 512 GB RAM. Aurora can push further with cluster caching. But you'll pay $10k+/month and you still have a single point of failure for writes.

**The hidden benefit of vertical scaling:** No application changes, no operational complexity, no partitioning logic. If your growth trajectory can be served with a big single node for 18 months, take that path and use the time to build more features.

---

### Read Replicas

Distribute read traffic to replica nodes while all writes go to the primary.

**How it works:** The primary streams its WAL (Write-Ahead Log) to replicas. Replicas apply changes asynchronously (or synchronously with performance penalty). Replicas are read-only.

**Replication lag is the central concern.** Asynchronous replicas can be anywhere from milliseconds to seconds behind the primary. This is fine for reports and analytics. It's a problem for "read-after-write" scenarios — a user creates a record, then immediately reads it from a replica that hasn't replicated yet and sees nothing.

**Architectural mitigations for replication lag:**
- Route writes and subsequent reads within the same session to the primary.
- Use a sticky session / session affinity for writes.
- Read from replica with a `min_replica_lag` threshold check.
- Store the replication position after each write and reject reads from replicas that haven't caught up.

**Limits:** Read replicas only help with read-heavy workloads. They don't help with write throughput. You can spin up 10 read replicas but your primary is still your write bottleneck.

**PostgreSQL replication:** Streaming replication via WAL sender/receiver. Can have synchronous_standby_names to force a replica to confirm receipt before the primary commits — provides durability without losing availability (the replica can be local rack, low latency).

---

### Horizontal Sharding

Split the data across multiple independent database nodes, each owning a subset of the data.

**Sharding key selection is the most important and irreversible decision in a sharded architecture.** Choose wrong and you'll have hot shards, cross-shard joins, and a re-sharding operation that takes months.

#### Range Sharding

Each shard owns a contiguous range of the key space (e.g., user IDs 1–1M on shard 1, 1M–2M on shard 2).

**Advantage:** Range queries are efficient — "get all orders between date X and date Y" only hits one shard.

**Fatal flaw: hot shards.** If your key is time-based or sequential (auto-increment ID, timestamp), all new writes go to the latest shard. One shard is always hot. The others are cold. You've built an un-balanced system.

**Fix:** Use range sharding only on truly uniform-access keys (e.g., customer IDs that are accessed uniformly regardless of recency).

#### Hash Sharding

Hash the sharding key and mod by the number of shards to determine placement. `shard = hash(user_id) % N`.

**Advantage:** Even distribution of writes across all shards. No hot shards.

**Fatal flaw: re-sharding.** When you add a new shard, almost all data needs to move. `N` changes, so `hash(user_id) % N` routes to a different shard for almost every row. The standard mitigation is **consistent hashing** — each shard owns an arc of a hash ring, and adding a shard only moves data from adjacent shards, not all data.

**Fatal flaw 2: range queries are expensive.** "Get all users from state CA" requires a scatter-gather to all shards. This is a full table scan distributed across your entire cluster.

#### Directory-Based (Lookup) Sharding

Maintain a lookup table: `user_id → shard_id`. Every query first looks up the shard mapping, then queries that shard.

**Advantage:** Maximum flexibility. You can move users between shards without changing the application. Hot users can be isolated. Re-sharding is surgical.

**Fatal flaw: lookup table is a bottleneck and single point of failure.** You must cache the lookup table aggressively (in-process, in Redis). If the lookup service is down, every query fails. You've introduced a new dependency into every database call.

**When to use it:** Multi-tenant SaaS where you want to isolate large tenants on their own shards. Customer-specific routing logic.

#### What to Shard By

- **Shard by tenant ID** in multi-tenant SaaS — all data for a tenant lives on one shard, no cross-shard joins for that tenant's queries. The risk is large tenants becoming hot shards (mitigate with directory-based routing to spread large tenants).
- **Shard by user ID** in consumer apps — all of a user's data is co-located.
- **Never shard by time** unless you want hot shards and the ability to archive cold shards.
- **Never shard by a low-cardinality field** (e.g., status, region with 5 values) — you'll have 5 shards max.

**Cross-shard transactions are the price you pay.** Any operation that spans two shards requires 2-phase commit or saga patterns. Design your schema so that the overwhelming majority of transactions are single-shard.

---

### Federation (Functional Partitioning)

Split the database by domain rather than by row. The users service has its own DB, the orders service has its own DB, the inventory service has its own DB.

**This is microservices at the database layer.** Each service owns its data. Cross-service data access goes through APIs, not joins.

**Advantages:**
- Independent scaling per domain.
- Independent schema evolution.
- Failure isolation — inventory DB going down doesn't take down the users DB.
- Technology choice per domain — use PostgreSQL for orders (ACID), Cassandra for activity feeds, Redis for session state.

**The cost:**
- Joins across domains are gone. You can't `SELECT orders JOIN users` anymore.
- Distributed transactions across services require sagas or 2PC — both are complex and error-prone.
- Data consistency across services is eventual and requires careful design (outbox pattern, event-driven updates).
- Reporting and analytics across domains requires a data warehouse that ingests from all services.

**Federation is not a sharding strategy — it's an organizational boundary strategy.** It's about domain ownership, not raw scale.

---

## 5. Indexing Strategies

### Why Indexes Are Architectural Decisions

An index is a deal with the devil: you pay extra on every write to maintain a data structure that makes reads faster. Index too little and reads are slow. Index too much and writes slow down, storage grows, and the optimizer may choose a bad index. **Index design must be driven by query workload analysis, not intuition.**

---

### B-Tree Indexes

**The default index type in PostgreSQL, MySQL, and most relational databases.**

**Data structure:** A balanced tree where each leaf node contains a pointer to the actual data row. The tree stays balanced through splits and merges on insert/delete.

**What it's good for:**
- Equality lookups: `WHERE user_id = 123`
- Range queries: `WHERE created_at BETWEEN '2026-01-01' AND '2026-01-31'`
- Sorting: `ORDER BY last_name` (if there's an index on last_name, the sort is free)
- Prefix matching on strings: `WHERE name LIKE 'Jo%'` (note: `'%Jo%'` is NOT a prefix match and won't use the B-tree)

**Limitations:**
- Not good for full-text search.
- Not good for very high-cardinality write-heavy columns — the tree has to be rebalanced constantly.
- Can't be used for `LIKE '%substring%'` queries.

**Write amplification:** Each insert modifies O(log N) nodes in the tree. On a 100M row table, this is roughly 27 page writes per insert. With many indexes on a table, this multiplies.

---

### LSM-Tree (Log-Structured Merge-Tree)

**Used by:** Cassandra, RocksDB (underlying LevelDB, TiKV, CockroachDB storage), LevelDB, InfluxDB.

**The design:** Writes go to an in-memory structure (memtable) first, which is periodically flushed to sorted immutable files (SSTables) on disk. Background compaction merges SSTables and removes deleted/old versions.

**The trade-off:**
- **Writes are fast** — sequential disk I/O, no random page updates, no B-tree rebalancing.
- **Reads can be slower** — a read must check the memtable, then potentially multiple SSTables at different levels. Bloom filters mitigate this (quickly rule out SSTables that don't contain the key).
- **Write amplification from compaction** — data gets rewritten multiple times as SSTables are compacted. A key inserted once may be rewritten 10+ times by compaction. This wears SSDs faster.
- **Space amplification** — deleted data isn't reclaimed until compaction. A table with high delete rates has lots of wasted space before the next compaction.

**When to choose LSM over B-tree:** Write-heavy workloads where write throughput and write latency matter more than read latency. Time-series data, metrics ingestion, event logs.

---

### Inverted Indexes

**Used by:** Elasticsearch, OpenSearch, PostgreSQL (GIN index with tsvector), Solr.

**Data structure:** A mapping from each unique term to the list of documents containing that term. Like the index at the back of a book, but for every word.

**What it enables:** Full-text search — `WHERE description CONTAINS 'distributed systems'`. Also used for array containment queries in PostgreSQL (`WHERE tags @> ARRAY['postgresql', 'indexing']`).

**The cost:**
- Index builds on large text corpora are slow.
- Inverted indexes are large.
- Updates require updating the posting lists for every term that changed.
- Not good for range queries on structured data.

**PostgreSQL's GIN index** is an inverted index implementation that supports `tsvector` full-text search, `jsonb` containment (`@>`), and array containment. If you need full-text search and your data is already in PostgreSQL, a GIN index on a `tsvector` column is often sufficient for millions of documents before you need Elasticsearch.

---

### Composite Indexes

A single index on multiple columns: `CREATE INDEX idx ON orders (customer_id, status, created_at)`.

**The left-prefix rule:** A composite index on `(A, B, C)` can be used for queries on `A`, `A+B`, or `A+B+C` — but NOT on `B` alone, `C` alone, or `B+C` alone. The index is only usable from the leftmost column.

**Column ordering matters enormously:**
- Put equality filters before range filters: `(customer_id, status, created_at)` works well for `WHERE customer_id = X AND status = 'shipped' AND created_at > Y`. The equality columns filter the most rows first, and then the range is applied on a small subset.
- Putting the range column first — `(created_at, customer_id, status)` — means the index can only efficiently use `created_at` as a filter; the remaining columns can't be used as effectively after a range scan.

---

### Partial Indexes

Index only rows that match a condition: `CREATE INDEX idx ON orders (customer_id) WHERE status = 'pending'`.

**When to use:** If you have a large table but only query a small subset frequently. A `pending` orders index might cover 1% of rows but 90% of your operational queries. The index is tiny, fits in memory, and is extremely fast.

**Example:** `CREATE INDEX idx_active_users ON users (email) WHERE active = true`. If 95% of users are inactive (deleted/churned), this index is 20x smaller than a full index and all login queries use it.

---

### Covering Indexes

A covering index contains all the columns needed by a query — the database can satisfy the query entirely from the index without touching the actual table rows (heap) at all.

```sql
-- Query:
SELECT customer_id, status, total FROM orders WHERE customer_id = 123 ORDER BY created_at;

-- Covering index:
CREATE INDEX idx ON orders (customer_id, created_at) INCLUDE (status, total);
```

The `INCLUDE` columns are stored in the leaf nodes of the B-tree but not in the internal nodes (they don't affect the tree structure, but they are available in a covering scan). PostgreSQL 11+ supports `INCLUDE` on B-tree indexes.

**Performance impact:** A covering index eliminates heap accesses entirely for qualifying queries. On a hot, frequently-run query, this can be a 2–10x speedup. It's especially impactful when the heap is not cached in memory (large table, sequential pages not in buffer pool).

**The cost:** Bigger index = more memory needed, more write overhead.

---

### Index Design Process

1. **Start with `EXPLAIN (ANALYZE, BUFFERS)`** to see exactly what the query planner does on your workload.
2. Identify queries with `Seq Scan` on large tables — these are missing index opportunities.
3. Identify `Index Scan` vs. `Index Only Scan` — if it's not `Index Only Scan`, consider adding `INCLUDE` columns to make it covering.
4. Look for high buffer hit counts on index pages — those indexes are hot and should fit in `shared_buffers`.
5. Review `pg_stat_user_indexes` to find indexes with zero or near-zero scans — these are dead weight, remove them.
6. Run `pg_stat_statements` to find your top-10 most expensive queries by total time, then index for those.

---

## 6. OLTP vs. OLAP vs. HTAP

### OLTP (Online Transaction Processing)

**Characteristic workload:** Many small, fast, concurrent read/write transactions. Low latency per operation (sub-millisecond to single-digit milliseconds). High throughput of concurrent connections.

**Schema design:** Normalized (3NF or BCNF). Small row sizes. Point reads and narrow range scans. Write patterns are random and distributed across the key space.

**Storage engine requirements:** B-tree indexes. Row-oriented storage (RDBMS store all columns for a row together, enabling fast single-row access). MVCC for concurrent transaction isolation.

**Examples:** PostgreSQL, MySQL, Oracle, SQL Server for the core application DB. DynamoDB, Cassandra for specific OLTP patterns requiring horizontal scale.

**Key metrics:** Transactions per second (TPS), P99 latency, connection count, lock contention.

---

### OLAP (Online Analytical Processing)

**Characteristic workload:** Few large queries, each touching many rows across many columns. Users are analysts or dashboards running complex aggregations, GROUP BYs, window functions, and multi-table joins. Queries can run for seconds to minutes.

**Schema design:** Denormalized (star schema or snowflake schema). Large fact tables (events, transactions) joined to small dimension tables (users, products, dates). Wide rows. Batch or micro-batch writes (not row-by-row OLTP writes).

**Storage engine requirements:** Columnar storage (store all values for a column together, enabling vectorized aggregation over a single column while skipping others). Compression per column (similar data compresses well). Vectorized query execution. Massive parallelism.

**Examples:** ClickHouse, Snowflake, BigQuery, Redshift, DuckDB, Apache Druid.

**Why columnar storage wins for OLAP:**
```
Query: SELECT avg(price), sum(quantity) FROM orders WHERE category = 'electronics'

Row store: Read every row (including customer_id, address, notes, etc.) to get price and quantity
Column store: Read only the 'price', 'quantity', and 'category' columns — skip everything else
```
For a query scanning 100M rows but touching 3 of 50 columns, columnar storage reads 3/50 = 6% of the data.

---

### The Dual-Database Pattern

Most serious data platforms run both: a PostgreSQL/MySQL OLTP cluster for application transactions, and a Snowflake/ClickHouse OLAP cluster for analytics. Data is moved from OLTP to OLAP via:
- **ETL pipelines** (daily batch jobs — cheap but stale)
- **Change Data Capture (CDC)** with tools like Debezium (captures WAL changes, streams to Kafka, loads into OLAP — near-real-time, sub-minute latency)
- **Direct S3/Parquet exports** from application events to a data lake, then queried by BigQuery or Athena

---

### HTAP (Hybrid Transactional/Analytical Processing)

**The promise:** Run both OLTP and OLAP workloads on the same database engine, on the same data, without ETL. Fresh analytical queries against live transactional data.

**How it's implemented:**
- **TiDB + TiFlash:** TiDB stores row-format data in TiKV (OLTP), which is asynchronously replicated to TiFlash (columnar) for OLAP queries. The optimizer routes queries to the appropriate storage engine automatically.
- **MySQL HeatWave:** Oracle's offering where a columnar in-memory accelerator is attached to MySQL, enabling analytical queries without leaving the MySQL ecosystem.
- **SQL Server HTAP:** In-memory columnstore indexes alongside row-store tables.
- **SingleStore (formerly MemSQL):** Unified row and column store with in-memory execution.

**The HTAP reality in 2026:** True HTAP is architecturally appealing but operationally complex. The "fresh analytics" value prop matters most when your business requires real-time operational decisions (fraud detection, live inventory, real-time pricing). For most BI use cases, 5-minute-old data is fine, and a CDC-based dual-database architecture is simpler and more proven.

---

### ClickHouse — The 2026 OLAP Default for Many Teams

ClickHouse deserves special attention because it's become the de facto choice for high-throughput event analytics:

**Why ClickHouse:** Ingests millions of events/sec. Column-oriented, vectorized. Aggressive compression (10–40x on event data). Blazing fast aggregations — can do `COUNT(*), sum(metric)` over 1B rows in seconds on a single server.

**ClickHouse limitations:**
- **Not designed for row-level updates or deletes.** `ALTER TABLE ... UPDATE` triggers a merge, not an in-place update. DML is mutation-based and async.
- **Not ACID transactional** at the row level. No multi-statement transactions.
- **Poor for low-latency point lookups.** If you need `SELECT * WHERE id = 123` in <1ms, ClickHouse is wrong. Use Redis or PostgreSQL.
- **Data must arrive somewhat ordered** by the sort key for best compression and query performance.

**DuckDB** is the embedded OLAP engine of the moment — runs in-process (no server), queries Parquet files directly from S3, ideal for analytical notebooks and lightweight ETL. Not for production OLAP serving.

---

## 7. Event Sourcing and CQRS

### Event Sourcing

**Core concept:** Instead of storing current state, store every event that led to that state. The "database" is an append-only log of immutable domain events.

```
Traditional:  orders table row: { id: 1, status: "shipped", total: 99.99 }

Event sourced: event log:
  { order_id: 1, type: "OrderPlaced", data: { total: 99.99 }, ts: ... }
  { order_id: 1, type: "PaymentReceived", data: { amount: 99.99 }, ts: ... }
  { order_id: 1, type: "OrderShipped", data: { tracking: "1Z..." }, ts: ... }
```

**Current state** is derived by replaying all events for an entity.

**Why this is architecturally significant:**

1. **Complete audit log built-in.** You have the full history of every state transition with timestamps. In financial systems or compliance-heavy domains, this is not optional — you must be able to reconstruct state at any point in time.

2. **Temporal queries become natural.** "What was the state of this order at 3pm yesterday?" is a replay operation, not a complex query on a history table.

3. **Event replay enables projection rebuilds.** If you discover a bug in how you calculated a field, you can replay all events and regenerate the correct state from scratch. This is not possible with state-based storage.

4. **Natural integration with event-driven architecture.** Your database writes are domain events — they can be published to a message bus for other services to consume.

**When event sourcing justifies its complexity:**
- **Financial systems** — every debit, credit, transfer is an immutable fact. Ledger integrity requires it.
- **Collaborative editing** — Google Docs stores operations, not document snapshots. Conflict resolution and CRDT-based merging requires the operation log.
- **Compliance-heavy domains** — healthcare (HIPAA audit trails), legal, banking.
- **Domains with complex state machines** — order lifecycle, loan approval workflow, claims processing.

**When event sourcing does NOT justify its complexity:**
- CRUD applications with no audit requirements.
- Simple reference data (lookup tables for countries, categories, etc.).
- Any domain where you don't need temporal queries or event replay.

**The event sourcing trap:** Teams adopt event sourcing for the wrong reasons ("it's modern", "our tech lead saw it at a conference"). The operational cost is real — event store management, schema evolution of event types, aggregate loading performance (replaying 10,000 events to get current state), snapshot strategies to avoid full replays. Don't use it unless your domain has explicit needs for it.

---

### Event Store Implementation Choices

| Option | Trade-offs |
|---|---|
| EventStoreDB | Purpose-built for event sourcing. Projections built-in. Good operational tooling. Not widely known. |
| PostgreSQL (append-only table) | Simple. ACID. WAL-based CDC for downstream consumers. Limited to vertical/replica scale. |
| Kafka | Distributed, high-throughput. But it's a message log, not a database — compaction can delete events, ordering per partition only, no direct event replay by aggregate ID without extra indexing. |
| DynamoDB (append-only) | Serverless, horizontally scalable. But streams for CDC have limited retention (24 hours). Needs external store for projections. |

For most applications starting with event sourcing, **PostgreSQL with an `events` table** is the right choice. You get ACID, WAL-based CDC, and familiar tooling. Migrate to EventStoreDB when you need its specific features (projections, subscriptions) at scale.

---

### CQRS (Command Query Responsibility Segregation)

**Core concept:** Separate the model used for writes (the command model / write side) from the model used for reads (the query model / read side). They can be different schemas, different tables, or even different databases.

```
Write side:  normalized relational schema, ACID transactions, append-only events
Read side:   denormalized read models optimized for specific UI views, eventually consistent
```

**How CQRS typically works in practice:**

1. A command (e.g., `PlaceOrder`) is validated and executed against the write model (PostgreSQL, normalized).
2. An event is emitted (`OrderPlaced`) — either synchronously via a domain event or asynchronously via CDC.
3. Event handlers update one or more read models (e.g., a `user_order_history` denormalized table in PostgreSQL, a search index in Elasticsearch, a cached view in Redis).
4. Reads query the appropriate read model directly — no joins, highly optimized.

**The independence this provides:**
- Read models can be rebuilt from events at any time without touching the write model.
- Read models can use different databases entirely — write in PostgreSQL, read from Elasticsearch (full-text search) or Redis (high-frequency point reads).
- Read and write models can be scaled independently.

**When CQRS justifies its complexity:**
- High-read, lower-write systems where you need multiple differently-optimized views of the same data.
- Systems with multiple distinct read patterns (a user portal, a mobile app, and an analytics dashboard all need different representations of the same domain objects).
- Event sourced systems — CQRS is the natural companion to event sourcing.

**CQRS anti-patterns:**
- Using CQRS for simple CRUD with no distinct read/write model differentiation.
- Building synchronous CQRS where the "separation" is just different service methods calling the same table.
- Not handling the eventual consistency window in the UI — users submit a command and then see stale data because the read model hasn't updated yet. This must be handled explicitly (optimistic UI updates, read-your-write guarantees, or loading spinners).

---

### The Outbox Pattern

The critical problem: after a command is persisted to the write DB, how do you reliably publish an event to a message bus without losing it?

A naive approach (`UPDATE db; publish to Kafka;`) fails because the second operation can fail after the first succeeds, leaving the event unpublished — a permanent inconsistency.

**The Outbox Pattern:**

1. In the same transaction that updates the domain state, INSERT the event into an `outbox` table.
2. A separate process (CDC on the `outbox` table, or a polling publisher) reads the outbox and publishes to Kafka/SNS/etc.
3. Mark the outbox entry as `published = true` (or delete it) after confirmed delivery.

This gives you **transactional outbox** — the event is guaranteed to be published exactly once (or at-least-once with idempotency), without distributed transactions.

---

## 8. Modern Trends in 2026

### Multi-Model Databases

**The pattern:** A single database engine that supports multiple data models — relational, document, graph, key-value — within the same storage layer.

**Examples:**
- **ArangoDB:** Natively supports documents, graphs, and key-value in one engine.
- **FerretDB:** MongoDB-compatible API on top of PostgreSQL.
- **PostgreSQL** is increasingly multi-model through extensions: `jsonb` (document), `pgvector` (vector), `PostGIS` (geospatial), `AGE` (graph via Apache Age extension), `TimescaleDB` (time-series).

**The appeal:** Reduce operational overhead. One database to manage, one backup strategy, one team skill set.

**The skepticism:** True multi-model systems often make every model "good enough" but none of them great. A dedicated graph database (Neo4j) will outperform PostgreSQL + AGE for deep graph traversals. A dedicated time-series DB will outperform TimescaleDB for extreme ingestion rates. Choose the right tool for the primary workload, and use PostgreSQL's multi-model capabilities as a convenience for secondary patterns.

**The PostgreSQL exception:** PostgreSQL's multi-model capabilities are remarkably good for most mid-scale applications. If you're not at the extreme end of any specific model's requirements, staying in the PostgreSQL ecosystem reduces operational complexity with minimal performance penalty.

---

### Serverless Databases

**The concept:** Database instances that auto-scale to zero when idle, auto-scale up under load, billed per request or per compute-second, no capacity planning.

**Aurora Serverless v2 (AWS):**
- Scales in increments of 0.5 ACU (Aurora Capacity Units) from 0.5 to 128 ACUs.
- Scaling is sub-second for gradual load increases, but cold starts from zero can take several seconds — painful for scheduled jobs or low-traffic applications with bursty patterns.
- Good for: Dev/staging environments, variable-load production apps with predictable warm periods, multi-tenant SaaS where most tenants are low-traffic.
- Cost model: More expensive per compute-unit than provisioned for sustained high loads. Do the math before committing.

**Neon:**
- Serverless PostgreSQL with storage/compute separation. Scales to zero. Branching (create a database branch for a PR, test schema migrations, discard it).
- **Database branching is the killer feature for CI/CD** — your test suite spins up a fresh branch of production data, runs migrations, tests, then deletes the branch. Zero data contamination.
- Targeting developer experience and DevOps workflows more than raw performance.
- Not yet suitable for sustained high-throughput production workloads (2026 — watch this space).

**PlanetScale:**
- Serverless MySQL with a Vitess-backed sharding layer.
- **Schema migrations without locks** (via the online DDL and shadow table approach).
- Branching similar to Neon.
- Removed the free tier in 2024; repositioned as enterprise. Strong for teams already on MySQL who need horizontal scale.

**The serverless database trade-offs:**
- **Cold start latency is real.** Scale-to-zero is great for cost, terrible for latency on first request after an idle period.
- **Connection model changes.** Traditional databases scale to a few thousand connections. Serverless functions spawn thousands of ephemeral connections. Use connection poolers (PgBouncer, RDS Proxy) or switch to HTTP-based query APIs.
- **Vendor lock-in is high.** Neon's branching, PlanetScale's schema workflow, and Aurora's ACU model are not portable.

---

### Vector Search Integration for AI Workloads

**2026 context:** Every production AI application has a retrieval layer. RAG (Retrieval Augmented Generation) is table stakes for LLM apps that need to query proprietary or recent data.

**Architecture patterns for vector search:**

**Pattern 1: PostgreSQL + pgvector**
```
Embedding model → float[] vector → pgvector column → HNSW index
```
- Best for: <10M vectors, existing PostgreSQL stack, simple use cases
- HNSW index in pgvector gives sub-10ms ANN search at this scale
- Supports hybrid search (BM25 keyword + vector semantic) with `pg_trgm` or `pg_search` (ParadeDB)

**Pattern 2: Dedicated vector store (Qdrant, Weaviate, Milvus)**
- Best for: 10M–1B+ vectors, multi-tenancy, complex metadata filtering
- Qdrant in 2026 is the engineering-favorite: written in Rust, excellent performance, strong filtering support, on-disk indexing for large corpora
- Weaviate integrates tightly with LLM providers (OpenSearch + generative modules)
- Milvus is the open-source standard for Billion-scale deployments

**Pattern 3: Managed vector stores (Pinecone)**
- Fully managed, serverless pricing, native metadata filtering
- Highest operational simplicity; highest cost at scale; least customizability

**The hybrid search problem (critical for 2026):**
Pure vector search has mediocre precision for exact-match queries (product SKUs, IDs, specific phrases). Pure keyword search (BM25) has poor recall for semantic queries. The state of the art is **hybrid retrieval** — run both vector and BM25 searches, then combine results with Reciprocal Rank Fusion (RRF) or a learned re-ranker.

Both Qdrant and Weaviate support hybrid search natively. In PostgreSQL, combine pgvector (vector) with `pg_search` or tsvector (keyword) at the application layer.

**Embedding versioning — the silent migration problem:**
When your embedding model changes (e.g., upgrading from `text-embedding-3-small` to a newer model), all existing vectors are incompatible with new query vectors. You need a migration strategy:
1. Add a `model_version` column to your vector records.
2. Run a background job to re-embed with the new model into a new index.
3. Serve from both indexes during migration, weighted toward the new one.
4. Swap fully to the new index once migration is complete.

---

### Rise of Embedded Analytics

**The shift:** Analytics is moving from centralized data warehouses to embedded, in-process engines that query data where it lives.

**DuckDB** is the center of this trend:
- Embeds directly in Python, Node.js, Java processes — no server to run.
- Queries Parquet, CSV, JSON directly from S3 or local disk.
- Full SQL with window functions, JOINs, CTEs.
- Columnar execution engine — analytically fast.
- Used in data engineering for local development, testing, and lightweight production pipelines.

**MotherDuck:** Managed DuckDB-as-a-service with the ability to run queries locally and in the cloud on the same engine. Designed for the "small data" tier that doesn't justify a full Snowflake cluster.

**Apache Arrow / DataFusion:** The in-memory columnar format (Arrow) and query engine (DataFusion) that underpins DuckDB and many other analytics tools. Arrow has become the universal interchange format between analytics components — query results from PostgreSQL, Kafka, S3, and DuckDB can all be materialized as Arrow and passed between systems without serialization overhead.

**The implication for architecture:** Not every analytics workload needs a dedicated warehouse. For internal tooling, data science notebooks, and lightweight dashboards, DuckDB + Parquet on S3 is a complete analytics stack with near-zero operational cost. Reserve Snowflake/BigQuery/ClickHouse for multi-team, multi-TB shared analytical data.

---

## 9. Schema Migrations in Production

### Why This Is Harder Than It Looks

Schema migrations are among the highest-risk operations in production database management. A naive `ALTER TABLE ADD COLUMN NOT NULL` on a 500M-row PostgreSQL table will:
1. Take an exclusive lock on the table for the duration of the migration.
2. Block all reads and writes.
3. Take potentially hours to complete.
4. Cause an outage.

**The goal:** Zero-downtime migrations. All schema changes must be applied without blocking production traffic.

---

### PostgreSQL Locking Behavior

Every DDL operation acquires a lock. The lock levels relevant to migrations:

| Operation | Lock Level | Blocks |
|---|---|---|
| `ADD COLUMN` (nullable, no default) | ShareUpdateExclusiveLock | Only other DDL |
| `ADD COLUMN NOT NULL` (older Postgres) | AccessExclusiveLock | All reads and writes |
| `ADD COLUMN NOT NULL DEFAULT x` (PG11+) | ShareUpdateExclusiveLock | Only DDL (the default is stored in catalog, not rewritten) |
| `DROP COLUMN` | AccessExclusiveLock | All reads and writes |
| `CREATE INDEX` | ShareLock | Writes (blocks) |
| `CREATE INDEX CONCURRENTLY` | ShareUpdateExclusiveLock | Only other DDL |
| `ALTER COLUMN TYPE` | AccessExclusiveLock | All reads and writes |
| Backfilling a column | No lock (plain UPDATE) | Row-level locks only |

**Critical PostgreSQL 11+ improvement:** Adding a column with a non-volatile default no longer rewrites the table. The default value is stored in `pg_attribute`. This makes `ADD COLUMN DEFAULT x` safe. Adding `NOT NULL` with a default is also safe in PG11+.

---

### The Expand-Contract Pattern

The gold standard for zero-downtime schema migrations. Three phases executed across multiple deploys:

**Phase 1 — Expand:** Add new columns/tables alongside old ones. The old schema still works. New code can start writing to both old and new.

```sql
-- Safe: nullable column, no lock concerns
ALTER TABLE users ADD COLUMN full_name TEXT;
```

Deploy new code that writes to both `first_name + last_name` (old) and `full_name` (new).

**Phase 2 — Migrate:** Backfill the new column from old data. Do this in batches to avoid locking.

```sql
-- Batch backfill to avoid long-running lock
UPDATE users SET full_name = first_name || ' ' || last_name
WHERE id BETWEEN 1 AND 10000 AND full_name IS NULL;
-- Repeat in batches of 10k
```

**Phase 3 — Contract:** Once all data is migrated and the new code fully uses the new column, drop the old columns.

```sql
ALTER TABLE users DROP COLUMN first_name;
ALTER TABLE users DROP COLUMN last_name;
```

**The timeline:** Each phase is a separate deploy. Phase 1 and Phase 3 especially must be separated by enough time to ensure all running application instances have rolled over to the new code before the old schema is removed.

---

### Online DDL Tools

**pt-online-schema-change (Percona Toolkit):** For MySQL. Creates a shadow table with the new schema, copies data in batches, uses triggers to propagate writes during migration, then does a fast rename swap. The table is never fully locked.

**gh-ost (GitHub):** MySQL online schema change without triggers. Uses MySQL binlog replication to capture changes during migration. More reliable than triggers for high-write tables.

**`CREATE INDEX CONCURRENTLY` in PostgreSQL:** Builds the index without locking writes. Takes longer (needs multiple passes), but production traffic continues. Use this for every production index creation.

**pgroll (Xata) and Atlas:** Newer tools that implement the expand-contract pattern as a first-class primitive. You describe the target schema, and the tool manages the multi-phase migration automatically, including backfilling and the dual-write period.

---

### Tooling Comparison

**Flyway:**
- Java-based, versioned SQL migrations.
- Each migration is a SQL file (`V1__init.sql`, `V2__add_column.sql`).
- Migrations run in version order; once applied, they're never re-run.
- Strong, mature, widely used in JVM ecosystems.
- Weakness: Doesn't help you with the application-level dual-write coordination — it just runs SQL.

**Liquibase:**
- Similar to Flyway but uses XML/YAML/JSON changesets in addition to SQL.
- More flexible for cross-database portability.
- Rollback support (though rolling back migrations on production data is almost never safe).
- Enterprise features for compliance (generating database audit reports).

**Atlas (ariga.io):**
- Schema-as-code, state-driven (declare the target schema, Atlas diffs and generates the migration).
- Integrates with Terraform, Kubernetes, GitHub Actions.
- HCL or SQL schema definition.
- The modern approach — you maintain the desired state, not the migration history.
- Native support for linting migration safety (will warn you if your migration will take an exclusive lock).

**Recommendation for new projects (2026):** Atlas for infrastructure-as-code shops. Flyway for teams that want simple, explicit, versioned SQL. Either way, your migration process must enforce that every migration is reviewed for lock safety before it runs in production.

---

### Non-SQL Migration Concerns

**DynamoDB:** Schema is implicit (items are schemaless). "Migrations" are application-level — add new attributes to new items; backfill old items; then remove old attributes when all reads handle the new format. No DDL statements to run.

**Cassandra:** `ALTER TABLE ADD COLUMN` is safe (cells are sparse, existing rows just don't have the new column). `ALTER TABLE DROP COLUMN` is safe after removing all reads of that column. `ALTER TABLE ALTER COLUMN TYPE` changes are extremely limited and risky.

**MongoDB:** Schema migrations are code changes (add migration scripts to your application, run them as a job). Tools like `migrate-mongo` provide versioned script management.

---

## 10. Interview Drill – 5 System Design Scenarios

These are your practice scenarios. For each one, walk through your database design decisions out loud, then read the follow-up critique and questions. Treat these as a real interview — don't scroll to the critique before giving your own answer.

---

### Scenario 1: Global Ride-Sharing Platform

**The ask:** Design the database layer for a ride-sharing app (Uber/Lyft scale). Key features: driver location tracking (updates every 5 seconds per active driver), rider-driver matching, trip records, pricing, payments, ratings.

**Scale targets:** 5M active drivers globally, 50M rides/day, location updates at 50k writes/second peak.

**Your task:** Walk me through:
1. What databases you'd use and why.
2. How you'd store and query driver locations for the matching problem.
3. How you'd handle trip records and payments with ACID guarantees.
4. How you'd scale to support multiple geographic regions.

---

> #### Critique and Follow-up Questions

**Strong answer elements:**
- **Driver locations** should NOT be in PostgreSQL. 50k writes/sec + geospatial range queries ("find all drivers within 5km of this rider") — this is the canonical Redis use case. Redis with the `GEOADD`/`GEORADIUS` commands handles geospatial indexing natively in-memory. Driver positions are ephemeral (only current position matters), so in-memory is fine. Persistence is nice-to-have, not critical — a driver's location 5 seconds ago is irrelevant if they go offline.

- **Trips and payments** go in PostgreSQL with full ACID. A trip creation must atomically: create the trip record, reserve the driver, debit a payment hold. These span entities. Multi-statement transaction in PostgreSQL.

- **Trip records at 50M/day** (~580 writes/sec average, with spikes) — PostgreSQL can handle this with connection pooling (PgBouncer). Shard by `rider_id` or `driver_id` when you hit the write ceiling.

- **Multi-region:** Each region (US, EU, APAC) has its own Redis cluster for location data (location queries are regional — you don't need NYC drivers in a London matching query). PostgreSQL can be federated by region with a global events bus for cross-region reporting.

**Challenge questions:**
- A driver moves and Redis is updated, but the rider queries the matching service 100ms later and gets a stale location — what's your tolerance for this staleness and how do you bound it?
- If a payment fails after a trip is completed, how do you handle the compensation? (This is a saga pattern / compensation transaction question.)
- How do you prevent a driver from being matched to two riders simultaneously? What isolation mechanism prevents this race condition?

---

### Scenario 2: Multi-Tenant SaaS Analytics Platform

**The ask:** You're building a B2B analytics SaaS where enterprise customers upload event data and query dashboards. Tenants range from "startup" (10k events/day) to "enterprise" (10B events/day). You need to support custom dimensions, flexible schemas per tenant, and sub-second dashboard queries.

**Your task:**
1. How do you store event data for each tenant?
2. How do you handle schema flexibility (different tenants have different event properties)?
3. How do you isolate tenants from each other (performance and data isolation)?
4. What's your strategy for the analytical query layer?

---

> #### Critique and Follow-up Questions

**Strong answer elements:**
- **ClickHouse** is the right OLAP engine here. It ingests events at extreme rates, compresses well, and aggregates fast. Schema per tenant can be handled via a `tenant_id` partition key with a wide event table (`tenant_id`, `event_type`, `timestamp`, `properties` as a JSON/Map column).

- **Flexible schema** — ClickHouse's Map columns or JSON allow per-tenant custom properties without schema migrations per tenant. Alternatively, use a wide sparse table model where custom dimension columns are pre-allocated (column-sparse storage means unused columns cost almost nothing in ClickHouse).

- **Tenant isolation strategies** — three levels:
  - **Row-level isolation** (shared table, filter by `tenant_id`) — cheapest, risk of noisy-neighbor affecting query performance.
  - **Schema-level isolation** (one ClickHouse database per tenant) — good isolation, works up to ~thousands of tenants.
  - **Cluster-level isolation** (dedicated ClickHouse node per large tenant) — maximum isolation, appropriate for enterprise tenants with contractual SLA requirements.

- **Write path** — events flow: client → Kafka → Kafka consumer → ClickHouse batch insert (ClickHouse is optimized for batch inserts, not single-row inserts). Buffer tables or async inserts can handle the ingestion pattern.

**Challenge questions:**
- A large enterprise tenant runs a dashboard query that scans 50B rows. How do you prevent this from affecting other tenants on the same cluster?
- Your ClickHouse schema has a `properties` JSON column for custom events. A customer asks for a dashboard that filters on a specific property (`properties['user_plan'] = 'enterprise'`). How does this affect query performance and what's your mitigation?
- How do you implement data retention (enterprise customer wants 2 years of data, starter plan gets 90 days) in ClickHouse?

---

### Scenario 3: Real-Time Fraud Detection System

**The ask:** Design the database layer for a fraud detection system at a payment processor. The system must evaluate every transaction (50k/sec peak) against fraud rules in real time (<50ms decision latency). Rules depend on: transaction amount, merchant category, user's recent transaction history (last 30 days), device fingerprint, and graph relationships (connected accounts, shared devices).

**Your task:**
1. How do you store and query user transaction history at 50k TPS with <50ms latency?
2. How do you represent and query the graph of connected accounts/devices?
3. How do you store the trained ML model features and serve them at low latency?

---

> #### Critique and Follow-up Questions

**Strong answer elements:**
- **Transaction history** — Cassandra or ScyllaDB with partition key on `user_id` and clustering key on `transaction_time DESC`. You can read the last 30 days of transactions for a user in a single partition scan. Write throughput is handled by LSM-tree. This is precisely the access pattern Cassandra is designed for.

- **Graph relationships** — this is where a graph database shines. Neptune or TigerGraph for connected component traversal: "is this device shared by an account that was previously flagged?" You maintain a graph where nodes are users, accounts, devices, IPs and edges are "shared", "owns", "used_from". Traversal queries ("find all accounts 2 hops away from this user that are flagged") are O(edges) in a graph vs. multi-join explosion in SQL.

- **ML feature serving** — pre-computed features (user's rolling 30-day spend, merchant category risk score, device risk score) stored in Redis with TTLs. Features are updated asynchronously by a streaming pipeline; the fraud decision reads pre-computed features from Redis in <1ms rather than computing them on the fly.

- **Decision path at evaluation time:** Redis (feature lookup, <1ms) + Cassandra (recent transactions, ~5ms) + Neptune (graph traversal, ~10ms) → rule engine or ML model (<5ms) → decision logged to PostgreSQL for case management and model retraining. Total: ~20-25ms, well within 50ms budget.

**Challenge questions:**
- Your graph traversal shows that account A is connected to flagged account B, but through 4 hops. At what hop depth do you stop trusting the signal, and how do you prevent the supernode problem (a popular merchant that's connected to millions of accounts)?
- A user disputes a transaction that your system approved. How do you reconstruct exactly what features and rules were evaluated at decision time to audit the decision?
- Your Cassandra cluster in us-east-1 has a transient partition. How does your fraud detection system behave during this period?

---

### Scenario 4: Collaborative Document Editor

**The ask:** Design the database layer for a Google Docs-style collaborative editor. Multiple users edit the same document simultaneously. You need to support: real-time concurrent editing, version history (full document history recoverable), offline editing with sync on reconnect, and full-text search across all documents.

**Your task:**
1. What data model do you use for document storage?
2. How do you handle concurrent edits and conflict resolution?
3. How do you implement version history efficiently?
4. How do you support full-text search at scale?

---

> #### Critique and Follow-up Questions

**Strong answer elements:**
- **Operations log, not document snapshots.** This is the canonical event sourcing use case. Every edit is an operation (`insert at position X`, `delete from position X to Y`). The document is reconstituted by replaying operations. **CRDTs (Conflict-free Replicated Data Types)** — specifically sequence CRDTs like YATA or the one used in Yjs — are the right conflict resolution mechanism. They allow concurrent edits to merge deterministically without coordination.

- **Document storage:** The raw CRDT state (which is the authoritative document model) stored in PostgreSQL. Operations streamed through a pub/sub layer (Redis Pub/Sub or Kafka) for real-time collaboration. Periodic snapshots to avoid replaying 100k operations to load a large document.

- **Version history:** The append-only operations log is your version history. Tagging a point in the log with a version label (`v1.0`, `submitted for review`) makes specific points navigable. Storing compressed snapshots every N operations keeps replay time bounded.

- **Full-text search:** OpenSearch or Elasticsearch with the latest document state indexed asynchronously. Use CDC on the document snapshot table to trigger index updates. For PostgreSQL-first shops, a GIN index on `tsvector` is viable for millions of documents.

- **Offline editing:** The CRDT model handles this gracefully — offline operations are buffered client-side; on reconnect, they are merged into the shared CRDT state using the standard conflict-free merge algorithm.

**Challenge questions:**
- A user edits a document offline for 3 hours, making 500 operations. They reconnect and there are 2000 operations from other users during that period. Walk me through the merge process and what the user experiences.
- Your search index shows a paragraph that was actually deleted 2 minutes ago. The user searches, clicks a result, and sees a different document than they expected. How do you handle eventual consistency in search?
- Version history shows 10,000 operations for a single document over 2 years. How do you make "view this document as it was on March 1st" performant?

---

### Scenario 5: IoT Sensor Data Platform

**The ask:** You're building the backend for a smart building platform. 100,000 buildings, each with 500 sensors (temperature, humidity, occupancy, energy usage). Sensors report every 30 seconds. That's ~1.67M data points/second sustained. You need to support: real-time alerting (anomaly detection), historical trend analysis (last 12 months by building, floor, sensor), and a dashboard showing current state of all sensors in a building.

**Your task:**
1. What database(s) do you use for ingestion and storage?
2. How do you model the data for the three different query patterns?
3. How do you handle data retention and downsampling?
4. How do you serve the "current state" dashboard at low latency?

---

> #### Critique and Follow-up Questions

**Strong answer elements:**
- **Ingestion and historical storage** — TimescaleDB (PostgreSQL extension) or InfluxDB for the time-series data. TimescaleDB with hypertables partitioned by time and building_id handles this ingestion rate on a reasonably sized cluster. InfluxDB if you want purpose-built time-series with better native compression and retention management.

- **The three query patterns require different optimizations:**
  1. *Real-time alerting:* Stream processing (Kafka Streams, Apache Flink) consuming raw sensor data, running anomaly detection rules, firing alerts. The DB isn't involved in the hot path — alerts are generated from the stream, then stored in PostgreSQL for case management.
  2. *Historical trend analysis:* TimescaleDB continuous aggregates — pre-compute hourly and daily averages per sensor at write time. `SELECT avg(temperature) FROM sensor_hourly WHERE building_id = X AND time > NOW() - INTERVAL '30 days'` hits a tiny aggregated table, not 30 days of raw readings.
  3. *Current state dashboard:* Redis. Each sensor's latest reading is a Redis key (`sensor:{id}:latest`). The sensor ingestion pipeline writes to both TimescaleDB (durable) and Redis (cache, low-latency reads). The dashboard reads from Redis in <1ms per sensor, handling 500 sensors per building in a single pipeline call.

- **Data retention and downsampling:** TimescaleDB data retention policies: keep raw data for 7 days, hourly aggregates for 90 days, daily aggregates forever. Drop raw chunks automatically. This is built into TimescaleDB's retention policy API.

**Challenge questions:**
- Building X has a sensor that's been offline for 6 hours and just reconnected with 720 buffered readings. How does your ingestion pipeline handle this burst and does it affect the "current state" Redis cache?
- Your continuous aggregate for daily building energy usage has a bug — it was double-counting one sensor type for 3 months. How do you correct historical data in a time-series model that's designed for append-only workloads?
- 100 buildings in the same region are all reporting anomalous temperature readings simultaneously (a city-wide HVAC failure event). How does your alerting system handle this thundering herd?

---

## Quick-Reference Cheat Sheet

### Data Model Selection Matrix

| Use Case | First Choice | Second Choice | Avoid |
|---|---|---|---|
| Financial transactions | PostgreSQL | CockroachDB | MongoDB |
| User activity feeds | Cassandra | DynamoDB | PostgreSQL |
| Real-time metrics | Redis + TimescaleDB | InfluxDB | MySQL |
| Full-text search | Elasticsearch | PostgreSQL + GIN | DynamoDB |
| Semantic / vector search | pgvector | Qdrant | MySQL |
| Graph traversals | Neo4j | Neptune | Cassandra |
| Event analytics | ClickHouse | BigQuery | PostgreSQL |
| Session / cache | Redis | Memcached | PostgreSQL |
| Multi-tenant OLAP | ClickHouse | Snowflake | MongoDB |
| Audit log / event store | PostgreSQL (events table) | EventStoreDB | Cassandra |

---

### Key Numbers to Know

| Metric | Ballpark |
|---|---|
| PostgreSQL max write TPS (single node) | ~50k–100k |
| Redis ops/sec (single node) | ~500k–1M |
| Cassandra write throughput (cluster) | 1M+ writes/sec |
| ClickHouse ingestion | 500k–1M rows/sec per server |
| B-tree index write amplification | O(log N) page writes per insert |
| LSM-tree write amplification | 10–30x (due to compaction) |
| Postgres WAL replication lag (async) | Typically <1 second in healthy cluster |
| DynamoDB single-item read latency | ~1–5ms (eventually consistent) |
| Redis get/set latency | <0.1ms (same AZ) |
| Pinecone ANN query latency (10M vectors) | ~20–50ms |
| Qdrant ANN query latency (10M vectors) | ~5–15ms |

---

### CAP/PACELC Quick Reference

| Database | CAP | Normal trade-off | Notes |
|---|---|---|---|
| PostgreSQL (sync replica) | CP | High C, higher latency | Sync replica blocks commit until replica confirms |
| PostgreSQL (async replica) | AP | Low latency, stale reads possible | Default; fine for most read workloads |
| Cassandra | AP | Tunable (ONE to ALL) | Default is eventual; QUORUM for stronger guarantees |
| DynamoDB (eventually consistent) | AP | Low latency | Default read mode |
| DynamoDB (strongly consistent) | CP | Higher latency | Forces read to primary |
| CockroachDB | CP | Higher latency | Raft consensus on every write |
| Redis (single) | CP | Very low latency | Single node, no partition concern |
| MongoDB (replica set, writeConcern majority) | CP | Higher write latency | Waits for majority replica acknowledgment |

---

### Migration Safety Rules

1. Never run `ALTER TABLE` without checking lock level first.
2. Always use `CREATE INDEX CONCURRENTLY` in production.
3. Adding `NOT NULL` columns: set a default (PostgreSQL 11+ is safe), OR do it in two steps — add nullable, backfill, add constraint with `NOT VALID`, validate separately.
4. Use expand-contract for any column rename or type change.
5. Test migrations on a production-size data copy (using Neon branching or a staging clone) before running in production.
6. Have a rollback plan. Usually: the expand phase is the rollback point — if the new code is bad, roll back the code, the old column still exists.
7. Never run backfill migrations as a single `UPDATE` on a large table — batch in 1k–10k row chunks with `pg_sleep(0.01)` between batches to avoid lock contention and WAL flooding.

---

*Last updated: April 2026. Database technology moves fast — verify specific version details for any production decision.*
