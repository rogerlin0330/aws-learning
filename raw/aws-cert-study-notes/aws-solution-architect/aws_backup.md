# AWS Backup

## Overview

**AWS Backup** is a **fully managed data protection service**. At the SAA level, think of it as a **backup and restore** product — but it adds significant **auditing and management oversight** on top.

Its core value is **consolidating the management and storage of backups in one place**:

- Across **multiple accounts** (via **Organizations** + **Control Tower**)
- Across **multiple regions** (region copy for extra data protection)

> 🎯 The main day-to-day benefit is **consolidation**. Instead of configuring RDS backups in one place, DynamoDB in another, and scripting EBS snapshots separately, AWS Backup does all of it centrally — managing both the **storage** and the **configuration** (frequency + retention) of backups.

---

## Diagram Walkthrough — Supported Products

![[aws-backup-supported-products.png]]

**How to read this diagram:** The **context bar** at the top states the theme — _"Supports a **wide range** of AWS products."_ The **main body** groups supported services into logical categories (color-coded columns), with **arrows all flowing right into the single AWS Backup icon** — reinforcing the central-management theme. The service **category labels** run along the footer.

Reading the columns left → right:

|Category|Services shown|
|---|---|
|**Compute**|EC2, VMware (running within AWS)|
|**Block Storage**|EBS|
|**File Storage**|EFS, FSx (the various types)|
|**Databases**|Aurora, RDS, Neptune, DynamoDB, DocumentDB|
|**Object Storage**|S3|

> ⚠️ This is an **illustrative (non-exhaustive) list** of major supported services, not the complete set. AWS Backup support continues to expand — always check the **[Feature availability by resource](https://docs.aws.amazon.com/aws-backup/latest/devguide/backup-feature-availability.html)** table for the current authoritative list.

---

## Key Concepts & Components

### 1. Backup Plans

The central component where you configure **how** backups behave:

- **Frequency** — hourly, every 12 hours, daily, weekly, or monthly. A **cron expression** can create snapshots as frequently as **hourly**.
    - _Weekly_ → choose which **days of the week**.
    - _Monthly_ → choose a specific **day of the month**.
- **Continuous backups** — enable for certain supported services to unlock **point-in-time restore (PITR)** within a window.
- **Backup window** — controls the **start time** and **duration** of the backup window.
- **Lifecycle** — defines when a backup transitions to **cold storage** and when it **expires**.
- **Vault** — sets the destination vault to use.
- **Region copy** — copy backups from one region to another.

### 2. Backup Resources

**What** is logically being backed up — e.g., an S3 bucket, an RDS database. A "resource" identifies which things you want backed up.

### 3. Vaults

The **destination** for backups — where the backup data is stored. You must configure **at least one**.

- By default, vaults are **read + write** (backups can be deleted).
- **AWS Backup Vault Lock** can be enabled for **WORM (Write Once, Read Many)** mode.

### 4. On-Demand Backups

You are **not limited to backup plans** — AWS Backup can perform **on-demand backups** as required.

### 5. Point-in-Time Recovery (PITR)

Some services support restoring to the state of a resource at a **specific date and time** within the retention window. Examples: **S3** and **RDS**.

---

## Lifecycle & Cold Storage

- Every backup starts in **warm storage**. 📚 Depending on retention, it can transition to a lower-cost **cold storage** tier.
- **Minimum cold storage duration = 90 days.** Once transitioned, a backup must remain in cold storage for at least 90 days.
- 📚 In practice this means the **retention (delete-after) setting must be at least 90 days _greater_ than the transition-to-cold setting** — the console enforces this. Backups deleted before 90 days in cold incur a **pro-rated charge** for the remaining days.
- 📚 The "transition to cold after days" value **cannot be changed** once a backup has already transitioned to cold.

> ⚠️ **Exam trap — cold storage is not universal.** The transition-to-cold lifecycle only applies to **eligible resource types** (e.g., EBS, EFS, DynamoDB, Timestream, VMware). AWS Backup **ignores** the cold-storage transition for other resource types. Don't assume every supported service can tier to cold. _(Source: [AWS Backup FAQs](https://aws.amazon.com/backup/faqs/))_

---

## AWS Backup Vault Lock (WORM)

Enables **Write Once, Read Many (WORM)** mode on a vault — an additional layer of defense against accidental or malicious deletion.

**From the course transcript:**

- Once enabled, a **72-hour cool-off period** applies.
- Once fully active, **nobody — including AWS — can delete anything** from the vault.
- Designed for **compliance-style** situations.
- **Retention periods still apply**, so backups can still age out. What it prevents is **early / bypass deletion**.

> ⚠️ **Do NOT confuse** AWS Backup Vault Lock with **S3 Glacier Vault Lock** or **S3 Object Lock**. Backup Vault Lock applies to recovery points in an **AWS Backup vault**; Glacier Vault Lock applies to an **individual S3 Glacier vault**. 🎯 If a question mentions immutable **AWS Backup** recovery points → **AWS Backup Vault Lock**.

### 📚 Discrepancy — Two Vault Lock Modes (transcript describes only one)

> ⚠️ The transcript describes a single behavior (72-hour cool-off, fully immutable, even AWS can't delete). **Current AWS Backup Vault Lock actually has TWO modes.** The transcript's description matches **Compliance mode** only. This is worth knowing, as newer exam questions may distinguish the two.

|Mode|Removable?|Cool-off period|Use case|
|---|---|---|---|
|**Governance mode**|✅ Yes — users with **sufficient IAM permissions** can remove/change the lock|No cooling-off requirement|Restrict management to authorized personnel; testing before compliance|
|**Compliance mode**|❌ No — once grace time expires, **immutable**; not even AWS can change/delete it|**Grace time ≥ 3 days (72 hours)**|Strict regulatory / compliance requirements|

- 📚 The **`ChangeableForDays`** parameter (CLI: `--changeable-for-days`) is what makes a lock **Compliance mode**. Omit it → **Governance mode**.
- 📚 In Compliance mode, during the **grace time** you can still remove/reconfigure the lock. **After** grace time it's permanent — the only escape is closing the AWS account.
- _(Source: [AWS Backup Vault Lock docs](https://docs.aws.amazon.com/aws-backup/latest/devguide/vault-lock.html))_

---

## Cross-Account & Cross-Region

- **Multi-account** operation uses **AWS Organizations** and **Control Tower**.
- **Cross-region copy** provides extra data protection (configured in the backup plan).

🎯 **Trigger keywords:** "centrally manage backups across accounts" → **AWS Backup + Organizations/Control Tower**. "Copy backups to another region for DR" → **AWS Backup region copy**.

---

## ⚠️ Exam Traps & Gotchas

- **Vault Lock ≠ Glacier Vault Lock ≠ S3 Object Lock** — three different features; don't mix them.
- **Cold storage 90-day minimum** — early deletion = pro-rated charge; retention must exceed transition + 90 days.
- **Cold storage isn't available for all resource types** — the transition is ignored for ineligible services.
- **Continuous backups / PITR** are only for **certain supported services** (e.g., S3, RDS) — not all.
- **72-hour cool-off** specifically applies to **Compliance mode** Vault Lock (the transcript's scenario). Governance mode has no such immutability.
- You are **not limited to backup plans** — **on-demand backups** exist too.
- Cron expressions allow backups **as frequent as hourly** (not more granular via the standard frequency picker).

---

## Quick-Reference Table

|Concept|Key fact|🎯 Trigger keyword|
|---|---|---|
|**AWS Backup**|Fully managed, centralized data protection|"consolidate / centralize backups", "one place"|
|**Backup Plan**|Frequency, window, lifecycle, vault, region copy|"schedule backups", "retention rules"|
|**Backup Resource**|_What_ is backed up (RDS DB, S3 bucket, etc.)|"resources to protect"|
|**Vault**|_Destination_ for backups; ≥1 required|"backup storage location"|
|**Vault Lock (Compliance)**|WORM, 72-hr cool-off, then fully immutable (even AWS can't delete)|"immutable", "WORM", "cannot be deleted by anyone"|
|**Vault Lock (Governance)**|WORM, removable by privileged IAM users|"flexible", "removable lock"|
|**Cold storage**|Min **90 days**; only eligible resource types|"lower-cost tier", "archive backups"|
|**PITR / Continuous**|Restore to a point in time; select services (S3, RDS)|"point-in-time restore", "specific date/time"|
|**On-demand backup**|Ad-hoc backup outside a plan|"manual backup", "one-off"|
|**Cross-account**|Uses Organizations + Control Tower|"across multiple accounts"|
|**Cross-region copy**|Copy backups to another region|"DR", "copy to another region"|

---

## Depth Note

> 📚 For SAA, AWS Backup is a **high-level awareness** topic — you need to recognize _what it does_ and _when to choose it_, not deep operational detail. The transcript explicitly frames this as foundational knowledge sufficient for the exam unless a course adds specialized deep-dive videos.