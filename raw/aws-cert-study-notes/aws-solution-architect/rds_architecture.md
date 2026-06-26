# RDS Architecture (and Databases on EC2)

## Overview

This note covers two related lessons: **why running databases directly on EC2 is generally bad practice** (and the rare cases where it's justified), and the **architecture of the Relational Database Service (RDS)** — AWS's managed database server product.

---

## Part 1 — Databases on EC2

### The core stance

Running a database directly on EC2 is **always a bad idea** — the only question is whether the benefits to you or your business outweigh that fact. It can always be done, as long as you're willing to accept the negatives.

### What "databases on EC2" looks like

People typically picture one of two architectures:

|Architecture|Description|
|---|---|
|**Single instance**|One (larger) EC2 instance running the DB platform, the application, and often a web server (e.g. Apache). Everything in one place, one AZ.|
|**Split architecture**|Two (smaller) instances — application/web server separated from the database. Can be in the same AZ or split across AZs (AZA + AZB).|

> 🎯 The Animals for Life WordPress stack used so far in the course is the **single-instance** model — all tiers on one instance in one AZ.

⚠️ **Splitting introduces a dependency**: there must be reliable communication between the app instance and the DB instance, or the app won't work.

💰 **Cross-AZ data cost**: traffic between AZs in the same region incurs a small charge. In contrast, communication between instances using **private IPs in the same AZ is free**.

### Reasons you _might_ run a DB on EC2 (with the level of scrutiny each deserves)

|Reason|Verdict|
|---|---|
|Need **OS-level / root access** to the DB|⚠️ Question hard. Rarely truly needed. Often an _application vendor_ demanding it, not the business. Many vendors now explicitly support AWS managed DBs.|
|DB tuning requiring **root-level access**|⚠️ Question — AWS lets you tune many historically root-only parameters _without_ root via managed products.|
|Need a **DB engine or version AWS doesn't provide**|✅ Often genuinely justified (niche/emerging databases).|
|Need a **specific OS + DB version combo** AWS doesn't offer|✅ Valid.|
|Need a **specific architecture / replication** AWS doesn't provide|✅ Valid.|
|**Decision-makers just want it** on EC2|Possibly unreasonable, but you may not have a choice.|

> 🎯 **Exam trap**: There are tricky exam questions where the _correct_ answer is "run the database on EC2." Keep these justification styles fresh — especially **unsupported engine/version** and **OS-level access requirements**.

### Why you really _shouldn't_ (the negatives)

- **Admin overhead** — you manage both the EC2 instance _and_ the DB host (patching, version compatibility, fault-finding out of core hours = time, stress, cost).
- **Backups & DR** — adds significant complexity; managed products automate much of this.
- **Single AZ** — EC2 instance + its EBS volume live in **one AZ**. If that AZ fails, DB access can fail. You must manage EBS snapshots / DB backups to S3 yourself.
- **Missing features** — managed DB products have advanced features you forgo.
- **No serverless / scaling** — EC2 is **on or off**; can't easily scale down or handle bursty load. Sets a base minimum cost (the hourly instance rate).
- **Replication** — you handle setup, monitoring, and effectiveness checks yourself.
- **Performance** — AWS invests heavily in DB performance optimizations you can't replicate off-the-shelf.

### Where the course goes next

The next lesson evolves the monolithic single-instance WordPress stack into **two instances**: one running Apache + WordPress (app server), one running **MariaDB** (DB server). This split is a stepping stone — once the DB is separated, it can later be migrated into a managed AWS database product to gain those features/performance and allow independent scaling.

---

## Part 2 — RDS Architecture

### What RDS actually is

> 🎯 **Not "database as a service."** With DBaaS you pay and get _a database_. With RDS you pay for and receive a **database server (instance)** — so it's more accurately a **"database server as a service."** This matters because **one RDS instance can host multiple databases**.

RDS provides a **managed** database server: AWS handles the hardware, OS, installation, and much of the DB engine maintenance.

- ⚠️ **No SSH access and no OS access.**
    - _Asterisk_: **RDS Custom** is a variant that grants more low-level access (covered separately).
- **Engines**: MySQL, MariaDB, PostgreSQL, plus commercial Oracle and Microsoft SQL Server. (Commercial engines = licensing implications.)

> ⚠️ **Disassociate Amazon Aurora from RDS.** Aurora is a _custom AWS-built engine/product_ (compatible with some of the above engines but designed entirely by AWS). Many RDS features differ for Aurora — treat them as separate. Notably, **Aurora handles storage differently** (see EBS note below).

### Diagram walkthrough

![[rds-architecture.png]]

**How to read this diagram** — work from the outside in: the two outer panels are the external networks (**On-Premises** on the left, **Internet** on the right). The large central panel is the **us-east-1 region** containing **VPC1**, split into three vertical AZ columns (**AZA / AZB / AZC**). A second region (**ap-southeast-2**) sits at the bottom to show cross-region elements.

- **Subnet Group** (top-right callout, → flask = "your additions" marker in the original): defines _what subnets RDS can use_. RDS runs **inside a VPC** (not a public service like S3/DynamoDB) and needs subnets in specific AZs. You create a DB subnet group as a _list of subnets_; when launching an instance you pick which group to use.
- **Top row of instances** (here, public subnets) → can optionally be given **public addressing** for internet access (⚠️ heavily frowned upon, but possible).
- **Bottom row of instances** (private subnets) → reachable from inside the VPC or connected networks (**VPN / Direct Connect**), shown by the **On-Premises** arrow on the left.
- **Primary → Standby (green arrow)**: **synchronous replication** within the _same region, different AZ_.
- **Primary → Read Replica (orange arrow, into ap-southeast-2)**: **asynchronous replication**, can be same-region or **cross-region**.
- **Backups & Snapshots → S3 (top-left)**: backups go to an AWS-managed S3 bucket (you don't see it in your account).
- **Dedicated storage (EBS) per instance** (bottom callout): each instance — primary _and_ standby — has its **own** EBS-backed storage.

> 📚 Region/AZ names (us-east-1, ap-southeast-2) and the public/private subnet split are **illustrative** — they demonstrate the patterns, not fixed values.

### Subnet groups — key behaviours

- When launching an instance (HA or not), you **must pick a DB subnet group**.
- For a **multi-AZ / HA** launch, RDS picks one subnet for the **primary** and another for the **standby**, **at random** unless you specify a preference — but always in **different AZs**.
- One DB subnet group **can** serve multiple instances, but they're then limited to the same subnets. To split DBs across different subnet sets, use **multiple** DB subnet groups.
- 🎯 _Cantrill best-practice (editorial — flagged for review)_: **one DB subnet group per RDS deployment** for maximum flexibility.

### Storage, replication & backups

|Aspect|RDS behaviour|
|---|---|
|**Databases per instance**|Multiple databases can live on one RDS instance.|
|**Storage**|Every instance has its **own dedicated EBS storage**. In a multi-AZ pair, primary _and_ standby each have separate storage. ⚠️ This differs from Aurora — remember it.|
|**Multi-AZ replication**|**Synchronous** primary → standby (data replicated as soon as the primary receives it). Standby holds an identical copy.|
|**Read replicas**|**Asynchronous** replication. Same-region or **cross-region**. Used to **scale read load** or add **DR resilience** in another region.|
|**Backups**|Go to **S3** (AWS-managed bucket) → replicated across multiple AZs in the region, surviving an AZ failure. In **multi-AZ mode, backups run from the standby** → no performance impact on the primary.|

### Cost architecture

RDS is **resource-allocation billed** (like EC2), _not_ usage-billed. Components:

|Cost component|Notes|
|---|---|
|**Instance size & type**|Bigger / more feature-rich = more cost. Hourly rate, **billed per second**.|
|**Multi-AZ**|More instances = additional cost (amount depends on the multi-AZ architecture).|
|**Storage**|Per-GB-month fee. Certain types (e.g. **provisioned IOPS**) cost more — aligned to EBS pricing.|
|**Data transfer**|Per-GB charge for data in/out of the instance to/from the internet and other AWS regions.|
|**Backups & snapshots**|You get snapshot storage equal to your DB storage **for free** (e.g. 2 TB DB → 2 TB free snapshots). Beyond that = per-GB-month. _1 TB for 1 month = 500 GB for 2 months._|
|**Commercial DB engine licensing**|Extra cost for commercial engines (Oracle, SQL Server).|

---

## Exam Quick-Reference

|Concept|Trigger / Key Point|
|---|---|
|🎯 DB on EC2 is the _right_ answer|Unsupported engine/version, specific OS+DB combo, OS-level/root access genuinely required, AWS-unsupported replication architecture|
|🎯 RDS = "DB **server** as a service"|Not DBaaS; one instance hosts **multiple databases**|
|⚠️ No SSH / no OS access|Exception: **RDS Custom**|
|⚠️ Aurora ≠ RDS|Separate product; different storage architecture|
|🎯 Multi-AZ replication|**Synchronous**, same region, different AZ, standby is identical|
|🎯 Read replicas|**Asynchronous**, same- or **cross-region**, for read scaling + DR|
|🎯 Storage|Dedicated **EBS per instance** (unlike Aurora)|
|🎯 Backups|Go to **S3** (AWS-managed); multi-AZ backups run from **standby** (no perf hit)|
|🎯 Subnet group|List of subnets RDS may use; required at launch; multi-AZ → primary + standby in different AZs (random unless specified)|
|💰 Billing model|Resource allocation (like EC2), hourly rate **billed per second**, not usage-based|

## Traps & Gotchas

- ⚠️ **"Database as a service" is a deliberately wrong characterisation of RDS** — it's a _server_ you receive, hence multiple DBs per instance.
- ⚠️ **RDS instances in public subnets _can_ be given public IPs** — possible but a security anti-pattern; know it exists as an option.
- ⚠️ **Same-AZ private-IP traffic is free; cross-AZ traffic is charged** (relevant to both EC2-DB and RDS multi-AZ designs).
- ⚠️ **EC2-hosted DB = single AZ** (instance + EBS volume) — an AZ failure can take down DB access.
- ⚠️ **Don't confuse RDS storage with Aurora storage** — RDS = dedicated EBS per instance; Aurora differs.