---
title: RDS — Relational Database Service
type: service
tags: [rds, database, relational, ec2, ebs, vpc, subnet-group, multi-az, read-replica, synchronous-replication, asynchronous-replication, aurora, dr, high-availability, acid, cost]
certs: [aws-saa]
updated: 2026-06-25
---

# RDS — Relational Database Service

RDS is AWS's **managed database server** product. AWS owns the hardware, OS, engine installation, and most of the engine maintenance, so you operate a database without operating its host. This page also covers the prior question every architect must answer first: **should the database run on plain [EC2](ec2.md) at all?**

RDS databases are almost always relational and use the [ACID transaction model](../concepts/db-transaction-model.md) — which is the exam's default assumption whenever it says "ACID".

## Databases on EC2 — Why You Usually Shouldn't

You *can* run any database directly on an [EC2](ec2.md) instance. The honest framing is that doing so is **always a bad idea** — the only real question is whether the benefit to you or the business outweighs that fact. It can always be done, as long as you accept the negatives.

Two architectures people picture:

| Architecture | Description |
| --- | --- |
| **Single instance** | One larger EC2 instance running the DB, the application, and often a web server (Apache). Everything in one place, one AZ. |
| **Split architecture** | Two smaller instances — app/web server separated from the database, in the same AZ or split across AZs. |

Splitting introduces a **dependency**: the app and DB instances need reliable communication or the app breaks. It also touches cost — traffic between [AZs](../concepts/resource-scope.md) in the same region is charged, while communication over **private IPs in the same AZ is free**.

### When EC2-hosted DB is the right answer

There *are* exam questions where "run the database on EC2" is correct. The justifications that genuinely hold up:

| Reason | Verdict |
| --- | --- |
| Need a **DB engine or version AWS doesn't provide** | ✅ Often genuinely justified (niche/emerging databases) |
| Need a **specific OS + DB version combo** AWS doesn't offer | ✅ Valid |
| Need a **specific architecture / replication** AWS doesn't provide | ✅ Valid |
| Need **OS-level / root access** to the DB host | ⚠️ Question hard — rarely truly needed; often a vendor demand, and many vendors now support managed DBs. **RDS Custom** also closes much of this gap |
| **Root-level DB tuning** | ⚠️ Question — RDS exposes many historically root-only parameters without OS access |
| **Decision-makers just want it** on EC2 | Possibly unreasonable, but you may not have a choice |

### The negatives you take on

- **Admin overhead** — you manage the EC2 instance *and* the DB host (patching, version compatibility, out-of-hours fault-finding).
- **Backups & DR** — significant extra complexity that managed products automate.
- **Single AZ** — the instance and its [EBS](ebs.md) volume live in **one AZ**; an AZ failure can take DB access down. You own the snapshot/backup-to-S3 process.
- **No serverless / scaling** — EC2 is on or off; it can't easily scale down or absorb bursty load, and it sets a base minimum cost (the hourly rate).
- **Replication, performance, advanced features** — all yours to build, monitor, and forgo. AWS invests heavily in DB performance optimizations you can't easily replicate.

> [!warning] Gotcha
> An EC2-hosted database is **single-AZ by nature** — instance plus EBS volume in one AZ. That is the architectural reason it loses to RDS Multi-AZ in HA scenarios, and the reason "DB on EC2" is the *wrong* answer to most resilience questions even though it's the *right* answer to unsupported-engine ones.

## What RDS Actually Is

> [!tip] "Database **server** as a service", not "database as a service"
> With DBaaS you pay and receive *a database*. With RDS you pay for and receive a **database server (instance)** — so **one RDS instance can host multiple databases**. This distinction is testable.

- **Managed**: AWS handles hardware, OS, install, and much of the engine upkeep.
- **No SSH and no OS access.** The exception is **RDS Custom**, a variant that grants more low-level access.
- **Engines**: MySQL, MariaDB, PostgreSQL, plus commercial Oracle and Microsoft SQL Server (commercial engines carry licensing cost).

> [!warning] Aurora ≠ RDS
> **Amazon Aurora** is a custom AWS-built engine (compatible with some of the above, but designed by AWS). Many RDS features differ for Aurora — treat them as separate products. The most exam-relevant difference is **storage**: RDS gives each instance its **own dedicated EBS volume**; Aurora uses a shared cluster storage layer. Don't carry RDS storage facts over to Aurora.

## Architecture

![RDS architecture diagram: VPC1 in us-east-1 spanning three AZ columns (AZA/AZB/AZC). A subnet group defines which subnets RDS may use. A Primary instance replicates synchronously (green) to a Standby in another AZ, and asynchronously (orange) to a cross-region Read Replica in ap-southeast-2. Backups and snapshots flow to an AWS-managed S3 bucket. Each instance has its own dedicated EBS storage. RDS sits inside the VPC, reachable privately from on-premises over VPN/Direct Connect, or optionally given public addressing for internet access.](../assets/rds-architecture.png)

Read the diagram from the outside in: the outer panels are external networks (**On-Premises** left, **Internet** right); the central panel is the **us-east-1** region holding **VPC1**, split into three AZ columns; a second region (**ap-southeast-2**) at the bottom hosts a cross-region element. Region/AZ names and the public/private split are **illustrative patterns**, not fixed values.

Key elements:

- **RDS runs inside a [VPC](vpc.md)** — it is *not* a public-zone service like S3 or DynamoDB. It needs subnets in specific AZs.
- **Subnet group** defines *which subnets RDS may use* (see below).
- **Private subnets** (the normal case) — reachable from inside the VPC or from connected networks over **VPN / Direct Connect**.
- **Public addressing** can optionally be given to instances in public subnets for internet access — possible but a **security anti-pattern**.
- **Primary → Standby** (green): **synchronous** replication, same region, different AZ.
- **Primary → Read Replica** (orange): **asynchronous** replication, same-region or **cross-region**.
- **Backups & snapshots → S3**: stored in an AWS-managed bucket you never see in your account.
- **Dedicated [EBS](ebs.md) storage per instance** — primary *and* standby each have their own.

### Subnet Groups

A **DB subnet group** is a named *list of subnets* RDS is allowed to place instances in.

- You **must pick a DB subnet group** at launch, whether or not the deployment is HA.
- For a **Multi-AZ / HA** launch, RDS places the primary in one subnet and the standby in another — chosen **at random** unless you express a preference, but always in **different AZs**.
- One subnet group **can** serve multiple instances, but they are then constrained to the same subnets. To spread databases across different subnet sets, use **multiple** subnet groups.
- Best practice (editorial): **one DB subnet group per RDS deployment** for maximum placement flexibility.

### Storage, Replication & Backups

| Aspect | RDS behaviour |
| --- | --- |
| **Databases per instance** | Multiple databases can live on one RDS instance |
| **Storage** | Each instance has its **own dedicated [EBS](ebs.md) storage**; in a Multi-AZ pair, primary and standby each have separate storage. ⚠️ Differs from Aurora |
| **Multi-AZ replication** | **Synchronous** primary → standby — data replicated as soon as the primary receives it; standby is an identical copy |
| **Read replicas** | **Asynchronous** replication; same-region or **cross-region**; used to **scale read load** or add **DR** resilience in another region |
| **Backups** | Go to AWS-managed **S3**, replicated across AZs in the region (survive an AZ failure). In **Multi-AZ mode, backups run from the standby** — no performance hit on the primary |

> [!tip] Exam Tip
> Two replication words to keep straight: **standby = synchronous** (HA, identical copy, automatic failover target — *not* readable by your app), **read replica = asynchronous** (read scaling and/or DR, can be cross-region, *is* readable). Mixing these up is a classic trap.

## Cost Model

RDS is **resource-allocation billed** (like [EC2](ec2.md)), *not* usage-billed — you pay for the instance whether or not it's busy.

| Cost component | Notes |
| --- | --- |
| **Instance size & type** | Bigger / more capable = more; hourly rate, **billed per second** |
| **Multi-AZ** | More instances = additional cost (amount depends on the Multi-AZ architecture) |
| **Storage** | Per-GB-month; provisioned-IOPS types cost more, aligned to [EBS](ebs.md) pricing |
| **Data transfer** | Per-GB for data in/out to the internet and other AWS regions |
| **Backups & snapshots** | Snapshot storage **equal to your DB storage is free** (2 TB DB → 2 TB free snapshots); beyond that, per-GB-month (1 TB for 1 month = 500 GB for 2 months) |
| **Commercial engine licensing** | Extra for Oracle / SQL Server |

## Exam Quick-Reference

| Concept | Trigger / key point |
| --- | --- |
| DB on EC2 is the *right* answer | Unsupported engine/version, specific OS+DB combo, genuine OS/root access, AWS-unsupported replication architecture |
| RDS = "DB **server** as a service" | Not DBaaS — one instance hosts **multiple databases** |
| No SSH / no OS access | Exception: **RDS Custom** |
| Aurora ≠ RDS | Separate product; different (shared-cluster) storage architecture |
| Multi-AZ replication | **Synchronous**, same region, different AZ, standby identical (and not readable) |
| Read replicas | **Asynchronous**, same- or **cross-region**, read scaling + DR |
| Storage | Dedicated **EBS per instance** (unlike Aurora) |
| Backups | Go to **S3** (AWS-managed); Multi-AZ backups run from **standby** |
| Subnet group | List of subnets RDS may use; required at launch; Multi-AZ → primary + standby in different AZs (random unless specified) |
| Billing | Resource-allocation (like EC2), hourly rate **billed per second**, not usage-based |

> [!warning] Gotchas
>
> - **"Database as a service" is a deliberately wrong characterisation of RDS** — you receive a *server*, hence multiple DBs per instance.
> - **RDS instances in public subnets *can* get public IPs** — possible, but a security anti-pattern; know it exists as an option.
> - **Same-AZ private-IP traffic is free; cross-AZ traffic is charged** — relevant to both EC2-DB and RDS Multi-AZ designs.
> - **EC2-hosted DB = single AZ** — instance plus EBS volume; an AZ failure can take DB access down.
> - **Don't confuse RDS storage with Aurora storage** — RDS = dedicated EBS per instance; Aurora differs.

## References

- Cantrill SAA-C03 study notes — *RDS Architecture (and Databases on EC2)*
- Related: [EC2](ec2.md) · [EBS](ebs.md) · [VPC](vpc.md) · [Database Transaction Models (ACID/BASE/CAP)](../concepts/db-transaction-model.md) · [Resource Scope](../concepts/resource-scope.md)
