---
title: "System Design Fundamentals - Load Balancer & API Gateway"
date: 2026-05-08
categories: [System Design]
tags: [load balancer, api gateway]
---

## Load Balancer

**Why it was needed:** In early web architectures, applications ran on a single, powerful server. As traffic grew, vertical scaling (buying bigger hardware) hit physical limits and became cost-prohibitive.

**What happens if we don't use it:** Without a load balancer, all clients connect directly to a single server. If that server crashes or is overwhelmed by traffic spikes, the entire application goes down (a single point of failure). Alternatively, relying purely on DNS to point to different servers doesn't account for server health or connection state.

**Benefits over previous technology:** Unlike a single-server setup or basic DNS round-robin, a load balancer enables horizontal scaling (adding more cheap commodity servers), actively monitors server health to pull dead nodes from rotation, and guarantees high availability.

A **load balancer** sits between clients and servers, distributing incoming traffic across a pool of backend servers to ensure high availability, reliability, and performance. It eliminates single points of failure and prevents any one server from being overwhelmed.

*Example: Netflix uses LBs so that 1 million concurrent users aren't all hitting the same video-streaming server.*

**Where LBs are placed (3 layers):**
1. Client ↔ Web servers
2. Web servers ↔ App/cache servers
3. App servers ↔ Database

### Key Concepts

- **LB Algorithm** — decides which server gets the next request. e.g. Round-robin, Least Connections, IP Hash
- **Health Checks** — periodic pings to detect dead servers; removes them from pool. e.g. HTTP GET `/health` every 10s
- **Session Persistence** — routes same client to same server to preserve state. e.g. sticky sessions via cookie in e-commerce carts
- **SSL/TLS Termination** — decrypts HTTPS at the LB; backends get plain HTTP, offloading CPU-heavy crypto from app servers
- **Backend Server Pool** — the set of servers the LB routes to. e.g. 10 EC2 instances behind an AWS ALB

### How a Request Flows

```
Client → LB (picks server via algorithm) → Backend Server → LB → Client
```

1. LB receives request from client.
2. Evaluates algorithm (capacity, response time, active connections, geo).
3. Forwards request to chosen server.
4. Server processes and returns response to LB.
5. LB relays response back to client.

---

### Load Balancing Algorithms

The goal of any LB algorithm: prevent overload, maximize throughput, minimize latency.

**Round Robin**
Cycles through servers in fixed order (A→B→C→A…). Best for homogeneous servers and stateless apps. Zero overhead — no server state needed. Ignores server load; a slow server still gets its turn, causing queue buildup.

**Weighted Round Robin**
Like RR but each server gets requests proportional to its assigned weight. Best for heterogeneous servers (e.g. 2 big + 3 small nodes). Lets you utilize high-capacity nodes more without manual traffic shaping. Weight tuning is manual and doesn't adapt to real-time load spikes.

**Least Connections**
Routes to the server with fewest active connections right now. Best for variable request durations and stateful apps. Naturally handles slow requests — a busy server stops receiving traffic until connections free up. Requires real-time connection tracking, adding LB-side state and overhead.

**Weighted Least Connections**
Least Connections ÷ server weight — favors high-capacity servers. Best for heterogeneous servers with unpredictable traffic. Adapts to load *and* respects server capacity differences. Most complex to tune; needs both accurate weights and live connection state.

**IP Hash**
`hash(client_IP) % n` always maps to the same server. Best for stateful apps needing sticky sessions without cookies. Session persistence without any shared session store (e.g. Redis) needed. Adding/removing servers re-maps many clients (consistent hashing mitigates this).

**Least Response Time**
Routes to the server with the lowest recent average response time. Best for low-latency apps (gaming, trading, streaming). Directly optimizes for user-perceived latency. Response time probes add network overhead; noisy measurements cause thrash.

**Random**
Picks a server at random. Best for homogeneous servers and stateless apps. Statistically converges to even distribution over time with no bookkeeping. No load awareness; unlucky streaks can temporarily overload a server.

**Least Bandwidth**
Routes to the server consuming least bandwidth right now. Best for video streaming, CDNs, large file transfers. Prevents a single server from becoming a network bottleneck during large transfers. Bandwidth metrics fluctuate rapidly, potentially causing frequent re-routing.

**Custom Load**
You define the metric (CPU%, memory, app KPIs) and routing rules. Best for complex apps with unique resource profiles. Can encode domain-specific knowledge (e.g. route ML inference to GPU-heavy nodes only). Hardest to configure correctly; a bad metric definition silently degrades performance.

#### Consistent Hashing

**Why it was needed:** Simple modulo hashing (`hash(key) % n`) is fragile because adding or removing a server changes `n`, causing nearly every client to remap to a different server.
**What happens if we don't use it:** Removing one node from a cluster causes a massive re-shuffling of traffic. All session state or cache data is lost at once, leading to sudden cache stampedes and database overloads.
**Benefits over previous technology:** By mapping servers and keys to a logical ring, only a fraction of keys (`k/n`) are remapped when a server is added or removed, preventing mass session disruption and preserving cache stability.

Consistent hashing fixes this. The idea:
- Arrange a logical ring of 2³² slots (0 to 2³²−1).
- Hash each server to a position on the ring.
- For each request, hash the client key and walk clockwise until you hit a server — that server owns this key.

When a server is removed, only the keys it owned move to the next server clockwise. Everything else stays put. On average only `k/n` keys remap (where `k` = total keys, `n` = number of servers), instead of nearly all of them.

**Virtual nodes** — a single physical server is placed at multiple points on the ring (e.g. 150 virtual nodes per server). This prevents hotspots when servers are unevenly distributed and makes load more uniform.

```
Ring (simplified, 3 servers):
  0 ──── S1 ──── S2 ──── S3 ──── 2³²
Client key hashes to position X → walk clockwise → hits S2 → S2 handles it.
S2 removed → those keys now walk to S3. S1 and its keys untouched.
```

*Why it matters for LBs:* used in IP Hash to prevent mass session disruption on topology changes. Also the foundation of distributed caches (Memcached, DynamoDB, Cassandra ring) — understanding it shows you can reason about distributed state, not just traffic routing.

#### Quick decision guide

```
Servers identical + stateless?          → Round Robin or Random
Servers differ in power?                → Weighted Round Robin
Request durations vary widely?          → Least Connections
Powerful servers + variable traffic?    → Weighted Least Connections
Must pin client to same server?         → IP Hash (or sticky cookies)
Latency is the #1 SLA?                  → Least Response Time
Traffic is bandwidth-heavy (video/CDN)? → Least Bandwidth
None of the above fit?                  → Custom Load
```

#### IP Hash example (interview-ready)
3 servers. Client IP `192.168.1.10` → `hash(IP) = 17` → `17 % 3 = 2` → Server C.  
Same client always hits Server C — natural session persistence with no cookie needed.  
*Caveat:* remove Server C and `17 % 2 = 1` → client now lands on Server B (session lost).

---

### Uses of Load Balancing

**1. High Availability & Fault Tolerance**
LB performs continuous health checks on backend servers. If a server fails to return a 200 OK within the timeout, it's removed from rotation instantly and traffic is rerouted to healthy servers.

*Example (Uber):* 50 API servers, Server #3 freezes from a memory leak. Without LB: 2% of users hit a dead server and churn to Lyft. With LB: health check fails, Server #3 is cut, traffic spreads across remaining 49 — users notice nothing.

**2. Horizontal Scalability**
Clients only know the LB's address (Virtual IP). When traffic spikes, spin up more backend instances and register them with the LB — the internet doesn't need to know anything changed.

*Example (Black Friday):* Traffic spikes from 1k to 100k RPS. Auto-scaling boots 100 new EC2 instances, they register with the LB, and the surge is absorbed instantly. No DNS changes, no client reconfiguration.

**3. Zero-Downtime Deployments**
LB supports connection draining (stop sending new requests to a server, let existing ones finish) and Blue-Green deployments (shift traffic gradually between old and new versions).

*Example (Banking API):* Route 1% to Green (new version) → monitor logs → 10% → 50% → 100%. Bug found at 1%? Instantly revert to Blue. No downtime, no failed transactions.

**4. Security & DDoS Mitigation**
LB acts as a reverse proxy — backend IPs are never exposed to the internet. Can integrate with WAF rules to drop malicious traffic at the edge before it reaches application servers.

*Example (Social Media):* Botnet launches a SYN Flood on the login page. AWS ALB + WAF detects the pattern, drops the connections at the edge. Backend servers only see legitimate traffic.

**5. SSL/TLS Termination**

**Why it was needed:** Decrypting HTTPS traffic requires significant CPU cycles. Managing SSL certificates across hundreds of backend servers is operationally complex and error-prone.
**What happens if we don't use it:** Every single backend server must spend a large chunk of its CPU capacity purely on cryptographic math rather than business logic. Updating an expiring SSL certificate requires touching every single server in the fleet.
**Benefits over previous technology:** Centralizes certificate management at the edge. By decrypting traffic at the load balancer (client speaks HTTPS to LB, LB speaks HTTP to internal backends), backend CPU usage drops dramatically, freeing up capacity for actual application logic.

HTTPS decryption is CPU-heavy. Offload it to the LB — client speaks HTTPS to LB, LB speaks HTTP to backends inside the private network. Backends spend cycles on business logic instead of crypto math.

*Example (HFT Dashboard):* Backends at 90% CPU, 30% of that just from OpenSSL. Move SSL certs to the LB → backend CPU drops to 60% → effectively 1/3 more capacity with no new hardware.

---

### Load Balancing Types

These 8 types fall into 3 buckets — a useful mental model for interviews:

#### Bucket A: Implementation (how is the LB built?)

**Hardware LB**
A dedicated physical appliance using ASICs/FPGAs. Highest throughput, lowest latency, but expensive and hard to scale — you buy more boxes. Best for large enterprises with predictable, high-volume traffic. *Example: F5 BIG-IP in front of a bank's data center.*

**Software LB**
Runs on a general-purpose server or VM (e.g. Nginx, HAProxy). Cheaper, flexible, cloud-deployable, easy to scale vertically or horizontally. Slightly lower raw performance than hardware under extreme load. *Example: Nginx distributing traffic across app servers on EC2.*

**Cloud-based LB**
Fully managed service from a cloud provider (AWS ALB/NLB, GCP Load Balancer, Azure LB). Auto-scales, zero maintenance, pay-per-use. Tradeoff: less control, potential vendor lock-in. *Example: AWS ALB routing API requests to an ECS cluster.*

#### Bucket B: Scope (how wide is the traffic distribution?)

**DNS Load Balancing**
A domain resolves to multiple IPs; clients get different IPs on each lookup (round-robin or geo-based). No server health awareness, no session persistence. Updates are slow due to TTL caching — a dead server can still receive traffic for minutes. *Example: Cloudflare has ~300 edge nodes worldwide. When you visit `example.com`, your DNS resolver queries Cloudflare's nameserver. It responds with the IP of the nearest edge node based on your location — a user in Mumbai gets a Mumbai IP, a user in London gets a Frankfurt IP. The browser connects to that edge node, not the origin server. No health checks, no connection state — just DNS returning different IPs to different clients.*

**Global Server Load Balancing (GSLB)**
Smart DNS + health checks across multiple data centers. Routes users to the closest or healthiest DC. Supports failover across regions. Still subject to DNS TTL lag. *Example: A multinational SaaS app runs two DCs — `eu-west` (Frankfurt) and `us-east` (Virginia). GSLB continuously health-checks both. A user in Berlin queries `app.example.com` → GSLB sees they're in Europe and Frankfurt is healthy → returns the Frankfurt IP. If Frankfurt goes down (health check fails), GSLB stops returning that IP and EU users are transparently failed over to Virginia — with some added latency but no outage. Compare this to plain DNS LB: without health checks, DNS would keep returning the Frankfurt IP even after it's dead, until TTL expires.*

**Hybrid LB**
Combines multiple types — e.g. hardware LBs inside Data Centers for raw performance + cloud LBs for elastic scale + DNS LB for global routing. Most flexible but most complex. *Example: Netflix uses hardware LBs in co-lo DCs, AWS LBs for cloud traffic, and DNS LB for global geo-routing.*

**Geo-routing** (used by DNS LB, GSLB, and CDNs — a routing policy, not a separate LB type)
Routes a client to a server based on their geographic location, determined from their IP address via databases like MaxMind or RIR registries (ARIN, RIPE). Why it matters:
- **Latency** — a user in Tokyo hitting a Virginia server adds ~150ms. Geo-routing sends them to a Tokyo/Singapore node instead.
- **Data residency** — GDPR requires EU user data to stay in the EU. Geo-routing ensures EU traffic never touches US servers.
- **Compliance & content rules** — stream licensing, regional pricing, or legal restrictions enforced at the routing layer.

#### Bucket C: Logic (what does it inspect to route?)

**Layer 4 (Transport Layer)**
Routes based on IP + port from the TCP/UDP header only. Never opens the packet. Fast and protocol-agnostic, but "dumb" — can't distinguish `/api/users` from `/api/orders`. *Example: Gaming platform distributing UDP traffic across game servers by IP + port.*

**Layer 7 (Application Layer)**
Reads HTTP headers, cookies, URL paths, and body content to make routing decisions. Enables content-based routing, sticky sessions, SSL termination, A/B testing. Slower than L4 due to deep packet inspection. *Example: Microservices gateway routing `/api/payments` to the payments service and `/api/search` to the search service.*

#### L4 vs L7 — the key tradeoff

L4 is faster (no content inspection) but blind to application context. L7 is smarter (can route by URL, inject headers, terminate SSL) but adds latency. Most modern systems use L7 at the edge and L4 internally where raw throughput matters.

---

### High Availability & Fault Tolerance for Load Balancers

#### Redundancy — failover configurations

**Active-Passive — implemented via VRRP**
One LB handles all traffic; the other sits idle on standby. VRRP (Virtual Router Redundancy Protocol) lets both nodes share a single Virtual IP (VIP). The MASTER owns the VIP and sends a multicast heartbeat every ~1s. If BACKUPs don't hear it for `(3 × interval)` seconds, the highest-priority BACKUP promotes itself, claims the VIP, and sends a gratuitous ARP to update the network's MAC-to-IP mapping. Clients keep hitting the same VIP — they never know a failover happened. Failover takes ~3s. Simple but the standby node sits idle.

*Example: Two HAProxy nodes share VIP `10.0.0.1`. Primary crashes → secondary takes the VIP within ~3s → zero config change needed on clients or backends.*

**Active-Active — implemented via BGP**
Multiple LB instances all process traffic simultaneously. BGP (Border Gateway Protocol) lets each node advertise the same IP prefix to upstream routers. Routers use ECMP (Equal-Cost Multi-Path) to spread traffic across all nodes — this is called Anycast. If a node dies, its BGP session drops and routers automatically withdraw its route; traffic stops going there within seconds. No election, no VIP takeover. All nodes are active so no resources are wasted, but it requires BGP-capable routers and proper AS/prefix configuration.

*Example: Cloudflare runs the same IP (`1.1.1.1`) on nodes in 300+ cities. If the Frankfurt node goes down, BGP withdraws its route and queries automatically reroute to the next closest node.*

#### Split-Brain (Active-Active failure mode)

Split-brain happens when two LB nodes lose connectivity to *each other* but can still reach clients and backends. Each node thinks the other is dead and starts handling all traffic independently — two LBs now have diverging views of session state, connection counts, and routing decisions.

This is particularly dangerous for stateful LBs where session data is node-local. Fixes:
- Use an external quorum system (etcd, ZooKeeper) — a node can only act as primary if it holds a quorum lease. Without quorum it stops serving rather than risking split-brain.
- Externalize all state (Redis, DB) so both nodes read/write the same source of truth — diverging local state becomes irrelevant.
- Design for it: prefer stateless LBs + externalized session stores so split-brain has no correctness impact, only a brief availability blip.

#### Health Checks & Monitoring

**Why it was needed:** Servers crash, get stuck in garbage collection, or lose database connectivity. A load balancer needs a way to know which servers are actually capable of serving traffic.
**What happens if we don't use it:** The load balancer blindly sends requests to a dead or frozen server. Users experience timeouts, 502 Bad Gateway errors, and overall reduced availability.
**Benefits over previous technology:** Continuously probes backend instances and automatically removes unhealthy nodes from the routing pool. This ensures users only ever interact with healthy servers, preventing cascading failures.

LBs run periodic health checks against backend servers — e.g. HTTP GET `/health` must return 200 within 2s. Failing servers are pulled from the pool automatically, preventing traffic from hitting dead nodes and avoiding cascading failures.

The LB itself must also be monitored. Key metrics to watch: response times, error rates (4xx/5xx), active connection count, CPU/memory. Alerts should fire before thresholds become outages, not after.

**Health check flapping & hysteresis**
A server that oscillates between healthy and unhealthy (e.g. GC pauses, intermittent DB timeouts) causes the LB to rapidly add and remove it from the pool — destabilizing traffic distribution and producing a noisy alert storm.

Fix with hysteresis: require N consecutive failures before marking a server unhealthy, and M consecutive successes before re-admitting it. A common production setting is 3 failures to pull, 2 successes to restore. This adds a small delay to failure detection but prevents flapping from one transient blip.

**Thundering herd on recovery**
When a pulled server comes back online and the LB re-admits it, all the traffic that was being held back floods in simultaneously. The server may be overwhelmed by the burst and crash again immediately — creating a crash loop.

Fix with slow-start: re-introduce the recovered server at a low traffic weight (e.g. 10%) and ramp up over 30–60s as it proves stability. Nginx and AWS ALB both support this natively. Without slow-start, recovery becomes a repeated failure cycle.

#### State Synchronization (Active-Active & Active-Passive)

When multiple LB instances run in parallel, they need a consistent view of the world — which backends are healthy, what session data exists, what config is current. Two approaches:

**Centralized config management** — a single source of truth (etcd, Consul, ZooKeeper) that all LB instances read from. Config changes propagate instantly to all nodes without manual sync.

**State sharing & replication** — for session data, use a distributed cache (Redis, Memcached) or DB replication so any LB instance can look up any session. This is what enables stateful apps to work in active-active setups without sticky sessions.

---

### Stateless vs. Stateful Load Balancing

**Stateless LB**
Makes routing decisions purely from the incoming request — IP, URL, headers. No memory of past requests. Fast and horizontally scalable since any LB node can handle any request.
*Example: A product search API — each search is independent, no session needed. LB routes by geo or round-robin without caring who the user is.*

**Stateful LB**
Remembers which server a client was assigned to and pins all subsequent requests from that client to the same server (sticky sessions). Required when session data lives on the server (not in a shared store).
*Example: A banking app where login state is stored in-memory on the server. LB must route all requests from that user to the same server, otherwise they're logged out mid-session.*

Stateful LB has two sub-types:

**Source IP Affinity** — pins a client to a server based on their IP address (`hash(IP) % n`). Simple, no cookie needed. Breaks on mobile networks where IPs change frequently (carrier NAT, switching towers).

**Session Affinity (Sticky Sessions)**
**Why it was needed:** Legacy or stateful applications store user session data (like login status or shopping carts) in local memory on a specific server, rather than a shared external database.
**What happens if we don't use it:** A user logs into Server A, but their next request is routed to Server B (which doesn't have their session). The user is randomly logged out or loses their cart contents.
**Benefits over previous technology:** Ensures that once a user starts interacting with a specific server, all their subsequent requests are routed back to that same server, preserving their state without requiring immediate application rewrites to externalize sessions.

LB injects a cookie (e.g. `SERVERID=s2`) on the first response. All subsequent requests carry that cookie and the LB reads it to route to the correct server. More reliable than IP affinity since it survives IP changes.

**When to use which:**

Stateless — default choice. Easier to scale, no LB-side state, any server can handle any request. Works best when session state is externalized (Redis, DB).

Stateful — use only when you can't externalize session state, or for legacy apps. Introduces a scaling constraint: losing the pinned server loses the session.

---

### Scalability & Performance

#### Scaling the LB itself

**Vertical scaling** — give the existing LB more CPU, memory, and NIC bandwidth. Simple but hits a hard ceiling. Good for short-term relief, not a long-term strategy.

**Horizontal scaling** — add more LB instances. Requires traffic to be split across them, typically via DNS LB (multiple A records) or a dedicated upstream LB tier. Pairs naturally with active-active. No theoretical ceiling — just add nodes.

#### Connection & Request Rate Limiting

Overloading an LB is as bad as overloading a backend. LBs can enforce rate limits based on IP, client domain, or URL pattern — e.g. max 100 req/s per IP. This serves two purposes: prevents any single client from monopolizing resources, and blunts DoS/DDoS attacks before they reach backend servers.

*Example: AWS ALB + WAF rate rule — block any IP sending >1000 req/10s to `/api/login`.*

#### Caching & Content Optimization

LBs can cache static assets (images, CSS, JS) at the edge, serving them directly without hitting backend servers. Some also support response compression (gzip/brotli) and minification, reducing payload size and bandwidth. This is more commonly handled by a CDN layer in front of the LB, but the capability exists.

#### LB Latency Impact

Every LB adds one extra network hop. The overhead is usually <1ms on a local network, but it compounds — client → LB → app server → LB → client is two extra hops vs. direct. Strategies to minimize:

**Geo-distribution** — deploy LB instances close to users so the first hop is short. A user in Mumbai hitting a Mumbai LB adds ~0.2ms; hitting a Virginia LB adds ~150ms.

**Connection reuse (keep-alive)** — instead of opening a new TCP connection to a backend for every request, the LB maintains a pool of persistent connections. Eliminates the TCP + TLS handshake overhead on every request. Critical at high RPS.

**Protocol optimizations** — HTTP/2 multiplexes multiple requests over one connection (no head-of-line blocking). QUIC (HTTP/3) runs over UDP, eliminating TCP handshake latency entirely. LBs that support these protocols pass the benefit downstream to clients.

---

### Challenges of Load Balancers

**Single Point of Failure** — if the LB itself goes down, the entire app goes down. Fix: run multiple LB instances in active-passive (VRRP) or active-active (BGP) configuration.

**Configuration Complexity** — wrong algorithm, misconfigured timeouts, or broken health check paths can cause uneven distribution or silent outages. Fix: version-control your LB config, use IaC (Terraform, Ansible), and test health check endpoints explicitly.

**Scalability Bottleneck** — at extreme traffic, the LB itself becomes the ceiling. Fix: horizontal scaling via DNS LB or a tiered LB architecture; prefer cloud-managed LBs that scale automatically.

**Added Latency** — every request passes through the LB, adding a network hop. Typically <1ms on LAN but meaningful at global scale. Fix: geo-distribute LB instances, use keep-alive connection pools, enable HTTP/2 or QUIC.

**Sticky Sessions trade-off** — session affinity ensures correctness for stateful apps but causes uneven load distribution (one server gets a "heavy" user stuck to it). Fix: externalize session state to Redis so any server can handle any request, making the LB truly stateless.

**Cost** — hardware LBs are expensive upfront; cloud LBs charge per LCU/hour and per GB processed. Fix: right-size to traffic patterns, use open-source software LBs (Nginx, HAProxy) for cost control, leverage spot/reserved pricing for cloud LBs.

**Health Check Gaps** — a server can pass a shallow `/health` ping but still be broken at the application layer (DB connection pool exhausted, downstream service down). Fix: implement deep health checks that exercise real dependencies, and set appropriate thresholds before marking a server unhealthy.

---

## API Gateway

**Why it was needed:** As architectures shifted from monoliths to microservices, clients had to keep track of dozens of different service endpoints, protocols, and authentication methods. This made client-side logic incredibly complex and coupled clients tightly to backend implementations.

**What happens if we don't use it:** Clients would have to communicate directly with each individual microservice. This means making multiple round trips over the network to gather data (e.g., fetching user data, then order data), handling authentication independently for every service, and exposing internal network structures to the public internet.

**Benefits over previous technology:** Compared to direct client-to-microservice communication, an API Gateway provides a single, unified entry point. It reduces client network round trips via response aggregation, offloads cross-cutting concerns (like auth and rate limiting) from the microservices, and hides the internal backend architecture from the outside world.

An **API Gateway** is a server-side component that acts as the **single entry point** for all client traffic into a backend system. It sits between clients (browser, mobile, other services) and your backend microservices — receiving requests, routing them to the right service, and returning responses.

*Example: In a food delivery app, a single POST `/order` from the mobile app hits the API gateway, which fans out to the inventory service, pricing service, and notification service — the client talks to one endpoint, not three.*

```
Client → API Gateway → [Auth + Rate Limit + Route] → Microservice(s)
```

---

### Is an API Gateway just an L7 Load Balancer?

Essentially yes — but it goes further. An API gateway operates at L7 and does everything an L7 LB does (content-based routing, SSL termination, header inspection), but adds a layer of **traffic governance** on top.

Both read HTTP content and route by URL/headers. The difference is what happens next:
- **L7 LB** stops there — it picks an instance and forwards the request.
- **API Gateway** also enforces auth, rate limits, transforms the request/response, translates protocols (REST↔gRPC), and can aggregate responses from multiple services. Its primary concern is not just *which instance* but *which service* and *what to do before and after*.

Auth and rate limiting are bolt-ons for an L7 LB; they are core features of a gateway. Response aggregation and protocol translation don't exist at the LB layer at all.

Some tools blur the line — Nginx with plugins or AWS ALB with Lambda authorizers can approximate a gateway — but purpose-built gateways (Kong, Apigee, AWS API Gateway) are the right tool when you need the full feature set.

**Rule of thumb:** Route and distribute only → L7 LB. Govern traffic (who can call what, how often, in what shape) → API Gateway.

---

### API Gateway vs. Load Balancer

**Primary job**
- API Gateway: route to the *right service* based on URL, method, headers.
- Load Balancer: spread requests *evenly* across instances of the same service.

**Traffic type**
- API Gateway: API traffic — HTTP, gRPC, WebSocket.
- Load Balancer: anything — TCP, UDP, HTTP.

**Routing logic**
- API Gateway: content-aware — URL path, headers, body.
- Load Balancer: connection-aware — IP, port, server health, weight.

**Cross-cutting concerns**
- API Gateway: auth, rate limiting, logging, transformation.
- Load Balancer: SSL termination, health checks, session persistence.

*Example: `/api/payments` hits the gateway → gateway routes to the payments service → an LB distributes that request across 10 instances of the payments service.*

**Key insight:** A load balancer distributes traffic *within* a service tier. An API gateway decides *which* service tier the traffic goes to. Most production systems use both — gateway at the edge, LBs behind each service cluster.

---

### Core Responsibilities

**1. Routing**
Maps incoming URL patterns to backend services. Can be path-based (`/users` → user-service), method-based (GET vs POST), or header-based (version routing via `API-Version: v2`).

*Example: AWS API Gateway routing `GET /products/{id}` to a Lambda function and `POST /orders` to an ECS container.*

**2. Authentication & Authorization**
Validates identity (JWT, OAuth2, API keys) at the gateway layer before the request ever touches a backend. Centralizes auth logic — no service needs to re-implement it.

*Example: Kong validates a Bearer token against an identity provider. Invalid token → 401 returned immediately, backend never called.*

**3. Rate Limiting**
Enforces request quotas per client/IP/API key. Protects backends from overload and prevents abuse.

*Example: Free tier → 100 req/min, Pro tier → 10,000 req/min. Gateway checks a Redis counter on every request.*

**4. Request/Response Transformation**
Modifies payloads on the fly — add/strip headers, translate protocols (REST ↔ gRPC), aggregate multiple service responses into one.

*Example: BFF (Backend for Frontend) pattern — mobile gateway aggregates user profile + notifications + cart into a single response, saving the mobile client 3 round trips.*

**5. SSL/TLS Termination**
Clients speak HTTPS to the gateway; internal service-to-service calls use plain HTTP on a private network. Same benefit as at the load balancer — offloads crypto overhead from application servers.

**6. Observability**
Centralized logging, metrics, and distributed tracing for all traffic. Every request passes through, so it's the natural place to capture latency, error rates, and usage analytics.

---

### Request Flow

```
Client
  │  HTTPS
  ▼
[Load Balancer]          ← distributes across multiple gateway instances
  │  HTTPS
  ▼
[API Gateway]            ← single logical entry point; all smart logic lives here
  │  1. TLS termination
  │  2. Auth check (JWT/API key)
  │  3. Rate limit check (Redis)
  │  4. Request transformation
  │  5. Route decision: which service?
  │
  ├──▶ [Load Balancer]   ← distributes across instances of the target service
  │         │
  │         ▼
  │    [Microservice A]  (e.g. /api/orders → order-service)
  │
  └──▶ [Load Balancer]
            │
            ▼
       [Microservice B]  (e.g. /api/users → user-service)
```

**Where each sits:**
- **Load Balancer (outer)** — in front of the gateway cluster. Its only job is to spread traffic evenly across multiple gateway instances so the gateway itself isn't a SPOF or bottleneck.
- **API Gateway** — the intelligent layer. Handles all cross-cutting concerns: auth, rate limiting, routing logic, transformations.
- **Load Balancer (inner)** — one per service, behind the gateway. Routes gateway traffic evenly across the N instances of that specific microservice.

The gateway decides *where* traffic goes. The inner LBs decide *which instance* of that destination handles it.

---

### Key Trade-offs

**Single point of failure** — all traffic flows through the gateway, so it must be highly available. Fix: run multiple instances behind a load balancer (ironically, LBs protect the gateway).

**Added latency** — every request takes an extra hop + auth/rate-limit checks. Typical overhead: 1–5ms. Fix: keep auth logic fast (local JWT validation > remote token introspection), use connection pooling to backends, run gateway instances close to clients.

**Bottleneck at scale** — at millions of RPS, the gateway itself can be the ceiling. Fix: horizontal scaling, stateless design (no session state in the gateway), push rate-limit counters to Redis.

**Configuration complexity** — routing rules, auth policies, rate limits, and transformations can become a sprawling config mess. Fix: treat gateway config as code (Terraform, Pulumi), use declarative config (Kubernetes Gateway API, Kong declarative config).

---

### Usages

**Request Routing** — maps URL + method to the right backend service.
*Example: `GET /products/{id}` → product-service, `POST /orders` → order-service.*

**Security Enforcement** — validates auth tokens and permissions before the request touches any backend. One place to fix, covers all services.
*Example: invalid Bearer token → 401 at the gateway, backend never called.*

**Rate Limiting**
**Why it was needed:** APIs are vulnerable to abuse, scraping, runaway scripts, and DDoS attacks. Resource allocation also needs to be controlled based on pricing tiers.
**What happens if we don't use it:** A single malicious or misconfigured client can overwhelm backend databases and crash the entire system. Free-tier users could consume expensive compute resources without limits.
**Benefits over previous technology:** Blocks excessive traffic at the network edge before it ever reaches internal servers. Enforces per-client/per-tier quotas (e.g., via Redis counters) returning a `429 Too Many Requests`, protecting backend stability and enforcing business models.

Enforces per-client/per-tier quotas via Redis counters. Returns `429` when exceeded.
*Example: free tier → 100 req/min, pro → 10,000 req/min.*

**Service Aggregation** — fans out to multiple services in parallel, merges into one response, eliminating client round trips.
*Example: mobile dashboard fans out to user-service + order-service + recommendations in parallel → one JSON blob back to client.*

**Protocol & Format Translation** — bridges mismatches between client and service protocols/formats without changing either side.
*Example: client speaks REST/JSON, backend speaks gRPC — gateway translates transparently.*

**Caching** — serves repeated read requests from cache, bypassing the backend entirely.
*Example: product catalog cached at the gateway → read latency drops 10× on high-traffic pages.*

---

### Common Patterns

**BFF (Backend for Frontend)**
**Why it was needed:** Different clients (mobile phones, web browsers, smart TVs) have drastically different UI requirements, bandwidth limits, and processing power. A single, monolithic API couldn't serve them all optimally.
**What happens if we don't use it:** The backend returns a massive "one-size-fits-all" JSON payload. A mobile app over a 3G network is forced to download 5MB of data just to display a simple list, wasting battery and bandwidth.
**Benefits over previous technology:** Creates a dedicated gateway instance per client type (mobile, web, partner). Each BFF aggregates and shapes the exact data its specific client needs, minimizing payload size and optimizing the user experience.

A separate gateway instance per client type (mobile, web, partner). Each BFF shapes responses for its client, avoiding the "one-size-fits-all" API problem.
*Example: Netflix has separate BFFs for TV, mobile, and browser — each aggregates different data and formats responses to match device constraints.*

**Gateway Aggregation**
**Why it was needed:** In microservices, displaying a single UI page (like a user dashboard) often requires data from 5+ different services.
**What happens if we don't use it:** The client makes 5 separate HTTP requests to the backend over the public internet. This causes high latency due to multiple network round trips and complicates client-side code.
**Benefits over previous technology:** The client makes one single request to the API Gateway. The Gateway fans out requests to the internal microservices over the high-speed internal network, stitches the results together, and returns one cohesive response to the client.

Fans out to multiple services in parallel, stitches results into one response, reducing client round trips.
*Example: dashboard needs user data + activity feed + billing status → one client request, three parallel backend calls, one merged response.*

**Gateway Offloading** — move cross-cutting concerns (auth, logging, SSL, compression) out of every microservice and into the gateway. Services stay focused on business logic.

---

### Advantages

**Centralized cross-cutting concerns** — auth, rate limiting, logging, and SSL handled once at the edge instead of re-implemented in every service.

**Simplified client integration** — clients talk to one stable endpoint regardless of how many services exist behind it or how they change.

**Observability** — complete traffic picture (latency, error rates, usage) in one place without instrumenting every service.

**Protocol flexibility** — clients and services can use different protocols/formats and evolve independently.

---

### Disadvantages

**SPOF** — all traffic flows through it; an outage takes down everything. Requires active-active clustering, which adds infra cost.

**Added latency** — extra hop + auth/rate-limit processing on every request. Typically 1–5ms, but it compounds under heavy transformation.

**Vendor lock-in** — managed gateways (AWS API Gateway, Apigee) tie you to that provider's pricing and limits.

**Configuration complexity** — routing rules, policies, and versioning accumulate fast. Without IaC it becomes a liability.

---

## LB + API Gateway in Production

Both components coexist in every large system. The pattern is always the same — LB handles distribution, gateway handles intelligence.

**Netflix**
- Global DNS LB (Route 53) routes users to the nearest AWS region.
- Inside each region, an LB (AWS ALB) spreads traffic across a cluster of Zuul gateway instances.
- Zuul handles auth, rate limiting, and per-device BFF routing (TV vs mobile vs browser get different response shapes).
- Behind Zuul, each microservice (streaming, recommendations, billing) has its own LB distributing across its instances.

**Uber**
- Edge LB receives all rider/driver app traffic.
- API gateway (built on Nginx + Lua) authenticates requests, enforces rate limits, and routes by service — trips, payments, maps, notifications each get their own backend cluster.
- Each cluster has an internal LB. At Uber's scale (~1M RPS peak), the gateway is stateless and horizontally scaled; rate-limit state lives in Redis.

**AWS (as a pattern, not just a vendor)**
- Route 53 (DNS LB) → CloudFront (edge caching + DDoS) → ALB (distributes to gateway instances) → API Gateway (auth, routing, throttling) → ALB per service → ECS/Lambda instances.
- The outer ALB protects the gateway from being a SPOF. The inner ALBs protect individual services.

**Key takeaway for interviews:** When asked to design any large system, always place an LB in front of your gateway cluster (HA for the gateway) and an LB behind it per service (distribution within each service). The gateway never talks directly to a single service instance.

---

## Service Mesh vs. Load Balancer

**Why it was needed:** In microservices, services must communicate with each other (east-west traffic). Initially, developers hardcoded retries, timeouts, and mutual TLS logic into every application's codebase using libraries. As the number of services and languages grew, managing and updating these networking libraries became a nightmare.

**What happens if we don't use it:** Services communicate directly over the internal network without encrypted traffic (mTLS) or rely on inconsistent, language-specific libraries for circuit breaking and retries. Observability into service-to-service calls becomes fragmented and hard to trace.

**Benefits over previous technology:** Compared to application-level networking libraries, a service mesh extracts networking logic entirely out of the application code into a dedicated infrastructure layer (sidecar proxies). This provides consistent security, routing, and observability across all services regardless of the language they are written in.

A traditional LB handles **north-south traffic** — requests coming in from the outside world to your services. A service mesh handles **east-west traffic** — service-to-service calls inside your cluster.

In a microservices architecture, Service A calling Service B doesn't go through the central LB. Instead, a service mesh (Istio, Linkerd, Envoy) deploys a sidecar proxy alongside every service instance. All outbound calls go through the sidecar, which handles:
- Load balancing between instances of the target service
- Mutual TLS (mTLS) between services
- Retries, timeouts, circuit breaking
- Distributed tracing and observability

**When to use a traditional LB vs. a service mesh:**

Use a traditional LB when: you have a monolith or a small number of services, traffic enters from outside, you need SSL termination at the edge, or you want simplicity.

Use a service mesh when: you have many microservices communicating with each other, you need per-service traffic policies (canary between internal services, not just at the edge), mTLS between every service pair is a security requirement, or you need fine-grained observability across service calls.

*Example: At Uber's scale, hundreds of microservices call each other millions of times per second. A central LB can't manage that — every service pair would need its own LB rule. A service mesh handles it at the pod level, automatically, without central configuration.*

The two aren't mutually exclusive — most large systems use both: an LB (AWS ALB, Nginx) at the edge for north-south, and a service mesh (Istio + Envoy) for east-west.

---

## Interview Pitfalls

- Conflating API gateway and load balancer — know their distinct roles and that most systems use both.
- Ignoring HA for the gateway itself — "single entry point" sounds like a SPOF until you clarify it runs as a clustered, load-balanced tier.
- Overlooking latency impact — always mention the overhead of auth checks and when to prefer local JWT validation over remote introspection.
- Forgetting the BFF pattern — interviewers often probe whether you know that a single monolithic gateway doesn't serve all client types well.
- Not addressing security depth — gateway handles edge auth, but services should still validate internally (defense in depth).
- Treating LB algorithms as interchangeable — know when to pick Least Connections over Round Robin and why IP Hash breaks on server removal.
- Ignoring the LB as a SPOF — the same HA reasoning that applies to backends applies to the LB tier itself.
- Conflating L4 and L7 tradeoffs — L4 for raw throughput, L7 for application-aware routing; most systems use both at different tiers.

---

## Recap

**Load balancer** = traffic distributor. Spreads requests across instances of the same service, enforces health checks, enables horizontal scaling and zero-downtime deployments. Its concern is *which instance* handles a request.

**API gateway** = smart traffic cop at the edge. Routes requests to the right service, enforces auth + rate limits, offloads cross-cutting concerns, and provides a single observability point — so microservices stay focused on business logic. Its concern is *which service* and *what to do before and after*.

In production: LB in front of the gateway cluster (HA for the gateway) → gateway (auth, routing, rate limits) → LB per service cluster (distribution within each service) → service instances.
