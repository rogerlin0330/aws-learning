---
title: EBS — Elastic Block Store
type: service
tags: [ebs, block-storage, storage, gp2, gp3, io1, io2, block-express, volume-types, iops, throughput, elastic-volumes, credit-bucket, ssd, general-purpose, provisioned-iops, latency, per-instance-ceiling, az-boundary, snapshot, incremental-backup, fsr, fast-snapshot-restore, lazy-restore, lifecycle, multi-attach, billing, resilience, hdd, st1, sc1, throughput-optimized-hdd, cold-hdd]
certs: [aws-saa]
updated: 2026-05-28
---

# EBS — Elastic Block Store

EBS provides **network-attached block storage** for EC2. Volumes are created independently of any instance and persist independently — surviving stop/start cycles, instance replacement, and even instance termination (unless `DeleteOnTermination` is set). Every EBS volume is **AZ-scoped**: it can only attach to instances in the same AZ.

For the broader vocabulary (direct-attached vs network-attached, ephemeral vs persistent, block vs file vs object, the IOPS/throughput/block-size triangle) see [Storage Fundamentals](../concepts/storage.md). For instance store architecture and the decision between EBS and instance store, see [EC2 — Instance Store](../services/ec2.md#instance-store) and [EBS vs Instance Store](#ebs-vs-instance-store--decision-framework).

## Service Architecture

![EBS service architecture showing two regions, two AZs with EC2 instances attached to EBS volumes, snapshot flow to S3, and cross-region snapshot copy|697](../assets/ebs-service-architecture.png)
![[wiki/assets/ec2-host-vs-instance-icon-legend.png]]
### What EBS Volumes Are

EBS takes allocations of physical disks and presents them to EC2 as **raw block devices** — indistinguishable from a locally attached disk. The EC2 instance's OS is then responsible for putting a **file system** on top:

| OS | Common file systems |
| --- | --- |
| Linux | EXT3, EXT4, XFS |
| Windows | NTFS |

> [!warning] Gotcha
> EBS provides the *raw disk*. The file system (EXT4, NTFS) comes from the **instance's OS**, not from EBS.

### AZ Boundary — The Central Constraint

**EBS is an Availability Zone–scoped service.** Every architectural decision about EBS follows from this single fact.

- A volume lives in **exactly one AZ** and cannot be moved directly.
- EBS in AZ-A and EBS in AZ-B are **completely separate deployments** — and both are separate from any other region.
- Within its AZ, EBS **replicates data across multiple physical devices** — a single disk failure does not destroy the volume.
- If the **entire AZ fails**, volumes in that AZ fail with it.

**Attachment rules:**

- A volume can normally be **attached to one EC2 instance at a time**.
- It can be detached from one instance and **reattached to another — but only within the same AZ**.
- An instance in AZ-B **cannot attach to a volume in AZ-A**. No exceptions.

> [!tip] Exam Tip
> If a question asks how to give an instance access to data in a volume in a different AZ, the answer is always: **snapshot the volume → restore in the target AZ**. Direct cross-AZ attachment is impossible.

### Volume Lifecycle — Persistence Independent of Instances

EBS volumes exist independently of any EC2 instance.

| EC2 instance event | Effect on attached EBS volume |
| --- | --- |
| Stop & start | Volume persists, stays attached |
| Restart (reboot) | Volume persists, stays attached |
| Move to another host (after stop/start) | Volume follows the instance |
| Terminated | Volume persists **unless** `DeleteOnTermination` was set |
| Detached | Volume persists; can be reattached to another instance in the same AZ |

> [!tip] Exam Tip
> EBS volumes **keep billing** even when the EC2 instance is stopped. Storage charges accrue against the volume, not the instance state.

### Snapshots — Crossing the AZ and Region Boundaries

Because EBS cannot cross AZs directly, **snapshots** are the bridge. A snapshot is stored in an **AWS-managed area of S3** — regionally resilient (replicated across 3+ AZs automatically) — lifting the data's resilience scope from AZ to region.

![EBS snapshot architecture: a volume in AZ-A produces a Full snapshot then an incremental Incr snapshot in S3, with restore paths to the same AZ, a different AZ (cross-AZ), and a different region (cross-region DR)](../assets/ebs-snapshot-architecture.png)

From S3, a snapshot can restore a new volume into:

- The **same AZ** as the original
- A **different AZ in the same region** → cross-AZ data migration
- A **different region** (after copying the snapshot there) → cross-region DR

#### Incremental Storage

| Snapshot | What is physically stored |
| --- | --- |
| **First** | Full copy of **used data only** — not the allocated volume size |
| **2nd onward** | Only the data **changed since the previous snapshot** |

- A 40 GB volume with 10 GB written → the first snapshot stores and bills for **10 GB**, not 40 GB
- Later snapshots **reference** earlier ones for unchanged blocks instead of re-storing them
- The first snapshot can take time (copying all used data to S3 in the background); **EBS performance is unaffected** during this

> [!warning] Gotcha — Incrementals are self-sufficient
> With traditional backup software, deleting an incremental breaks every backup after it. **EBS does not work this way.** Deleting any snapshot causes AWS to automatically relocate the referenced data — all later snapshots remain fully restorable. You can safely delete any individual snapshot.

#### Snapshot Billing

![EBS snapshot billing diagram tracking Vol. 1 through three snapshots — Snap A stores 10 GiB (full), Snap B stores 4 GiB (changed, references 6 GiB from A), Snap C stores 2 GiB (new, references 6 GiB from A and 4 GiB from B)](../assets/ebs-snapshot-consumption-and-billing.png)

Three billing rules:

1. **Unit: GB-month.** Size × duration — 10 GB for 1 month = 20 GB for half a month (same charge).
2. **Billed on used data, not allocated size.** The volume itself bills for all 40 GB (reserved capacity); the snapshot bills only for the data actually written.
3. **Incremental storage keeps frequent snapshots cheap.** Each snapshot after the first stores only new/changed data — the total bill converges on "original used data + net changes" regardless of how many snapshots you slice it into.

**Worked example (the diagram):**

| Snapshot | Volume | Data stored | References | Billed |
| --- | --- | --- | --- | --- |
| **Snap A** | 10 GiB | 10 GiB (full) | — | **10 GiB** |
| **Snap B** | 10 GiB | 4 GiB (changed) | 6 GiB → Snap A | **4 GiB** |
| **Snap C** | 12 GiB | 2 GiB (new) | 6 GiB → A, 4 GiB → B | **2 GiB** |

Total physical storage: **16 GiB** — not 10 + 10 + 12 = 32 GiB.

#### Restore Performance

| Scenario | Performance |
| --- | --- |
| **New blank volume** | Full performance immediately |
| **Volume restored from snapshot** | **Lazy restore** — blocks pulled from S3 on first access; unrestored blocks are slower |

Two ways to force full performance immediately after restoring from a snapshot:

- **`dd` (manual, free):** Read every block sequentially to force EBS to pull all data from S3 into the volume before production use.
- **Fast Snapshot Restore (FSR):** Enable on a snapshot so restores are instant — no lazy load. Costs extra.
  - Enabled **per snapshot + AZ combination** — one snapshot enabled across 4 AZs = 4 FSR sets consumed
  - **Limit: 50 FSR sets per region**
  - Use only when warm-up latency **and** `dd` admin overhead are both unacceptable — FSR incurs additional cost, so default to `dd` if either tradeoff is tolerable

> [!tip] Exam Tip
> Degraded I/O immediately after restoring a volume from a snapshot = **lazy restore** behavior, not hardware failure. Fix with `dd` (free) or FSR (instant, extra cost).

> [!warning] Gotcha — FSR limit counts per snapshot + AZ, not per snapshot
> One snapshot enabled across 4 AZs consumes **4** of your 50 FSR sets per region, not 1. Enabling FSR broadly across many snapshots and AZs can exhaust the limit quickly.

### Resilience Hierarchy

| Tier | Mechanism | Protects against |
| --- | --- | --- |
| 1 | EBS internal replication within the AZ | Individual disk failure |
| 2 | Snapshot to S3 | AZ-level failure → restore to another AZ |
| 3 | Snapshot copy across regions | Region-level failure → restore in another region |

### Multi-Attach

Certain volume types support **multi-attach** — simultaneous attachment to multiple instances within the same AZ.

- Requires **cluster-aware application software** to coordinate writes; without it, concurrent writes can corrupt the volume.
- Not a default behavior — must be explicitly enabled on the volume.

### Billing Model

EBS bills by **GB-per-month**:

- Billing is continuous: it doesn't matter whether an instance is running, stopped, or the volume is detached.
- 1 GB for 1 month = 2 GB for half a month — same charge.
- High-performance volume types (Provisioned IOPS) add **separate line items** for provisioned IOPS.

## Volume Types Overview

EBS offers multiple volume types across two media families:

| Family | Types | Primary use |
| --- | --- | --- |
| **General Purpose SSD** | GP2, GP3 | Most workloads; default choice |
| **Provisioned IOPS SSD** | io1, io2, io2 Block Express | High-throughput databases; latency-critical |
| **Throughput Optimised HDD** | st1 | Big data, log processing; sequential reads |
| **Cold HDD** | sc1 | Archival; infrequent access |

This page covers the **General Purpose SSD** types, the **Provisioned IOPS SSD** types, and the **HDD-based** types in depth.

## GP2 — General Purpose SSD (Credit-Bucket Model)

![GP2 credit-bucket diagram showing the I/O credit accumulation and consumption flow between EBS volume and EC2 instance](../assets/ebs-gp2.png)

### Reading the GP2 Diagram

The GP2 diagram is built around one central metaphor: an **I/O credit bucket** sitting between the EBS volume (storage icon, left) and the EC2 instance (chip icon, right). Every read or write the instance performs costs credits drawn from this bucket.

**Context bar (top row):**

- **Top-left — size range annotation:** "Volumes can be as small as 1 GB, or as large as 16 TB." Size matters here because, unlike GP3, GP2 performance is directly tied to volume size.
- **Top-middle — use cases:** "Boot volumes, low-latency interactive apps, dev & test." GP2 was the AWS default until AWS switched to GP3.
- **Top-right — unit definitions (exam-critical):** "An IO Credit is 16 KB / IOPS assume 16 KB / 1 IOPS is 1 IO in 1 second." A 160 KB transfer in one second = 10 IOPS = 10 credits consumed.

**The bucket (centre):**

The bucket is labelled "3,000 IOPS Burst" and has a **red lid** representing the burst ceiling — credits flow out at up to 3,000 IOPS, no faster. Key facts attached to the bucket:

- **Capacity: 5.4 million IO credits**, refilling continuously at the baseline rate.
- **Starts full on creation** — enough for 30 minutes at 3,000 IOPS, which is why fresh instances boot quickly and handle initial spike workloads well.

**Left side — credit accumulation (how the bucket fills):**

Three stacked bands reading bottom-to-top, each representing a progressively higher refill rate:

- **Bottom thin band** — minimum floor: every GP2 volume, regardless of size, refills at least **100 credits/second**.
- **Pink middle band** — size-scaled formula: **3 credits/second per GB of volume size**. This formula beats the floor above 33.33 GB and is the normal operating range.
- **Green top band** — hard ceiling: refill rate is capped at **16,000 credits/second** regardless of volume size. Reached at ~5.33 TB (`16,000 ÷ 3 ≈ 5,333 GB`).
- **Note on the green band** — volumes larger than 1 TB have a baseline (3 × 1,000 = 3,000) that already equals or exceeds the burst rate, so the bucket is never fully depleted and the credit system becomes irrelevant.

**Right side — credit consumption (how the bucket drains):**

Red arrows flow upward from the bucket toward the EC2 instance. When the workload demands more IOPS than the refill rate, the difference is covered by drawing from the bucket — this is **bursting**. If consumption consistently exceeds refill, the bucket drains to zero and performance collapses to the baseline refill rate until credits rebuild.

**Mental model:** GP2 = a piggy bank that fills slowly based on how large your volume is. Small volumes have a slow drip but start full, so they can burst hard for a while. Large volumes (> 1 TB) have such a fast drip that the piggy bank never empties.

**Size range:** 1 GB – 16 TB

**Use cases:** boot volumes, low-latency interactive apps, dev/test. GP2 was the AWS default for new volumes until AWS switched the default to GP3.

### The I/O Credit System

GP2 performance is governed by an **I/O credit bucket** that sits between the volume and the instance. Every read and write costs credits from this bucket.

**Unit definitions (exam-critical vocabulary):**

- 1 I/O credit = 16 KB of data
- 1 IOPS = one 16 KB I/O operation per second = 1 credit/second
- A 160 KB transfer in one second = 10 IOPS = 10 credits consumed that second

**The bucket itself:**

- Capacity: **5.4 million credits**
- New volumes start **full** → enough for **30 minutes at 3,000 IOPS** straight out of the gate (great for boot and initial workloads)
- Refills continuously at the volume's **baseline performance rate**
- Burst ceiling: **3,000 IOPS** — credits can drain no faster than this

> [!warning] Gotcha — Credit Exhaustion
> If a workload consistently demands more IOPS than the baseline refill rate, the bucket eventually drains to zero and performance collapses to the baseline rate. You are responsible for monitoring credit balance across all GP2 volumes.

### Baseline Performance (Refill Rate)

The bucket refills at the volume's baseline rate, which is size-dependent:

| Volume size | Baseline IOPS |
| --- | --- |
| < 33.33 GB | **100 IOPS** (floor — minimum regardless of size) |
| 33.33 GB – 1 TB | **3 IOPS/GB** (formula beats floor above 33.33 GB) |
| 1 TB – 5.33 TB | **3,000 IOPS** (baseline = burst; credit system irrelevant) |
| ≥ 5.33 TB | **16,000 IOPS** (hard ceiling; constant regardless of further size increase) |

**Key crossover points:**

- **33.33 GB** — where `3 × GB = 100`; above this, the formula wins
- **1 TB** — where `3 × 1,000 = 3,000 IOPS`; baseline equals the burst rate, so the bucket never depletes and never matters
- **5.33 TB** — where `3 × 5,333 ≈ 16,000`; maximum IOPS reached; adding more size buys nothing

### Burst Performance

Any GP2 volume can **burst to 3,000 IOPS** by drawing down the credit bucket faster than it refills. This is what allows small volumes to handle short, heavy workloads — but it is finite.

**Throughput ceiling:** 250 MB/s (hard cap regardless of volume size or IOPS)

### GP2 Summary

| Attribute | Value |
| --- | --- |
| Performance model | Credit bucket — size-dependent |
| Baseline IOPS | 3 IOPS/GB (min 100, max 16,000) |
| Burst IOPS | 3,000 (via credit bucket) |
| Credit bucket capacity | 5.4 million credits |
| Initial credit balance | Full (5.4 M = 30 min @ 3,000 IOPS) |
| Max throughput | 250 MB/s |

## GP3 — General Purpose SSD (Flat-Rate Model)

![GP3 two-tier performance model showing standard base tier and optional provisioned extras tier](../assets/ebs-gp3.png)

### Reading the GP3 Diagram

GP3 drops the credit bucket entirely — no such element appears. Instead the diagram shows a flat **two-tier performance model** between the EBS volume (left) and the EC2 instance (right).

**Context bar (top row):**

- **Top-left — "GP2 and IO1 had a baby" metaphor:** GP3 inherits GP2's affordability and IO1's (Provisioned IOPS SSD) higher, more predictable performance profile.
- **Top-right — use cases:** "Virtual desktops, medium-sized single-instance databases such as MSSQL Server and Oracle DB, low-latency interactive apps, dev & test, boot volumes."

**Middle — the two-tier model (read bottom-to-top):**

- **Pink band (bottom layer) with rabbit icons — labelled "STANDARD":** every GP3 volume, regardless of size, gets **3,000 IOPS and 125 MB/s** included in the base price. No formula, no bucket, no crossover arithmetic.
- **Green band (upper layer) with leopard icons and red upward arrows — labelled "Extra cost":** if you need more, you pay separately for up to **16,000 IOPS** and up to **1,000 MB/s** throughput. The leopards signal faster-than-standard speed; the arrows show these are additive upgrades on top of the standard tier, not replacements for it.

The critical design point the diagram illustrates: the IOPS dial and the throughput dial are **completely independent**. You can add extra IOPS without extra throughput, or vice versa. GP2 tied performance to volume size — you couldn't separate them.

**Footer (bottom row):**

- **Left — "~20% cheaper vs GP2 base price":** at the same capacity, GP3 undercuts GP2 at the baseline.
- **Right — "4× faster max throughput vs GP2 — 1,000 MB/s vs 250 MB/s":** the throughput ceiling improvement is the most exam-tested headline difference.

**Mental model:** GP3 = a flat-rate broadband plan. Everyone gets the same baseline speed; if you want more, you pay for an upgrade — no credit accounting, no surprises.

**Size range:** 1 GB – 16 TB (same as GP2)

**Use cases:** virtual desktops, medium-sized single-instance databases (MSSQL Server, Oracle DB), low-latency interactive apps, dev/test, boot volumes. **Current AWS default** for new EBS volumes.

### The Two-Tier Performance Model

GP3 eliminates the credit bucket entirely. Instead it uses a simple **standard + provisioned** model:

**Standard tier (included in base price, every volume, every size):**

- **3,000 IOPS**
- **125 MB/s** throughput

No formula, no bucket, no crossover points. A 1 GB GP3 volume and a 16 TB GP3 volume both get exactly 3,000 IOPS and 125 MB/s.

**Provisioned extras (extra cost, independently configurable):**

- Up to **16,000 IOPS**
- Up to **1,000 MB/s** throughput

> [!tip] Key Insight
> The IOPS dial and the throughput dial are **independent on GP3**. You can pay for extra IOPS without extra throughput, or vice versa. On GP2, performance was bundled with size — you couldn't separate them.

### GP3 Advantages Over GP2

- **~20% cheaper** base price at the same capacity
- **4× the max throughput** — 1,000 MB/s vs 250 MB/s
- **Size-independent baseline** — a 1 GB volume gets the same 3,000 IOPS baseline as a 1 TB volume; no need to over-provision size just to hit a performance target

### GP3 Summary

| Attribute | Value |
| --- | --- |
| Performance model | Flat standard + paid extras (size-independent) |
| Standard IOPS | 3,000 (all volumes, all sizes) |
| Standard throughput | 125 MB/s (all volumes, all sizes) |
| Max IOPS (provisioned) | 16,000 |
| Max throughput (provisioned) | 1,000 MB/s |
| Base price vs GP2 | ~20% cheaper |

## GP2 vs GP3 — Side-by-Side

| Attribute              | GP2                                   | GP3                         |
| ---------------------- | ------------------------------------- | --------------------------- |
| Performance model      | Credit bucket — size-dependent        | Flat standard + paid extras |
| Baseline IOPS (= credit refill rate for GP2) | 3 IOPS/GB (min 100, max 16,000) | 3,000 for all volumes |
| Burst IOPS             | Up to 3,000 (via credits)             | N/A — no bursting concept   |
| Max IOPS               | 16,000 (at ≥ 5.33 TB)                 | 16,000 (paid extra)         |
| Standard throughput    | Scales with size                      | 125 MB/s for all volumes    |
| Max throughput         | **250 MB/s**                          | **1,000 MB/s** — 4× GP2     |
| Initial credit balance | 5.4 M credits (≈ 30 min @ 3,000 IOPS) | N/A                         |
| Base price             | Higher                                | **~20% cheaper than GP2**   |
| AWS default today      | No (was default; AWS switched to GP3) | **Yes — current default**   |

## Migration and Elastic Volumes

You can **change volume type on the fly** (GP2 → GP3 and vice versa) using the **Elastic Volumes** feature — no downtime required.

> [!warning] Gotcha — GP2 → GP3 Migration
> When migrating from GP2 to GP3, performance **does not auto-scale with volume size** anymore. GP2 gave you `3 × GB` IOPS automatically; on GP3, anything above the 3,000 IOPS standard must be **explicitly provisioned** (and paid for). Check your workload's actual IOPS requirements before migrating and provision accordingly.

Despite this, GP3 typically remains cheaper than the equivalent GP2 configuration even after provisioning the needed extras.

## Key Takeaways

- **GP2 is credit-based; GP3 is flat-rate.** This is the single most important conceptual difference.
- **GP2 baseline = 3 IOPS/GB**, minimum 100, max 16,000. Crossover with floor at **33.33 GB**; baseline equals burst at **1 TB**; max IOPS reached at **5.33 TB**.
- **GP2 burst = 3,000 IOPS**, fuelled by a **5.4 million credit bucket** that starts full.
- **GP3 standard = 3,000 IOPS + 125 MB/s** for every volume, regardless of size.
- **GP3 max = 16,000 IOPS + 1,000 MB/s** (paid extras, independently provisioned).
- **GP3 is ~20% cheaper** at base price and has **4× the max throughput** of GP2.
- **Both** support boot volumes, dev/test, low-latency workloads.
- Use **Elastic Volumes** to swap GP2 ↔ GP3 live — but remember to explicitly provision the IOPS/throughput GP3 needs after migrating from GP2.

## Provisioned IOPS SSD — IO1, IO2, IO2 Block Express

![EBS IO1/IO2 infographic showing per-volume and per-instance performance ceilings for the Provisioned IOPS SSD family](../assets/ebs-io1-and-io2.png)

When the GP2/GP3 ceiling of 16,000 IOPS isn't enough — or when you need *consistently* low, jitter-free latency regardless of burst credits — you reach for the **Provisioned IOPS SSD** family. Unlike GP2/GP3, IOPS are **completely decoupled from volume size**: you set the size and the IOPS independently, subject only to a per-GB ratio cap.

### Reading the IO1/IO2 Diagram

The infographic lays out the IO1/IO2/Block Express family across three zones reading top-to-bottom.

**Context bar (top row):**

- **Top-left — F1 car icon + per-volume performance numbers:** the F1 car signals "extreme speed." Four bullets state the maximum a single volume can deliver:
  - **Up to 64,000 IOPS per volume** — shared by IO1 and IO2; four times the GP2/GP3 ceiling.
  - **Up to 256,000 IOPS per volume (Block Express only)** — four times the IO1/IO2 ceiling.
  - **Up to 1,000 MB/s throughput** — shared by IO1 and IO2.
  - **Up to 4,000 MB/s throughput (Block Express only)** — four times IO1/IO2.
- **Top-right — flat wavy line + "Consistent Low Latency & Jitter":** the flat, green wavy line is the visual cue. It's not just that latency is low — it stays low and predictable over time. Low jitter (small variation between requests) is the whole reason you pay extra for these volumes. Databases and latency-sensitive apps need *predictable* response times, not just fast averages.

**Middle — Per-Instance Performance panel (the most important concept):**

The pink panel in the middle, labelled "Per Instance Performance," calls out a ceiling that exists *separately* from the per-volume ceilings above.

- **Stacked SSD icons on the left:** four disk icons stacked vertically symbolise that reaching the per-instance ceiling almost always requires *multiple* volumes attached to one instance. A single volume cannot saturate it.
- **Three per-instance limits listed:**
  - **io1 — 260,000 IOPS & 7,500 MB/s** — to saturate this with IO1 volumes (64,000 IOPS each), you need just over four volumes at maximum.
  - **io2 — 160,000 IOPS & 4,750 MB/s** — lower than IO1, which seems counterintuitive. AWS placed the extra headroom into Block Express rather than standard IO2.
  - **io2 Block Express — 260,000 IOPS & 7,500 MB/s** — matches IO1's per-instance ceiling, and because Block Express volumes individually reach 256,000 IOPS, far fewer volumes are needed to get there.
- **Right side of the panel — use case callout:** "High performance, latency sensitive workloads. I/O-intensive NoSQL & relational databases."

> [!warning] Why per-instance matters
> The per-instance ceiling is also gated by the **EC2 instance type and size** — only the newest, largest instances support these peak numbers. Three things must align for maximum performance: **volume type**, **instance type**, and **instance size**.

**Footer (bottom row):**

- **Bottom-left — slider icon + "IOPS can be adjusted independently of size":** unlike GP2 (where IOPS scaled with size) and GP3 (where the standard tier is fixed), here the IOPS dial is fully decoupled from the size dial. You can have a small disk with massive performance.
- **Middle — speedometer + IOPS/GB ratio ceilings (in red — these are upper bounds, not defaults):**
  - io1: **50 IOPS/GB MAX**
  - io2: **500 IOPS/GB MAX** (10× IO1)
  - Block Express: **1,000 IOPS/GB MAX** (2× IO2)
- **Bottom-right — volume size ranges:**
  - **4 GB – 16 TB** for IO1 and IO2 (note: minimum is 4 GB, not 1 GB like GP2/GP3).
  - **4 GB – 64 TB** for Block Express.

### The Three Variants

| Attribute | IO1 | IO2 | IO2 Block Express |
| --- | --- | --- | --- |
| Release status | GA | GA | Preview (at time of notes) |
| Size range | 4 GB – 16 TB | 4 GB – 16 TB | 4 GB – **64 TB** |
| Max IOPS per volume | 64,000 | 64,000 | **256,000** |
| Max throughput per volume | 1,000 MB/s | 1,000 MB/s | **4,000 MB/s** |
| Max IOPS/GB ratio | 50 | 500 | 1,000 |
| Per-instance max IOPS | 260,000 | 160,000 | 260,000 |
| Per-instance max throughput | 7,500 MB/s | 4,750 MB/s | 7,500 MB/s |

**Common to all three:**

- SSD-backed.
- IOPS configurable **independently of volume size** (subject to the IOPS/GB ratio cap).
- Designed for **consistent, sub-millisecond, low-jitter latency** — the defining benefit over GP2/GP3.
- Pricing: **separate line items** for storage size and provisioned IOPS.
- Minimum size is **4 GB** (GP2/GP3 start at 1 GB).

### IOPS/GB Ratio Caps

IOPS are decoupled from size, but there is a ceiling on how aggressively you can dial performance up relative to how small the disk is:

| Volume type | Max IOPS/GB |
| --- | --- |
| IO1 | 50 (16× GP2's 3 IOPS/GB) |
| IO2 | 500 (10× IO1) |
| IO2 Block Express | 1,000 (2× IO2) |

> [!tip] Exam Tip
> "MAX" is a hard upper bound, not a default. A 10 GB IO2 volume can be provisioned with up to 5,000 IOPS (500 × 10), not the full 64,000 — to reach 64,000 IOPS you need at least 128 GB.

### The Two-Ceiling Rule

This is the concept most likely to appear as an exam trick. There are **two independent ceilings** that both apply simultaneously:

- **Per-volume ceiling**: the maximum IOPS a single volume can deliver (IO1/IO2 = 64,000 IOPS; Block Express = 256,000 IOPS).
- **Per-instance ceiling**: the maximum IOPS that can flow from the EBS service into a **single EC2 instance**, regardless of how many volumes are attached.

To reach the per-instance ceiling you almost always need multiple volumes working in parallel:

| Volume type | Per-instance IOPS | Per-instance throughput | Volumes needed to saturate (IO1 example) |
| --- | --- | --- | --- |
| IO1 | 260,000 | 7,500 MB/s | ~4+ volumes at max |
| IO2 | 160,000 | 4,750 MB/s | ~3+ volumes at max |
| IO2 Block Express | 260,000 | 7,500 MB/s | ~2 volumes (256,000 per volume) |

> [!warning] Gotcha — Three things must align for top performance
> The per-instance ceiling is also gated by **EC2 instance type and size** — only the newest, largest instances support these peak numbers. The formula for maximum performance: **volume type + instance type + instance size** must all be at the top tier simultaneously.

### EBS Volume Type Selection Guide

| Need | Pick |
| --- | --- |
| Default, general-purpose boot/app volume | **GP3** |
| Up to 16,000 IOPS, cost-effective | **GP3** |
| Up to 64,000 IOPS, consistent low latency | **IO1** or **IO2** |
| Small volume with extreme IOPS (size-independent dial) | **IO1/IO2** |
| Up to 256,000 IOPS, up to 64 TB | **IO2 Block Express** |
| RAID 0 for combined IOPS (per-instance ceiling ~260,000 IOPS) | Multiple **IO2 Block Express** volumes + large instance — see [RAID 0 and the Per-Instance Ceiling](#raid-0-and-the-per-instance-ceiling) |
| Frequent sequential access, big data / logs / data warehouses | **ST1** |
| Infrequent / cold access, lowest cost | **SC1** |
| Maximum possible performance, persistence not required | [Instance Store](../services/ec2.md#instance-store) |

> [!tip] Exam Tip
> Instance store can exceed even Block Express performance numbers, but the storage is **ephemeral** — it lives and dies with the instance. Provisioned IOPS SSD is the top choice when you need both extreme performance **and** durability.

### IO1/IO2 Use Cases

- I/O-intensive relational databases (Oracle, MSSQL, PostgreSQL under heavy OLTP load).
- I/O-intensive NoSQL databases (Cassandra, MongoDB with strict SLAs).
- Any workload where **jitter** (variation in latency) is unacceptable, not just average latency.
- Small volumes that need disproportionately high performance — the decoupled IOPS dial is the key capability.

### IO1/IO2 Key Takeaways

- IO1/IO2 IOPS are **fully decoupled from volume size** — but capped by the ratio: **50 / 500 / 1,000 IOPS/GB** for IO1 / IO2 / Block Express.
- Per-volume IOPS ceiling: **64,000** (IO1/IO2), **256,000** (Block Express).
- Per-volume throughput ceiling: **1,000 MB/s** (IO1/IO2), **4,000 MB/s** (Block Express).
- Per-instance ceilings exist **separately** and require multiple volumes: **260,000 IOPS** (IO1/Block Express), **160,000 IOPS** (IO2).
- Size ranges: **4 GB – 16 TB** (IO1/IO2), **4 GB – 64 TB** (Block Express).
- Pay for **size** and **provisioned IOPS** as separate line items.
- Three things must align for maximum performance: **volume type**, **instance type**, **instance size**.

## HDD-Based Volumes — ST1 and SC1

![EBS HDD-based volumes infographic comparing ST1 (Throughput Optimized HDD) on the left and SC1 (Cold HDD) on the right, showing size ranges, use cases, and performance numbers for each type](../assets/ebs-hdd-based-volumes.png)

HDD volumes use physical spinning platters — moving parts mean lower performance ceilings than SSD, but also lower cost per GB. There are two in general use: **ST1** (Throughput Optimized HDD) and **SC1** (Cold HDD). A third, legacy HDD type exists but is not in scope.

### Reading the HDD Diagram

The diagram is split down the middle into **ST1 on the left, SC1 on the right** — reading them as a pair makes the trade-off obvious: SC1 is a cheaper, slower version of ST1. Each half has the same layout: header icons (a train indicates ST1's sequential-throughput identity; an iceberg and penguin for SC1's "cold" positioning), a size range, a use-case description, and performance numbers. The most important detail is the **"(1MB)"** label next to the IOPS figures on both sides — it signals that one I/O operation on HDD moves a 1 MB block (vs 16 KB on SSD), which is why the IOPS and MB/s numbers are always equal. The centre bars (red and denser for ST1, blue and sparser for SC1) visually reinforce the relative throughput difference.

### The 1 MB I/O Block — the Key HDD Mechanic

On SSD volumes, 1 IOPS = one **16 KB** operation. On HDD volumes, 1 IOPS = one **1 MB** operation. This single fact explains why HDD IOPS ceilings and MB/s ceilings are the same number:

- **ST1**: 500 IOPS × 1 MB = **500 MB/s**
- **SC1**: 250 IOPS × 1 MB = **250 MB/s**

Large I/O blocks play to a spinning disk's strength — sequential reads of big contiguous chunks. Random small reads (where SSDs excel) expose a spinning disk's weakness (head seeks).

### ST1 vs SC1 — Comparison

| Attribute | ST1 (Throughput Optimized) | SC1 (Cold HDD) |
| --- | --- | --- |
| Relative cost | Cheap (less than any SSD) | **Cheapest EBS storage of all** |
| Size range | 125 GB – 16 TB | 125 GB – 16 TB |
| I/O block size | 1 MB | 1 MB |
| Max IOPS | 500 | 250 |
| Max throughput | 500 MB/s | 250 MB/s |
| Baseline performance | 40 MB/s per TB | 12 MB/s per TB |
| Burst performance | 250 MB/s per TB | 80 MB/s per TB |
| Access pattern | Frequent, sequential, throughput | Infrequent, cold, archival |
| Typical use cases | Big data, data warehouses, log processing | Cold data, archives, few scans/day |

### Credit Bucket on HDD vs GP2

ST1 and SC1 reuse the same **credit-bucket concept** from [GP2](#gp2--general-purpose-ssd-credit-bucket-model), with one key difference: the unit is **MB/s** (1 MB I/O blocks), not IOPS (16 KB credits). Baseline and burst performance both scale **per TB of volume size** — a larger volume earns proportionally more baseline throughput and burst headroom, just as a larger GP2 volume earns more IOPS.

| Volume type | Credit unit | Baseline refill rate |
| --- | --- | --- |
| GP2 | IOPS (16 KB I/O credits) | 3 IOPS/GB (min 100) |
| ST1 | MB/s (1 MB I/O blocks) | 40 MB/s per TB |
| SC1 | MB/s (1 MB I/O blocks) | 12 MB/s per TB |

### When to Choose HDD

1. **Need IOPS / random access / high performance?** → Avoid both HDD types. Use SSD (GP3, or IO1/IO2 for extreme needs).
2. **Access data infrequently (a few scans per day)?** → **SC1** — the cheapest EBS storage available, full stop.
3. **Frequently access data sequentially?** → **ST1** — still cheap, with enough throughput for big data, warehouses, and log processing.

> [!tip] Exam Tip
> HDD = economy of *storage capacity*, not performance. The two discriminators: (1) **random vs sequential** — HDD is only appropriate for sequential; any IOPS or random-access requirement rules it out. (2) **frequent vs infrequent** — ST1 for frequently accessed sequential data, SC1 for cold/rarely touched archival data.

> [!warning] Gotcha — Minimum size is 125 GB
> Both ST1 and SC1 have a **125 GB minimum** — much larger than GP2/GP3's 1 GB minimum. A question about a small volume with a sequential workload still cannot use HDD. Also, neither type supports boot volumes.

### HDD Key Takeaways

- **Two HDD types in use: ST1 (Throughput Optimized) and SC1 (Cold).** A legacy third type is out of scope.
- **HDD I/O is measured in 1 MB blocks** (vs 16 KB on SSD) — so IOPS and MB/s ceilings are the same number.
- **ST1: max 500 IOPS / 500 MB/s**, baseline **40 MB/s per TB**, burst **250 MB/s per TB**.
- **SC1: max 250 IOPS / 250 MB/s**, baseline **12 MB/s per TB**, burst **80 MB/s per TB**.
- **Both range 125 GB – 16 TB** (minimum is 125 GB — much larger than SSD's 1 GB floor).
- **Credit bucket measured in MB/s per TB**, not IOPS — same bucket mechanism as GP2, different unit.
- **ST1 = frequent, sequential, throughput-heavy** (big data, warehouses, log processing).
- **SC1 = infrequent, cold, archival** — the **lowest-cost EBS storage of all**.
- **Neither type is suitable for IOPS-heavy or random-access workloads** — use SSD instead.

## EBS vs Instance Store — Decision Framework

For instance store architecture details (physical attachment, ephemeral behaviour, naming, performance numbers), see [EC2 — Instance Store](../services/ec2.md#instance-store).

### Default Rule

> **If you need persistent storage → use EBS.** Instance store is not persistent, so it is the wrong starting point whenever durability matters.

### Clear-Cut: Choose EBS

| Requirement | Why instance store does not fit |
| --- | --- |
| **Persistence** | Instance store data is lost on stop/start, resize, host failure, or volume hardware failure |
| **Resilience without application-level help** | EBS replicates across multiple physical disks within the AZ; snapshots bridge AZs and regions. Instance store has neither |
| **Lifecycle isolation** | Need to detach a volume and reattach to a different instance? Only EBS supports this |

> [!warning] Accuracy note — reboot does not lose instance store data
> A **restart (reboot)** keeps the instance on the same host — instance store data survives. Data is lost only when the instance **moves to a new host** (stop/start, resize, host failure, volume hardware failure). "Reboot" and "stop/start" are not the same operation; don't conflate them.

### "It Depends" — When Instance Store May Win

| Scenario | Choice | Reason |
| --- | --- | --- |
| App has **built-in replication** (e.g. Cassandra, HDFS, distributed NoSQL) | **Instance store** across many instances | The app handles durability; you keep performance and skip EBS cost |
| Need **super-high performance** (> ~260,000 IOPS per instance) | **Instance store** | i3 instances reach ~2 million IOPS — EBS tops out at ~260,000 per instance |
| **Cost** is primary and the instance type already includes instance store | **Instance store** | Already included in the instance price — no reason not to use it |

### EBS HDD Type Selection

When EBS is right but cost or throughput matters more than IOPS:

| Need | Type |
| --- | --- |
| Cheapest storage per GB | **SC1** — the lowest-cost EBS storage overall |
| Throughput / streaming / sequential access | **ST1** |
| Boot volume | **Neither ST1 nor SC1** — HDD types cannot be used as boot volumes |

### RAID 0 and the Per-Instance Ceiling

Multiple EBS volumes can be striped into a **RAID 0 set** to sum their IOPS:

- A RAID 0 set over IO2 Block Express volumes can approach very high combined IOPS
- But there is a **hard per-instance ceiling: ~260,000 IOPS** — no matter how many volumes you stripe, the EC2 instance cannot consume more than this from EBS
- Reaching ~260,000 IOPS requires a **large, newer instance type** plus enough volumes to saturate it

> [!tip] Exam Tip
> The per-instance ceiling (~260,000 IOPS) is separate from, and independent of, the per-volume ceiling. See [The Two-Ceiling Rule](#the-two-ceiling-rule) above. Adding more EBS volumes beyond what saturates the per-instance ceiling yields nothing.

### Beyond 260,000 IOPS

| IOPS requirement | Storage |
| --- | --- |
| Up to 16,000 per volume | GP2 / GP3 |
| 16,000–64,000 per volume | IO1 / IO2 |
| Up to 256,000 per volume | IO2 Block Express |
| Up to ~260,000 per instance (RAID 0) | IO2 Block Express + large instance |
| **> 260,000 per instance, non-persistent acceptable** | **[Instance Store](../services/ec2.md#instance-store)** (i3: ~2 million IOPS) |

> [!warning] Gotcha — trade persistence for performance
> Instance store can far exceed EBS's per-instance ceiling, but the data is **ephemeral**. An i3 instance store reaches millions of IOPS; IO2 Block Express (EBS's maximum) reaches 256,000 per volume and ~260,000 per instance. Choose instance store only when the workload can tolerate total data loss on any host-change event.

### Quick Decision Cheat-Sheet

| If the question emphasizes… | Pick |
| --- | --- |
| Persistence / durability | **EBS** |
| Resilience within an AZ + snapshots | **EBS** |
| Detach & reattach to another instance | **EBS** |
| Resilience **with** app-level replication | **Instance store** (spread across many instances) |
| Lowest cost HDD | **ST1 / SC1** |
| Throughput / streaming | **ST1** |
| Boot volume | **Not ST1/SC1** |
| 16,000–64,000 IOPS per volume | **IO1 / IO2** |
| Up to 256,000 IOPS per volume | **IO2 Block Express** |
| Max per instance (RAID 0) | **~260,000 IOPS** |
| More than 260,000 IOPS, non-persistent OK | **Instance store** (millions of IOPS) |

## References

- Source: `raw/aws-cert-study-notes/aws-solution-architect/ebs_service_architecture.md`
- Source: `raw/aws-cert-study-notes/aws-solution-architect/ebs_gp2_and_gp3.md`
- Source: `raw/aws-cert-study-notes/aws-solution-architect/eb2_io1_and_io2.md`
- Source: `raw/aws-cert-study-notes/aws-solution-architect/ebs_hdd_based_volumes.md`
- Source: `raw/aws-cert-study-notes/aws-solution-architect/ec2_instance_store_vs_ebs.md`
- Source: `raw/aws-cert-study-notes/aws-solution-architect/ebs_snapshots.md`
- [EC2 — Instance Store](../services/ec2.md#instance-store)
- [Storage Fundamentals](../concepts/storage.md)
