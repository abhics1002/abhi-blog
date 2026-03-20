---
title: "Relational Database Storage Engines — A Deep Dive"
date: 2026-03-19
draft: false
tags: ["storage", "lsm", "database"]
categories: ["Infrastructure"]
description: "introduction to database storage engine"
author: abhishek
---
# Relational Database Storage Engines 

The key insight going into this topic is that a "storage engine" is the component responsible for *how data is physically laid out on disk, how it's indexed, and how reads and writes are served*. Everything above it — SQL parsing, query planning, transactions — sits on top and is largely engine-agnostic. Understanding what's underneath these abstractions is what separates engineers who just accept defaults from those who can make informed architectural decisions.

---

## The Landscape at a Glance

| Need | Engine family | Why |
|------|--------------|-----|
| General OLTP, mixed read/write | B-Tree (InnoDB/PostgreSQL) | Balanced, good read latency, mature MVCC |
| Extreme write throughput, SSD wear reduction | LSM (MyRocks) | Sequential writes, ~2× space savings |
| Analytical queries, aggregations on wide tables | Columnar (Redshift, ClickHouse) | Column pruning, compression, vectorized execution |
| Sub-millisecond latency, temp/session data | In-memory (MEMORY, VoltDB) | No disk I/O at all |
| Time-series or append-only audit logs | Archive / LSM hybrid | Compression, no random update cost |
| Mixed OLTP+OLAP on same data | HTAP engines (TiDB, SingleStore) | Row + columnar in one engine via delta stores |

Let's go deep on each family.

---

## 1. B-Tree / Row-Oriented — The Dominant OLTP Family

This is what almost every relational database uses by default. Data is stored in **fixed-size pages** (typically 8KB or 16KB), organized as a B+ tree where leaf pages hold actual row data and internal nodes hold keys for navigation.

**InnoDB** (MySQL's default) and **PostgreSQL's heap engine** are the two most important implementations, but they differ significantly in how they handle updates.

The biggest architectural difference is that InnoDB uses a **clustered index** — the primary key B+ tree's leaf nodes *are* the rows. This means rows are physically sorted by primary key on disk. In PostgreSQL, data lives in unordered **heap files**, and every index (including the primary key index) is a separate structure pointing back to heap locations via a physical tuple ID (TID).

This has a real consequence in practice. In InnoDB, a range scan on the primary key (`WHERE id BETWEEN 100 AND 200`) is a single contiguous read. In PostgreSQL the same scan may jump around the heap, though index-only scans and HOT (Heap Only Tuples) updates partially mitigate this.

**MyISAM** is InnoDB's older MySQL sibling — table-level locking, no transactions, no foreign keys, but lighter weight and faster for pure read workloads. It's largely a legacy engine today.

---

## 2. LSM-Tree Engines — Write-Optimized

LSM trees flip the B-tree's write model entirely. Instead of performing in-place updates on disk pages (which requires random I/O), LSM trees convert all writes to sequential appends and reconcile the data during background compaction. The result is dramatically lower write amplification.

In the relational context the big player is **MyRocks** — Meta's MySQL storage engine built on RocksDB, now open source. It replaced InnoDB for a large portion of Meta's MySQL fleet.

Here's how the two compare side-by-side:

| | InnoDB (B-Tree) | MyRocks (LSM) |
|--|--|--|
| Write amplification | High — random seeks | Low — sequential appends |
| Read latency | Lower — one B-tree traversal | Higher — may check multiple levels |
| Space usage | Higher — fragmentation + free pages | Lower — compression-friendly, ~2× savings at Meta |
| MVCC | Undo log in rollback segments | Old versions in LSM levels |
| Compaction cost | No background compaction | Background CPU/IO spikes |

**TokuDB** (now open source as PerconaFT) took yet another approach — it used a **Fractal Tree** index, which is essentially B-tree shaped but with buffers at internal nodes to batch writes downward. This gives LSM-like write throughput while retaining B-tree read semantics.

---

## 3. Columnar Storage Engines — Built for Analytics

Traditional row engines store a full row contiguously on disk: `[id, name, age, salary, dept, ...]`. Columnar engines store each column separately: all `salary` values together, all `dept` values together, and so on.

The gains are twofold. First, you only read the columns you actually touch — a query like `SELECT AVG(salary)` doesn't need to load `name`, `age`, or `dept` at all. Second, since a column is a homogenous sequence of the same type, it compresses extraordinarily well through run-length encoding, dictionary encoding, and bit-packing. In practice you often see 5–10× compression ratios compared to row stores.

Notable columnar engines in the relational world include **Amazon Redshift** (PostgreSQL-derived), **ClickHouse MergeTree** (an LSM-flavored columnar engine), **DuckDB** (in-process analytical), and PostgreSQL's `cstore_fdw` extension. **Apache Arrow** is the in-memory columnar format that many of these share under the hood.

---

## 4. In-Memory Engines — When Disk Latency Is Unacceptable

When durability latency is unacceptable, you remove the disk entirely from the hot path. MySQL's **MEMORY** engine keeps all data in hash tables in RAM — extremely fast, but the table is gone on restart. It's mostly used for temp tables, lookup caches, and session state.

More serious in-memory relational engines like **VoltDB** and **SingleStore (MemSQL)** persist via a WAL that is replayed on startup (or via async replication to disk), so you get durability eventually without paying the synchronous write cost on every transaction. The tradeoff is that your working dataset must fit in memory, which makes capacity planning much more rigid.

---

## 5. MVCC — The Concurrency Mechanism Inside Every Engine

MVCC deserves its own section because it cuts across all engine families. Every production relational engine implements **Multi-Version Concurrency Control** — instead of locking readers out while a writer works, the engine maintains multiple versions of each row simultaneously. Readers always see a consistent snapshot of the database at the moment their transaction started, without blocking writers.

How each engine implements MVCC differs dramatically:

| Engine | Where old versions live |
|--|--|
| InnoDB | Undo log in rollback segments (separate tablespace) |
| PostgreSQL | Old tuple versions stay in the heap; VACUUM cleans up |
| SQL Server | Version store in `tempdb` |
| Oracle | Undo tablespace, similar to InnoDB |

PostgreSQL's approach — keeping dead tuples in the heap — is what makes `VACUUM` mandatory. Without it, tables bloat indefinitely with old row versions. InnoDB's undo log approach avoids heap bloat, but the rollback segments can grow large under long-running transactions.

---

## 6. Special-Purpose Engines Worth Knowing

**Archive (MySQL)** is a row-compressed, sequential-append-only engine. It supports no updates or deletes. It's used for audit logs and cold storage, achieving roughly 10:1 compression ratios.

**Spider (MySQL)** is a sharding engine that partitions tables across multiple MySQL instances and presents them as a single logical table. It's essentially a poor man's distributed SQL layer.

**Federated / FDW (PostgreSQL)**: Foreign Data Wrappers let PostgreSQL query external systems — other Postgres instances, MySQL, Redis, CSV files, REST APIs — as if they were local tables. Google's F1 does something architecturally similar, where its SQL engine can join Spanner data with Bigtable and CSV files in a single query.

---

## 7. HTAP — The Convergence of OLTP and OLAP

The **HTAP** (Hybrid Transactional/Analytical Processing) category is where a lot of current innovation is happening. Engines like TiDB, SingleStore, and YugabyteDB try to serve both OLTP and OLAP workloads without ETL-ing data into a separate analytical system.

The trick is usually maintaining both a row store (for OLTP) and a columnar replica (for analytics) in sync internally. TiDB, for example, uses its TiFlash component as a Raft learner that receives every write from the row-store layer and stores it in columnar format — the query optimizer then routes OLTP queries to the row store and analytical queries to TiFlash automatically, with no application changes required.

This is the same architectural challenge that Spanner and F1 address at Google scale, using read-only replicas to handle OLAP traffic while leader replicas serve OLTP.

---

*Understanding which storage engine is appropriate for your workload is one of the highest-leverage decisions in database architecture. The default engine is usually the right starting point, but knowing when you're hitting its structural limits — and what the alternatives trade off — is what makes the difference between a system that scales and one that fights you.*
