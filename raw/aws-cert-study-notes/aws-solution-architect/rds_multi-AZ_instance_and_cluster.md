# RDS Multi-AZ

## Overview

RDS provides **high availability (HA)** through Multi-AZ deployments. Historically there was only one option ("Multi-AZ"), but RDS now offers two distinct architectures with different benefits and trade-offs:

- **Multi-AZ instance deployment** — the original, single standby that _cannot_ be used for reads or writes.
- **Multi-AZ cluster deployment** — one writer + two **readable** standbys, faster failover, and read scaling.

> 🎯 Both modes only span **availability zones within a single region**. Multi-AZ is _not_ a cross-region or DR mechanism — it's an in-region HA mechanism.

> ⚠️ **Editorial note (verify):** Don't confuse Multi-AZ _cluster_ mode with **Amazon Aurora**. They look superficially similar but differ in storage model (cluster mode keeps **local per-instance storage**; Aurora uses a shared cluster storage layer) and in scale (cluster mode caps at **2 readers**; Aurora supports more). Cantrill flags this comparison deliberately to prepare for the Aurora video.

---

## Multi-AZ Instance Deployment

![[rds-multi-AZ-instance-mode.png]]

### How to read this diagram

- **Left (green band):** the on-premises / client side. A single client device sends all traffic.
- **Middle arrow ("Database CNAME"):** the DNS name clients connect through. By default it resolves to the **primary** instance. _(The IP/name shown is illustrative — the point is that access is always indirected through this CNAME, never hard-wired to an instance.)_
- **Coloured vertical bands (AZA / AZB / AZC):** separate availability zones inside one region (`us-east-1` here is illustrative).
- **Top row of each AZ band:** the database instances — **Primary** in AZA, **Standby** in AZB, and an empty AZC (no instance lives there in instance mode).
- **Green arrow between Primary → Standby:** synchronous storage-level replication.
- **Bottom row (S3 buckets, dotted red arrows):** backups/snapshots flowing **from the standby** into S3, then replicated across AZs in-region.

### Mechanics

- One **primary** instance holds your databases. When Multi-AZ is enabled, it replicates **synchronously** to a **standby replica** in another AZ.
- Replication here is at the **storage level** — less efficient than cluster mode (more below).
- The exact replication mechanism depends on the engine, but it's abstracted away:
    - **MariaDB, MySQL, Oracle, PostgreSQL** → Amazon failover technology.
    - **Microsoft SQL Server** → SQL Server database mirroring or Always-On availability groups.
- All access (reads **and** writes) goes to the **primary** via the database **CNAME**. The standby serves **no traffic** — not even reads. Its only job is to wait for a failover.
- **Backups/snapshots are taken from the standby**, so there's **no I/O load on the primary** during backup.

### Synchronous replication

Data is written to the primary, then **immediately replicated to the standby before being viewed as committed**.

- 🎯 This gives an effective **RPO of zero** on failover — no committed data is lost. _(Editorial — confirmed by AWS blog.)_

### Failover

![[rds-multi-AZ-instance-failover.png]]

#### How to read this diagram (vs. the steady-state diagram above)

This is the **same architecture after a failover** — compare it to the first instance-mode diagram to see what changed:

- **The CNAME arrow now bends across to AZB:** the pink client-traffic line no longer terminates at AZA. It routes through the **Database CNAME**, which has been **repointed to the instance in AZB** — the former standby, now the **new primary**.
- **AZA's instance is no longer the target:** what was the primary is now out of the serving path (failed, or being patched/maintained).
- **Everything else is unchanged:** still one CNAME indirection, still synchronous replication and S3 backups in the same positions. The _only_ moving part on failover is **which instance the CNAME resolves to** — which is exactly why failover speed is gated by **DNS**, not by data movement (the standby already had a synchronous copy).

> 🎯 The takeaway the two diagrams teach together: failover in instance mode = **a DNS repoint**, not a data-recovery operation. The data was already there synchronously; the 60–120 s cost is DNS propagation + client caching.

#### Mechanics

RDS detects primary failure and **fails over automatically** (can also be triggered manually for testing/maintenance). On failover:

- The **CNAME is repointed** from the old primary to the standby, which is promoted to new primary.
- Because it's a **DNS change**, failover typically takes **60–120 seconds**.
- 🎯 Reduce the outage window by **removing DNS caching** in the application for that name, so the app picks up the new target as soon as failover completes.

Failover triggers include:

- AZ outage / loss of availability in the primary AZ
- Loss of network connectivity to primary
- Primary instance (compute) failure
- Storage failure on primary
- Manual failover
- Instance type change
- Software patching _(fail over → patch the now-idle instance → fail back)_

### Key points

- Synchronous replication primary → standby.
- **Not** in the free tier (extra cost of the standby).
- **Exactly one** standby replica.
- Standby **cannot** serve reads or writes.
- Failover: **60–120 seconds**.
- Same-region, different-AZ only.

---

## Multi-AZ Cluster Deployment

![[rds-multi-AZ-cluster-mode.png]]

### How to read this diagram

- **Left (green band):** the client side — note there are now **three client devices**, reflecting that reads can be spread across multiple targets.
- **Blue arrow ("Writes => Writer"):** all writes go to the single **Writer** instance (AZA).
- **Pink arrows ("Reads <= Writer" / "Reads <= Reader"):** reads can be served by **either** the writer **or** a reader.
- **Top/centre instances:** one **Writer** (AZA) and two **Readers** (AZB, AZC), each in its own AZ.
- **Green arrows ("Synchronous Replication To Readers"):** the writer replicates to both readers. _(See editorial flag below — AWS documents this as semi-synchronous/quorum, not fully synchronous.)_
- **Right-hand callouts:** the three endpoint types (Cluster, Reader, Instance) — see below.

### Mechanics

- One **writer** + **two readers**, each in a different AZ.
    - 🎯 Exactly **two** readers in cluster mode — no more, no fewer. (Aurora allows more — a key exam distinction.)
- The **writer** behaves like the primary: handles writes **and** reads.
- The **readers are usable** (unlike the instance-mode standby) — but for **read operations only**.
    - Using readers requires **application support**: the app must know to send writes to the writer and reads to the readers.
    - 🎯 This is what lets cluster mode **scale read workloads** — instance mode cannot.
- Each instance still has its **own local storage** (contrast with Aurora's shared storage).

### Commit / replication

- A write is viewed as **committed once at least one reader confirms it has written the data** — at which point it is resilient across multiple AZs.

> ⚠️ **Editorial flag — source discrepancy (verified against AWS):** Cantrill's diagram labels this **"synchronous replication."** Official AWS documentation describes Multi-AZ DB cluster replication as **semi-synchronous** (engine-native), using a **two-of-three quorum**: a commit needs acknowledgement from **at least one** of the two standbys. Data is then copied **asynchronously** from local SSD to the attached EBS volumes. For exam purposes, "commit acknowledged by ≥1 reader" is the safe mental model; if a question hinges on the exact term, AWS uses **semi-synchronous / quorum**. 📚 [AWS blog](https://aws.amazon.com/blogs/database/choose-the-right-amazon-rds-deployment-option-single-az-instance-multi-az-instance-or-multi-az-database-cluster/)

### Endpoints (cluster mode)

|Endpoint|Points at|Used for|
|---|---|---|
|**Cluster endpoint**|The **writer**|Reads, writes, administration _(analogous to the database CNAME in instance mode)_|
|**Reader endpoint**|Any **available reader** (and in some cases the writer too)|Read scaling — apps balance reads across readers via this endpoint|
|**Instance endpoint**|One **specific instance** (every instance has one)|**Testing / fault-finding only** — does **not** fail over, so don't use in production|

### Hardware & performance

- Runs on **faster hardware**: **Graviton** architecture + **local NVMe SSD** storage.
- Writes hit **local super-fast SSD first**, then flush through to **EBS** — combining local-storage speed with EBS durability/resilience.
- 🎯 Readers can **scale read operations** (offloading the writer) — highest performance of any RDS mode short of Aurora.

### Failover

- Replication uses **transaction logs** → more efficient, enables faster failover.
- Failover can occur in **as little as ~35 seconds** + time to apply transaction logs to the reader being promoted.
    - ⚠️ **Editorial (verified):** The AWS blog gives the typical range as **25–75 seconds** for cluster mode (depends on replica lag). Cantrill's "~35s" is the optimistic floor. Either way, **much faster than instance mode's 60–120s**, partly because the **crash-recovery step is eliminated**.

---

## Instance vs. Cluster — Comparison

> 🎯 = exam trigger keyword. Some rows reconcile Cantrill's framing with official AWS wording (flagged inline).

|Feature|Multi-AZ **Instance**|Multi-AZ **Cluster**|
|---|---|---|
|Topology|1 primary + **1 standby** (2 AZs)|1 writer + **2 readable standbys** (3 AZs)|
|🎯 Standby usable for reads?|**No** — passive, failover only|**Yes** — readers serve read-only traffic|
|🎯 Read scaling?|**No** (need a separate read replica)|**Yes** via reader endpoint|
|Write access|Primary only|Writer only|
|Replication|**Synchronous**, storage-level|**Semi-synchronous** (quorum, ≥1 reader ack), transaction-log-based ⚠️_Cantrill says "synchronous"_|
|🎯 Commit latency|Higher than Single-AZ|**Up to 2× faster** write commits than instance mode|
|🎯 Failover time|**60–120 s** (DNS change)|**~35 s** floor / **25–75 s** typical (no crash recovery)|
|RPO on failover|**Zero** (synchronous)|Effectively zero (quorum guarantees commit on ≥1 standby)|
|Storage|EBS|**Local NVMe SSD → flushed to EBS**, per-instance|
|Hardware|Standard|**Graviton**|
|🎯 Supported engines|MariaDB, MySQL, Oracle, PostgreSQL, **SQL Server**|**MySQL & PostgreSQL only** 📚|
|Backups/snapshots|Taken from **standby** (no primary I/O hit)|EBS crash-consistent snapshot from primary, no I/O suspension 📚|
|Access pattern|Single **database CNAME**|**Cluster / Reader / Instance** endpoints|
|Free tier|No|No|
|Region scope|Same region, multi-AZ|Same region, multi-AZ|

📚 Engine restriction and backup/snapshot mechanics per the [AWS deployment-options blog](https://aws.amazon.com/blogs/database/choose-the-right-amazon-rds-deployment-option-single-az-instance-multi-az-instance-or-multi-az-database-cluster/).

---

## Bonus: Single-AZ (baseline, from AWS blog)

> ⚠️ **Editorial — not in the transcript**, added from the AWS blog for completeness and contrast.

A **Single-AZ instance** is one managed instance on EBS, no standby. Implications:

- **No automatic failover** during an AZ outage (nothing to fail over to).
- **RPO ~5 minutes** (driven by the interval for copying transaction logs to S3).
- **RTO** can range from minutes to hours.
- You _can_ attach read replicas (same- or cross-region) for read workloads and promote one manually after a primary failure.
- Fine for **dev/test**; **not** appropriate for production HA.

📚 [AWS deployment-options blog](https://aws.amazon.com/blogs/database/choose-the-right-amazon-rds-deployment-option-single-az-instance-multi-az-instance-or-multi-az-database-cluster/)

---

## Exam Traps & Gotchas

- ⚠️ **"Multi-AZ standby serves reads"** → **FALSE for instance mode.** Only **cluster mode** readers (or a separately created **read replica**) can serve reads. A classic distractor.
- ⚠️ **"Multi-AZ for disaster recovery / cross-region"** → No. Multi-AZ is **same-region HA**. Cross-region needs **read replicas** (or other DR tooling).
- 🎯 **Read scaling** keyword → points to **read replicas** or **Multi-AZ cluster**, _never_ Multi-AZ **instance**.
- 🎯 **"Faster failover" / "lower commit latency" / "readable standby"** → **Multi-AZ cluster**.
- 🎯 **SQL Server / Oracle + Multi-AZ** → must be **instance** mode; **cluster mode is MySQL/PostgreSQL only**.
- ⚠️ **DNS caching** is the enemy of fast failover in **instance** mode — disable app-side caching of the CNAME to minimise the 60–120 s window.
- 🎯 **Instance endpoint** in cluster mode = **testing/fault-finding only** (no failover); production traffic uses **cluster** (writes) and **reader** (reads) endpoints.
- ⚠️ **Synchronous vs. semi-synchronous:** Cantrill calls cluster replication "synchronous"; AWS calls it **semi-synchronous (quorum)**. Know both framings — answer to the wording in the question.