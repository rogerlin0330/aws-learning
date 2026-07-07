# Elastic File System (EFS)

## Overview

**EFS** provides **network-based file systems** that can be mounted within Linux EC2 instances and used by **multiple instances simultaneously**. It's AWS's implementation of the **NFS (Network File System)** standard — the course specifies **NFS v4**. 📚 _Current AWS docs list support for both NFSv4.0 and NFSv4.1._

The core value: storage exists **separately from EC2 instances** (like EBS does), so data survives instance termination. In the Animals4Life WordPress example, this lets post media (images, video, audio) live outside individual instances — so media isn't lost when instances are added or removed. This moves the architecture toward **stateless** EC2 instances, enabling better **scaling** and **self-healing**.

> 📚 **EBS vs EFS in one line:** EBS is **block** storage (typically attached to _one_ instance, in _one_ AZ); EFS is **file** storage that many instances across many AZs mount at once. Linux mounts EFS as though it's connected directly to the instance.

---

## Diagram Walkthrough

![[efs-architecture.png]]

Read the diagram in three passes:

- **Context bar (top-left):** The whole architecture lives inside a **VPC** (`10.16.0.0/16`, `us-east-1`). This reinforces that **EFS is a private service isolated to its VPC by default**.
- **Main objects (left → right):**
    - The **EFS Filesystem** (green folder, far left) is the base entity. It uses **POSIX permissions** — a Linux interoperability standard every distro understands.
    - Two red arrows fan out from the filesystem into **two Availability Zones (AZA, AZB)**, each landing on a **Mount Target** (`10.16.32.20` in AZA, `10.16.96.20` in AZB). Note the mount target IPs fall **inside each subnet's range**.
    - Green arrows show **EC2 Instances** connecting _to the mount targets_ — not directly to the filesystem. The mount target is the access point.
- **Footer / right side (hybrid access):** A **VPN** and **Direct Connect** link the VPC to **On-Premises Infrastructure**. The green line shows an on-prem Linux server reaching the _same mount targets_ through hybrid networking.

🎯 **Key visual takeaway:** Instances never touch the filesystem directly — **everything goes through mount targets**, and mount targets are per-AZ.

---

## Architecture & Mechanics

- **File systems** are the base entity; they're created inside a VPC and use **POSIX permissions**.
- **Mount targets** make the file system available inside the VPC. They:
    - Run **from subnets** inside the VPC.
    - Take an **IP address from the subnet's range**.
    - Are what **instances connect through** to reach the file system.
- **High availability:** Put a mount target in **every AZ the VPC uses** — same rule as NAT gateways. ⚠️ _A single mount target in one AZ is not highly available._
- **Private by default:** EFS is isolated to its VPC. But it **can** be reached from outside via **hybrid networking** you configure: **VPC peering, VPN, or Direct Connect**. On-prem Linux servers connect through these to the same mount targets.

---

## Exam-Critical Facts

### Linux only

🎯 **EFS is for Linux instances only** — officially supported on Linux only. If a question mentions **Windows** shared file storage, EFS is the **wrong answer** (think **FSx for Windows File Server** instead). 📚

### Performance Modes (2)

|Mode|Best for|Trade-off|
|---|---|---|
|**General Purpose** _(default)_|Latency-sensitive: web servers, CMS, home directories, general file serving|— (lowest latency)|
|**Max I/O**|Highly parallel workloads: big data, media processing, scientific analysis|Higher throughput/ops **but increased latency**|

> ⚠️ 📚 **Course vs current AWS reality:** AWS now classifies **Max I/O as a _previous-generation_ performance type** and recommends **General Purpose for _all_ file systems** (General Purpose was improved to support hundreds of thousands of IOPS). Max I/O is **not supported** for One Zone file systems or file systems using **Elastic throughput**. The course's two-mode framing may still appear on the exam, so know both — but the modern default guidance is "just use General Purpose." _Source: AWS EFS performance docs — [docs.aws.amazon.com/efs/latest/ug/performance.html](https://docs.aws.amazon.com/efs/latest/ug/performance.html)_

### Throughput Modes

The **course teaches two** — **Bursting** and **Provisioned**:

|Mode|Behavior|EBS analogy (course)|
|---|---|---|
|**Bursting** _(course default)_|Throughput **scales with file system size** (more data stored → more throughput); has a burst pool|like **GP2**|
|**Provisioned**|Specify throughput **independently of size** — more flexible|like **IO1**|

> ⚠️ 📚 **Major discrepancy — there are now THREE throughput modes.** AWS added **Elastic throughput**, which auto-scales throughput to demand (pay per data read/written, no burst credits). **Elastic is now the default** when you create a file system via the console — **not Bursting** as the course states. Bursting still exists and still scales with stored size. Both framings can appear; the course default (**Bursting**) may still be tested, but real-world/newer questions may reference **Elastic**. _Source: AWS re:Post & EFS docs (confirmed 2025–2026)._

### Storage Classes

|Class|Use|
|---|---|
|**Standard** _(default)_|**Frequently** accessed files — day-to-day data|
|**Infrequent Access (IA)**|Lower-cost class for data **not accessed often**|

- Conceptually mirror **S3** object storage class trade-offs.
- **Lifecycle policies** can move data between classes automatically (just like S3).

> 📚 **Current AWS reality:** EFS now offers **four** storage classes — the two **Regional** classes above (**Standard** / **Standard-IA**, replicated across multiple AZs) **plus two One Zone** classes (**One Zone** / **One Zone-IA**, data in a single AZ, cheaper but lower resilience). There's also an **Archive** class for rarely-accessed data. The course's two-class model still covers the exam's core comparison.

---

## Exam Traps & Gotchas

- ⚠️ **Linux only** — no Windows. Windows shared file storage → **FSx for Windows**, not EFS.
- ⚠️ **Access is via mount targets**, not directly to the file system. Mount targets live in subnets and take a subnet IP.
- ⚠️ **HA requires a mount target per AZ** (same as NAT gateways). Missing an AZ = not fully HA.
- ⚠️ **Private by default**, but hybrid access **is** possible over **VPN / Direct Connect / VPC peering** — _if configured_. Don't assume EFS is unreachable from on-prem.
- ⚠️ **Defaults have shifted** (see callouts): course says default throughput = **Bursting** and there are 2 performance / 2 throughput / 2 storage modes. Current AWS: default throughput = **Elastic**, Max I/O is previous-gen, and there are more storage classes. Watch the phrasing of the question.
- 🎯 **EBS = block / single-AZ / usually one instance; EFS = file / multi-AZ / many instances.** A classic "shared storage across multiple instances" trigger points to **EFS**.

---

## Quick-Reference Table

|Attribute|Detail|Trigger keyword 🎯|
|---|---|---|
|Protocol|NFS v4 (docs: v4.0 & 4.1)|"NFS", "shared file system"|
|OS support|**Linux only**|"Windows" → _not_ EFS (→ FSx)|
|Storage type|**File** storage (not block)|"shared across instances"|
|Access method|**Mount targets** (in subnets, subnet IPs)|"mount target"|
|HA rule|Mount target in **every AZ** used|"highly available", "multi-AZ"|
|Scope|**Private**, VPC-isolated by default|"private service"|
|Hybrid access|VPN / Direct Connect / VPC peering (if configured)|"on-premises access to shared files"|
|Permissions|**POSIX**|"POSIX", "Linux permissions"|
|Performance modes|General Purpose _(default)_ / Max I/O _(prev-gen)_|"latency-sensitive" → GP; "highly parallel / big data" → Max I/O|
|Throughput modes|Bursting _(course default)_ / Provisioned / **Elastic _(current default)_**|"scales with size" → Bursting; "constant, size-independent" → Provisioned|
|Storage classes|Standard _(default)_ / IA (+ One Zone & Archive today)|"infrequently accessed" → IA|
|Cost optimization|**Lifecycle policies** move data between classes|"move to cheaper tier automatically"|