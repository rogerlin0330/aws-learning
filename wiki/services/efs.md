---
title: EFS
type: service
tags: [efs, storage, file-storage, nfs, linux, posix, mount-target, hybrid]
certs: [aws-saa]
updated: 2026-07-06
---

# EFS (Elastic File System)

Amazon EFS provides **network-based file systems** that can be mounted by **many Linux [EC2](ec2.md) instances simultaneously, across multiple AZs**. It is AWS's implementation of the **NFS (Network File System)** standard — the course teaches **NFS v4**; current AWS docs list support for both **NFSv4.0 and NFSv4.1** ([EFS User Guide — NFS support](https://docs.aws.amazon.com/efs/latest/ug/how-it-works.html)). In the three-category model of [Storage Fundamentals](../concepts/storage.md#the-three-storage-categories), EFS is AWS's **file storage** offering for Linux, alongside FSx for Windows and other file-system flavors.

The core architectural value: like [EBS](ebs.md), the storage exists **separately from the instances that use it**, so data survives instance termination. Unlike EBS, it is a **shared file system** — a WordPress fleet, for example, can keep post media (images, video, audio) on EFS so nothing is lost as instances are added or removed. This pushes the architecture toward **stateless** EC2 instances, which is the foundation for [horizontal scaling](../patterns/scaling.md) and self-healing designs.

> [!tip] Exam Tip
> **EBS versus EFS in one line:** [EBS](ebs.md) is **block** storage, typically attached to **one** instance in **one** AZ; EFS is **file** storage that **many instances across many AZs** mount at once. Linux mounts EFS as though the storage were connected directly to the instance. A question describing "shared storage across multiple instances" is a classic EFS trigger.

## Architecture

![EFS architecture: an EFS file system inside a VPC exposes one mount target per AZ (each with an IP from its subnet's range); EC2 instances in each AZ connect through their local mount target, and on-premises Linux servers reach the same mount targets over VPN or Direct Connect](../assets/efs-architecture.png)

- The **file system** is the base entity. It is created **inside a [VPC](vpc.md)** and uses **POSIX permissions** — the Linux interoperability standard every distro understands.
- **Mount targets** make the file system available inside the VPC. They run **from subnets**, take an **IP address from the subnet's range**, and are **what instances connect through** — instances never touch the file system directly.
- **High availability:** put a mount target in **every AZ the VPC uses** — the same rule as NAT gateways (see [VPC](vpc.md)). A single mount target in one AZ is not highly available; per [Resource Scope](../concepts/resource-scope.md), the file system is a regional entity but the mount targets that expose it are AZ-scoped.
- **Private by default:** EFS is isolated to its VPC. It **can** be reached from outside via hybrid [networking](../concepts/networking.md) you configure — **VPC peering, VPN, or Direct Connect**. On-premises Linux servers connect through these to the same mount targets.

> [!warning] Gotcha
> Access is always **via mount targets**, never directly to the file system. And "private by default" does not mean unreachable from on-premises — hybrid access over VPN, Direct Connect, or VPC peering works **if configured**.

## Linux Only

EFS is officially supported on **Linux instances only**. If a question mentions **Windows** shared file storage, EFS is the wrong answer — the intended answer is **FSx for Windows File Server**.

## Performance Modes

The course teaches two:

| Mode | Best for | Trade-off |
| --- | --- | --- |
| **General Purpose** *(default)* | Latency-sensitive workloads: web servers, CMS, home directories, general file serving | Lowest latency |
| **Max I/O** | Highly parallel workloads: big data, media processing, scientific analysis | Higher aggregate throughput and operations, but **increased latency** |

> [!warning] Gotcha — course vs current AWS
> AWS now classifies **Max I/O as a previous-generation performance type** and recommends **General Purpose for all file systems** (General Purpose was improved to support hundreds of thousands of IOPS). Max I/O is **not supported** for One Zone file systems or file systems using Elastic throughput ([AWS EFS performance docs](https://docs.aws.amazon.com/efs/latest/ug/performance.html)). The two-mode framing may still appear on the exam, so know both — but the modern default guidance is "just use General Purpose."

## Throughput Modes

The course teaches two — **Bursting** and **Provisioned**:

| Mode | Behavior | EBS analogy (course) |
| --- | --- | --- |
| **Bursting** *(course default)* | Throughput **scales with file system size** (more data stored means more throughput); has a burst pool | Like [GP2](ebs.md#gp2--general-purpose-ssd-credit-bucket-model) |
| **Provisioned** | Specify throughput **independently of size** | Like [IO1](ebs.md#provisioned-iops-ssd--io1-io2-io2-block-express) |

> [!warning] Gotcha — there are now THREE throughput modes
> AWS added **Elastic throughput**, which auto-scales throughput to demand (pay per data read and written, no burst credits) — and **Elastic is now the default** when creating a file system in the console, not Bursting as the course states ([AWS EFS performance docs](https://docs.aws.amazon.com/efs/latest/ug/performance.html)). Bursting still exists and still scales with stored size. The course default (Bursting) may still be tested; newer questions may reference Elastic. Watch the phrasing.

## Storage Classes and Lifecycle

The course teaches two classes:

| Class | Use |
| --- | --- |
| **Standard** *(default)* | **Frequently** accessed files — day-to-day data |
| **Infrequent Access (IA)** | Lower-cost class for data **not accessed often** |

- Conceptually these mirror the [S3 storage class](s3.md#storage-classes) trade-offs: pay less per GB for data you rarely touch.
- **Lifecycle policies** can move data between classes automatically, just like [S3 lifecycle configuration](s3.md#lifecycle-configuration).

> [!warning] Gotcha — course vs current AWS
> EFS now offers more than two classes: the **Regional** classes above (**Standard** / **Standard-IA**, replicated across multiple AZs) plus **One Zone** classes (**One Zone** / **One Zone-IA** — single-AZ, cheaper, lower resilience), and an **Archive** class for rarely-accessed data ([EFS storage classes docs](https://docs.aws.amazon.com/efs/latest/ug/storage-classes.html)). The course's two-class model still covers the exam's core comparison.

## Exam Quick Reference

| Attribute | Detail | Trigger keyword |
| --- | --- | --- |
| Protocol | NFS v4 (docs: v4.0 and v4.1) | "NFS", "shared file system" |
| OS support | **Linux only** | "Windows" disqualifies EFS; pick FSx |
| Storage type | **File** storage (not block) | "shared across instances" |
| Access method | **Mount targets** (in subnets, subnet IPs) | "mount target" |
| HA rule | Mount target in **every AZ** used | "highly available", "multi-AZ" |
| Scope | **Private**, VPC-isolated by default | "private service" |
| Hybrid access | VPN / Direct Connect / VPC peering (if configured) | "on-premises access to shared files" |
| Permissions | **POSIX** | "POSIX", "Linux permissions" |
| Performance modes | General Purpose *(default)* / Max I/O *(previous-gen)* | "latency-sensitive" means GP; "highly parallel / big data" means Max I/O |
| Throughput modes | Bursting *(course default)* / Provisioned / **Elastic *(current default)*** | "scales with size" means Bursting; "constant, size-independent" means Provisioned |
| Storage classes | Standard *(default)* / IA (plus One Zone and Archive today) | "infrequently accessed" means IA |
| Cost optimization | **Lifecycle policies** move data between classes | "move to cheaper tier automatically" |

## References

- Source: `raw/aws-cert-study-notes/aws-solution-architect/efs.md`
- [AWS EFS performance docs](https://docs.aws.amazon.com/efs/latest/ug/performance.html) — performance and throughput mode current guidance
- [AWS EFS storage classes docs](https://docs.aws.amazon.com/efs/latest/ug/storage-classes.html) — current storage class lineup
- [Storage Fundamentals](../concepts/storage.md) — block / file / object framing
- [EBS](ebs.md) — the block-storage counterpart
