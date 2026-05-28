# EBS Volume Types — HDD-Based (ST1 & SC1)

> **Context:** Rounding out the EBS volume types, after the SSD families (GP2/GP3 and IO1/IO2/Block Express) come the **HDD-based** types. HDD = hard disk drive, meaning **physical moving parts**: spinning platters and read/write heads on little robot arms that travel across them. Moving parts = slower, so you only reach for these in specific, cost-sensitive situations. There are two in general use: **ST1** (Throughput Optimized HDD) and **SC1** (Cold HDD). _(A third, older HDD type exists but is legacy and not covered.)_

---

## 1. The two HDD types side-by-side

![[ebs-hdd-based-volumes.png]]

### 1.1 What the infographic shows

Unlike the SSD diagrams, this one is split down the middle by a dotted vertical line into **two mirrored halves** — **ST1 on the left**, **SC1 on the right**. Each half follows the same top-to-bottom layout: a header row of icons, a size range, a use-case description, and the performance numbers. Reading them as a pair makes the trade-off obvious: SC1 is a cheaper, slower version of ST1.

#### A note on the key unit quirk (applies to both halves)

Before reading either side, internalise the single most important HDD detail, shown as **"(1MB)"** next to the IOPS figures: on HDD-based volumes, **one I/O operation moves a 1 MB block** (versus 16 KB on SSD). This is why the IOPS number and the MB/s number line up so neatly — **500 IOPS = 500 MB/s**, **250 IOPS = 250 MB/s**. The credit-bucket mechanism from GP2 still applies here, but it's measured in **MB/s per TB** rather than IOPS.

#### Left half — ST1 (Throughput Optimized HDD)

- **Header icons — money + train + hard disk + "st1 / Throughput Optimized":** the money icon labelled **"Cheap"** signals ST1 is less expensive than any SSD type. The **train** is the key metaphor — a train is not agile (bad at darting around = bad at random access) but moves a huge load steadily in one direction (great at sequential throughput). The hard-disk icon plus "st1 / Throughput Optimized" names the volume.
- **Size range — three disk icons + "125 GB – 16 TB":** ST1 volumes start at 125 GB (much larger minimum than SSD's 1–4 GB) and go up to 16 TB.
- **Use case text — "Frequent Access, Throughput-intensive, Sequential":** ST1 is for data accessed _often_ and _sequentially_, where total throughput matters more than IOPS. The flask icon marks this as the use-case callout.
- **Examples — "Big data, data warehouses, log processing":** classic sequential, throughput-heavy workloads.
- **The red dashed vertical bars in the centre:** a visual motif representing throughput "flowing" — the taller, denser red bars (vs SC1's thinner blue ones) signal ST1's higher performance.
- **Performance numbers (right of the bars):**
    - **Max 500 IOPS (1MB)** → because each I/O is 1 MB.
    - **Max 500 MB/s** → the direct consequence of 500 × 1 MB.
    - **40 MB/s per TB Base** → baseline performance scales at 40 MB/s for every 1 TB of volume size.
    - **250 MB/s per TB Burst** → can burst up to 250 MB/s per TB, capped by the 500 MB/s / 500 IOPS overall maximum.

#### Right half — SC1 (Cold HDD)

- **Header icons — money + iceberg + hard disk + "sc1 / Cold HDD" + penguin:** the money icon here says **"Cheaper"** — SC1 is the **lowest-cost EBS storage available, full stop**. The **iceberg and penguin** reinforce "cold" — infrequently accessed, archival data that just sits there.
- **Size range — three disk icons + "125 GB – 16 TB":** identical to ST1.
- **Use case text — "Lowest cost HDD volume designed for less frequently accessed workloads":** SC1 trades performance for maximum economy.
- **Examples — "Colder data requiring fewer scans per day":** the **spider web** icon reinforces "rarely touched" (cobwebs = neglected). If your data needs only a few loads or scans per day, SC1 fits.
- **The blue dashed vertical bars in the centre:** the same throughput motif as ST1, but **thinner and sparser** — a deliberate visual cue that SC1 delivers significantly less performance.
- **Performance numbers (right of the bars):**
    - **Max 250 IOPS (1MB)** → half of ST1's IOPS ceiling.
    - **Max 250 MB/s** → 250 × 1 MB.
    - **12 MB/s per TB Base** → much lower baseline than ST1's 40.
    - **80 MB/s per TB Burst** → much lower burst than ST1's 250.

---

## 2. The numbers in writing

|Attribute|ST1 (Throughput Optimized)|SC1 (Cold HDD)|
|---|---|---|
|Nickname|"The fast hard drive"|"The cold/cheap hard drive"|
|Relative cost|Cheap (cheaper than SSD)|**Cheapest EBS storage of all**|
|Size range|125 GB – 16 TB|125 GB – 16 TB|
|I/O block size|1 MB|1 MB|
|Max IOPS|500|250|
|Max throughput|500 MB/s|250 MB/s|
|Baseline performance|40 MB/s per TB|12 MB/s per TB|
|Burst performance|250 MB/s per TB|80 MB/s per TB|
|Access pattern|Frequent, sequential, throughput|Infrequent, cold, archival|
|Typical use cases|Big data, data warehouses, logs|Cold data, archives, few scans/day|

---

## 3. How the HDD credit bucket differs from GP2's

The HDD types reuse the **same credit-bucket concept** introduced with GP2, with one key difference:

- **GP2:** the bucket is measured in **IOPS** (16 KB I/O credits), refilling at a rate tied to volume size.
- **ST1 / SC1:** the bucket is measured in **MB/s** (1 MB I/O blocks), with baseline and burst both expressed **per TB of volume size**.

So a larger HDD volume gets proportionally more baseline _and_ more burst throughput — bigger volume, faster disk — just as a larger GP2 volume earns more IOPS.

---

## 4. Why "1 MB I/O" matters

This is the easiest HDD detail to trip over:

- On **SSD** volumes, 1 IOPS = one **16 KB** operation.
- On **HDD** volumes, 1 IOPS = one **1 MB** operation.
- That's why the IOPS and MB/s numbers are the same value on HDD: **500 IOPS × 1 MB = 500 MB/s**. The large block size is also _why_ HDDs suit sequential workloads — reading big contiguous chunks plays to a spinning disk's strength, while random small reads (where SSDs win) play to its weakness.

---

## 5. Choosing between them (and when to avoid both)

A simple decision flow, straight from the lesson:

1. **Do you need IOPS / random access / high performance?** → **Avoid both HDD types.** Use SSD (GP3, or IO1/IO2 for extreme needs).
2. **Can you tolerate SC1's lower performance?** → Use **SC1**. It's the cheapest, perfect for anything not accessed day-to-day.
3. **Otherwise** → Use **ST1**. Still cheap, but with enough throughput for frequently accessed sequential workloads.

> 🧠 **Rule of thumb:** HDD = economy of _storage_, not performance. The moment IOPS or random access enters the requirements, HDD is the wrong answer.

---

## 6. Key takeaways (exam-focused)

- **Two HDD types in use: ST1 (Throughput Optimized) and SC1 (Cold).** A third is legacy.
- **HDD I/O is measured in 1 MB blocks** (vs 16 KB on SSD) — so IOPS and MB/s numbers match.
- **ST1: max 500 IOPS / 500 MB/s**, baseline **40 MB/s per TB**, burst **250 MB/s per TB**.
- **SC1: max 250 IOPS / 250 MB/s**, baseline **12 MB/s per TB**, burst **80 MB/s per TB**.
- **Both range 125 GB – 16 TB** (note the 125 GB _minimum_ — larger than SSD).
- **Both use the GP2-style credit bucket**, but measured in **MB/s per TB**, not IOPS.
- **ST1 = frequent, sequential, throughput-heavy** (big data, warehouses, logs).
- **SC1 = infrequent, cold, archival** — the **lowest-cost EBS storage**.
- **Neither is suitable for IOPS-heavy or random-access workloads** — use SSD instead.