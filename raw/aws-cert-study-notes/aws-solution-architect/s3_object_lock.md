# S3 Object Lock

## Foundational Concepts

**S3 Object Lock** is a group of related features that implement a **Write-Once-Read-Many (WORM)** architecture on S3 buckets. Once enabled, object versions can be protected from being overwritten or deleted.

### Enablement Rules

- **Must be enabled at bucket creation time** for new buckets.
- For **existing buckets**, you must contact AWS Support to enable it (subject to change).
- Enabling Object Lock **automatically enables versioning** on the bucket.
- Once enabled, you **cannot disable Object Lock** or **suspend versioning** on that bucket — these are permanent decisions.

### Key Architectural Point

Because versioning is required, **Object Lock applies to individual object versions, not to objects as a whole**. Each version is independently locked or unlocked.

### Two Retention Methods

Object Lock manages retention through two distinct mechanisms. An object version can have:

- A retention period only
- A legal hold only
- **Both** (effects overlap — if either restricts an action, the action is blocked)
- Neither

Both can be configured **per object version** or as **bucket defaults** (similar to default encryption settings).

---

## Mechanics

### Retention Period

You specify a duration in **days and/or years**. The lock expires when the period ends. Within the retention period, behavior depends on the **mode** you choose.

#### Compliance Mode (Strictest)

|Action|Allowed during retention?|
|---|---|
|Delete object version|❌ No|
|Overwrite object version|❌ No|
|Shorten retention period|❌ No|
|Change retention mode|❌ No|
|Bypass (even as account root)|❌ **No — not even root**|

**Compliance mode is permanent for the duration of the retention period.** No identity in the AWS account — including the root user — can modify the object version or the lock settings until the retention period expires.

> ⚠️ Don't use compliance mode unless you genuinely require the object version to remain unchanged for the full retention period. Mistakes here cannot be reversed.

#### Governance Mode (Bypassable)

Same protections as compliance — object version cannot be deleted, overwritten, or have its retention shortened — **except** that identities with the right permission can override the lock.

To bypass governance mode, an identity needs **both**:

1. The IAM permission `s3:BypassGovernanceRetention`
2. The request header `x-amz-bypass-governance-retention: true`

> 📝 The bypass header is **set by default in the AWS Console UI**, so console users with the IAM permission can override governance locks without manually adding it.

### Legal Hold

- **No retention period** — it's a binary flag: **ON** or **OFF**.
- While ON, the object version cannot be deleted or overwritten.
- Toggling legal hold (on or off) requires the `s3:PutObjectLegalHold` permission.
- Useful for **flagging specific object versions** as critical to a legal case or project, with no predefined end date.

---

## Visual Comparison

The diagram below shows the three Object Lock modes side by side — how each is applied, what restrictions it places on the object version, and how (or whether) the lock can be lifted.

![[s3-object-lock-comparison.png]]

Key things to read off the diagram:

- **Legal Hold (top)**: Toggled via `s3:PutObjectLegalHold` Status=ON/OFF. Once OFF, normal permissions apply and the object version can be deleted.
- **Governance (middle)**: Locks for a set period in days/years. Can be bypassed with `s3:BypassGovernanceRetention` + the bypass header.
- **Compliance (bottom)**: Same lock-for-days/years, but **no bypass exists, even for the account root user**. Only retention expiry can unlock it.
- All three can be set on **individual objects** or as **bucket defaults**.

---

## Decision Guidance

|Situation|Recommended mode|
|---|---|
|Regulatory requirement to retain data unchanged (e.g., medical, financial records) for a fixed period|**Compliance**|
|Want to prevent accidental deletion but keep an escape hatch for authorized admins|**Governance**|
|Testing Object Lock behavior before committing to compliance|**Governance**|
|Internal governance/process reasons to retain object versions|**Governance**|
|Indefinite hold for an active legal case or critical project (no fixed end date)|**Legal Hold**|
|Need both a fixed minimum retention AND ability to flag certain versions for a case|**Combine retention period + legal hold**|

---

## Required Permissions Summary

|Action|Permission needed|
|---|---|
|Configure object lock on a bucket|`s3:PutBucketObjectLockConfiguration`|
|Set/change a retention period on an object version|`s3:PutObjectRetention`|
|Bypass governance mode|`s3:BypassGovernanceRetention` + `x-amz-bypass-governance-retention: true` header|
|Add or remove a legal hold|`s3:PutObjectLegalHold`|

---

## Exam Cheat Sheet

|Keyword / Scenario|Answer|
|---|---|
|Write-Once-Read-Many on S3|**S3 Object Lock**|
|Enable Object Lock on an existing bucket|**Contact AWS Support**|
|Object Lock prerequisite|**Versioning** (auto-enabled)|
|Object Lock applies to...|**Individual object versions**|
|Cannot be deleted or modified, ever, **even by root**, until retention expires|**Compliance mode**|
|Cannot be deleted or modified, but can be bypassed with the right IAM permission|**Governance mode**|
|Permission to override governance mode|`s3:BypassGovernanceRetention`|
|Header required to override governance mode|`x-amz-bypass-governance-retention: true`|
|Is the bypass header set by default in the console?|**Yes**|
|Binary on/off lock with no retention period|**Legal Hold**|
|Permission to toggle legal hold|`s3:PutObjectLegalHold`|
|Retention period units|**Days and/or years**|
|Can Object Lock be disabled once enabled on a bucket?|**No**|
|Can versioning be suspended on an Object Lock bucket?|**No**|
|Can retention period and legal hold coexist on the same object version?|**Yes** — effects overlap|
|Can Object Lock be applied as a bucket default?|**Yes** (same pattern as default encryption)|

---

## Common Gotchas

- **Compliance mode is irreversible during retention** — even the root user is locked out. Treat it as a one-way door.
- **Object Lock cannot be retrofitted** to an existing bucket without contacting AWS Support.
- The bypass header (`x-amz-bypass-governance-retention: true`) is **default in the console**, so console-based deletes by users with `s3:BypassGovernanceRetention` may succeed unexpectedly — be careful with how you scope this permission.
- Retention and legal hold are **independent and additive** — removing one doesn't unlock the other.
- Locks are on **versions**, not objects — a new PUT creates a new (unlocked) version unless the bucket default applies.