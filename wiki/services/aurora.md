---
title: Aurora
type: service
tags: [aurora, database, relational, rds, dms, cluster, cluster-volume, shared-storage, replicas, failover, cluster-endpoint, reader-endpoint, custom-endpoint, instance-endpoint, storage-billing, high-watermark, dynamic-resizing, backtrack, fast-clone, copy-on-write, serverless, aurora-serverless, acu, proxy-fleet, data-api, scale-to-zero, global-database, cross-region, dr, rpo, rto, switchover, multi-master, quorum, read-scaling, high-availability, mysql, postgresql, savings-plans, reserved-instances]
certs: [aws-saa]
updated: 2026-11-13
---

# Aurora

**Amazon Aurora** is AWS's custom-built relational database engine (MySQL- and PostgreSQL-compatible). Officially it is part of [RDS](rds.md), but its architecture is different enough that it should be treated as its own product: where RDS gives each instance its own dedicated [EBS](ebs.md) storage, Aurora is built around a **cluster** of instances sharing a single distributed **cluster volume**. Like the rest of RDS it is a relational, [ACID](../concepts/db-transaction-model.md) database product.

For migrations into Aurora, [DMS](dms.md) moves and replicates data while DMS Schema Conversion or AWS SCT handles schema/code conversion when the source engine changes.

Aurora comes in two compute models sharing the same storage architecture:

- **Aurora Provisioned** — you choose and pay for specific instances (a primary plus replicas). This is the baseline covered first below.
- **[Aurora Serverless](#aurora-serverless)** — capacity scales automatically, measured in ACUs, with no instances to manage.

## The Cluster

Aurora's base entity is the **cluster** — something the other RDS engines don't have (aside from the loosely analogous [RDS Multi-AZ cluster mode](rds.md#multi-az-deployments)). A cluster is:

- **Compute** — one **primary** instance plus **0 to 15 replicas**.
- **Storage** — a single **shared cluster volume** used by every instance.

### Replicas do both jobs

This looks like RDS's primary-plus-standby model but behaves very differently:

- In RDS (instance mode), the standby is **passive** — unusable until a failover.
- In Aurora, replicas **serve reads during normal operation and act as failover targets at the same time**. One Aurora replica delivers the benefit of both an RDS Multi-AZ standby (availability) and an RDS [read replica](rds.md#read-replicas) (read scaling) — you don't choose between them.

### Failover

- Up to **15 replicas**, and **any** of them can be promoted on failure.
- Failover is much faster than RDS because **no storage needs to be moved or re-attached** — every instance already uses the same cluster volume; only the compute role changes.

> [!tip] Exam Tip
> "Read scaling **and** availability from the same replicas", "no wasted standby", "up to 15 failover targets", "fast failover with no storage changes" are all Aurora-versus-RDS trigger phrases.

## Shared Cluster Volume

![Aurora storage architecture: a cluster spanning AZ-A, AZ-B and AZ-C with the primary instance in AZ-A and replicas in the other AZs; all instances read from a single shared cluster volume drawn as a red band across the AZs, only the primary writes to it, and the data is stored on six SSD storage nodes (two per AZ). Footer: Cluster Volume — Max 128 TiB, 6 Replicas, AZs](../assets/aurora-storage-architecture.png)

Aurora does **not** use local or per-instance storage. The cluster has one logical volume shared by all instances:

- **SSD-based only** — high IOPS, low latency; no magnetic option.
- Data is written to **6 storage nodes across 3 AZs** (2 per AZ). Writes to the primary are **synchronously replicated** to all six, and a write is acknowledged once a **quorum of 4 of 6** nodes confirms.
- Replication happens **at the storage level**, consuming **no CPU on the database instances** — fundamentally different from RDS, where replication is instance-driven.
- By default **only the primary writes**; the primary and all replicas read.
- **Self-healing**: Aurora automatically detects failed disk segments and repairs them from the other copies — disk failure alone rarely requires a snapshot or point-in-time restore.
- Because storage belongs to the cluster, **adding or removing replicas involves no storage provisioning** — replica changes are fast and cheap.

[Neptune](neptune.md), AWS's graph database, uses the same cluster design: a primary and up to 15 replicas sharing one volume with six copies across three AZs.

> [!warning] Gotcha — maximum volume size: 128 TiB vs 256 TiB
> The course (and most exam material) says **128 TiB**. Since **July 2025** AWS supports **256 TiB** on recent engine versions (Aurora MySQL 3.10+, Aurora PostgreSQL 15.13+/16.9+/17.5+). Expect **128 TiB** on the exam; know 256 TiB is the current real-world ceiling.

## Endpoints

![Aurora endpoints: the cluster endpoint (pink) points at the primary instance in AZ-A for reads and writes, while the reader endpoint (blue) fans out with dotted lines across the replicas in AZ-B and AZ-C, load-balancing reads; all instances sit on the shared cluster volume](../assets/aurora-endpoints.png)

Unlike RDS's single database CNAME, an Aurora cluster exposes **multiple endpoints**:

| Endpoint              | Points at                                                                    | Use for                                           |
| --------------------- | ---------------------------------------------------------------------------- | ------------------------------------------------- |
| **Cluster endpoint**  | Always the **primary**                                                       | Reads **and writes**                              |
| **Reader endpoint**   | **Load-balances** across **all replicas** (or the primary if there are none) | Reads                                             |
| **Custom endpoint**   | A user-defined subset of instances                                           | Tailored routing (e.g. an analytics replica pool) |
| **Instance endpoint** | One specific instance                                                        | Direct access, testing/fault-finding              |

- Every cluster has at minimum a **cluster endpoint** and a **reader endpoint**.
- The reader endpoint **updates automatically** as replicas come and go — read scaling needs no client reconfiguration.
- Custom and instance endpoints enable much more complex architectures than RDS supports.

### How the reader endpoint balances load

- It balances **connections, not queries**: *"The reader endpoint balances connections to available Aurora Replicas in an Aurora DB cluster. It doesn't balance individual queries. If you want to balance each query to distribute the read workload for a DB cluster, open a new connection to the reader endpoint for each query."* ([Aurora User Guide — Reader endpoints](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Endpoints.Reader.html))
- The mechanism is **round-robin DNS**: each resolution of the reader endpoint name returns one replica's IP address, rotating through the replicas, so distribution happens at connection-establishment time ([Aurora MySQL DBA Handbook — Connection Management, AWS whitepaper](https://d1.awsstatic.com/whitepapers/RDS/amazon-aurora-mysql-database-administrator-handbook.pdf)).
- Aurora DNS zones use a **5-second TTL**; clients must not cache DNS longer or the rotation is defeated and connections pile onto one replica — the JVM is called out as notorious for caching DNS indefinitely unless configured otherwise (same whitepaper).

> [!warning] Gotcha
> A long-lived connection pool that opens all its connections at startup (or a client that caches DNS) can end up **unevenly distributed** — every statement on an established connection stays on the replica it landed on. For per-query distribution AWS suggests a new connection per query, or [**RDS Proxy**](rds-proxy.md#read-only-endpoints-aurora-only) read-only endpoints in front of the cluster.

## Billing Model

- **Storage is not allocated** — unlike RDS you never pick a storage size. Aurora has two storage configurations:
  - **Aurora Standard** bills consumed GB-month storage plus per-request read/write I/O.
  - **Aurora I/O-Optimized** bills usage and storage with **no separate read/write I/O operation charges**; AWS positions it for workloads where I/O is at least 25% of Aurora database spend.
- **Compute** — hourly rate per instance, **billed per second with a 10-minute minimum**.
- **Backup allowance** — free backup storage equal to **100% of the cluster's consumed storage**; for most low/medium-churn workloads backups are effectively included.
- **No free tier** — Aurora doesn't support micro instances. Beyond a single-AZ micro RDS instance, though, it generally offers better value.

> [!warning] Gotcha — high watermark vs dynamic resizing
> Older material (including the course) describes **high-watermark billing**: you pay for the *maximum* storage ever consumed, freed space is reusable but not un-billed, and the only way to shrink the bill is to **create a new cluster and migrate**. Since **late 2020** Aurora **dynamically resizes**: allocated storage (and the bill) drops automatically when data is physically removed (`DROP TABLE`, `DROP DATABASE`, `TRUNCATE`) — though not for plain row `DELETE`s (run `OPTIMIZE TABLE` on MySQL or `VACUUM` on PostgreSQL to reclaim), and reclamation is gradual (up to roughly 10 TB/day). The exam may still present the high-watermark model and its "new cluster + migrate" answer as correct — know both versions.

## Backups, Backtrack & Fast Clones

**Backups and restores** work like the rest of RDS ([automated backups, manual snapshots, PITR](rds.md#backups-snapshots--restore)); a restore creates a **brand-new cluster**. Two features go beyond that baseline:

### Backtrack

- **Rewinds the existing cluster in place** to a previous point in time — no new cluster, no endpoint changes, applications keep working unchanged.
- The fix for **known-time data corruption**: rewind to just before the bad write instead of restoring to a new database and re-pointing everything.
- Must be **enabled per cluster** (window is adjustable), and is **Aurora MySQL only** — not available on Aurora PostgreSQL.

### Fast clones

- Creates a new database that **references the source cluster's storage** instead of copying it (copy-on-write): the clone stores only **differences** — its own changes, plus pre-change copies of source data modified after cloning.
- Clones are created **very quickly** and consume **only the delta** in storage, not a full copy. Ideal for test/dev copies of production data.

> [!tip] Exam Tip
> *"Roll back the same database in place after corruption"* means **Backtrack** (MySQL only); *"copy of production quickly / without duplicating storage"* means **fast clone**; *"restore"* always means **a new cluster**.

## Aurora Serverless

**Aurora Serverless is to Aurora what Fargate is to [ECS](ecs.md)**: same product, but you stop provisioning and managing instances. You still create a cluster, but compute is measured in **ACUs (Aurora Capacity Units)** — an ACU is a certain amount of compute plus a matching amount of memory (roughly **2 GiB per ACU**). You set a **minimum and maximum ACU**, and the cluster scales between them with load. Compute is **billed per second**; the storage layer is the same 6-copies-across-AZs cluster volume as Provisioned, billed the same way.

> [!note] Naming
> AWS renamed **Aurora Serverless v2** to **Aurora serverless** in April 2026. The older "v2" wording still appears in docs, blogs, consoles, and exam material, so this page keeps "Serverless v2" when contrasting it with retired Serverless v1.

### v1 architecture (what the course diagram shows)

![Aurora Serverless v1 architecture: a user's application connects to an AWS-managed proxy fleet (a red strip of proxy instances) rather than to database compute directly; below it an Aurora Serverless cluster spans AZ-A, AZ-B and AZ-C with a band of differently-sized ACUs on top of cluster storage replicated across the AZs; on the right a warm pool of AWS-managed ACUs is shown being allocated into the cluster on demand](../assets/aurora-serverless-architecture.png)

The classic exam-tested architecture is **Serverless v1**:

- ACUs are allocated from a **warm pool managed by AWS** and shared across customers. Pool ACUs are **stateless with no local storage**, which is why they attach to a cluster in seconds.
- Applications never connect to ACU compute directly — connections land on a shared, AWS-managed **proxy fleet** that brokers them to whatever ACUs currently exist. Because the client is decoupled from the compute, the cluster **scales without dropping connections**.
- The cluster can **scale to zero and pause**, leaving only storage charges.
- The only things you manage are the **min/max ACU values**.

> [!warning] Gotcha — Serverless v1 is retired
> Aurora Serverless **v1 reached end of life on March 31, 2025** (no new v1 clusters after January 8, 2025; remaining clusters were auto-upgraded to v2 from April 7, 2025). The warm-pool/proxy-fleet description above is the **v1** architecture the exam has historically tested — the shipping product is **v2**.

### v1 vs v2

**Serverless v2** (GA 2022; now branded **Aurora serverless**) is architected differently — it's an **instance type inside a normal Aurora cluster**, not a separate cluster type:

| Aspect | Serverless v1 (legacy) | Serverless v2 / Aurora serverless (current) |
| --- | --- | --- |
| Implementation | Dedicated cluster type | A serverless **instance type** in a standard Aurora cluster |
| Scaling granularity | Capacity doubles (e.g. 8 to 16 ACU) | Fine-grained **0.5 ACU** increments, in place |
| Connection layer | AWS-managed **proxy fleet** | Direct, or optional [**RDS Proxy**](rds-proxy.md) |
| Scale to zero / auto-pause | Native (pause on inactivity) | Added **Nov 2024** (min 0 ACU on recent engine versions); resume takes up to ~15 seconds; storage still billed while paused |
| Max capacity | Lower ceiling (64/128 ACU by engine) | **256 ACUs** (~512 GiB) |
| Multi-AZ / Global Database / readers | Limited | Full parity with Provisioned (failover, [Global Database](#aurora-global-database), up to 15 readers) |
| Data API | Yes (HTTP endpoint) | **Yes** — relaunched for Serverless v2 **and provisioned** clusters (PostgreSQL first, then MySQL), without v1's 1,000 requests/second cap |

> [!warning] Gotcha — cost-commitment discounts
> **Reserved Instances never apply to Aurora Serverless** (RIs cover provisioned instance types only) — "buy an RI to save on the serverless workload" is a distractor. Since **December 2025**, **Database Savings Plans** (1-year, no-upfront, up to ~35%) do cover Aurora serverless alongside provisioned Aurora, RDS, DynamoDB and other database services. See [Cost Optimization](../patterns/cost-optimization.md).

### When Serverless fits

| Use case | Why |
| --- | --- |
| Infrequently used apps | Pay per second only when active (e.g. a low-traffic blog) |
| New applications | Unknown load — no up-front sizing, no disruptive resize later |
| Variable workloads | Scale for peaks instead of provisioning for them |
| Unpredictable workloads | Set a wide min–max ACU range and let it scale |
| Dev/test databases | Pause to zero when idle — storage-only cost |
| Multi-tenant apps | Infrastructure cost scales with customer count/revenue |

## Aurora Global Database

![Aurora Global Database spanning two regions: the primary region (us-east-1) has one read/write instance and up to 15 read replicas on a cluster volume, receiving writes from the application; the secondary region (ap-southeast-2) has up to 16 read-only replicas on its own copy of the cluster volume serving local reads; a horizontal arrow between the two cluster volumes is labelled about 1 second replication](../assets/aurora-global-database.png)

**Aurora Global Database** replicates one Aurora cluster from a **primary region** to up to **10 secondary regions** (raised from 5 in **May 2025** — older material and the exam may still say 5).

### Mechanics

- **Replication happens at the storage layer** (cluster volume to cluster volume), not in the database engine — so it has **no performance impact** on the instances.
- Cross-region replication latency is typically **1 second or less**, and it is **one-way** (primary out to secondaries).
- **Primary region**: 1 read/write instance plus up to 15 read replicas (16 instances total). **Each secondary region**: up to **16 read-only replicas** — the extra slot exists because there's no writer occupying it, not because the limit differs.
- Secondaries are **entirely read-only until promoted**. On a region failure you **promote** a secondary to a full read/write cluster.
- The two "storage-layer" legs differ: **in-region replication across AZs is synchronous** (6 copies, 4-of-6 quorum, zero in-region data loss); **cross-region replication is asynchronous** — which is exactly why the cross-region **RPO is roughly 1 second, not zero**.
- AWS distinguishes **switchover** (planned, syncs first, **RPO 0**) from **failover** (unplanned, RPO = replication lag at failure time, typically about 1 second; RTO can be **on the order of minutes**).

### Use cases

1. **Cross-region DR and business continuity** — pre-provisioned secondaries plus roughly 1-second replication give very low RPO and RTO when a whole region fails.
2. **Global read scaling** — put secondaries near international users so their application tier reads from the nearest region at low latency.

> [!tip] Exam Tip
> *"Cross-region DR", "region-wide outage", "low RPO/RTO across regions", "low-latency reads for international users"* all point at **Aurora Global Database**. Common traps: replication is **one-way** (not bidirectional), **storage-layer** (no performance impact), secondaries are **read-only until promoted**, and the cross-region leg is **asynchronous** (in-region across AZs is the synchronous part).

## Aurora Multi-Master (retired)

> [!warning] Gotcha — effectively retired
> Multi-master only ever existed on **Aurora MySQL v1 (5.6-compatible)**, which reached **end of life on February 28, 2023**. It was never offered on Aurora MySQL v2/v3 or Aurora PostgreSQL. Know the **concept** for older exam questions; don't expect to use it.

![Aurora multi-master: an application connects directly to two read/write Aurora instances with no cluster endpoint between them; a bidirectional replication (redo) arrow links the two writers for cache updates, and both propose writes down to the shared cluster volume spanning AZ-A, AZ-B and AZ-C](../assets/aurora-multi-master.png)

In default **single-master** Aurora, a failover still takes time: a replica must be promoted and the cluster endpoint re-pointed. **Multi-master** made **every instance a writer** so an application could operate straight through an instance failure:

- **No cluster or reader endpoint** — the application connects to instances **directly** and does its own load balancing and failure rerouting. Aurora does not balance multi-master writes.
- **Quorum writes**: a writer proposes each change to all storage nodes; nodes **reject conflicting in-flight changes**; the write commits only if a **quorum agrees**, otherwise the application gets an error to handle.
- **Cache coherence**: after a commit, the change is replicated **peer-to-peer (redo)** to the other writers so their in-memory caches stay consistent.
- The payoff is **fault tolerance, not write scaling**: with connections held to both writers, the app can shift 100% of traffic instantly when one fails — but only if the **application is designed for it**. Multi-master is a foundation for a fault-tolerant app, not an automatic guarantee.

## Exam Quick Reference

| Concept | Key fact |
| --- | --- |
| Cluster | 1 primary + up to **15 replicas** + shared cluster volume |
| Replicas | Serve **reads and failover simultaneously** |
| Storage | **6 copies across 3 AZs**, synchronous, storage-level, self-healing, SSD-only |
| Max volume | **128 TiB** (exam) / 256 TiB (current, recent engines) |
| Storage billing | **Standard** = consumed GB-month + per-request I/O; **I/O-Optimized** = no separate read/write I/O charges; high watermark (exam) vs dynamic resizing (current) |
| Backups | Free allowance = 100% of cluster storage |
| Compute billing | Per-second, **10-minute minimum**; **no free tier** |
| Endpoints | **Cluster** (primary, R/W) · **Reader** (round-robin DNS load-balancing **per connection**, auto-updates) · custom · instance |
| Backtrack | **In-place rewind** for corruption; enable per cluster; **MySQL only** |
| Fast clone | Copy-on-write clone; stores **only differences**; near-instant |
| Serverless | **ACUs** (~2 GiB each), min/max range, per-second billing, pause to zero = storage-only cost |
| Serverless v1 | Warm pool + proxy fleet; **retired March 2025** |
| Serverless v2 / Aurora serverless | Instance type in a normal cluster; 0.5-ACU steps; 0–**256 ACUs**; scale-to-zero since Nov 2024; Data API supported |
| Global Database | 1 primary region + up to **5 (exam) / 10 (current)** secondaries; storage-layer, **async**, ~1 s, one-way; secondaries **read-only, up to 16 replicas**; promote on disaster |
| Multi-master | All instances write; **no cluster endpoint**; app load-balances; quorum writes with conflict rejection; **Aurora MySQL 5.6 only, EOL Feb 2023** |

## References

- Adrian Cantrill SAA course — Aurora Architecture, Aurora Serverless, Aurora Global Database, Aurora Multi-Master lessons (`raw/aws-cert-study-notes/aws-solution-architect/`)
- [Amazon Aurora storage and reliability](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Overview.StorageReliability.html) (cluster volume, Standard vs I/O-Optimized storage configurations, automatic resizing)
- [Aurora MySQL 256 TiB announcement (Jul 2025)](https://aws.amazon.com/about-aws/whats-new/2025/07/amazon-aurora-mysql-database-clusters-256-tib-storage/) · [Aurora PostgreSQL 256 TiB announcement](https://aws.amazon.com/about-aws/whats-new/2025/07/amazon-aurora-postgresql-database-clusters-256-tib-storage-volume/)
- [Amazon Aurora high availability](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Concepts.AuroraHighAvailability.html)
- [Reader endpoints for Amazon Aurora](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Endpoints.Reader.html) · [Aurora MySQL DBA Handbook — Connection Management (whitepaper)](https://d1.awsstatic.com/whitepapers/RDS/amazon-aurora-mysql-database-administrator-handbook.pdf)
- [Using Amazon Aurora Global Database](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database.html) · [Global Database switchover and failover](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database-disaster-recovery.html) · [10 secondary Regions (May 2025)](https://aws.amazon.com/blogs/database/amazon-aurora-global-database-introduces-support-for-up-to-10-secondary-regions/)
- [Aurora Serverless v2 scaling to zero (Nov 2024)](https://aws.amazon.com/blogs/database/introducing-scaling-to-0-capacity-with-amazon-aurora-serverless-v2/) · [Automatic pause and resume](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless-v2-auto-pause.html)
- [Aurora Serverless v1 end of support (Mar 31, 2025)](https://repost.aws/questions/QUhcMVoChXRm2HLi8F-yih1g/announcement-support-for-aurora-serverless-v1-ending-soon)
- [Data API for Aurora Serverless v2 and provisioned clusters](https://aws.amazon.com/blogs/database/introducing-the-data-api-for-amazon-aurora-serverless-v2-and-amazon-aurora-provisioned-clusters/) (includes April 2026 rename note)
- [Database Savings Plans announcement (Dec 2025)](https://aws.amazon.com/about-aws/whats-new/2025/12/database-savings-plans-savings/)
- Related: [RDS](rds.md) · [DMS](dms.md) · [Database Transaction Models](../concepts/db-transaction-model.md) · [Resource Scope](../concepts/resource-scope.md)
