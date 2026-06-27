---
title: RDS — Relational Database Service
type: service
tags: [rds, database, relational, ec2, ebs, vpc, subnet-group, multi-az, multi-az-instance, multi-az-cluster, read-replica, synchronous-replication, semi-synchronous-replication, asynchronous-replication, quorum, failover, reader-endpoint, cluster-endpoint, graviton, nvme, aurora, dr, high-availability, acid, cost]
certs: [aws-saa]
updated: 2026-06-26
---

# RDS — Relational Database Service

RDS is AWS's **managed database server** product. AWS owns the hardware, OS, engine installation, and most of the engine maintenance, so you operate a database without operating its host. This page also covers the prior question every architect must answer first: **should the database run on plain [EC2](ec2.md) at all?**

RDS databases are almost always relational and use the [ACID transaction model](../concepts/db-transaction-model.md) — which is the exam's default assumption whenever it says "ACID".

## Databases on EC2 — Why You Usually Shouldn't

You *can* run any database directly on an [EC2](ec2.md) instance. The honest framing is that doing so is **always a bad idea** — the only real question is whether the benefit to you or the business outweighs that fact. It can always be done, as long as you accept the negatives.

Two architectures people picture:

| Architecture | Description |
| --- | --- |
| **Single instance** | One larger EC2 instance running the DB, the application, and often a web server (Apache). Everything in one place, one AZ. |
| **Split architecture** | Two smaller instances — app/web server separated from the database, in the same AZ or split across AZs. |

Splitting introduces a **dependency**: the app and DB instances need reliable communication or the app breaks. It also touches cost — traffic between [AZs](../concepts/resource-scope.md) in the same region is charged, while communication over **private IPs in the same AZ is free**.

### When EC2-hosted DB is the right answer

There *are* exam questions where "run the database on EC2" is correct. The justifications that genuinely hold up:

| Reason | Verdict |
| --- | --- |
| Need a **DB engine or version AWS doesn't provide** | ✅ Often genuinely justified (niche/emerging databases) |
| Need a **specific OS + DB version combo** AWS doesn't offer | ✅ Valid |
| Need a **specific architecture / replication** AWS doesn't provide | ✅ Valid |
| Need **OS-level / root access** to the DB host | ⚠️ Question hard — rarely truly needed; often a vendor demand, and many vendors now support managed DBs. **RDS Custom** also closes much of this gap |
| **Root-level DB tuning** | ⚠️ Question — RDS exposes many historically root-only parameters without OS access |
| **Decision-makers just want it** on EC2 | Possibly unreasonable, but you may not have a choice |

### The negatives you take on

- **Admin overhead** — you manage the EC2 instance *and* the DB host (patching, version compatibility, out-of-hours fault-finding).
- **Backups & DR** — significant extra complexity that managed products automate.
- **Single AZ** — the instance and its [EBS](ebs.md) volume live in **one AZ**; an AZ failure can take DB access down. You own the snapshot/backup-to-S3 process.
- **No serverless / scaling** — EC2 is on or off; it can't easily scale down or absorb bursty load, and it sets a base minimum cost (the hourly rate).
- **Replication, performance, advanced features** — all yours to build, monitor, and forgo. AWS invests heavily in DB performance optimizations you can't easily replicate.

> [!warning] Gotcha
> An EC2-hosted database is **single-AZ by nature** — instance plus EBS volume in one AZ. That is the architectural reason it loses to RDS Multi-AZ in HA scenarios, and the reason "DB on EC2" is the *wrong* answer to most resilience questions even though it's the *right* answer to unsupported-engine ones.

## What RDS Actually Is

> [!tip] "Database **server** as a service", not "database as a service"
> With DBaaS you pay and receive *a database*. With RDS you pay for and receive a **database server (instance)** — so **one RDS instance can host multiple databases**. This distinction is testable.

- **Managed**: AWS handles hardware, OS, install, and much of the engine upkeep.
- **No SSH and no OS access.** The exception is **RDS Custom**, a variant that grants more low-level access.
- **Engines**: MySQL, MariaDB, PostgreSQL, plus commercial Oracle and Microsoft SQL Server (commercial engines carry licensing cost).

> [!warning] Aurora ≠ RDS
> **Amazon Aurora** is a custom AWS-built engine (compatible with some of the above, but designed by AWS). Many RDS features differ for Aurora — treat them as separate products. The most exam-relevant difference is **storage**: RDS gives each instance its **own dedicated EBS volume**; Aurora uses a shared cluster storage layer. Don't carry RDS storage facts over to Aurora.

## Architecture

![RDS architecture diagram: VPC1 in us-east-1 spanning three AZ columns (AZA/AZB/AZC). A subnet group defines which subnets RDS may use. A Primary instance replicates synchronously (green) to a Standby in another AZ, and asynchronously (orange) to a cross-region Read Replica in ap-southeast-2. Backups and snapshots flow to an AWS-managed S3 bucket. Each instance has its own dedicated EBS storage. RDS sits inside the VPC, reachable privately from on-premises over VPN/Direct Connect, or optionally given public addressing for internet access.](../assets/rds-architecture.png)

Read the diagram from the outside in: the outer panels are external networks (**On-Premises** left, **Internet** right); the central panel is the **us-east-1** region holding **VPC1**, split into three AZ columns; a second region (**ap-southeast-2**) at the bottom hosts a cross-region element. Region/AZ names and the public/private split are **illustrative patterns**, not fixed values.

Key elements:

- **RDS runs inside a [VPC](vpc.md)** — it is *not* a public-zone service like S3 or DynamoDB. It needs subnets in specific AZs.
- **Subnet group** defines *which subnets RDS may use* (see below).
- **Private subnets** (the normal case) — reachable from inside the VPC or from connected networks over **VPN / Direct Connect**.
- **Public addressing** can optionally be given to instances in public subnets for internet access — possible but a **security anti-pattern**.
- **Primary → Standby** (green): **synchronous** replication, same region, different AZ.
- **Primary → Read Replica** (orange): **asynchronous** replication, same-region or **cross-region**.
- **Backups & snapshots → S3**: stored in an AWS-managed bucket you never see in your account.
- **Dedicated [EBS](ebs.md) storage per instance** — primary *and* standby each have their own.

### Subnet Groups

A **DB subnet group** is a named *list of subnets* RDS is allowed to place instances in.

- You **must pick a DB subnet group** at launch, whether or not the deployment is HA.
- For a **Multi-AZ / HA** launch, RDS places the primary in one subnet and the standby in another — chosen **at random** unless you express a preference, but always in **different AZs**.
- One subnet group **can** serve multiple instances, but they are then constrained to the same subnets. To spread databases across different subnet sets, use **multiple** subnet groups.
- Best practice (editorial): **one DB subnet group per RDS deployment** for maximum placement flexibility.

### Storage, Replication & Backups

| Aspect | RDS behaviour |
| --- | --- |
| **Databases per instance** | Multiple databases can live on one RDS instance |
| **Storage** | Each instance has its **own dedicated [EBS](ebs.md) storage**; in a Multi-AZ pair, primary and standby each have separate storage. ⚠️ Differs from Aurora |
| **Multi-AZ replication** | **Synchronous** primary → standby — data replicated as soon as the primary receives it; standby is an identical copy |
| **Read replicas** | **Asynchronous** replication; same-region or **cross-region**; used to **scale read load** or add **DR** resilience in another region |
| **Backups** | Go to AWS-managed **S3**, replicated across AZs in the region (survive an AZ failure). In **Multi-AZ mode, backups run from the standby** — no performance hit on the primary |

> [!tip] Exam Tip
> Two replication words to keep straight: **standby = synchronous** (HA, identical copy, automatic failover target — *not* readable by your app), **read replica = asynchronous** (read scaling and/or DR, can be cross-region, *is* readable). Mixing these up is a classic trap.

## Multi-AZ Deployments

Multi-AZ is how RDS delivers **high availability**. Historically there was a single "Multi-AZ" option; RDS now offers **two distinct architectures** that differ in topology, replication, failover speed, and — crucially — whether the standby can serve reads:

1. **Multi-AZ instance deployment** — the original. One primary plus **one standby** that **cannot** serve reads or writes; the standby exists only to be failed over to.
2. **Multi-AZ cluster deployment** — one writer plus **two readable standbys**, with faster failover and read scaling.

> [!warning] Gotcha — Multi-AZ is in-region HA, not DR
> Both modes span **only availability zones within a single region**. Multi-AZ is **not** a cross-region or disaster-recovery mechanism — for that you need a cross-region [read replica](#storage-replication--backups) (or other DR tooling). "Multi-AZ for DR / cross-region" is a classic distractor.

### Multi-AZ Instance Deployment

![RDS Multi-AZ instance mode: a single client connects through a Database CNAME that resolves to the Primary instance in AZA. The Primary replicates synchronously (green arrow) to a Standby in AZB; AZC is empty. Backups flow from the standby into S3, replicated across AZs in-region. The standby serves no traffic.](../assets/rds-multi-AZ-instance-mode.png)

One **primary** instance holds your databases and, when Multi-AZ is enabled, replicates **synchronously** to a **standby replica** in another AZ. Key mechanics:

- All access (reads **and** writes) goes to the primary via the **database CNAME** — a DNS name that by default resolves to the primary, so clients are never hard-wired to an instance. The standby serves **no traffic, not even reads**; its only job is to wait for a failover.
- Replication is at the **storage level** — less efficient than cluster mode. The exact mechanism is engine-specific but abstracted away (Amazon failover technology for MariaDB/MySQL/Oracle/PostgreSQL; database mirroring or Always-On availability groups for SQL Server).
- **Backups and snapshots are taken from the standby**, so there is **no I/O load on the primary** during backup.

**Synchronous replication** means data is written to the primary and immediately replicated to the standby **before the write is viewed as committed**. This gives an effective **RPO of zero** on failover — no committed data is lost.

![RDS Multi-AZ instance failover: the same architecture after failover. The Database CNAME has been repointed across to AZB — the former standby, now promoted to primary — so client traffic now terminates there. The old AZA instance is out of the serving path. Replication and S3 backups are otherwise unchanged.](../assets/rds-multi-AZ-instance-failover.png)

On failure RDS **fails over automatically** (and you can trigger failover manually for testing or maintenance). The standby is promoted to primary and the **CNAME is repointed** to it. Because failover is fundamentally a **DNS change** — the data was already there synchronously — it typically takes **60–120 seconds**, gated by DNS propagation and client-side caching rather than by any data movement.

> [!tip] Exam Tip
> Failover in instance mode is **a DNS repoint, not a data-recovery operation**. The 60–120 s cost is DNS propagation plus client caching. Reduce the outage window by **disabling DNS caching** for that name in the application so it picks up the new target as soon as failover completes.

Failover triggers include: AZ outage, loss of network connectivity to the primary, primary compute or storage failure, manual failover, instance-type change, and software patching (RDS fails over, patches the now-idle old primary, then can fail back).

Instance-mode quick facts: synchronous replication, **exactly one** standby, standby **cannot** serve reads/writes, **not** in the free tier (extra cost of the standby), failover **60–120 s**, same-region/different-AZ only.

### Multi-AZ Cluster Deployment

![RDS Multi-AZ cluster mode: multiple clients send writes to a single Writer instance in AZA (blue arrow) and reads to either the Writer or one of two Readers in AZB and AZC (pink arrows). The Writer replicates to both readers (green arrows). Three endpoint types are called out: Cluster, Reader, and Instance.](../assets/rds-multi-AZ-cluster-mode.png)

Cluster mode runs **one writer plus two readers**, each in a different AZ.

- **Exactly two readers** — no more, no fewer. (Aurora allows more; this cap is a key exam distinction.)
- The **writer** handles writes **and** reads; the **readers are usable** for **read operations only** (unlike the instance-mode standby). Using readers requires **application support** — the app must send writes to the writer and reads to the readers.
- This is what lets cluster mode **scale read workloads**; instance mode cannot.
- Each instance still has its **own local storage** — contrast with Aurora's shared cluster storage layer.

A write is viewed as **committed once at least one reader confirms it has written the data**, at which point it is resilient across multiple AZs.

> [!warning] Gotcha — "synchronous" vs semi-synchronous
> Some materials label cluster replication "synchronous." Official AWS documentation describes Multi-AZ DB cluster replication as **semi-synchronous** (engine-native), using a **two-of-three quorum**: a commit needs acknowledgement from **at least one** of the two standbys, after which data is copied **asynchronously** from local SSD to the attached EBS volumes. For the exam, "commit acknowledged by ≥ 1 reader" is the safe mental model; if a question hinges on the exact term, AWS uses **semi-synchronous / quorum**.

> [!tip] Mental model — the readers are read replicas with stronger replication
> The two readable standbys behave much like [read replicas](#storage-replication--backups): they serve read traffic and each can exhibit its own **replica lag**. The difference is the replication mode. As the AWS deployment-options blog puts it:
>
> > The behavior of readable standby instances observed in a Multi-AZ DB cluster with respect to replica lag is similar to what we observe in a Single-AZ or Multi-AZ instance with one or more read replicas, where each read replica may lag by different values. However, unlike standalone read replicas of Single-AZ instances and Multi-AZ instances, where replication is asynchronous, the replication in a Multi-AZ DB cluster from the primary to readable standby instances is semi-synchronous.
>
> So a cluster gives you read-replica-style read scaling **and** a quorum-backed commit in a single HA deployment, instead of bolting separate async read replicas onto a Multi-AZ instance.

**Three endpoint types** in cluster mode:

| Endpoint | Points at | Used for |
| --- | --- | --- |
| **Cluster endpoint** | The **writer** | Reads, writes, administration (analogous to the database CNAME in instance mode) |
| **Reader endpoint** | Any available **reader** (sometimes the writer too) | Read scaling — apps balance reads across readers here |
| **Instance endpoint** | One **specific instance** | **Testing / fault-finding only** — does **not** fail over, so never use in production |

**Hardware and performance:** cluster mode runs on **Graviton** processors with **local NVMe SSD** storage. Writes hit the **local fast SSD first**, then flush through to **EBS** — combining local-storage speed with EBS durability. Readers offload the writer, giving the highest performance of any RDS mode short of Aurora.

**Failover** uses **transaction logs** (more efficient than storage-level replication) and **eliminates the crash-recovery step**, so it is much faster than instance mode — typically quoted at **~35 s** as an optimistic floor, with AWS giving a **25–75 s** typical range depending on replica lag.

### Instance vs Cluster — Comparison

| Feature | Multi-AZ **Instance** | Multi-AZ **Cluster** |
| --- | --- | --- |
| Topology | 1 primary + **1 standby** (2 AZs) | 1 writer + **2 readable standbys** (3 AZs) |
| Standby usable for reads? | **No** — passive, failover only | **Yes** — readers serve read-only traffic |
| Read scaling? | **No** (need a separate read replica) | **Yes** via the reader endpoint |
| Write access | Primary only | Writer only |
| Replication | **Synchronous**, storage-level | **Semi-synchronous** (quorum, ≥ 1 reader ack), transaction-log-based |
| Commit latency | Higher than Single-AZ | **Up to 2× faster** write commits than instance mode |
| Failover time | **60–120 s** (DNS change) | **~35 s** floor / **25–75 s** typical (no crash recovery) |
| RPO on failover | **Zero** (synchronous) | Effectively zero (quorum guarantees commit on ≥ 1 standby) |
| Storage | EBS | **Local NVMe SSD flushed to EBS**, per-instance |
| Hardware | Standard | **Graviton** |
| Supported engines | MariaDB, MySQL, Oracle, PostgreSQL, **SQL Server** | **MySQL & PostgreSQL only** |
| Backups | Taken from **standby** (no primary I/O hit) | Crash-consistent EBS snapshot, no I/O suspension |
| Access pattern | Single **database CNAME** | **Cluster / Reader / Instance** endpoints |
| Free tier | No | No |
| Region scope | Same region, multi-AZ | Same region, multi-AZ |

> [!tip] Exam Tip
> Three resilience tools, three jobs: **Multi-AZ instance** = in-region HA with a passive standby; **Multi-AZ cluster** = in-region HA **plus** read scaling and faster failover (MySQL/PostgreSQL only); **[read replica](#storage-replication--backups)** = read scaling and/or **cross-region DR** (asynchronous, app-readable). Trigger words: "readable standby" / "faster failover" / "lower commit latency" → **cluster**; "SQL Server or Oracle + Multi-AZ" → must be **instance** (cluster is MySQL/PostgreSQL only); "read scaling" → cluster **or** read replica, **never** instance mode.

### Single-AZ (baseline)

For contrast: a **Single-AZ** instance is one managed instance on EBS with **no standby** — so **no automatic failover** during an AZ outage, an **RPO of roughly 5 minutes** (driven by the transaction-log copy interval to S3), and an RTO of minutes to hours. You can still attach read replicas and promote one manually after a failure. Fine for dev/test; **not** appropriate for production HA.

## Cost Model

RDS is **resource-allocation billed** (like [EC2](ec2.md)), *not* usage-billed — you pay for the instance whether or not it's busy.

| Cost component | Notes |
| --- | --- |
| **Instance size & type** | Bigger / more capable = more; hourly rate, **billed per second** |
| **Multi-AZ** | More instances = additional cost (amount depends on the Multi-AZ architecture) |
| **Storage** | Per-GB-month; provisioned-IOPS types cost more, aligned to [EBS](ebs.md) pricing |
| **Data transfer** | Per-GB for data in/out to the internet and other AWS regions |
| **Backups & snapshots** | Snapshot storage **equal to your DB storage is free** (2 TB DB → 2 TB free snapshots); beyond that, per-GB-month (1 TB for 1 month = 500 GB for 2 months) |
| **Commercial engine licensing** | Extra for Oracle / SQL Server |

## Exam Quick-Reference

| Concept | Trigger / key point |
| --- | --- |
| DB on EC2 is the *right* answer | Unsupported engine/version, specific OS+DB combo, genuine OS/root access, AWS-unsupported replication architecture |
| RDS = "DB **server** as a service" | Not DBaaS — one instance hosts **multiple databases** |
| No SSH / no OS access | Exception: **RDS Custom** |
| Aurora ≠ RDS | Separate product; different (shared-cluster) storage architecture |
| Multi-AZ replication | **Synchronous**, same region, different AZ, standby identical (and not readable) |
| Multi-AZ **instance** mode | 1 primary + 1 passive standby; failover = **DNS repoint**, **60–120 s**; all engines incl. SQL Server/Oracle |
| Multi-AZ **cluster** mode | 1 writer + **2 readable** readers; semi-synchronous quorum; failover **~35 s / 25–75 s**; Graviton + NVMe; **MySQL/PostgreSQL only** |
| Multi-AZ endpoints (cluster) | **Cluster** (writer), **Reader** (read scaling), **Instance** (testing only — no failover) |
| Read replicas | **Asynchronous**, same- or **cross-region**, read scaling + DR |
| Storage | Dedicated **EBS per instance** (unlike Aurora) |
| Backups | Go to **S3** (AWS-managed); Multi-AZ backups run from **standby** |
| Subnet group | List of subnets RDS may use; required at launch; Multi-AZ → primary + standby in different AZs (random unless specified) |
| Billing | Resource-allocation (like EC2), hourly rate **billed per second**, not usage-based |

> [!warning] Gotchas
>
> - **"Database as a service" is a deliberately wrong characterisation of RDS** — you receive a *server*, hence multiple DBs per instance.
> - **RDS instances in public subnets *can* get public IPs** — possible, but a security anti-pattern; know it exists as an option.
> - **Same-AZ private-IP traffic is free; cross-AZ traffic is charged** — relevant to both EC2-DB and RDS Multi-AZ designs.
> - **EC2-hosted DB = single AZ** — instance plus EBS volume; an AZ failure can take DB access down.
> - **Don't confuse RDS storage with Aurora storage** — RDS = dedicated EBS per instance; Aurora differs.

## References

- Cantrill SAA-C03 study notes — *RDS Architecture (and Databases on EC2)*, *RDS Multi-AZ (Instance and Cluster)*
- [AWS — Choose the right Amazon RDS deployment option: Single-AZ, Multi-AZ instance, or Multi-AZ DB cluster](https://aws.amazon.com/blogs/database/choose-the-right-amazon-rds-deployment-option-single-az-instance-multi-az-instance-or-multi-az-database-cluster/) (semi-synchronous quorum replication, engine restriction, failover-timing ranges, backup mechanics, Single-AZ baseline)
- Related: [EC2](ec2.md) · [EBS](ebs.md) · [VPC](vpc.md) · [Database Transaction Models (ACID/BASE/CAP)](../concepts/db-transaction-model.md) · [Resource Scope](../concepts/resource-scope.md)
