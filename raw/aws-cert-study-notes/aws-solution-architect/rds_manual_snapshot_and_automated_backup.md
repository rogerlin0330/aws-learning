# RDS — Backups, Snapshots & Restore

## Overview / Problem Context

RDS has **two types of backup-like functionality**: **manual snapshots** and **automated backups**. Both are stored in **S3, but inside AWS-managed buckets** — so they are visible in the RDS console but **never appear in your own S3 console**. 🎯 ⚠️ This is a known exam trap.

Because the data lands in S3, all backups are **regionally resilient** (S3 replicates across multiple AZs within the region).

**Performance impact of taking a backup:**

- **Multi-AZ**: backup is taken from the **standby** instance → I/O pause happens there → no noticeable application impact.
- **Single-AZ** (e.g. test/dev): backup is taken from the **only** instance → you may see a performance pause.

---

## Reading the Diagram

![[rds-backup-snapshot-and-automated-backup.png]]

**Reading order: context bar → middle objects → footer.**

- **Context bar (top):** Title "RDS – Backups – General". Everything sits inside the **`us-east-1`** region (dashed blue boundary).
- **Middle (the architecture):**
    - The blue box is **AZ-B**, holding the **SingleAZ / Standby** RDS instance. This is the instance that backups are _read from_ (standby on Multi-AZ; the sole instance on Single-AZ).
    - The green box beneath it represents the **backup/snapshot data being produced** from that instance.
    - **Left text block** describes the snapshot mechanic: _First Snap is **FULL**_ (sized to consumed data), and _then onward = **Incremental**_ (only changed blocks). ⚠️ This rule applies to the data feeding **both** arrows below — see the note on it.
    - **Three arrows leave the green box — and this is the key to the diagram:**

|Arrow|Color / position|What it represents|Destination|
|---|---|---|---|
|1|**Blue, left**|**Manual snapshots**|S3 (left bucket)|
|2|**Orange, middle**|**Automated backups** (the daily snapshot)|S3 (middle/right bucket)|
|3|**Orange, right**|**Transaction logs** — the _additional_ stream that automated backups capture on top of the daily snapshot|S3 (same bucket)|

- The orange gear labelled **"Every 5 minutes – Transaction Logs"** (bottom right) cross-references **arrow 3** specifically. These logs are what give automated backups their PITR capability — and they are **exclusive to automated backups**. Manual snapshots have no transaction-log stream.
- **Footer / lower text:**
    - **Left labels** ("Automated Backups", "Manual Snapshots", "AWS Managed S3 Buckets") name the concepts; the **red arrow** points at S3 to reinforce that the managed buckets are where everything lands.
    - **Bottom right:** _Retention **0 to 35 days**_ — applies to **automated backups only** (manual snapshots never expire).

> ⚠️ **Correcting a common misread of this diagram:** It is tempting to attach "First Snap is FULL → Incremental" only to the manual-snapshot (blue) arrow. In fact, AWS treats an **automated backup as an automated snapshot**, so the full-first-then-incremental behavior is true of **both** the blue (manual) and orange (automated) paths. 📚 _AWS docs: "The first snapshot of a DB instance contains the data for the full database. Subsequent snapshots… are incremental."_ — so:
> 
> - **Arrows split the two mechanisms** (manual vs. automated vs. its transaction logs).
> - **The full→incremental rule is shared** by both snapshot types.
> - **Transaction logs (arrow 3) and the 0–35 day retention are automated-backup-only.**

---

## Manual Snapshots

- **Not automatic** — you trigger them explicitly, or via a script / custom application, against an RDS **instance**.
- Stored in AWS-managed S3; behave like **EBS snapshots**.
- Taken at the **instance level** → captures **all databases** in the instance, not a single database.
- **First snapshot = full** (sized to consumed data); **subsequent = incremental** (changed blocks only).
    - Initial snapshot can take a while; later ones are faster — **except** on high-change-rate databases, where even later snapshots can take significant time.
- A brief **I/O interruption** occurs (impacts app on Single-AZ; absorbed by standby on Multi-AZ).
- 🎯 **Snapshots do NOT expire.** They persist **past deletion of the RDS instance** and are only removed when **you** delete them manually (or via an external process). ⚠️ Heavily tested.
- **Frequency is your choice** (hourly / daily / weekly / monthly). More frequent snapshots → **lower RPO** (less maximum data loss on failure).
    - 📚 Manual snapshot limit is **100 per Region**. _(Editorial addition — not in the video; AWS docs. ⚠️ review)_

---

## Automated Backups

- Occur **once per day** during a defined **backup window** (AWS-chosen at random, or set to fit your business).
    - Single-AZ: schedule during low/no-use periods (I/O pause). Multi-AZ: not a concern (runs off standby).
- Architecturally an **automated snapshot**: first is full, subsequent are incremental change-only — same mechanic as a manual snapshot.
- **PLUS transaction logs every 5 minutes** → written to S3 (the right-hand orange arrow). Transaction logs record the **actual operations** that change data.
- Daily automated backup + 5-minute transaction logs together enable **point-in-time recovery (PITR)** with **~5-minute granularity** → theoretical **5-minute RPO**. 🎯
    - ⚠️ 🎯 **PITR requires automated backups.** Manual snapshots alone cannot do PITR — they restore only to the snapshot's fixed moment.
- **Retention: 0 to 35 days** (settable per instance). 🎯
    - **0 = automated backups disabled** (also disables PITR).
    - **35 = maximum.** You can restore to any point within that window; anything older than the retention period is auto-removed by AWS.

---

## Backup Retention on Deletion ⚠️

- When you **delete the database**, you _can_ choose to **retain automated backups** — **BUT** 🎯 they **still expire** based on the retention period. Retaining them does not make them permanent.
- The **only** way to preserve an instance's contents **beyond the 35-day max** is to take a **final manual snapshot** on deletion. That final snapshot is **fully under your control** and must be **manually deleted**.

---

## Cross-Region Replication

- RDS can **replicate backups to another AWS Region** — this means **both snapshots AND transaction logs**.
- ⚠️ 🎯 **NOT the default.** Must be **explicitly enabled** within automated backups.
- **Charges apply** for the cross-region data copy **and** for storage in the destination region.

---

## Restores

🎯 **A restore always creates a brand-new RDS instance** (true for both snapshot restores and automated-backup PITR).

- ⚠️ The new instance has a **new endpoint address** → you **must update applications** to point at it.

|Restore source|Recovery point|RPO quality|
|---|---|---|
|**Manual snapshot**|Fixed to the **single point in time** the snapshot was created|Often suboptimal (unless taken right before failure)|
|**Automated backup (PITR)**|**Any chosen point in time** (down to ~5-min granularity)|Much better — can pick minutes before failure|

**How PITR works:** restore the **closest snapshot**, then **replay transaction logs** forward from that point up to your chosen time.

⚠️ **Restores are NOT fast.** Restoring a large database can take significant time → this is your **RTO**, and it must be factored into DR / business-continuity planning. Backups protect against failure and data corruption, but the restore _time_ is the cost.

> 📚 **Read replicas** (covered in a separate lesson) are the mechanism for significantly improving **RTO** when recovering from failure — backups improve RPO, replicas improve RTO.

---

## 🎯 Exam Quick Reference

|Trigger keyword|Answer|
|---|---|
|Backups visible in S3 console?|**No** — AWS-managed buckets, RDS console only|
|Regionally resilient backups|Stored in S3 (multi-AZ replication)|
|Backup taken from standby|**Multi-AZ** (no perf impact)|
|First backup full, rest incremental|**Both** manual snapshots **and** automated backups|
|Snapshot expiry|**Never** — manual deletion only; survives instance deletion|
|Automated backup retention|**0–35 days** (0 = disabled)|
|Transaction log frequency|**Every 5 minutes** (automated backups only)|
|Achievable RPO with automated backups|**~5 minutes** (PITR)|
|"Restore to exact moment before failure"|**Automated backup PITR**, not manual snapshot|
|Keep data past 35 days on deletion|Take a **final manual snapshot**|
|Cross-region backup replication|Manual / explicit opt-in; **not** default; extra cost|
|Restore behavior|Creates a **new instance + new endpoint**|
|Snapshot vs automated scope|Whole **instance** (all DBs), not one database|

## ⚠️ Traps & Gotchas

- RDS backup buckets are **invisible in S3** — don't expect to find them there.
- The three diagram arrows separate **manual snapshot / automated backup / transaction logs** — but the **full→incremental** rule is shared by both snapshot types.
- **PITR needs automated backups** (transaction logs); a manual snapshot alone restores only to its fixed moment.
- "Retain on delete" for automated backups **still expires** — only a **final snapshot** is permanent.
- **Cross-region replication is opt-in**, not automatic.
- A restore **never** overwrites the existing DB — it always spins up a **new endpoint** you must re-point apps to.
- Restore **RTO is slow** for large DBs — backups alone are a poor RTO solution; pair with read replicas.
- Snapshot/automated backup is **instance-level** (all databases) — not a per-database backup.