# Elastic Block Store (EBS) — Service Architecture

> **Scope of this lesson:** A high-level introduction to what EBS is, how it relates to EC2 and Availability Zones, and the foundational architecture you must understand before going deeper into volume types, snapshots, and encryption.

---

## 1. What EBS Is

**EBS (Elastic Block Store)** is an AWS service that provides **block storage** — storage addressable using **block IDs**, in contrast to object storage (S3) or file storage (EFS).

- EBS takes raw physical disks and presents an **allocation** of them as a **volume**.
- Volumes are read from / written to using block numbers.
- Volumes can be **unencrypted** or **encrypted using KMS** (covered separately).

### How EC2 Sees an EBS Volume

When attached to an EC2 instance, an EBS volume appears as a **raw block device** — indistinguishable from a locally attached disk. The OS on the instance is then responsible for putting a **file system** on top of it:

|OS|Common File Systems|
|---|---|
|Linux|EXT3, EXT4, XFS, …|
|Windows|NTFS|

> 🔑 **Key idea:** EBS provides the _raw disk_. The instance/OS provides the _file system_.

---

## 2. The Availability Zone Boundary (The Single Most Important Concept)

> ⚠️ **EBS is an Availability Zone–scoped service.** This single fact drives almost every architectural decision and exam question about EBS.

- A volume lives in **exactly one AZ**.
- EBS in AZ-A is a **completely separate deployment** from EBS in AZ-B — and from EBS in any other region.
- Within its AZ, EBS is **resilient**: data is replicated across multiple physical devices, so a single disk failure does not destroy your volume.
- However, if the **entire AZ fails**, the volumes in that AZ are likely lost — just like the EC2 instances in that AZ.

### Attachment Rules

- A volume can normally be **attached to one EC2 instance at a time**.
- It can be **detached** from one instance and **reattached** to another — but **only within the same AZ**.
- **Multi-attach** is a feature of _some_ volume types that allows attachment to multiple instances simultaneously. This is intended for **clustered applications**, and the cluster software must coordinate writes to prevent corruption.

> 🚫 **No cross-AZ attachment.** An instance in AZ-B cannot attach to a volume in AZ-A. Period.

---

## 3. Lifecycle: EBS Volumes Are Persistent

EBS volumes have a lifecycle **independent of any single EC2 instance**.

|Event on the EC2 instance|Effect on the attached EBS volume|
|---|---|
|Instance **stops & starts**|Volume persists, stays attached|
|Instance **restarts**|Volume persists, stays attached|
|Instance **moves to another host** (e.g., after stop/start)|Volume follows the instance|
|Instance is **terminated**|Volume persists _unless_ "delete on termination" was set|
|Volume is **detached**|Volume persists, can be reattached to another instance in same AZ|

> 💡 **Mental model:** An EBS volume exists until _you_ explicitly delete it. It is not tied to any one instance's life.

---

## 4. Crossing the AZ Boundary: Snapshots to S3

Because EBS itself cannot cross AZs, AWS provides **snapshots** as the bridge.

- A **snapshot** is a backup of a volume stored inside an AWS-managed portion of **S3**.
- Snapshots are **regionally resilient** — replicated across **3+ AZs** in the region.
- A snapshot can be used to **create a new volume in a different AZ** → this is how you migrate or recover data across AZs.
- A snapshot can also be **copied to another region**, and then used to create a volume there → cross-region migration and DR.

Snapshots will be covered in depth in a dedicated lesson; here they're introduced only as the mechanism for crossing the AZ boundary.

---

## 5. Architecture Diagram — Putting It All Together

![[ebs-service-architecture.png]]

This diagram captures every concept from the lesson in one picture. Read it in the following logical order:

### 5.1 The Two Regions and Their AZs

- **us-east-1** contains two AZs shown: **AZA** and **AZB**.
- **ap-southeast-2** is a second region, used later to demonstrate cross-region snapshot copy.
- **S3** spans the bottom of each region — it is a regional service available from all AZs in that region.

### 5.2 EBS Is Deployed Per-AZ

Inside each AZ you can see two stacked layers:

1. An **EC2 HOST** layer (top) containing instances.
2. An **EBS** layer (bottom) containing volumes.

Crucially, **AZA has its own EBS deployment, and AZB has its own, separate EBS deployment.** Same architecture pattern as EC2 hosts — isolated per AZ.

### 5.3 Normal Attachment (Green Arrows)

Within **AZA**, green arrows connect instances to volumes:

- An instance is attached to a single volume.
- Another instance is attached to **two** volumes (multiple volumes per instance is allowed).
- A **dashed green arrow** represents a volume that has been **detached** from one instance and is available to be **reattached** to another instance — _within the same AZ_.

The same pattern appears in **AZB**: instances attach to volumes that live in AZB's EBS deployment.

### 5.4 The Forbidden Path (Red Stop Sign)

A pink/red arrow attempts to go from an instance in **AZB** across to a volume in **AZA**, and is blocked by a **red "no entry" symbol**. This visually enforces the rule: **no cross-AZ attachment is possible.** EBS storage traffic does not traverse AZs.

### 5.5 Replication Within an AZ

The caption _"EBS replicates within an AZ. Failure of an AZ means failure of a volume"_ summarizes EBS's resilience model:

- ✅ Protects against single-disk failure inside the AZ.
- ❌ Does **not** protect against AZ-wide failure.

### 5.6 Snapshots to S3 (Pink Arrow Down)

A pink arrow flows from a volume in AZA _down_ into the S3 bucket at the bottom of us-east-1. This represents creating a **snapshot** of the volume into S3.

Once in S3, the data benefits from S3's resilience model — the orange arrow labeled **"S3 Replication 3+ AZs in a region"** shows that the snapshot is now replicated across 3 or more AZs within the region. The volume's data is no longer confined to a single AZ.

### 5.7 Restoring a Snapshot into a Different AZ (Orange Arrow Up)

An orange arrow flows from the S3 snapshot _up_ into AZB's EBS, labeled **"Volume from Snapshot"**. This is how you effectively "move" a volume across AZs: snapshot in AZA → restore in AZB.

### 5.8 Cross-Region Resilience (Pink Arrow Across)

The long pink arrow labeled **"Snapshots copied across regions provide global resilience"** runs from the S3 bucket in us-east-1 to a **Snapshot Copy** bucket in ap-southeast-2.

From that copied snapshot, another orange "Volume from Snapshot" arrow restores into an EBS volume inside ap-southeast-2, which is then attached to an instance on an EC2 host there.

### 5.9 The Complete Resilience Hierarchy

Reading the diagram top-to-bottom, the resilience tiers EBS gives you are:

|Tier|Mechanism|Protects Against|
|---|---|---|
|1|EBS internal replication (within AZ)|Individual disk failure|
|2|Snapshot to S3|AZ-level failure → restore to another AZ|
|3|Snapshot copy across regions|Region-level failure → restore in another region|

---

## 6. Volume Variety (Preview)

EBS supports volumes built on different underlying physical storage:

- **SSD-based** (general purpose)
- **High-performance SSD** (provisioned IOPS)
- **Mechanical (HDD)–based**

Volumes also vary by **size** and **performance profile**. Specific volume types (gp2, gp3, io1, io2, st1, sc1, etc.) are covered in upcoming lessons.

---

## 7. Billing Model

EBS is billed using a **GB-per-month** metric.

- 1 GB for 1 month = 2 GB for half a month = 0.5 GB for 2 months — all the same charge.
- Certain volume types add **extra charges for enhanced performance** (e.g., provisioned IOPS), covered later.

> ⚠️ **Exam trap reminder (from EC2 lessons):** Storage charges accrue even when the EC2 instance is _stopped_, because EBS volumes are billed independently of instance state.

---

## 8. Key Takeaways

1. **EBS = block storage**, presented as volumes, attached over a storage network to EC2.
2. **EBS is AZ-scoped.** Volume and instance must be in the same AZ to attach.
3. **EBS is resilient within an AZ** but **fails with the AZ**.
4. **Volumes are persistent** — independent of any single instance's lifecycle.
5. **Snapshots → S3** are the mechanism for crossing AZ and region boundaries.
6. **Multi-attach** exists but requires cluster-aware software to avoid corruption.
7. **Billing is GB-per-month**, with extras for high-performance profiles.

---

## 9. Likely Exam-Style Insights

- _"An EC2 instance in AZ-A needs to access a volume currently in AZ-B."_ → **Not directly possible.** Snapshot the volume to S3, then create a new volume in AZ-A from that snapshot.
- _"What happens to an EBS volume when its attached instance is stopped?"_ → **It persists and remains attached**; storage charges continue.
- _"How can you achieve cross-region DR for EBS data?"_ → **Snapshot, then copy snapshot to the target region**, then create a volume from the copy.
- _"Can two EC2 instances share an EBS volume?"_ → **Only with the multi-attach feature**, only on supported volume types, only within the same AZ, and only with cluster-aware applications.
- _"Where does the file system (EXT4/NTFS) come from?"_ → **From the EC2 instance's OS, not from EBS.** EBS only provides the raw block device.