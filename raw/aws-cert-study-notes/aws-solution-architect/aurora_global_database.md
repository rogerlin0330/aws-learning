# Aurora Global Database

## Overview

**Aurora Global Database** provides **global-level replication** using Aurora — from a **primary (master) region** out to multiple **secondary AWS regions**. It's designed for cross-region disaster recovery, business continuity, and global read scaling.

> 🎯 For the SAA exam this is an **awareness-level** topic. It won't feature heavily, but you need to recognise _what it does_ and _when you'd choose it_. Know the architecture and the two main use cases.

---

## Architecture Walkthrough

![[aurora-global-database.png]]

The diagram shows a global database spanning **two regions**, with a single **APPLICATION** (top) interacting with both.

**Primary region** (`us-east-1` on the left) — behaves like a normal Aurora cluster:

- **1 read/write instance** (red box) — the application sends **writes** here (bidirectional red arrow), and can also read from it.
- **Up to 15 read-only replicas** (green stack) — the application reads from these (green arrow up).
- Both instance types sit on top of the shared **Cluster Volume** (the pink storage layer at the bottom).

**Secondary region** (`ap-southeast-2` / Sydney on the right) — read-only during normal operation:

- **Up to 16 read-only replicas** (green stack) — the application performs **read-only** operations here.
- **No read/write instance.** Because there's no write primary consuming one of the slots, _all 16_ instances can be read replicas (vs. 15 in the primary, where 1 slot is the writer).
- Also sits on top of its own copy of the Cluster Volume.

**Replication (center, `~1s Replication`):**

- Occurs at the **storage layer** (cluster volume → cluster volume), _not_ the database engine.
- Typically **≤ 1 second** from primary to all secondaries.
- **One-way** (primary → secondary), **not bidirectional**.

> 📚 **Reading order recap:** context = one app, two regions → primary region mirrors a standard Aurora cluster (1 writer + up to 15 readers) → secondary region is entirely read-only (up to 16 readers) → the horizontal arrow is storage-layer replication at ~1s. The footer labels tell you which region is primary vs. secondary.

---

## Key Mechanics

- **Replication is at the storage layer**, so it has **no impact on database performance** — no additional CPU/engine work is needed on the instances to perform replication.
- **Replication latency** is typically **~1 second or less**, primary → all secondaries.
- **One-way only** (primary → secondary). Secondary clusters are read-only until promoted.
- **Instance counts:**
    - Primary region: 1 read/write + up to 15 read replicas = **16 instances total**.
    - Each secondary region: **up to 16 read replicas** (all read-only) — the writer slot is freed because there's no write primary.
- On disaster, a **secondary cluster can be promoted** to become a full read/write primary.

> 📚 **Synchronous vs. asynchronous — the two "storage-layer" legs are not the same** (beyond the transcript):
> 
> - **In-region (across AZs) = synchronous.** Writes to the primary instance are synchronously replicated across AZs to the **6 copies of the cluster volume spread over 3 AZs**. The write is acknowledged once a **quorum of 4 of 6** copies confirm → zero data loss within the region.
> - **Cross-region (to secondaries) = asynchronous.** The primary region asynchronously replicates to each secondary region (~1s latency). Writes don't wait for secondaries to confirm.
> 
> **Why the split:** synchronous is cheap within a region (AZs are close, low latency), but doing it synchronously across regions would add too much write latency. The async cross-region leg is exactly why Global Database has a **small non-zero RPO (~1s)** rather than zero. Source: [Aurora High Availability docs](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Concepts.AuroraHighAvailability.html).

---

## When to Use It

**1. Cross-region disaster recovery (DR) & business continuity** 🎯

- Stand up secondary region(s) ahead of time; if a disaster takes out an entire region, **promote a secondary cluster to primary** (read/write).
- Because replication is ~1s, both **RPO** (data loss) and **RTO** (recovery time) are **very low** on a cross-region failover.

**2. Global read scaling / low-latency international reads** 🎯

- Place secondary regions close to international customers so the local application tier performs **read operations against the nearest secondary cluster**.
- Low latency → better performance for globally distributed users.

> 💡 Exam trigger mapping: _"cross-region DR," "region-wide outage," "business continuity," "low RPO/RTO across regions," "low-latency reads for global/international users"_ → **Aurora Global Database**.

---

## ⚠️ Exam Traps & Gotchas

- ⚠️ **Replication is one-way, not bidirectional.** Writes only happen in the primary region during normal operation.
- ⚠️ **Storage-layer replication** — don't confuse it with engine-based replication; it's why there's **no performance impact**.
- ⚠️ **Secondary regions are entirely read-only** until a member is promoted.
- ⚠️ **16 replicas in a secondary vs. 15 in the primary** — the difference is the freed-up writer slot, not a different limit. Easy to misremember.
- ⚠️ **Sync vs. async depends on the leg.** In-region across-AZ replication is **synchronous** (6 copies / 3 AZs, 4-of-6 quorum); **cross-region** replication is **asynchronous**. Don't answer "synchronous" for the cross-region part — that's why RPO is ~1s, not zero.

### ⚠️ Discrepancy: max secondary regions (5 vs. 10)

- **Course transcript & diagram (older):** maximum of **5 secondary regions**.
- **Current AWS (as of May 21, 2025):** AWS raised the limit to **up to 10 secondary regions**. <cite index="2-1">This increases the number of secondary Regions that can be added to a global cluster from the previously supported limit of up to 5 to up to 10.</cite> Source: [AWS What's New](https://aws.amazon.com/about-aws/whats-new/2025/05/amazon-aurora-global-database-support-10-secondary-region-clusters/) and [AWS Database Blog](https://aws.amazon.com/blogs/database/amazon-aurora-global-database-introduces-support-for-up-to-10-secondary-regions/).

> 🎯 The transcript even notes "this is likely to change" — and it did. **The exam may still use the older "5" figure**, so know both. If a question implies the current limit, it's **10**; if it references the classic Cantrill number, it's **5**.

> 📚 **Related current facts worth knowing** (beyond the transcript): each secondary region still supports **up to 16 read-only DB instances**; a global cluster needs at least 1 secondary region (min. 2 regions total); AWS quotes an effective **RPO of ~1 second** and **RTO of under 1 minute** on failover. AWS also distinguishes **switchover** (planned, no data loss) from **failover** (unplanned recovery). Source: [Aurora Global Database docs](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database.html).

---

## Exam Quick Reference

|Concept|Value / Behaviour|Trigger Keywords|
|---|---|---|
|Purpose|Global replication, primary → secondary regions|"global database," "multi-region Aurora"|
|Primary region|1 read/write + up to 15 read replicas|"writer + readers"|
|Secondary region|Read-only, up to 16 read replicas|"read-only region," "16 replicas"|
|Max secondary regions|**5** (transcript) / **10** (current AWS, May 2025)|"how many secondary regions"|
|Replication location|**Storage layer** (cluster volume)|"no performance impact," "no CPU"|
|Replication latency|**~1 second or less**|"sub-second," "low lag"|
|Direction|**One-way** (primary → secondary)|"not bidirectional"|
|DR failover|Promote secondary → primary (read/write)|"region outage," "low RPO/RTO," "business continuity"|
|Read scaling|Local reads from nearest secondary|"low latency," "global/international customers"|