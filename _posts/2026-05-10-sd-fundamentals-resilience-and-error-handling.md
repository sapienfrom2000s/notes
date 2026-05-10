---
title: "System Design Fundamentals - Resilience & Error Handling"
date: 2026-05-10 22:30:00 +0530
categories: [System Design]
tags: [resilience, fault-tolerance, circuit-breaker, retries, chaos-engineering, observability]
---

## Why this whole topic exists

In a single-machine program, "failure" usually means a crash. You see a stack trace, you fix it, you restart. Easy.

In a distributed system, failure is the **default state** of the world:
- Network packets get lost or delayed.
- Disks die — usually at 3 AM on a Sunday.
- A single bad deployment in one service freezes 50 other services that depend on it.
- A traffic spike from one viral tweet melts your database.
- The data center cooling fails, and an entire region goes offline (this has happened to AWS, Azure, and GCP — multiple times).

Famous quote from Werner Vogels, Amazon's CTO: *"Everything fails, all the time."*

So the question is no longer *"How do I prevent failure?"* (impossible). It's *"How do I keep my system useful even when parts of it are on fire?"*

That's resilience. The five core ideas:
1. **Fault Tolerance** — survive when components die.
2. **Graceful Degradation** — give users a worse-but-working experience instead of total failure.
3. **Cascading Failure Containment** — stop one failure from taking down everything else (retries, timeouts, circuit breakers, bulkheads, backpressure).
4. **Error Handling & Reporting** — see what's broken, fast.
5. **Chaos Engineering** — break things on purpose, in production, so you trust your safety nets.

---

## A. Fault Tolerance — surviving the death of components

### The fundamental problem

Take a simple e-commerce site running on one server. The server has:
- One CPU (could overheat).
- One disk (could fail — average lifespan ~5 years).
- One network card (could die).
- One power supply (data center could lose power).
- One copy of the database (corruption is permanent).

If *any one* of these dies, your entire business stops until you fix it. This is called a **Single Point of Failure (SPOF)** — any component whose failure brings down the whole system.

**Fault tolerance is the practice of eliminating SPOFs through redundancy** — having backups so the system survives any single failure.

---

### Technique 1: Redundancy — "have a spare"

**Mental picture:** A passenger jet has 2-4 engines, not 1. If one engine fails, the others keep the plane flying. Engineers don't try to make engines that never fail — they assume they will and plan for it.

#### Two flavors of redundancy:

- **Active-Passive (a.k.a. hot standby):** One server actively serves traffic; a backup sits idle, ready to take over if the active one dies.
  - *Example:* AWS RDS Multi-AZ — your database has a standby replica in another availability zone. If the primary dies, RDS promotes the standby in ~60 seconds.
  - *Pros:* Simple. The backup is guaranteed clean.
  - *Cons:* You pay for hardware that does nothing 99.99% of the time. Failover takes time (seconds to minutes), so you have a brief outage.

- **Active-Active (a.k.a. load-shared):** All servers serve traffic simultaneously. If one dies, the rest absorb its load.
  - *Example:* A web tier with 10 servers behind a load balancer. If 1 dies, the remaining 9 each handle ~11% more traffic. No outage at all.
  - *Pros:* No wasted hardware. Zero failover time.
  - *Cons:* Harder to coordinate (especially for stateful services). Need capacity headroom — if all 10 are at 95% load and 1 dies, the other 9 immediately overload.

---

### Technique 2: Replication — "have a copy of the data"

**Why needed:** Redundant servers are useless if they all read from the same database. The database is now the SPOF. So you replicate the data too.

**Mental picture:** Important documents are kept in multiple safes in multiple cities. If one city floods, the others still have the document.

#### Synchronous vs Asynchronous replication (a critical trade-off):

- **Synchronous replication:** A write isn't acknowledged until *all* (or a quorum of) replicas have it.
  - *Example:* PostgreSQL synchronous standby. AWS RDS Multi-AZ.
  - *Pro:* Zero data loss on failover.
  - *Con:* Every write waits for the slowest replica. Cross-region writes can be 100ms+.

- **Asynchronous replication:** Primary acknowledges the write immediately, then ships it to replicas in the background.
  - *Example:* MySQL default replication. MongoDB default. Most read replicas.
  - *Pro:* Fast writes.
  - *Con:* If the primary dies before the replica caught up, you **lose** the most recent writes (typically a few seconds' worth).

#### Where you actually see this:
- **Cassandra:** Replication factor of 3 = every piece of data lives on 3 nodes. Configurable consistency level (write to 1, quorum, or all).
- **HDFS:** Default replication factor 3 — same data on 3 different machines, ideally on different racks.
- **AWS S3:** Stores every object on multiple disks across multiple data centers automatically.
- **RAID 1 (mirroring):** Same idea on a single machine — every disk has a twin.

---

### Technique 3: Sharding for fault isolation

**Why needed:** If all your users live on one giant database, *any* failure affects everyone. Shard the data so a single shard failure only affects a slice of users.

**Mental picture:** Apartment buildings have firewalls between units so a fire in apartment 4B doesn't burn down the whole building.

#### Real example:
- A multi-tenant SaaS app with 1000 customers.
- Without sharding: 1 database. A bad query brings the whole site down.
- With sharding: 10 databases, 100 customers each. A bad query brings down only 100 customers' data.
- "Cell-based architecture" (popularized by AWS) takes this further — entire copies of the stack per "cell," with users routed to a specific cell.

---

### Technique 4: Failover — the actual switch-over

**Why needed:** Just having a backup isn't enough — you need a mechanism to detect failure and route traffic to the backup. This sounds easy, isn't.

#### The three hard problems of failover:
1. **Detection:** How do you know the primary is *really* dead and not just temporarily slow? Health checks help, but a brief network blip can look like death.
2. **Promotion:** Who decides the backup becomes the new primary? (Coordination service like ZooKeeper/etcd usually owns this — see Concurrency & Coordination notes.)
3. **Split brain:** What if the old primary wakes up and *also* thinks it's primary? Now you have two primaries, both writing. Data corruption.
   - *Fix:* Fencing tokens, STONITH ("Shoot The Other Node In The Head" — literally), or quorum-based decisions where the old primary realizes it's no longer in the majority.

#### Real-world horror:
- **GitLab database outage, 2017:** A series of failures left them with a corrupted primary and stale backups. They lost 6 hours of production data because the failover/backup system was untested.
- **The lesson:** Untested failover ≈ no failover.

**One-sentence summary:** Fault tolerance = redundancy at every layer (compute, data, network, power, region) + a tested switchover process.

---

## A.1 Companion topic: Fault Tolerance vs High Availability

These two terms get used interchangeably in conversation and even in vendor docs. They are **not** the same thing. Mixing them up will cost you marks in interviews and money in real architecture decisions. Let's pull them apart carefully.

### Why this confusion exists

Both concepts attack the same enemy (failure) and use overlapping techniques (redundancy, failover, replication). So people lump them together. But they have **different definitions of "success"** — and that difference drives radically different cost and complexity.

**Mental picture:**
- **Fault Tolerance** = a fighter jet that keeps flying after one engine is hit by a missile. The pilot doesn't even feel a bump. Zero interruption is the requirement.
- **High Availability** = a passenger airline that runs flights 99.99% of the time. Occasionally a flight is delayed by 30 minutes due to a mechanical issue, but the airline as a whole is "available" almost always.

Both are reliable. But "reliable" means very different things.

---

### Fault Tolerance (FT) — defined precisely

**The promise:** When a component fails, the system continues to operate **without any visible impact** to users. No dropped requests. No latency spike. No data loss. As if the failure never happened.

#### How it's achieved (the demanding way):
- **N+1 or 2N redundancy at every layer.** If you need 1 server's worth of capacity, you run 2 (or more) in lockstep.
- **Hot standbys running in parallel** — not just sitting idle, but actively processing the same input so they're always perfectly current.
- **Synchronous replication** — every write is committed on all replicas before the client gets a response. Zero data loss on failure.
- **Failover in milliseconds**, often using hardware-level mechanisms.
- **Often involves specialized hardware** — lockstep CPUs, ECC memory, redundant power supplies, dual NICs.

#### Real fault-tolerant systems:
- **Stratus ftServer / HP NonStop:** Two physical servers run the *same* CPU instructions in lockstep. If one fails, the other is already at the exact same state — zero failover time, zero data loss. Used in stock exchanges, ATM networks, telco switches.
- **Aircraft flight control systems:** Triple-redundant computers (e.g., Boeing 777 has 3 independent flight control computers running different software written by different teams). They vote on every output. One failing means the other two outvote it.
- **Spacecraft:** Same idea — quadruple redundancy on the Space Shuttle.
- **NASDAQ's matching engine:** Continuously available — measured in microseconds.

#### What FT costs you:
- **Money.** Lockstep hardware is 3-10x the cost of equivalent commodity hardware.
- **Complexity.** Synchronous replication adds latency and consensus overhead.
- **Throughput.** Synchronous coordination is the enemy of speed.

#### When you actually need FT:
- Stock exchange order matching (microseconds of downtime = lawsuits).
- Pacemakers and life-support systems.
- Aircraft control surfaces.
- Nuclear reactor controls.
- Some financial settlement systems (SWIFT, central bank rails).
- Anywhere the cost of *one second* of downtime exceeds the cost of full redundancy.

---

### High Availability (HA) — defined precisely

**The promise:** The system is operational and accessible **for a very high percentage of time** — typically measured in "nines." A brief outage during failover is acceptable, as long as the system recovers quickly.

#### What "nines" actually mean (memorize this for interviews):
- **99% ("two nines")** = ~3.65 days of downtime per year. Pretty bad.
- **99.9% ("three nines")** = ~8.77 hours of downtime per year. Fine for many SaaS apps.
- **99.95%** = ~4.4 hours/year. Standard for paid B2B services.
- **99.99% ("four nines")** = ~52.6 minutes/year. Strong target for serious infrastructure.
- **99.999% ("five nines")** = ~5.26 minutes/year. Telco-grade. Hard.
- **99.9999% ("six nines")** = ~31.5 seconds/year. Approaches FT territory.

#### How it's achieved (the pragmatic way):
- **N+1 redundancy** (often just one spare).
- **Active-active or active-passive** clustering with load balancers detecting failures.
- **Asynchronous replication** is acceptable — small data loss windows tolerated.
- **Failover in seconds to minutes** is acceptable.
- **Commodity hardware** — replication and failover handled in software.
- **Geographic distribution** across availability zones / regions.

#### Real HA systems:
- **AWS RDS Multi-AZ:** Standby in another AZ; failover takes 60-120 seconds.
- **Kubernetes:** Pods reschedule if a node dies (10s to a few minutes downtime per pod).
- **Cassandra clusters:** Lose a node, others handle the load instantly. Maybe one stale read.
- **A web tier behind a load balancer:** One server dies; load balancer health check notices in 10-30s and stops sending traffic to it.
- **Google, Meta, Netflix infrastructure:** All HA, not FT. They tolerate brief partial outages and focus on quick recovery.

#### What HA costs you:
- **Some money** — but a fraction of FT (often just 2x base cost, not 10x).
- **Some downtime** — measured but bounded.
- **Possible tiny data loss** during failover (typically the last few seconds of writes).

#### When HA is enough:
- E-commerce sites.
- SaaS platforms.
- Streaming services.
- Most enterprise applications.
- Almost everything you'll build.

---

### Side-by-side: the dimensions that matter

- **Goal**
  - FT: zero impact during failure.
  - HA: minimal downtime; recovery is acceptable.

- **Failover time**
  - FT: instantaneous (microseconds, or zero).
  - HA: seconds to minutes — measured and bounded.

- **Data loss tolerance**
  - FT: zero. Synchronous replication mandatory.
  - HA: a few seconds of writes can be lost (RPO > 0).

- **Replication mode**
  - FT: synchronous, often lockstep at the instruction level.
  - HA: usually asynchronous; sometimes synchronous within a region.

- **Hardware**
  - FT: often specialized (Stratus, NonStop, redundant CPUs).
  - HA: commodity hardware + clever software.

- **Cost**
  - FT: 3-10x baseline.
  - HA: ~2x baseline.

- **Complexity**
  - FT: high — lockstep, voting, specialized stacks.
  - HA: moderate — load balancers, health checks, replicas.

- **Common SLA**
  - FT: 99.999%+ (five nines or better, often "continuous").
  - HA: 99.9% to 99.99% (three to four nines).

- **Examples**
  - FT: stock exchange matching engine, pacemaker, flight control, NASA spacecraft.
  - HA: AWS, Netflix, Google services, your SaaS app.

- **What user sees during failure**
  - FT: nothing.
  - HA: a brief loading spinner, a "please refresh," or one failed request.

---

### The single most important takeaway

Most engineers (and most interviewers) really mean **High Availability** when they say "fault tolerant." True FT is rare and expensive — almost no consumer-facing system needs it.

When you hear "we need a fault-tolerant system" in a design interview, **ask the clarifying question:** *"By fault tolerant, do you mean truly zero downtime / zero data loss like a stock exchange, or do you mean highly available — recovers in under a minute with maybe a second of lost writes? They have very different cost profiles."* This shows senior-level judgment.

---

### Why you usually choose HA, even for "critical" systems

Consider Amazon.com. It processes ~$700K of orders per minute on average. So 1 minute of downtime = $700K lost.

Sounds like a case for fault tolerance, right? Wrong. Building Amazon.com fault-tolerant (Stratus-style lockstep, synchronous global replication) would cost *billions* of extra dollars per year and slow every transaction. Building it as HA at 99.99% costs orders of magnitude less and "only" loses ~$37M per year of revenue to downtime — a clear win.

The decision is always: **does the cost of an extra nine of availability exceed the cost of the downtime it prevents?** For most systems, somewhere around four nines, the answer becomes "yes, stop here."

---

**One-sentence summary:** Fault Tolerance = "no failure is ever visible, at any cost." High Availability = "very rarely down, recover fast when we are." Pick HA unless lives or markets depend on it.

---

## B. Graceful Degradation — "give them less rather than nothing"

### The fundamental problem

Imagine your e-commerce site at checkout. The flow needs:
- Inventory check
- Pricing engine
- Personalized recommendations ("you might also like...")
- Tax calculation
- Payment processing
- Order confirmation email

What happens if the recommendations service is down?

- **Naive design:** Recommendations call throws an error → checkout endpoint returns 500 → user can't buy. Recommendations failure caused a 100% revenue loss.
- **Graceful design:** Recommendations call times out after 200ms → checkout shows a static "Top Sellers" list (or just hides the section) → user completes purchase normally. Recommendations failure caused a slightly worse UX. Revenue loss: $0.

**Graceful degradation = identifying which features are essential vs nice-to-have, and making the nice-to-have features fail safely.**

**Mental picture:** A car has two headlights. If one burns out, you don't crash — you keep driving with reduced visibility. The car *degrades gracefully*. (Versus a car where the entire electrical system shuts down if any bulb fails — that's catastrophic failure.)

---

### Common patterns:

#### 1. Fallback responses
- **Why:** Better to return *something* than an error.
- *Example:* Netflix homepage. If the personalized recommendation service is down, Netflix serves a generic "Popular on Netflix" list cached at the edge. The user might never notice.

#### 2. Cached stale responses
- **Why:** Yesterday's data is better than no data.
- *Example:* Twitter has historically served slightly-stale feeds during incidents. You see tweets from 5 minutes ago instead of an error page.
- Implementation: HTTP `Cache-Control: stale-while-revalidate` lets the browser/CDN serve a stale response while fetching a fresh one in the background.

#### 3. Feature flags
- **Why:** Turn off expensive non-essential features during incidents.
- *Example:* Black Friday traffic spike. Operations team flips a flag that disables product reviews, related-products carousel, and image zoom. Site stays up; users can still buy.
- Tools: LaunchDarkly, Unleash, Statsig.

#### 4. Read-only mode
- **Why:** When writes fail, at least let users browse.
- *Example:* GitHub regularly degrades to read-only during database failovers. You can browse code; you just can't push or comment for a few minutes.

#### 5. Reduced quality
- **Why:** Lower-quality data is better than no data.
- *Example:* YouTube drops video quality from 4K → 1080p → 480p → audio-only as your bandwidth degrades. The video keeps playing. (Adaptive Bitrate Streaming.)

---

### What graceful degradation requires:

- **Classifying every dependency:** "If this service is down, what should we do?" — return cached, return default, hide feature, or fail hard. Document this for every external call.
- **Timeouts on every external call** (see next section). Without timeouts, a "graceful fallback" never triggers because you're stuck waiting.
- **Default responses pre-baked** in your code: empty lists, sensible defaults, friendly error messages.

**One-sentence summary:** Decide ahead of time which features are critical and which can fail; design the non-critical ones to fail invisibly.

---

## C. Cascading Failure Containment — stopping the dominos

### The fundamental problem: cascading failures

This is the single most important resilience topic, and the one most often missed.

**Mental picture:** A line of dominos. The first one falls. It knocks down the second, which knocks down the third, etc. In distributed systems, one slow service can knock down its caller, which knocks down *its* caller, until the whole platform is down.

#### Concrete cascade scenario:
- Your "Profile Service" depends on "Auth Service."
- Auth Service starts responding slowly (1s instead of 10ms) — maybe its database is overloaded.
- Profile Service waits 1s for every Auth call. Its threads pile up waiting.
- Profile Service runs out of threads. Now Profile Service is unresponsive.
- "Web Frontend" depends on Profile Service. It hangs waiting on Profile.
- Frontend runs out of threads. Site is down.
- A small Auth slowness becomes a complete outage in 30 seconds.

**Without containment patterns, the entire system has the reliability of its weakest link, multiplied by the dependencies.**

The defensive patterns below all attack different parts of the cascade.

---

### Pattern 1: Timeouts — "I refuse to wait forever"

**Why needed:** The default in most HTTP/RPC libraries is **no timeout** or a giant default like 60 seconds. So if a downstream service hangs, your service hangs with it.

**Mental picture:** Calling a friend. If they don't pick up in 30 rings, you hang up — you don't sit there for 6 hours.

#### Without timeouts:
- 1000 requests/sec come in.
- Downstream becomes slow (10s per request).
- Your server has 100 worker threads.
- After 100 ms, all 100 threads are stuck waiting on downstream.
- Request 101 has nothing to handle it. Queue fills up. New requests get rejected or hang.
- Your service is now "down" even though nothing crashed — it's just stuck.

#### Rules of thumb:
- **Set a timeout on EVERY remote call.** No exceptions.
- Timeouts should be **short** — typically 1-5 seconds for user-facing APIs, milliseconds for hot paths.
- Set the **client timeout shorter than the server's processing time budget**, otherwise the server keeps doing work the client gave up on.

#### Real-world examples:
- AWS SDKs default to 50-second timeouts — almost always too long. Override them.
- gRPC requires explicit deadlines (`Context.WithTimeout` in Go). It's idiomatic.
- HTTP clients in Java (`HttpClient`) and Python (`requests`) need explicit `timeout=` arguments.

---

### Pattern 2: Retries with Exponential Backoff and Jitter

**Why needed:** Transient failures are common (a brief network blip, a temporarily overloaded service). Retrying often succeeds.

**The naive approach (do not do this):** Just retry immediately on failure.
- Problem: If the downstream is overloaded, immediate retries make the load *worse*. The service stays down longer.

**Mental picture:** Imagine a busy restaurant phone. If you call, get a busy signal, and immediately redial — and 1000 other people do the same — the line never clears. If everyone waits a random amount of time and tries again, the calls spread out and most get through.

#### The fix: exponential backoff
- 1st retry: wait 100ms
- 2nd retry: wait 200ms
- 3rd retry: wait 400ms
- 4th retry: wait 800ms
- ...with a max retry count (e.g., 5).

#### Plus jitter (random delay)
- Without jitter, every client that hit the same failure retries at the same instant — creating "thundering herd" spikes.
- With jitter: actual delay = `random(0, 2^attempt * base_delay)`. This spreads retries out.

#### Critical rule: only retry idempotent operations
- **Idempotent** = doing it twice has the same effect as doing it once. (e.g., "set balance to $100.")
- **Non-idempotent** = "deduct $10 from balance" — retrying double-deducts.
- For non-idempotent operations, use **idempotency keys** (a unique ID with the request; the server detects duplicates and returns the prior result).

#### Real systems:
- **AWS SDK:** Built-in retry with exponential backoff and jitter.
- **Stripe API:** Requires `Idempotency-Key` header for retries on payments.
- **Kafka producer:** Has `retries`, `retry.backoff.ms`, idempotent producer mode.

---

### Pattern 3: Circuit Breaker — "stop trying when it's clearly broken"

**Why needed:** Even with timeouts and retries, hammering a dead service is wasteful — every call costs the client a timeout's worth of time, and adds load to the downstream when it's struggling. Better to **stop calling for a while** and give it space to recover.

**Mental picture:** The electrical breaker in your house. If a circuit is overloaded, the breaker trips and cuts power instead of letting the wires catch fire. After things calm down, you flip it back on.

#### How it works (three states):

- **CLOSED:** Normal operation. All calls go through. Failures are counted.
- **OPEN:** Too many failures. The breaker "trips." For a cooldown period (e.g., 30 seconds), all calls **fail immediately** without even trying. The downstream gets a break.
- **HALF-OPEN:** After cooldown, allow a few test calls through. If they succeed, breaker closes (back to normal). If they fail, breaker re-opens.

#### Walkthrough:
- Profile Service calls Auth Service. Auth is healthy. Breaker is CLOSED.
- Auth's database falls over. Calls start failing.
- After (say) 50% failures in a 1-minute window, the breaker trips OPEN.
- For the next 30s, Profile Service immediately returns a fallback ("Anonymous user") instead of calling Auth. Auth has zero load from Profile.
- After 30s, breaker goes HALF-OPEN. Lets 1 call through. It works → breaker closes. Normal operation resumes.

#### Why this is so important:
- **Saves the dying service.** A struggling service often recovers if you stop hammering it.
- **Frees the caller's threads.** Without a breaker, your threads sit waiting on timeouts. With it, they fail fast and serve other requests.
- **Provides a fallback hook.** Each "fast fail" can return cached data or a default.

#### Real systems:
- **Netflix Hystrix** (the original; archived but conceptually foundational).
- **Resilience4j** (Java).
- **Polly** (.NET).
- **Istio service mesh** has built-in circuit breaking via Envoy.
- **AWS App Mesh.**

---

### Pattern 4: Bulkheads — "isolate resources so one failure can't drain everything"

**Why needed:** A single thread pool / connection pool shared across all calls becomes a single failure domain. One slow downstream consumes all threads, starving everyone else.

**Mental picture:** A ship's hull is divided into watertight compartments (bulkheads). If one compartment floods, the ship still floats. The Titanic famously had bulkheads that didn't go high enough — water spilled over the top.

#### Without bulkheads:
- Your service has one connection pool of 100 connections.
- Downstream A, B, C all use it.
- A becomes slow → uses all 100 → B and C requests fail with "no connections available."
- A localized failure became a global one.

#### With bulkheads:
- Pool of 50 for A, 30 for B, 20 for C.
- A becomes slow → uses its 50 → B and C still have their pools, keep working.

#### Implementation flavors:
- **Separate thread pools per dependency.**
- **Separate connection pools per downstream service.**
- **Separate process / container per workload (the strongest form).**
- **Hystrix and Resilience4j** support thread-pool bulkheads natively.

---

### Pattern 5: Rate Limiting / Throttling — "I refuse to accept too much"

**Why needed:** A traffic spike (legitimate or attack) can take down a service if it accepts every request. Better to **reject excess load** than collapse under it.

**Mental picture:** A nightclub bouncer. Once it's at fire-code capacity, no more entries — even if there's a line. Better to turn 100 people away than have a fatal stampede inside.

#### Common algorithms:

- **Token Bucket:** Bucket holds N tokens. Each request consumes one. Tokens refill at a fixed rate. If empty, reject. Allows brief bursts.
- **Leaky Bucket:** Requests enter a queue (bucket) that drains at a fixed rate. If the bucket overflows, reject. Smooths spiky traffic.
- **Fixed Window:** Count requests per minute. If > 1000, reject. Simple but allows 2x bursts at window boundaries.
- **Sliding Window:** More accurate version of fixed window.

#### Where applied:
- **Per-user / per-API-key:** "Free tier: 100 requests/min." (Stripe, GitHub API.)
- **Global:** "This service can handle max 10K req/s, anything above gets 429."
- **Per-IP:** Block abusive scrapers and basic DDoS.

#### Real systems:
- **NGINX `limit_req`.**
- **Envoy / Istio rate limiting.**
- **AWS API Gateway throttling.**
- **Cloudflare** at the edge.

---

### Pattern 6: Backpressure — "tell upstream to slow down"

**Why needed:** Throttling rejects requests after the fact. Backpressure prevents them from being sent in the first place by signaling "I'm full" up the pipe.

**Mental picture:** A grocery checkout. When the bagger can't keep up, the cashier stops scanning. The customer waits. Items don't pile up on the floor.

#### Concrete example: Streaming
- A Kafka consumer is reading 10K msgs/sec but can only process 5K/sec.
- Without backpressure: messages accumulate in memory → out-of-memory crash.
- With backpressure: consumer stops asking for more messages from Kafka until it's caught up. Kafka holds them on disk. No crash.

#### Implementations:
- **Reactive Streams (Java, RxJS, Akka Streams):** the spec is literally about backpressure. Subscribers signal demand to publishers.
- **TCP itself:** has built-in backpressure via the receive window.
- **gRPC streaming:** flow control built in.
- **Kafka:** consumer-driven pull model is naturally backpressured.

---

### Pattern 7: Load Shedding — "drop low-priority work first"

**Why needed:** Sometimes throttling isn't enough. When overloaded, drop the *least important* requests so the most important ones still succeed.

#### Example: An e-commerce site under load
- Priority 1 (must succeed): Checkout, payment.
- Priority 2: Browsing, search.
- Priority 3: Recommendations, analytics writes.
- When the system is at 90%+ capacity, start dropping P3 → then P2 → only drop P1 if absolutely necessary.

This is how Google Search and Amazon stay up under extreme load — non-essential features quietly disappear.

---

**One-sentence summary of section C:** Timeouts stop you from waiting forever. Retries fix transient errors. Circuit breakers protect dying services. Bulkheads isolate failures. Rate limiting protects you from overload. Backpressure pushes overload upstream. Load shedding sacrifices low-priority work. Combine them all.

---

## D. Error Handling and Reporting — seeing what's broken

### The fundamental problem: silent failures

A bug that crashes loudly is a *good* bug — you see it, you fix it. The dangerous bugs are silent: data quietly corrupted, requests quietly dropped, alerts quietly missed.

#### The horror scenario:
- A nightly job that emails customer reports has been failing for 3 weeks.
- Nobody noticed because errors went to `/dev/null`.
- Customers are silently angry. Some leave.
- You find out from a support ticket. Trust is destroyed.

**Error handling and reporting is the practice of making failures impossible to miss.**

---

### Pillar 1: Structured Logging

**Why needed:** Old-school logs were free-text strings: `"User 42 failed to pay because of error in module"`. Searching, aggregating, and alerting on these is brittle (regex hell).

**The fix:** log structured data — typically JSON — with consistent fields.

#### Compare:

Old:
```text
2026-05-10 22:30:01 INFO User checkout failed for user 42 amount 19.99
```

Structured:
```json
{"ts":"2026-05-10T22:30:01Z","level":"INFO","event":"checkout.failed","user_id":42,"amount":19.99,"reason":"insufficient_funds","trace_id":"abc-123"}
```

Now you can query "all checkout.failed events in the last hour, grouped by reason" in Elasticsearch/Datadog/Loki in one line.

---

### Pillar 2: Correlation IDs (Request IDs)

**Why needed:** A single user request fans out to 20 microservices. When something fails, you need to find every log line related to that one request, across every service.

**Mental picture:** A FedEx tracking number that follows your package across every truck, plane, and sorting facility. Without it, you'd never know where it went.

#### How it works:
- The first service receiving a request generates a UUID like `req-7f3e9c`.
- It includes this ID in every log line, and passes it as a header (e.g., `X-Request-ID`) to every downstream service.
- Every downstream service propagates it.
- When debugging, you grep the trace ID across all services and reconstruct the entire journey.

This is the foundation of **distributed tracing** (next pillar).

---

### Pillar 3: The Three Pillars of Observability — Metrics, Logs, Traces

These three together let you understand what's happening in production:

#### 1. Metrics — numerical aggregates over time
- **What for:** "Is this service healthy *right now*?" "Is latency increasing over the past week?"
- **Examples:** Requests per second, p50/p95/p99 latency, error rate, CPU usage, queue depth.
- **Tools:** Prometheus + Grafana, Datadog, AWS CloudWatch.

#### 2. Logs — discrete event records
- **What for:** "What exactly happened during this request?" Detailed forensics.
- **Tools:** Loki, Elasticsearch + Kibana, Splunk, Datadog Logs.

#### 3. Distributed Traces — end-to-end journey of a request
- **What for:** "Where did the latency come from?" "Which service called which?"
- **Mental picture:** A waterfall chart showing every service hop and how long each took. You can see "Auth took 800ms, that's the bottleneck."
- **Tools:** Jaeger, Zipkin, AWS X-Ray, Datadog APM, Honeycomb.
- **Standard:** OpenTelemetry — vendor-neutral way to instrument code for traces (and metrics + logs too).

#### The mantra:
- **Metrics tell you something is wrong.**
- **Traces tell you which service is wrong.**
- **Logs tell you why.**

---

### Pillar 4: Error Categorization

**Why needed:** Not all errors deserve the same response. Treating them the same wastes ops time on noise and misses real fires.

#### Useful categories:
- **Transient vs Permanent:** Transient (network blip, lock contention) → retry. Permanent (validation error, 404) → don't retry, just return.
- **Client error vs Server error:** 4xx (caller's fault — bad input) vs 5xx (your fault — bug or outage). Pages should fire on 5xx, not 4xx.
- **Expected vs Unexpected:** "Card declined" is expected (log it, don't alert). "NullPointerException in payment handler" is unexpected (page someone).
- **Retryable vs Not:** Make this explicit in your error types. Force callers to handle each case.

---

### Pillar 5: Alerting — paging humans when needed

**Why needed:** Logs sitting in a dashboard nobody looks at = no value. You need *active* notification when things go wrong.

#### Good alerts vs bad alerts:

- **Good alert:** Symptom-based and actionable. "Checkout success rate dropped below 99%" → wake someone up.
- **Bad alert:** Cause-based and noisy. "CPU > 80%" → fires constantly, no one reads it.

#### The SLO/SLI/SLA framework:
- **SLI (Service Level Indicator):** What you measure. "p99 checkout latency."
- **SLO (Service Level Objective):** Your internal target. "p99 checkout latency < 500ms, 99.9% of the time over 30 days."
- **SLA (Service Level Agreement):** What you promise customers (usually weaker than SLO, with penalties).
- **Error budget:** If your SLO is 99.9% (= 43.2 minutes of downtime per month), you have a 43.2-minute "budget" to spend on outages, deploys, experiments. Burn through it and feature work pauses for reliability work. Pioneered by Google SRE.

#### Alerting tools:
- **Prometheus Alertmanager + PagerDuty/OpsGenie/VictorOps.**
- **Datadog Monitors.**
- **Grafana Alerting.**

---

### Pillar 6: Dead Letter Queues (DLQ)

**Why needed:** In async / message-based systems, what do you do with a message that fails repeatedly? Drop it (data loss) or retry forever (poison message blocks the queue)?

**The fix:** After N failed attempts, move the message to a separate "Dead Letter Queue." The main queue keeps moving. Engineers inspect the DLQ to understand and fix the issue.

#### Real systems:
- **AWS SQS:** Built-in DLQ configuration.
- **Kafka:** DLQ topics by convention.
- **RabbitMQ:** Dead Letter Exchanges.

---

**One-sentence summary of section D:** Make failures loud (structured logs, alerts, dashboards), make them traceable (correlation IDs, distributed tracing), and make them categorized (transient vs permanent, expected vs unexpected) so you can act on them.

---

## E. Chaos Engineering — break things on purpose

### The fundamental problem: untested resilience is theater

You set up redundancy, replication, circuit breakers, retries, alerts. You feel safe.

But ask yourself:
- Did you ever actually *test* what happens when a database failover triggers under live traffic?
- Did you ever test that your circuit breaker actually breaks?
- Did you ever test that your alerts fire and the on-call gets paged?

For most teams, the answer is **no, we'll find out during a real outage**. And then you discover:
- The "automatic failover" actually requires a manual step nobody documented.
- The circuit breaker has a typo and never trips.
- The PagerDuty key expired 6 months ago.
- The runbook references a wiki page that was deleted.

**Chaos engineering = injecting failures intentionally and continuously, in production, to verify your resilience actually works.**

---

### The origin story

Netflix moved to AWS in 2010 and immediately faced a problem: AWS instances die randomly. They needed every Netflix service to handle instance death gracefully. But asking 100 dev teams to "test for instance death" never works in practice.

So they built **Chaos Monkey**: a tool that *randomly killed production EC2 instances during business hours*. If your service couldn't survive a random instance death, you'd find out fast — and you'd fix it, because the alternative was being paged constantly.

Within months, every Netflix service was resilient to instance death. Not because devs wanted to be — because the chaos forced them to be.

This grew into the "Simian Army":
- **Chaos Monkey** — kills instances.
- **Latency Monkey** — adds artificial latency to network calls.
- **Conformity Monkey** — kills instances that don't match best practices.
- **Janitor Monkey** — cleans up unused resources.
- **Chaos Gorilla** — kills entire availability zones.
- **Chaos Kong** — kills entire AWS regions.

---

### The principles (from principlesofchaos.org)

1. **Hypothesize about steady state** — define what "normal" looks like (e.g., "checkout success rate is > 99%").
2. **Vary real-world events** — inject the kinds of failures that actually happen in production.
3. **Run experiments in production** — staging doesn't capture real traffic patterns.
4. **Automate experiments** — run them continuously, not as one-off events.
5. **Minimize blast radius** — start small (1% of traffic, one service), expand only when confident.

---

### Concrete failure injection examples

- **Kill an instance:** Does another take its place? Do users notice?
- **Add 500ms latency to all DB calls:** Does the application stay responsive? Do circuit breakers trip?
- **Drop 10% of network packets between Service A and Service B:** Do retries handle it?
- **Saturate the CPU on a node:** Does the load balancer route traffic away?
- **Fill a disk to 100%:** Do alerts fire? Does the system fail safely?
- **Block DNS resolution:** Do services still work via cached entries?
- **Trigger a database failover:** Does the app reconnect? How long is the outage?
- **Kill the entire datacenter:** Does cross-region failover work?

---

### Game days

A scheduled exercise where the team intentionally breaks something and watches the response. Focused on:
- Verifying alerts fire correctly.
- Practicing the on-call rotation.
- Exposing gaps in runbooks.
- Building team confidence.

Amazon, Google, Microsoft all do quarterly "DiRT" (Disaster Recovery Testing) drills.

---

### Tools

- **Chaos Monkey** — Netflix OSS, the original.
- **Gremlin** — commercial, polished UI, runs as an agent.
- **Chaos Mesh** — Kubernetes-native, CRDs for failure injection.
- **LitmusChaos** — Kubernetes-native, CNCF project.
- **AWS Fault Injection Simulator (FIS)** — managed AWS service.
- **Pumba** — chaos for Docker containers.
- **toxiproxy** — programmable network proxy that injects latency, drops, slow connections.

---

### Cultural prerequisites (often missed)

Chaos engineering only works if the org accepts:
- **Blameless post-mortems.** People must feel safe surfacing failures, not punished.
- **Engineering time spent on resilience** (not just features).
- **Leadership buy-in.** A VP needs to believe "intentionally breaking prod" is sane.

Without these, chaos engineering gets shut down the first time it causes a visible incident.

---

**One-sentence summary of section E:** Resilience you haven't tested doesn't exist. Chaos engineering forces you to test it under real conditions, repeatedly, until you actually trust your system.

---

## How these pieces fit together — a real system walkthrough

Let's design **a video streaming service like Netflix**.

1. **Fault Tolerance:**
   - Stateless web tier with 100s of servers behind a load balancer (active-active).
   - Database (Cassandra) with replication factor 3 across availability zones.
   - Multi-region active-active deployment so an entire AWS region failure is survivable.

2. **Graceful Degradation:**
   - If personalization service is down → show "Trending Now" (generic top 10).
   - If subtitles service is down → play video without subtitles.
   - If image CDN is down → show a colored placeholder, video still streams.
   - If 4K source is unavailable → fall back to 1080p automatically.

3. **Cascading Failure Containment:**
   - 200ms timeout on all internal calls.
   - Retries with exponential backoff + jitter on transient failures.
   - Circuit breaker around the recommendations service (with cached fallback).
   - Bulkhead: separate thread pools per downstream dependency.
   - Rate limit: max 5 concurrent streams per account.
   - Load shedding: under extreme load, stop accepting new accounts but let existing streams continue.

4. **Error Handling and Reporting:**
   - Every request gets a UUID; logs are structured JSON; traces flow through Jaeger.
   - Metrics in Prometheus; dashboards in Grafana.
   - SLO: 99.9% of "play" requests succeed within 500ms.
   - Alerts wake on-call when error budget burns 10x normal rate.
   - Failed transcoding jobs go to a DLQ for review.

5. **Chaos Engineering:**
   - Chaos Monkey kills random instances every business day.
   - Quarterly "Chaos Kong" drill: simulate full region failure, validate failover.
   - Game days where on-call practices responding to specific failure scenarios.

The result: a service that streams 250 million hours per day with 99.99% uptime, even though individual components fail constantly.

---

## TL;DR Cheat-Sheet

- **Fault Tolerance = no single points of failure.** Active-active, replication (sync vs async), failover, fencing tokens, sharding for isolation.
- **Graceful Degradation = give them less, not nothing.** Fallbacks, cached responses, feature flags, read-only mode, reduced quality.
- **Cascading Failure Containment = stop the dominoes.** Timeouts → retries with backoff+jitter → circuit breakers → bulkheads → rate limiting → backpressure → load shedding.
- **Error Handling & Reporting = make failure visible.** Structured logs + correlation IDs + metrics/logs/traces + categorization + SLO-based alerts + DLQs.
- **Chaos Engineering = test it, in prod, on purpose.** Chaos Monkey was the breakthrough. If you haven't broken it on purpose, you don't know it's resilient.
- **The whole game:** assume every component will fail; design so the user never notices.
