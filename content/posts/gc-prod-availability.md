---
title: "When Garbage Collection Becomes an Availability Incident"
date: 2026-06-07
draft: false
tags: ["performance"]
categories: ["Infrastructure"]
description: "When Garbage Collection Becomes an Availability Incident"
author: abhishek
---


We tend to think of garbage collection as a background chore — the JVM quietly tidying up after itself while our service does the real work. Most of the time that's true. But push enough allocation pressure through a service, and GC stops being invisible plumbing and starts being the thing that decides whether your service stays up.

This is the story of how a legacy metadata service, i worked on went from "occasionally has a slow GC pause" to "dropping live write traffic under load" — and the month-long effort to fix it properly, at every layer from JVM flags down to the JDK version itself.

## The trigger: an 11x jump in dataset size, overnight

The service in question holds topology metadata for the infrastructure it serves — think of it as a discovery-adjacent inventory system that clients query to find out what exists and where. It had been running comfortably for a long time, holding on the order of tens of thousands of records per deployment region.

Then we backfilled a new class of data into it. Dataset size per region jumped roughly **11x** — from the ~17K records it had been designed around to over 200K.

That single change rippled through everything downstream.

The service's request-handling pattern generates a large number of short-lived objects per request — deserializing rows from a database, mapping them into internal domain objects, then mapping those into the objects it actually returns to clients. All of that is normally garbage almost as soon as it's created. At 17K records, the allocation volume from that pattern was manageable. At 200K+ records, especially for clients who queried a full dataset in one call, it wasn't.

## The failure mode: GC overload, then dropped traffic

A heap dump taken during this period showed that **93% of heap memory was garbage** — objects that were dead on arrival, just waiting for a collection cycle to reclaim them. The service's allocation rate was high enough that it filled the heap quickly and forced the JVM into very frequent GC cycles just to keep up.

Two things made this worse than ordinary GC pressure:

**Humongous objects.** In G1 (the collector in use here), any object at or above 50% of the configured region size is treated as a "humongous object" and allocated directly into the old generation — bypassing the young generation entirely. That forces a mixed GC collection, which takes roughly 4x longer than a young-gen-only collection. A handful of clients querying the entire dataset in a single call were reliably producing objects large enough to trigger this path.

**Overload protection kicking in.** The infrastructure has a host-level safeguard that watches for GC time exceeding a threshold and starts shedding requests from that host so it can recover. Under normal conditions this is a good thing — it protects a struggling host from making its own situation worse. But once GC pauses became frequent enough, this safeguard started triggering constantly, and it doesn't distinguish between reads and writes. **Write traffic was getting dropped**, which meant real failures on the client side, not just elevated latency.

What made this more than a one-line JVM flag fix was that it was a feedback loop cutting across every layer of the stack at once: more GC time meant less time serving requests, which meant more backlog, which meant more allocation pressure. An unoptimized query layer underneath the service kept feeding it oversized result sets. The service was quietly redoing the same allocation work for duplicate data on every request. A memory-heavy JSON serialization framework sat in the hot path. Page sizes on the (still unpaginated) APIs were unbounded. Underneath all of it, memory fragmentation was occasionally getting the process OOM-killed in a way that looked, at first glance, unrelated to GC entirely. And the service was still running an older JDK version, which meant real GC and latency improvements were sitting on the table, unclaimed. None of these were individually exotic — what made the incident real was that they all compounded at the same time, right as the data volume the service had to hold jumped by an order of magnitude.

{{< image src="/images/posts/gc-prod-availability/01_data_growth.png" alt="Chart showing dataset size jumping from 17K to 207K records" width="600" >}}

{{< image src="/images/posts/gc-prod-availability/05_humongous_object.png" alt="Diagram of humongous object allocation bypassing young generation and landing directly in old generation, forcing a mixed GC pass" width="700" >}}

## Diagnosis before treatment

Before changing anything, we needed to know exactly where the allocation was coming from. JFR (Java Flight Recorder) profiling and heap dump analysis pointed at a few consistent offenders:

- A JSON serialization library known to be memory-heavy, used across the mapping layer.
- A mapping step that converted raw database rows into domain objects, done freshly for every single record on every request — with no reuse even when the underlying data was identical.
- A second mapping layer that converted those domain objects into the wire-format objects returned to clients, which turned out to be its own significant allocation source.
- Elevated CPU and memory overhead from SSL/TLS termination.
- A framework's default HTTP header cache that was configured far larger than necessary — over 1GB, more than 40% of used heap at the time it was discovered.
- Widespread (and largely unnecessary) use of a boxed `Optional` type for values that were just as easily null-checked directly.

None of these individually was catastrophic. Together, at 11x the data volume, they added up to a service that couldn't keep its own GC in check.

## The fix: two tracks, run in parallel

### Short term: buy breathing room with JVM tuning

The fastest lever available was JVM configuration. Using an internal experimentation harness, we tried a range of heap sizes and GC parameters and landed on:

- Increasing heap size (both initial and max, kept equal to avoid resize pauses) by roughly 40%.
- Tuning survivor ratio, parallel GC thread count, and max pause time targets.
- Enabling `AlwaysPreTouch` so memory pages are committed at startup rather than causing page faults mid-request.
- Right-sizing the G1 region size for the object sizes actually in play.

This didn't fix the underlying allocation problem, but it bought real headroom while the deeper fixes were built.

### Long term: attack the allocation rate itself

The more durable fix was to reduce how much garbage the service produced per request in the first place:

1. **Removed the heavy JSON library from the mapping layer**, replacing it with a more memory-efficient serialization framework.
2. **Cached domain objects keyed by their natural identity.** A large fraction of records shared duplicate values for high-cardinality fields — the same string values appeared over and over across rows. Instead of constructing a fresh domain object for every duplicate, we cached and reused object references, so a given unique value was only ever materialized into an object once.
3. **Cached the second mapping layer too** — the domain-object-to-wire-object conversion — since object construction there was as expensive as the first mapping step. In one measured 15-minute window, this dropped allocation from a single hot method from 2.36 GB down to 805 MB.
4. **Fixed the oversized HTTP header cache**, eliminating over a gigabyte of unnecessary heap residency.
5. **Removed unnecessary `Optional` usage.**
6. **Switched the OS-level memory allocator.** The service was occasionally getting OOM-killed by the container runtime even though heap usage looked fine — the actual cause was memory fragmentation from a high allocation/deallocation churn rate. Switching allocators resolved it.
7. **Upgraded the JDK**, first as a minor-version bump that resolved a known SSL-related issue, and later as a full migration to a newer major version. The JDK upgrade alone measured out to double-digit percentage improvements in GC pause time and call latency (more on that below).

{{< image src="/images/posts/gc-prod-availability/04_allocation_reduction.png" alt="Chart showing cumulative allocation rate reduction across each optimization" width="650" >}}

## Results

Across a few months of rollout, the combined effect across both tracks was substantial:

- **GC overload trigger count dropped to zero.**
- **Dropped-traffic incidents dropped to zero** — the overload-protection safeguard stopped needing to shed requests.
- **GC health score during peak load improved from roughly 10 to over 90** (on a 0–100 internal scoring metric).
- Post-JDK-upgrade measurements alone showed **~24% reduction in total GC pause time**, **~12% reduction in max GC pause duration**, **~12% reduction in call latency (p90)**, and **~42% reduction in call latency (p99)** on the busiest instances.

{{< image src="/images/posts/gc-prod-availability/02_gc_overload_trend.png" alt="Chart showing GC overload trigger count trending to zero over the rollout period" width="650" >}}

{{< image src="/images/posts/gc-prod-availability/03_gc_score.png" alt="Chart showing GC health score improving from 10 to 92 during peak load" width="600" >}}

## Recommendations for GC improvements

If you're facing something similar — a service whose GC behavior has started to threaten availability rather than just latency — here's the order of operations that worked for us:

1. **Profile and heap-dump before you touch anything.** Don't guess where memory is going. JFR profiling and a heap dump will tell you, concretely, which allocation sites and object types dominate — and that list is rarely what intuition predicts.
2. **Buy time with JVM tuning first, but don't stop there.** Heap sizing, survivor ratios, and pause-time targets are the fastest lever you have. Use them to stabilize the service while you work the real fix — but treat them as a stopgap, not a solution.
3. **Treat allocation rate as the metric to reduce, not GC pause time.** Pause time is a symptom. If you only chase pause time, you'll keep tuning the collector forever. Reducing how much garbage you produce per request is what actually moves the needle.
4. **Cache aggressively wherever the same logical data gets re-materialized into objects.** If duplicate values are getting turned into new objects on every request, that's close to free memory to reclaim — cache by natural identity and reuse references.
5. **Audit your serialization layer.** JSON libraries and mapping frameworks vary enormously in memory efficiency. If serialization or object mapping shows up high in your allocation profile, it's worth benchmarking alternatives.
6. **Know your collector's edge cases.** For G1 specifically, understand your region size and what counts as a "humongous" object for your workload — objects that bypass young-gen collection entirely are a common, underappreciated source of long pauses.
7. **Check your allocator, not just your heap, when you see unexplained OOM kills.** High allocation/deallocation churn can fragment memory in ways that a heap that "looks fine" won't reveal. An OS-level allocator switch can be worth trying before you assume it's a leak.
8. **Don't skip the JDK upgrade.** Newer JDK versions often bring real GC and runtime improvements for free. If you're several major versions behind, that gap alone may be leaving double-digit percentage gains on the table.
9. **Fix the code hygiene issues while you're in there.** Things like unnecessary boxed-type usage or oversized framework defaults (cache sizes, buffer sizes) rarely cause an incident on their own, but they add up, and they're usually cheap to fix once you've found them.
10. **Land the fixes together, not in isolation.** JVM tuning, code-level allocation reduction, and a JDK upgrade compound. Measuring each one in isolation will understate how much headroom you actually get when they land as a set.

When a service's job is fundamentally to hold and hand out a lot of data quickly, GC isn't a background concern you tune once and forget. It's a production availability property, and it deserves to be treated like one.

