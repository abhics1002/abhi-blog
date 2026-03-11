---
title: "Service Discovery in the Age of Microservices"
date: 2026-03-10
draft: false
tags: ["service discovery", "distributed-systems", "cloud-native", "xDS", "platform engineering", "CAP theorem"]
categories: ["Infrastructure"]
description: "introduction to service discovery"
---

> **SERVICE DISCOVERY PLATFORMS — PART 2 OF 6**

*Registries, health checks, client vs server-side discovery, CAP theorem, and push vs pull — demystified*

---

In Part 1 we covered the "why" — why static config breaks down, how the industry evolved from properties files to service meshes, and why getting this right matters more as your system grows.

Part 2 is about the "what". Before you can evaluate platforms like Consul, Istio, or Eureka meaningfully, you need a solid grip on the underlying concepts they're all built on. These aren't abstract computer science ideas — they're the specific design decisions that determine how a system behaves when a pod crashes at 3am.

Let's build the mental model.

---

## 1. The Service Registry

The service registry is the heart of any discovery system. It's a database — but not like your application database. It has a very specific job: maintain an accurate, up-to-date map of every service instance that's currently alive and ready to receive traffic.

Think of it like air traffic control. Planes (services) check in when they enter airspace, broadcast their position continuously, and check out when they land. The control tower (registry) has a live view of everything in the sky. Without it, everyone's flying blind.

```
┌──────────────────────────────────────────────────────────────┐
│                     Service Registry                          │
│                                                               │
│  Service        Instances              Health    Metadata     │
│  ─────────────────────────────────────────────────────────── │
│  payment-svc    10.0.1.12:8080         ✓ UP      v2, us-w1   │
│                 10.0.2.44:8080         ✓ UP      v2, us-w1   │
│                 10.0.3.11:8080         ✗ CRIT    v1, us-w1   │
│                                                               │
│  checkout-svc   10.0.4.23:3000         ✓ UP      v5, us-w1   │
│                 10.0.1.87:3000         ✓ UP      v5, us-w1   │
│                                                               │
│  fraud-svc      10.0.1.55:9090         ✓ UP      v3, us-w1   │
└──────────────────────────────────────────────────────────────┘
```

The registry needs to answer a few questions reliably:

- Is this service currently registered? (existence check)
- Which instances of this service are healthy right now? (health-filtered lookup)
- What's the address and port for each instance? (endpoint resolution)
- Any metadata I should know about — version, zone, weight? (rich lookup)

That last point matters more than it sounds. Modern registries store rich metadata alongside each registration. This is what enables traffic shaping — "only send requests to v2 pods" or "prefer instances in the same availability zone" — without any changes to the application code.

### Self-Registration vs Third-Party Registration

There are two schools of thought on how services get into the registry:

#### Self-Registration

The service itself is responsible for registering on startup and deregistering on shutdown. Eureka and Consul both support this model. It's simple — the service knows its own address and metadata better than anyone — but it couples your application code to the registry client library.

```bash
# Self-registration: service calls the registry directly on boot
# Consul HTTP API
PUT /v1/agent/service/register
{
  "ID":      "payment-svc-10.0.1.12",
  "Name":    "payment-svc",
  "Address": "10.0.1.12",
  "Port":    8080,
  "Tags":    ["v2", "us-west-1"],
  "Check": {
    "HTTP":     "http://10.0.1.12:8080/health",
    "Interval": "10s",
    "Timeout":  "2s"
  }
}
```

#### Third-Party Registration

An external system — a deployment platform, an orchestrator, a sidecar — registers and deregisters services on their behalf. Kubernetes uses this model: when a pod starts, the kubelet and controller manager update the Endpoints object automatically. The application has zero awareness of any registry.

```bash
# Kubernetes does third-party registration automatically.
# When pods matching a Service selector come up/go down,
# the Endpoints object is updated by the control plane — no app code needed.

$ kubectl get endpoints payment-svc
NAME          ENDPOINTS                           AGE
payment-svc   10.0.1.12:8080,10.0.2.44:8080      14m

# One pod crashes — control plane removes it within seconds:
payment-svc   10.0.2.44:8080                     14m
```

Third-party registration is cleaner architecturally — your service code stays ignorant of infrastructure concerns — but it requires a platform with the smarts to do it correctly, including detecting crashes and deregistering promptly.

---

## 2. Health Checking: The Difference Between a Registry and a Graveyard

A service registry without health checking is just a list of addresses that may or may not work. Health checking is what keeps the registry honest — continuously probing instances and removing unhealthy ones from rotation before clients get burned by them.

There are three main approaches, and most mature platforms support all three:

### Active Health Checks (Server-Side Probing)

The registry itself sends periodic probes to each registered instance. If an instance fails N consecutive checks, it's marked unhealthy and removed from the live set. This is Consul's default model.

```
                                every 10s
┌─────────────┐   HTTP GET /health ──────►  ┌──────────────┐
│   Consul    │                              │ payment-svc  │
│  (registry) │   ◄── 200 OK ───────────    │  instance A  │
└─────────────┘                              └──────────────┘
      │
      │          HTTP GET /health ──────►  ┌──────────────┐
      └──────────                          │ payment-svc  │
                 ◄── timeout (3x) ──────   │  instance B  │  ✗
                                           └──────────────┘
                 → Mark instance B critical
                 → Remove from DNS / API responses
```

```yaml
# Consul health check config (in service definition)
checks:
  - id: payment-http
    name: HTTP health on port 8080
    http: http://localhost:8080/healthz
    interval: 10s
    timeout: 2s
    deregister_critical_service_after: 30s  # auto-remove if down >30s
```

### Heartbeat / TTL Checks

The service itself is responsible for periodically reporting to the registry that it's still alive. If the registry doesn't hear from a service within the TTL window, it marks it unhealthy. Eureka uses this model — each instance sends a heartbeat every 30 seconds, and if three consecutive heartbeats are missed, the instance is evicted.

```bash
# Eureka heartbeat — sent automatically by the client library every 30s
PUT /eureka/apps/PAYMENT-SERVICE/10.0.1.12

# If missed 3x (90 seconds), Eureka evicts the instance.
# This is why Eureka has an 'emergency mode' that kicks in
# during network partitions — it stops evicting to avoid
# mass-removing instances that are actually still running.
```

### Client-Side Health Checks

Rather than the registry probing services, the calling client tracks failures itself — using circuit breakers or outlier detection. Envoy's outlier detection works this way: if a backend returns 5xx errors above a threshold, Envoy ejects it from the load balancing pool for a cooldown period.

```yaml
# Istio DestinationRule: configure Envoy outlier detection
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: payment-svc
spec:
  host: payment-svc
  trafficPolicy:
    outlierDetection:
      consecutive5xxErrors: 3      # eject after 3 consecutive 5xx
      interval: 10s               # check every 10 seconds
      baseEjectionTime: 30s       # eject for at least 30 seconds
      maxEjectionPercent: 50      # never eject more than 50% of pool
```

> **A good rule of thumb:** use server-side active checks for coarse-grained liveness ("is the process up?"), and client-side outlier detection for fine-grained quality signals ("is this instance returning errors?"). They complement each other.

---

## 3. Client-Side vs Server-Side Discovery

Once you have a registry with live, healthy endpoints — how does the calling service actually use it? There are two fundamentally different patterns, and they have different implications for where intelligence lives in your system.

### Client-Side Discovery

The client queries the registry directly, gets back a list of healthy instances, and then decides which one to call — running its own load balancing logic. This is Eureka's model, and it's what Spring Cloud's Ribbon implemented.

```
┌──────────────────────────────────────────────────────────────┐
│  CLIENT-SIDE DISCOVERY                                        │
│                                                               │
│  checkout-svc                                                 │
│  ┌──────────────────────────────────────────┐                │
│  │  App Code                                 │                │
│  │    │                                      │                │
│  │    ├─1─► Query registry for payment-svc  │                │
│  │    │      [10.0.1.12, 10.0.2.44]         │                │
│  │    │                                      │                │
│  │    ├─2─► Pick instance (round-robin)      │                │
│  │    │      → 10.0.1.12:8080               │                │
│  │    │                                      │                │
│  │    └─3─► Call 10.0.1.12:8080 directly    │                │
│  └──────────────────────────────────────────┘                │
└──────────────────────────────────────────────────────────────┘
```

**Upside:** the client has full visibility and control over routing. It can implement sophisticated strategies — zone-aware routing, sticky sessions, latency-weighted balancing — without any infrastructure changes.

**Downside:** every service needs to embed this logic, usually via a shared library. That library becomes a critical dependency. Upgrading it across 200 services is a painful coordinated effort. And if you have services in multiple languages, you need to implement (and maintain) this logic in each one.

### Server-Side Discovery

The client calls a stable virtual address — an IP, a DNS name, a load balancer. Something else (a proxy, a load balancer, a service mesh sidecar) handles the registry lookup and instance selection transparently. The client knows nothing about the actual backend instances.

```
┌──────────────────────────────────────────────────────────────┐
│  SERVER-SIDE DISCOVERY                                        │
│                                                               │
│  checkout-svc                                                 │
│  ┌───────────────┐    ┌──────────────┐   ┌────────────────┐  │
│  │   App Code    │    │  Envoy       │   │    Registry    │  │
│  │               │    │  (sidecar)   │   │                │  │
│  │  call         │    │              │   │                │  │
│  │  payment-svc ─┼──► │ intercepts   │   │                │  │
│  │               │    │ → looks up   ├──►│ [healthy IPs]  │  │
│  │               │    │ → picks IP   │◄──┤                │  │
│  │               │    │ → forwards   │   │                │  │
│  └───────────────┘    └──────────────┘   └────────────────┘  │
│          App has zero knowledge of actual backend IPs         │
└──────────────────────────────────────────────────────────────┘
```

Kubernetes Services + kube-proxy is server-side discovery. Istio with Envoy sidecars is server-side discovery. The app just calls "payment-svc" and the infrastructure handles the rest.

**Upside:** zero library dependency in application code. Language-agnostic. Upgrading routing logic is an infrastructure-level operation, not an application-level one. Observability, retries, and circuit breaking can be added to the proxy layer without touching app code.

**Downside:** you now have an extra network hop (the proxy). And debugging gets harder — when something goes wrong, is it the app, the proxy, or the registry? The abstraction that makes it easy to use also makes it harder to reason about.

### Which Should You Use?

|  | Client-Side | Server-Side |
|---|---|---|
| **App awareness of infra** | High | None |
| **Language support** | Library per language | Universal (proxy) |
| **Routing flexibility** | Very high | High (proxy config) |
| **Operational complexity** | Low infra, high app | Higher infra, low app |
| **Observability** | Per-library | Centralized in proxy |
| **Examples** | Eureka + Ribbon, Nacos | kube-proxy, Envoy, Istio |

The industry has been moving toward server-side discovery — specifically the service mesh model — as the default for greenfield microservices. But client-side discovery is still perfectly valid, especially in Java-heavy shops where Spring Cloud makes it trivially easy and you don't want to operate a service mesh.

---

## 4. Push vs Pull: How Config Gets to the Client

Here's a question that doesn't get asked enough: when the registry changes — a new instance comes up, a pod crashes, a config update is applied — how does that information actually reach the clients?

### Pull (Polling)

Clients periodically ask the registry: "what's changed?" The registry responds with the current state. Clients cache the result and use it until the next poll.

```
Client                          Registry
  │                                │
  │── GET /instances/payment ─────►│
  │◄─ [10.0.1.12, 10.0.2.44] ─────│   t=0
  │   (client caches this)         │
  │                                │
  │   ... 30 seconds pass ...      │
  │   ... pod 10.0.1.12 crashes .. │
  │   ... registry removes it  ... │
  │                                │
  │── GET /instances/payment ─────►│
  │◄─ [10.0.2.44] ─────────────────│   t=30s
  │   (stale for up to 30s!)       │
```

Eureka uses polling. The default interval is 30 seconds, which means there's a 30-second window where a client can be routing to a dead instance. In practice this is mitigated by client-side retry logic, but it's a real latency in the system's ability to react to failures.

```yaml
# Eureka client config — tune the polling interval
eureka:
  client:
    registry-fetch-interval-seconds: 30  # default, lower = more load on server
    cache-refresh-executor-thread-pool-size: 2
```

### Push (Streaming)

The registry maintains a persistent connection to each client and pushes updates as they happen. When an instance goes down, the registry immediately pushes the updated state to all connected clients. This is what xDS does — each Envoy sidecar holds a long-lived gRPC stream to the control plane.

```
Envoy Sidecar                   Istiod (control plane)
  │                                │
  │── open gRPC stream ───────────►│
  │◄─ initial config push ─────────│   connection established
  │                                │
  │   ... pod 10.0.1.12 crashes .. │
  │   ... Istiod detects change ... │
  │◄─ EDS update pushed ────────────│   ~milliseconds later
  │   (remove 10.0.1.12 from pool) │
  │── ACK ──────────────────────── │
  │                                │
  │   client never polled,         │
  │   update arrived immediately   │
```

Push is faster and more efficient at scale — no thundering herd of polling clients hammering the registry. But it's more complex to implement correctly, especially around reconnection logic, back-pressure, and what happens when the control plane restarts.

### Delta xDS: The Best of Both

Early xDS implementations pushed the entire state on every update — if one endpoint changed, every Envoy got a full config dump. At scale with thousands of services, this becomes a serious bandwidth and CPU problem.

Delta xDS (incremental xDS) addresses this by only pushing what actually changed. It's now the recommended xDS mode and what production Istio installations use.

```yaml
# Delta xDS: only the diff is sent, not the full state
#
# State of the world (SotW) xDS — bad at scale:
#   1 endpoint changes → push all 10,000 endpoints to all proxies
#
# Delta xDS — efficient:
#   1 endpoint changes → push just that 1 change to affected proxies

# Envoy bootstrap config to enable delta xDS
dynamic_resources:
  ads_config:
    api_type: DELTA_GRPC   # vs GRPC for state-of-the-world
    transport_api_version: V3
    grpc_services:
      - envoy_grpc:
          cluster_name: xds_cluster
```

---

## 5. CAP Theorem: The Trade-off You Can't Escape

Every distributed system that stores state has to make a choice when network partitions happen — and they will happen. The CAP theorem says you can only guarantee two of three properties:

- **Consistency (C)** — every read gets the most recent write or an error
- **Availability (A)** — every request gets a response, though it might not be the latest data
- **Partition Tolerance (P)** — the system keeps working when network partitions occur

Since network partitions are a fact of life in distributed systems, you're really choosing between CP and AP. And this choice has direct consequences for how a service registry behaves under failure.

### AP Systems: Eureka

Eureka chooses availability. If the Eureka server loses contact with some instances due to a network partition, it doesn't remove them — it assumes they might still be alive and continues serving the last known data. This is called Eureka's "self-preservation mode".

```bash
# Eureka self-preservation — what you see in the dashboard during a partition:
#
# EMERGENCY! EUREKA MAY BE INCORRECTLY CLAIMING INSTANCES ARE UP WHEN
# THEY'RE NOT. RENEWALS ARE LESSER THAN THRESHOLD AND HENCE THE
# INSTANCES ARE NOT BEING EXPIRED JUST TO BE SAFE.
#
# This is intentional: Eureka prefers stale data over no data.
# Your clients might hit dead instances, but they won't get
# 'no instances found' errors during a partial outage.

# You can tune this behavior:
eureka:
  server:
    enable-self-preservation: true   # default
    renewal-percent-threshold: 0.85  # trigger if <85% of expected heartbeats
```

For Netflix's consumer-facing workloads — where hitting a slightly stale backend is far better than returning an error to millions of users — this was exactly the right call.

### CP Systems: etcd, Zookeeper

etcd (which backs Kubernetes) and Zookeeper choose consistency. If a network partition means a node can't confirm it has the latest data, it will refuse to answer rather than return potentially stale information.

```bash
# etcd uses the Raft consensus algorithm.
# A write is only committed when a quorum of nodes agrees.
#
# 3-node cluster: needs 2 nodes to agree (quorum = n/2 + 1)
# 5-node cluster: needs 3 nodes to agree
#
# During a partition that splits (2, 3) nodes:
#   minority partition (2 nodes): stops accepting writes
#   majority partition (3 nodes): continues normally
#
# This means: during a partition, some clients see an unavailable
# etcd, but the data is always consistent.

$ etcdctl endpoint health
https://10.0.1.10:2379 is healthy: successfully committed proposal
https://10.0.1.11:2379 is healthy: successfully committed proposal
https://10.0.1.12:2379 is unhealthy: failed to commit proposal  # partitioned
```

> **CP vs AP isn't about which is better** — it's about which failure mode you'd rather have. Would you prefer to sometimes route to a dead instance (AP), or would you prefer requests to fail cleanly when the registry is uncertain (CP)? Know your answer before you pick a platform.

### Where Does Consul Land?

Consul is interesting because it supports both modes. By default its service catalog is AP (like Eureka) — it prioritizes availability for service lookups. But its KV store operations can be configured to use strong consistency via Raft (like etcd). This flexibility is part of why Consul has such broad adoption across different use cases.

---

## 6. Putting It All Together

These five concepts — registries, health checking, client vs server-side discovery, push vs pull, and CAP theorem trade-offs — don't exist in isolation. Every platform in this space is a specific combination of choices across all five dimensions.

Here's how the major platforms stack up:

| Platform | Registry Type | Health Check | Discovery Pattern | Propagation | CAP |
|---|---|---|---|---|---|
| **Eureka** | Self-reg | Heartbeat (TTL) | Client-side | Pull (30s) | AP |
| **Consul** | Self or 3rd party | Active HTTP/TCP | Both | Pull / Push | AP+CP |
| **etcd** | 3rd party (K8s) | K8s controller | Server-side | Watch (push) | CP |
| **Zookeeper** | Self-reg | Ephemeral nodes | Client-side | Watch (push) | CP |
| **Istio/xDS** | 3rd party (K8s) | Active + Envoy OD | Server-side | Push (gRPC) | AP |
| **Nacos** | Self-reg | Heartbeat + Active | Both | Pull / Push | AP+CP |

Notice that no platform is best across every dimension. Eureka's 30-second pull interval makes it the slowest to react to failures — but it's also the simplest to operate and the most resilient during control-plane outages. Istio's push model is the most responsive, but it requires running a full service mesh with all the operational overhead that entails.

The goal of this series isn't to tell you which one to use. It's to give you the vocabulary to make that call yourself, with eyes open to the trade-offs.

---

## What's Next

You now have the conceptual vocabulary to reason about any service discovery platform. In Part 3, we zoom in on one specific piece of this puzzle that deserves its own treatment: the xDS protocol.

xDS is the reason Envoy became the de facto data plane for service meshes, and why platforms as different as Istio, Consul Connect, and even gRPC's client-side load balancing all converged on the same API. Understanding it is key to understanding the modern service mesh landscape — and it's a surprisingly elegant piece of systems design once you see how the pieces fit together.

**→ Continue to Part 3: The xDS Protocol**

---
