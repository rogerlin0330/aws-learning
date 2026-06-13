# EBS Optimized Instances

## Overview

Whether an instance is **EBS optimized** depends on a single option set **per instance** — it's either **on** or **off**.

In short: EBS optimization means certain **stack optimizations** have been applied and **dedicated capacity** has been provisioned on the instance specifically for **EBS (storage) networking**.

---

## Context: Why It Exists

To understand what EBS optimization does, recall the relevant background:

- **EBS is block storage for EC2 that is delivered over the network** — it is _not_ local to the instance.
- **Historically**, EC2 instances used a **single shared network stack** for **both** data networking **and** EBS storage networking.

⚠️ Sharing one stack caused **contention** between the two traffic types, **limiting performance for both**.

---

## What EBS Optimization Does

Enabling EBS optimization:

- Applies **stack optimizations**, and
- Provides **dedicated capacity** for the instance's EBS usage.

Results:

- **Faster speeds** are possible with EBS.
- The **storage side no longer impacts data networking** performance, **and vice versa** — the two are isolated.

🎯 Think of it simply as: **adding dedicated storage-networking capacity to an EC2 instance.**

---

## When It Matters

EBS optimization is **required** on instance types and sizes that offer **higher levels of performance** — i.e., those promising high **throughput** and **IOPS**.

This is especially relevant when using volume types that promise **low, consistent latency** and **high IOPS**, such as:

- **gp2**
- **io1**

> _Note (illustrative, not exhaustive)_: gp2 and io1 are the specific volume types called out in the source as benefiting from EBS optimization. They are examples of high-performance volume types, not the complete set of EBS volume types.

---

## Availability & Cost

- On **most modern instances**, EBS optimization is **supported and enabled by default at no extra charge**.
- **Disabling it has no effect** on modern instances, because the hardware now ships with the capability **built in**.
- On some **older instances**, it is **supported but enabling it costs extra**.

---

## 🎯 Exam Quick-Reference Recap

- **What it is**: A per-instance **on/off** option that gives the instance **dedicated capacity for EBS storage networking**.
- **Problem it solves**: Historically, **data and EBS storage traffic shared one network stack** → contention → limited performance for both.
- **Effect**: Faster EBS speeds; **storage and data networking no longer contend** with each other.
- **Required for**: High-performance instances needing high **throughput / IOPS**, especially with **gp2** and **io1** volumes.
- **Cost/default**: Enabled by default & free on modern instances (disabling does nothing); costs extra to enable on some older instances.
- **Key reminder**: EBS is **network-attached** block storage — which is precisely why dedicated storage-networking capacity matters.

---

_References: AWS docs on EBS-optimized instances — https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-optimized.html_

> **Editorial note**: The exam-trigger callouts and recap section were added editorially per your note conventions. The clarification that gp2/io1 are illustrative high-performance examples (not the full volume-type list) reflects your "precision in scope" principle — the transcript only names these two.