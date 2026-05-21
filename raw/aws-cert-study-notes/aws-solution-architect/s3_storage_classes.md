# S3 Object Storage Classes — Study Notes

## Core Concept

S3 offers multiple storage classes that trade off **cost vs. access speed vs. durability guarantees**. Choosing the right one comes down to three questions:

1. How often is the data accessed?
2. How quickly do I need it back?
3. Can I tolerate losing it (replica vs. only copy)?

A foundational fact to anchor everything else: **S3 Standard replicates data across at least 3 Availability Zones**, providing **11 nines of durability**. When S3 returns `HTTP/1.1 200 OK`, your data has been durably stored. Most other classes share this AZ-replication model — the major exception is One Zone-IA.

---

## Understanding "11 Nines of Durability"

### What it means literally

"11 nines" is shorthand for **99.999999999%** — eleven 9 digits after the decimal point. It expresses the annual probability that S3 will *not* lose a given object.

Flipped around (the more useful framing): the probability of losing any single object in a year is **1 in 100 billion** (10⁻¹¹).

### Where the "10,000 years" figure comes from

The math behind the often-quoted phrase *"store 10 million objects, lose one every 10,000 years"*:

- Probability of losing one object in one year = 1 / 100,000,000,000
- Store 10,000,000 objects → expected losses per year = 10,000,000 / 100,000,000,000 = **0.0001 objects/year**
- Inverse: 1 / 0.0001 = **10,000 years between losses**

You can re-derive it at any scale:

| Objects stored | Expected losses/year | Time between losses |
| -------------- | -------------------- | ------------------- |
| 10 million     | 0.0001               | ~10,000 years       |
| 10 billion     | 0.1                  | ~10 years           |
| 100 billion    | 1                    | ~1 year             |

### Where the number itself comes from

Durability is **not measured empirically** — AWS hasn't been running long enough to observe 11 nines directly (that would take thousands of years of data). It's a **modeled figure** based on:

1. **Replication factor** — S3 Standard stores each object across ≥ 3 Availability Zones, typically with multiple copies within each AZ. Independent failure probabilities multiply together, and multiplying small numbers gives a very small number very quickly.
2. **Integrity checking** — MD5 checksums and cyclic redundancy checks (CRCs) detect bit rot or corruption, and S3 self-heals by restoring from a healthy replica.
3. **Failure rate assumptions** — modeled rates for disk failure, AZ-level events, and correlated failures, combined into a probabilistic durability model.

So 11 nines is the **designed-for** durability — what the architecture should achieve given the model's assumptions. AWS publishes it as *"designed for 99.999999999% durability of objects over a given year"* — careful wording: it's a design target, not a measured SLA.

### Durability vs. Availability — don't confuse them

|                 | Durability                 | Availability              |
| --------------- | -------------------------- | ------------------------- |
| **Question**    | Will the data still exist? | Can I reach it right now? |
| **S3 Standard** | 11 nines (99.999999999%)   | 4 nines (99.99%)          |

You can have perfect durability with degraded availability — your data is safe, you just can't get to it for a few minutes during an outage. Durability is the much stronger guarantee, and the much higher number.

### Why every S3 class quotes the same 11 nines

Standard, Standard-IA, Glacier classes, and even **One Zone-IA** all claim 11 nines. For multi-AZ classes this comes from 3-AZ replication. For **One Zone-IA**, the 11 nines is calculated *assuming the single AZ doesn't fail* — which is exactly why this is flagged as a meaningful caveat. The durability math is conditional on AZ survival, so the real-world risk profile is genuinely different even though the headline number matches.

---

## How Billing Works (general model)

Every class has some combination of these cost components:

- **Storage** ($/GB/month) — gets cheaper as you move down the tiers
- **Requests** (per 1,000)
- **Data transfer out** ($/GB) — transfer in is always free
- **Retrieval fee** ($/GB retrieved) — applies to IA and Glacier classes
- **Minimum billable object size** — penalizes storing tiny files
- **Minimum billable duration** — penalizes short-term storage
- **Monitoring/automation fee** — only Intelligent-Tiering

S3 Standard is the only class with **no retrieval fee, no minimum size, and no minimum duration**.

---

## Comparison Table

| Metric                                  | S3 Standard                                          | S3 Standard-IA                                              | S3 One Zone-IA                                    | Glacier Instant Retrieval                                 | Glacier Flexible Retrieval                                            | Glacier Deep Archive                                                 | Intelligent-Tiering                                                             |
| --------------------------------------- | ---------------------------------------------------- | ----------------------------------------------------------- | ------------------------------------------------- | --------------------------------------------------------- | --------------------------------------------------------------------- | -------------------------------------------------------------------- | ------------------------------------------------------------------------------- |
| **AZs replicated**                      | ≥ 3                                                  | ≥ 3                                                         | 1 only                                            | ≥ 3                                                       | ≥ 3                                                                   | ≥ 3                                                                  | ≥ 3 (across tiers)                                                              |
| **Durability**                          | 11 nines                                             | 11 nines                                                    | 11 nines*                                         | 11 nines                                                  | 11 nines                                                              | 11 nines                                                             | 11 nines                                                                        |
| **First-byte latency**                  | milliseconds                                         | milliseconds                                                | milliseconds                                      | milliseconds                                              | minutes to hours                                                      | hours                                                                | varies by tier                                                                  |
| **Retrieval process required?**         | No                                                   | No                                                          | No                                                | No                                                        | Yes (job)                                                             | Yes (job)                                                            | No for frequent/IA/archive-instant; Yes for archive-access & deep-archive tiers |
| **Retrieval options**                   | N/A                                                  | N/A                                                         | N/A                                               | N/A                                                       | Expedited (1–5 min), Standard (3–5 hr), Bulk (5–12 hr)                | Standard (12 hr), Bulk (up to 48 hr)                                 | Mirrors underlying tier                                                         |
| **Min. billable duration**              | None                                                 | 30 days                                                     | 30 days                                           | 90 days                                                   | 90 days                                                               | 180 days                                                             | None (90d for archive-instant tier)                                             |
| **Min. billable object size**           | None                                                 | 128 KB                                                      | 128 KB                                            | 128 KB                                                    | 40 KB                                                                 | 40 KB                                                                | None                                                                            |
| **Retrieval fee**                       | None                                                 | $0.01/GB                                                    | $0.01/GB                                          | $0.03/GB                                                  | Expedited $0.03/GB, Standard $0.01/GB, Bulk free                      | Standard $0.02/GB, Bulk $0.0025/GB                                   | None — replaced by monitoring fee                                               |
| **Can be made public?**                 | Yes                                                  | Yes                                                         | Yes                                               | Yes                                                       | No                                                                    | No                                                                   | Yes (for accessible tiers)                                                      |
| **Storage cost** (relative to Standard) | 1× (baseline)                                        | ~½×                                                         | cheaper than Standard-IA                          | cheaper than Standard-IA                                  | ~⅙×                                                                   | cheapest of all                                                      | Same as underlying tier + monitoring fee                                        |
| **Best for**                            | Frequently accessed, important, non-replaceable data | Long-lived, important data accessed infrequently (~monthly) | Replicas, regenerable data, intermediate data     | Long-lived data accessed ~quarterly, but needed instantly | Archival data, yearly access OK, minutes-to-hours retrieval tolerable | Deep archive, legal/regulatory retention, up-to-48-hour retrieval OK | Long-lived data with unknown or changing access patterns                        |
| **Avoid for**                           | (it's the safe default)                              | Small files, short-term data, frequently accessed data      | Only copy of data, critical data, frequent access | Frequently accessed data                                  | Data needing instant access, public objects                           | Primary backups (restore too slow)                                   | Static known access patterns (use Standard or IA directly)                      |

*One Zone-IA durability assumes the single AZ doesn't fail.

---

## The Glacier Retrieval Process (Important Detail)

For **Glacier Flexible Retrieval** and **Glacier Deep Archive**, you can't just `GET` an object the way you would with Standard or even Standard-IA. The data sits in a "cold" or "frozen" state and must be **explicitly thawed via a retrieval job** before it's accessible.

### How it actually works

1. **The object is visible but unreadable in place.** When you list the bucket, you still see the object key and metadata. What you *can't* do is read its contents directly — the storage class field tells you it's in Glacier.
2. **You initiate a retrieval job.** This is a specific API call (`RestoreObject`) that asks S3 to thaw a copy of the object. You specify the **tier** (speed) and **how many days** the restored copy should remain available.
3. **S3 restores a temporary copy into Standard-IA.** The original Glacier object is *not* moved — a temporary copy is staged in Standard-IA storage for the duration you specified. You access *that* copy.
4. **The temporary copy expires.** When the requested availability window ends, the Standard-IA copy is deleted. The original Glacier object remains untouched in cold storage.
5. **To "un-Glacier" permanently**, you must change the storage class back to Standard or IA — a separate operation from the retrieval job.

### Retrieval tier options

**Glacier Flexible Retrieval** — three speeds, faster = more expensive:

| Tier          | Restore time | Retrieval fee | Use case                                  |
| ------------- | ------------ | ------------- | ----------------------------------------- |
| **Expedited** | 1–5 minutes  | $0.03/GB      | Urgent, occasional access; most expensive |
| **Standard**  | 3–5 hours    | $0.01/GB      | Default; balanced cost                    |
| **Bulk**      | 5–12 hours   | **Free**      | Large volumes; no per-GB charge           |

> **Note**: Bulk retrievals from Glacier Flexible Retrieval are free of per-GB charges — you only pay the per-request fee. This is unique among Glacier classes.

**Glacier Deep Archive** — only two speeds, both much slower:

| Tier         | Restore time    | Retrieval fee | Use case                                 |
| ------------ | --------------- | ------------- | ---------------------------------------- |
| **Standard** | within 12 hours | $0.02/GB      | Default                                  |
| **Bulk**     | up to 48 hours  | $0.0025/GB    | Cheapest; large-volume archive retrieval |

Notice Deep Archive has **no Expedited option** — that's the trade-off for its rock-bottom storage price.

**Glacier Instant Retrieval** — no retrieval job required, but every `GET` incurs a retrieval fee:

| Access type | Latency      | Retrieval fee |
| ----------- | ------------ | ------------- |
| Direct GET  | milliseconds | $0.03/GB      |

This is the same $0.03/GB as Glacier Flexible Retrieval's Expedited tier — but you get the data instantly, no job needed. The trade-off vs. Flexible Retrieval is higher storage cost ($0.004 vs $0.0036/GB-month) in exchange for instant access.

### First-byte latency vs. retrieval time — they mean different things

These two terms are easy to conflate but they describe different things:

- **First-byte latency** for Standard / IA / Glacier Instant Retrieval = **milliseconds**. You issue a `GET`, bytes start flowing almost immediately.
- **First-byte latency** for Glacier Flexible / Deep Archive = **minutes to hours**. There's no direct `GET` — you must run a retrieval job first, *then* read the restored copy. The "latency" essentially includes the job runtime.

This is why Glacier Flexible and Deep Archive **cannot be made publicly accessible** and don't work with static website hosting — there's no way to serve an object on demand when accessing it requires a multi-hour job.

### Cost implications of retrieval

Every retrieval has two cost components:

1. **The retrieval fee** ($/GB retrieved) — see tables above.
2. **Temporary Standard-IA storage** while the restored copy sits in its availability window — billed at IA rates.

**Worked example — restoring 10 TB (10,000 GB):**

| Storage class & tier         | Retrieval fee | Cost for 10 TB | Wait time    |
| ---------------------------- | ------------- | -------------- | ------------ |
| Glacier Flexible — Expedited | $0.03/GB      | $300           | 1–5 min      |
| Glacier Flexible — Standard  | $0.01/GB      | $100           | 3–5 hr       |
| Glacier Flexible — Bulk      | Free          | $0 (per-GB)    | 5–12 hr      |
| Deep Archive — Standard      | $0.02/GB      | $200           | within 12 hr |
| Deep Archive — Bulk          | $0.0025/GB    | $25            | up to 48 hr  |
| Glacier Instant Retrieval    | $0.03/GB      | $300           | milliseconds |

The Bulk vs. Standard gap for Deep Archive is an **87% discount for waiting an extra ~36 hours**. This is why Glacier classes are only cost-effective for data you *rarely* touch — restoring a 10 TB archive on the fastest tier can easily cost more than a year of storing it ($0.0036/GB-month × 10,000 GB × 12 months = ~$432 for Flexible Retrieval storage vs. $300 for one Expedited restore).

### Intelligent-Tiering's hybrid model

Intelligent-Tiering's **Archive Access** and **Deep Archive** tiers use the same retrieval-job mechanism as the standalone Glacier classes. This is why those two tiers are **optional** and require application support — your app has to handle asynchronous retrieval. The first three tiers (Frequent, Infrequent, Archive Instant Access) all support direct reads with no job required.

---

## Key Distinctions Worth Memorizing

**Standard vs. Standard-IA** — Same architecture (3 AZs, 11 nines, ms latency, can be public). The IA variant cuts storage cost roughly in half but adds a retrieval fee, a 30-day minimum duration, and a 128 KB minimum size. Use IA only when access is genuinely infrequent.

**Standard-IA vs. One Zone-IA** — Identical in every cost dimension *except* storage price. Both charge the same retrieval fee ($0.01/GB), same request pricing, same 30-day minimum duration, and same 128 KB minimum object size. The **only** differences:

- One Zone-IA stores in a single AZ (vs. ≥ 3 for Standard-IA)
- One Zone-IA storage is ~20% cheaper ($0.01/GB-month vs $0.0125/GB-month)
- Availability SLA drops from 99.9% to 99.5%
- The 11 nines durability claim is conditional on the single AZ not failing

The class is built for **replica copies, regenerable intermediate data**, or anything that can be rebuilt — never the only copy of something critical. Because retrieval costs are identical, the choice between the two is purely about whether you can tolerate AZ-level data loss in exchange for 20% storage savings.

**Glacier Instant Retrieval** — Think of it as "Standard-IA extended": cheaper storage, more expensive retrievals, longer 90-day minimum. Still has millisecond first-byte latency, no retrieval job needed. Designed for data accessed maybe quarterly.

**Glacier Flexible Retrieval** — This is where the model changes. Objects become **"cold"** — visible in the bucket as a pointer, but you must run a retrieval job to access them. Retrieved objects are temporarily restored into Standard-IA. Cannot be made public. Three retrieval speeds: Expedited (1–5 min), Standard (3–5 hr), Bulk (5–12 hr). 40 KB min size, 90-day min duration.

**Glacier Deep Archive** — The cheapest tier. Objects are **"frozen"**. Same retrieval-job model as Flexible Retrieval, but slower: Standard is 12 hours, Bulk up to 48 hours. 40 KB min size, 180-day min duration. For secondary backups and regulatory retention — *not* primary backups because restore times are too long.

**Intelligent-Tiering** — Not a single class but a wrapper with 5 tiers (Frequent Access, Infrequent Access, Archive Instant Access, Archive Access, Deep Archive). AWS automatically moves objects between them based on access patterns. The first three are automatic and require no app changes; the last two are *optional* and require app support because retrieval is asynchronous. Instead of retrieval fees, you pay a small monitoring/automation fee per 1,000 objects. Only worth using when access patterns are **unknown or changing** — for static patterns, use the underlying class directly to avoid the management fee.

---

## Quick Mental Model for the Exam

When a question describes a workload, match it against these anchors:

| Workload description                                                | Storage class                  |
| ------------------------------------------------------------------- | ------------------------------ |
| "Frequently accessed, can't lose it"                                | **Standard**                   |
| "Important, accessed monthly"                                       | **Standard-IA**                |
| "Replica, regenerable, secondary copy"                              | **One Zone-IA**                |
| "Accessed quarterly but must be instant"                            | **Glacier Instant Retrieval**  |
| "Archival, yearly access, hours OK to retrieve"                     | **Glacier Flexible Retrieval** |
| "Compliance archive, rarely touched, up to 48 hours OK to retrieve" | **Glacier Deep Archive**       |
| "Access pattern unknown or shifting over time"                      | **Intelligent-Tiering**        |

**Default to S3 Standard** unless there's a specific reason to move elsewhere — and the "reason" almost always boils down to access frequency or whether the data is replaceable.
