---
title: "System Design Fundamentals - Concurrency & Coordination in Distributed Systems"
date: 2026-05-10
categories: [System Design]
tags: [concurrency, coordination, consistency, locking, synchronization, consensus]
---

## Why this whole topic exists

Picture a single computer running one program. Memory has one copy of every variable, the CPU executes instructions one at a time, and a write you just made is *immediately* visible to the next read. Life is simple.

Now picture **Amazon on Black Friday**:
- Thousands of servers across continents.
- Each server holds a *copy* of the same product catalog and inventory.
- A user in Tokyo and a user in Berlin both click "Buy" on the last PlayStation 5 in stock at the same millisecond.
- The two requests hit two different servers, which don't know about each other yet.

Without rules, both servers say "yes, you got it" and Amazon now has to apologize to one customer, refund them, and damage trust. **Concurrency and coordination is the set of techniques to prevent this kind of disaster** while still serving millions of users in parallel.

Two distinct problems show up:
1. **Concurrency control** → "How do I stop two operations from corrupting the same piece of data?" (a *data* problem).
2. **Synchronization** → "How do I make sure operations happen in the right *order* or at the right *time*?" (a *timing* problem).

They sound similar. Here's the difference with a cooking example:
- **Concurrency control:** Two chefs can't both reach for the same knife at once — they'd cut each other. Whoever grabs the knife first uses it; the other waits.
- **Synchronization:** The frosting can only go on *after* the cake has cooled. The frosting chef has to wait for a signal from the baking chef, even though they're using totally different tools.

---

## A. Concurrency Control — protecting shared data

### The fundamental problem: the "Lost Update"

Walk through this slowly:

- Account balance is `$100`.
- **Server A** reads balance → sees `$100` → subtracts `$30` → writes `$70`.
- **Server B** reads balance → sees `$100` (because A hasn't written yet) → subtracts `$50` → writes `$50`.
- Final balance: `$50`. The customer just spent `$80` but only `$50` was deducted. The bank loses `$30`.

This is called a **race condition** — the result depends on who "races" to finish first. Concurrency control exists to make this kind of bug impossible.

There are three main families of solutions, each born from the failures of the previous one.

---

### Family 1: Pessimistic Locking — "Assume the worst"

**The idea:** Before you touch shared data, claim exclusive ownership of it. Nobody else can touch it until you let go.

**Mental picture:** Imagine a single bathroom in an office. You lock the door from the inside. Anyone else who wants in has to wait outside. When you're done, you unlock and the next person enters.

#### Walking through the bank example with locks:
- Server A says: "I want to modify the balance row." → It acquires a **lock** on that row.
- Server B says the same thing → it's told "wait."
- Server A reads `$100`, subtracts `$30`, writes `$70`, releases the lock.
- Server B is now allowed in. It reads `$70` (the *correct* current value), subtracts `$50`, writes `$20`.
- Final balance: `$20`. Correct!

#### Where you actually see this:
- **PostgreSQL:** `SELECT * FROM accounts WHERE id = 5 FOR UPDATE;` → locks that row until your transaction commits.
- **MySQL InnoDB:** Same `FOR UPDATE` syntax, plus implicit row locks during `UPDATE`.
- **Redis distributed lock:** `SET lockkey "owner-id" NX EX 30` → "set this key only if it doesn't exist, expire in 30s." If you got the key, you hold the lock.
- **Java:** `synchronized` keyword.
- **Linux kernel:** `mutex_lock()` for mutual exclusion.

#### Different *flavors* of locks (each solving a sub-problem):

- **Shared lock (read lock):** Multiple readers can hold it together. Why? Because reading doesn't change anything, so reads don't conflict with each other. Example: 100 servers can read a config row at once.
- **Exclusive lock (write lock):** Only one holder. Blocks both readers and writers. Used when you're going to mutate the data.
- **Two-phase locking (2PL):** A transaction first *acquires* all the locks it needs (growing phase), then once it releases any lock, it can't acquire any new ones (shrinking phase). This guarantees serializable transactions. Most relational databases use a variant of 2PL internally.

#### The dark side of locking — what goes wrong in production:

- **Deadlock:** Server A holds lock on row 1 and wants row 2. Server B holds lock on row 2 and wants row 1. They wait for each other forever.
  - *Real-world example:* Money transfer between two accounts. Transfer A→B locks A then B. Transfer B→A locks B then A. They deadlock.
  - *Fix:* Always acquire locks in a consistent global order (e.g., always lock the lower account ID first). Or use deadlock detection that kills one transaction.

- **Lock convoys:** Many threads pile up waiting for the same hot lock, throughput collapses to one operation at a time.
  - *Real-world example:* Every user incrementing the same global counter. Even with billions of CPU cycles available, you can only increment it ~millions of times per second because everything serializes through that one lock.

- **Stale locks (in distributed systems):** Server A acquires a Redis lock, then crashes. The lock sits there held by a corpse. Eventually the TTL (time-to-live) expires and someone else takes it. But what if Server A wakes up after a long pause and *thinks* it still holds the lock? It writes data, corrupting whatever Server B has done. This is the **fencing token problem** — the standard fix is to attach a monotonically increasing token to every operation, so the storage layer rejects stale writes.

- **Latency:** Every lock acquisition is a network round-trip in distributed systems. Locks make systems slower.

**One-sentence summary:** Pessimistic locking is correct and simple, but slow, prone to deadlocks, and dangerous when holders crash.

---

### Family 2: Optimistic Concurrency Control (OCC) — "Assume conflicts are rare"

**Why it was invented:** Locks waste enormous time waiting in scenarios where collisions almost never happen. Imagine Wikipedia: millions of articles, edits scattered across them — two people editing the *same* article at the same second is rare. Why pay the lock cost on every edit?

**The idea:** Don't lock anything. Just read the data, do your work, and right before you commit, check: "Did anyone else change this while I was working?" If yes, throw away your work and start over. If no, commit.

**Mental picture:** Instead of locking the bathroom door, you walk in, do your business, and on the way out you check a sticky note on the door. If the note says "still version 5 like when I came in," you write "now version 6" and leave. If someone else has updated the note to version 6, you realize someone went in after you started — you throw away whatever you did and try again.

#### Walking through the bank example with OCC:
- The balance row has a hidden field `version = 5`.
- Server A reads balance `$100`, version `5`. It computes `$70`.
- Server B also reads balance `$100`, version `5`. It computes `$50`.
- Server A writes: `UPDATE accounts SET balance = 70, version = 6 WHERE id = 5 AND version = 5;` → succeeds (1 row updated).
- Server B writes: `UPDATE accounts SET balance = 50, version = 6 WHERE id = 5 AND version = 5;` → fails (0 rows updated, because version is now 6).
- Server B sees the failure, *re-reads* (`$70`, version `6`), recomputes `$70 - $50 = $20`, writes with `version = 7`. Succeeds.

Notice nobody ever blocked. Both ran in parallel; only the *commit* of the loser was rejected.

#### Real-world examples:
- **DynamoDB conditional writes:** `PutItem` with `ConditionExpression: "version = :v"`. If the condition fails, the write is rejected and you retry.
- **Git:** When you `git push`, Git checks "is the remote still at the commit I branched from?" If someone pushed in between, it rejects with "non-fast-forward, please pull and merge." That's OCC.
- **JPA / Hibernate:** `@Version` annotation on an entity field — Hibernate appends `WHERE version = ?` to every update.
- **HTTP `If-Match` header with ETags:** Browser sends `If-Match: "abc123"` when updating; server rejects with `412 Precondition Failed` if the ETag has changed.
- **Google Docs in earlier versions** (before they switched to operational transformation) used something like this for conflict detection.

#### Where OCC breaks down:
- **High-conflict workloads kill it.** Imagine a "global counter" — every increment conflicts with every other. With OCC, you'd retry, retry, retry, in an exponential pile-up. Locking is actually faster here because at least it doesn't waste work.
- **Wasted work on retries.** If a transaction did 30 minutes of computation and then loses the version check, all 30 minutes are thrown away.
- **Starvation:** A long transaction may keep losing to short ones forever and never commit.

**One-sentence summary:** OCC is great when conflicts are rare and operations are short — it gives you parallelism without locks. It's terrible for hotspots.

---

### Family 3: Transactional Memory — "Group operations, let the runtime handle it"

**Why it was invented:** Even with locks, programmers make mistakes — they forget to lock something, lock things in the wrong order (causing deadlocks), or compose two locked functions and accidentally hold both locks at once. Transactional memory lifts the burden up to the language/runtime.

**The idea:** You wrap a block of code in `atomically { ... }`. The runtime makes the entire block appear to happen all-at-once or not-at-all, just like a database transaction. Internally, it might use OCC, locks, or hardware support — you don't care.

**Mental picture:** Instead of you (the programmer) carrying around a ring of physical keys for every door, you just say "do this whole thing as one move" and the building handles the locks for you.

#### Example in Clojure (which has Software Transactional Memory built in):

```clojure
(dosync
  (alter account-a - 50)
  (alter account-b + 50))
```

Both updates either commit together or both abort together. No deadlocks possible, no manual lock ordering, no forgotten unlocks.

#### Hardware support:
- **Intel TSX (Transactional Synchronization Extensions)** — CPU instructions that try to run a region of code atomically using cache coherence. If a conflict is detected, the CPU rolls back and re-runs.
- **IBM z-series and Power chips** also have hardware transactional memory.

#### Why it isn't everywhere:
- **I/O can't be rolled back.** If your transaction sent an email or fired a missile, you can't un-send it on retry.
- **Performance overhead** of tracking reads/writes is real.
- Most production systems use database transactions (which give you the same guarantees at the storage layer) instead of in-memory STM.

**One-sentence summary:** Atomic blocks for memory operations — beautiful in theory, mostly used in research languages and database internals.

---

### A fourth approach worth knowing: Multi-Version Concurrency Control (MVCC)

**Why it was invented:** With pure locking, readers block writers and writers block readers — terrible for read-heavy workloads. MVCC says: "What if reads never blocked anything?"

**The idea:** Every write creates a *new version* of the row, tagged with a timestamp/transaction ID. Readers see the version that was current when they started — they never block, they just read an older snapshot.

**Mental picture:** Like a Git history of every row. Old versions stick around so readers can see the past.

#### Walking through it:
- Transaction T1 starts at time 100. It reads row X = "apple" (version 50).
- Transaction T2 starts at time 110, updates X to "banana", commits. Now version 110 of X = "banana" exists.
- T1 (still running) reads X again. It sees "apple" — its snapshot view from time 100. No blocking.
- T1 finishes at time 120. New transactions start seeing "banana".

#### Where you see it:
- **PostgreSQL** is fundamentally MVCC. That's why `VACUUM` exists — to clean up old row versions.
- **Oracle, MySQL InnoDB, SQL Server (with `READ COMMITTED SNAPSHOT`), CockroachDB, Spanner** — all MVCC.
- **Git itself** is conceptually MVCC for files.

**Benefit over pure locking:** Reads scale linearly without contention.

---

## B. Synchronization — coordinating *when* things happen

Even if your data is safe, you often need things to happen in a specific *order*. Synchronization is about that ordering and timing.

### The fundamental problem: ordering across independent workers

Imagine a wedding photo where 30 people need to be ready before the photographer clicks. If even one person is still tying their shoes, the photo is wasted. You need a way to *wait until everyone is ready*.

---

### Tool 1: Mutex (Mutual Exclusion) — already covered as locks

The simplest synchronization primitive. Why mention again? Because in the synchronization world, the focus is "who goes next" not "what data is protected."

---

### Tool 2: Semaphores

**Why needed:** A mutex says "1 at a time." But often you want "up to N at a time."

**Mental picture:** A parking garage with 50 spots. The gate has a counter. Each car arriving decrements the counter. Each car leaving increments it. When the counter hits zero, new cars wait at the gate.

#### Concrete examples:

- **Connection pool:** A web server is allowed 50 connections to the database. Each request "acquires" a permit before opening a connection, "releases" it when done. The 51st request waits.
- **Rate limiting downloads:** "Allow 5 concurrent video downloads." A semaphore initialized to 5 enforces this.
- **Producer/consumer with bounded buffer:** Two semaphores — one counting empty slots (producers wait when 0), one counting filled slots (consumers wait when 0).

#### Two types:
- **Counting semaphore:** Counter can be any non-negative integer (the parking garage).
- **Binary semaphore:** Counter is 0 or 1 — basically a mutex, but signal-based (any thread can release, not just the holder).

#### Real systems:
- Linux kernel's `sem_wait` / `sem_post`.
- Java's `java.util.concurrent.Semaphore`.
- Goroutine pool patterns in Go using buffered channels (a channel of size N is effectively a semaphore).

---

### Tool 3: Condition Variables

**Why needed:** Sometimes a thread needs to wait until *some condition becomes true*, not just for a lock to be free. Spinning in a loop checking the condition (a "busy wait") burns 100% CPU.

**Mental picture:** Imagine sitting in a doctor's waiting room. Instead of getting up every 5 seconds to check if it's your turn, you just sit and wait until the receptionist calls your name. That call is a "signal" on a condition variable.

#### Producer/consumer queue example:
- Consumer thread: acquires lock → checks "is queue empty?" → if yes, calls `condition.wait()` (which atomically releases the lock and sleeps).
- Producer thread: acquires lock → puts item in queue → calls `condition.signal()` to wake one waiting consumer → releases lock.
- Consumer wakes up, re-acquires the lock, processes the item.

The pattern is:

```text
lock(mutex)
while (!condition_holds) {
  cond.wait(mutex)   // sleep, release lock atomically
}
do_work()
unlock(mutex)
```

The `while` (not `if`) is because of "spurious wakeups" — sometimes threads wake up without being signaled, so you must re-check.

#### Where you see them:
- POSIX `pthread_cond_wait`, `pthread_cond_signal`.
- Java `Object.wait()` / `Object.notify()`, or `Condition` from `java.util.concurrent`.
- Python `threading.Condition`.

---

### Tool 4: Barriers

**Why needed:** In phased computation, the next phase can't start until *all* workers finish the current phase.

**Mental picture:** A relay race where all four runners must finish lap 1 before any can start lap 2. (Not a normal relay — but imagine it.)

#### Example: Distributed machine learning
- Each of 100 worker GPUs computes gradients for one mini-batch.
- Before updating model weights, *all* gradients must be averaged together.
- A barrier says: "Every worker calls `barrier.wait()` after computing. Only when all 100 have called it does anyone proceed."

#### Real systems:
- **MPI (Message Passing Interface):** `MPI_Barrier()` in HPC clusters.
- **Apache Spark:** Stage boundaries are barriers — the next stage starts only when all tasks in the current stage finish.
- **MapReduce:** Reducers wait until all mappers finish.
- **Java:** `CyclicBarrier`, `CountDownLatch`.

---

### Tool 5: Read-Write Locks

**Why needed:** A regular mutex blocks readers from each other unnecessarily. If 100 threads only want to read, they can all read simultaneously without conflict.

**Mental picture:** A library reading room. Many people can read silently together (read lock). But when someone needs to rearrange the shelves (write lock), everyone reading must leave first, and no new readers can enter until the rearranger is done.

#### Trade-off — writer starvation:
- If readers keep arriving, writers may wait forever. Real implementations use "writer preference" or fairness policies to avoid this.

#### Where you see it:
- POSIX `pthread_rwlock`.
- Java `ReentrantReadWriteLock`.
- Go's `sync.RWMutex`.

---

### Tool 6: Distributed Synchronization Primitives

The above tools were originally for single-machine threads. In a distributed system, you need network-aware versions:
- **Distributed locks** (via Redis Redlock, ZooKeeper ephemeral nodes, etcd leases).
- **Distributed barriers** (ZooKeeper recipe: each worker creates a node; coordinator watches; once N nodes exist, signal everyone).
- **Leader election** (covered in section C).

---

## C. Coordination Services — outsourcing the consensus problem

### Why needed (deep dive)

Imagine you want to implement leader election yourself: out of 5 servers, exactly one is the "primary" that handles writes; the others are backups.

Sounds simple. Try to write it. Now consider:
- The current leader's network connection drops for 30 seconds. The other 4 elect a new leader. The old leader's network comes back — it still thinks it's leader. You now have two leaders writing to the same data ("split brain"). Disaster.
- Server A votes for itself, Server B votes for itself, they each get 2 votes, tie. Now what?
- Network partition splits 3 servers from 2 servers. Each side elects its own leader. Two leaders again.

The mathematically rigorous solution to "get N machines to agree on something even when some fail or the network is unreliable" is called **consensus**, and the famous algorithms for it are:
- **Paxos** (Leslie Lamport, 1989) — original, very hard to understand.
- **Raft** (2014) — designed to be understandable, used in modern systems.
- **Zab** (ZooKeeper Atomic Broadcast) — used in ZooKeeper.

These algorithms are *extremely* hard to implement correctly. There are entire careers dedicated to verifying correctness with formal proofs. Bugs in them have caused real outages (e.g., MongoDB had several consistency bugs in early versions).

**So instead of implementing Raft yourself, you use a service that already has it.** That's what coordination services are.

---

### What coordination services give you

A typical coordination service (ZooKeeper, etcd, Consul) provides a small set of *primitives* that you build on top of:

#### 1. A consistent key-value store
- Write a key, all nodes see the same value (linearizable reads).
- Example: Store `/config/db_password = "xyz"`. Any service reads it and gets the latest committed value.

#### 2. Leader election
- Pattern: All candidates create a node like `/leaders/server-N`. The one with the lowest N is leader. If it dies, ZooKeeper auto-deletes its node (via "ephemeral nodes" tied to the session) and the next-lowest takes over.
- Used by: **Kafka** (controller election before KRaft), **HBase** (HMaster election), **HDFS NameNode HA**.

#### 3. Distributed locks
- Pattern: Try to create `/locks/myresource`. If the create succeeds, you have the lock. If not, watch for it to be deleted, then retry.
- Used by: Many distributed batch jobs ("only one job should run at a time across the cluster").

#### 4. Service discovery
- Each service instance registers itself in a directory like `/services/payment-service/instance-3`. Clients query the directory to find live instances.
- If an instance dies, the entry is auto-removed (ephemeral node).
- Used by: **Consul** is built around this. **Kubernetes** uses **etcd** for its entire cluster state.

#### 5. Configuration management
- Centralized config that updates propagate in real time. Apps "watch" a key and react to changes.
- Example: Feature flags. Flip a flag in etcd, all microservices see it within seconds.

#### 6. Group membership
- "Who is alive in the cluster right now?" — service maintains a list, with auto-cleanup on session loss.

---

### Real production examples

- **Kubernetes:** Every single piece of cluster state — pods, services, secrets, deployments — is stored in **etcd**. The API server is essentially a translator between Kubernetes objects and etcd keys. Lose etcd and you lose your cluster.
- **Kafka (pre-KRaft):** Used **ZooKeeper** to elect the controller broker, store topic metadata, and track which brokers are alive.
- **HashiCorp stack:** **Consul** for service discovery, **Vault** uses Consul or its own Raft for HA storage.
- **Google Spanner / Bigtable:** Use **Chubby** (Google's internal coordination service, the inspiration for ZooKeeper) for locks and metadata.

---

### The CAP theorem trap (worth understanding)

CAP says: during a network Partition, you must choose Consistency or Availability — you cannot have both.

These coordination services are typically **CP** (Consistent + Partition tolerant, sacrificing Availability). When a network partition splits the cluster, the minority side **stops accepting writes**. Why? Because if both sides kept writing, you'd have two truths.

This means: **if your coordination service is down, parts of your application stop.** Treat it as critical infrastructure. Run odd-numbered clusters (3, 5, 7) for quorum.

---

## D. Consistency Models — *when* writes become visible to reads

### Why this whole subject exists

On a single computer, after you write `x = 5`, the next read of `x` returns `5`. Period.

In a distributed system, your write went to *one* replica. There are 4 other replicas that don't have it yet. If a reader hits one of those, what should happen?
- Wait until all replicas are updated? (slow, possibly never if a replica is down)
- Return the old value? (fast but confusing)
- Return an error? (annoying)
- Pretend we updated and update the others later? (risk of losing the write)

A **consistency model** is a contract between the database and your application that defines exactly what readers will see. Different models trade off:
- **Latency** (how fast can we respond?)
- **Availability** (can we still serve requests during failures?)
- **Throughput** (how many ops/sec can we handle?)
- **Programmer sanity** (how confusing is the behavior?)

Let's walk from strongest (most predictable, least scalable) to weakest (least predictable, most scalable).

---

### Model 1: Linearizability (Strong Consistency)

**The promise:** Every operation appears to take effect at a single instant between when you sent it and when you got the response. After your write completes, *everyone everywhere* sees the new value on their very next read. As if there's only one copy.

**Mental picture:** A single bulletin board everyone in the world can see. The instant you pin a note, everyone — Tokyo, New York, Sydney — sees it on their next glance.

#### Detailed scenario:
- Time 10:00:00.000 — Alice writes `balance = $100` from New York. Write returns at 10:00:00.050.
- Time 10:00:00.051 — Bob reads `balance` from Tokyo. He **must** see `$100` (or newer). Not `$0` from yesterday.

#### How it's implemented:
- Every write goes through a leader (or quorum) that orders it.
- Reads either go to the leader, or use quorum reads (read from a majority and pick the latest).
- This requires network round-trips to multiple nodes for *every* operation.

#### Real systems:
- **etcd, ZooKeeper** for metadata.
- **Google Spanner** — globally linearizable across continents using GPS-synchronized atomic clocks (TrueTime).
- **CockroachDB, FoundationDB.**
- A single-node PostgreSQL is trivially linearizable (only one copy).

#### What you give up:
- **Latency:** Every operation pays for cross-node coordination. Cross-region writes can be 100ms+.
- **Availability:** During a network partition, the side without quorum **stops accepting writes**.

#### When to use it:
- Bank ledgers, inventory ("only one PS5 left"), distributed locks, leader election, configuration. Anything where being wrong is worse than being slow.

---

### Model 2: Sequential Consistency

**The promise:** All nodes see operations in the same order, but that order does not have to match real-world time.

**Mental picture:** Imagine everyone watches the same TV show, but on a delay. Episode 1 always plays before Episode 2 for every viewer, but I might be 10 minutes ahead of you.

#### Concrete scenario:
- Alice writes `x = 5` at 10:00.
- Bob writes `y = 7` at 10:01.
- Carol (a reader) sees `y = 7` first, then `x = 5`.
- Dave (a reader) **also** sees `y = 7` first, then `x = 5`.
- The order doesn't match real time, but everyone agrees on the order.

#### Why weaker than linearizability:
- Linearizability: order matches real time.
- Sequential: order is consistent across observers but doesn't have to match real time.
- This means you don't need synchronized clocks — much cheaper to implement across geographies.

#### Where you see it:
- Some distributed log systems.
- Within a single multi-core CPU's memory model, x86 is roughly sequentially consistent (within bounds).

---

### Model 3: Causal Consistency

**The promise:** If operation A *caused* operation B (e.g., B happened after reading A's result), then everyone sees A before B. Operations that are *unrelated* can be seen in any order.

**Mental picture:** Imagine a group chat. "Did you eat lunch?" → "Yes, pizza." Everyone must see the question before the answer (because the answer is a reply to the question). But if someone unrelated says "happy birthday Sara" at the same time, that message can appear in any order relative to the lunch chat.

#### Concrete failure scenario *without* causal consistency:
- You post on Facebook: "I quit my job!"
- Then you comment on your own post: "It was about time."
- Friend Alex sees: "It was about time" — with no parent post visible. Confusing.

#### How it's enforced:
- Every operation carries metadata (a "vector clock" or "version vector") describing what it depends on. When propagating, replicas wait for dependencies before applying.

#### Real systems:
- **MongoDB causal sessions.**
- **COPS, Bayou** (research systems).
- **Riak** has options for causal consistency via vector clocks.

#### Why weaker than sequential:
- Doesn't require a global order — only that *related* operations agree. Way cheaper, especially across regions.

---

### Model 4: Read-Your-Writes Consistency

**The promise:** A user always sees their own writes on subsequent reads. Other users may or may not see them yet.

**Mental picture:** You write a note on a sticky and put it on your desk. You can always see your sticky. Your coworkers might not see it for a while.

#### Scenario *without* it (terrible UX):
- You change your profile picture.
- Page refreshes — you see the OLD picture, because the read went to a replica that hasn't received the update yet.
- You think the update failed and try again.
- Now there are two pending updates.

#### How it's implemented:
- After writing, route that user's reads to either the primary or to the same replica that received the write.
- Often done via **session stickiness** — a user's session is pinned to one replica.
- Or by tracking a "last write timestamp" per user and ensuring reads come from a replica caught up to that timestamp.

#### Real systems:
- DynamoDB has a "ConsistentRead" flag.
- MongoDB read concerns can give this guarantee.
- Most well-designed user-facing apps implement this on top of an eventually-consistent store.

---

### Model 5: Session Consistency

**The promise:** Like read-your-writes, but with the additional guarantees of monotonic reads (next slide), monotonic writes, and write-follows-read — all within the scope of a single session.

**Mental picture:** Within one phone call, the conversation makes sense. Across two separate phone calls, no guarantees.

#### Concrete example:
- E-commerce shopping cart. Within your session:
  - You add iPhone to cart → next page shows iPhone in cart (read-your-writes).
  - You then add headphones → next page shows iPhone *then* headphones (monotonic, plus order preserved).
- If your session ends and you log in from another device, the cart might briefly show an older state — but within one session, it's consistent.

---

### Model 6: Monotonic Read Consistency

**The promise:** Once you've seen a value (or a newer one), you never see an older one.

**Mental picture:** Time only moves forward. The flight status app may say "On Time," then "Delayed 30 min," then "Boarding." It never goes back to "On Time" once you've seen "Delayed."

#### Failure scenario *without* it:
- You refresh airline app: "Flight delayed 1 hour."
- You refresh again: "On time" (you accidentally hit a stale replica).
- You refresh again: "Delayed."
- You go insane.

#### How it's implemented:
- Track the timestamp/version of the latest read per session.
- Subsequent reads only return data at least as new as that.
- Typically done via session stickiness.

---

### Model 7: Monotonic Write Consistency

**The promise:** Writes from a single client/session are applied in the order issued.

**Mental picture:** You give instructions to a tailor: "Cut the fabric, then sew the buttons." The tailor doesn't sew buttons before cutting.

#### Failure scenario *without* it:
- Bank app: "Deposit $100" → "Withdraw $50."
- The withdraw arrives at a replica before the deposit.
- Withdraw is rejected for insufficient funds. Deposit then arrives. Total mess.

---

### Model 8: Eventual Consistency

**The promise:** If no new writes happen, eventually all replicas converge to the same value. No timing guarantee.

**Mental picture:** Gossip in a school. If a juicy rumor starts, eventually everyone has heard it. But for a while, some people have it, some don't, some have the wrong version. Eventually it settles down.

#### Real failure modes you must design around:
- **Stale reads:** Tweet count says 42 likes, then 39, then 47. Numbers fluctuate.
- **Conflict resolution:** Two replicas both update the same key — when they sync, who wins? Common strategies:
  - **Last-write-wins (LWW):** Newest timestamp wins. Simple but can lose data if clocks are skewed.
  - **CRDTs (Conflict-Free Replicated Data Types):** Data structures designed to merge automatically without conflict. E.g., a counter where each replica's increments are tracked separately and summed.
  - **Application-level resolution:** Store all conflicting versions, let the user/app decide. Riak does this.

#### Real systems:
- **Amazon DynamoDB** (default reads).
- **Apache Cassandra.**
- **Amazon S3** (was eventually consistent for years, became strong in 2020 — a famous shift in industry direction).
- **DNS** — a domain change takes hours to fully propagate.
- **Git** — your local copy can be ahead or behind origin; eventually you sync.

#### When to use:
- Like counts, view counts, recommendation feeds, recently-viewed lists. Data where being slightly stale is acceptable.

---

### A note on PACELC (an extension of CAP)

CAP says: during a Partition, choose Consistency or Availability. PACELC adds: Else (no partition), choose Latency or Consistency.
- **PA/EL systems:** Cassandra, DynamoDB — pick availability when partitioned, low latency when not.
- **PC/EC systems:** etcd, Spanner — pick consistency always, even at cost of latency.

---

## How these pieces fit together — a real system walkthrough

Let's design **a flash sale: 1000 PS5s for sale at noon, expecting 1 million visitors**.

1. **Catalog (product info, images, descriptions):** Eventual consistency. Use Cassandra or read from a CDN. Stale reads of "specs" don't hurt anyone.

2. **Inventory counter:** Linearizable. Must use a strongly consistent store (etcd, a SQL database with row locks, or DynamoDB conditional writes). Two buyers must not both succeed for the last unit.

3. **Cart (per user):** Read-your-writes via session stickiness. Each user must see their own cart instantly.

4. **Checkout transaction:** Pessimistic lock on the inventory row OR optimistic with retry. Likely pessimistic — high contention, OCC would thrash.

5. **Leader election for the inventory service** (so only one service writes inventory updates): ZooKeeper or etcd.

6. **Rate limiting** ("max 10 checkout attempts per second per user"): Semaphore implemented in Redis.

7. **Order processing pipeline** (validate → charge → ship label → email):
   - Each stage waits for the previous (sequential, but uses message queues).
   - A barrier-like wait for "all 1000 orders confirmed" before generating the morning report.

8. **Recommendations** ("people who bought this also bought..."): Eventually consistent. Recompute hourly.

You can see how every layer picks the *cheapest* consistency model that's still correct. That's the heart of distributed systems engineering.

---

## TL;DR Cheat-Sheet

- **Concurrency control = data safety.** Pessimistic locks (safe, slow), Optimistic CC (fast, retries), STM (rare), MVCC (read-heavy workloads).
- **Synchronization = timing.** Mutex, semaphores (N-at-a-time), condition variables (wait-until), barriers (everyone-meet-here), read-write locks.
- **Coordination services = "don't roll your own consensus."** ZooKeeper, etcd, Consul. Use them for leader election, distributed locks, service discovery, config.
- **Consistency models = the contract about when writes become visible.** Linearizable (perfect, slow) → Sequential → Causal → Session (read-your-writes + monotonic) → Eventual (fast, weird).
- **The whole game is choosing the weakest model that's still correct for each piece of your system.**
