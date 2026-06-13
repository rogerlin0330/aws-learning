# EC2 Placement Groups

## Overview

Normally when you launch an EC2 instance, AWS selects its **physical location**, placing it on whatever EC2 host makes the most sense within the chosen Availability Zone (AZ).

**Placement groups** let you _influence_ this placement — ensuring instances are either physically **close together** or **deliberately separated** — to optimise for performance or resilience.

There are currently **three types** of placement group. All influence how instances are arranged on physical hardware, but each does so for a different underlying reason:

| Type          | Goal                | One-line summary                                                        |
| ------------- | ------------------- | ----------------------------------------------------------------------- |
| **Cluster**   | Maximum performance | Instances packed physically close together (same rack, often same host) |
| **Spread**    | Maximum resilience  | Instances kept on separate, isolated infrastructure                     |
| **Partition** | Resilience at scale | Groups of instances on isolated infrastructure, with topology awareness |

---

## Cluster Placement Groups

![[ec2-placement-groups-cluster.png]]

### Diagram walkthrough

- **Context bar (bottom):** the whole group sits within a _single_ AZ (`AZA`).
- **Main object (centre):** a 3×3 grid of instances all interconnected with direct links — the diagram shows **all members have direct connections to each other**, illustrating the full-mesh, low-latency networking.
- **Right callouts:** instances are in the **Same Rack** and _sometimes_ the **Same host**; networking achieves **10 Gbps per stream** versus **5 Gbps normally**.
- **Left callout:** the result is the **lowest latency and max packets-per-second (PPS) possible in AWS**.

### Purpose

Used when you need the **absolute highest level of performance** possible within EC2 — fast node-to-node speed and the lowest consistent latency.

### How it works

- You create the group, then (best practice) **launch all instances at the same time** so AWS can allocate capacity for everything you need in one suitable location.
    - ⚠️ If you launch 9 instances and AWS places you somewhere with capacity for ~12, later **doubling** the instance count may fail due to lack of local capacity.
- Best practice is also to use the **same instance type** for all members so AWS places them in a location with capacity for everything.
- You do **not** specify an AZ when creating the group. The group becomes **locked to the AZ** of the _first_ instance(s) launched into it.
- All instances generally share the **same rack**, and often the **same EC2 host**, giving them fast direct bandwidth to each other.

### Performance characteristics

- **10 Gbps single-stream** transfer rate between instances, versus **5 Gbps** normally.
    - ⚠️ This is a **single-stream** figure — the cap on a single connection. Some instances offer much faster aggregate networking, but any one stream is still limited.
- Lowest possible latency and maximum PPS within AWS.
- 🎯 To actually achieve these levels you must use **instances with high-performance networking** (more than 10 Gbps single-stream bandwidth) **and** **enhanced networking** on all instances.

### Trade-off

- Because instances are physically co-located, a **hardware failure can take down the entire group**.
- Cluster placement groups offer **little to no resilience**.

### 🎯 Exam key points

- ❌ **Cannot span AZs** — locked to a single AZ when the first instance launches.
- ✅ **Can span VPC peers**, but this **significantly degrades performance**.
- ⚠️ **Not supported on every instance type** — requires a _supported_ instance type.
- Same instance type → best results (recommended, **not mandatory**).
- Launch all instances at the same time (recommended, **not mandatory**).

### Use cases

Workloads demanding fast speeds and low latency, e.g. **High Performance Compute (HPC)** and **scientific analysis** that need fast node-to-node speed and low consistent latency.

---

## Spread Placement Groups

![[ec2-placement-groups-spread.png]]

### Diagram walkthrough

- **Context bar (bottom):** the group spans **two AZs** (`AZA` and `AZB`) — spread groups _can_ span multiple AZs.
- **Main objects:** within each AZ, instances sit inside separate dotted boxes labelled **Distinct Racks** — each instance has its own isolated rack.
- **Right callout:** the **7 Instances per AZ** isolated-infrastructure limit.

### Purpose

The **inverse of cluster** — designed for **maximum availability and resilience** of an application.

### How it works

- Can **span multiple AZs** (e.g. AZA and AZB).
- Each instance is placed on a **separate, isolated infrastructure rack** within each AZ.
- Each instance therefore has its **own isolated networking and power supply**, separate from every other instance in the same group.
- If a single rack fails (network or power), the fault is **isolated to that one rack** — only that one instance is affected.

### The 7-instance limit

- ⚠️ **Hard limit of 7 instances per AZ**, because each instance needs its own isolated rack and there are limits on the number of these per AZ.
- More AZs in a region → more total instances possible in the group, but always **7 per AZ** maximum.
    - 🎯 _Illustrative:_ a region with 3 AZs → up to 3 × 7 = 21 instances in one spread group. (The per-AZ cap of 7 is the fact to remember; the regional total just follows from AZ count.)

### 🎯 Exam key points

- Provides **infrastructure isolation** — guaranteed separation of every instance from every other in the group.
- Each instance → **different rack**, each rack → **own network and power source**.
- ⚠️ Hard limit of **7 instances per AZ**.
- ❌ **Dedicated Instances and Dedicated Hosts are not supported**.

### Use cases

A **small number of critical instances** that must be kept separated, e.g. **mirrors of a file server** or **multiple domain controllers**. Anywhere you want a **separate blast radius per server**, so that if one fails the chance of others failing is as small as possible.

---

## Partition Placement Groups

![[ec2-placement-groups-partition.png]]

### Diagram walkthrough

- **Context bar (bottom):** spans **two AZs** (`AZA` and `AZB`).
- **Main objects:** each AZ is **divided into 'partitions'** (dotted boxes labelled _Partition 1–6_), and **each partition can hold multiple instances** — note the partitions contain varying numbers of instances, unlike spread's one-instance-per-box.
- **Top callouts:** **MAX 7 partitions per AZ**, and **each partition has its own racks — no sharing between partitions**.

### Purpose

Designed for when you have **more than 7 instances per AZ** but still need to separate them into distinct **fault domains** — i.e. large-scale, distributed/replicated applications with **infrastructure awareness**.

### How it works

- Can be created **across multiple AZs** in a region.
- When creating the group you **specify a number of partitions** — **maximum 7 per AZ**.
- Each partition has its **own racks with isolated power and networking**, and there is a **guarantee of no infrastructure sharing between partitions**.
- _(So far this resembles spread...)_ **but**:
    - You can launch **as many instances as you need** into the group.
    - You either **select the partition explicitly** or let **EC2 decide** placement on your behalf.

### Spread vs Partition — the key distinction

|                       | Spread                         | Partition                                                                      |
| --------------------- | ------------------------------ | ------------------------------------------------------------------------------ |
| Instance cap          | **7 per AZ** (hard)            | **As many as needed** (7 _partitions_ per AZ)                                  |
| Isolation unit        | Per **instance**               | Per **partition**                                                              |
| Who manages placement | **AWS** handles it all         | **You / a topology-aware app** controls which partition                        |
| Resilience guarantee  | Every instance fully separated | Every _partition_ separated; instances _within_ a partition share fault domain |

- ⚠️ If you put **10 instances in one partition** and it fails, you **lose all 10**.
- If you put **7 instances, one per partition**, it behaves much like a spread placement group.

### Topology awareness (the real point)

- Partition placement groups offer **visibility into the partitions** — you can see which instances are in which partition.
- This information can be shared with **topology-aware applications** such as **HDFS, HBase, and Cassandra**, which use it to make **intelligent data-replication decisions**.

> **Worked example (replication):** An app uses 75 EC2 instances, each with its own storage, replicating every piece of data **3 times** → effectively three replication groups of 25 instances.
> 
> - _Without_ partition placement groups, all 75 instances could in theory land on the **same hardware** → no real resilience.
> - _With_ partition placement groups, a topology-aware app can ensure each of the 3 replicas lands in a **separate partition**, achieving spread-like resilience **at large scale**.

### 🎯 Exam key points

- **7 partitions per AZ** maximum.
- Instances can be placed in a **specific partition**, or **EC2 controls placement automatically**.
- Ideal for **topology-aware applications**: **HDFS, HBase, Cassandra**.
- Lets a topology-aware app **contain the impact of a failure** to a specific part of the application — AWS and the application working together for large-scale resilience and effective replication.

---

## 🎯 Exam Quick Reference

### Choose by intent

| If the question emphasises...                                                               | Choose        |
| ------------------------------------------------------------------------------------------- | ------------- |
| Highest performance, lowest latency, fastest node-to-node (HPC, scientific)                 | **Cluster**   |
| Small number of critical instances, max availability, separate blast radius                 | **Spread**    |
| Large-scale distributed/replicated app, >7 instances, topology-aware (HDFS/HBase/Cassandra) | **Partition** |

### Numbers and limits to memorise

| Fact                              | Value                              |
| --------------------------------- | ---------------------------------- |
| Cluster single-stream throughput  | **10 Gbps** (vs **5 Gbps** normal) |
| Spread instances per AZ           | **7** (hard limit)                 |
| Partition **partitions** per AZ   | **7** (max)                        |
| Partition instances per partition | Unlimited (your choice)            |

### Trap-spotting (⚠️)

- **Cluster cannot span AZs** — single AZ only.
- **Cluster = low/no resilience** (single point of physical failure).
- **Spread caps at 7 instances per AZ** — if a scenario needs more separated instances, it's pointing at **Partition**, not Spread.
- **Spread ≠ Dedicated** — Dedicated Instances/Hosts are **not supported** with spread.
- "Need separation **and** lots of instances" → **Partition** (the most common spread-vs-partition trap).
- Cluster requires **supported instance type + enhanced networking** to hit its performance numbers.

### Mnemonic (editorial addition — not from source)

- **C**luster → **C**lose together / **C**ompute performance
- **S**pread → **S**eparated / **S**even per AZ
- **P**artition → **P**artitions (7 per AZ) for **P**arallel-processing apps at scale

---

_Source: Adrian Cantrill — AWS Solutions Architect Associate course (learn.cantrill.io), "EC2 Placement Groups" lesson. Infographics co-located in `images/`._