# Amazon S3 Replication — Study Note

## 1. Overview

S3 Replication copies objects from a **source bucket** to one or more **destination buckets**. There are two flavors, and they differ only in where the destination bucket lives:

- **Cross-Region Replication (CRR)** — source and destination buckets are in different AWS regions. The older, longer-available form.
- **Same-Region Replication (SRR)** — source and destination buckets are in the same AWS region. A more recent addition.

Both CRR and SRR support the buckets being in either the **same AWS account** or **different AWS accounts**.

---

## 2. Architecture

The diagram below shows the two architectural variants: same-account (top half) and cross-account (bottom half).

![[s3-replication.png]]

*Figure 1 — S3 Replication architecture.* Source: learn.cantrill.io

### 2.1 Shared building blocks

- **Replication configuration** — applied to the **source bucket**. It specifies the destination bucket and the IAM role to use.
- **IAM role** —
  - Trust policy: allows the **S3 service** to assume it.
  - Permissions policy: grants read on the source bucket and write/replicate permissions on the destination bucket.
- **Encryption in transit** — replication traffic is always encrypted (SSL).

### 2.2 Same-account replication

Both buckets sit in the same AWS account, so they both trust that account — and therefore both trust IAM and the replication role. As long as the role's permissions policy grants read on the source and write on the destination, replication just works. **No bucket policy needed.**

### 2.3 Cross-account replication

The destination bucket does **not** implicitly trust the source account or its IAM role. To bridge that gap, you must attach a **bucket policy** (a resource-based policy) on the destination bucket that explicitly allows the source-account role to replicate objects into it.

> **Mental shortcut:** Same account → IAM role is enough. Different account → IAM role **plus** a destination bucket policy.

---

## 3. Replication Configuration Options

### 3.1 What to replicate

- **Default:** the entire source bucket — all objects, prefixes, and tags.
- Can be scoped down with rules that filter by **prefix**, by **tags**, or a combination of both.

### 3.2 Destination storage class

- **Default:** destination objects use the same storage class as the source.
- Can be overridden — e.g. use **One Zone-IA** in the destination if it's a secondary copy where reduced resiliency is acceptable. The primary copy in the source cushions the risk.

### 3.3 Object ownership

- **Default:** replicated objects are owned by the **source bucket's account**.
- In cross-account setups this is a trap — the destination account may end up unable to read objects in its own bucket because they belong to the source account. Override the setting so the **destination account** becomes the owner of replicated objects.

### 3.4 Replication Time Control (RTC)

- Adds a **guaranteed 15-minute replication SLA**, plus extra monitoring (you can see which objects are queued).
- Without RTC, replication is best-effort.

> **Exam shortcut:** if a question mentions "15 minutes" for replication, the answer involves RTC.

---

## 4. Considerations & Gotchas

### 4.1 Not retroactive by default

- When you enable replication, only objects created **from that point forward** are replicated. Pre-existing objects are not.
- **S3 Batch Replication** can backfill existing objects, but you must explicitly configure it.

### 4.2 Versioning is required

- Both source and destination buckets must have versioning enabled. You can turn it on as part of the replication setup wizard.

### 4.3 One-way by default

- Objects flow source → destination. Manual writes to the destination are **not** replicated back.
- **Bidirectional replication** exists but must be explicitly configured.

### 4.4 Encryption support

| Encryption type               | Replication support                                                            |
| ----------------------------- | ------------------------------------------------------------------------------ |
| Unencrypted                   | Supported by default                                                           |
| SSE-S3                        | Supported by default                                                           |
| SSE-KMS                       | Supported, but requires extra configuration and additional IAM/KMS permissions |
| SSE-C (customer-managed keys) | Supported only recently; historically incompatible with CRR/SRR                |

### 4.5 Source-account ownership requirement

- Replication only replicates objects that the **source bucket's account** actually owns.
- If cross-account writes (via a bucket policy on the source) have placed third-party-owned objects in the source bucket, those will **not** replicate.

### 4.6 System events are not replicated

- Changes made by **S3 Lifecycle Management** (transitions, expirations) do not replicate.
- Only **user-driven events** are replicated.

### 4.7 Glacier classes are excluded

- Objects in **Glacier** or **Glacier Deep Archive** storage classes cannot be replicated. Think of those classes as conceptually separate storage products that just happen to be addressable through S3.

### 4.8 Deletes are not replicated by default

- Delete markers (which represent deletions on a version-enabled bucket) are **not** replicated by default.
- You can opt in to replicating delete markers, but it must be turned on explicitly.

---

## 5. When to Use Each Type

### 5.1 Same-Region Replication (SRR) use cases

- **Log aggregation** — centralize logs from many buckets into a single bucket for analysis.
- **Prod ↔ test sync** — periodically push real data from prod into test, or push test data into prod, depending on the workflow.
- **Sovereignty-bound resilience** — when data must remain within one region, replicate cross-account inside that region. The separate account provides isolation (often used for audit or security teams).

### 5.2 Cross-Region Replication (CRR) use cases

- **Global resilience** — survive large-scale, region-wide failures by keeping a copy elsewhere.
- **Latency reduction** — place data closer to end users in a remote region for faster reads (e.g., latency-sensitive web apps).

---

## 6. Quick Recall Cheat Sheet

| Concept                     | Key point                                                                        |
| --------------------------- | -------------------------------------------------------------------------------- |
| CRR vs. SRR                 | Different regions vs. same region. Both work same-account or cross-account.      |
| What configures replication | Replication configuration on the **source bucket** + IAM role.                   |
| Cross-account extra step    | **Bucket policy** on the destination bucket allowing the source-account role.    |
| Versioning                  | **Required** on both buckets.                                                    |
| Retroactive?                | No — use **S3 Batch Replication** to backfill.                                   |
| Direction                   | One-way by default; **bidirectional** is opt-in.                                 |
| Storage class               | Same as source by default; can be a cheaper class on the destination.            |
| Object ownership            | Source account by default; **override** in cross-account setups.                 |
| RTC                         | Guaranteed **15-minute SLA** + monitoring. "15 minutes" in a question → RTC.     |
| Encryption                  | Unencrypted, SSE-S3, SSE-KMS (config needed), SSE-C (recent).                    |
| Not replicated              | Lifecycle (system) events, Glacier / Deep Archive objects, deletes (by default). |
