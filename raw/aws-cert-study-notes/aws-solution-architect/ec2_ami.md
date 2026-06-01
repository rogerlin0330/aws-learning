# Amazon Machine Images (AMIs)

## What an AMI is

An **AMI** is an _image_ of EC2 — a template of an instance configuration that can be used to launch one or many instances from that same configuration. Every EC2 instance you launch is created from an AMI, even when you pick something like "Amazon Linux 2" in the console; behind that choice is an Amazon-provided AMI.

The interesting case (and the focus of this note) is creating **your own** AMIs from already-configured instances.

## Where AMIs come from

AMIs you launch from can come from several sources:

- **AWS-provided** — official images (e.g. Amazon Linux 2).
- **Community-provided** — vendors publishing their own OS distributions (e.g. Red Hat, CentOS, Ubuntu).
- **Marketplace-provided** — may bundle commercial software. Billing in this case is the normal instance hourly rate **plus** an extra charge covering the software license.

## Core properties

**Regional.** An AMI exists in one region and can only be used in that region. The _same_ OS (e.g. Amazon Linux 2) has a **different AMI ID in each region** — there is a distinct AMI in `us-east-1`, another in `us-west-1`, another in Sydney, and so on. IDs use the format `ami-<random alphanumeric>`.

**Permissions.** An AMI controls who can use it. There are three permission models:

1. **Private** — only your own account (the default).
2. **Public** — anyone can use it.
3. **Explicit** — you grant specific named AWS accounts access (e.g. other accounts in your org, or partner accounts).

## The AMI lifecycle

Think of the lifecycle as four phases — and note the **two separate launch phases** are intentional.

![[ec2-ami-lifecycle.png]]

**1. Launch** — Use an AMI to launch an EC2 instance. _Most people only ever experience this phase_, and that's fine — custom AMIs are one of several ways to provision in AWS, not a requirement.

**2. Configure** — Take the launched instance (the instance plus its attached EBS volumes) and customize it into a state that's ideal for your needs. That might mean an OS with specific applications/services installed, a particular set/size of volumes, or a fully installed and configured application suite ready to use.

**3. Create Image** — Capture that customized configuration as a new AMI (see "What the AMI actually contains" below).

**4. Launch (again)** — Use your new AMI to stamp out brand-new instances that have the _same data and the same volume configuration_ as the original.

So the flow most people know (AMI → instance) can be run in **reverse** (instance → AMI) to capture an existing configuration as a reusable template.

## What the AMI actually contains

An AMI is best understood as a **logical container** — it does **not** itself hold any real volume data. It holds:

- An **AMI ID**.
- **Permissions** (who can use it — see above).
- A **block device mapping** that references EBS snapshots.

When you create an AMI from an instance, the key step is that **an EBS snapshot is taken of each attached EBS volume first**. (Recall that EBS snapshots are incremental, but the _first_ snapshot is a full copy of the volume's used data.)

The **block device mapping** is essentially a small table. For each snapshot, it records:

- the **snapshot ID**, and
- the **device ID** the original volume had on the source instance.

For example, a two-volume instance produces a mapping with two entries:

|Snapshot|Original device ID|Role|
|---|---|---|
|snapshot of boot volume|`/dev/xvda`|boot|
|snapshot of data volume|`/dev/xvdf`|data|

### How that mapping is used on launch

When the AMI is later used to launch a new instance, the referenced **snapshots are used to create new EBS volumes in the target Availability Zone**, and those volumes are attached using the **same device IDs** recorded in the block device mapping. This is what reproduces the original volume layout on the new instance.

### Regional storage detail

The AMI object is stored in the region. The underlying snapshots live on S3, which is already regional. Because of this, an AMI created from an instance in one AZ can deploy new instances **back into that same AZ or into any other AZ in the region**.

## Key operational concepts & gotchas

**AMI baking.** The practice of taking an instance, installing all software and applying all configuration, and then "baking" that finished state into an AMI. Example: instead of manually installing/configuring WordPress on every instance, do it once, bake an AMI, then launch tens or hundreds of pre-configured instances from it.

**AMIs cannot be edited.** There is no "edit AMI" operation. To change an existing AMI's configuration you must: launch an instance _from_ it → make your changes → create a **brand-new** AMI. (Exam trap.)

**AMIs can be copied between regions.** Since AMIs are regional, copying is how you get the same image into another region. Copied AMIs retain the default private permissions unless you change them.

## Billing

You are billed for an AMI based on the **storage capacity used by the EBS snapshots it references** — not for the AMI container itself.

Because snapshots only store the _used_ data (not the full allocated volume size), the cost can be much smaller than the volume allocation suggests. A large volume that is mostly empty produces a small, cheaper snapshot.

---

### Quick exam recall

- AMI = regional; one region only, but deployable to **all AZs** in that region.
- Same OS → **different AMI ID per region**.
- Permissions: **private (default) / public / explicit accounts**.
- AMI is a **container**: ID + permissions + block device mapping → snapshots. No volume data lives in the AMI itself.
- **Cannot edit** an AMI — relaunch, reconfigure, recreate.
- **Can copy** AMIs between regions.
- Billed for the **snapshot storage** the AMI references (used data only).
- **AMI baking** = installing/configuring an instance, then capturing it as an AMI.