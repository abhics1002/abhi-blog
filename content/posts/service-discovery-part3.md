---
title: "The xDS Protocol"
date: 2026-03-14
draft: false
tags: ["xDS", "Envoy", "service mesh",  "Istio", "control plane", "data plane", "proxyless mesh", "go-control-plane"]
categories: ["Infrastructure"]
description: "introduction to service discovery"
author: abhishek
---
*How a Lyft-born proxy API became the universal language of service networking*

---

Parts 1 and 2 gave us the vocabulary — registries, health checking, push vs pull, CAP trade-offs. Now we get to the protocol that redefined how all of that is delivered in modern infrastructure.

xDS is the reason Envoy exploded. It's why Istio, Consul Connect, Gloo, and a dozen other platforms all converged on the same data plane. It's why gRPC — a completely separate project from a different company — shipped native xDS support. And it's one of the more elegant pieces of distributed systems design you'll encounter in this space.

Let's dig in.

---

## 1. What Is xDS and Why Did It Emerge from Envoy?

xDS stands for "x Discovery Service" — a family of APIs for dynamically configuring a proxy. The "x" is literally a placeholder: LDS, RDS, CDS, EDS, SDS are all members of the xDS family. We'll cover each one shortly.

To understand why xDS exists, you need to understand the problem Envoy was trying to solve — and the state of proxy configuration before it showed up.

### The Old Way: Static Config Files

Before Envoy, the dominant proxy was NGINX (and HAProxy for load balancing). Both are excellent, battle-tested tools. But they share a fundamental constraint: configuration is file-based, and changes require a reload.

```nginx
# NGINX upstream block — static, file-based
upstream payment_backend {
    server 10.0.1.12:8080;
    server 10.0.2.44:8080;
    server 10.0.3.11:8080;  # still in here even if pod is dead
}

# To add/remove a backend, you:
#   1. Edit the config file
#   2. Run: nginx -s reload
#   3. NGINX does a graceful restart
#   4. New config takes effect
#
# In a Kubernetes cluster with 500 services and pods
# rescheduling every few minutes, this is untenable.
```

NGINX reload is fast (sub-second), but it's still a process-level event. In a dynamic cluster where the endpoint set is changing constantly, you're either reloading too often (CPU overhead, brief connection disruption) or accepting stale config between reloads.

What Lyft needed — and what they built Envoy to solve — was a proxy that could receive configuration updates continuously, at runtime, without any restarts or reloads. The mechanism they invented for this is xDS.

### Envoy's Design Principle: Everything is Dynamic

Envoy was designed from day one around the assumption that its configuration would change constantly. Instead of reading a file on startup, Envoy connects to a management server and subscribes to configuration resources. The management server streams updates. Envoy applies them live.

```
┌─────────────────────────────────────────────────────────────┐
│  OLD MODEL (NGINX / HAProxy)                                 │
│                                                              │
│   Config file ──► Proxy process                             │
│                   (reload required for any change)           │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  ENVOY / xDS MODEL                                           │
│                                                              │
│   Management    gRPC stream    Envoy                        │
│   Server    ─────────────────► Proxy                        │
│   (Istiod,      (continuous     (applies updates            │
│   Consul, etc)   push)           live, no restart)          │
└─────────────────────────────────────────────────────────────┘
```

This sounds simple, but the implications are profound. It means the control plane can react to infrastructure changes — a pod crash, a new deployment, a traffic policy update — and have every affected proxy updated within milliseconds. No file writes. No reloads. No restarts. The data plane is always in sync with the control plane's intent.

> **xDS turned proxy configuration from a deployment artifact into a streaming API.** That shift — from config as a file to config as a data stream — is what makes modern service meshes possible.

---

## 2. The xDS Resource Types: LDS, RDS, CDS, EDS, SDS

xDS isn't a single API — it's a family of discovery services, each responsible for a specific aspect of proxy configuration. Together they cover the complete lifecycle of a request through Envoy: how traffic arrives, how it's routed, where it goes, which endpoints receive it, and how it's secured.

Here's the full request flow:

```
Incoming Request
      │
      ▼
┌─────────────┐
│  LDS        │  Listener Discovery Service
│  Port 443   │  "What ports/protocols should I listen on?"
└──────┬──────┘
       │  matched to a FilterChain
       ▼
┌─────────────┐
│  RDS        │  Route Discovery Service
│  /payment/* │  "Where should this request be routed?"
└──────┬──────┘
       │  routed to a named cluster
       ▼
┌─────────────┐
│  CDS        │  Cluster Discovery Service
│  payment-v2 │  "What are the properties of this upstream cluster?"
└──────┬──────┘
       │  resolved to actual endpoints
       ▼
┌─────────────┐
│  EDS        │  Endpoint Discovery Service
│  10.0.1.12  │  "Which IPs/ports are healthy in this cluster?"
└──────┬──────┘
       │  connection secured by
       ▼
┌─────────────┐
│  SDS        │  Secret Discovery Service
│  TLS cert   │  "What certificates should I use for mTLS?"
└─────────────┘
```

### LDS — Listener Discovery Service

A Listener tells Envoy what to listen on and what to do with incoming connections. It defines the port, the protocol (HTTP/1.1, HTTP/2, TCP), and a chain of network filters that process the connection.

In an Istio setup, every sidecar Envoy typically has listeners on port 15001 (outbound traffic) and 15006 (inbound traffic), plus additional listeners for each port the service exposes. These are all configured via LDS.

```yaml
# LDS resource — simplified Envoy v3 API
name: outbound|8080||payment-svc.default.svc.cluster.local
address:
  socket_address:
    address: 0.0.0.0
    port_value: 8080
filter_chains:
  - filters:
    - name: envoy.filters.network.http_connection_manager
      typed_config:
        '@type': type.googleapis.com/...HttpConnectionManager
        stat_prefix: outbound_8080
        route_config_name: payment-rds   # ← pointer to RDS resource
        http_filters:
          - name: envoy.filters.http.router
```

Notice the `route_config_name` field — rather than embedding routing rules directly in the listener, it references an RDS resource by name. This indirection is intentional: it lets Envoy update routing rules independently of listener config, via RDS, without touching the LDS resource at all.

### RDS — Route Discovery Service

Routes are where the interesting traffic management happens. An RDS resource defines how HTTP requests get matched and forwarded — path prefixes, headers, methods, weights, retry policies, timeouts, and more. This is the layer that implements `VirtualService` semantics in Istio.

```yaml
# RDS resource — route config for payment-svc
name: payment-rds
virtual_hosts:
  - name: payment-svc
    domains: ["payment-svc", "payment-svc.default.svc.cluster.local"]
    routes:
      # Canary: 10% to v2
      - match: { prefix: "/" }
        route:
          weighted_clusters:
            clusters:
              - name: outbound|8080|v1|payment-svc.default.svc.cluster.local
                weight: 90
              - name: outbound|8080|v2|payment-svc.default.svc.cluster.local
                weight: 10
            total_weight: 100
          retry_policy:
            retry_on: 5xx,gateway-error
            num_retries: 3
            per_try_timeout: 2s
```

Because RDS is a separate resource from LDS, Istiod can push a routing update — say, changing the canary weight from 10% to 25% — without touching anything else. Envoy applies it live. Zero-downtime traffic shifts with a single API push.

### CDS — Cluster Discovery Service

In Envoy terminology, a "cluster" is a named group of upstream hosts that Envoy can forward traffic to. CDS defines the properties of each cluster — how load balancing works, circuit breaker settings, connection pool limits, and whether TLS is required upstream.

Crucially, a CDS resource doesn't list the actual IP addresses. It references an EDS resource (by cluster name) for the live endpoint list. Again, that separation lets each layer update independently.

```yaml
# CDS resource — cluster definition for payment-svc v2
name: outbound|8080|v2|payment-svc.default.svc.cluster.local
type: EDS                          # resolve endpoints via EDS
eds_cluster_config:
  service_name: payment-v2-eds     # ← pointer to EDS resource
connect_timeout: 10s
lb_policy: ROUND_ROBIN
circuit_breakers:
  thresholds:
    - max_connections: 1024
      max_pending_requests: 1024
      max_requests: 1024
      max_retries: 3
transport_socket:
  name: envoy.transport_sockets.tls
  typed_config:
    common_tls_context:
      tls_certificate_sds_secret_configs:
        - name: default               # ← fetch cert from SDS
```

### EDS — Endpoint Discovery Service

EDS is the piece most directly analogous to what a traditional service registry does — it's the live list of healthy IP:port pairs for each cluster. But unlike a registry that clients poll, EDS is pushed to Envoy by the control plane in real time.

EDS also carries locality information — which availability zone an endpoint is in — which Envoy uses for zone-aware load balancing. Traffic is preferentially kept within the same zone to reduce latency and cross-AZ data transfer costs.

```yaml
# EDS resource — live endpoints for payment-svc v2
cluster_name: payment-v2-eds
endpoints:
  # Zone: us-west-1a
  - locality:
      region: us-west-1
      zone: us-west-1a
    lb_endpoints:
      - endpoint:
          address:
            socket_address: { address: 10.0.1.12, port_value: 8080 }
        health_status: HEALTHY
      - endpoint:
          address:
            socket_address: { address: 10.0.2.44, port_value: 8080 }
        health_status: HEALTHY
  # Zone: us-west-1b
  - locality:
      region: us-west-1
      zone: us-west-1b
    lb_endpoints:
      - endpoint:
          address:
            socket_address: { address: 10.0.3.55, port_value: 8080 }
        health_status: UNHEALTHY   # ← Envoy won't route here
```

When a pod crashes, Istiod updates the EDS resource for the affected cluster and pushes it to all relevant Envoy sidecars. Those sidecars immediately stop routing to that endpoint — no TTL expiry, no polling interval. The update propagates in the time it takes for the gRPC push to complete.

### SDS — Secret Discovery Service

SDS is often the most underappreciated part of the xDS family, but it's what makes automatic mTLS operationally tractable at scale.

Without SDS, rotating TLS certificates means writing new cert files to disk and restarting (or reloading) Envoy. In a cluster with thousands of services, all running short-lived certificates (Istio defaults to 24 hours), this would be a continuous operational nightmare.

With SDS, the certificate authority (Istiod's built-in CA, or an external one via SPIFFE) pushes new certificates to Envoy via the same gRPC streaming mechanism as every other xDS resource. Envoy swaps them in live. No restarts. No file mounts. No cert rotation scripts.

```yaml
# SDS resource — a certificate + key pair delivered to Envoy
name: default
tls_certificate:
  certificate_chain:
    inline_bytes: "<base64-encoded cert chain>"
  private_key:
    inline_bytes: "<base64-encoded private key>"

# Combined validation context (trust bundle)
name: ROOTCA
validation_context:
  trusted_ca:
    inline_bytes: "<base64-encoded CA bundle>"

# Envoy uses these to:
#   - Present its own cert when acting as a TLS client (outbound)
#   - Verify peer certs when acting as a TLS server (inbound)
#   - Implement mutual TLS (mTLS) transparently for the app
```

---

## 3. How xDS Enables Dynamic Config Without Restarts

Now that we know what each resource type does, let's trace exactly what happens end-to-end when something in the cluster changes. This is where the elegance of the design becomes visible.

### The ADS: Aggregated Discovery Service

In practice, Envoy doesn't open five separate gRPC streams (one per discovery service). It opens a single **Aggregated Discovery Service (ADS)** stream to the control plane and multiplexes all resource types over it. This has two important benefits:

- **Ordering guarantees** — the control plane can ensure, for example, that a CDS update with a new cluster is delivered before the RDS update that references it, preventing a brief window of broken routing.
- **Connection efficiency** — one gRPC connection instead of five, with a single reconnection flow.

```
Envoy Sidecar                        Istiod (ADS server)
     │                                      │
     │── BiDi gRPC stream (ADS) ───────────►│
     │                                      │
     │── DiscoveryRequest {LDS} ───────────►│
     │◄─ DiscoveryResponse {listeners} ─────│
     │── ACK ────────────────────────────── │
     │                                      │
     │── DiscoveryRequest {CDS} ───────────►│
     │◄─ DiscoveryResponse {clusters} ───── │
     │── ACK ────────────────────────────── │
     │                                      │
     │── DiscoveryRequest {EDS} ───────────►│
     │◄─ DiscoveryResponse {endpoints} ─────│
     │── ACK ────────────────────────────── │
     │                                      │
     │          ... pod crashes ...         │
     │                                      │
     │◄─ EDS push (updated endpoints) ───── │  <~50ms
     │── ACK ────────────────────────────── │
     │   (dead endpoint removed from pool)  │
```

### The ACK/NACK Protocol

Notice the ACK messages in the diagram. xDS is not fire-and-forget. Every `DiscoveryResponse` from the control plane must be acknowledged by the client. This gives the control plane confidence that config updates are actually being applied, and enables meaningful error reporting when they're not.

If Envoy receives a config update it can't apply — malformed config, a referenced resource that doesn't exist, a cert it can't parse — it sends a NACK with an error message. The control plane can log this, alert on it, or retry with a corrected resource. This makes debugging config issues significantly easier than the alternative (silent failures).

```json
// xDS ACK — Envoy accepted the update
{
  "version_info": "2",
  "response_nonce": "abc123"
  // no error_detail field = ACK
}

// xDS NACK — Envoy rejected the update
{
  "version_info": "1",          // still on old version
  "response_nonce": "abc123",
  "error_detail": {
    "code": 3,                   // INVALID_ARGUMENT
    "message": "Referenced cluster 'payment-v3' not found"
  }
}
```

### SotW vs Delta xDS

The original xDS protocol used a **State of the World (SotW)** model: every `DiscoveryResponse` contains the complete current set of resources for that type. If you have 5,000 endpoints across 200 clusters, every EDS push sends all 5,000 endpoints — even if only one changed.

```
SotW xDS (original) — expensive at scale
─────────────────────────────────────────
1 endpoint changes
→ control plane sends ALL 5,000 endpoints to ALL proxies
→ at 1,000 proxies × 5,000 endpoints = 5M endpoint records sent
→ per update

Delta xDS (incremental) — efficient
─────────────────────────────────────────
1 endpoint changes
→ control plane sends ONLY that 1 changed endpoint
→ to ONLY the proxies that subscribe to that cluster
→ orders of magnitude less data
```

**Delta xDS** (stabilized in the Envoy v3 API) changes the protocol so each message carries only the diff — resources added, modified, or removed since the last ACK. At scale, this is not a minor optimization. It's the difference between a control plane that can handle 10,000 proxies and one that falls over under the write amplification.

```yaml
# Delta xDS DeltaDiscoveryRequest — Envoy subscribes to specific resources
{
  "type_url": "type.googleapis.com/envoy.config.endpoint.v3.ClusterLoadAssignment",
  "resource_names_subscribe": [
    "payment-v1-eds",
    "payment-v2-eds"
  ],
  "initial_resource_versions": {
    "payment-v1-eds": "version-42",   # Envoy tells server what it already has
    "payment-v2-eds": "version-17"
  }
}

# Server only sends what changed since those versions.
# Much less data. Much less CPU on both sides.
```

---

## 4. Why gRPC Adopted xDS Natively

Here's the thing that surprises a lot of engineers: gRPC — developed by Google, completely separate from the Envoy project — implemented native xDS support in its client libraries. Go, Java, C++, Python, and other gRPC language implementations can now use xDS for client-side load balancing, without a sidecar proxy at all.

Why would Google adopt a protocol that Lyft invented for their proxy? Because xDS solved a genuinely hard problem well, and reinventing it would have been wasteful.

### The Problem gRPC Was Trying to Solve

gRPC is an HTTP/2-based RPC framework. One of its design goals is efficient client-side load balancing — instead of routing through a load balancer, gRPC clients can maintain connection pools directly to backend instances.

This works great until you need to tell the client about backend changes. Early gRPC relied on DNS for service discovery — but as we covered in Part 1, DNS is slow to propagate and carries no health information. For long-lived gRPC streams in particular, DNS-based discovery is a poor fit.

What gRPC needed was exactly what xDS provides: a push-based mechanism to deliver live endpoint lists, load balancing config, and routing rules to the client library in real time.

```
gRPC WITHOUT xDS
────────────────────────────────────────────────────────
gRPC client                          DNS
    │── resolve payment-svc ────────►│
    │◄─ [10.0.1.12] ─────────────────│
    │   (stale after TTL)             │
    │   (no health info)              │
    │   (no routing rules)            │

gRPC WITH xDS (proxyless service mesh)
────────────────────────────────────────────────────────
gRPC client                          xDS Control Plane
    │── subscribe to payment-svc ───►│
    │◄─ EDS: [10.0.1.12, 10.0.2.44] ─│  live endpoints
    │◄─ CDS: lb=ROUND_ROBIN ──────────│  lb policy
    │◄─ RDS: retry on 5xx ────────────│  routing rules
    │                                 │
    │◄─ EDS push (pod removed) ───────│  real-time update
    │   (client stops sending there)  │
```

### Proxyless Service Mesh

When gRPC uses xDS directly — without a sidecar proxy — this is called a **proxyless service mesh**. The service mesh intelligence (load balancing, retries, traffic management) is embedded in the gRPC library itself, talking to the same xDS control plane that Envoy sidecars use.

```go
// gRPC Go: connect via xDS (proxyless service mesh)
// The xds:// scheme tells gRPC to use xDS for name resolution
conn, err := grpc.Dial(
    "xds:///payment-svc.default.svc.cluster.local",
    grpc.WithTransportCredentials(insecure.NewCredentials()),
)

// gRPC bootstraps xDS from a config file or env variable:
// GRPC_XDS_BOOTSTRAP=/etc/grpc/xds-bootstrap.json
```

```json
{
  "xds_servers": [{
    "server_uri": "istiod.istio-system.svc:15010",
    "channel_creds": [{"type": "google_default"}]
  }],
  "node": {
    "id": "payment-client~10.0.1.12",
    "cluster": "default"
  }
}
```

The appeal of the proxyless model is lower resource overhead — no sidecar container means less memory and CPU per pod. The trade-off is that you're limited to gRPC services (it only works where both client and server are gRPC), and observability is more fragmented since there's no central proxy collecting telemetry.

### xDS as an Open Standard

The gRPC adoption of xDS is significant beyond just the technical integration. It signals that xDS has graduated from being "Envoy's config API" to being a genuine open standard for service networking configuration.

Today, the xDS API surface is maintained by the CNCF's xDS API working group, with contributors from Google, Lyft, Microsoft, Solo.io, HashiCorp, and others. The proto definitions live in the `envoyproxy/data-plane-api` GitHub repository and are versioned independently from Envoy itself.

```
# The xDS v3 API proto definitions — referenced by both Envoy and gRPC
#
# github.com/envoyproxy/data-plane-api
#
# envoy/service/discovery/v3/ads.proto    ← ADS service definition
# envoy/service/endpoint/v3/eds.proto    ← EDS service definition
# envoy/config/listener/v3/listener.proto ← LDS resource type
# envoy/config/route/v3/route.proto      ← RDS resource type
# envoy/config/cluster/v3/cluster.proto  ← CDS resource type
# envoy/config/endpoint/v3/endpoint.proto ← EDS resource type
# envoy/extensions/transport_sockets/tls/v3/  ← SDS types
#
# Any control plane that implements these protos can serve
# any xDS-compatible data plane (Envoy, gRPC, etc.)
```

> **xDS is now the de facto standard interface between control planes and data planes in the service mesh ecosystem.** If you're building a new control plane or a new proxy, implementing xDS means you're immediately compatible with the entire ecosystem of tooling built around it.

---

## 5. Building Your Own xDS Control Plane (And When You'd Want To)

Most engineers will use xDS indirectly — through Istio, Consul, or another platform that implements it. But if you're building internal infrastructure tooling, or you need fine-grained control over proxy configuration that off-the-shelf control planes don't expose, you might find yourself writing one.

The Envoy project maintains `go-control-plane` — a Go library that handles the xDS server boilerplate (gRPC transport, resource versioning, delta protocol, ACK/NACK tracking) and lets you focus on building the logic that generates configuration from your service registry.

```go
// go-control-plane: minimal xDS server skeleton
package main

import (
    clusterservice  "github.com/envoyproxy/go-control-plane/envoy/service/cluster/v3"
    discoverygrpc   "github.com/envoyproxy/go-control-plane/envoy/service/discovery/v3"
    endpointservice "github.com/envoyproxy/go-control-plane/envoy/service/endpoint/v3"
    "github.com/envoyproxy/go-control-plane/pkg/cache/v3"
    "github.com/envoyproxy/go-control-plane/pkg/server/v3"
)

func main() {
    // SnapshotCache stores the current config for each Envoy node
    snapshotCache := cache.NewSnapshotCache(false, cache.IDHash{}, nil)

    // Create the xDS server
    xdsServer := server.NewServer(context.Background(), snapshotCache, nil)

    // Register gRPC services (one per xDS API)
    grpcServer := grpc.NewServer()
    discoverygrpc.RegisterAggregatedDiscoveryServiceServer(grpcServer, xdsServer)
    clusterservice.RegisterClusterDiscoveryServiceServer(grpcServer, xdsServer)
    endpointservice.RegisterEndpointDiscoveryServiceServer(grpcServer, xdsServer)

    // Your job: watch your service registry and call
    // snapshotCache.SetSnapshot() whenever something changes
    go watchRegistryAndPushUpdates(snapshotCache)

    grpcServer.Serve(listener)
}
```

The key insight: `go-control-plane` handles all the xDS protocol mechanics. Your job is to translate your service registry state into xDS resource objects and call `SetSnapshot()`. The library handles versioning, delta diffs, ACK tracking, and streaming to all connected Envoy instances.

When would you actually build this? The most common scenario is a hybrid infrastructure — Kubernetes plus VMs plus some legacy bare-metal — where no off-the-shelf control plane handles all your environments. You build a custom control plane that reads from all your service sources and emits a unified xDS config that Envoy instances everywhere can consume.

---

## Putting It All Together

xDS is a layered, composable protocol. Each resource type has a specific responsibility, they reference each other by name rather than embedding, and the whole thing is delivered over a single gRPC stream with explicit ACK/NACK semantics.

The key properties that make it powerful:

- **Dynamic by design.** Every resource type can be updated independently, without restarting the proxy. LDS, RDS, CDS, EDS, SDS — any of them can change while traffic is flowing.
- **Push-based.** Changes propagate in milliseconds, not polling intervals. The control plane knows immediately when a proxy has applied an update.
- **Incrementally efficient.** Delta xDS means only diffs are transmitted. A single pod crash doesn't cause a full config dump to every proxy in the cluster.
- **Open and portable.** Any control plane that speaks xDS works with any xDS-compatible data plane — Envoy, gRPC, and growing. It's not a proprietary API.
- **Composable.** LDS references RDS, RDS references CDS, CDS references EDS. Updates to one layer don't require touching others. This separation of concerns is what makes complex traffic management possible without dangerous big-bang config changes.

---

## What's Next

With the xDS protocol model firmly in mind, we're ready to look at how the major service discovery platforms implement it — and where they make different choices. Part 4 goes deep on Consul, Istio, Eureka, etcd, Zookeeper, and Nacos: how each one handles the registry, the health checking, the discovery pattern, and the trade-offs that make each one the right tool in some contexts and the wrong one in others.

**→ Continue to Part 4: Platform Deep Dives**

---
