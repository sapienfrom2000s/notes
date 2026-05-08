---
title: "System Design Fundamentals - API Gateway"
date: 2026-05-08
categories: [System Design]
tags: [api gateway]
---

## What It Is

An **API Gateway** is a server-side component that acts as the **single entry point** for all client traffic into a backend system. It sits between clients (browser, mobile, other services) and your backend microservices — receiving requests, routing them to the right service, and returning responses.

*Example: In a food delivery app, a single POST `/order` from the mobile app hits the API gateway, which fans out to the inventory service, pricing service, and notification service — the client talks to one endpoint, not three.*

```
Client → API Gateway → [Auth + Rate Limit + Route] → Microservice(s)
```

---

## Is an API Gateway just an L7 Load Balancer?

Essentially yes — but it goes further. An API gateway operates at L7 and does everything an L7 LB does (content-based routing, SSL termination, header inspection), but adds a layer of **traffic governance** on top.

Both read HTTP content and route by URL/headers. The difference is what happens next:
- **L7 LB** stops there — it picks an instance and forwards the request.
- **API Gateway** also enforces auth, rate limits, transforms the request/response, translates protocols (REST↔gRPC), and can aggregate responses from multiple services. Its primary concern is not just *which instance* but *which service* and *what to do before and after*.

Auth and rate limiting are bolt-ons for an L7 LB; they are core features of a gateway. Response aggregation and protocol translation don't exist at the LB layer at all.

Some tools blur the line — Nginx with plugins or AWS ALB with Lambda authorizers can approximate a gateway — but purpose-built gateways (Kong, Apigee, AWS API Gateway) are the right tool when you need the full feature set.

**Rule of thumb:** Route and distribute only → L7 LB. Govern traffic (who can call what, how often, in what shape) → API Gateway.

---

## API Gateway vs. Load Balancer

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

## Core Responsibilities

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

## Request Flow

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

## Key Trade-offs

**Single point of failure** — all traffic flows through the gateway, so it must be highly available. Fix: run multiple instances behind a load balancer (ironically, LBs protect the gateway).

**Added latency** — every request takes an extra hop + auth/rate-limit checks. Typical overhead: 1–5ms. Fix: keep auth logic fast (local JWT validation > remote token introspection), use connection pooling to backends, run gateway instances close to clients.

**Bottleneck at scale** — at millions of RPS, the gateway itself can be the ceiling. Fix: horizontal scaling, stateless design (no session state in the gateway), push rate-limit counters to Redis.

**Configuration complexity** — routing rules, auth policies, rate limits, and transformations can become a sprawling config mess. Fix: treat gateway config as code (Terraform, Pulumi), use declarative config (Kubernetes Gateway API, Kong declarative config).

---

## Usages

**Request Routing** — maps URL + method to the right backend service.
*Example: `GET /products/{id}` → product-service, `POST /orders` → order-service.*

**Security Enforcement** — validates auth tokens and permissions before the request touches any backend. One place to fix, covers all services.
*Example: invalid Bearer token → 401 at the gateway, backend never called.*

**Rate Limiting** — enforces per-client/per-tier quotas via Redis counters. Returns `429` when exceeded.
*Example: free tier → 100 req/min, pro → 10,000 req/min.*

**Service Aggregation** — fans out to multiple services in parallel, merges into one response, eliminating client round trips.
*Example: mobile dashboard fans out to user-service + order-service + recommendations in parallel → one JSON blob back to client.*

**Protocol & Format Translation** — bridges mismatches between client and service protocols/formats without changing either side.
*Example: client speaks REST/JSON, backend speaks gRPC — gateway translates transparently.*

**Caching** — serves repeated read requests from cache, bypassing the backend entirely.
*Example: product catalog cached at the gateway → read latency drops 10× on high-traffic pages.*

---

## Common Patterns

**BFF (Backend for Frontend)** — a separate gateway instance per client type (mobile, web, partner). Each BFF shapes responses for its client, avoiding the "one-size-fits-all" API problem.
*Example: Netflix has separate BFFs for TV, mobile, and browser — each aggregates different data and formats responses to match device constraints.*

**Gateway Aggregation** — fans out to multiple services in parallel, stitches results into one response, reducing client round trips.
*Example: dashboard needs user data + activity feed + billing status → one client request, three parallel backend calls, one merged response.*

**Gateway Offloading** — move cross-cutting concerns (auth, logging, SSL, compression) out of every microservice and into the gateway. Services stay focused on business logic.

---

## Advantages

**Centralized cross-cutting concerns** — auth, rate limiting, logging, and SSL handled once at the edge instead of re-implemented in every service.

**Simplified client integration** — clients talk to one stable endpoint regardless of how many services exist behind it or how they change.

**Observability** — complete traffic picture (latency, error rates, usage) in one place without instrumenting every service.

**Protocol flexibility** — clients and services can use different protocols/formats and evolve independently.

---

## Disadvantages

**SPOF** — all traffic flows through it; an outage takes down everything. Requires active-active clustering, which adds infra cost.

**Added latency** — extra hop + auth/rate-limit processing on every request. Typically 1–5ms, but it compounds under heavy transformation.

**Vendor lock-in** — managed gateways (AWS API Gateway, Apigee) tie you to that provider's pricing and limits.

**Configuration complexity** — routing rules, policies, and versioning accumulate fast. Without IaC it becomes a liability.

---

## Real-World: LB + API Gateway Together

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

## Interview Pitfalls

- Conflating API gateway and load balancer — know their distinct roles and that most systems use both.
- Ignoring HA for the gateway itself — "single entry point" sounds like a SPOF until you clarify it runs as a clustered, load-balanced tier.
- Overlooking latency impact — always mention the overhead of auth checks and when to prefer local JWT validation over remote introspection.
- Forgetting the BFF pattern — interviewers often probe whether you know that a single monolithic gateway doesn't serve all client types well.
- Not addressing security depth — gateway handles edge auth, but services should still validate internally (defense in depth).

---

## Recap

API gateway = smart traffic cop at the edge. Routes requests to the right service, enforces auth + rate limits, offloads cross-cutting concerns, and provides a single observability point — so microservices stay focused on business logic.
