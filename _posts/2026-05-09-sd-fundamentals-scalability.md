---
title: "System Design Fundamentals - Scalability & Availability"
date: 2026-05-09
categories: [System Design]
tags: [scalability, availability, consistency]
---

## What It Is

**Scalability** is the ability of a system to handle an increasing workload, either by adding more resources (scaling out) or by upgrading the capacity of existing resources (scaling up). In distributed systems, scalability is essential to manage growing demands of users, data, and processing power.

---

## Horizontal Scaling (Scale Out)

Involves adding more machines or nodes to distribute the workload evenly. Allows the system to handle more requests without overloading individual nodes. Particularly useful in distributed systems — provides a cost-effective way to manage fluctuating workloads and maintain high availability.

*Example: Cassandra and MongoDB scale horizontally by adding more machines to meet growing needs — no downtime, no manual reconfiguration.*

---

## Vertical Scaling (Scale Up)

Refers to increasing the capacity of individual nodes by upgrading hardware — more CPU, memory, or storage. Allows a single node to handle more workload. Has a physical limit on how much can be added to one machine, and can lead to single points of failure.

*Example: MySQL scales vertically by switching from smaller to bigger machines. However, this process often involves downtime.*

---

## Horizontal vs. Vertical

Horizontal scaling is easier to scale dynamically — just add more machines to the existing pool. Vertical scaling is limited to the capacity of a single server; scaling beyond that capacity often involves downtime and comes with an upper limit.

---

## Availability

**Availability** is a measure of how accessible and reliable a system is to its users. In distributed systems, high availability ensures the system remains operational even in the face of failures or increased demand. Downtime leads to financial losses and reputational damage — high availability is a critical requirement across industries.

High availability is measured as **uptime** — the ratio of time a system is operational to the total time it is supposed to be operational. Achieving it involves minimizing planned and unplanned downtime, eliminating single points of failure, and implementing redundant systems.

In distributed systems, high availability also means the system can handle increased load without compromising performance — not just staying up, but staying performant under demand spikes.

---

## Strategies for Achieving High Availability

### 1. Redundancy and Replication

Duplicate critical components so that if one fails, a redundant system takes over seamlessly with no interruption. Replication creates multiple copies of data so it remains available even if one copy becomes inaccessible.

*Example: In data centers, multiple servers handle the same workload. If one crashes, a redundant server takes over — users notice nothing.*

### 2. Load Balancing

Distribute workloads across multiple servers so no single server is overwhelmed. Intelligent load-balancing algorithms optimize resource utilization, prevent bottlenecks, and enhance availability by evenly spreading traffic.

*Example: In a web app with thousands of concurrent users, a load balancer distributes requests across a fleet of servers — no single server becomes overloaded.*

### 3. Distributed Data Storage

Store data across multiple locations or data centers to reduce the risk of data loss or corruption. Data is replicated across geographically diverse locations — if one site goes down, data remains accessible from others.

*Example: A global e-commerce platform replicates its order database across three regions. A full data center outage in one region doesn't cause data loss or downtime.*

### 4. Health Monitoring and Alerts

Proactively identify and address issues before they impact availability. Real-time monitoring and automated alerts enable rapid response, minimizing downtime.

Health monitoring continuously tracks system performance, resource utilization, and key metrics. Alerts fire when predefined thresholds are exceeded, triggering immediate remediation before users are affected.

### 5. Regular System Maintenance and Updates

Keep systems up to date with patches, security enhancements, and bug fixes to mitigate failures and vulnerabilities. Routine hardware inspections, software updates, and component checks ensure all components function correctly.

### 6. Geographic Distribution

Deploy system components across multiple regions or data centers. If one region experiences an outage, users are served from other locations with no interruption.

*Example: A SaaS product with users in the US, EU, and Asia deploys in three regions. A full AWS `us-east-1` outage is transparent to users — traffic fails over to `eu-west-1` and `ap-southeast-1`.*

---

## Consistency Models

Consistency models define how a distributed system maintains a coherent view of data across all replicas. Each model trades off between availability, performance, and data correctness.

### Strong Consistency

All replicas have the same data at all times. Every read reflects the most recent write, regardless of which node handles the request. Achieved by requiring all replicas to acknowledge a write before it is considered complete.

- Guarantees: reads always return the latest write.
- Cost: reduced availability and higher latency. The system must wait for all replicas to acknowledge a write before returning a response. During that window, reads to the same row are blocked until replication completes. If any replica is unreachable — due to a slow node, crash, or network partition — the system rejects the request entirely rather than risk returning stale data. This means strong consistency directly trades availability for correctness.
- When to use: financial transactions, inventory systems, anything where stale reads cause correctness problems.

*Example: A bank transfer. After debiting account A and crediting account B, any read from any node must reflect the new balances immediately. A stale read showing the old balance could allow a double-spend.*

*Real system: Google Spanner achieves strong consistency globally using TrueTime — atomic clocks + GPS to bound clock uncertainty and order transactions correctly across data centers.*

### Weak Consistency

The system makes no guarantee that a read will reflect the most recent write. Replicas may serve stale data. No synchronization is enforced after a write.

- Guarantees: none on read recency.
- Cost: none — writes are fast, availability is high.
- When to use: scenarios where stale data is acceptable and performance/availability matter more than correctness.

*Example: Live video streaming view counts. Whether a viewer sees "1.2M views" or "1.19M views" doesn't matter. Blocking every read for global consistency would destroy performance.*

*Real system: Memcached operates with weak consistency — it's a cache, not a source of truth. Stale cache entries are acceptable because the DB is always the authoritative source.*

### Eventual Consistency

All replicas will eventually converge to the same value — but reads may return stale data in the window between a write and full propagation. Given no new writes and enough time, all nodes will agree.

- Guarantees: convergence, not recency. The "eventually" window is typically milliseconds to seconds.
- Cost: temporary inconsistencies. Conflicts can arise if two replicas accept writes to the same key simultaneously — requires a conflict resolution strategy (last-write-wins, vector clocks, CRDTs).
- When to use: social feeds, shopping carts, DNS, anything where brief staleness is acceptable and availability must not be compromised.

*Example: You post a tweet. Your followers in Asia may not see it for a few hundred milliseconds while it propagates across replicas. The system eventually converges and everyone sees the tweet.*

*Real system: Amazon DynamoDB defaults to eventual consistency for reads (with an option for strongly consistent reads at higher cost). Cassandra prioritizes availability and partition tolerance, eventual consistency by default.*

### Weak vs. Eventual Consistency

Both allow stale reads, but the key difference is the **guarantee of convergence**:

- Weak consistency makes no promise that replicas will ever agree. A write may propagate to some replicas but never all. The system doesn't attempt to reconcile differences.
- Eventual consistency guarantees that all replicas *will* converge to the same value, given no new writes and enough time. Propagation is still async, but the system actively replicates and resolves conflicts.

Weak consistency is "fire and forget" — the system doesn't track or ensure propagation. Eventual consistency is "async but guaranteed" — propagation happens in the background and conflicts are resolved via a defined strategy (last-write-wins, vector clocks, CRDTs).

*Example: Memcached (weak) — a cache entry is updated on one node, other nodes may never see it until the cache is invalidated or expires. No reconciliation happens. DynamoDB (eventual) — a write to one replica is guaranteed to propagate to all replicas; any replica serving a stale read will eventually serve the correct value.*

In practice, weak consistency is used where the data source of truth lives elsewhere (a cache in front of a DB). Eventual consistency is used where the replicated store itself is the source of truth and correctness must eventually be guaranteed.

**Performance cost:**

Weak consistency has virtually zero coordination cost — write to one node, done. There is no replication between nodes at all (each key lives on exactly one node). No background sync, no conflict resolution, no replication overhead.

Eventual consistency has a real coordination cost — after a write, the system replicates it to all replicas in the background, consuming network I/O, CPU, and storage writes on every replica. Conflict resolution adds further overhead when two replicas accept writes to the same key simultaneously. Reads are still fast (no blocking), but the replication pipeline is always running.

Weak consistency is cheaper because replication simply doesn't exist. Eventual consistency pays the cost of background replication on every write to guarantee convergence.

**Drift:**

Weak consistency has no background sync, so replicas can drift arbitrarily far apart over time with no reconciliation. This is only acceptable when no consequential decision is made on the stale value.

*Example: A cricket scorecard displays "India: 245/3" on Replica 1 and "India: 243/3" on Replica 2. User A and User B see different scores at the same moment. Neither user is harmed — they are passively watching, no transaction or action is triggered by reading the stale score, and the value self-corrects on the next read when the replica catches up. The business impact is zero.*

*Contrast with a bank balance — if two replicas drift and both show a stale balance of ₹10,000, two users could each withdraw ₹9,000 simultaneously against the same account, both succeeding because neither replica knew about the other's pending write. Real money is lost. Drift had a real-world consequence.*

The rule: drift is harmless when the stale value is consumed passively with no consequential action triggered. The moment a read result drives a write or a decision with real-world impact, drift becomes dangerous. Weak consistency is never appropriate when the replicated store itself is the source of truth.

### The Core Trade-off — CAP Theorem

The CAP theorem states that a distributed system can only guarantee two of three: Consistency, Availability, Partition Tolerance. Since network partitions are unavoidable in real distributed systems, the real choice is between **consistency and availability** during a partition.

- Strong consistency + partition tolerance → sacrifices availability (requests are rejected or blocked during a partition).
- Eventual consistency + partition tolerance → sacrifices strong consistency (stale reads allowed, system stays up).

*Example: During a network partition between two data center replicas — a strongly consistent system rejects both reads and writes on the isolated node to avoid divergence. An eventually consistent system accepts writes on both nodes and resolves conflicts after reconnection.*

---

## Latency and Performance

Latency and performance directly impact user experience and a system's ability to handle large volumes of data and traffic. The three primary levers are data locality, load balancing, and caching.

### Data Locality

Organize and distribute data to minimize data transfer between nodes. Store related data close together, or near the nodes that access it most frequently — reducing retrieval latency and improving throughput. Achieved via partitioning, sharding, and replication.

*Example: In Cassandra(OLAP), rows frequently queried together are stored on the same partition using a composite partition key — a cross-node read that would add network RTT is avoided entirely.*

### Load Balancing

Distribute incoming traffic or compute workload across multiple nodes so no single node is overwhelmed. Optimizes resource utilization, minimizes response times, and prevents overloads. Common algorithms:

- **Round-robin** — requests distributed sequentially across nodes; works well when all nodes are homogeneous and requests are roughly equal in cost.
- **Least connections** — new request goes to the node with the fewest active connections; better when request cost varies significantly.
- **Consistent hashing** — a node is selected deterministically based on a hash of the request key; minimizes redistribution when nodes are added or removed, critical for cache affinity and stateful workloads.

*Example: An API gateway uses consistent hashing to route requests for the same user ID to the same backend node — keeping that user's session cache warm and avoiding redundant DB lookups.*

### Caching Strategies

Store frequently accessed data or precomputed results temporarily so the system can serve them without hitting the primary data source. Dramatically reduces latency and backend load. Common strategies:

- **In-memory caching** — data stored in RAM on the application node or a dedicated cache node (e.g., Redis, Memcached). Sub-millisecond reads. Limited by available memory; volatile.
- **Distributed caching** — cache is spread across a cluster of nodes, partitioned by key. Scales horizontally; survives individual node failures. Used when the working set exceeds single-node memory.
- **CDN (Content Delivery Network)** — static assets (images, JS, HTML) are cached at edge nodes geographically close to users. Eliminates long-haul network latency for content that rarely changes.

*Example: An e-commerce product page is cached in Redis with a 60-second TTL. 10,000 concurrent users requesting the same page hit the cache — the database receives almost no read traffic, and p99 latency drops from ~200ms to ~2ms.*

---

## Interview Pitfalls

- Treating availability and reliability as the same thing — availability is about uptime; reliability also includes correctness and consistent behavior under load.
- Saying "strong consistency" without acknowledging the latency and availability trade-off — interviewers will probe this.
- Not knowing when eventual consistency causes real problems — it is not suitable for financial systems, inventory counts, or any scenario where two actors read-then-write the same record.
- Confusing replication and redundancy — replication is about data copies; redundancy is about component copies.
- Ignoring conflict resolution in eventual consistency — "what happens if two nodes accept writes to the same key simultaneously?" is a standard follow-up.
