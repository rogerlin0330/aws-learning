# EC2 Instance Store Volumes — Architecture

## Core Concept

Instance store volumes provide **block storage devices** (raw volumes) that attach to an instance, get presented to the OS, and serve as the basis for a file system used by applications — functionally like EBS, **but local rather than presented over the network**.

---

## The Three Essentials

### 1. Ephemeral, NOT Persistent

Instance store volumes are **temporary storage**. Treat them as such — never use them for data you rely on or that isn't replaceable.

Data is **lost** whenever the instance moves between hosts. Moves are triggered by:

- **Stop/Start** (forces host migration)
- **Resize / instance type change** (forces host migration)
- **Host failure** (maintenance or hardware)
- **Volume hardware failure** (the physical disk dies)

When an instance moves, it receives **new, blank** ephemeral volumes. Old volumes are wiped before reassignment — the data is gone.

> ⚠️ **Reboot ≠ data loss.** A restart keeps the instance on the same host, so data survives. Only events that _move the instance between hosts_ wipe the data.

### 2. Physically Attached to the Host (vs. EBS = Network-Based)

- Instance store volumes are **physically connected to one EC2 host** and isolated to that host.
- Only instances **on that host** can access them.
- Because they're locally attached, they deliver the **highest storage performance available in AWS** — much higher than EBS, which is presented over the network.

### 3. Attach at Launch Time ONLY

- You can **only add instance store volumes when the instance launches**.
- Unlike EBS, **you cannot attach them afterward** — there is no "add later."
- You may choose _not to use_ the allocated volumes, but you can never add more later.

> 🎯 **Exam trap:** Questions about adding instance store volumes _after_ launch — the answer is always **no, launch time only**.

---

## Architecture Detail

- Each EC2 host has its own set of instance store volumes backed by **physical devices on that host**.
- Volumes are named `ephemeral0`, `ephemeral1`, `ephemeral2`, ...
- Multiple instances on one host divide its volumes among themselves (e.g., Instance 1 uses `ephemeral0`; Instance 2 uses `ephemeral1` + `ephemeral2`).
- If an instance migrates to a new host, it still gets an `ephemeral0`, but it's a **new, blank** physical volume.

---

## Cost: "Use It or Lose It"

- Instance store is **included in the instance price**.
- Different instance types come with different selections of instance store volumes.
- Since you pay for it regardless, there's **generally no advantage to leaving allocated volumes unused**.

---

## Performance — The Trade-Off

This is the whole point: you accept the restrictions (temporary, can't survive moves/resizes/restarts onto new hosts, risk of loss) in exchange for **the highest data performance in AWS**.

|Instance Family|Type|Storage Media|Peak Throughput|Notes|
|---|---|---|---|---|
|**D3**|Storage-optimized|Traditional HDD|~4.6 GB/s|Great value for **large** amounts of storage; beats max HDD-based EBS|
|**i3**|Storage-optimized|NVMe SSD|up to ~16 GB/s|Far exceeds even high-performance EBS|

- Certain **i3** instances reach **~2 million read IOPS** and **~1.6 million write IOPS** when optimally configured.
- The IOPS gap vs. EBS is even more pronounced than the throughput gap.

---

## Availability by Instance Type/Size

- Volume **size and count vary** by instance type and size.
- **Some instance types don't support instance store at all.**
- Larger instance sizes are generally allocated **more (and larger)** volumes.

---

## Exam Power-Ups (Quick Recall)

- **Local to the host** → move hosts = lose the data.
- **Launch time only** → can't add instance store volumes later.
- **Data lost on:** host move, resize, host failure, or volume hardware failure.
- **Highest performance in AWS** — the reward for accepting all restrictions.
- **Paid for anyway** (in instance price) → no benefit to not using.
- **Temporary by nature** → never for persistent/irreplaceable data.