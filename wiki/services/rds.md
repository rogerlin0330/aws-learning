---
title: RDS — Relational Database Service
type: service
tags: [rds, rds-custom, database, relational, db2, ec2, ebs, vpc, subnet-group, multi-az, multi-az-instance, multi-az-cluster, read-replica, synchronous-replication, semi-synchronous-replication, asynchronous-replication, quorum, failover, reader-endpoint, cluster-endpoint, graviton, nvme, aurora, dms, dr, high-availability, acid, cost, backup, snapshot, automated-backup, manual-snapshot, transaction-logs, pitr, point-in-time-recovery, retention, restore, rpo, rto, s3, cross-region-backup, security, encryption, encryption-in-transit, encryption-at-rest, ssl, tls, kms, kms-key, cmk, dek, envelope-encryption, aes-256, tde, transparent-data-encryption, cloudhsm, iam-authentication, generate-db-auth-token, authentication, authorization, credentials-provider-chain, imds, postgresql, jdbc, aws-advanced-jdbc-wrapper, connection-pool, druid, hikaricp, rds-iam, ssl-mode, ca-bundle, session-manager, ssh, rdp, shared-responsibility]
certs: [aws-saa]
updated: 2026-07-03
---

# RDS — Relational Database Service

RDS is AWS's **managed database server** product. AWS owns the hardware, OS, engine installation, and most of the engine maintenance, so you operate a database without operating its host. This page also covers the prior question every architect must answer first: **should the database run on plain [EC2](ec2.md) at all?**

RDS databases are almost always relational and use the [ACID transaction model](../concepts/db-transaction-model.md) — which is the exam's default assumption whenever it says "ACID".

For migration into or out of RDS with minimal downtime, use [DMS](dms.md) with change data capture rather than trying to invent database replication yourself.

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
- **Engines**: IBM Db2, MySQL, MariaDB, PostgreSQL, plus commercial Oracle and Microsoft SQL Server (commercial engines carry licensing cost).

> [!warning] Aurora ≠ RDS
> **[Amazon Aurora](aurora.md)** is a custom AWS-built engine (compatible with some of the above, but designed by AWS). Many RDS features differ for Aurora — treat them as separate products. The most exam-relevant difference is **storage**: RDS gives each instance its **own dedicated EBS volume**; Aurora uses a [shared cluster storage layer](aurora.md#shared-cluster-volume). Don't carry RDS storage facts over to Aurora.

## RDS Custom

> [!note] Exam scope — awareness only
> RDS Custom is a **niche** topic. For the SAA exam you only need to know that it **exists** and **what gap it fills**. The detail below is for context; don't over-invest.

RDS Custom is the **middle ground** on the spectrum between the two extremes this page has already drawn: fully managed [standard RDS](#what-rds-actually-is) at one end and a fully self-managed [database on EC2](#databases-on-ec2--why-you-usually-shouldnt) at the other.

| Option | Who manages the OS + engine | Access you get |
| --- | --- | --- |
| **Standard RDS** | AWS — fully managed database-server-as-a-service | **No** OS or engine access |
| **RDS Custom** | Shared — RDS automation **plus** your own customization | **OS + DB engine** access (SSH, RDP, Session Manager) |
| **DB on [EC2](ec2.md)** | You — fully self-managed | Everything, from the OS up (high overhead) |

RDS Custom keeps the RDS managed-service automation but *also* lets you into the operating system and database engine to customize them — the thing standard RDS deliberately forbids. This is aimed at **legacy, custom, and packaged applications** that need OS/DB-level tweaks (vendor agents, specific patches, native features) yet still want managed-service benefits.

### Key facts

- **Supported engines: Microsoft SQL Server and Oracle only** — the *same pair* that supports engine-native [TDE](#2-tde--transparent-data-encryption-engine-native). MySQL, MariaDB, and PostgreSQL are **not** available on RDS Custom.
- **OS + engine access via SSH, RDP, and Session Manager** — the distinguishing feature versus standard RDS (which gives no OS access at all).
- **Resources live in *your* account.** Unlike standard RDS, you **will see** the underlying [EC2](ec2.md) instance, [EBS](ebs.md) volumes, and [S3](s3.md) backups in your own console. Standard RDS hides all of this in an AWS-managed environment and only injects **elastic network interfaces (ENIs)** into your [VPC](vpc.md) for connectivity.

### Customization workflow — pause the automation

Because RDS automation and your manual changes would otherwise fight each other, customizing follows a three-step dance:

1. **Pause** database automation.
2. Perform your customizations against the OS or engine.
3. **Resume** (re-enable) automation to make the database production-ready again.

### Shared-responsibility position

![RDS Custom shared-responsibility comparison: four columns from most customer-managed (On-Premises, left) to most AWS-managed (right) — On-Premises, RDS, RDS Custom, EC2. Rows are the stack layers from Hardware at the bottom up to Optimization at the top. Colour legend: blue = customer, orange = AWS, gradient = shared. On-Premises is all-customer; RDS is customer-optimization with everything below it AWS; EC2 is all-customer above AWS-owned hardware; RDS Custom sits in between — AWS owns the hardware, the customer owns application optimization, and every layer between (OS install, OS patches, DB patches, backups, HA, scaling) is shared.](../assets/rds-custom.png)

Read the diagram as four columns from most-customer-managed (left) to most-AWS-managed (right), with each row a layer of the stack. The takeaway: **RDS Custom = AWS owns the hardware, the customer owns application optimization, and everything in between is shared** — RDS automation benefits *plus* the ability to customize.

| Layer (bottom to top) | On-Premises | RDS | **RDS Custom** | EC2 |
| --- | --- | --- | --- | --- |
| Optimization | Customer | Customer | Customer | Customer |
| Scaling | Customer | AWS | **Shared** | Customer |
| HA | Customer | AWS | **Shared** | Customer |
| Backups | Customer | AWS | **Shared** | Customer |
| DB patches | Customer | AWS | **Shared** | Customer |
| OS patches | Customer | AWS | **Shared** | Customer |
| OS install | Customer | AWS | **Shared** | Customer |
| Hardware | Customer | AWS | AWS | AWS |

> [!warning] Gotchas
>
> - **Engines: SQL Server + Oracle only** — the same pair as [TDE](#2-tde--transparent-data-encryption-engine-native). Don't confuse with the fuller standard-RDS engine list.
> - **Visibility:** with **RDS Custom** you *see* the EC2/EBS/S3-backup resources in your account; with **standard RDS** you do **not** (AWS-managed, ENI injection only).
> - **Access:** SSH / RDP / Session Manager access is the feature that separates RDS Custom from standard RDS (no OS access).

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
| **Backups** | Go to AWS-managed **S3**, replicated across AZs in the region (survive an AZ failure). In **Multi-AZ instance** mode, backups for MariaDB/MySQL/Oracle/PostgreSQL are taken from the standby so the primary avoids I/O suspension; **SQL Server still briefly suspends I/O** during backup |

> [!tip] Exam Tip
> Two replication words to keep straight: **standby = synchronous** (HA, identical copy, automatic failover target — *not* readable by your app), **read replica = asynchronous** (read scaling and/or DR, can be cross-region, *is* readable). Mixing these up is a classic trap.

## Multi-AZ Deployments

Multi-AZ is how RDS delivers **high availability**. Historically there was a single "Multi-AZ" option; RDS now offers **two distinct architectures** that differ in topology, replication, failover speed, and — crucially — whether the standby can serve reads:

1. **Multi-AZ instance deployment** — the original. One primary plus **one standby** that **cannot** serve reads or writes; the standby exists only to be failed over to.
2. **Multi-AZ cluster deployment** — one writer plus **two readable standbys**, with faster failover and read scaling.

> [!warning] Gotcha — Multi-AZ is in-region HA, not DR
> Both modes span **only availability zones within a single region**. Multi-AZ is **not** a cross-region or disaster-recovery mechanism — for that you need a cross-region [read replica](#read-replicas) (or other DR tooling). "Multi-AZ for DR / cross-region" is a classic distractor.

### Multi-AZ Instance Deployment

![RDS Multi-AZ instance mode: a single client connects through a Database CNAME that resolves to the Primary instance in AZA. The Primary replicates synchronously (green arrow) to a Standby in AZB; AZC is empty. Backups flow from the standby into S3, replicated across AZs in-region. The standby serves no traffic.](../assets/rds-multi-AZ-instance-mode.png)

One **primary** instance holds your databases and, when Multi-AZ is enabled, replicates **synchronously** to a **standby replica** in another AZ. Key mechanics:

- All access (reads **and** writes) goes to the primary via the **database CNAME** — a DNS name that by default resolves to the primary, so clients are never hard-wired to an instance. The standby serves **no traffic, not even reads**; its only job is to wait for a failover.
- Replication is at the **storage level** — less efficient than cluster mode. The exact mechanism is engine-specific but abstracted away (Amazon failover technology for MariaDB/MySQL/Oracle/PostgreSQL; database mirroring or Always-On availability groups for SQL Server).
- For **MariaDB, MySQL, Oracle, and PostgreSQL**, backups and snapshots are taken from the standby, so the primary avoids I/O suspension. **SQL Server is the exception**: AWS still briefly suspends I/O during Multi-AZ backup.

**Synchronous replication** means data is written to the primary and immediately replicated to the standby **before the write is viewed as committed**. This gives an effective **Recovery Point Objective (RPO) of zero** on failover — no committed data is lost.

![RDS Multi-AZ instance failover: the same architecture after failover. The Database CNAME has been repointed across to AZB — the former standby, now promoted to primary — so client traffic now terminates there. The old AZA instance is out of the serving path. Replication and S3 backups are otherwise unchanged.](../assets/rds-multi-AZ-instance-failover.png)

On failure RDS **fails over automatically** (and you can trigger failover manually for testing or maintenance). The standby is promoted to primary and the **CNAME is repointed** to it. Because failover is fundamentally a **DNS change** — the data was already there synchronously — it typically takes **60–120 seconds**, gated by DNS propagation and client-side caching rather than by any data movement.

> [!tip] Exam Tip
> Failover in instance mode is **a DNS repoint, not a data-recovery operation**. The 60–120 s cost is DNS propagation plus client caching. Reduce the outage window by **disabling DNS caching** for that name in the application so it picks up the new target as soon as failover completes.

Failover triggers include: AZ outage, loss of network connectivity to the primary, primary compute or storage failure, manual failover, instance-type change, and software patching (RDS fails over, patches the now-idle old primary, then can fail back).

Instance-mode quick facts: synchronous replication, **exactly one** standby, standby **cannot** serve reads/writes, **not** in the free tier (extra cost of the standby), failover **60–120 s**, same-region/different-AZ only.

### Multi-AZ Cluster Deployment

![RDS Multi-AZ cluster mode: multiple clients send writes to a single Writer instance in AZA (blue arrow) and reads to either the Writer or one of two Readers in AZB and AZC (pink arrows). The Writer replicates to both readers (green arrows). Three endpoint types are called out: Cluster, Reader, and Instance.](../assets/rds-multi-AZ-cluster-mode.png)

Cluster mode runs **one writer plus two readers**, each in a different AZ.

- **Exactly two readers** — no more, no fewer. ([Aurora](aurora.md) allows more; this cap is a key exam distinction.)
- The **writer** handles writes **and** reads; the **readers are usable** for **read operations only** (unlike the instance-mode standby). Using readers requires **application support** — the app must send writes to the writer and reads to the readers.
- This is what lets cluster mode **scale read workloads**; instance mode cannot.
- Each instance still has its **own local storage** — contrast with Aurora's shared cluster storage layer.

A write is viewed as **committed once at least one reader confirms it has written the data**, at which point it is resilient across multiple AZs.

> [!warning] Gotcha — "synchronous" vs semi-synchronous
> Some materials label cluster replication "synchronous." Official AWS documentation describes Multi-AZ DB cluster replication as **semi-synchronous** (engine-native), using a **two-of-three quorum**: a commit needs acknowledgement from **at least one** of the two standbys, after which data is copied **asynchronously** from local SSD to the attached EBS volumes. For the exam, "commit acknowledged by ≥ 1 reader" is the safe mental model; if a question hinges on the exact term, AWS uses **semi-synchronous / quorum**.

> [!tip] Mental model — the readers are read replicas with stronger replication
> The two readable standbys behave much like [read replicas](#read-replicas): they serve read traffic and each can exhibit its own **replica lag**. The difference is the replication mode. As the AWS deployment-options blog puts it:
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
| Supported engines | Db2, MariaDB, MySQL, Oracle, PostgreSQL, **SQL Server** | **MySQL & PostgreSQL only** |
| Backups | Standby-sourced for MariaDB/MySQL/Oracle/PostgreSQL; **SQL Server briefly suspends I/O** | Crash-consistent EBS snapshot, no I/O suspension |
| Access pattern | Single **database CNAME** | **Cluster / Reader / Instance** endpoints |
| Free tier | No | No |
| Region scope | Same region, multi-AZ | Same region, multi-AZ |

> [!tip] Exam Tip
> Three resilience tools, three jobs: **Multi-AZ instance** = in-region HA with a passive standby; **Multi-AZ cluster** = in-region HA **plus** read scaling and faster failover (MySQL/PostgreSQL only); **[read replica](#read-replicas)** = read scaling and/or **cross-region DR** (asynchronous, app-readable). Trigger words: "readable standby" / "faster failover" / "lower commit latency" → **cluster**; "SQL Server or Oracle + Multi-AZ" → must be **instance** (cluster is MySQL/PostgreSQL only); "read scaling" → cluster **or** read replica, **never** instance mode.

### Single-AZ (baseline)

For contrast: a **Single-AZ** instance is one managed instance on EBS with **no standby** — so **no automatic failover** during an AZ outage, an **RPO of roughly 5 minutes** (driven by the transaction-log copy interval to S3), and a Recovery Time Objective (RTO) of minutes to hours. You can still attach [read replicas](#read-replicas) and promote one manually after a failure. Fine for dev/test; **not** appropriate for production HA.

## Read Replicas

Where Multi-AZ exists for **availability**, read replicas exist for **read scaling** and **cross-region disaster recovery**. A read replica is usually a **read-only copy of an RDS instance** kept in sync from the primary, with two engine-specific DR exceptions called out below. It delivers three benefits worth separating in your head:

1. **Read performance / read scaling** — offload read queries from the primary.
2. **Cross-region resilience** — a replica in another region survives a regional failure.
3. **Low RTO recovery** — it can be promoted to a standalone instance quickly, *as long as the disaster is a failure and not data corruption*.

The sharpest contrast with [Multi-AZ instance mode](#multi-az-instance-deployment): the instance-mode standby **cannot** be used for anything by default, whereas a read replica **can** — but only for **reads**. ([Multi-AZ cluster mode](#multi-az-cluster-deployment) is conceptually instance-mode HA combined with read-replica-style read scaling, since its standbys are readable.)

![RDS read replicas: two stacked regions. In us-east-1 (a4l-vpc1) across three AZs, a Standby in AZ-A replicates synchronously to the Primary in AZ-B (red dotted arrow, the Multi-AZ relationship), while a same-region Read Replica in AZ-C connects to the Primary by a solid orange Asynchronous Replication line. A second orange Asynchronous Replication line runs from the Primary down into ap-southeast-2 (a4l-vpc10, AZ-B) to a cross-region Read Replica. Orange solid = async read-replica replication; red dotted = synchronous Multi-AZ standby. One primary feeds both a same-region and a cross-region replica at once.](../assets/rds-read-replicas.png)

The diagram's colour code is the thing to lock in: **orange solid = asynchronous read-replica replication**; **red dotted = synchronous Multi-AZ standby replication**. A single primary can feed both a same-region and a cross-region read replica simultaneously, and AWS handles the inter-region link transparently and **encrypted in transit**.

### They Are Separate Things

> [!warning] Gotcha — a read replica does nothing without application support
> A read replica is **not part of the main DB instance**. It has its **own endpoint address**, so the application must be **changed to point reads at that endpoint** — an app such as WordPress has **zero knowledge** of any read replica by default. There is also **no automatic failover** to a read replica; it just sits off to one side. Without application support, read replicas are functionally inert.

> [!warning] Gotcha — Db2 and Oracle have DR-only replica modes
> Current AWS docs call out two exceptions to the "read replica = read-only workload target" rule: **RDS for Db2 standby replicas** and **RDS for Oracle mounted replicas** don't accept user connections, so they can't serve read traffic. Their main purpose is **cross-region disaster recovery**, not read scaling.

### Asynchronous Replication

Read replicas use **asynchronous** replication. Data is written to the primary **first** and is considered **committed at that point**, and only *then* replicated out to the replicas. The consequence is a small **replication lag** (potentially seconds) that grows with write volume and network distance.

> [!tip] Exam Tip — the keyword mapping
> For RDS (excluding Aurora): **synchronous maps to Multi-AZ**, **asynchronous maps to read replicas**. Multi-AZ writes the primary and standby as a single conceptual operation; async replication commits on the primary *before* replicating. Trigger words *"async"*, *"potential lag"*, *"offload reads"*, *"another region"* all point at read replicas.

### Same-Region and Cross-Region

Read replicas can live in the **same region** as the primary or in **another region** (cross-region read replicas). For cross-region replicas AWS handles all inter-region networking transparently and **encrypts it in transit** — you don't build the link.

### Read Scaling

- You can create up to **5 read replicas directly per DB instance** *(the long-standing number Cantrill teaches)*, each adding a full instance of read capacity — a simple way to **scale out** reads.

  > [!warning] Gotcha — the "5" figure is dated
  > Current AWS documentation states a default quota of **up to 15 read replicas per primary DB instance**. For Oracle, AWS also recommends limiting replicas to **5** to minimize replication lag. The **5** is the older number that may still appear on the exam — know both.

- **Cascading replicas** (replicas of replicas) are possible for further scaling, but because replication is asynchronous, **lag compounds** at each hop. Take that into account before chaining them.

A **cross-region** read replica also serves reads for workloads in *that* region directly, without ever touching the primary — global read performance for free once the replica exists.

### RPO / RTO and Disaster Recovery

Read replicas are a **recovery-time (RTO)** tool, complementing the **recovery-point (RPO)** improvement that [backups](#backups-snapshots--restore) give you:

- **Near-zero RPO** — data is continuously synced from the primary, so little is at risk of loss (assuming the data is sound).
- **Near-zero RTO** — a replica can be **promoted quickly** to a standalone instance during a disaster, far faster than restoring a large snapshot.

> [!warning] Gotcha — DR only for failure, not corruption
> Use a read replica for DR **only when recovering from failure**. If you are recovering from **data corruption**, the replica almost certainly holds a **copy of the corrupted data** — it replicated it. Read replicas give a great low RTO for **failure**; they are **not** a defence against bad/deleted/corrupted data (that's what point-in-time [backups](#backups-snapshots--restore) are for).

A cross-region read replica can also act as a **failover region** if AWS suffers a major regional issue.

### Promotion

A read replica is **read-only until promoted**. Once **promoted**, it becomes a normal, fully usable read/write RDS instance and **stops being a replica** of the original — the replication link is severed and it goes its own way.

> [!tip] Exam Tip — three resilience tools, three jobs
> **Multi-AZ standby** = in-region HA, synchronous, not readable, automatic failover. **Multi-AZ cluster readers** = in-region HA plus read scaling, semi-synchronous. **Read replica** = read scaling and/or **cross-region DR**, asynchronous, app-readable, **manual** promotion, no automatic failover. *"Read scaling"* points at a cluster **or** a read replica, **never** an instance-mode standby; *"cross-region DR"* or *"promote to standalone"* points specifically at a **read replica**.

## Backups, Snapshots & Restore

RDS has **two backup-like mechanisms**: **manual snapshots** (you trigger) and **automated backups** (RDS triggers daily). Both land in **AWS-managed [S3](s3.md) buckets** — visible in the RDS console but **never in your own S3 console** (a classic exam trap). Because the data sits in S3, all backups are **regionally resilient**: S3 replicates them across AZs within the region, so they survive an AZ failure.

![RDS backups in us-east-1: a SingleAZ/Standby RDS instance in AZ-B produces backup data. Three arrows leave it — a blue arrow (manual snapshots) and an orange arrow (the daily automated backup) both write full-first-then-incremental snapshots to AWS-managed S3 buckets, and a third orange arrow streams transaction logs every 5 minutes to S3 for automated backups only. Footer notes automated-backup retention of 0 to 35 days; manual snapshots never expire.](../assets/rds-backups-snapshots-restore.png)

The diagram's three arrows separate the **three data streams** — manual snapshot, automated daily snapshot, and the 5-minute transaction-log stream — but the **full-first-then-incremental** rule is shared by **both** snapshot types (AWS treats an automated backup as an automated snapshot). The transaction-log stream and the 0–35 day retention are **automated-backup-only**.

**Backup performance impact:** on **Single-AZ** the backup is read from the only instance, so you may see an I/O pause. On **Multi-AZ instance** mode, MariaDB/MySQL/Oracle/PostgreSQL backups are read from the **standby**, so the primary avoids I/O suspension; **SQL Server still briefly suspends I/O** during backup (see [Multi-AZ Instance Deployment](#multi-az-instance-deployment)).

### Manual Snapshots

- **Not automatic** — you trigger them explicitly (or via a script). Taken at the **instance level**, so a snapshot captures **all databases** on the instance, not a single database.
- Stored in AWS-managed S3; behave like [EBS](ebs.md) snapshots. **First snapshot is full** (sized to consumed data); **subsequent are incremental** (changed blocks only). High-change-rate databases can make even incremental snapshots slow.
- A brief **I/O interruption** occurs (felt on Single-AZ, absorbed by the standby on Multi-AZ for MariaDB/MySQL/Oracle/PostgreSQL; SQL Server still briefly suspends I/O).
- 🎯 **Snapshots never expire.** They **survive deletion of the RDS instance** and are removed only when **you** delete them manually. Heavily tested.
- **Frequency is your choice** (hourly, daily, weekly). More frequent snapshots mean a **lower RPO** (less maximum data loss). Default quota is **100 manual snapshots per Region**.

### Automated Backups

- Run **once per day** during a defined **backup window** (AWS-chosen at random, or set to fit your business). On Single-AZ schedule it for low-use periods; on Multi-AZ it is a non-issue (runs off the standby).
- Architecturally an **automated snapshot** — same full-first-then-incremental mechanic as a manual snapshot — **plus transaction logs written to S3 every 5 minutes**. Transaction logs record the actual operations that change data.
- The daily snapshot plus 5-minute logs together enable **point-in-time recovery (PITR)** with **~5-minute granularity**, giving a theoretical **5-minute RPO**. 🎯
- **Retention is 0 to 35 days** (per instance). **0 disables automated backups** (and PITR); **35 is the maximum**. Anything older than the retention period is auto-removed.

> [!warning] Gotcha — PITR needs automated backups
> Only automated backups can do point-in-time recovery, because only they capture the 5-minute transaction-log stream. A **manual snapshot alone** restores to its **single fixed moment** — no PITR.

### Retention on Deletion

- When you delete the database you **can choose to retain automated backups** — but they **still expire** on the retention clock. Retaining them does **not** make them permanent.
- The **only** way to preserve an instance's contents **beyond the 35-day max** is to take a **final manual snapshot** at deletion. That snapshot is fully yours and must be **manually deleted**.

### Cross-Region Backup Replication

- RDS can **replicate backups to another Region** — both the **snapshots and the transaction logs**.
- ⚠️ **Not the default** — it must be **explicitly enabled** within automated backups, and **charges apply** for the cross-region copy and for storage in the destination Region.

### Restores

🎯 **A restore always creates a brand-new RDS instance** — true for both snapshot restores and automated-backup PITR. The new instance gets a **new endpoint address**, so you **must re-point applications** at it. A restore never overwrites the existing database.

| Restore source | Recovery point | RPO quality |
| --- | --- | --- |
| **Manual snapshot** | Fixed to the **single moment** the snapshot was taken | Often suboptimal unless taken right before failure |
| **Automated backup (PITR)** | **Any chosen point in time** (~5-min granularity) | Much better — pick minutes before failure |

**How PITR works:** RDS restores the **closest snapshot**, then **replays transaction logs** forward to your chosen time.

> [!warning] Gotcha — restore time is your RTO
> Restores are **not fast** — restoring a large database takes significant time, and that time **is your RTO**. Increasing backup frequency improves **RPO** (how much data you lose) but does little for RTO. To improve RTO, use a [read replica](#read-replicas) you can promote — backups improve RPO, replicas improve RTO.

## Data Security

RDS data security splits into **four separate concerns**. The exam likes to test whether you can tell them apart — especially authentication from authorization, and in-transit from at-rest encryption.

| Concern | What it covers |
| --- | --- |
| **Authentication** | How a user proves who they are when logging in |
| **Authorization** | What a logged-in user is allowed to do |
| **Encryption in transit** | Protecting data moving between client and RDS |
| **Encryption at rest** | Protecting data written to disk |

Authentication and authorization are the [identity-and-access](../concepts/identity-and-access.md) split applied to the database; in-transit protection is ordinary [TLS](../concepts/tls-https.md); at-rest protection leans on [KMS](kms.md) envelope encryption over [EBS](ebs.md).

### Encryption in Transit

- Supported across **all RDS engines**.
- Traffic between client and instance is encrypted with **SSL/TLS** (see [TLS/HTTPS](../concepts/tls-https.md)).
- Can be made **mandatory on a per-user basis** — the server rejects any non-TLS connection for that user.

> [!tip] Exam Tip
> Trigger words *"force / require SSL"* or *"encrypt the connection"* map to **per-user mandatory SSL/TLS**, available on every engine.

### Encryption at Rest

There are two mechanisms, and the whole game is knowing **who does the encrypting** — the host, or the database engine.

#### 1. KMS + EBS encryption (the default, host-side)

This is supported by default across the RDS engines and happens **below the database engine**:

- The **RDS host** (the server the instance runs on) performs encryption and decryption against the underlying [EBS](ebs.md) storage. As far as the engine is concerned it is writing **plaintext** — it has **no encryption awareness**; the host encrypts before data reaches the disk.
- It uses [KMS](kms.md): you pick a **KMS key** (the course calls it a CMK) — either a **customer managed key** or the **AWS managed key** (`aws/rds`). If you don't specify one at creation, RDS defaults to the AWS managed key.
- That key generates **data encryption keys (DEKs)** — standard [envelope encryption](kms.md) — which do the bulk **AES-256** work and are loaded onto the hosts as needed.
- **Coverage is uniform:** storage, logs, snapshots, and read replicas are all encrypted with the **same KMS key**.

> [!warning] Gotcha — encryption is a one-way, create-time-only decision
> Two linked rules the exam tests hard: **(1)** encryption **cannot be removed** once an instance is encrypted, and **(2)** it can **only be enabled at instance creation** — you can't add it in place to an existing unencrypted instance. The workaround for an existing instance: take a snapshot, **copy the snapshot with encryption enabled**, then restore from the encrypted copy. A read replica's encryption state must also **match its source** — no encrypted replica of an unencrypted instance, or vice versa.

#### 2. TDE — Transparent Data Encryption (engine-native)

- Only **Microsoft SQL Server** and **Oracle** support TDE.
- Encryption happens **inside the database engine itself**, not on the host. Because the engine encrypts data **before it leaves the instance**, the trust boundary shrinks — data is protected from the moment the engine writes it.

> [!warning] Gotcha — TDE is engine-restricted
> **MySQL, MariaDB, and PostgreSQL do NOT support TDE** — it only exists where the underlying engine ships it. Those three still get full at-rest protection via the **host-side KMS + EBS** path above; they just can't do engine-native TDE. Don't let "MySQL is encrypted" trick you into "MySQL supports TDE."

Edition nuances:

- **Oracle:** TDE requires the **Oracle Advanced Security** option on **Oracle Enterprise Edition**.
- **SQL Server:** RDS supports TDE on **SQL Server 2022 Standard and Enterprise**, **2019 Standard and Enterprise**, and **2017/2016 Enterprise**. On RDS, SQL Server TDE is enabled via an **option group** (the `TDE` option).

#### 3. Oracle + TDE via CloudHSM (strongest key control)

- **RDS Oracle** can additionally use **CloudHSM** for the TDE key material.
- CloudHSM is managed **entirely by you, with no key exposure to AWS**, enabling a trust chain that **does not involve AWS** — valuable for demanding regulatory scenarios.

> [!tip] Exam Tip
> *"AWS must have no access to the keys"* or *"no AWS in the chain of trust"* maps to **RDS Oracle + TDE + CloudHSM**. This is Oracle-specific — don't generalize it to other engines.

#### Diagram — KMS encryption vs TDE

![Amazon RDS KMS Encryption and TDE: on the left a CloudHSM appliance connects by a blue arrow to an Oracle instance, illustrating that with RDS Oracle the TDE keys can come from CloudHSM and remove AWS from the chain of trust. Two orange HOST boxes in the middle each run RDS instances (left host: Oracle plus a MySQL; right host: another MySQL), with a small key-lock icon showing DEKs resident on each host. Green double arrows between the MySQL engines and the EBS Volumes strip at the bottom are labelled data is encrypted and decrypted by the host — the KMS host-side path. A red double arrow between the two MySQL instances shows replica traffic is also encrypted with the same key. On the right, a KMS key icon sends red arrows loading DEKs onto the hosts as required, and a snapshot icon at the bottom-right shows snapshots of encrypted instances are also encrypted with the same key. Footer text reiterates TDE is native DB-engine encryption, with data encrypted before leaving the instance.](../assets/rds-kms-encryption-vs-tde.png)

The visual contrast to lock in: the **TDE path encrypts inside the engine** (Oracle, left); the **KMS path encrypts on the host** (the green arrows under the MySQL engines). Everything KMS-encrypted — storage, the cross-instance replica traffic, and the snapshot — uses the **same key**.

### IAM Authentication

**Normal RDS logins** use **local database users** with their own username and password. These are **not** IAM identities and live **outside AWS's control** — one local user is created when you provision the instance, and managing the rest is up to you.

**IAM authentication** lets an [IAM](iam.md) identity log in using a **short-lived AWS token instead of a password**. The mechanics:

1. On the RDS instance, create a **local DB user configured to authenticate via an AWS authentication token** (rather than a password).
2. Attach a **policy** to an **IAM user or role** (for example an [EC2](ec2.md) instance role) that **maps that IAM identity onto the local RDS database user**.
3. The identity calls **`GenerateDBAuthToken`** (CLI: `generate-db-auth-token`), which spans IAM and RDS.
4. That produces a **token valid for 15 minutes**, used in place of the password to log in as the mapped DB user.

IAM authentication only works **if you explicitly enable it** on the instance.

> [!warning] Gotcha — IAM auth is authentication ONLY, never authorization
> This is *the* exam trap for the topic. IAM gets the identity **in the door**; once inside, **permissions are still governed by the local DB user** in the engine (the SQL `GRANT`s on that user). IAM is never used to authorize what the user can do in the database.

#### Diagram — IAM authentication

![Amazon RDS IAM Authentication: on the left an RDS instance holds a local DB account configured to use an AWS authentication token. A green double arrow between RDS and a central human figure is labelled DBUser and Token — the successful authenticated login path where the user presents the DB username plus token. The human (an IAM user) and a chip icon (an EC2 role) are the IAM identities; each has a checklist (policy) above it, and a red arrow at top-right shows the policy maps the IAM identity onto the local RDS user. Blue arrows along the bottom show the generate-db-auth-token call flowing to IAM (red icon, right) which issues a token with 15-minute validity (key-and-clock icon) that becomes the credential used in the green login path. Top-left text states authorization is controlled by the DB engine and IAM is not used to authorize, only to authenticate.](../assets/rds-iam-authentication.png)

Three arrow colors summarize the flow: **green** = the authenticated DB login (DB user plus token); **blue** = token generation via IAM; **red** = the policy mapping the IAM identity onto the local DB user.

#### Practical mechanics

The piece most people miss is that **`GenerateDBAuthToken` itself needs AWS credentials to sign the token**. The SDK doesn't ask for them — it walks the **default credentials provider chain** and uses the first source that resolves, in roughly this order: JVM system properties (Java only), environment variables, web-identity token (EKS IRSA), the shared `~/.aws/credentials` profile, ECS container credentials, and finally the **EC2 instance profile via [IMDS](ec2.md)**.

The payoff for an app deployed on EC2: set **none** of the earlier sources, attach an **[IAM role to the instance](iam.md)**, and the chain falls through to IMDS and uses the role's **temporary, auto-rotating credentials**. The same binary resolves your local profile on a laptop and the instance role in production — **zero credential code, no long-lived database password anywhere**.

Two practical traps:

- **SSL/TLS is mandatory** for IAM auth — the token travels to the DB as a cleartext password, so without TLS the server rejects it. Always ship the RDS CA bundle.
- **Generate the token against the real DB instance endpoint**, not a custom Route 53 record, and **mint a fresh token per new connection** — the 15-minute validity only matters at the connection handshake, so connection pools must regenerate it when opening new physical connections.

> [!tip] Exam Tip
> *"Log in to RDS without a stored database password, using an AWS identity"* maps to **IAM authentication** (`GenerateDBAuthToken`, 15-minute token). If a question then asks *"does IAM control what the user can do in the database?"* the answer is **No** — authorization stays with the local DB user.

#### Quickest manual test — CLI token plus psql

Before wiring any application code, the fastest way to prove IAM auth works end-to-end is to mint a token with the AWS CLI and feed it to the native client. This is also how you connect from a [GUI client](#connecting-from-a-gui-client) that has no built-in IAM support. Prerequisites are the same as below (auth enabled on the instance, an `rds_iam` DB user, an `rds-db:connect` policy on your identity), plus the **RDS CA bundle** downloaded locally:

```bash
# Download the CA bundle once (used by every engine, every region)
curl -o /opt/certs/global-bundle.pem \
  https://truststore.pki.rds.amazonaws.com/global/global-bundle.pem
```

**PostgreSQL.** Mint the token, stash it in `PGPASSWORD`, and connect over TLS:

```bash
# 1. Sign a 15-minute token using the default credentials chain
export PGPASSWORD="$(aws rds generate-db-auth-token \
  --hostname appdb.abcdefghijkl.us-east-1.rds.amazonaws.com \
  --port 5432 \
  --region us-east-1 \
  --username app_iam_user)"

# 2. Present the token as the password; verify-full forces TLS + CA validation
psql "host=appdb.abcdefghijkl.us-east-1.rds.amazonaws.com \
      port=5432 dbname=appdb user=app_iam_user \
      sslmode=verify-full sslrootcert=/opt/certs/global-bundle.pem"
```

**MySQL.** The token is the password, but the client must be told to send it in cleartext (it is protected by the mandatory TLS, not by MySQL's normal challenge-response):

```bash
TOKEN="$(aws rds generate-db-auth-token \
  --hostname appdb.abcdefghijkl.us-east-1.rds.amazonaws.com \
  --port 3306 --region us-east-1 --username app_iam_user)"

mysql --host=appdb.abcdefghijkl.us-east-1.rds.amazonaws.com --port=3306 \
      --ssl-ca=/opt/certs/global-bundle.pem --enable-cleartext-plugin \
      --user=app_iam_user --password="$TOKEN"
```

Three things to notice, all of which generalize to every IAM-auth client:

- **The CLI uses the same default credentials chain as the SDK** ([Practical mechanics](#practical-mechanics)) — on a laptop it signs with your profile, on EC2 with the instance role. You are not passing a database password anywhere.
- **Pass the token via an env var, not inline**, because it contains URL-encoded characters (`%`, `/`, `+`) that the shell would otherwise mangle.
- **The 15-minute clock only matters at the handshake.** Once `psql`/`mysql` is connected the session stays open indefinitely; the expiry just means you must mint a fresh token for the *next* connection — which is exactly why pooled apps need the per-connection refresh covered next.

#### Connecting from a GUI client

GUI clients fall into two camps. **DataGrip** (Authentication: **AWS IAM**) and **DBeaver** (auth model: **AWS IAM**) generate the token for you from a chosen AWS profile and region on every connect, so they handle the 15-minute expiry transparently — you only configure the profile, region, and TLS (point them at `global-bundle.pem`). Clients with **no native support** (pgAdmin, MySQL Workbench) require pasting a CLI-generated token (above) into the password field and re-pasting it whenever it expires. In all cases TLS is mandatory, exactly as in the CLI flow.

#### Wiring IAM auth into a JDBC connection pool (PostgreSQL)

> [!note] Hands-on detail, not exam material
> SDK and JDBC wiring is **not** tested by the SAA exam — it tests the concepts above. This section is for building real applications. The examples use **PostgreSQL**; MySQL differs in the DB-user setup and TLS handling (noted inline).

**Prerequisites (engine-specific).** Three things must be true before any code connects:

1. **IAM DB authentication enabled** on the instance (a toggle, no downtime).
2. **A local DB user bound to IAM auth.** On PostgreSQL this is the **`rds_iam` role** — granting it forces token authentication for that user (MySQL instead uses `IDENTIFIED WITH AWSAuthenticationPlugin AS 'RDS'`):

   ```sql
   -- PostgreSQL
   CREATE USER app_iam_user;
   GRANT rds_iam TO app_iam_user;                       -- forces IAM-token auth
   GRANT SELECT, INSERT, UPDATE, DELETE                 -- authorization = the engine's job, never IAM's
     ON ALL TABLES IN SCHEMA public TO app_iam_user;
   ```

3. **An IAM policy** allowing `rds-db:connect` on the specific DB-user resource, attached to the identity (on EC2, the instance role):

   ```json
   {
     "Version": "2012-10-17",
     "Statement": [{
       "Effect": "Allow",
       "Action": "rds-db:connect",
       "Resource": "arn:aws:rds-db:us-east-1:111122223333:dbuser:db-ABCDEFGHIJKL01234/app_iam_user"
     }]
   }
   ```

There are **two ways** to feed the rotating token into a connection pool.

**Approach 1 — AWS Advanced JDBC Wrapper (recommended, least code).** The wrapper is a driver (`software.amazon.jdbc.Driver`) that wraps the real PostgreSQL driver and, via its **`iam` plugin**, signs a fresh 15-minute token for **every new physical connection** using the [default credentials provider chain](#practical-mechanics) — so the pool-expiry problem is handled for you and there is **no password and no callback** to write. Two URL details make it work: the **`jdbc:aws-wrapper:postgresql://`** scheme (the `aws-wrapper:` prefix is what activates the wrapper) and **`wrapperPlugins=iam`**.

```yaml
spring:
  datasource:
    druid:
      driver-class-name: software.amazon.jdbc.Driver
      url: jdbc:aws-wrapper:postgresql://appdb.abcdefghijkl.us-east-1.rds.amazonaws.com:5432/appdb?wrapperPlugins=iam&ssl=true&sslmode=verify-full&sslrootcert=/opt/certs/global-bundle.pem
      username: app_iam_user
      # no password, no password-callback — the iam plugin mints and refreshes the token
```

Maven needs `software.amazon.jdbc:aws-advanced-jdbc-wrapper` plus `org.postgresql:postgresql`. As a bonus, the same wrapper offers other plugins (fast failover, read/write splitting) that pair naturally with [Multi-AZ cluster](#multi-az-cluster-deployment) endpoints.

**Approach 2 — manual token via a pool callback (explicit control, no extra driver).** Use the **standard PostgreSQL driver** (`org.postgresql.Driver`) and a hook that mints a token with **`RdsUtilities`** each time the pool opens a physical connection. Druid exposes a clean hook for this (`DruidPasswordCallback`); other pools need their own per-connection refresh mechanism (e.g. HikariCP has no native callback, so you typically wrap the `DataSource` or set the password before each new connection).

```java
import com.alibaba.druid.pool.DruidPasswordCallback;
import software.amazon.awssdk.auth.credentials.DefaultCredentialsProvider;
import software.amazon.awssdk.regions.Region;
import software.amazon.awssdk.services.rds.RdsUtilities;

import java.util.Properties;

public class RdsIamPasswordCallback extends DruidPasswordCallback {

    // No explicit credentials → DefaultCredentialsProvider walks the chain;
    // on EC2 it lands on the instance-profile role via IMDS.
    private final RdsUtilities rds = RdsUtilities.builder()
            .credentialsProvider(DefaultCredentialsProvider.create())
            .region(Region.US_EAST_1)
            .build();

    private static final String HOST = "appdb.abcdefghijkl.us-east-1.rds.amazonaws.com";
    private static final int    PORT = 5432;            // PostgreSQL
    private static final String USER = "app_iam_user";

    @Override
    public void setProperties(Properties properties) {
        super.setProperties(properties);
        // Fresh 15-min token per physical connection → pooled connections stay valid.
        String token = rds.generateAuthenticationToken(b -> b
                .hostname(HOST).port(PORT).username(USER));
        setPassword(token.toCharArray());
    }
}
```

```yaml
spring:
  datasource:
    druid:
      driver-class-name: org.postgresql.Driver
      url: jdbc:postgresql://appdb.abcdefghijkl.us-east-1.rds.amazonaws.com:5432/appdb?ssl=true&sslmode=verify-full&sslrootcert=/opt/certs/global-bundle.pem
      username: app_iam_user
      # password supplied dynamically by the callback below
      password-callback-class-name: com.example.config.RdsIamPasswordCallback
```

| | Approach 1 — JDBC Wrapper | Approach 2 — manual callback |
| --- | --- | --- |
| Driver class | `software.amazon.jdbc.Driver` | `org.postgresql.Driver` |
| URL scheme | `jdbc:aws-wrapper:postgresql://…` | `jdbc:postgresql://…` |
| Token generation | wrapper `iam` plugin (automatic) | your code via `RdsUtilities` |
| Refresh on new pool connection | automatic | you mint per connection |
| Code to write | none | a callback class |
| Extra capabilities | failover / read-write-split plugins | none — just the token |

> [!warning] Gotcha — TLS is mandatory, and PostgreSQL handles the CA bundle more simply
> IAM auth requires TLS (the token is sent as a cleartext password). PostgreSQL's JDBC driver reads the **RDS CA PEM bundle directly** via `sslmode=verify-full&sslrootcert=/opt/certs/global-bundle.pem` — **no JKS truststore needed**. (MySQL Connector/J instead needs the bundle imported into a **JKS** file referenced by `trustCertificateKeyStoreUrl`.) Both approaches above also assume the token is generated against the **real instance endpoint**, not a custom Route 53 record.

### Security Exam Quick-Reference

| Trigger / scenario | Answer |
| --- | --- |
| Encrypt the client-to-RDS connection, per user | **SSL/TLS in transit** (all engines, can be mandatory) |
| Default at-rest encryption, host does the work | **KMS + EBS** (KMS key generates DEKs; engine unaware; AES-256) |
| Engine-native at-rest encryption | **TDE** — **SQL Server and Oracle only** (not MySQL/MariaDB/PostgreSQL) |
| "AWS must have no key access" / no AWS in the chain of trust | **RDS Oracle + TDE + CloudHSM** |
| Log in without a DB password, using an AWS identity | **IAM authentication** (`GenerateDBAuthToken`, 15-min token) |
| "Does IAM control what the user can do in the DB?" | **No** — authorization stays with the local DB user |
| Storage, logs, snapshots, replicas all encrypted | KMS at-rest, **same key** |
| Add encryption to an existing unencrypted instance | Snapshot, **copy snapshot with encryption**, restore |

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
| **RDS Custom** | Middle ground between RDS and DB-on-EC2; **SQL Server + Oracle only**; OS/engine access via **SSH/RDP/Session Manager**; resources (EC2/EBS/S3) **visible in your account**; pause automation to customize |
| Aurora ≠ RDS | Separate product; different (shared-cluster) storage architecture |
| Multi-AZ replication | **Synchronous**, same region, different AZ, standby identical (and not readable) |
| Multi-AZ **instance** mode | 1 primary + 1 passive standby; failover = **DNS repoint**, **60–120 s**; all engines incl. SQL Server/Oracle |
| Multi-AZ **cluster** mode | 1 writer + **2 readable** readers; semi-synchronous quorum; failover **~35 s / 25–75 s**; Graviton + NVMe; **MySQL/PostgreSQL only** |
| Multi-AZ endpoints (cluster) | **Cluster** (writer), **Reader** (read scaling), **Instance** (testing only — no failover) |
| Read replicas | **Asynchronous**, same- or **cross-region**, read scaling + DR; **own endpoint**, app must be changed, **no automatic failover** |
| Read replica count | **5** per instance (course) / **up to 15** (current AWS docs); **cascading** allowed but **lag compounds** |
| Read replica DR limit | Near-zero **RPO and RTO** via fast **promotion** — but only for **failure**, **not data corruption** (replica holds the corrupted data) |
| Promotion | Replica is read-only until promoted; once promoted it is a standalone read/write instance and **stops replicating** |
| Storage | Dedicated **EBS per instance** (unlike Aurora) |
| Backups | Go to **S3** (AWS-managed, **invisible in your S3 console**); regionally resilient; Multi-AZ instance backups are standby-sourced for MariaDB/MySQL/Oracle/PostgreSQL, but SQL Server briefly suspends I/O |
| Manual snapshot | Instance-level (all DBs); **never expires**, survives instance deletion, manual delete only; 100/Region |
| Automated backup | Daily snapshot **+ transaction logs every 5 min**; retention **0–35 days** (0 = disabled) |
| PITR | Needs **automated backups** (the 5-min logs); ~**5-min RPO**; manual snapshot alone restores only to its fixed moment |
| Keep data past 35 days on deletion | Take a **final manual snapshot** (retained automated backups still expire) |
| Cross-region backup replication | **Opt-in**, not default; copies snapshots **and** logs; extra cost |
| Restore | Always a **new instance + new endpoint** — re-point apps; restore time is your **RTO** |
| Subnet group | List of subnets RDS may use; required at launch; Multi-AZ → primary + standby in different AZs (random unless specified) |
| Billing | Resource-allocation (like EC2), hourly rate **billed per second**, not usage-based |

> [!warning] Gotchas
>
> - **"Database as a service" is a deliberately wrong characterisation of RDS** — you receive a *server*, hence multiple DBs per instance.
> - **RDS instances in public subnets *can* get public IPs** — possible, but a security anti-pattern; know it exists as an option.
> - **Same-AZ private-IP traffic is free; cross-AZ traffic is charged** — relevant to both EC2-DB and RDS Multi-AZ designs.
> - **EC2-hosted DB = single AZ** — instance plus EBS volume; an AZ failure can take DB access down.
> - **Don't confuse RDS storage with Aurora storage** — RDS = dedicated EBS per instance; Aurora differs.
> - **RDS backup buckets are invisible in your S3 console** — AWS-managed; find them only in the RDS console.
> - **"Retain automated backups on delete" still expires** — only a **final manual snapshot** is permanent.
> - **PITR is automated-backups-only** — a manual snapshot restores to its single fixed moment, not an arbitrary time.
> - **A restore is always a new instance + new endpoint** — it never overwrites the source; re-point apps.

## References

- Cantrill SAA-C03 study notes — *RDS Architecture (and Databases on EC2)*, *RDS Multi-AZ (Instance and Cluster)*, *RDS Backups, Snapshots & Restore*, *RDS Data Security*
- [AWS — What is Amazon RDS?](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Welcome.html) (current RDS engine list including IBM Db2)
- [AWS — Working with DB instance read replicas](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_ReadRepl.html) and [RDS quotas](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_Limits.html) (asynchronous replication, Db2/Oracle non-readable exceptions, read-replica quota)
- [AWS — Creating a DB snapshot](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_CreateSnapshot.html) (Single-AZ I/O suspension and SQL Server Multi-AZ backup exception)
- [AWS — Choose the right Amazon RDS deployment option: Single-AZ, Multi-AZ instance, or Multi-AZ DB cluster](https://aws.amazon.com/blogs/database/choose-the-right-amazon-rds-deployment-option-single-az-instance-multi-az-instance-or-multi-az-database-cluster/) (semi-synchronous quorum replication, engine restriction, failover-timing ranges, backup mechanics, Single-AZ baseline)
- [AWS — Amazon RDS Custom](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-custom.html) (RDS Custom supports Oracle Database and Microsoft SQL Server)
- [AWS — Encrypting Amazon RDS resources](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Overview.Encryption.html) (KMS + EBS host-side encryption, AES-256, create-time-only restriction, snapshot-copy workaround, same-key coverage)
- [AWS — SQL Server Transparent Data Encryption (TDE) option](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Appendix.SQLServer.Options.TDE.html) and [RDS security features](https://aws.amazon.com/rds/features/security/) (engine/edition restrictions, option-group mechanism)
- [AWS — IAM database authentication](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/UsingWithRDS.IAMDBAuth.html) and the [SDK default credentials-provider chain](https://docs.aws.amazon.com/sdk-for-java/latest/developer-guide/credentials-chain.html) (token mechanics, 15-min validity, mandatory TLS, IMDS-on-EC2 resolution)
- [AWS Advanced JDBC Wrapper — IAM authentication plugin](https://github.com/aws/aws-advanced-jdbc-wrapper) and [`RdsUtilities.generateAuthenticationToken`](https://sdk.amazonaws.com/java/api/latest/software/amazon/awssdk/services/rds/RdsUtilities.html) (the two JDBC connection-pool wiring approaches; PostgreSQL `rds_iam` role and `sslmode=verify-full` CA handling)
- Related: [EC2](ec2.md) · [EBS](ebs.md) · [VPC](vpc.md) · [KMS](kms.md) · [IAM](iam.md) · [DMS](dms.md) · [Identity and Access](../concepts/identity-and-access.md) · [TLS/HTTPS](../concepts/tls-https.md) · [Database Transaction Models (ACID/BASE/CAP)](../concepts/db-transaction-model.md) · [Resource Scope](../concepts/resource-scope.md)
