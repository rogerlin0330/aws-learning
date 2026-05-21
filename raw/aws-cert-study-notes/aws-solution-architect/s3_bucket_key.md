# AWS S3 Bucket Keys — Study Notes

> Source: Adrian Cantrill — S3 Bucket Keys
>
> Context: builds on SSE-KMS. Read after the S3 SSE notes.

---

## Visual Overview

![[s3-bucket-key-infographic.png]]

At a glance: standard SSE-KMS hits KMS for **every object**, leading to throttling and rising costs. With Bucket Keys, S3 requests a **time-limited bucket key** periodically and handles per-object DEK derivation locally.

---

## 1. The Problem — Why Bucket Keys Exist

### Baseline SSE-KMS behavior

With standard SSE-KMS, **every single object PUT** results in a separate call to KMS to generate a unique Data Encryption Key (DEK):

1. Bob uploads object → S3 calls KMS → `GenerateDataKey` against the KMS key.
2. KMS returns a unique DEK for that object.
3. S3 encrypts the object with the DEK and stores them side by side.
4. Repeat for **every** object.

### The two costs of this pattern

1. **💰 Per-call KMS cost** — KMS charges per API request. Cost scales linearly with object count. At 300k PUTs/sec, this gets painful fast.
2. **🚦 KMS throttling** — `GenerateDataKey` is rate-limited per region:
   - **5,500 / 10,000 / 50,000 requests per second** depending on region.
   - This limit is **shared across the region** for a given KMS key.
   - At scale, this effectively caps your S3 PUT rate when using SSE-KMS.

---

## 2. How Bucket Keys Fix It

The architecture shifts the workload **from KMS to S3**:

| Without Bucket Keys                    | With Bucket Keys                                            |
| -------------------------------------- | ----------------------------------------------------------- |
| KMS key generates a DEK **per object** | KMS key generates a **time-limited bucket key**             |
| 1 KMS API call per PUT                 | 1 KMS API call per bucket key generation (covers many PUTs) |
| KMS does all DEK generation work       | S3 uses the bucket key to derive DEKs locally               |

### The flow

1. S3 calls KMS **once** to generate a **bucket key** (time-limited, conceptually scoped to the bucket).
2. S3 then uses that bucket key to generate per-object DEKs **itself**, without calling KMS again.
3. When the bucket key expires, S3 requests a new one from KMS.

### Result

- ✅ **Fewer KMS API calls** → lower cost.
- ✅ **Bypasses the per-region `GenerateDataKey` throttle** for high-throughput workloads.
- ✅ **Improved scalability** for S3 PUT operations.

---

## 3. Important Caveats

### Not retroactive

- Enabling a bucket key **only affects new objects** PUT *after* it's enabled.
- Existing objects keep their original per-object KMS DEKs. To convert them, you'd need to re-upload or copy them in place.

### CloudTrail behavior changes

After enabling a bucket key, KMS-related CloudTrail logs change:

- ❗ Logs show the **bucket ARN** instead of the **object ARN** — because KMS is now being called for the bucket, not for individual objects.
- ❗ **Fewer KMS CloudTrail events overall** — logical, since you're making fewer KMS calls.
- ⚠️ This affects audit/forensics workflows that rely on per-object KMS audit trails.

---

## 4. Replication Behavior (SRR & CRR)

Bucket keys work with both **Same-Region Replication** and **Cross-Region Replication**, but with nuances:

### General rule

S3 **preserves the encryption settings** of the source object when replicating an encrypted object to the destination bucket.

### Plaintext source → encrypted destination

- If the source object is unencrypted, and the destination bucket has default encryption / a bucket key configured:
  - S3 **encrypts the object on the way in** using the destination bucket's settings.
  - ⚠️ This can cause **ETag changes** between source and destination — important for any system that uses ETags for integrity checking or change detection.

---

## 5. Exam-Day Cheat Sheet

- **Why bucket keys exist** → reduce KMS cost + bypass KMS `GenerateDataKey` throttling at scale.
- **What changes** → KMS generates a bucket key, not a per-object DEK; S3 derives DEKs from the bucket key.
- **Throttle limits** → 5,500 / 10,000 / 50,000 per second per region (shared).
- **Not retroactive** → only applies to new PUTs after enablement.
- **CloudTrail** → bucket ARN appears in logs (not object ARN); fewer KMS events overall.
- **Replication** → source encryption settings preserved; plaintext → encrypted destination can change ETags.
- **When to enable** → high-volume SSE-KMS workloads where KMS cost or throttling is a concern.

---

## 6. Quick Mental Model

> *Without bucket keys:* KMS is on the critical path for every single object.
>
> *With bucket keys:* KMS is on the critical path only when the bucket key rotates. Everything in between is handled by S3 alone.