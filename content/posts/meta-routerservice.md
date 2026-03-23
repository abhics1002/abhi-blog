---
title: "ServiceRouter: How Meta Built a Service Mesh at Hyperscale"
date: 2026-03-22
draft: false
tags: ["servicediscovery", "meta"]
categories: ["Infrastructure"]
description: "ServiceRouter: How Meta Built a Service Mesh at Hyperscale"
---
*The learnings shared in this post are based on Meta's excellent paper, ["ServiceRouter: Hyperscale and Minimal Cost Service Mesh at Meta"](https://www.usenix.org/system/files/osdi23-saokar.pdf)

---

Most engineers work with service meshes as a given — you pick Istio or Linkerd, configure some sidecars, and move on. But if you've ever wondered what a service mesh looks like when you need to route **tens of billions of requests per second across millions of hosts spanning tens of thousands of services**, Meta's ServiceRouter is a masterclass in what that actually requires.

I spent time going deep on Meta's @Scale writeup of ServiceRouter, and the design decisions here are genuinely worth understanding — both because the problems are fascinating, and because the patterns scale down to problems you'll encounter at any large company.

Let's walk through the key ideas.

---

## Why ServiceRouter Exists

Meta's journey mirrors what many engineering orgs go through at scale. Initially, major services like Ads and News Feed built their own custom routing solutions optimized for their specific traffic patterns. Smaller services used a minimal interface with basic service discovery. Classic "every team solves it their own way" sprawl.

The obvious answer is to build a centralized routing library. But getting mature, opinionated teams to adopt a *new* shared library is always a hard sell — until you offer something they simply can't build themselves.

For ServiceRouter, that thing was **first-class observability for all RPC traffic**, out of the box.

The key insight: ServiceRouter wasn't built *on top of* the transport layer — it was integrated directly into Meta's primary RPC transport, Thrift. That means it has direct access to internal transport data that no wrapper library could ever see. Logging all RPC traffic with rich metadata became a feature that essentially sold itself. Once teams saw that visibility, adoption followed.

**Takeaway for staff+ engineers:** When you're trying to get adoption of a platform or library, find the *asymmetric* value — something teams genuinely can't build themselves. Convenience isn't enough; unique capability is.

---

## The Foundation: Decentralized Service Discovery

The canonical service mesh architecture has a central control plane (think Istio's istiod) that distributes configuration and routing information to data plane proxies. This works well at moderate scale. At Meta's scale, centralization becomes your bottleneck.

ServiceRouter flips this on its head. Services register themselves with a central Service Discovery System (the source of truth), but then a highly scalable **distribution layer** pushes routing information out to individual clients. Crucially, each client only requests the subset of routing data relevant to it and subscribes to updates for that specific subset.

The result: routing information is replicated across clients, not served from a central dispatcher on every request. The control plane scales horizontally because each client is largely self-sufficient.

This is the same philosophy behind systems like Zookeeper-based service discovery or Consul — but taken to an extreme where the client-side replication has to handle millions of subscribers.

**Why this matters:** Centralized routing decisions are a latency and reliability risk. Every request that requires a round trip to a central authority is a request that fails when that authority is slow or down. Pushing routing state to the edge makes the fast path entirely local.

---

## Load Balancing: More Nuanced Than You'd Think

Once you have service discovery, the next question is: *how do you pick which host to send a request to?*

ServiceRouter's answer is a multi-layer approach that's worth understanding in full.

### Step 1: Stable Host Selection via Rendezvous Hashing

A backend service might run on thousands of hosts. Rather than having every client maintain connections to all of them, each client selects a **stable subset** using a modified form of weighted rendezvous hashing.

The key property rendezvous hashing gives you: the subset remains stable even as the full host list changes (hosts added, removed, or fail). Only a minimal set of connections needs to be recalculated when the topology changes. Compare this to consistent hashing or random sampling, which can churn dramatically on membership changes.

Locality is baked into this selection — hosts in the same geographic region as the client are preferred, with expansion to farther regions only as a fallback.

### Step 2: Pick-2 Load Balancing

Once you have a stable subset of hosts, how do you pick one for each request? ServiceRouter uses **Power of Two Choices (pick-2)**, a well-studied algorithm from Michael Mitzenmacher's 1996 dissertation.

The idea is deceptively simple: instead of sending every request to the globally least-loaded host, randomly pick two hosts from your subset and route to whichever has lower load.

Why is this better than just picking the least-loaded host globally? The thundering herd problem. If every client always routes to the globally least-loaded host, that host gets hammered and immediately becomes the most-loaded host. Pick-2's randomness breaks the coordination that causes thundering herds, while still achieving load distribution far better than pure random selection. (Mathematically, pick-2 reduces the maximum load from O(log n / log log n) to O(log log n) — a dramatic improvement.)

### Step 3: Load Determination — Two Strategies

Pick-2 requires knowing the load on each of those two hosts. ServiceRouter offers two mechanisms, each with tradeoffs:

**Load Polling:** Before sending an RPC, proactively fetch the current load from each of the two candidate hosts, then route to the lower one. Fresh data, but you pay extra latency for the poll before every real request.

**Cached-Piggyback Load:** Servers piggyback their current load value on every response they send. Clients cache these values locally and use them for future routing decisions. Zero additional latency — you just read from cache. The downside: if the client isn't sending requests frequently, the cached load goes stale and you're making decisions on old data.

The selection between these two strategies used to be a manual service-by-service configuration. But this created operational overhead at scale — too many teams, too many services, too much configuration drift.

Meta's solution: **Automatic Load Balancing**, where the SR library itself monitors request rate and response latency in real time and dynamically switches the load-balancing strategy based on the current traffic profile. Engineers stopped needing to tune this, and load distribution across hosts measurably improved.

The selection matrix:

| | Low Request Rate | High Request Rate |
|---|---|---|
| **Low Latency Sensitivity** | Load Polling | Cached Load |
| **High Latency Sensitivity** | Random | Cached Load |

**Key insight:** When you're operating at enough scale, the cost of asking engineers to configure things correctly exceeds the cost of building the system to figure it out itself. Automatic decisions made from local data beat manual tuning.

---

## Connection Pooling: The Microsharding Problem

Establishing a new network connection for every RPC is expensive — especially over secure transport with per-connection authentication. Connection pooling (reusing established connections) is a standard optimization.

But pooling breaks down in a specific pattern: **microsharded services**.

Imagine a service split across 1000 shards. A client sending 20 requests per second to that service will send, on average, 0.02 RPS to each shard — one request every 50 seconds. The connection to each shard goes cold between requests. You end up paying connection establishment overhead on nearly every request, and you're maintaining thousands of half-warm connections simultaneously.

ServiceRouter's answer: a **lightweight proxy tier** sitting between clients and the sharded backend. Clients maintain a small, warm connection pool to the proxy. The proxy fans in traffic from many clients, amplifying the per-backend request rate enough to keep connections warm. The proxy itself is thin — it's essentially the SR library code running as a forwarding service.

This is a clean example of a general pattern: when the direct client-to-server model creates a resource problem, introduce an aggregation layer that solves the problem through fan-in. The key is keeping that layer stateless and lightweight enough that its overhead is clearly justified.

---

## Cross-Region Routing: Latency vs. Load

As Meta's infrastructure expanded across many geographic regions, a new problem emerged: when do you route cross-region?

In-region requests have lower network latency. But if the in-region fleet is overloaded, routing to a farther region might be better for the user. Neither "always stay in region" nor "always go to lowest-load region" is right in all cases.

ServiceRouter introduces **locality rings** — a way for services to declare their own latency/load tradeoff. Each geographic region is assigned to a ring relative to the client, where each ring represents a network latency threshold. A service can configure, for example: "Stay in region unless in-region load exceeds 70%, then expand to the next ring."

This lets services self-describe their priorities. A latency-sensitive, user-facing service stays local aggressively. A batch processing service might tolerate cross-region routing more readily.

Complementing this, Meta runs a **Cross-Region Load Balancer** — a centralized component that aggregates *global* traffic information and distributes it back to local SR clients as a routing table. Local clients use this table to guide their cross-region decisions.

The architectural subtlety here is deliberate: the global load information is collected in the background and then distributed to clients. Clients still make the actual routing decision locally. The Cross-Region Load Balancer gives you the *benefits* of a global view without making it a synchronous dependency in the request path. No single point of contention, no cascading failure if it's slow.

---

## Putting It All Together

Here's what strikes me most about ServiceRouter's design: almost every architectural decision is a specific answer to the question *"where is the bottleneck if we don't do this?"*

- Decentralized service discovery → removes central dispatch as a bottleneck at millions of clients
- Client-side host subset selection → removes the need for per-request central coordination
- Pick-2 with local load data → avoids thundering herd while keeping decisions local
- Automatic load balancing → removes human configuration as a scaling constraint  
- Proxy tier for microshards → solves cold connection problem via aggregation
- Locality rings + background global routing table → gives global awareness without synchronous centralization

The through-line is **push decisions to the edge, but make the edge smart**. Every routing decision that ServiceRouter can make locally, it does. Every piece of global state that's needed, it's distributed proactively rather than fetched on demand.

This architecture has been running in production since 2012. At some point, "it works at Meta scale" stops being an impressive footnote and starts being a proof of concept that's hard to argue with.

---

## What You Can Apply Today

Even if you're not routing tens of billions of RPCs per second, these patterns are worth internalizing:

1. **Rendezvous hashing for stable client-side selection** is available in most ecosystems and dramatically reduces connection churn on topology changes.

2. **Pick-2 load balancing** is implemented in many modern load balancers and client libraries. If you're using round-robin or random, it's worth understanding what you're giving up.

3. **The pick-2 load determination matrix** (polling vs. piggyback vs. random based on request rate × latency sensitivity) is a useful mental model even if you implement it manually per-service.

4. **Automatic policy selection from local telemetry** is the right direction for any shared infrastructure library. The more you can reduce per-service configuration burden, the better your operational posture.

5. **Background-distributed global state** is a clean pattern for giving local decision-makers global awareness without creating runtime dependencies on a central authority.
