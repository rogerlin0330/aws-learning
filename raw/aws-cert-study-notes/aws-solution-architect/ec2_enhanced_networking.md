# EC2 Enhanced Networking

## Overview

Enhanced networking is an EC2 feature designed to improve overall **networking performance** for an instance. It's a foundational capability — it's **required** for any high-end performance feature such as **cluster placement groups**.

It works using a technique called **SR-IOV** (Single Root I/O Virtualization). At a high level, SR-IOV makes the **physical network interface card (NIC) inside the EC2 host aware of virtualization**.

---

## The Problem: Networking Without Enhanced Networking

To understand the benefit, it helps to see what networking looks like _without_ the feature.

- Each EC2 instance has a **virtual network interface**.
- All of those virtual interfaces talk back to the **EC2 host**.
- The host has a **single physical NIC** that is **not aware of virtualization**.

Because the physical card can't manage multiple instances on its own, the **host has to sit in the middle**, controlling which instance gets access to the physical card at any one time.

⚠️ This mediation happens **in software**, which means:

- It's **slow**.
- It **consumes a lot of host CPU**.
- Under heavy load (CPU or I/O), it causes **drops in performance, spikes in latency, and inconsistent (changing) bandwidth**.

It's an inefficient system — the host CPU is a bottleneck.

---

## The Solution: SR-IOV / Enhanced Networking

With enhanced networking, the host's NICs are **virtualization-aware**. Instead of presenting a single physical card the host must manage, the physical card offers **multiple logical cards** (think: many logical cards per physical card).

- Each instance is given **exclusive access to one logical card**.
- The instance sends data to its logical card **as if it had its own dedicated physical card**.
- The physical NIC handles the process **end-to-end** without consuming large amounts of host CPU.

The key shift: the **host CPU is taken out of the middle**, and the work is **offloaded to the physical NIC**.

---

## Diagram Walkthrough

![[ec2-enhanced-networking-architect.png|237]]

The infographic stacks two scenarios vertically — read the **top half first** (No Enhanced Networking), then the **bottom half** (Enhanced Networking).

**Top — No Enhanced Networking:**

- Two EC2 instances (the orange chip icons) sit at the top.
- Each instance has **one virtual network interface** (the purple circular NIC icons) hanging below it.
- Both dotted connection lines converge on a **single physical NIC** (the green card) embedded in the red **HOST** bar.
- The takeaway from the visual: two instances, **one shared physical card**, host in the middle.

**Bottom — Enhanced Networking:**

- Same two instances, each again with its virtual interface.
- This time each dotted line connects to its **own separate logical card** (the two small grey card icons) before reaching the host's green physical NIC.
- The takeaway: each instance gets a **dedicated logical card**, so there's no software mediation contention at the host.

The visual difference to internalize: _one shared card with the host arbitrating_ (top) versus _per-instance logical cards offered directly by a virtualization-aware physical card_ (bottom).

---

## Benefits (What Matters to a Solutions Architect)

Because the host CPU is no longer doing the heavy lifting, enhanced networking delivers:

|Benefit|Why it happens|
|---|---|
|**Higher I/O across all instances + lower host CPU**|The host CPU isn't involved the way it is without the feature.|
|**More bandwidth / faster networking speeds**|The system can scale without impacting host CPU.|
|**Higher packets per second (PPS)**|Traffic flows directly between the instance's virtual interface and the card's logical interface.|
|**Low _and consistent_ latency**|Work is offloaded to the physical NIC, so latency isn't subject to host CPU load.|

🎯 **Exam trigger keywords**: "consistent low latency," "high packets per second / PPS," "small packets around a small isolated network," "required for cluster placement groups." Any of these point toward enhanced networking.

---

## Availability & Cost

- On most **modern EC2 instance types**, enhanced networking is **either enabled by default or available for no charge**.
- There's operational detail involved in confirming it's enabled, but **for the Solutions Architect exam stream, the implementation details are not important**.

---

## 🎯 Exam Quick-Reference Recap

- **What it is**: A feature improving EC2 networking performance using **SR-IOV** (Single Root I/O Virtualization).
- **How SR-IOV works**: Makes the host's **physical NIC aware of virtualization**, presenting **logical cards** that each instance accesses exclusively.
- **Without it**: Host mediates a single physical card **in software** → slow, high host CPU, latency spikes, inconsistent bandwidth.
- **With it**: Direct virtual-interface-to-logical-card path, host CPU offloaded.
- **Delivers**: ↑ bandwidth, ↑ PPS, **consistent low latency**, ↓ host CPU.
- **Required for**: **cluster placement groups** and other high-end performance features.
- **Cost/default**: Enabled by default or free on most modern instance types.

---

_References: AWS docs on enhanced networking — https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/enhanced-networking.html_

> **Editorial note**: The benefits table and the exam trigger-keyword callout were added editorially to fit your note conventions; the underlying points all come from the transcript. The diagram-walkthrough icon descriptions (colors/shapes) are my reading of the supplied infographic to help you map prose to the visual.