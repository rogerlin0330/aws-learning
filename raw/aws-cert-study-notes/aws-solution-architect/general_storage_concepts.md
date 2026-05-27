# Storage Concepts — Study Note

A foundational refresher on storage terminology, categories, and performance. General concepts (not vendor-specific), with AWS examples included where the original lecture used them as illustrations.

---

## 1. Key Storage Terminology

### Direct-Attached (Local) Storage

- Physical disks connected **directly** to a device (laptop, server, or virtualization host).
- **Pros:** Very fast — sits right next to the hardware.
- **Cons:** Fragile. Storage can be lost if:
    - The disk fails
    - The host hardware fails
    - The instance/VM moves to a different host
- AWS example: the **Instance Store** on an EC2 host.

### Network-Attached Storage

- Storage volumes are created separately and attached over a network.
- On-premises protocols: **iSCSI**, **Fibre Channel**.
- AWS example: **EBS (Elastic Block Store)**.
- **Pros:** Highly resilient; survives host-level failures because it lives independently of the compute hardware.

### Ephemeral vs. Persistent Storage

|Type|Lifetime|Example|
|---|---|---|
|**Ephemeral**|Temporary — disappears or becomes unreliable when the device/instance is gone|Instance Store (local to EC2 host)|
|**Persistent**|Exists independently; outlives the device it's attached to|EBS network volumes|

> 📌 Exam tip: be ready to identify which storage types are ephemeral vs. persistent.

---

## 2. The Three Main Categories of Storage

### a) Block Storage

- Presented as a **volume made of addressable blocks** (no inherent structure).
- The operating system lays a **file system** (NTFS, ext3/4, etc.) on top and mounts it (e.g., C: drive, root volume).
- Delivered by physical HDDs/SSDs or as a logical volume backed by them.
- Real-world forms: a server's internal disk, SAN, network-attached block volumes.
- **Can be mounted ✅** and **can be booted from ✅**.
- AWS example: EBS volumes — used as the **boot volume** for most EC2 instances.

### b) File Storage

- Presented as a **ready-made file system** with folders and files.
- You access files by traversing the directory structure.
- Typically accessed over the network from a file server.
- **Can be mounted ✅** but **cannot be booted from ❌** (the OS doesn't get low-level block access).

### c) Object Storage

- A **flat collection of objects** in a container — no folder hierarchy.
- Each object has a **key** (identifier), a **value** (the data), and optional **metadata**.
- Retrieval: give the key, get the value back.
- Objects can be anything: images, videos, binaries, documents, etc.
- AWS example: **S3 buckets** holding objects.
- **Cannot be mounted ❌** and **cannot be booted from ❌**.
- **Strength:** massively scalable; supports huge concurrent read/write access.

### Quick Decision Guide

|Need|Best Category|
|---|---|
|Boot an OS|Block|
|High-performance disk inside an OS|Block|
|Shared file system across multiple servers/clients|File|
|Massive, web-scale read/write of discrete items|Object|

---

## 3. Storage Performance — Three Linked Metrics

### The Three Terms

1. **IO / Block Size** — the size of each chunk of data read/written (in KB or MB).
2. **IOPS** (Input/Output Operations Per Second) — how many read/write operations happen per second.
3. **Throughput** — total data transferred per second (typically MB/s).

### The Race Car Analogy

- **IOPS** = engine RPM
- **Block size** = wheel size
- **Throughput** = the car's actual speed

All three are connected. Change one and the others are affected.

### The Core Relationship

```
Throughput  =  Block Size  ×  IOPS
```

**Example:** 16 KB block size × 100 IOPS = 1.6 MB/s throughput.

But there are caveats:

- If your app only writes in 4 KB chunks and the disk caps at 100 IOPS, throughput is just 400 KB/s.
- Increasing block size doesn't always scale linearly — many storage systems have **throughput caps** or see **IOPS drop** as block size grows.

### The Performance Chain

Performance is limited by the **slowest link** in the chain:

```
Application → OS → Storage subsystem → Transport (network or SATA bus) → Drive interface → Drive itself
```

Any one of these can be the bottleneck (the "lowest common denominator").

### Other Factors

- **Media type** matters: SSDs vs. HDDs deliver different IOPS and throughput profiles.
- **Network latency** between an instance and network storage can severely limit IOPS.
- To maximize throughput: use the right block size **and** push enough IOPS — but watch for system-imposed caps.

---

## 4. Why This Matters for the Exam / Real Decisions

When choosing a storage type, you'll weigh:

- **Persistence requirements** — does the data need to outlive the compute instance?
- **Access pattern** — boot device? Shared file system? Scalable object store?
- **Performance targets** — required IOPS, required throughput, acceptable latency.
- **Workload block size** — affects how much throughput you can actually realize.

Different storage products advertise different caps (e.g., max IOPS, max MB/s). Knowing these limits is how you match a workload to the right storage type.

---

## Key Takeaways

- **Direct-attached** = fast but fragile; **network-attached** = resilient.
- **Ephemeral** dies with the instance; **persistent** survives independently.
- **Block** = boot + raw volumes. **File** = shared file systems. **Object** = flat, web-scale storage.
- Performance is a **triangle**: Block Size × IOPS = Throughput, all constrained by the weakest link in the chain.