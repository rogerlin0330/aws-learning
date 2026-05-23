# S3 Lifecycle Configuration

## Core Concept

A **lifecycle configuration** is a set of rules applied to an S3 bucket that automatically transitions objects between storage classes or expires them. Rules follow a simple pattern: _do X if Y is true_.

**Scope of rules:**

- Whole bucket, OR
- Subset of objects filtered by **prefix** or **tags**

**Versioning:** Rules can also act on object versions in version-enabled buckets — powerful but adds complexity, so plan carefully.

---

## The Two Action Types

### 1. Transition Actions

Change the **storage class** of an object.

Example pipeline:

- Standard → Standard-IA after 30 days (object access drops off)
- Standard-IA → Glacier Deep Archive after 90 days (rarely accessed)

### 2. Expiration Actions

**Delete** the object (or specific versions) after a defined period. Useful for keeping buckets tidy and controlling storage costs over the long tail.

> ⚠️ Important: Lifecycle rules are **time-based, not access-based**. If you want movement driven by access patterns, that's a different feature — **S3 Intelligent-Tiering**.

---

## The Transition Waterfall

Transitions only flow **downward** — never up. Think of it as a waterfall through the storage class hierarchy:

![[s3-lifecycle-config.png]]

|From|Can transition to|
|---|---|
|**S3 Standard**|All other classes|
|**Standard-IA**|Intelligent-Tiering, One Zone-IA, Glacier Instant, Glacier Flexible, Glacier Deep Archive|
|**Intelligent-Tiering**|One Zone-IA, Glacier Instant, Glacier Flexible, Glacier Deep Archive|
|**One Zone-IA**|Glacier Flexible, Glacier Deep Archive ⚠️ (**not** Glacier Instant Retrieval)|
|**Glacier Instant Retrieval**|Glacier Flexible, Glacier Deep Archive|
|**Glacier Flexible Retrieval**|Glacier Deep Archive|
|**Glacier Deep Archive**|(terminal — no transitions out)|

**Key exception to memorize:** One Zone-IA → Glacier Instant Retrieval is **not allowed**.

You can jump directly between most classes (skip stages), but in practice multi-stage pipelines that mirror real-world access patterns are more common.

---

## Restrictions & Gotchas (Exam-Relevant)

### 1. Small object cost trap

Standard-IA, Intelligent-Tiering, and One Zone-IA all have **minimum object size charges**. Transitioning many small objects into these classes can cost **the same or more** than leaving them in Standard.

### 2. The 30-day minimum in Standard

An object must live in **S3 Standard for at least 30 days** before a lifecycle rule can transition it to **Standard-IA** or **One Zone-IA**.

- Direct upload into Standard-IA / One Zone-IA: ✅ allowed
- Manual storage class change via CLI/console: ✅ allowed
- **Lifecycle transition before 30 days**: ❌ not allowed

### 3. The "single rule" 30-day gap (the obscure one)

Within a **single lifecycle rule**, you cannot transition:

> Standard → Standard-IA / One Zone-IA → Glacier (any class)

…in less than 30 days at the IA stage. The object must sit in the IA tier for 30 days before the same rule pushes it to Glacier.

**Workaround:** Use **two separate rules** — that constraint doesn't apply across rules.

---

## Quick Reference — When to Use What

|Goal|Action type|
|---|---|
|Cost optimization on aging data|Transition|
|Bucket cleanup / retention compliance|Expiration|
|Access-driven cost optimization (unpredictable patterns)|Intelligent-Tiering (not lifecycle)|
|Predictable access decay over time|Lifecycle transitions|

---

## Mental Model

Lifecycle = **scheduled, time-based** automation. Intelligent-Tiering = **reactive, access-based** automation.

They solve different problems — don't conflate them.