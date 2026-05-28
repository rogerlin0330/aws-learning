---
title: Storage Fundamentals
type: concept
tags: [storage, block-storage, file-storage, object-storage, ebs, instance-store, s3, efs, fsx, iops, throughput, ephemeral, persistent, direct-attached, network-attached]
certs: [aws-saa]
updated: 2026-05-26
---

# Storage Fundamentals

A cross-cutting reference for storage terminology, the three storage categories, and performance mechanics.
AWS services map directly onto these general concepts: [EC2 instance store](../services/ec2.md#storage-options) = direct-attached ephemeral block; [EBS](../services/ec2.md#storage-options) = network-attached persistent block; [S3](../services/s3.md) = object; EFS/FSx = file.

## Delivery Mode

### Direct-Attached (Local) Storage

Storage physically connected to the host — sits right next to the CPU and memory.

- **Pros:** Very fast; no network hop.
- **Cons:** Fragile — the data is gone if the disk fails, the host fails, or the VM migrates to a different physical host.
- **AWS example:** [EC2 instance store](../services/ec2.md#storage-options) — NVMe/SSD volumes on the EC2 host, free with the instance, lost on stop or terminate.

### Network-Attached Storage

Storage created independently and accessed over a network fabric.

- **Pros:** Resilient — the volume lives outside the host, so a host failure doesn't destroy it.
- **Cons:** Network latency between the instance and the storage adds overhead; the network can become the bottleneck.
- **AWS example:** [EBS (Elastic Block Store)](../services/ec2.md#storage-options) — block volumes attached over AWS's internal network; survive instance stop/start.
- **On-premises protocols:** iSCSI, Fibre Channel.

### Ephemeral vs Persistent

| Type | Lifetime | AWS example |
| --- | --- | --- |
| **Ephemeral** | Tied to the device/instance — lost when the host or instance goes away | Instance store (local to EC2 host) |
| **Persistent** | Independent of any compute; outlives the instance it's attached to | EBS network volumes |

> [!tip] Exam Tip
> Distinguish ephemeral from persistent before picking a storage type. Instance store is fast and free but ephemeral. EBS costs money even when the instance is stopped — but survives stop/start and host failure.

## The Three Storage Categories

### Block Storage

- Exposed as a **volume of addressable blocks** with no inherent structure.
- The OS lays a **file system** (NTFS, ext4, …) on top and mounts it (C: drive, `/` root, etc.).
- **Can be mounted** and **can be booted from** — the OS gets low-level block access.
- Forms: physical server disk, SAN, NVMe, network-attached block volumes.
- **AWS services:** [EBS](../services/ebs.md) (network-attached, persistent), [EC2 instance store](../services/ec2.md#storage-options) (direct-attached, ephemeral). See [EBS](../services/ebs.md) for EBS volume types (GP2, GP3, Provisioned IOPS, HDD).

### File Storage

- Exposed as a **ready-made file system** with folders and files — no formatting required.
- Access by traversing the directory tree (NFS, SMB/CIFS paths).
- Typically delivered over a network from a dedicated file server.
- **Can be mounted** but **cannot be booted from** — the OS doesn't get low-level block access.
- **AWS services:** EFS (Linux NFS), FSx (Windows / Lustre / NetApp / OpenZFS).

### Object Storage

- A **flat collection of objects** — no folder hierarchy (prefixes simulate folders but are just part of the key).
- Each object = **key** (identifier) + **value** (bytes) + optional **metadata**.
- Accessed via HTTP API (GET / PUT / DELETE on a URL); not mountable.
- **Cannot be mounted** and **cannot be booted from**.
- **Strength:** massively scalable; supports high concurrency of whole-object reads and writes.
- **AWS service:** [S3](../services/s3.md).

### Quick Decision Guide

| Need | Best category | AWS service |
| --- | --- | --- |
| Boot volume for an OS | Block | EBS |
| High-performance disk inside an OS | Block | EBS / instance store |
| Shared file system across multiple servers | File | EFS (Linux), FSx (Windows/Lustre) |
| Whole-object HTTP access, data lake, web-scale | Object | S3 |

> [!tip] Exam Tip
> The **access verb** picks the category. *Mount / drive letter / volume / format* → Block. *Shared file share between EC2 instances* → File (EFS/FSx). *Whole-object API / data lake / static assets / service output* → Object (S3). When S3, EBS, and EFS/FSx appear as options, find the verb.

> [!warning] Gotcha
> Object storage (S3) **cannot be mounted as a drive**. Tools like Mountpoint for S3 exist but they are not a transparent POSIX replacement and carry significant caveats — the exam treats "mount" as a disqualifier for S3.

## Storage Performance

Three metrics define storage performance — they are interconnected, not independent.

### The Three Metrics

| Metric | What it measures | Unit |
| --- | --- | --- |
| **Block size (IO size)** | Size of each read/write chunk | KB or MB |
| **IOPS** | Input/Output Operations per second | count/s |
| **Throughput** | Total data transferred per second | MB/s |

### Core Relationship

![[wiki/assets/storage-performance.png]]

```text
Throughput  = IO (Block) Size  ×  IOPS
```

Example: 16 KB block size × 100 IOPS = **1.6 MB/s throughput**.

Change one and the others move. But the relationship isn't frictionless:

- Increasing block size doesn't always scale IOPS linearly — many systems see IOPS drop as block size grows.
- Both IOPS and throughput may have hard **provisioned caps** (e.g. EBS volume type limits).
- If your application writes in small chunks (4 KB) but the storage is tuned for large blocks (1 MB), you under-use provisioned throughput while hitting IOPS limits.

### The Performance Chain

Performance is limited by the **slowest link** in the path from application to drive:

```text
Application → OS → Storage subsystem → Transport (network or SATA/NVMe bus) → Drive interface → Drive
```

Any one link can be the bottleneck:

- **Media type:** SSDs deliver dramatically higher IOPS than HDDs; NVMe over PCIe beats SATA.
- **Network latency:** for network-attached storage (EBS), the hop between the EC2 instance and the EBS service can cap effective IOPS — and the **instance type** sets a ceiling on storage network bandwidth independently of the EBS volume's provisioned IOPS.
- **Provisioned caps:** EBS volume types have both max IOPS and max throughput limits; the lower of the two always wins.

> [!tip] Exam Tip
> EBS-heavy workloads have **two** independent ceilings: the volume's provisioned IOPS/throughput, and the instance's storage network bandwidth. Both must be sized for the workload — either can be the bottleneck.

## Key Takeaways

- **Direct-attached** = fast but fragile (ephemeral); **network-attached** = resilient but adds a network hop.
- **Ephemeral** dies with the instance; **persistent** survives independently.
- **Block** = bootable raw volumes; **File** = shared mounted file systems; **Object** = flat, web-scale, API-accessed.
- Performance is a triangle: `Throughput = Block Size × IOPS`, capped by the weakest link in the chain.
- For AWS: instance store (ephemeral block) → EBS (persistent block) → EFS/FSx (file) → S3 (object). Each step trades raw speed and low-level control for resilience, shareability, and scale.

## References

- Source: `raw/aws-cert-study-notes/aws-solution-architect/general_storage_concepts.md`
- [EC2 Storage Options](../services/ec2.md#storage-options)
- [S3 — Object vs File vs Block](../services/s3.md#object-vs-file-vs-block-storage)
