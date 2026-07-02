# RDS Aurora Serverless

## Overview

**Aurora Serverless is to Aurora what Fargate is to ECS.** It provides a version of the Aurora database product where you don't statically provision database instances of a certain size, and you don't manage those instances. It's another step toward a database-as-a-service product, removing the admin overhead of managing individual database instances.

From this point on, the Aurora product covered so far in the course is referred to as **Aurora Provisioned**, versus **Aurora Serverless**.

Key mechanics:

- You **still create a cluster**, but Aurora Serverless uses **ACUs (Aurora Capacity Units)** instead of provisioned instances.
- A **capacity unit** represents a certain amount of compute and a corresponding amount of memory.
- For a cluster you set **minimum and maximum ACU values**, and Aurora Serverless scales between them by adding or removing capacity based on load.
- It can scale down to **zero and be paused**, at which point you're billed **only for the storage** the cluster consumes.
- **Billing is per-second**, based on the resources you use.
- Provides the **same resilience as Aurora Provisioned**: cluster storage replicated across **six storage nodes across multiple availability zones**.

> [!WARNING] ⚠️ Version discrepancy — this note describes the _legacy_ product (Aurora Serverless v1) The transcript and diagram describe **Aurora Serverless v1** architecture — the "warm pool" of ACUs, the shared "proxy fleet," and scale-to-zero/auto-pause behavior are all **v1** characteristics.
> 
> The current shipping product is **Aurora Serverless v2** (GA since 2022), which is architected differently. **The exam has historically tested v1-style concepts**, and the instructor explicitly says this topic isn't yet heavily featured — but be aware that current AWS documentation and newer exam questions may reflect v2. See the **v1 vs v2** section below. `📚`

---

## Diagram Walkthrough

![[aurora-serverless-architecture.png]]`

Reading the diagram context → main objects → footer:

**Top (context):** A user interacts with an **Application** (red box). The application connects _into the cluster_, but never directly to the compute — this is the crucial architectural point.

**Middle-upper — the Proxy Fleet (red bordered strip):** A row of proxy instances (with a dotted line indicating there are many more). The application's connections land here. Any proxy fleet instance can be used, and it **brokers a connection** between the application and the ACUs. This fleet is **managed by AWS** and operates transparently to you.

**Middle — the Aurora Serverless Cluster (orange border):** Spans three availability zones (**AZ-A, AZ-B, AZ-C**).

- The tan **ACU** band across the top is where compute lives. Note the ACUs are drawn as differently-sized database icons — a smaller one in AZ-B and a larger one in AZ-C — illustrating that capacity is dynamically sized. `📚` _(The two sizes are illustrative of scaling, not a fixed layout.)_
- The pink band at the bottom holds the **cluster storage** (drive icons in each AZ), replicated across all three AZs.

**Right — the Pool (red border, labelled "Pool"):** A collection of AWS-managed ACUs of various sizes sitting in a **warm pool**. Dotted arrows show ACUs being **allocated from this shared pool into the cluster** on demand. These pool ACUs are shared across AWS customers.

**Footer:** "Aurora Serverless Cluster" (spanning the AZs) and "Pool" (the shared warm resource) — reinforcing the two-part model: your cluster consumes compute drawn from AWS's shared pool.

---

## Architecture & Mechanics

The architecture shares much with Aurora Provisioned but has crucial differences.

### Similarities

- Same **Aurora cluster architecture**, now in the form of an **Aurora Serverless cluster**.
- Same **cluster volume architecture** (6 copies / 6 storage nodes across multiple AZs).

### Differences — the ACU model

- Instead of provisioned servers, the cluster uses **ACUs (Aurora Capacity Units)**.
- ACUs are allocated from a **warm pool of capacity units managed by AWS**.
- These pool ACUs are **stateless**, **shared across many AWS customers**, and have **no local storage** — which is what lets them be allocated to your cluster **rapidly**.
- Once an ACU is allocated to your cluster, it accesses the **cluster storage** exactly the way a provisioned Aurora instance would. _It's the same storage access — the only difference is where the compute comes from._

### Scaling behavior

- If load increases beyond current capacity **and the max ACU setting allows it**, **more ACUs are allocated**.
- Once the new (potentially bigger) ACU is active, **old compute representing unused capacity is deallocated**.

### 🎯 Connection management via the Proxy Fleet

Because ACU count changes dynamically, connection handling must be more complex than a provisioned cluster:

- There is a **shared proxy fleet, managed by AWS**.
- Application traffic goes **via the proxy fleet** (transparently to you). Any proxy fleet instance can broker the connection between application and ACUs.
- 🎯 **Because the client never connects directly to an ACU's compute**, scaling can be **fluid** — the cluster scales in or out **without disrupting application connections**.

**The only things you manage:** the **min and max ACU values** (plus you pay for cluster storage). Everything else — pool, proxy fleet, allocation — is AWS-managed.

---

## Use Cases (when Aurora Serverless makes sense)

|Use case|Why it fits|
|---|---|
|**Infrequently used applications**|e.g. a low-volume blog accessed a few minutes several times a day — pay only per-second for what you consume.|
|**New applications**|Load (and required instance size) is unknown; avoid provisioning in advance and disruptive resizes.|
|**Variable workloads**|Normally light with peaks (e.g. 30 min/hour, or sale periods) — scale in/out on demand instead of provisioning for peak or average.|
|**Unpredictable workloads**|Can't forecast load; set a wide min–max ACU range, monitor, and let it scale. Can leave it permanently in Serverless.|
|**Development & test databases**|Can **pause to zero** during no-load periods — pay only for storage. Very cost-effective.|
|**Multi-tenant applications**|When infrastructure scaling aligns with incoming revenue (e.g. per-license billing), scaling cost up alongside customer count makes sense.|

---

## ⚠️ Exam Traps / Gotchas

- ⚠️ **Serverless vs Provisioned comparison is often the real question.** The instructor stresses that comparing Aurora Serverless against the other RDS products is "just as important" as knowing it in isolation. Expect questions framed as "which product fits this scenario?"
- ⚠️ **You never connect directly to an ACU** — always via the proxy fleet. This is why scaling is non-disruptive. A common misconception is that scaling breaks connections.
- ⚠️ **Scale-to-zero = pay for storage only.** Pausing still incurs **storage charges**, never zero total cost.
- ⚠️ **Billing is per-second** for compute (ACUs) consumed, unlike Provisioned where you pay for provisioned instances whether used or not.
- ⚠️ **ACUs come from a shared, stateless, AWS-managed warm pool** — no local storage on the ACU is the reason they attach fast.

---

## `📚` Aurora Serverless v1 vs v2 (beyond the transcript)

The transcript predates / describes **v1**. Since the exam may test either, here's the distinction verified against current AWS material:

|Aspect|**Serverless v1** (this note / diagram)|**Serverless v2** (current product)|
|---|---|---|
|**Implementation**|A dedicated cluster type (**Aurora Serverless cluster**)|A **new instance type** inside a normal Aurora cluster — much richer feature parity with Provisioned|
|**Scaling granularity**|Doubles capacity (e.g. 8 → 16 ACU)|Fine-grained **0.5 ACU increments**|
|**Connection layer**|AWS-managed **proxy fleet** (v1 also had the **Data API** HTTP endpoint)|Uses **RDS Proxy** (optional, separately configured); **no Data API**|
|**Scale to zero / auto-pause**|Supported natively (pause on inactivity)|Originally min **0.5 ACU**; **scale-to-zero / auto-pause added Nov 2024** on supported engine versions (Aurora PostgreSQL 13.15+/14.12+/15.7+/16.3+, Aurora MySQL 3.08+). Resume ≈ 15 seconds.|
|**Multi-AZ / Global DB / read replicas**|Limited|Full support (same failover as Provisioned, Global Database, up to 15 readers)|
|**Max capacity**|Lower ceiling|Up to **256 ACUs** (256 ACU = ~512 GiB; 1 ACU ≈ 2 GiB memory)|

> [!NOTE] `📚` **On "ACUs come from a shared warm pool" (transcript claim)** The warm-pool + proxy-fleet model is specifically **v1** architecture. **v2** implements serverless as an instance type and scales the instance in place in fine-grained increments; connection stability with v2 is handled via **RDS Proxy** rather than the v1 proxy fleet. If an exam question describes the "warm pool of shared, stateless ACUs" and "proxy fleet," it is testing **v1** concepts.

> [!WARNING] `📚` **Cost commitment discounts** **Reserved Instances do NOT apply to Aurora Serverless** (RIs cover provisioned instance types only). Historically Savings Plans didn't apply either; **Database Savings Plans (launched at re:Invent 2025)** do now cover Serverless v2. This is a frequently-tested distinction — _don't_ pick "buy a Reserved Instance" to save money on a Serverless workload.

---

## 🎯 Exam Quick Reference

|Trigger keyword / scenario|Answer|
|---|---|
|"Don't want to manage / provision DB instances" + Aurora|**Aurora Serverless**|
|"Scales to zero / pauses when idle / pay only for storage when unused"|**Aurora Serverless** (v1, or v2 with scale-to-zero)|
|"Fargate for databases"|**Aurora Serverless**|
|"Infrequent / unpredictable / variable / spiky workload"|**Aurora Serverless**|
|"Dev/test DB, cost-effective when idle"|**Aurora Serverless** (pause to zero)|
|"Multi-tenant, cost scales with revenue"|**Aurora Serverless**|
|"Aurora Capacity Units (ACUs)"|**Aurora Serverless**|
|"Per-second billing for compute"|**Aurora Serverless**|
|"Warm pool + proxy fleet + stateless shared ACUs"|**Aurora Serverless v1**|
|"Fine-grained 0.5 ACU scaling / instance type / RDS Proxy"|**Aurora Serverless v2** `📚`|
|"Save money with a Reserved Instance on a serverless workload"|⚠️ **Wrong** — RIs don't apply to Serverless `📚`|

---

_Source: Adrian Cantrill AWS SAA course — Aurora Serverless lesson. v1/v2 comparison verified against AWS documentation and AWS Database Blog (2024–2026)._