---

tags:

- aws
- saa
- ec2
- auto-scaling
- launch-templates
- launch-configurations

---

# Launch Configuration and Templates

## Overview

**Launch Configurations (LC)** and **Launch Templates (LT)** are both _documents_ that let you define the configuration of EC2 instances **in advance**. Everything you would normally set at the point of launching an instance can instead be captured in one of these documents and reused.

The two do fundamentally the same job, but **launch templates are the newer, superset technology** — they include everything launch configurations offer _plus_ extra features, and AWS recommends using them.

What can be defined inside either document:

- The **AMI** to use
- **Instance type and size**
- **Storage** configuration (the disks the instance uses)
- The **key pair** used to connect to the instance
- **Networking** configuration and **security groups**
- **User data** provided to the instance (bootstrapping)
- The **IAM role** attached to the instance (permissions)

> [!warning] ⚠️ Both are **NOT editable** — this is the single most important behavioural fact Once defined, the configuration is **locked**. To change anything, you create a _new_ document and point whatever consumes it (e.g. an Auto Scaling Group) at the new one.
> 
> - **Launch Configurations**: no editing, **no versioning** — one immutable document.
> - **Launch Templates**: still not editable _in place_, but they **support versions**, so related configurations can live under one template as version 1, 2, 3, etc.

---

## Diagram Walkthrough

![[ec2-launch-configuration-and-template-architecture.png]]

**How to read this diagram:** it's a two-lane comparison. The **top lane** is Launch Configuration; the **bottom lane** is Launch Template. Each lane starts with the document (left), and arrows show what that document is allowed to feed into (right). The callouts (the red curved arrow at top, the "Bob" person at bottom) highlight the two defining differences between the lanes.

**Header/context:** Title reads _"LC and LT Architecture"_ — this is Cantrill's side-by-side of the two mechanisms and the resources each can drive.

**Top lane — Launch Configuration:**

- The **Launch Configuration** document sits on the left.
- The red **"Not Editable ... No Versioning"** callout with the curved arrow loops back onto the LC document — visually reinforcing that an LC is a single, frozen document with no version history.
- A single red arrow runs from the LC **only** to **Auto Scaling Groups**. This is the key limitation: **an LC has exactly one use — feeding an Auto Scaling Group.**

**Bottom lane — Launch Template:**

- The **Launch Template** document sits on the left.
- One blue arrow goes **up and across into Auto Scaling Groups** — an LT can do everything an LC can do.
- A second blue arrow goes **directly to EC2 Instances** — an LT can _also_ launch instances directly.
- The **"Bob" callout** at the bottom explains the second arrow: _"Launch templates can be used to save time when provisioning EC2 instances from the console UI/CLI."_ A human operator can pre-define instance config and reuse it for one-off manual launches.

**The footer takeaway:** LC feeds one target (ASG); LT feeds two (ASG **and** direct EC2 launches). That architectural difference — plus versioning — is why LT is the recommended, more flexible choice.

---

## Mechanics & Architecture

### Launch Configuration

- **One and only one use case:** providing the instance configuration that an **Auto Scaling Group** uses to launch EC2 instances.
- **Immutable, no versioning.** Need a change? Create a brand-new LC and update the ASG to use it.

### Launch Template

- **Superset of LC.** Provides all LC capabilities plus more.
- **Two use cases:**
    1. Provide EC2 configuration to an **Auto Scaling Group** (same as LC).
    2. **Launch EC2 instances directly** from the **console or CLI**, so an operator can define config once and reuse it for manual launches.
- **Supports versions** — a subset of parameters can be defined and reused to create other templates or template versions (e.g. a "default" template with common parameters, extended by other versions). _(Course framing; version behaviour per current AWS docs.)_

### Newer features exposed only by Launch Templates 📚

The transcript calls these out as things LTs let you control on newer instance types / features:

- **T2 / T3 unlimited** CPU options (burst credits)
- **Placement groups**
- **Capacity reservations**
- **Elastic Graphics**

> [!note] 📚 Supplement: Mixed Instances Policy A commonly-tested LT-only capability the transcript doesn't mention: launch templates support a **Mixed Instances Policy**, letting a single ASG combine **multiple instance types** and mix **Spot + On-Demand** purchasing. Launch configurations cannot do this. _(AWS EC2 Auto Scaling documentation.)_

---

## ⚠️ Course vs. Reality Gap — Launch Configurations are being retired

This is the biggest discrepancy between the course recording and current AWS. The lesson presents LC and LT as coexisting choices with AWS merely _recommending_ LT. **AWS has since formally deprecated launch configurations.** Per current AWS EC2 Auto Scaling documentation:

- **As of January 1, 2023** — new EC2 instance types are **no longer supported** in launch configurations (including any instance type added to a Region after that Region's initial launch).
- **Accounts created on or after June 1, 2023** — cannot create new launch configurations **using the console**.
- **Accounts created on or after October 1, 2024** — cannot create new launch configurations using **any method** (console, API, CLI, _or_ CloudFormation). The API/CLI returns an `UnsupportedOperationException`.
- **Existing** launch configurations and ASGs that use them **continue to function**, but AWS strongly recommends migrating to launch templates. A **console migration tool** ("Copy to launch template") exists to convert them.

> [!tip] 🎯 What this means for the exam The SAA exam still tests the **conceptual comparison** (LC = ASG-only, immutable, no versioning; LT = superset, versioned, direct-launch + ASG, mixed instances). But real-world guidance is unambiguous: **use launch templates.** If a question asks for the _recommended_ or _modern_ approach, or mentions **new instance types, mixed/Spot instances, or versioning**, the answer is **launch templates**.
> 
> _Sources: [AWS EC2 Auto Scaling — Launch configurations](https://docs.aws.amazon.com/autoscaling/ec2/userguide/launch-configurations.html); [AWS Compute Blog — LCs will no longer add new EC2 features](https://aws.amazon.com/blogs/compute/amazon-ec2-auto-scaling-will-no-longer-add-support-for-new-ec2-features-to-launch-configurations/)._

---

## Exam Traps & Trigger Keywords

> [!warning] ⚠️ Neither LC nor LT is editable "How do I update a launch configuration?" — you **can't**. You create a new one. An LT isn't edited in place either; you create a **new version**. Watch for answer options implying in-place edits — they're wrong.

> [!warning] ⚠️ Launch Configuration cannot launch instances directly Only **launch templates** can be used to launch EC2 instances directly from the console/CLI. An LC's _only_ consumer is an Auto Scaling Group.

> [!warning] ⚠️ Versioning is LT-only If a question mentions **versions** of a launch document, that's a **launch template**. LCs have no versioning.

|Exam trigger 🎯|Points to|
|---|---|
|"versions", "version 1 / latest / default version"|**Launch Template**|
|"launch instances directly from console/CLI"|**Launch Template**|
|"mixed instance types", "Spot + On-Demand in one ASG"|**Launch Template** (Mixed Instances Policy)|
|"T2/T3 unlimited", "placement groups", "capacity reservations", "elastic graphics"|**Launch Template**|
|"newest instance types" / "recommended" / "modern approach"|**Launch Template**|
|"only used by Auto Scaling Groups", "no versioning", "older mechanism"|**Launch Configuration**|
|"edit the existing launch config/template"|**Trap** — neither is editable; create new (LC) or new version (LT)|

---

## Quick-Reference Summary

|Attribute|Launch Configuration (LC)|Launch Template (LT)|
|---|---|---|
|**Purpose**|Pre-define EC2 config for reuse|Pre-define EC2 config for reuse|
|**Editable?**|❌ No|❌ No (create a new version)|
|**Versioning?**|❌ No|✅ Yes|
|**Used by Auto Scaling Groups?**|✅ Yes (its _only_ use)|✅ Yes|
|**Launch instances directly (console/CLI)?**|❌ No|✅ Yes|
|**Newer-instance features** (T2/T3 unlimited, placement groups, capacity reservations, elastic graphics)|❌ No|✅ Yes|
|**Mixed Instances Policy** (multi-type, Spot + On-Demand) 📚|❌ No|✅ Yes|
|**Feature relationship**|Subset|**Superset** of LC|
|**AWS recommendation**|Legacy / **deprecated** ⚠️|✅ Use this|
|**Can new accounts create it?**|❌ Not since Oct 1, 2024 (any method) ⚠️|✅ Yes|

**One-line takeaway:** _Launch templates are launch configurations plus versioning, direct-launch, and newer-instance features — and are now the only supported option going forward._

---

## Open Questions for Roger

- Nothing blocking. Note that the **next lesson is Auto Scaling Groups**, which consume these documents — the LC/LT concepts here (immutability, "create-new-to-change") feed directly into how ASG instance refreshes and updates work, so worth cross-linking once that note exists.

## Source

- Adrian Cantrill, _AWS SAA — Launch Configuration and Templates_ (learn.cantrill.io), transcript + course diagram.
- Deprecation/current-behaviour facts cross-referenced against AWS EC2 Auto Scaling documentation and the AWS Compute Blog (cited inline above), current as of the note's creation date.