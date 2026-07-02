# Aurora Multi-Master

## Overview

**Multi-master** is an advanced Aurora mode that allows a cluster to have **multiple instances capable of both reads AND writes**. This contrasts with Aurora's default mode, **single-master**, which has **one writer** and **zero or more read-only replicas**.

The key motivation is **faster/better availability** and being able to build **fault-tolerant applications** — there is no lengthy failover process because there are multiple writers already active.

> ⚠️ **CRITICAL — feature availability** Aurora Multi-Master was **only ever available on Aurora MySQL version 1 (MySQL 5.6-compatible)**, which reached **end of life on February 28, 2023**. It was **never** offered on Aurora MySQL v2/v3 or Aurora PostgreSQL. In practice this feature is now effectively retired. Adrian himself notes: _"It's not something I expect to immediately feature in detail on the exam, so we can keep it relatively brief."_ **Exam takeaway:** know the _concept_ (multiple read/write instances, no cluster endpoint, quorum writes, app-managed failover) rather than deep operational detail. `📚` Availability/EOL details verified against AWS docs (aurora-multi-master; Aurora MySQL 5.6 EOL page), beyond the transcript.

---

## Single-Master Recap (the default)

- **One read/write (primary) instance** + **0 or more read-only replicas**.
- **Endpoints** used to interact with the cluster:
    - **Cluster endpoint** → used for **read _or_ write** operations; always points at the **primary** instance.
    - **Reader endpoint** → **load balances reads** across the read-only replicas.
- 🎯 **Failover takes time.** A replica must be **promoted** from read-only → read/write. This is faster than standard RDS (shared cluster storage, more possible replicas) but it is **not instant** and **causes disruption**.

---

## Diagram Walkthrough

![[aurora-multi-master.png]]

**Context (top):** A user drives an **APPLICATION**. Notice the application connects **directly to the database instances** — there is no cluster endpoint in the middle. Both red arrows go straight from the app to the two instances.

**Main objects (blue DB icons):** Two Aurora **read/write** instances (both are writers). Between them runs a purple **"Replication (redo)"** arrow — a **bidirectional, peer-to-peer** link. This is how a committed change is pushed to the _other_ writer so it can update its **in-memory cache**.

**Storage layer (red band):** The shared **Cluster Volume** spans **AZ-A, AZ-B, AZ-C**. Orange arrows show each writer proposing/committing changes to the storage nodes across all AZs. The footer reads:

> **Cluster Volume: Max 128 TiB, 6 Replicas, AZs**

This describes the standard Aurora shared-storage architecture (six copies of data across three AZs). 🎯 The **128 TiB** max volume size and **6-way replication across 3 AZs** are the same shared-storage properties Aurora uses in single-master mode — multi-master reuses that storage design.

> ⚠️ The **128 TiB** figure appears in the course diagram. Current AWS docs list a higher Aurora max cluster volume size (now **up to 256 TiB** for recent engine versions). **Both may appear** — the older 128 TiB figure is common on the exam. `📚` (docs verification)

---

## How Multi-Master Works

### No cluster endpoint / no load balancing

- 🎯 In multi-master there is **no cluster endpoint** and **no reader endpoint** — i.e. **no load-balanced endpoint** at all.
- The **application** is responsible for connecting to instances and for **manually load balancing** across them. This is _not_ handled by the cluster.
- The app can connect to **one or all** instances and initiate operations **directly**.

### Write flow (quorum-based commit)

1. A read/write node receives a **write request** from the application.
2. It **proposes** the change be committed to **all storage nodes** in the cluster.
3. Each storage node **confirms or rejects** the proposed change.
    - It **rejects** if the change **conflicts with something already in flight** (e.g. another write from another instance touching the same data).
4. The writing instance looks for a **quorum** of nodes to agree.
    - **Quorum agrees** → change is **committed to shared storage** and replicated across every storage node (same as single-master).
    - **Quorum rejects** → the change is **cancelled** and an **error** is returned to the application.

### Cache coherence (the major difference)

- 🎯 After a committed write, the change is **replicated to the _other_ writer instance(s)** via the peer-to-peer "Replication (redo)" link.
- Each other writer updates its **in-memory cache**, so **any read from any instance is consistent** with what's on shared storage.
- **Why this matters:** because instances cache data, committing to disk alone isn't enough — caches on the other writers must also be updated so reads stay consistent.

---

## Single-Master Failover vs Multi-Master (the payoff)

**Single-master scenario:**

- App (Bob) connects via the **cluster endpoint**, which points at the **primary**.
- Primary **fails** → cluster access is **interrupted** → app is **not** fault tolerant.
- Cluster detects the failure and **repoints the cluster endpoint** to a replica chosen as the new primary — but this **configuration change takes time and causes disruption**.

**Multi-master scenario:**

- **Both** instances are writers; the app can **hold connections to both** at once.
- If one writer fails, the app can **immediately send 100% of operations** to the surviving writer.
- Result: **little or no disruption** — the app can be **designed to operate through the failure**.
- 🎯 Multi-master is a **foundation/component** for building a fault-tolerant application — **not an automatic guarantee**. The application must be designed to exploit it (maintain multiple connections, redirect on failure).

---

## Exam Traps & Gotchas

- ⚠️ **No cluster/reader endpoint in multi-master.** If a question implies Aurora auto-load-balances writes across multi-master instances — that's wrong. **Load balancing is the application's job.**
- ⚠️ **Fault tolerance is not automatic.** Multi-master _enables_ fault tolerance but requires **application logic** (multiple connections, failure detection, rerouting). "Multi-master alone = fault tolerant" is a distractor.
- ⚠️ **Writes are quorum-based with conflict rejection.** Conflicting concurrent writes get rejected → application must handle the error. Multi-master is **not** a magic conflict-free multi-writer.
- ⚠️ **Availability, not (primarily) scaling.** Multi-master's headline benefit is **faster availability / fault tolerance**, not write scale-out. (For write scaling patterns think sharding; AWS also positioned it for sharded/multi-tenant apps.) `📚`
- ⚠️ **Only Aurora MySQL 5.6 (v1), now EOL.** Don't assume it exists on Aurora PostgreSQL or modern Aurora MySQL. `📚`

---

## Exam Quick Reference

|Concept|Single-Master (default)|Multi-Master|
|---|---|---|
|Writers|**1** (primary)|**Multiple** (all instances)|
|Readers|0+ read-only replicas|All instances can read/write|
|**Cluster endpoint**|Yes (points at primary, R/W)|❌ **None**|
|**Reader endpoint**|Yes (load balances reads)|❌ **None**|
|Load balancing|Handled by cluster (reader endpoint)|**Application must do it manually**|
|Failover|Promote replica → R/W; **takes time, disrupts**|**No promotion needed**; app reroutes instantly|
|Fault tolerance|App not inherently fault tolerant|**Foundation** for FT app (needs app design)|
|Write commit|Standard|**Quorum** of storage nodes must agree; conflicts rejected|
|Cache updates|N/A|Peer-to-peer **redo replication** updates other writers' caches|
|Shared storage|Cluster volume (128 TiB, 6 replicas, 3 AZs)|**Same** cluster volume design|
|Availability|Aurora MySQL / PostgreSQL|**Aurora MySQL v1 (5.6) only — EOL 28 Feb 2023**|

### Trigger keywords

- "multiple read/write instances" / "multiple writers" → **Multi-Master**
- "no cluster endpoint" / "application load balances across instances" → **Multi-Master**
- "immediate failover / operate through writer failure / fault-tolerant DB tier" → **Multi-Master** (with app design)
- "single writer, many readers" / "reader endpoint" / "promote replica on failover" → **Single-Master** (default)
- "quorum agrees / rejects conflicting writes" → **Multi-Master** write flow