# S3 Access Points

## 1. Foundational Concept

**S3 Access Points** are named network endpoints attached to an S3 bucket (or an Amazon FSx for OpenZFS/ONTAP volume) that simplify managing data access at scale. They are especially valuable when:

- A bucket holds **billions of objects** across many prefixes
- The bucket is shared by **many teams, identities, or accounts**
- Objects serve **a wide range of functions** with different access requirements

### The Problem They Solve

Without access points, a heavily-shared bucket is governed by a single bucket policy. As the number of identities and use cases grows, that policy becomes:

- Large (bucket policies are capped at 20 KB)
- Complex
- Hard to maintain
- Error-prone

### The Mental Model

Think of an access point as a **mini-bucket** or a **view onto a bucket** with its own DNS name, its own policy, and its own network origin. Instead of one entry point with one monolithic policy, you create many doorways — each independently controlled — into the same underlying data.

> An access point is conceptually similar to a **DNS CNAME**: it does not itself grant access to the data; the underlying bucket policy must still allow the request.

---

## 2. Architecture & Mechanics

![[s3-access-points.png]]

### What Each Access Point Provides

Every access point has **three independent attributes**:

|Attribute|Purpose|
|---|---|
|**Unique DNS / endpoint name**|Replaces the default S3 endpoint; given to specific teams or used by specific resources. Access point names are published in **public DNS** — do not put sensitive info in them.|
|**Access point policy** (≤ 20 KB)|A resource policy functionally equivalent to a bucket policy. Controls permissions for requests made _through this access point_.|
|**Network origin**|`Internet` (default) or `VPC`. **Immutable after creation.**|

Plus a **Block Public Access** setting that is also **immutable after creation** (default: all four sub-settings enabled).

### Example Scenario (Cantrill — Animals for Life)

A single bucket holds sensitive health information. Four access points are layered on top:

- **Admin Staff** — Internet origin, scoped to employee records
- **Field Workers** — Internet origin, scoped to fieldwork data
- **Vet Staff** — Internet origin, scoped to animal medical records
- **VPC analytics** — VPC origin, used by EC2 instances doing data analysis

Each group uses its own DNS endpoint and is governed by its own access point policy.

### Access Point ARN Format

This format is critical because access point policies reference objects via this ARN, **not** the bucket ARN:

```
arn:aws:s3:<region>:<account-id>:accesspoint/<access-point-name>          ← the access point itself (used for s3:ListBucket-style actions)
arn:aws:s3:<region>:<account-id>:accesspoint/<access-point-name>/object/* ← objects accessed through it (s3:GetObject, s3:PutObject, etc.)
```

Compare to the bucket ARN (`arn:aws:s3:::bucket-name/key`) which is **regionless and account-less**. Access point ARNs are region- and account-qualified.

### Access Point Policy Capabilities

An access point policy can scope identities to:

- Specific **prefixes** (via the `/object/<prefix>/*` portion of the resource ARN)
- Specific **object tags** (via the `s3:ExistingObjectTag/<key>` condition)
- Specific **actions** (GetObject, PutObject, ListBucket, etc.)

### VPC-Origin Access Points

When an access point is created with `--vpc-configuration VpcId=vpc-xxxxxxxx`:

- It is **tied to that one VPC** and S3 rejects any request not originating from it
- The VPC needs a **VPC endpoint for S3** (Gateway or Interface)
- The **VPC endpoint policy** must allow access to **both** the access point ARN **and** the underlying bucket ARN — otherwise the call fails

Minimal VPC endpoint policy permitting `GetObject` through one access point:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Principal": "*",
    "Effect": "Allow",
    "Action": ["s3:GetObject"],
    "Resource": [
      "arn:aws:s3:::amzn-s3-demo-bucket/*",
      "arn:aws:s3:us-west-2:123456789012:accesspoint/example-vpc-ap/object/*"
    ]
  }]
}
```

> For the access point DNS to resolve _only_ inside the VPC, also create a **private hosted zone** for the VPC endpoint and ensure `enableDnsHostnames` and `enableDnsSupport` are both `true` on the VPC.

---

## 3. Permissions Model — The Two-Layer Rule

This is the most important operational point and the most common exam trap.

> **An access point policy is only effective if the underlying bucket policy also permits the same access.** Restrictions in an access point policy apply _only_ to requests through that access point — they do not change how the bucket behaves for direct requests.

There are **two ways** to satisfy this rule.

### Pattern A — Matching permissions (tedious)

Duplicate every access point grant in the bucket policy. Defeats most of the manageability win; rarely the right choice.

### Pattern B — Delegation (AWS-recommended)

Write a single permissive bucket policy that says _"any access point owned by my account is fine — go check its policy"_. Push all granular control into the access point policies.

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "AWS": "*" },
    "Action": "*",
    "Resource": [
      "arn:aws:s3:::amzn-s3-demo-bucket",
      "arn:aws:s3:::amzn-s3-demo-bucket/*"
    ],
    "Condition": {
      "StringEquals": { "s3:DataAccessPointAccount": "111122223333" }
    }
  }]
}
```

The `s3:DataAccessPointAccount` condition is the linchpin — it scopes the wide-open `Allow` to requests routed through access points owned by the specified account.

### Useful Condition Keys

|Condition key|Matches on|
|---|---|
|`s3:DataAccessPointArn`|The access point ARN (supports `StringLike` wildcards)|
|`s3:DataAccessPointAccount`|The AWS account that owns the access point|
|`s3:AccessPointNetworkOrigin`|`Internet` or `VPC`|

---

## 4. Cross-Account Access Points

You can create an access point in **your account** that targets a bucket owned by **another account**. Important nuances:

- Creating the access point does **not** grant access to the data — the bucket owner must update the bucket policy to authorize the access point's account.
- The bucket owner always retains ultimate control.
- Access point management API calls (e.g. `PutAccessPoint`, `GetAccessPointPolicy`) **do not support cross-account calls**.
- Cross-account access points are **only supported for S3 buckets**, not for FSx volumes.

Bucket-owner-side policy snippet (delegating GET/LIST to a trusted external account's access points):

```json
{
  "Effect": "Allow",
  "Principal": { "AWS": "*" },
  "Action": ["s3:GetObject", "s3:ListBucket"],
  "Resource": ["arn:aws:s3:::amzn-s3-demo-bucket", "arn:aws:s3:::amzn-s3-demo-bucket/*"],
  "Condition": {
    "StringEquals": { "s3:DataAccessPointAccount": "<trusted-account-id>" }
  }
}
```

---

## 5. Governance — Using SCPs to Enforce Access Point Patterns

Directly relevant for Control Tower / multi-account governance.

### Force all new access points to be VPC-only

Useful as an organization-wide SCP guardrail:

```json
{
  "Effect": "Deny",
  "Action": "s3:CreateAccessPoint",
  "Resource": "*",
  "Condition": {
    "StringNotEquals": { "s3:AccessPointNetworkOrigin": "VPC" }
  }
}
```

### Force a bucket to only be reachable via VPC-origin access points

Bucket policy variant — denies any S3 object operation that doesn't arrive through a VPC-origin access point:

```json
{
  "Effect": "Deny",
  "Principal": "*",
  "Action": ["s3:GetObject", "s3:PutObject", "s3:DeleteObject", "..."],
  "Resource": "arn:aws:s3:::amzn-s3-demo-bucket/*",
  "Condition": {
    "StringNotEquals": { "s3:AccessPointNetworkOrigin": "VPC" }
  }
}
```

> ⚠️ Before enabling a deny-all-non-VPC pattern, verify you don't need features that don't support access points — most notably **S3 Replication cannot use access points as a destination**.

---

## 6. Creating Access Points

### Console

S3 console → **Access Points** → **Create access point**. Required inputs: name, data source (bucket), network origin, block public access settings, and optional access point policy.

### CLI

```bash
# Same-account bucket
aws s3control create-access-point \
  --name example-ap \
  --account-id 111122223333 \
  --bucket amzn-s3-demo-bucket

# Cross-account bucket (note --bucket-account-id)
aws s3control create-access-point \
  --name example-ap \
  --account-id 111122223333 \
  --bucket amzn-s3-demo-bucket2 \
  --bucket-account-id 444455556666

# VPC-restricted
aws s3control create-access-point \
  --name example-vpc-ap \
  --account-id 123456789012 \
  --bucket amzn-s3-demo-bucket \
  --vpc-configuration VpcId=vpc-1a2b3c
```

> The CLI namespace is **`aws s3control`**, not `aws s3` or `aws s3api`. Cantrill abbreviates this to `create-access-point` in the video.

### Naming Rules

- DNS-compliant: lowercase letters / numbers / hyphens, **3–50 chars**
- Must start with a number or lowercase letter; can't begin or end with `-`
- No underscores, uppercase, spaces, or periods
- Can't end with `-s3alias` or `-ext-s3alias` (reserved for access point aliases)
- Must be unique **within an account + region** (not globally)
- Published in public DNS — avoid sensitive content in the name

---

## 7. Limits & Constraints (memorize)

|Limit / Constraint|Value|
|---|---|
|Access points per account per Region|**10,000** (soft limit; raisable via Service Quotas)|
|Access point policy size|**20 KB**|
|Region binding|Access point must be in the **same Region as the bucket**|
|Network origin after creation|**Immutable**|
|Block Public Access settings after creation|**Immutable**|
|Bucket association after creation|**Immutable** (delete + recreate to change)|
|Protocol|**HTTPS only** (HTTP is auto-redirected)|
|Signing|**AWS SigV4 only**|
|Anonymous access|**Not supported**|
|Addressing style|**Virtual-host style only**|
|Can be an S3 Replication **destination**|❌ No|
|Management API calls (`PutAccessPoint`, etc.)|**No cross-account support**|

---

## 8. Exam Cheat Sheet

|Keyword / scenario|Answer|
|---|---|
|"Bucket policy is too large / complex / hitting size limits"|**S3 Access Points**|
|"Many teams need different access to the same bucket"|**S3 Access Points**|
|"Restrict bucket access to one specific VPC"|Access point with **VPC origin** + VPC endpoint with policy referencing both bucket and access point ARNs|
|Each access point has a unique...|**DNS endpoint name**|
|Access point policy is functionally equivalent to...|A **bucket policy**|
|Access point policy grants access — does it work without bucket policy changes?|**No** — bucket policy must also allow it (matching permissions or delegation)|
|AWS-recommended pattern for delegation|Bucket policy uses `s3:DataAccessPointAccount` to allow `*` to any access point in the account|
|CLI command|`aws s3control create-access-point`|
|Can restrict requests by...|**Prefix, object tags, actions**|
|Network origins|**Internet** or **VPC** (immutable after creation)|
|Max per account per Region|**10,000**|
|Policy size limit|**20 KB**|
|Can be used as S3 Replication destination?|**No**|
|Cross-account access point — who grants the underlying permission?|The **bucket owner** (via bucket policy)|
|SCP to force VPC-only access points|`Deny s3:CreateAccessPoint` when `s3:AccessPointNetworkOrigin != VPC`|
|Region rule|Access point and bucket must be in the **same Region**|

---

## 9. Key Takeaways

- Access points are a **manageability layer**, not a new permissions system. The bucket policy is still the ultimate authority — they work like DNS CNAMEs in spirit.
- Each access point is independently configured: **endpoint + policy + network origin + BPA settings**, most of which are **immutable after creation**.
- Use **delegation** (`s3:DataAccessPointAccount` in the bucket policy) and put all real restrictions in access point policies — that's the architectural pattern worth memorizing.
- VPC-origin access points + VPC endpoint policies + a deny-non-VPC bucket policy give you end-to-end _"this bucket is only reachable from this VPC via this access point"_ enforcement.
- The matching **SCP guardrail** (`Deny s3:CreateAccessPoint` unless VPC-origin) is the org-level enforcement mechanism — useful for Control Tower-managed environments.
- Watch the **replication caveat**: access points can't be replication destinations, so don't lock down a bucket with a VPC-only policy if it's a replication target.

---

## 10. Sources

- [Creating an access point — AWS S3 User Guide](https://docs.aws.amazon.com/AmazonS3/latest/userguide/creating-access-points.html)
- [Configuring IAM policies for using access points](https://docs.aws.amazon.com/AmazonS3/latest/userguide/access-points-policies.html)
- [Creating access points restricted to a VPC](https://docs.aws.amazon.com/AmazonS3/latest/userguide/access-points-vpc.html)
- [Access points naming rules, restrictions, and limitations](https://docs.aws.amazon.com/AmazonS3/latest/userguide/access-points-restrictions-limitations-naming-rules.html)
- Adrian Cantrill — S3 Access Points lesson ([learn.cantrill.io](https://learn.cantrill.io/))