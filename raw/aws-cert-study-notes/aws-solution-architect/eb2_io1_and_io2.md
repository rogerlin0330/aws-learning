# EBS Volume Types — Provisioned IOPS SSD (IO1, IO2, IO2 Block Express)

> **Context:** After the General Purpose SSD types (GP2/GP3), the next tier up on EBS is **Provisioned IOPS SSD**. These volumes are built for workloads where you need _extreme_ performance, _consistently low_ latency, and the ability to dial in performance independently of how big the disk is. There are three variants: **IO1**, its successor **IO2**, and the preview-stage **IO2 Block Express**.

---

## 1. The Provisioned IOPS SSD family at a glance

![[ebs-io1-and-io2.png]]

### 1.1 What the infographic shows

The diagram shows the EBS volume (left) connecting to the EC2 instance (right) — same layout as the GP2/GP3 diagrams — but instead of a credit bucket or a two-tier band in the middle, the focal point here is the **per-instance performance** zone, which is the hard ceiling for how much these high-performance volumes can deliver to a single EC2 instance.

#### Context bar (top of the diagram)

The top of the infographic lays out the headline performance numbers of the family, plus the defining characteristic that makes provisioned IOPS volumes worth their price:

- **F1 car icon + per-volume numbers:** the F1 car signals "extreme speed". The four bullets next to it state the maximum performance achievable by a _single_ volume:
    - **Up to 64,000 IOPS per volume (4× GP2/3)** — **shared by IO1 and IO2**: both reach four times the IOPS ceiling of General Purpose SSD.
    - **Up to 256,000 IOPS per volume (Block Express)** — **Block Express only**: takes that ceiling to a completely different level (4× IO1/IO2).
    - **Up to 1,000 MB/s throughput** — **shared by IO1 and IO2**: matches GP3's max throughput and far exceeds GP2's 250 MB/s.
    - **Up to 4,000 MB/s throughput (Block Express)** — **Block Express only**: 4× the throughput of IO1/IO2.
- **Top-right — wavy line + "Consistent Low latency & jitter":** the _flat_ green wavy line is the visual cue. It's not just that latency is low — it stays low and predictable over time (low jitter = small variation between requests). This consistency is the whole reason you pay extra for these volumes; databases and latency-sensitive apps need _predictable_ response times, not just fast averages.

#### Middle — Per-Instance Performance (the main object in the diagram)

The pink panel in the middle of the diagram, labelled **"Per Instance Performance"**, is the most important concept on the entire slide. It calls out a ceiling that exists _separately_ from the per-volume ceilings above:

- **What it represents:** the maximum performance that can flow between the EBS service and a _single EC2 instance_, regardless of how many volumes you attach. Each per-volume number from the context bar is impressive on its own, but a single instance can't simply add them up indefinitely — there's a cap.
- **Stacked SSD icons on the left:** four disk icons stacked vertically symbolise that you typically need _multiple_ volumes attached to one instance to even reach this per-instance ceiling. One volume on its own cannot saturate it.
- **The three per-instance limits listed:**
    - **io1 — 260,000 IOPS & 7,500 MB/s** — to reach this with IO1 volumes (max 64,000 IOPS each), you'd need just over four volumes operating at maximum simultaneously.
    - **io2 — 160,000 IOPS & 4,750 MB/s** — slightly _lower_ than IO1, which seems counterintuitive. AWS split the next-gen capability into two: standard IO2 sits at this level, and the headroom went into Block Express (next bullet).
    - **🚀 io2 Block Express — 260,000 IOPS & 7,500 MB/s** — the rockets signal "next-level". Block Express matches IO1's per-instance ceiling, and combined with its much higher _per-volume_ limits, fewer volumes are needed to reach it.
- **Right side of the panel — flask icon + "High performance, latency sensitive workloads. I/O-intensive NoSQL & relational databases":** the use cases. Anything where sub-millisecond, _consistent_ latency is non-negotiable — think transactional databases, large OLTP systems, NoSQL clusters with strict SLAs.

> 🧠 **Why per-instance matters:** the per-instance ceiling is also influenced by the _type_ and _size_ of the EC2 instance — only the most modern, largest instances support these maximums. To achieve top performance, **three things must align**: the volume type, the instance type, and the instance size.

#### Footer — the two defining features (size-independent IOPS, and size ranges)

The bottom row drives home what makes Provisioned IOPS SSD architecturally different from General Purpose SSD:

- **Bottom-left — slider icon + "With io1/2/BlockExpress IOPS can be adjusted Independently of size":** unlike GP2 (where IOPS were tied to size) and GP3 (where the standard tier is fixed and extras are bundled differently), here the IOPS dial is fully decoupled from the size dial. You can have a small disk with massive performance.
- **Middle of the footer — speedometer + size-to-IOPS ratio ceilings:** there is still a ceiling on how aggressively you can dial up IOPS relative to size:
    - **io1: 50 IOPS/GB (MAX)** — already more than 16× GP2's 3 IOPS/GB.
    - **io2: 500 IOPS/GB (MAX)** — a 10× jump over IO1.
    - **BlockExpress: 1,000 IOPS/GB (MAX)** — double IO2 again.
    - "MAX" is in red as a reminder: these are _upper bounds_, not defaults.
- **Bottom-right — volume size ranges:**
    - **4 GB – 16 TB for io1/2** — note the minimum size is 4 GB (vs 1 GB for GP2/GP3).
    - **4 GB – 64 TB for BlockExpress** — four times the maximum capacity, in addition to the higher performance.

---

## 2. The three variants — the numbers in writing

|Attribute|IO1|IO2|IO2 Block Express|
|---|---|---|---|
|Release status|General release|General release|Preview (at time of lecture)|
|Size range|4 GB – 16 TB|4 GB – 16 TB|4 GB – **64 TB**|
|Max IOPS per volume|64,000|64,000|**256,000**|
|Max throughput per volume|1,000 MB/s|1,000 MB/s|**4,000 MB/s**|
|Max IOPS/GB ratio|50|500|1,000|
|Per-instance max IOPS|260,000|160,000|260,000|
|Per-instance max throughput|7,500 MB/s|4,750 MB/s|7,500 MB/s|

**Common to all three:**

- SSD-backed.
- IOPS configurable **independently** of volume size (subject to the IOPS/GB ratio cap).
- Designed for **consistent, sub-millisecond, low-jitter latency**.
- You pay for **both** the volume size **and** the provisioned IOPS — these are separate line items.

---

## 3. Per-instance vs per-volume — the two-ceiling rule

This is the concept most likely to show up as an exam trick, so it's worth stating plainly:

- **Per-volume ceiling:** the maximum IOPS/throughput a single volume can deliver (e.g. IO1 = 64,000 IOPS).
- **Per-instance ceiling:** the maximum IOPS/throughput that can flow between EBS and a single EC2 instance, regardless of how many volumes you attach (e.g. IO1 = 260,000 IOPS).
- To **saturate** the per-instance ceiling, you almost always need **multiple volumes**. For IO1, that's roughly **four+ volumes** each running at max.
- The per-instance ceiling is **further capped** by the EC2 instance type and size — only the newest, largest instances support the top numbers.
- **Three things must align** for maximum performance: volume type, instance type, instance size.

---

## 4. Where Provisioned IOPS SSD fits in the EBS landscape

|Need|Pick|
|---|---|
|Cheap, general-purpose, default|**GP3** (or GP2 legacy)|
|Up to 16,000 IOPS, flexible standard tier|**GP3**|
|Up to 64,000 IOPS, low-latency, **consistent**|**IO1** or **IO2**|
|Smaller volume but extreme performance|**IO1/IO2** (size-independent IOPS)|
|Up to 256,000 IOPS, up to 64 TB|**IO2 Block Express**|
|Even higher performance, but non-persistent OK|**Instance Store** (covered later)|

> ℹ️ **Instance Store note:** instance store volumes can deliver even higher performance than IO2 Block Express, but the storage is **not persistent** — it lives and dies with the instance. Provisioned IOPS SSD remains the top choice when you need both extreme performance _and_ durability.

---

## 5. Use cases

- **I/O-intensive relational databases** (Oracle, MSSQL, PostgreSQL under heavy load).
- **I/O-intensive NoSQL databases** (Cassandra, MongoDB, etc.).
- **Latency-sensitive transactional workloads** where consistent sub-millisecond response times are required.
- **Small volumes that need disproportionately high performance** — the size-independent IOPS dial makes this possible.
- Anything where **jitter** (variation in latency) would be unacceptable.

---

## 6. Key takeaways (exam-focused)

- **IO1, IO2, IO2 Block Express** are all SSD-backed, all designed for **extreme performance** and **consistent low latency**.
- **IOPS are decoupled from size** — but a max ratio applies: **50 / 500 / 1,000 IOPS per GB** for IO1 / IO2 / Block Express respectively.
- **Per-volume max IOPS:** 64,000 for IO1/IO2; **256,000 for Block Express**.
- **Per-volume max throughput:** 1,000 MB/s for IO1/IO2; **4,000 MB/s for Block Express**.
- **Per-instance ceilings exist separately** and require multiple volumes to saturate: 260,000 IOPS for IO1/Block Express; 160,000 IOPS for IO2.
- **Size ranges:** 4 GB – 16 TB for IO1/IO2; **4 GB – 64 TB for Block Express**.
- **Pricing:** you pay for size **and** provisioned IOPS separately.
- Don't memorise exact numbers; memorise the **ranges and ratios** between volume types — that's what AWS exam questions actually test.