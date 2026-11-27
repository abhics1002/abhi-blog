---
title: "Engineering Distributed Systems at Scale"
description: "Practical engineering notes on distributed systems, cloud infrastructure, performance, and reliability."
---

I’m Abhishek, a software engineer focused on the infrastructure and systems that make large-scale services reliable. This is where I share detailed explorations of architecture, production behavior, and the engineering trade-offs behind them.

The goal is simple: explain not just **how a system works**, but **why it was designed that way, where it breaks, and what it takes to operate it at scale**.

## What I write about

- **Distributed systems** — service discovery, routing, consistency, and communication between services.
- **Cloud-native infrastructure** — Kubernetes, platform engineering, compute, and developer infrastructure.
- **Performance engineering** — latency, garbage collection, benchmarking, and capacity.
- **Reliability** — availability, failure modes, observability, and operating production systems.

## Start with these deep dives

### [Service Discovery in the Age of Microservices](/posts/service-discovery/)

A practical, multi-part exploration of how services find and communicate with each other — from static configuration to control planes and service meshes.

**Topics:** service discovery · Kubernetes · xDS · service mesh

### [When Garbage Collection Becomes an Availability Incident](/posts/gc-prod-availability/)

How allocation pressure and dataset growth can turn garbage collection into a production reliability problem, and how to reason about the issue across the stack.

**Topics:** performance · JVM · latency · availability

### [Relational Database Storage Engines — A Deep Dive](/posts/storage-engine/)

A look beneath SQL abstractions at how storage engines organize data and make read/write trade-offs.

**Topics:** databases · B-trees · LSM trees · storage

### [ServiceRouter: How Meta Built a Service Mesh at Hyperscale](/posts/meta-routerservice/)

Lessons from Meta’s approach to service routing at enormous scale, and the architectural ideas that apply beyond hyperscale environments.

**Topics:** service mesh · routing · distributed infrastructure

## How I approach engineering

I’m most interested in the boundary between architecture and production reality: bottlenecks that only appear under load, failure modes hidden by happy-path designs, and trade-offs between performance, reliability, and operational complexity.

I aim to make each article useful to engineers making real design decisions. Where possible, I include diagrams, concrete examples, measurements, and links to primary sources.

## Follow along

Browse [all posts](/posts/), explore the [topics](/tags/), or learn more [about me](/about/). You can also find my work on [GitHub](https://github.com/abhics1002) and [LinkedIn](https://www.linkedin.com/in/abbharadwaj/).
