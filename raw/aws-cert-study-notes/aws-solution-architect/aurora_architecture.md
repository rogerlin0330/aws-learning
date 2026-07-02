# Aurora Architecture

## Overview

**Amazon Aurora** is a managed database product from AWS. Officially it's part of RDS, but its features and the architecture it uses to deliver them are radically different from the other RDS engines — different enough that it's best treated as its own distinct product.

> [!NOTE] 📚 Provisioned vs. Serverless This lesson covers **Aurora Provisioned** — the mode where you choose and pay for specific compute instances (a primary + replicas). Many of the foundational concepts here (the cluster, the shared cluster volume, 6 storage copies across 3 AZs, endpoints, Backtrack, fast clones) are **shared with Aurora Serverless**, which is covered separately. The key difference: Serverless replaces the manually-provisioned instances with capacity that scales automatically (measured in ACUs), but it sits on the same underlying storage architecture. Where the transcript talks about _instances_ and _replicas_ you're provisioning and choosing failover targets for, that's specifically the Provisioned model.

Two things sit at the foundation of Aurora and separate it from normal RDS: the **cluster** and the **shared cluster volume**.

---

## The Cluster

Aurora's base entity is the **cluster** — something the other RDS engines don't have. A cluster is made up of:

- **Compute**: a single **primary instance** plus **zero or more replicas**.
- **Storage**: a **shared cluster volume** (covered below).

### Replicas are different from RDS

This _looks_ like RDS's primary + standby replica model, but it works very differently:

- 🎯 In RDS, the standby replica **cannot** be used during normal operation — it just sits there waiting for a failover.
- 🎯 In Aurora, **replicas can be used for reads during normal operation** _and_ serve as failover targets. A single Aurora replica delivers the benefits of **both RDS Multi-AZ (availability) and RDS Read Replicas (read scaling)** at the same time — you don't have to choose between the two.

### Failover

- Aurora supports **up to 15 replicas**, and **any** of them can be the failover target.
- Failover is **much faster** than RDS because there are **no storage modifications** to make — all instances already share the same storage (see below).

> [!TIP] 🎯 Contrast with non-Aurora RDS Non-Aurora engines: 1 primary + 1 standby (Multi-AZ instance mode), and separate read replicas. Aurora: 1 primary + up to 15 replicas that do _both_ jobs, with faster failover.

---

## Storage Architecture

![[aurora-storage-architecture.png]]

Aurora does **not** use local storage attached to each compute instance. Instead, the cluster has a single **shared cluster volume** available to **all** compute instances in the cluster.

**Reading the diagram (context → objects → footer):**

- **Context**: The cluster spans three availability zones — **AZ-A, AZ-B, AZ-C** (the three blue columns).
- **Main objects (compute, top)**: The **Primary** instance sits in AZ-A; **Replicas** sit in the other AZs. Blue up-arrows show every instance **reading** from the shared storage; the single red down-arrow from the primary shows that by default **only the primary writes**.
- **Main objects (storage, bottom)**: The red band is the **shared cluster volume** — a single logical volume. The SSD icons represent the **6 storage nodes** the data is replicated across. The yellow double-arrow indicates the volume is shared and spans all AZs.
- **Footer**: _"Cluster Volume: Max 128 TiB, 6 Replicas, AZs"_ — the headline storage facts.

### Key storage facts

- 🎯 **SSD-based** by default → high IOPS, low latency. **No magnetic option.**
- 🎯 **Maximum size: 128 TiB.**
- 🎯 Data is replicated across **6 storage nodes**, spread across **3 availability zones** (2 copies per AZ).
- When data is written to the primary, Aurora **synchronously replicates** it across all 6 storage nodes.
- 🎯 Replication happens at the **storage level**, so it consumes **no extra CPU/resources on the instances or replicas**. (This is fundamentally different from RDS, where replication is instance-driven.)
- By default the **primary is the only instance that can write**; the primary and all replicas can **read**.

### Storage resilience & self-healing

- Because Aurora keeps multiple copies across 3 AZs, the chance of data loss from disk failure is greatly minimized.
- Aurora **automatically detects** failures in the disks making up the cluster volume. When a segment fails, it **immediately repairs** that area using data from the other storage nodes and recreates it with no corruption.
- 🎯 This means Aurora largely **avoids the need for point-in-time or snapshot restores to recover from disk failures** — the storage subsystem is far more resilient than standard RDS storage.

### Replica agility

Because storage belongs to the **cluster** and not to individual instances, **replicas can be added or removed with no storage provisioning or removal**. This makes replica changes dramatically faster and more efficient.

---

## Storage Billing

- 🎯 **You don't allocate storage.** Unlike normal RDS, you never specify a storage amount when creating an Aurora cluster. Storage is billed **based on what you consume**, up to the 128 TiB limit.
- Metric: **GB-month consumed**.
- There is also an **I/O cost per request** to the cluster shared storage.

### ⚠️ High Watermark (transcript) vs. current AWS behavior

The transcript describes billing based on a **high watermark**:

> Bill = the _maximum_ storage ever consumed. Free up 10 GiB (50 → 40 GiB) and you're still billed for 50 GiB. You _can reuse_ freed space, but to actually **reduce** the bill you must create a **new cluster and migrate** the data. The instructor notes this is "being changed by AWS" and says to still assume the high-watermark model for the exam for now.

> [!WARNING] ⚠️ This is now out of date — verified against AWS docs (2026) AWS introduced **dynamic resizing** starting **September–November 2020**. The allocated (and billed) storage now **automatically decreases** when you physically remove data. Key current facts:
> 
> - Dynamic resizing applies to operations that **physically remove/resize** data: `DROP TABLE`, `DROP DATABASE`, `TRUNCATE TABLE`, and reorganizations — **not** plain `DELETE` of rows (for those, run `OPTIMIZE TABLE` on MySQL or `VACUUM` on PostgreSQL to reclaim).
> - Reclamation is **gradual** (up to ~**10 TB/day**), not instant.
> - For Aurora MySQL, set `innodb_file_per_table = 1` to benefit fully; space in the shared _system_ tablespace is never reclaimed.
> 
> **For the SAA exam:** the older **high-watermark** concept may still appear in questions and the "create a new cluster + migrate to shrink costs" answer may still be presented as correct. Know the transcript's version for the exam, but be aware real-world Aurora now shrinks dynamically.

### Backup billing

- 🎯 You get **free backup storage equal to 100% of the cluster's storage consumption**. (100 GiB cluster → 100 GiB of free backup allocation.)
- For most low/medium-usage workloads — unless you have high data turnover or long retention — **backup costs are effectively included** in what you already pay for the cluster.

### Compute billing

- **Hourly charge** per instance, billed **per second with a 10-minute minimum**.

### ⚠️ No free tier

- 🎯 Aurora has **no free tier**, because it **doesn't support micro instances**. For anything beyond an RDS single-AZ micro instance, though, Aurora generally offers **much better value**.

---

## Endpoints

![[aurora-endpoints.png]]

RDS uses endpoints (DNS addresses) to connect to a database. Aurora also uses endpoints, but unlike RDS it exposes **multiple** endpoints per cluster.

**Reading the diagram:** The **Cluster Endpoint** (pink) always points at the **Primary** in AZ-A. The **Reader Endpoint** (blue) fans out (dotted lines) across the **Replicas** in AZ-B and AZ-C — load-balancing reads across them. All instances sit above the shared **Cluster Volume**.

### Endpoint types

|Endpoint|Points at|Use for|
|---|---|---|
|🎯 **Cluster endpoint**|Always the **primary** instance|**Read _and_ write**|
|🎯 **Reader endpoint**|**Load-balances across all replicas** (or the primary if no replicas exist)|**Reads**|
|**Custom endpoint**|A user-defined subset of instances|Custom/tailored routing|
|**Instance endpoint**|One specific instance (primary or a given replica)|Direct access to a single instance|

- 🎯 Every cluster has, **at minimum**, a **cluster endpoint** and a **reader endpoint**.
- The reader endpoint **auto-updates** as replicas are added/removed, so read scaling is managed for you — no manual reconfiguration.
- Because of custom + per-instance endpoints, Aurora supports **much more complex/custom architectures** than RDS.

---

## Advanced Features

### Backups (baseline)

Automatic backups and manual snapshots work the **same as any other RDS engine**. **Restores create a brand-new cluster** (the default RDS behavior is unchanged).

### Backtrack

- 🎯 **Roll back an existing cluster _in place_** to a previous point in time — no new cluster, no reconfiguring applications.
- Use case: known-time data corruption. Instead of restoring to a new database, rewind the _same_ cluster to just before the corruption; apps keep using the same endpoints.
- Must be **enabled per-cluster**, and the backtrack window is adjustable.
- ⚠️ **Aurora MySQL only** (per current AWS docs — Backtrack is not available for Aurora PostgreSQL). The transcript notes it was Aurora-exclusive at recording time.

### Fast Clone

- 🎯 Creates a **new database from an existing one without copying all the storage**.
- The clone **references the original storage** and only stores **differences** — changes made in the clone, _or_ the pre-change version of data that's later modified in the source.
- Result: clones are created **very fast** and consume **only a tiny amount of storage** (just the deltas), rather than a full 1:1 copy.

---

## Exam Quick Reference

| Concept                     | Key fact                                                                           | Trigger keywords                                    |
| --------------------------- | ---------------------------------------------------------------------------------- | --------------------------------------------------- |
| Cluster                     | Primary + 0–15 replicas + shared volume                                            | "Aurora cluster"                                    |
| Replicas                    | Serve **reads + failover** simultaneously                                          | "read scaling AND availability", "no standby waste" |
| Failover targets            | **Up to 15**, any can be promoted, fast (no storage move)                          | "fast failover", "15 replicas"                      |
| Storage copies              | **6 copies across 3 AZs**                                                          | "6 copies", "three AZs"                             |
| Max storage                 | **128 TiB**                                                                        | "maximum Aurora storage"                            |
| Storage type                | SSD only, high-perf, no magnetic                                                   | "high IOPS by default"                              |
| Write/read                  | Primary writes; primary + replicas read                                            | "only primary writes"                               |
| Self-healing                | Auto-detect + auto-repair disk segments                                            | "no restore needed for disk failure"                |
| Storage billing             | Consumed GB-month; ⚠️ **high watermark** (transcript) / dynamic resizing (current) | "high water mark", "billed for max used"            |
| Backup allowance            | 100% of cluster storage free                                                       | "free backup storage"                               |
| Free tier                   | **None** (no micro instances)                                                      | "Aurora free tier" (→ not available)                |
| Compute billing             | Per-second, 10-min minimum                                                         | "10 minute minimum"                                 |
| Cluster endpoint            | Always primary; read + write                                                       | "read/write endpoint"                               |
| Reader endpoint             | Load-balances replicas; auto-updates                                               | "load balance reads", "scale reads"                 |
| Custom / instance endpoints | Subset / single instance                                                           | "custom endpoint"                                   |
| **Backtrack**               | In-place rewind to a point in time (MySQL only)                                    | "roll back", "rewind", "in-place", "corruption"     |
| **Fast Clone**              | New DB referencing source storage, stores only diffs                               | "clone", "copy-on-write", "minimal storage"         |