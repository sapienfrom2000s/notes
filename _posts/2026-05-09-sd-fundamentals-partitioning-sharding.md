---
title: "System Design Fundamentals - Partitioning, Sharding & DB Scaling"
date: 2026-05-09
categories: [System Design]
tags: [partitioning, sharding, olap, oltp, mysql]
---

## Starting Point

A single MySQL instance handling all reads and writes. As traffic grows, this becomes the bottleneck. The following sections cover the optimizations — in the order you would actually introduce them.

---

## OLTP vs. OLAP

Before scaling, understand what you are scaling — because the solution differs.

**OLTP (Online Transaction Processing)**
Handles high-frequency, short-lived transactions — inserts, updates, deletes, and point reads. Data is normalized to avoid redundancy. Optimized for write throughput and transactional correctness (ACID).

- Queries: `SELECT * FROM orders WHERE order_id = 123`, `INSERT INTO payments ...`
- Row-oriented storage — fetches entire rows quickly.
- Examples: MySQL, PostgreSQL, Aurora.

**OLAP (Online Analytical Processing)**
Handles complex analytical queries that scan large volumes of data — aggregations, GROUP BY, time-series analysis. Data is denormalized (wide tables, star/snowflake schema). Optimized for read throughput over large datasets.

- Queries: `SELECT product_id, SUM(revenue) FROM orders GROUP BY product_id WHERE date > '2024-01-01'`
- Column-oriented storage — scans only the columns needed, ignores the rest. Far more efficient for aggregations.
- Examples: Redshift, BigQuery, Snowflake, ClickHouse.

**The core difference** — OLTP reads/writes individual rows frequently. OLAP scans millions of rows across a few columns rarely. Running analytical queries on an OLTP database is expensive — a `GROUP BY` across 100M rows locks resources and starves transactional queries. This is why they are eventually split into separate systems.

*Example: An e-commerce platform uses MySQL (OLTP) for placing orders and updating inventory. A separate Redshift cluster (OLAP) runs nightly revenue reports and cohort analysis — analytical queries never touch the production MySQL instance.*

---

## Order of Optimizations

This is the sequence you follow as a system scales. Each step has a cost — don't jump ahead prematurely.

### Step 1 — Single MySQL Instance

The starting point. One primary handling all reads and writes. Sufficient for low traffic. Simple to operate.

**Bottleneck:** reads and writes compete for the same resources. Read-heavy workloads slow down writes and vice versa.

### Step 2 — Add a Read Replica

Add one or more read replicas. Writes go to the primary; reads are distributed across replicas. Replication is asynchronous — replicas are eventually consistent with the primary.

*Example: An e-commerce site where 90% of traffic is product browsing (reads) and 10% is checkout (writes). Three read replicas absorb all browse traffic; the primary handles only writes. Primary CPU drops from 80% to 20%.*

**What this solves:** read scalability, read availability (replica can serve reads if primary is slow).
**What it doesn't solve:** write scalability (all writes still go to one primary), storage limit (each replica holds a full copy of the data).

### Step 3 — Add a Caching Layer

Put Redis or Memcached in front of the DB. Serve repeated reads from memory. Eliminates DB hits for hot data entirely.

*Example: Product detail pages are read thousands of times per second but updated rarely. Cache the product row in Redis with a 5-minute TTL. DB read load drops by 80% instantly.*

**What this solves:** read throughput for hot/repeated data, latency.
**What it doesn't solve:** write throughput, cold reads (cache miss still hits DB), storage.

### Step 4 — Partitioning (Within a Single Instance)

Divide a large table into smaller, manageable pieces within the same MySQL instance. The DB engine handles routing transparently — the application queries the table normally.

**Horizontal Partitioning (most common)**
Split rows across partitions based on a partition key.

- **Range partitioning** — rows are assigned to partitions based on a range of values. `orders` table partitioned by `created_at`: Jan–Mar in partition 1, Apr–Jun in partition 2, etc. Old partitions can be archived or dropped cheaply. Best for time-series data.
- **List partitioning** — rows are assigned based on discrete values. `orders` partitioned by `region`: `IN`, `US`, `EU` each in their own partition. Best when data maps to a known, finite set of categories.
- **Hash partitioning** — `hash(partition_key) % n` assigns rows to partitions. Distributes data evenly without any natural range or category. Best when no obvious range or list exists and you just want even distribution.

**Vertical Partitioning**
Not a MySQL feature — not supported by any RDBMS natively. It is a manual schema design pattern where you split a wide table into multiple narrower tables yourself and join them when needed. The DB engine has no awareness of it.

*Example: A `users` table has 40 columns. Most queries only need `id`, `email`, `name`. You manually move blob columns (`profile_picture`, `bio`, `preferences`) to a `user_details` table. The hot path never touches the heavy columns — but this is your design decision, not something MySQL enforces or manages.*

The spirit of "only read the columns you need" is natively solved by **columnar OLAP databases** (Redshift, BigQuery, ClickHouse) — they physically store each column separately on disk, so a query touching 3 out of 40 columns reads only those 3 columns' data. MySQL always reads the full row off disk regardless.

**What this solves:** query performance (partition pruning — MySQL scans only relevant partitions), manageability (drop old partitions cheaply), index size.
**What it doesn't solve:** write throughput beyond a single instance, storage beyond a single machine.

### Step 5 — Sharding (Across Multiple Instances)

When a single MySQL instance — even with replicas and partitioning — can no longer handle the write load or the dataset exceeds single-machine storage, split the data across multiple independent MySQL instances. Each instance (shard) holds a subset of the data and has its own read replicas.

The application (or a proxy layer like Vitess) is responsible for routing queries to the correct shard.

**Shard key selection** is the most critical decision:
- Must distribute data evenly across shards — a bad key creates hot shards.
- Must align with your most common query pattern — queries that don't include the shard key require scatter-gather (fan out to all shards), which is expensive.
- Cannot be changed easily after the fact — resharding is painful.

*Example: Shard the `orders` table by `user_id`. `hash(user_id) % 4` gives 4 shards. All orders for a given user live on one shard — queries like "fetch all orders for user 123" hit exactly one shard. Each shard has its own primary + read replicas.*

**Cross-shard queries** — queries that don't include the shard key must fan out to all shards and merge results. Expensive. Design your shard key to minimize these.

*Example: "Fetch all orders placed in India today" — if sharded by `user_id`, this must query all shards and merge. If this is a frequent query, `region` would have been a better shard key.*

**Hotspot problem** — if one shard key value receives disproportionate traffic, that shard becomes a hotspot while all others are idle.

*Example: A celebrity user with 50M followers has all their data on one shard (shard key is `user_id`). Every follower read, every new post write, every notification hits that one shard.*

**Composite keys** — instead of sharding by `user_id` alone, append a random suffix to the key (`user_id + suffix 0 to N-1`). The hot user's data is spread across N shards instead of one.

*Example: User `123` with suffix range `0–9` maps to keys `123_0` through `123_9`, each hashing to a different shard. Writes pick a random suffix and distribute across shards. Reads must query all 10 sub-keys and merge results — you eliminate write hotspots at the cost of read scatter-gather overhead. Best for write-heavy hotspots.*

**Sub-sharding** — when an existing shard becomes hot, split it further using a secondary key. The hot shard is divided into N child shards without redesigning the entire sharding scheme.

*Example: Shard 3 holds users `300k–400k` and is overloaded. Sub-shard by `user_id % 4` — Shard 3 splits into 3A, 3B, 3C, 3D. Each child shard now handles 25k users instead of 100k. The rest of the shards are untouched. Sub-sharding is done reactively as hotspots emerge.*

**Add read replicas to the hot shard** — if the hotspot is read-heavy (millions of followers reading a celebrity's feed), add more read replicas specifically to that shard without restructuring data.

**Dedicated shard for known hot keys** — pre-identify high-traffic entities and isolate them in their own shard with more resources, rather than letting them compete with regular users.

**Rebalancing** — adding a new shard requires moving data from existing shards to the new one. Consistent hashing minimizes the data that needs to move.

**What this solves:** write scalability (writes distributed across shards), storage (each shard holds a fraction of the data).
**What it introduces:** cross-shard query complexity, no cross-shard joins, distributed transactions become hard, operational overhead multiplied by number of shards.

### Step 6 — Separate OLAP from OLTP

At scale, analytical queries (reports, dashboards, aggregations) running on the MySQL cluster slow down transactional workloads. Move analytics to a dedicated OLAP system.

Stream data from MySQL to the OLAP store via CDC (Change Data Capture) using tools like Debezium or AWS DMS. The OLAP store receives a real-time (or near-real-time) copy of the data, transformed into a schema optimized for analytical queries.

*Example: MySQL (OLTP) handles all order transactions. Debezium captures every insert/update from the MySQL binlog and streams it to Redshift (OLAP). The finance team runs revenue reports on Redshift — complex GROUP BY queries that would take minutes on MySQL complete in seconds on Redshift and never touch production.*

---

## Interview Pitfalls

- Jumping straight to sharding — interviewers expect you to exhaust simpler options first (replicas, caching, partitioning) before sharding.
- Choosing the wrong shard key — always reason about your most common query pattern. A key that causes every query to scatter-gather defeats the purpose of sharding.
- Forgetting that sharding breaks joins — cross-shard joins don't exist. Data that is frequently joined should live on the same shard.
- Running analytical queries on OLTP — always flag this as a problem at scale and propose a separate OLAP pipeline.
- Not knowing the difference between partitioning and sharding — partitioning is within one instance (transparent to the app), sharding is across multiple instances (app must be aware).
