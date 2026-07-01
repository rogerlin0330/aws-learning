# RDS Custom

> **Exam scope:** This is a niche topic. For the SAA exam you only need surface-level awareness — knowing it _exists_ and _what gap it fills_ is usually enough.

## Overview

**RDS Custom fills the gap between standard RDS and running a database engine on EC2.**

- **Standard RDS** — fully managed database-server-as-a-service. AWS manages the OS and engine; you get **no** OS/engine access.
- **DB on EC2** — fully self-managed. Total flexibility, but you own everything from the OS upwards (high overhead).
- **RDS Custom** — the **middle ground**: use RDS and its automation, but _also_ get access to the OS and DB engine for customization.

## Key facts

- **Supported engines:** **Microsoft SQL Server** and **Oracle** only.
- **Access methods:** you can connect into the instance via **SSH, RDP, and Session Manager** to reach the OS and DB engine.
- **Runs in YOUR account:** unlike standard RDS, RDS Custom resources live in your AWS account — you **will see** the EC2 instance, EBS volumes, and S3 backups.
    - (Standard RDS hides all of this in an AWS-managed environment and only injects **elastic network interfaces (ENIs)** into your VPC for connectivity.)

## Customization workflow (database automation)

To customize without RDS automation interfering:

1. **Pause** database automation
2. Perform your customizations
3. **Resume** (re-enable) automation to make the DB production-ready

## Shared responsibility comparison

![[rds-custom.png]]

**How to read the diagram:** four columns from most-customer-managed (left) to most-AWS-managed. Colour legend at the bottom: **blue = customer**, **orange = AWS**, **gradient = shared**.

|Layer (bottom → top)|On-Premises|RDS|**RDS Custom**|EC2|
|---|---|---|---|---|
|Optimization|Customer|Customer|Customer|Customer|
|Scaling|Customer|AWS|**Shared**|Customer|
|HA|Customer|AWS|**Shared**|Customer|
|Backups|Customer|AWS|**Shared**|Customer|
|DB patches|Customer|AWS|**Shared**|Customer|
|OS patches|Customer|AWS|**Shared**|Customer|
|OS install|Customer|AWS|**Shared**|Customer|
|Hardware|Customer|AWS|AWS|AWS|

**Takeaway:** RDS Custom = AWS owns hardware, **customer owns application optimization**, and **everything in between is shared** — giving you RDS automation benefits _plus_ increased customization.

## Exam traps / gotchas

- ⚠️ **Engines:** RDS Custom = **SQL Server + Oracle only** (same pair as TDE support — don't confuse with standard RDS engine list).
- ⚠️ **Visibility:** With **RDS Custom** you _see_ EC2/EBS/S3 backups in your account; with **standard RDS** you do **not** (AWS-managed, ENI injection only).
- ⚠️ **Access:** SSH / RDP / Session Manager access is a distinguishing feature vs standard RDS (no OS access).

## 📚 Current AWS status (beyond the course)

> Added from [aws.amazon.com/rds/custom](https://aws.amazon.com/rds/custom/) — not in the transcript.

- **⚠️ RDS Custom for Oracle end of support: March 31, 2027.** After this date, RDS Custom for Oracle resources (instances, snapshots, custom engine versions) will no longer be accessible; AWS recommends migrating those workloads to **EC2**. This does **not** affect standard RDS for Oracle or **RDS Custom for SQL Server**. ([migration guidance](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/RDS-Custom-for-Oracle-end-of-support.html))
- AWS positions RDS Custom for **legacy, custom, and packaged applications** that need OS/DB-level customization while keeping managed-service benefits.