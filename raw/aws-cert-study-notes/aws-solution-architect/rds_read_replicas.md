# RDS Read Replicas

## Overview

Read replicas are **read-only replicas of an RDS instance**. They deliver three main benefits:

- 🎯 **Read performance / read scaling** — offload read operations from the primary.
- 🎯 **Cross-region failover capability** — global resilience.
- 🎯 **Low RTO recovery** — fast promotion to a standalone instance, _as long as the disaster is failure and not data corruption_.

Key contrast with Multi-AZ: with Multi-AZ (instance mode) you **cannot** use the standby for anything by default. With read replicas you **can** — but only for **read** operations.

> 📚 Multi-AZ **cluster mode** (the newer version) is conceptually like the old Multi-AZ instance mode combined with read replicas, since its readable standbys can serve reads.

---

## Core Mechanics

### They are separate things

⚠️ Read replicas are **not part of the main DB instance**. They:

- Have their **own database endpoint address**.
- Require **application changes** to be used — apps must be pointed at the replica endpoint.
- Provide **no automatic failover** by default. They just sit off to one side.

An application (e.g. WordPress) using an RDS instance has **zero knowledge** of any read replicas by default. ⚠️ **Without application support, read replicas do nothing** — they aren't functional from a usage perspective.

### Asynchronous replication

- Read replicas are kept in sync using **asynchronous replication**.
- Data is written to the primary **first** → at that point it's considered **committed** → **then** replicated to the read replicas.
- This means there can be a small **replication lag** (potentially seconds), depending on network conditions and write volume.

> 🎯 **Exam keyword mapping (RDS, excluding Aurora):**
> 
> - **Synchronous** → **Multi-AZ**
> - **Asynchronous** → **Read Replicas**

📚 _Replication contrast:_ Multi-AZ synchronous replication writes to the primary disk and the standby as a single conceptual write operation. Async (read replicas) commits on the primary before replicating.

### Same-region and cross-region

- Read replicas can be created in the **same region** as the primary, or in **other AWS regions** (cross-region read replicas).
- For cross-region replicas, AWS handles all inter-region networking **transparently**, and it is **fully encrypted in transit**.

---

## Diagram Walkthrough

![[rds-read-replicas.png]]

**Context (outer frame):** Two regions stacked vertically. The top region is `us-east-1`; the bottom is `ap-southeast-2`. Each region contains a VPC (`a4l-vpc1` and `a4l-vpc10`). The big "Read Only DB Replicas" label on the left is the topic banner, not an object.

**Main objects (top region, `us-east-1`, across three AZs):**

- **AZ-A — Standby:** A standby instance. The **red dotted arrow** from Standby to Primary represents the **synchronous Multi-AZ** relationship (this is the standby of the primary, separate from the read replica story).
- **AZ-B — Primary:** The source DB instance. Clients read/write here.
- **AZ-C — ReadReplica:** A **same-region** read replica, connected to the Primary by a solid orange line labelled **Asynchronous Replication**.

**Cross-region object (bottom region, `ap-southeast-2`, AZ-B):**

- **ReadReplica:** A **cross-region** read replica. The orange line runs from the Primary in `us-east-1` down into `ap-southeast-2`, again labelled **Asynchronous Replication** — emphasizing that even across regions, replication is async and AWS handles the encrypted inter-region link.

**How to read it:** Orange (solid) = asynchronous read-replica replication; red (dotted) = synchronous Multi-AZ standby replication. The diagram shows one primary feeding both a same-region and a cross-region read replica simultaneously.

---

## Why Read Replicas Matter

### 1. Read performance & read scaling

- You can create **5 direct read replicas per DB instance** _(per the course)_, each adding an additional instance of read capacity — a simple way to **scale out** reads.
    
    > ⚠️ **Discrepancy flag:** Current AWS documentation now states **up to 15 read replicas** per source DB instance for MySQL, MariaDB, PostgreSQL, and SQL Server (Oracle up to 15, with 5 recommended to limit lag). The "5" figure is the older/long-standing number Cantrill teaches and may still appear on the exam. (Sources: [AWS RDS Read Replicas](https://aws.amazon.com/rds/features/read-replicas/), [RDS Quotas](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_Limits.html))
    
- ⚠️ **Cascading replicas (replicas of replicas):** Possible for further read scaling, but because replication is asynchronous, **lag compounds** at each level — take this into account.
    

### 2. Global read performance

- Read workloads in other regions can connect directly to a **cross-region read replica**, serving those reads **without impacting the primary**.

### 3. RPO / RTO improvements

- 📚 _Snapshots/backups_ improve **RPO** (less data loss with more frequent backups) but **don't help RTO** much — restoring snapshots is slow, especially for large databases.
- Read replicas offer a **near-zero RPO** — data is synced from the primary, so little potential for loss (assuming no corruption).
- Read replicas offer a **near-zero RTO** — they can be **promoted quickly** to a standalone instance in a disaster.

⚠️ **Critical DR caveat:** Only use read replicas for DR when **recovering from failure**. If you're recovering from **data corruption**, the replica likely holds a **copy of the corrupted data**. Read replicas → great low RTO for **failure**, **not** for data corruption.

### 4. Global availability / resilience

- A cross-region read replica can serve as a **failover region** if AWS experiences a major regional issue.

---

## Promotion

- Read replicas are **read-only until promoted**.
- ⚠️ Once **promoted**, a replica becomes a normal, fully usable RDS instance (read/write) and **stops being a replica** of the original.

---

## Exam Quick-Reference

|Concept|Detail|Trigger keywords|
|---|---|---|
|Replication type|**Asynchronous**|"async", "read replica", "potential lag"|
|vs Multi-AZ|Multi-AZ = **synchronous**|"sync", "standby", "automatic failover"|
|Endpoint|Replica has its **own endpoint**; apps must be changed|"app has no knowledge", "separate endpoint"|
|Automatic failover|❌ None by default|"no automatic failover"|
|Use of standby/replica|Replica usable for **reads**; Multi-AZ standby **not** usable by default|"read scaling", "offload reads"|
|Count|**5 per instance** (course) ⚠️ AWS docs now say **up to 15**|"scale reads", "how many replicas"|
|Cascading|Replicas of replicas allowed; **lag compounds**|"replica of a replica", "more lag"|
|Cross-region|Supported; AWS-managed, **encrypted in transit**|"global", "another region", "cross-region"|
|RPO|**Near-zero** (synced from primary)|"minimize data loss"|
|RTO|**Near-zero** (fast promotion)|"fast recovery", "low RTO"|
|⚠️ DR limit|Good for **failure**, **NOT data corruption**|"corruption", "deleted/bad data" → ❌ replica|
|Promotion|Read-only until promoted → then standalone instance|"promote", "standalone"|