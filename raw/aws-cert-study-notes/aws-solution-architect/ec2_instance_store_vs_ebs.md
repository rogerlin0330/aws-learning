# Choosing Between EC2 Instance Store and EBS

## The Big Picture

This lesson is two things: a **decision framework** (EBS vs. instance store vs. "it depends") and a **set of performance numbers the lecturer explicitly says to memorize**. The numbers are an unusual exception — this is one of the few AWS topics where rote memorization of figures genuinely pays off on the exam.

---

## Default Rule

> **If you need persistent storage → default to EBS** (i.e., default _away_ from instance store).

Instance store is not persistent, so it's the wrong starting point whenever durability matters.

---

## Clear-Cut: Use EBS

### Persistence

Data must survive. Instance store data is lost on hardware failure, host failure, maintenance, and **anything that moves an instance between hosts**.

> ⚠️ **Accuracy flag — "reboot":** This transcript lists _"instances rebooting"_ among the data-loss causes. That's imprecise and conflicts with the exam-correct behavior (and with your Instance Store Volumes notes). A **reboot keeps the instance on the same host → instance store data is preserved.** It's **stop/start** (and resize, host failure, volume hardware failure, terminate) that loses the data, because those move the instance to a new host. Treat the lecturer's wording here as loose; remember the AWS rule: **reboot = data survives, stop/start = data lost.**

### Resilience

EBS hardware is resilient **within an Availability Zone**, and you can **snapshot volumes to S3**. Instance store has neither. Need resilient storage → EBS.

### Lifecycle Isolation

Need a volume you can attach to one instance, detach, and reattach to a different instance → EBS. Instance store can't be decoupled from its instance like this.

---

## "It Depends" (exam watch-outs)

These are the scenarios where the answer isn't automatically EBS:

|Scenario|Choice|Why|
|---|---|---|
|Need resilience, **but the app has built-in replication**|**Instance store** (many volumes across many instances)|Replication handles durability; you keep the performance and lose the risk|
|Need **high performance**|Either, up to a point|Both go high; for **super high** performance → instance store|
|**Cost** is the primary concern|**Instance store**|It's included in the price of many instances — use it|

---

## EBS Type Selection (cost & throughput)

- **Cost-efficiency + you've decided on EBS → ST1 or SC1.** These are mechanical (HDD), so cheaper than the SSD-based volumes.
- **Throughput / streaming workloads → ST1.**
- **🎯 Critical exam trap — boot volumes:** Neither **ST1 nor SC1** can be a boot volume. If the question mentions _boot volume_, both mechanical types are **excluded**.

---

## EBS Performance Numbers (memorize these)

|Volume Type|Max IOPS per Volume|Notes|
|---|---|---|
|**GP2 / GP3**|**16,000**|GP2 scales with volume size; GP3 starts at 3,000 IOPS, pay for more — but both cap at 16,000/volume|
|**IO1 / IO2**|**64,000**|Use when you need **between 16,000 and 64,000** IOPS per volume|
|**IO2 Block Express**|**256,000**|Only achievable on **larger instance types**|

> Pair any volume choice with an EC2 instance big enough to actually deliver that performance.

---

## RAID 0 and the Instance Ceiling

- You can combine multiple EBS volumes into a **RAID 0 set** → you get the **combined IOPS** of all volumes.
- **But there's a hard ceiling: ~260,000 IOPS per EC2 instance**, no matter how many volumes you stripe together.
- To reach 260,000 you need a **large instance size** + **enough EBS volumes** to consume that capacity.

---

## Beyond EBS

- Need **more than 260,000 IOPS** **AND** the app can tolerate **non-persistent** storage → **instance store**.
- Instance store can reach **millions of IOPS** with the correct instance type.
- The deal: **trade persistence for performance.**

---

## Quick Decision Cheat-Sheet

| If the question emphasizes…               | Pick                                                    |
| ----------------------------------------- | ------------------------------------------------------- |
| Persistence / durability                  | **EBS**                                                 |
| Resilience within an AZ + snapshots       | **EBS**                                                 |
| Detach & reattach to another instance     | **EBS**                                                 |
| Resilience **with** app-level replication | **Instance store** (across many instances)              |
| Lowest cost (HDD)                         | **ST1 / SC1**                                           |
| Throughput / streaming                    | **ST1**                                                 |
| Boot volume                               | **NOT ST1/SC1**                                         |
| 16,000–64,000 IOPS per volume             | **IO1 / IO2**                                           |
| Up to 256,000 IOPS per volume             | **IO2 Block Express**                                   |
| Max per instance (RAID 0)                 | **~260,000 IOPS** (limit is on the EC2 instances' side) |
| More than 260,000 IOPS, non-persistent OK | **Instance store** (millions of IOPS)                   |
