# EBS Volume Types — General Purpose SSD (GP2 & GP3)

> **Context:** EBS (Elastic Block Store) provides several volume _types_. This note covers the two **General Purpose SSD** types: **GP2** (the older, credit-based one) and **GP3** (the newer, simpler, cheaper successor). Both are SSD-backed and designed for general workloads.

---

## 1. GP2 — General Purpose SSD (the credit-bucket model)

![[ebs-gp2.png]]

### 1.1 What the infographic shows

The GP2 diagram is built around one central metaphor: an **I/O credit bucket** sitting between the EBS volume (the storage icon on the left) and the EC2 instance (the chip icon on the right). Every read or write the instance performs has to be "paid for" with credits drawn from this bucket.

#### Context bar (top of the diagram)

Before diving into the bucket, the top row of the infographic gives three pieces of background:

- **Top-left — "Volumes can be as small as 1 GB, or as large as 16 TB":** GP2's size range. Size matters here because, unlike GP3, performance is tied directly to size.
- **Top-middle — "Boot volumes, low-latency interactive apps, dev & test":** the typical use cases. At the time the video was recorded, GP2 was the _default_ general-purpose disk. **AWS has since changed the default to GP3**, so today GP2 is mostly something you'd pick deliberately (for legacy compatibility) or inherit from older infrastructure.
- **Top-right — "An IO Credit is 16 KB / IOPS assume 16 KB / 1 IOPS is 1 IO in 1 second":** the unit definitions. This is the exam-critical vocabulary:
    - **I/O credit** = one 16 KB chunk of data.
    - **IOPS** = one I/O operation per second (so one 16 KB credit consumed in one second).
    - A 160 KB file transferred in one second = 10 IOPS = 10 credits spent that second.

#### The bucket (centre of the diagram)

The bucket icon in the middle, labelled **"3,000 IOPS Burst"**, is the heart of GP2. Three facts about the bucket itself, taken from the labels around it:

- **"IO 'Credit' Bucket / Capacity 5.4 million IO Credits / Fills at rate of Baseline Performance":** the bucket's total capacity is 5.4 million credits, and it refills at whatever the volume's baseline performance happens to be.
- **Rabbit + "All volumes get an initial 5.4 million IO credits. 30 minutes @ 3,000 IOPS. Great for boots and initial workloads":** new volumes start with the bucket _full_, which is enough to sustain 30 minutes at the burst rate. This is why fresh instances boot quickly and handle initial spiky workloads well.
- **The red lid on the bucket** represents the burst ceiling — credits can flow _out_ at up to 3,000 IOPS, but no faster.

The diagram is then split into two sides around this bucket:

- **Left side = credit _accumulation_** — how the bucket fills up (green arrows flowing _into_ the bucket).
- **Right side = credit _consumption_** — how the bucket drains (red arrows flowing _out_ of the bucket, up to the EC2 instance).

#### Left side — credit accumulation (how the bucket fills)

Reading from the bottom up:

- **Bottom-left thin band — tap icon + "Bucket fills with min 100 IO Credits per second Regardless of volume size":** the absolute minimum refill rate. Even a tiny 1 GB volume gets at least 100 credits/sec. This is the _floor_ — no GP2 volume ever refills slower than this.
- **Pink band above it — "Beyond the 100 minimum, the bucket fills with 3 IO credits per second, per GB of volume size (Baseline Performance)":** the size-scaled refill formula. Every GB of volume size buys you 3 credits/sec of refill. So a 100 GB volume refills at 300 credits/sec, a 500 GB volume at 1,500/sec, and so on. This rate **replaces** (not adds to) the 100 floor once it exceeds it — the crossover happens at 33.33 GB.
- **Green band on top — "Up to maximum for GP2 of 16,000 IO credits per second (Baseline Performance)":** the ceiling on the refill rate. No matter how large the volume gets, the bucket never refills faster than 16,000 credits/sec. (This cap is reached at ~5.33 TB, i.e., $16,000\:credits/sec \div 3\:credits/sec = 5333.\overline{3333}\:GB$)
- **Note attached to the green band — "Volumes larger than 1,000 GB — Baseline is above burst. Credit system isn't used & you always achieve Baseline":** once the refill rate exceeds the burst rate (which happens at 1 TB, where 3 × 1,000 = 3,000), the bucket effectively becomes irrelevant — you can always sustain your baseline directly, so there's no bursting _or_ depleting to think about.

> 🧠 **Mental model for the left side:** three stacked "fill speeds" — a guaranteed minimum trickle (100/sec), a size-based stream (3/sec per GB), capped by a hard ceiling (16,000/sec).

#### Right side — credit consumption (how the bucket drains)

- **Red up-arrows + "Burst up to 3,000 IOPS by depleting the bucket":** when EC2's workload demands more IOPS than the refill rate, the extra demand is satisfied by drawing credits _out_ of the bucket faster than they flow in. This is **bursting**, and it's capped at 3,000 IOPS (the lid).
- **The implicit balance:** if consumption > refill → the bucket drains. If consumption < refill → the bucket replenishes. If the bucket hits empty, performance collapses back to the baseline refill rate — you can no longer burst until credits build back up.

### 1.2 GP2 — the mechanics in writing

**Size:** 1 GB → 16 TB.

**The I/O unit:**

- 1 I/O credit = 16 KB of data.
- 1 IOPS = 1 I/O operation (one 16 KB chunk) in 1 second.
- If you have **no credits**, you cannot perform I/O on the disk.

**The credit bucket:**

- Capacity: **5.4 million credits**.
- Starts **full** on volume creation → enough for **30 minutes at 3,000 IOPS** straight out of the gate.
- Refills at the **baseline performance rate** of the volume.

**Baseline performance (the refill rate):**

- **Minimum floor:** 100 IOPS, regardless of size.
- **Formula:** 3 IOPS per GB of volume size.
- **Crossover point:** at 33.33 GB, the formula (3 × 33.33 ≈ 100) matches the floor. Below that → you get the 100 IOPS minimum. Above that → you get `3 × size` IOPS.

**Burst performance:**

- Any GP2 volume can burst up to **3,000 IOPS** by _spending_ credits faster than they refill.
- This is what lets a small volume handle short heavy workloads.

**The "no bucket needed" zone:**

- At **1 TB (1,000 GB)**, baseline = 3 × 1,000 = 3,000 IOPS, which equals the burst rate. Above this, baseline > burst, so the credit system becomes meaningless — you simply get your baseline all the time.
- The **maximum** GP2 performance is **16,000 IOPS**, reached at ~5.33 TB and held constant for any larger volume.

**Throughput ceiling:** 250 MB/s max.

**Use cases:** boot volumes, low-latency interactive apps, dev/test, anything without a specific reason to pick something else.

> 🧠 **Mental model:** GP2 = a piggy bank that fills slowly based on how big your volume is. Small volumes have a slow drip but a full piggy bank to start, so they can burst hard for a while. Big volumes (>1 TB) have such a fast drip that the piggy bank doesn't matter.

> ⚠️ **Exam trap:** With GP2 you are responsible for managing the credit balance across all your volumes. A workload that consistently exceeds baseline will eventually drain the bucket to zero, and disk performance will collapse to the baseline rate.

---

---

## 2. GP3 — General Purpose SSD (the simpler, cheaper successor)

![[ebs-gp3.png]]

### 2.1 What the infographic shows

GP3 drops the credit bucket mechanism entirely (so no such element appears in the illustration). Instead, the diagram shows a flat, **two-tier performance model** between the EBS volume (left) and the EC2 instance (right).

#### Context bar (top of the diagram)

- **Top-left — baby + Baby Yoda + "GP3 is like GP2 and IO1 had a baby — benefits of both":** the headline metaphor. GP3 inherits GP2's affordability and IO1's higher, more predictable performance. (IO1 is the *Provisioned IOPS* SSD type, covered in a later lesson — it lets you dial in exact performance.)
- **Top-right — "Virtual desktops, medium sized single instance databases such as MSSQL Server and Oracle DB, low-latency interactive apps, dev & test, boot volumes":** GP3's use cases — essentially the same as GP2, plus some moderate database workloads.

#### Middle — the two-tier performance model

This is the heart of the diagram: two stacked bands sitting between the EBS volume (left) and the EC2 instance (right). Read from the bottom up, because the bottom is what every volume gets by default and the top is the optional upgrade:

- **Pink band (bottom layer) with rabbit icons — "3,000 IOPS & 125 MiB/s — STANDARD":** every GP3 volume, *regardless of size*, gets this performance as a flat baseline. The rabbits = speed by default. No bucket, no formula, no floor-vs-burst — just a guaranteed standard, included in the base price.
- **Green band (upper layer) with leopard icons + red up-arrows — "Extra cost for up to 16,000 IOPS or 1,000 MiB/s":** if you need more than the standard tier, you pay separately for additional IOPS and additional throughput. The leopards = faster-than-standard speed. The red arrows visualise that these are *additive* upgrades layered on top of the standard tier — not replacements for it.

> 🧠 **Key insight:** the IOPS dial and the throughput dial are **independent**. You can pay for extra IOPS without extra throughput, or vice versa, or both. With GP2 you couldn't do this — performance was bundled with size.

#### Footer — GP3 vs GP2 at a glance

The bottom row of the diagram drives home the two headline advantages of GP3 over GP2:

- **Bottom-left — money icon + "GP3 is Cheaper (~20%) vs GP2 *Base price":** at the same capacity, GP3's base price is roughly 20% lower than GP2.
- **Bottom-right — speedometer + "4x Faster Max throughput vs GP2 — 1,000 MiB/s vs 250 MiB/s":** GP3's maximum throughput is four times that of GP2.

### 2.2 GP3 — the mechanics in writing

**Size:** 1 GB → 16 TB (same as GP2).

**Standard performance (included in the base price, every volume, every size):**
- **3,000 IOPS**
- **125 MB/s** throughput

**Provisioned extras (extra cost, on top of standard):**
- Up to **16,000 IOPS**
- Up to **1,000 MB/s** throughput

**Pricing:** base price is **~20% cheaper than GP2**. Even when you provision extra IOPS or throughput, GP3 typically still works out cheaper than GP2 for the same performance.

**Use cases:** virtual desktops, medium-sized single-instance databases (MSSQL, Oracle DB), low-latency interactive apps, dev/test, boot volumes.

> 🧠 **Mental model:** GP3 = a flat-rate broadband plan. Everyone gets the same baseline speed; if you want more, you pay for more — no credit accounting, no surprises.

---

## 3. GP2 vs GP3 — side-by-side comparison

| Attribute              | GP2                                                       | GP3                                                |
| ---------------------- | --------------------------------------------------------- | -------------------------------------------------- |
| Storage media          | SSD                                                       | SSD                                                |
| Size range             | 1 GB – 16 TB                                              | 1 GB – 16 TB                                       |
| Performance model      | Credit bucket (size-dependent)                            | Flat standard + paid extras (size-independent)     |
| Baseline IOPS          | 3 IOPS/GB (min 100, max 16,000)                           | 3,000 IOPS standard for all volumes                |
| Burst IOPS             | Up to 3,000 (by depleting credits)                        | n/a — no bursting concept                          |
| Max IOPS               | 16,000 (at ≥ 5.33 TB)                                     | 16,000 (paid extra)                                |
| Standard throughput    | Scales with size                                          | 125 MB/s for all volumes                           |
| Max throughput         | **250 MB/s**                                              | **1,000 MB/s** (paid extra) — **4× faster**        |
| Initial credit balance | 5.4 million (≈ 30 min @ 3,000 IOPS)                       | n/a                                                |
| Base price             | Higher                                                    | **~20% cheaper than GP2**                          |
| Default in AWS today   | No longer default (was default when the lecture was filmed) | **Yes — current AWS default** for new volumes    |

---

## 4. Migration & elastic volumes

- You can **change volume type on the fly** (GP2 → GP3 and vice versa) using the **Elastic Volumes** feature.
- ⚠️ **Important caveat:** when you move from GP2 to GP3, performance does **not** scale with volume size anymore. GP2 gave you `3 × GB` IOPS automatically; with GP3, anything above the 3,000 standard must be **explicitly provisioned** (and paid for). Same goes for throughput above 125 MB/s.
- Despite that, GP3 typically remains cheaper than the equivalent GP2 configuration.

---

## 5. Key takeaways (exam-focused)

- **GP2 is credit-based; GP3 is flat-rate.** This is the single biggest conceptual difference.
- **GP2 baseline = 3 IOPS/GB**, minimum 100, max 16,000. Crossover with the 100 floor is at **33.33 GB**. Crossover where baseline exceeds burst (3,000 IOPS) is at **1 TB**.
- **GP2 burst = 3,000 IOPS**, fuelled by a **5.4 million credit bucket** that volumes start full.
- **GP3 standard = 3,000 IOPS + 125 MB/s** for *every* volume regardless of size.
- **GP3 max = 16,000 IOPS + 1,000 MB/s** (paid extras).
- **GP3 is ~20% cheaper** at base price and has **4× the max throughput** of GP2.
- **Both** support boot volumes, low-latency apps, dev/test workloads.
- Use **Elastic Volumes** to swap between types — but remember GP3 doesn't auto-scale performance with size.