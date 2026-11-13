---
title: AWS Backup
type: service
tags: [aws-backup, backup, data-protection, vault, vault-lock, worm, cold-storage, pitr, dr]
certs: [aws-saa]
updated: 2026-11-13
---

# AWS Backup

AWS Backup is a **fully managed data protection service**. At the SAA level, think of it as a backup-and-restore product that adds significant **auditing and management oversight** on top. Its core value is **consolidation**: instead of configuring [RDS](rds.md) automated backups in one place, DynamoDB backups in another, and scripting [EBS](ebs.md) snapshots separately, AWS Backup manages both the **storage** and the **configuration** (frequency and retention) of backups centrally — across **multiple accounts** (via [Organizations](organizations.md) and [Control Tower](control-tower.md)) and **multiple regions** (cross-region copy for extra data protection).

> [!tip] Exam Tip
> The trigger phrase is *"centrally manage backups across services/accounts"* or *"in one place"*. When a scenario asks for consolidated backup management rather than a per-service feature, the answer is **AWS Backup**.

## Supported Services

AWS Backup covers most of the stateful services documented in this wiki:

| Category | Services |
| --- | --- |
| Compute | [EC2](ec2.md), VMware running within AWS |
| Block storage | [EBS](ebs.md) |
| File storage | [EFS](efs.md), FSx (all variants) |
| Databases | [Aurora](aurora.md), [RDS](rds.md), [Neptune](neptune.md), [DynamoDB](dynamodb.md), DocumentDB |
| Object storage | [S3](s3.md) |

![AWS Backup supported products diagram: a context bar at the top reads "Supports a wide range of AWS products"; below it, color-coded columns group services by category — Compute (EC2, VMware), Block Storage (EBS), File Storage (EFS, FSx), Databases (Aurora, RDS, Neptune, DynamoDB, DocumentDB), Object Storage (S3) — with arrows from every service flowing right into a single central AWS Backup icon, reinforcing the central-management theme.](../assets/aws-backup-supported-products.png)

This is an **illustrative, non-exhaustive list** of major supported services. AWS Backup support continues to expand — the authoritative current list is the [Feature availability by resource](https://docs.aws.amazon.com/aws-backup/latest/devguide/backup-feature-availability.html) table in the AWS Backup Developer Guide.

## Core Components

### Backup Plans

The central component: a backup plan configures **how** backups behave.

- **Frequency** — hourly, every 12 hours, daily, weekly, or monthly. Weekly plans choose the days of the week; monthly plans choose a day of the month. A **cron expression** can schedule snapshots as frequently as **hourly** (nothing more granular through the standard frequency picker).
- **Continuous backups** — enable for supported services to unlock **point-in-time restore (PITR)** within a window.
- **Backup window** — the **start time** and **duration** during which backups run.
- **Lifecycle** — when a backup transitions to **cold storage** and when it **expires** (see [Lifecycle and Cold Storage](#lifecycle-and-cold-storage)).
- **Vault** — the destination vault to use.
- **Region copy** — copy backups to another region.

### Backup Resources

**What** is logically being backed up — an S3 bucket, an RDS database. A resource identifies which things you want protected.

### Vaults

The **destination** where backup data is stored. You must configure **at least one**. By default a vault is **read and write** — backups in it can be deleted. Enabling [Vault Lock](#backup-vault-lock-worm) adds WORM (Write Once, Read Many) protection.

### On-Demand Backups

You are **not limited to backup plans** — AWS Backup can perform ad-hoc, **on-demand backups** whenever required.

### Point-in-Time Recovery (PITR)

Some services support restoring a resource to its state at a **specific date and time** within the retention window — for example [S3](s3.md) and [RDS](rds.md) (RDS also has its own native PITR via [automated backups](rds.md#automated-backups); with AWS Backup the equivalent is a **continuous backup**).

> [!warning] Gotcha
> Continuous backups / PITR are only available for **certain supported services** — not everything AWS Backup protects.

## Lifecycle and Cold Storage

Every backup starts in **warm storage** and can transition to a lower-cost **cold storage** tier depending on retention settings:

- **Minimum cold storage duration is 90 days.** Once transitioned, a backup must remain in cold storage for at least 90 days; deleting earlier incurs a **pro-rated charge** for the remaining days.
- In practice the console enforces that the **retention (delete-after) setting must be at least 90 days greater than the transition-to-cold setting**.
- The "transition to cold after N days" value **cannot be changed** once a backup has already transitioned to cold.

> [!warning] Gotcha — cold storage is not universal
> The transition-to-cold lifecycle only applies to **eligible resource types** (for example [EBS](ebs.md), [EFS](efs.md), DynamoDB, Timestream, VMware). AWS Backup **ignores** the cold-storage transition for other resource types — don't assume every supported service can tier to cold. (Source: [AWS Backup FAQs](https://aws.amazon.com/backup/faqs/))

## Backup Vault Lock (WORM)

Vault Lock enables **Write Once, Read Many (WORM)** mode on a vault — an additional layer of defense against accidental or malicious deletion, designed for **compliance-style** situations. **Retention periods still apply**, so backups can still age out normally; what Vault Lock prevents is **early or bypass deletion**.

There are **two modes** ([AWS Backup Vault Lock docs](https://docs.aws.amazon.com/aws-backup/latest/devguide/vault-lock.html)):

| Mode | Removable? | Cool-off period | Use case |
| --- | --- | --- | --- |
| **Governance** | Yes — users with sufficient IAM permissions can remove or change the lock | None | Restrict backup management to authorized personnel; testing before committing to Compliance |
| **Compliance** | No — once the grace time expires, the vault is **immutable**; not even AWS can change or delete it | **Grace time of at least 3 days (72 hours)** | Strict regulatory or compliance requirements |

- The **`ChangeableForDays`** parameter (CLI: `--changeable-for-days`) is what makes a lock **Compliance mode**; omit it and you get **Governance mode**.
- In Compliance mode, the lock can still be removed or reconfigured **during** the grace time. **After** the grace time it is permanent — the only escape is closing the AWS account.

> [!warning] Gotcha — three different WORM features
> Do **not** confuse **AWS Backup Vault Lock** with **S3 Glacier Vault Lock** or **[S3 Object Lock](s3.md#object-lock)**. Backup Vault Lock applies to recovery points in an **AWS Backup vault**; Glacier Vault Lock applies to an individual **S3 Glacier vault**; Object Lock applies to **S3 object versions**. If a question mentions immutable **AWS Backup recovery points**, the answer is **AWS Backup Vault Lock**. (Note the naming symmetry: Backup Vault Lock's Governance/Compliance modes mirror [Object Lock's retention modes](s3.md#object-lock).)

## Cross-Account and Cross-Region

- **Multi-account** operation uses [AWS Organizations](organizations.md) and [Control Tower](control-tower.md) — a backup policy can be applied organization-wide.
- **Cross-region copy** is configured in the backup plan and provides extra data protection for DR.

> [!tip] Exam Tip
> *"Centrally manage backups across accounts"* means **AWS Backup + Organizations/Control Tower**. *"Copy backups to another region for DR"* means **AWS Backup region copy**.

## Exam Quick Reference

| Concept | Key fact | Trigger keywords |
| --- | --- | --- |
| AWS Backup | Fully managed, centralized data protection | "consolidate / centralize backups", "one place" |
| Backup plan | Frequency, window, lifecycle, vault, region copy | "schedule backups", "retention rules" |
| Backup resource | What is backed up (RDS DB, S3 bucket, …) | "resources to protect" |
| Vault | Destination for backups; at least 1 required | "backup storage location" |
| Vault Lock (Compliance) | WORM; 72-hour minimum grace, then fully immutable (even AWS can't delete) | "immutable", "WORM", "cannot be deleted by anyone" |
| Vault Lock (Governance) | WORM; removable by privileged IAM users | "flexible", "removable lock" |
| Cold storage | Minimum 90 days; only eligible resource types | "lower-cost tier", "archive backups" |
| PITR / continuous | Restore to a point in time; select services (S3, RDS) | "point-in-time restore", "specific date/time" |
| On-demand backup | Ad-hoc backup outside a plan | "manual backup", "one-off" |
| Cross-account | Uses Organizations + Control Tower | "across multiple accounts" |
| Cross-region copy | Copy backups to another region | "DR", "copy to another region" |

> [!tip] Depth Note
> For SAA, AWS Backup is a **high-level awareness** topic — recognize *what it does* and *when to choose it*, not deep operational detail.

## References

- [AWS Backup Developer Guide — Feature availability by resource](https://docs.aws.amazon.com/aws-backup/latest/devguide/backup-feature-availability.html)
- [AWS Backup Vault Lock](https://docs.aws.amazon.com/aws-backup/latest/devguide/vault-lock.html)
- [AWS Backup FAQs](https://aws.amazon.com/backup/faqs/)
- Source note: `raw/aws-cert-study-notes/aws-solution-architect/aws_backup.md` (Cantrill SAA)
