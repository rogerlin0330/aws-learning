# EBS Snapshots, Restore & Fast Snapshot Restore (FSR)

## 1. What a Snapshot Is

A **snapshot** is a backup of an EBS volume that is **stored on S3**. Snapshots give a solutions architect three core capabilities:

1. **Efficient backup** of EBS volumes to S3.
2. **Resiliency improvement** — protecting volume data against availability zone (AZ) failures or local storage failures.
3. **Data migration** — moving volume data between AZs (using S3 as an intermediary) or between regions.

### The resiliency upgrade

- **EBS volumes are AZ-resilient** → they are vulnerable to anything that takes out an entire AZ.
- **Snapshots are stored on S3**, and S3 is a **regional** service → snapshot data becomes **region-resilient**.
- So taking a snapshot effectively _raises the resiliency level_ of your volume's data from AZ to region.

---

## 2. Snapshots Are Incremental

This is the most important conceptual point.

|Snapshot|What it stores|
|---|---|
|**First snapshot**|A **full copy** — but only of the **used data**, not the allocated size|
|**2nd snapshot onward**|**Only the changed/new data** since the previous snapshot|

- **"Used data," not "allocated size":** Use 10 GB of a 40 GB volume → the first snapshot is **10 GB**, not 40 GB.
- The first (full) snapshot can take time because it copies all used data to S3 in the background. **EBS performance is not impacted** during this — it just takes time.
- Future snapshots are **fully incremental**: only the difference since the last snapshot is stored, so they consume less space and complete faster.

### ⚠️ Exam trap — incrementals are SELF-SUFFICIENT

With traditional backup software, deleting an incremental backup could break every backup after it. **EBS does not work this way.** If you delete an incremental snapshot, AWS automatically **relocates the data** so that all later snapshots still function. Each snapshot can be treated as **self-sufficient / complete** even though it is stored incrementally.

---

## 3. Creating Volumes From Snapshots

When you create an EBS volume you can:

- Create a **blank** volume, **or**
- Create a volume **from a snapshot** (a great way to **clone** a volume).

Because the snapshot lives in S3 (regional), a volume created from it can be placed in:

- The **same AZ** as the original,
- A **different AZ in the same region** (→ move volumes between AZs), or
- **Another region** entirely, after copying the snapshot there (→ **global DR** and cross-region migration).

---

## 4. Snapshot Architecture

![EBS Snapshot Architecture](https://claude.ai/chat/ebs-snapshot-architecture.png)

Walking through the diagram:

- A **volume** in **AZ-A / us-east-1** is attached to an EC2 instance.
- Snapshots are taken to **S3**: the **Full** snapshot stores all used data; the **Incr** (incremental) snapshot stores only the changes since the last snapshot, **referencing** the full snapshot for unchanged data.
- From those snapshots you can restore a volume into:
    - The **same AZ** (Restore Volume in AZ-A),
    - A **different AZ in the same region** (Cross-AZ Restore Volume in AZ-B), or
    - A **different region** (Restore Volume in ap-southeast-2).

> **Key reminder shown in the diagram:** _1st snapshot size = volume data used; 2nd+ = data change._

---

## 5. Performance & Restore Behavior (Exam-Heavy)

|Scenario|Performance behavior|
|---|---|
|**New EBS volume (no snapshot)**|Full performance **immediately** — no initialization needed|
|**Volume restored from a snapshot**|Restores **lazily** — data is pulled from S3 in the background over time|

When restoring from a snapshot:

- Reading a block **already restored** → full EBS performance.
- Reading a block **not yet restored** → it is fetched from S3 on demand, at **lower performance** than reading from EBS directly.

### Two ways to force full performance immediately

**Option A — Force-read every block (manual)**

- Use an OS tool such as **`dd` on Linux** to read every block one-by-one.
- This forces EBS to pull _all_ snapshot data from S3 into the volume.
- Typically done **right after restore, before production use**, to guarantee performance when customers hit the data.
- Historically this was the _only_ method.

**Option B — Fast Snapshot Restore (FSR)**

- An option set **on a snapshot** that makes it **instantly restore** (no lazy load).
- You enable it per **snapshot + AZ** combination — each combination = **one FSR set**.
- **Limit: 50 FSR sets per region.**
    - Example: one snapshot enabled for **4 AZs** = **4** of your 50 allowance.
- **FSR costs extra** and **can get expensive**, especially with many snapshots.
- It achieves the same end result as the manual `dd` method — you're paying to avoid the admin overhead.

---

## 6. Billing

![[ebs-snapshot-consumption-and-billing.png]]

### Billing model — the three rules

**1. The unit is the GB-month.** You're charged a set price for every **gigabyte of snapshot data stored for one month**. It's a _rate over time_, not a flat per-snapshot fee — size and duration multiply together:

|Stored|Duration|GB-month billed|
|---|---|---|
|10 GB|1 month|**10 GB-month**|
|20 GB|½ month|**10 GB-month**|
|40 GB|¼ month|**10 GB-month**|

All three cost the same, because GB × fraction-of-month is identical. A small snapshot kept a long time and a large snapshot kept briefly can land on the same bill.

**2. You pay for USED data, not ALLOCATED size.** This is the headline cost benefit of snapshots and a classic exam point. Snapshots only copy the blocks that actually contain data:

- A **40 GB** volume with only **10 GB** written → the first (full) snapshot stores and bills for **10 GB**, never 40 GB. EBS does not charge for the empty/unused regions of a volume when snapshotting.
- **Contrast with the volume itself:** an EBS _volume_ is billed on its full **allocated** size (you pay for all 40 GB because it's reserved for you). A _snapshot_ of that same volume is billed only on the **used** data. Same source, two different billing bases — know the distinction.

**3. Incremental storage makes frequent snapshots almost free.** Because each snapshot after the first stores only _changed_ data, taking snapshots very often does **not** scale the cost the way you'd expect. Snapshotting **every 5 minutes** won't necessarily cost more than **hourly** — between any two snapshots only the delta is stored, and total stored data converges on roughly "original used data + net changes" regardless of how many snapshots you slice that into. This is why aggressive backup schedules are viable on EBS.

---

### Detailed walkthrough of the diagram

The diagram tracks **one volume (Vol. 1)** left-to-right across three points in **time**, taking a snapshot at each. The colors are the key:

- 🟦 **Blue** = the original 10 GiB of data (and the 6 GiB of it that survives unchanged).
- 🟨 **Yellow** = 4 GiB of _existing_ data that gets **overwritten/changed**.
- 🟥 **Red** = 2 GiB of brand-new data **added** to the volume.

The dashed curved arrows labelled _"X GiB referenced"_ are the crucial part: they show a later snapshot **pointing back** to data physically stored in an earlier snapshot, instead of re-storing it.

**State 1 — Volume = 10 GiB → take Snap A**

- The volume holds 10 GiB of used data (all blue).
- Snap A is the **first** snapshot, so it's a **full** copy of all used data.
- **Snap A physically stores 10 GiB → billed 10 GiB.**

**State 2 — 4 GiB changed → take Snap B**

- The volume is still **10 GiB total**: 6 GiB unchanged (blue) + 4 GiB changed (yellow). Nothing was added; 4 GiB of the original was overwritten.
- Snap B is incremental. It only needs to physically store the **4 GiB that changed (yellow)**.
- For the **6 GiB that didn't change**, Snap B simply **references Snap A** (the "6 GiB referenced" dashed arrow) — it does not copy that data again.
- **Snap B physically stores 4 GiB → billed 4 GiB.**
- Note the diagram's _"10 GiB total"_ label next to Snap A: that's reminding you the _restorable_ picture is still a complete 10 GiB volume (6 from A + 4 from B), even though B itself only added 4 GiB to storage.

**State 3 — 2 GiB added → take Snap C**

- The volume grows to **12 GiB**: the original 6 GiB (blue) + the 4 GiB changed earlier (yellow) + 2 GiB newly added (red).
- Snap C is incremental and only needs the **2 GiB of new data (red)**.
- It **references Snap A** for the unchanged 6 GiB ("6 GiB referenced") **and references Snap B** for the 4 GiB changed data ("4 GiB referenced"). Neither of those is re-stored.
- **Snap C physically stores 2 GiB → billed 2 GiB.**

### What's actually being stored vs. what each snapshot represents

|State|Volume size|Snapshot|New/changed data this snap physically stores|References (not re-stored)|**Billed**|
|---|---|---|---|---|---|
|1|10 GiB|**Snap A**|10 GiB (full)|—|**10 GiB**|
|2|10 GiB|**Snap B**|4 GiB (changed)|6 GiB → Snap A|**4 GiB**|
|3|12 GiB|**Snap C**|2 GiB (added)|6 GiB → Snap A, 4 GiB → Snap B|**2 GiB**|

**Total physical storage across all three snapshots = 10 + 4 + 2 = 16 GiB**, even though Snap C on its own represents a full, restorable 12 GiB volume. You are billed on the **16 GiB of actually-stored data** (as GB-months), _not_ 10 + 10 + 12.

**The takeaway:** at every stage a snapshot physically stores **only what is new or changed** and **references earlier snapshots** for everything that hasn't changed. That single mechanism is what makes snapshots incremental, what keeps each one cheap, and what lets you snapshot frequently without runaway cost.

---

## 7. Quick-Reference Summary

- Snapshots = EBS backups on **S3** → **region-resilient**.
- **1st snapshot = full** (used data only); **subsequent = incremental** (changes only).
- Incrementals are **self-sufficient** — deleting one does **not** break later snapshots.
- Restore can target **same AZ / cross-AZ / cross-region**.
- New volume = instant performance; **restored-from-snapshot volume = lazy restore** (slower until warmed).
- Force performance via **`dd`** (free, manual) or **FSR** (instant, costs extra, **50 sets/region**, counted per snapshot+AZ).
- Billing = **GB-month**, on **used** (not allocated) data; frequent snapshots stay cheap.

---

### 🔑 Top exam traps

1. First snapshot bills for **used data**, not allocated volume size.
2. Deleting an incremental snapshot is **safe** — AWS preserves later snapshots.
3. Restored volumes load **lazily** — use `dd` or **FSR** to pre-warm.
4. **FSR limit = 50 per region**, counted **per snapshot + AZ combination** (not per snapshot).
5. FSR **costs extra**; manual block read achieves the same result for free.