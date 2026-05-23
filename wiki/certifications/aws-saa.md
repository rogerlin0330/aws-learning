---
title: AWS Solutions Architect Associate — Exam Notes
type: certification
tags: [iam, vpc, networking, ec2, route53, organizations, cost, s3, cloudwatch, multipart, transfer-acceleration, kms, encryption, sse, sse-kms, sse-s3, sse-c, cse, storage-classes, glacier, intelligent-tiering, lifecycle, replication, crr, srr, rtc, presigned-urls, s3-select, glacier-select, athena]
certs: [aws-saa]
updated: 2026-05-23
---

# AWS Solutions Architect Associate (SAA-C03)

A running collection of exam tips, gotchas, and comparison tables. Deep-dive pages: [IAM](../services/iam.md) · [VPC](../services/vpc.md) · [EC2](../services/ec2.md) · [S3](../services/s3.md) · [Route 53](../services/route53.md) · [Organizations](../services/organizations.md) · [CloudWatch](../services/cloudwatch.md) · [KMS](../services/kms.md) · [Networking](../concepts/networking.md) · [Identity & Access](../concepts/identity-and-access.md) · [Resource vs Identity Policies](../concepts/resource-vs-identity-policies.md) · [DNS](../concepts/dns.md) · [TLS/HTTPS](../concepts/tls-https.md) · [Cost Optimization](../patterns/cost-optimization.md).

## IAM

See [IAM](../services/iam.md) and [Identity and Access Concepts](../concepts/identity-and-access.md) for full details.

- **Deny overrides Allow**: explicit Deny in any policy wins over any Allow — across all policy sources
- Managed policies > inline policies: prefer managed for reusability and lower overhead
- IAM Users = long-term credentials; [IAM Roles](../services/iam.md#iam-roles) = temporary credentials — always use roles for [EC2](../services/ec2.md), Lambda, cross-account access
- IAM is **global** — not region-specific
- Hard limit: **5,000 IAM users per account** — for larger user bases use roles + federation
- **IAM Groups** are containers, **not identities**: no credentials, no nesting, cannot be referenced by [resource policies](../services/iam.md#policy-evaluation--priority) (only users/roles can). Limits: **10 groups per user**, default **300 groups per account**, no "all users" built-in group. Effective permissions = user policies ∪ all group policies (same Deny-wins evaluation)

### IAM Roles

See [IAM Roles](../services/iam.md#iam-roles) for full details.

- A role has **two policies**: **trust policy** (who can assume it) and **permissions policy** (what it can do)
- A principal denied by the trust policy never gets credentials — `sts:AssumeRole` fails before permissions are checked
- Assuming a role returns a **triplet**: Access Key ID + Secret Access Key + **Session Token** — the session token is the marker of temporary credentials
- Permissions changes apply to **in-flight sessions immediately** — STS does not re-issue credentials, AWS re-checks the permissions policy on every API call
- Roles can be referenced in **resource policies** (e.g. S3 bucket policy granting a role ARN) — both the role policy and the resource policy are evaluated together
- **External identities cannot directly access AWS** — Active Directory users, Google/Facebook users, and users in other AWS accounts must all assume a role first

#### When the answer is "use a role"

| Scenario | Mechanism |
| --- | --- |
| AWS service acting on your behalf (Lambda, EC2 → S3) | **Service role** — trust policy trusts the service principal (e.g. `lambda.amazonaws.com`) |
| Emergency / rare elevated access | **Break-glass role** — assume only on demand, CloudTrail logged |
| Existing corporate IdP (Active Directory, Okta) | **SAML federation** — `sts:AssumeRoleWithSAML` |
| Mobile/web app with millions of users (Google/Facebook login) | **Web identity federation** — `sts:AssumeRoleWithWebIdentity` |
| Access to another AWS account / AWS Organizations | **Cross-account role** — `sts:AssumeRole` |

> [!tip] Exam Tip
> Pattern-match: "company has existing AD" or "millions of mobile users" → IAM **role + federation**, not IAM users. "Lambda needs to write to S3" → **execution role**, never embedded access keys. "Two AWS accounts" → cross-account **role assumption**, not a duplicate IAM user.

> [!warning] Gotcha
> Never bake long-term IAM user access keys into Lambda code, EC2 user-data, or container images. Use a role — exam questions describing "access keys in code" are usually pointing at this anti-pattern.

## Resource Policies vs Identity Policies

See [Resource Policies vs Identity Policies](../concepts/resource-vs-identity-policies.md) for full details.

- **Identity policy** attaches to a user/group/role; **no `Principal` field** (identity is implicit). **Resource policy** attaches to a resource (bucket, KMS key, queue, …) and **must** have a `Principal`
- **Only resource policies can grant**: (a) **cross-account** access, (b) **anonymous / public** access (`Principal: "*"`)
- **Same-account access**: Allow in **either** the identity policy **or** the resource policy is sufficient (it's a **union**, modulo Deny)
- **Cross-account access**: **both sides must Allow** (genuine **intersection**) — bucket policy names the principal *and* their home account's admin grants them the action via identity policy
- **IAM Groups cannot be principals** in resource policies — only IAM users and roles
- An IAM **role's trust policy** is a resource policy in disguise (lives on the role, names who can assume it)

> [!tip] Exam Tip
> Pattern-match by what's controlled: *"What can this user do?"* → identity policy. *"Who can access this bucket / key / queue?"* → resource policy. *"Grant anonymous read"* → resource policy (only option). *"Grant another AWS account"* → resource policy on this side + identity policy on theirs.

> [!warning] Gotcha
> Same-account behaviour is NOT an intersection — that's only true cross-account. A bucket policy in your own account that grants `s3:GetObject` to your IAM user is sufficient even if the user's identity policy says nothing about that bucket.

## S3 Security

See [S3](../services/s3.md) for full details.

- S3 buckets are **private by default** — only the **account root** user of the owning account has initial access
- **Bucket policies** (resource policies) are how S3 implements: cross-account access, anonymous access, and conditional access — none of which identity policies can fully express
- Bucket ARNs: `arn:aws:s3:::name` (bucket itself, for list ops) vs `arn:aws:s3:::name/*` (objects). Most real policies need both
- **ACLs are legacy** — 5 permission types (READ, WRITE, READ_ACP, WRITE_ACP, FULL_CONTROL), no conditions, no Deny semantics. New buckets default to **ACLs disabled** (Object Ownership = Bucket owner enforced)
- **Block Public Access** is an **override**, not a grant: it blocks *anonymous* exposure regardless of bucket policy or ACL. Authenticated cross-account access is unaffected. New buckets get all four switches on by default
- **Explicit Deny wins** — bucket policy `Deny + Condition` is the standard way to enforce IP allow-lists or MFA-required prefixes

### Common condition patterns

| Goal | Condition |
| --- | --- |
| Lock to corporate IP | `"NotIpAddress": {"aws:SourceIp": "1.3.3.7/32"}` with `Deny` |
| Require MFA on a prefix | `"Null": {"aws:MultiFactorAuthAge": true}` with `Deny` (matches *no MFA*) |
| Restrict to a VPC endpoint | `"StringNotEquals": {"aws:SourceVpce": "vpce-…"}` with `Deny` |

> [!tip] Exam Tip
> *"Bucket policy says public-read but anonymous requests get 403"* → **Block Public Access** is on. *"Want to keep accidental public-policy mistakes from leaking data"* → leave Block Public Access on at the **account** level. *"Need a public website bucket"* → must explicitly disable the relevant Block Public Access switches.

> [!warning] Gotcha
> Enabling ACLs on a new bucket is almost always the **wrong** exam answer. Modern S3 design uses bucket policies + IAM + Object Ownership (Bucket owner enforced) and skips ACLs entirely.

### S3 Pre-Signed URLs

See [S3 — Pre-Signed URLs](../services/s3.md#pre-signed-urls) for full details.

- A pre-signed URL gives one external caller **time-limited, object-scoped** access to one S3 object using the **generator's** credentials — no AWS identity needed for the caller
- Inputs to generate: bucket, key, expiry, HTTP method (**`GET`** or **`PUT`**) + generator's credentials. S3 returns a self-contained URL
- When used, S3 treats the request as if it came from the **generating identity** — same effective permissions, until expiry
- Supports **`GET`** (download) and **`PUT`** (upload). Direct browser ↔ S3 in both directions is the canonical pattern
- The bucket stays **private** — no bucket-policy or [Block Public Access](../services/s3.md#block-public-access) change required
- **Generate from an IAM user, not a role** — STS role credentials may expire before the URL does, breaking the URL even though its own expiry hasn't been reached
- **You can generate a URL for an object you don't have access to** — generation succeeds; the URL just won't work when used
- **Permissions are evaluated at use time** — current generator permissions, not at generation time. Revoking the generator invalidates outstanding URLs

#### AccessDenied diagnosis

| Symptom | Cause |
| --- | --- |
| 403 immediately, never worked | Generator never had permission to the object |
| Worked briefly, now 403 | Generator was a role; **STS credentials expired** before the URL did — switch to an IAM user |
| Worked yesterday, now 403, identity unchanged | URL has **expired** past its encoded timestamp |
| Worked yesterday, now 403, URL still in window | Generator's permissions were **revoked** or modified |

#### Pre-signed URL vs bucket policy

| Goal | Tool |
| --- | --- |
| One external user, one object, short-lived | **Pre-signed URL** |
| Many users, one bucket, persistent | [Bucket policy](../services/s3.md#bucket-policies) |
| Public read of many objects | [Bucket policy](../services/s3.md#bucket-policies) with `Principal: "*"` |
| Direct browser-to-S3 upload from a web app | **Pre-signed URL** signed for `PUT` |
| Offload large media downloads from app server | **Pre-signed URL** signed for `GET` |

> [!tip] Exam Tip
> Pattern-match: *"time-limited access to one object for someone without AWS creds"* → **pre-signed URL**. *"users upload directly to S3 from the browser"* → pre-signed URL signed for **`PUT`**. *"app server's bandwidth is saturated serving large media"* → pre-signed URL for **`GET`**, browser fetches directly from S3. *"pre-signed URL works briefly then 403s"* → role-as-generator + STS expiry — **use an IAM user**. *"AccessDenied on a freshly-issued URL"* → generator never had the underlying permission.

> [!warning] Gotcha
> The most common exam trap: a pre-signed URL is generated by an assumed role, works for a while, then starts returning 403 even though its own expiry hasn't elapsed. The role's STS session expired. The URL inherits the **shorter** of the two expirations. Use an IAM user for the generating identity in production patterns.

### S3 Versioning & MFA Delete

See [S3 — Versioning and MFA Delete](../services/s3.md#versioning-and-mfa-delete) for full details.

- Versioning is configured at the **bucket** level; new buckets start **Disabled**
- State machine: Disabled → Enabled → Suspended ↔ Enabled. **Enabled → Disabled is impossible** — once on, the only "off-ish" state is Suspended (existing versions are kept)
- Each write to a versioned bucket creates a new **version ID**; a `GetObject` without a `versionId` returns the current (latest) version
- `DELETE object` without a `versionId` adds a **delete marker** (soft delete — reversible by deleting the marker). `DELETE object?versionId=…` permanently removes that version (irreversible)
- **Storage cost trap**: every version is billed at full size — 5 × 5 GB = **25 GB**. Suspending versioning **does not** delete existing versions; use a lifecycle policy on noncurrent versions
- **MFA Delete** requires versioning to be enabled and gates exactly two operations: **changing the versioning state** and **permanently deleting a specific version**. Ordinary (soft) deletes and reads/writes are unaffected
- MFA Delete can only be **enabled/disabled by the bucket owner's root user**, via CLI/API (not the console). API calls must carry the MFA device serial + current code

> [!tip] Exam Tip
> Pattern-match: *"protect against accidental object deletion"* → **versioning** (sufficient on its own). *"protect against accidental **and** malicious permanent deletion, including by a compromised IAM admin"* → **versioning + MFA Delete**. *"object deleted but the team wants it back"* → if versioning was on, delete the delete marker (or `GetObject` with the previous `versionId`); if versioning was off, the data is gone.

> [!warning] Gotcha
> If a question describes "disabling versioning to reclaim storage cost", that's a trap — there's no Disable state, and Suspend doesn't drop existing versions. The right answer is lifecycle-expire noncurrent versions, or recreate the bucket.

### S3 Upload Performance — Multipart + Transfer Acceleration

See [S3 — Upload Performance](../services/s3.md#upload-performance) for full details.

- **Single `PutObject` is capped at 5 GB** — anything larger **requires** multipart upload (not optional)
- AWS recommends multipart for objects **≥ 100 MB**; the SDK / CLI transfer manager auto-picks multipart above ~8 MB, so this is usually handled for you
- Multipart parts: **5 MiB–5 GiB each** (last part can be smaller), **max 10,000 parts** → max object size ~5 TB
- Failed multipart parts **retry individually** — single PUT failures restart from byte 0
- **Incomplete multipart uploads still incur storage costs.** Standard fix: lifecycle rule with **`AbortIncompleteMultipartUpload`** after N days
- **Transfer Acceleration** uses **CloudFront edge locations** + the **AWS global backbone** to bypass the public-internet long-haul. Bucket-level opt-in (off by default), separate endpoint `<bucket>.s3-accelerate.amazonaws.com`
- TA bucket-name requirement: **DNS-compatible, no periods** (the dotted hostname would break TLS hostname matching)
- TA costs **extra per GB** transferred; AWS provides a Speed Comparison tool to verify benefit before enabling
- Multipart and TA are **orthogonal and combinable** — multipart parallelizes the bytes, TA shortens the network path

| Goal | Mechanism |
| --- | --- |
| Upload a 50 GB file (reliability, throughput) | **Multipart upload** (required — > 5 GB) |
| Globally distributed clients uploading to one bucket | **Transfer Acceleration** |
| Large files **and** geographically far clients | **Both** — multipart over the TA endpoint |
| Bulk transfer of petabytes from on-prem | **Snowball / DataSync** (beyond TA's domain) |

> [!tip] Exam Tip
> Pattern-match: *"large object, unreliable connection, want to resume on failure"* → **multipart upload**. *"users on multiple continents uploading to one bucket"* → **Transfer Acceleration**. *"file > 5 GB"* → multipart is **required**, not a choice. *"reduce orphaned-upload storage charges"* → lifecycle rule with `AbortIncompleteMultipartUpload`.

> [!warning] Gotcha
> Transfer Acceleration cannot be enabled on a bucket whose name contains **a period** (`.`). If a question mentions a bucket like `my.company.data` and asks why TA isn't working, the answer is **the bucket name must be DNS-compatible and dotless**.

### S3 Storage Classes

See [S3 — Storage Classes](../services/s3.md#storage-classes) for full details.

- **Default to S3 Standard** — only move off it when the question explicitly mentions infrequent access, archival, or a replaceable/regenerable copy
- Every class **except One Zone-IA** is designed for **11 nines durability** across **≥ 3 AZs**. One Zone-IA's 11 nines is **conditional on the single AZ not failing**
- **Durability ≠ availability.** Standard = 11 nines durability + 4 nines (99.99%) availability. Standard-IA availability drops to 99.9%; One Zone-IA to 99.5%
- **S3 Standard is the only class with no retrieval fee, no minimum size, and no minimum duration** — every other class adds at least one
- **Public access requires hot tiers**: Standard, IA, One Zone-IA, GIR, accessible IT tiers. Glacier Flexible and Glacier Deep Archive **cannot be made public** — there's no way to serve an object when reading it requires a multi-hour job
- **Restoring a Glacier object** stages a temporary copy in Standard-IA storage for the requested availability window — the **original Glacier object isn't moved**. Costs: retrieval fee + temporary IA storage. To permanently un-Glacier, change class via `CopyObject` after restore

#### Class-by-class anchor table

| Class | First-byte latency | Min. duration | Min. size | Retrieval fee | Key trade-off |
| --- | --- | --- | --- | --- | --- |
| **Standard** | ms | None | None | None | The default — baseline cost |
| **Standard-IA** | ms | 30 days | 128 KB | $0.01/GB | ~½× storage cost; retrieval + minimums |
| **One Zone-IA** | ms | 30 days | 128 KB | $0.01/GB | ~20% cheaper than IA; **single AZ** |
| **Glacier Instant Retrieval** | ms | 90 days | 128 KB | $0.03/GB | Cheap storage + instant access; per-GET fee |
| **Glacier Flexible Retrieval** | minutes–hours | 90 days | 40 KB | Expedited $0.03 / Std $0.01 / **Bulk free** | Archival; retrieval job required |
| **Glacier Deep Archive** | hours | 180 days | 40 KB | Std $0.02 / Bulk $0.0025 | Cheapest tier; **no Expedited** option |
| **Intelligent-Tiering** | varies by tier | None (90d archive-instant) | None | None (monitoring fee) | Auto-moves objects; useful only for **unknown** access patterns |

#### Glacier retrieval-tier timing

| Tier | Restore time | Per-GB fee |
| --- | --- | --- |
| Glacier Flexible — Expedited | 1–5 min | $0.03/GB |
| Glacier Flexible — Standard | 3–5 hr | $0.01/GB |
| Glacier Flexible — Bulk | 5–12 hr | **Free** |
| Deep Archive — Standard | within 12 hr | $0.02/GB |
| Deep Archive — Bulk | up to 48 hr | $0.0025/GB |

#### Lifecycle Configuration

See [S3 — Lifecycle Configuration](../services/s3.md#lifecycle-configuration) for full details.

- **Two action types**: **Transition** (change storage class) and **Expiration** (delete object or version). Rule scope: whole bucket, prefix filter, tag filter, object size filter, or — on versioned buckets — object versions
- **Lifecycle is time-based, not access-based.** *"Move when frequently/rarely accessed"* → that's **Intelligent-Tiering**, not lifecycle
- **Transitions are one-way down the cost ladder** — to move *back up* to a warmer class, use `CopyObject`, not a lifecycle rule
- **30-day Standard minimum** between upload and any lifecycle transition to Standard-IA or One Zone-IA (direct PUT into IA at upload time is fine; only the lifecycle path is gated). **Same rule applies to noncurrent versions** — they must be at least 30 days noncurrent before transitioning to Standard-IA / One Zone-IA
- **Single-rule minimum-duration gap**: within one rule, consecutive transitions must respect the **destination class's minimum storage duration**. The IA-to-Glacier case (30-day IA minimum) is the most common, but the rule generalizes — e.g. a rule that says GIR @ 4 days then Deep Archive @ 20 days is rejected (GIR's 90-day minimum means Deep Archive can't fire before day 94). **Workaround: use two separate rules** — the constraint doesn't apply across rules (you still pay the destination's minimum-duration charges)
- **Destination-class minimums apply** — a lifecycle move into Standard-IA still triggers IA's 30-day minimum charge
- **Objects < 128 KB don't transition by default** (since September 2024). Override with an `ObjectSizeGreaterThan` filter on the rule, or set the `x-amz-transition-default-minimum-object-size` header on `PutBucketLifecycleConfiguration`. Pre-Sept-2024 configurations retain the old behavior until edited
- **Pending / Failed replication objects are skipped** by lifecycle on versioning-enabled / versioning-suspended buckets — fix replication before expecting transitions to fire
- **Lifecycle is the same engine** behind `AbortIncompleteMultipartUpload`, noncurrent-version expiration, and storage-class transitions — different actions, same machinery

##### The transition waterfall

Every class can transition down to a colder class, with **one exception**: **One Zone-IA cannot transition to Glacier Instant Retrieval**. Every other source class (Standard, Standard-IA, Intelligent-Tiering, even Glacier Instant Retrieval itself) can target GIR. Memorize this single asymmetry.

| From | Can transition to |
| --- | --- |
| Standard | All other classes |
| Standard-IA | Intelligent-Tiering, One Zone-IA, GIR, Glacier Flexible, Glacier Deep Archive |
| Intelligent-Tiering | Varies by IT access tier — narrows as the source tier gets colder (Frequent/Infrequent → One Zone-IA + Glacier classes; Archive Instant Access → GIR + Glacier classes only; Archive Access → Glacier Flexible/Deep only; Deep Archive Access → Deep Archive only) |
| One Zone-IA | Glacier Flexible, Glacier Deep Archive ⚠️ (**not** GIR) |
| Glacier Instant Retrieval | Glacier Flexible, Glacier Deep Archive |
| Glacier Flexible Retrieval | Glacier Deep Archive |
| Glacier Deep Archive | (terminal) |

> [!tip] Exam Tip
> Pattern-match: *"transition objects to a cheaper tier automatically as they age"* → **lifecycle transition**. *"delete objects after N days"* → **lifecycle expiration**. *"move data based on access pattern"* → **Intelligent-Tiering**, not lifecycle. *"Standard → Glacier on day 5"* → blocked by the **30-day Standard minimum**. *"single rule: GIR day 4 → Deep Archive day 20"* → blocked by the **single-rule minimum-duration gap** (GIR's 90-day minimum); the Deep Archive transition must be on day 94 or later, or split into two rules. *"One Zone-IA → Glacier Instant Retrieval"* → **not allowed**; route via a hot tier with `CopyObject` first. *"my lifecycle rule isn't moving any of my small log files"* → since Sept 2024, **objects < 128 KB don't transition by default** — add `ObjectSizeGreaterThan`.

> [!warning] Gotcha — small objects don't transition by default, and archive overhead makes them worse
> Two compounding factors make small-object archival a trap: **(1)** since September 2024, lifecycle skips objects < 128 KB by default unless you explicitly override the size filter, and **(2)** every object archived to Glacier Flexible / Deep Archive carries **8 KB at Standard rates + 32 KB at Glacier rates** of fixed metadata overhead. For tiny files the overhead alone can exceed the object itself — aggregate into larger objects (tarballs, columnar blobs) before archiving.

> [!tip] Exam Tip
> Pattern-match by **access frequency + retrieval tolerance**:
>
> - *"frequently accessed, important"* → **Standard**
> - *"accessed monthly, important"* → **Standard-IA**
> - *"replica / regenerable / derived data"* → **One Zone-IA**
> - *"accessed quarterly but must be instant"* → **Glacier Instant Retrieval**
> - *"archived, hours OK to retrieve"* → **Glacier Flexible Retrieval**
> - *"compliance archive, up-to-48 hours OK"* → **Glacier Deep Archive**
> - *"access pattern unknown or shifting over time"* → **Intelligent-Tiering** (and *only* then — for static patterns, the monitoring fee eats the savings)

> [!warning] Gotcha — "Glacier" doesn't mean public
> Glacier Flexible Retrieval and Glacier Deep Archive **cannot be made publicly accessible** and don't work with static website hosting. If a question describes *"archival storage that must remain publicly downloadable"*, the answer is **Glacier Instant Retrieval**, not Flexible or Deep.

> [!warning] Gotcha — One Zone-IA's 11 nines is conditional
> Marketing says 11 nines across all classes. For One Zone-IA, that math **assumes the single AZ doesn't fail** — an AZ-level loss event takes the data with it. Never use One Zone-IA for the only copy of critical data.

> [!warning] Gotcha — Intelligent-Tiering isn't free
> IT replaces retrieval fees with a **monitoring/automation fee per 1,000 objects**. For static, predictable access patterns, IT costs *more* than the right manual class. The right answer is IT only when the prompt explicitly says access patterns are **unknown or changing**.

### S3 Replication

See [S3 — Replication](../services/s3.md#replication) for full details.

- **CRR** (Cross-Region) vs **SRR** (Same-Region) differ only in *where* the destination lives. Both work **same-account** or **cross-account**
- Replication is configured on the **source bucket** + an IAM role (S3 service trust policy; permissions on source `Get*` + destination `Replicate*`)
- **Same-account**: IAM role is enough. **Cross-account**: also need a **bucket policy on the destination** allowing the source-account role
- **Versioning is required on both buckets** — non-negotiable
- **Not retroactive** — only new objects from configuration time onward. To backfill existing objects, use **S3 Batch Replication** (a separate, explicit job)
- **One-way by default** — bidirectional requires two configurations (one on each bucket)
- **Replication Time Control (RTC)**: adds a **15-minute SLA** + per-object replication metrics. *"15 minutes"* → **RTC**
- **Object ownership**: by default replicated objects stay owned by the **source account** — in cross-account setups, **override** so the destination account owns them, otherwise the destination's IAM users can't read their own bucket
- **Storage class**: same as source by default; can be overridden (often **One Zone-IA** for the secondary copy)

#### What is NOT replicated

| Not replicated | Why / detail |
| --- | --- |
| **Pre-existing objects** | Configure **S3 Batch Replication** to backfill |
| **Lifecycle transitions / expirations** | System events aren't replicated; user-driven events are. Configure the same lifecycle rule on **both** buckets if you want symmetric behaviour |
| **Glacier Flexible Retrieval and Deep Archive objects** | Treat those classes as a separate product. (Glacier *Instant* Retrieval **is** replicable) |
| **Delete markers** (by default) | Can be opted in per rule |
| **Versioned (permanent) deletes** | **Never** replicated, by design — replication is for survival, not synchronized destruction |
| **Objects not owned by the source account** | Replication requires source-account ownership. Cross-account writes with third-party ownership are silently skipped |

#### Encryption support

| Method | Replication |
| --- | --- |
| Unencrypted | ✅ Default |
| **SSE-S3** | ✅ Default |
| **SSE-KMS** | ✅ Requires extra config — KMS key permissions on both sides, destination key named in the replication rule |
| **SSE-C** | ⚠️ Only recently supported — historically blocked |

#### Use-case lookup

| Scenario | CRR or SRR? |
| --- | --- |
| DR / survive region failure | **CRR** |
| Read latency reduction for global users | **CRR** |
| Centralize logs from many buckets | **SRR** |
| Prod ↔ test data sync | **SRR** |
| Sovereignty-bound resilience (data must stay in region, isolated audit account) | **SRR** cross-account |

> [!tip] Exam Tip
> Pattern-match: *"replicate to another region for DR"* → **CRR**. *"15-minute SLA"* → **RTC**. *"backfill existing objects after enabling replication"* → **S3 Batch Replication**. *"destination account can't read replicated objects in its own bucket"* → **object-ownership override** wasn't set on the rule. *"replication doesn't propagate deletes"* → **default behaviour** — delete markers can be opted in; permanent versioned deletes never replicate. *"lifecycle expirations don't propagate"* → by design — system events aren't replicated.

> [!warning] Gotcha — Versioning prerequisite vs the one-way door
> Replication requires versioning on both buckets. Once enabled, [versioning can only be Suspended, never Disabled](../services/s3.md#state-machine--one-way-door). Turning on replication is therefore a partial one-way door for the destination bucket — you can stop replication later, but the destination is permanently a versioned bucket.

> [!warning] Gotcha — Cross-account ownership
> The most common cross-account exam trap: replication succeeds, the bytes arrive in the destination, but the destination account's IAM users get **403** trying to read them — because the source account still owns the objects. The fix is the **"Change object ownership to destination bucket owner"** option in the replication rule, set at configuration time.

### S3 Select & Glacier Select

See [S3 — Server-Side Filtering](../services/s3.md#server-side-filtering--s3-select--glacier-select) for full details.

- **S3 Select** and **Glacier Select** push a SQL-like filter **into the storage service** so the application only retrieves the rows/columns it actually needs — not the whole object
- The architectural win: filtering moves **from the application to S3**, so the network only carries the bytes the app uses. Up to **400% faster**, up to **80% cheaper** on data transfer
- **Supported formats**: **CSV**, **JSON**, **Parquet** (S3 Select); **BZIP2** compression for CSV and JSON
- **One object at a time** — no joins, no aggregations across objects. Multi-object queries belong on **Athena**, not S3 Select
- **S3 Select** = synchronous, against any hot tier (Standard / IA / One Zone-IA / GIR), millisecond latency
- **Glacier Select** = a **retrieval job** against Glacier Flexible / Deep Archive — inherits the source class's [retrieval tier](../services/s3.md#retrieval-tiers) (Expedited / Standard / Bulk). Not an instant query

#### Decision lookup

| Scenario | Answer |
| --- | --- |
| "Retrieve only specific rows/columns from a large CSV/JSON/Parquet object" | **S3 Select** |
| Same scenario but the object is in Glacier Flexible / Deep Archive | **Glacier Select** (still a retrieval job) |
| "Query across many S3 objects, with joins or aggregations" | **Athena**, not S3 Select |
| "App downloads the whole object then filters in memory" | The **anti-pattern** S3 Select fixes |
| Source object is an image / video / binary archive | **Plain GetObject** — S3 Select doesn't apply |

> [!tip] Exam Tip
> Pattern-match: *"reduce data transferred from S3 / filter at the source"* → **S3 Select**. *"SQL-like expression on one object"* → **S3 Select**. *"400% faster / 80% cheaper"* → S3 Select / Glacier Select. *"object lives in Glacier and we need to filter on restore"* → **Glacier Select** (still a retrieval job — Expedited / Standard / Bulk choice still applies). *"query across thousands of files / joins / aggregations"* → **Athena**.

> [!warning] Gotcha — S3 Select is not a query engine
> The exam likes to dangle S3 Select as a tempting answer to multi-object query scenarios. Don't bite. **One object per call** is the hard boundary. Joins, aggregations across objects, schema catalog → **Athena**. Format outside CSV / JSON / Parquet → **not S3 Select** either.

## VPC & Networking

See [VPC](../services/vpc.md) and [Networking Concepts](../concepts/networking.md) for full details.

### Firewall Rules

- Both [NACL](../services/vpc.md#network-acl-nacl) and [Security Group](../services/vpc.md#security-groups-sg) operate at **Layer 3/4** — not Layer 7; they filter by IP, protocol, and port only
- ICMP rules = Layer 3 only (no ports); TCP/UDP rules = Layer 4
- Layer 7 (HTTP content, headers, URLs) is handled by **WAF**, not NACL or SG
- **NACL** = stateless, attaches to **subnets** (L3 constructs), supports explicit Deny → use to block bad IPs
- **Security Group** = stateful, attaches to **ENIs** (L3/4 constructs), no explicit Deny → use for fine-grained per-resource control
- Because NACLs are stateless: always allow **ephemeral ports (1024–65535)** for return traffic
- The real distinction between NACL and SG is **stateful vs stateless** + **attachment point**, not OSI layer

### IGW vs NAT GW

| | IGW | NAT GW |
| --- | --- | --- |
| Direction | Bidirectional (public subnet) | Outbound only (private subnet) |
| IP mapping | 1:1 (private ↔ Elastic IP) | Many-to-one |
| Resilience | Regional | Zonal or Regional (see below) |
| IPv6 support | Yes | No |

### Zonal vs Regional NAT GW (availability mode — as of Nov 2025)

| | Zonal NAT GW | Regional NAT GW (RNAT) |
| --- | --- | --- |
| Scope | Single AZ | VPC-level |
| Public subnet required? | Yes | No |
| High availability | Manual — one per AZ, per-AZ routes | Automatic — single ID, auto-expands |
| Private NAT support | Yes | No |
| Use when | Private NAT; strict per-AZ IP control | Most public egress use cases |

> [!tip] Exam Tip
> Pre-Nov 2025 courses only cover **zonal** NAT GW. For region resilience with zonal: deploy one per AZ, configure each AZ's route table to its local NAT GW. With **regional** NAT GW, a single ID covers all AZs automatically — no public subnet needed.

### NAT GW vs NAT Instance

- **NAT GW**: AWS-managed, no SG support, charged by duration + data
- **NAT Instance**: self-managed, supports SG, cheaper at high data volumes — must disable **source/destination checks**

### IPv4 vs IPv6

- Public IPv4 is **not configured in the OS** — only visible at the IGW level
- IPv6 is configured directly in the OS (all IPv6 is publicly routable in AWS)
- IPv6 outbound-only: use **Egress-Only IGW**, not NAT GW (NAT GW doesn't support IPv6)

### VPC Design

- Default VPC CIDR: `172.31.0.0/16` — avoid in production
- 5 reserved IPs per subnet: network, router (+1), DNS (+2), reserved (+3), broadcast (last)
- `enableDnsHostnames` = false by default on custom VPCs; set to true to get public DNS names

## DNS and Route 53

See [Route 53](../services/route53.md) and [DNS Concepts](../concepts/dns.md) for full details.

- Route 53 is an **authoritative DNS** service — **globally resilient**, name servers span multiple regions
- Each hosted zone gets **4 public name servers**; NS records at the registrar must point to them
- **Alias record**: Route 53-specific; works at the zone apex (`example.com`); CNAME cannot; **no query charge** to AWS resources
- CNAME only works for **subdomains** — never for the root domain
- Alias type must **match the target resource's record type** (e.g. ELB returns A → create A Alias)
- **Private Hosted Zone**: resolves only within a VPC; requires `enableDnsSupport` and `enableDnsHostnames` on the VPC
- **Cross-account private hosted zone association**: CLI/API only — not available in console
- **Split-View DNS**: same domain name, public + private zone → different answers based on query origin
- TTL = how long resolvers cache a record; **lower TTL before migrations**, raise after

### Route 53 Routing Policies

| Policy | When to use |
| --- | --- |
| Simple | Single endpoint, no health checks |
| Weighted | A/B testing, gradual traffic shifts |
| Latency | Route to lowest-latency AWS region |
| Failover | Active/passive HA; requires health check on primary |
| Geolocation | Route by user's country/continent |
| Multi-value | Return multiple IPs; basic load distribution |

> [!tip] Exam Tip
> **Failover routing** requires a Route 53 **health check** on the primary record. If the primary fails the health check, Route 53 automatically returns the secondary record.

### Route 53 Health Checks

See [Route 53 Health Checks](../services/route53.md#health-checks) for full details.

- Health checks are **separate objects** from DNS records — created under Route 53 → Health Checks, then attached to records
- Can monitor **any public endpoint** by IP or domain — not just AWS resources
- Three categories: **Endpoint** (direct probe), **CloudWatch Alarm** (any metric), **Calculated** (boolean of other checks)
- Endpoint probe types: **TCP** (handshake in 10 s), **HTTP/HTTPS** (status 200–399), **HTTP/HTTPS + string match** (matches first 5,120 bytes of body)
- **String matching is the most accurate** application health signal — catches "200 OK but app is broken" cases
- Default interval **30 s**; fast mode **10 s** at extra cost
- **18 % rule**: if more than 18 % of distributed checkers report healthy → overall healthy; otherwise unhealthy
- Firewalls ([SG](../services/vpc.md#security-groups-sg)/[NACL](../services/vpc.md#network-acl-nacl)) must **allow Route 53 health checker IP ranges** or checks will false-fail
- Unhealthy records are normally **excluded from DNS responses**
- **SNS integration** for alerting on state change — independent of routing behaviour

> [!tip] Exam Tip
> When a question describes verifying that an app is **actually working** (not just reachable), the answer is **HTTP/HTTPS with string matching**. TCP only proves the port is open; plain HTTP only proves the web server responds; string matching proves the response body is correct.

> [!warning] Gotcha
> Health checks live **outside hosted zones**. Look for them under Route 53 → Health Checks in the console, not inside the record set.

## AWS Organizations

See [Organizations](../services/organizations.md) for full details.

- **Management account** (formerly master) creates the org and is the **payer** for all member accounts
- **Member accounts** lose their individual billing — single consolidated bill
- **OUs** allow nested grouping (root → OU → sub-OU → account); SCPs attach at root/OU/account and inherit downward
- **Volume discounts + RI/SP benefits pool org-wide by default** — bought-in-account-A floats to account-B's matching usage
- New accounts created inside the org get an automatic **`OrganizationAccountAccessRole`** trusting the management account → one-click Switch Role

### SCPs

See [Service Control Policies](../services/organizations.md#service-control-policies-scps) for full details.

- **SCPs are guardrails, not grants** — they set the **maximum** permissions; a principal still needs an IAM identity policy that *allows* the action
- **Effective permissions = SCP allows ∩ IAM identity policy allows** — the intersection. Anything outside the overlap is denied
- **SCPs do NOT apply to the management account** — even `Deny *` is ignored. Keep workloads in member accounts so SCPs actually take effect
- SCPs **can** indirectly restrict the **account root user** — one of the few mechanisms in AWS that bounds root
- Default model is **deny list**: AWS attaches a `FullAWSAccess` SCP everywhere; you add explicit `Deny` statements for what you don't want. Low overhead, new AWS services are automatically permitted
- **Allow list** model = remove `FullAWSAccess`, add explicit `Allow`s. More secure, much higher admin overhead — every service must be added

> [!tip] Exam Tip
> "Many accounts, want one bill" / "maximize RI utilization across accounts" → consolidated billing via **Organizations**. "Prevent accounts from using region X" / "block disabling CloudTrail" / "restrict EC2 instance sizes" → **SCP** at the right OU.

> [!warning] Gotcha
> A `Deny *` SCP attached to the management account does nothing — SCPs are skipped for the management account by design. If the question asks how to restrict the management account, the trick answer is "you can't with SCPs — move workloads to a member account."

## Cost Optimization

See [Cost Optimization](../patterns/cost-optimization.md) for the decision tree and pooling rules.

### Purchase Plans

| Plan | Term | Discount | What it covers |
| --- | --- | --- | --- |
| **Compute Savings Plan** | 1 / 3 yr | up to ~66 % | EC2 (any family/region/OS), Fargate, Lambda |
| **EC2 Instance Savings Plan** | 1 / 3 yr | up to ~72 % | Specific family in a specific region (size+OS flex) |
| **Standard RI** | 1 / 3 yr | up to ~72 % | Family/OS/region locked; size flexibility only |
| **Convertible RI** | 1 / 3 yr | up to ~54 % | Exchangeable for different family/OS/tenancy |
| **Reserved Capacity** (RDS / ElastiCache / Redshift / OpenSearch / DynamoDB) | 1 / 3 yr | up to ~72 % | The only purchase plan for these services |
| **Spot** | None | up to ~90 % | Spare EC2 capacity, 2-minute interruption warning |

> [!warning] Gotcha
> **Savings Plans do NOT cover RDS, ElastiCache, Redshift, OpenSearch, or DynamoDB.** For long-running databases, the answer is a **Reserved Instance / Reserved Capacity**, not a Savings Plan.

> [!tip] Exam Tip
> Pattern-match purchase plans: workload moves between EC2/Fargate/Lambda → **Compute SP**. Pinned `m5` in `us-east-1` → **EC2 Instance SP** (deeper discount). Long-running RDS → **Reserved DB Instance**. Batch / CI / checkpointable → **Spot**. Many accounts → consolidate via **Organizations** so SPs/RIs pool automatically.

## TLS, HTTPS, and ACM

See [TLS/HTTPS Concepts](../concepts/tls-https.md) for full details.

- **ACM** (AWS Certificate Manager): free, auto-renewed certs for ALB, CloudFront, API GW
- ACM certificates **cannot be exported** — private key never leaves AWS
- **CloudFront + ACM**: the certificate **must be in `us-east-1`** (N. Virginia), regardless of distribution origin regions
- **ALB + ACM**: certificate must be in the **same region as the ALB**
- ACM domain validation via **DNS** (add TXT record to Route 53) is fully automated if hosted zone is in the same AWS account
- **ACM Private CA**: issues private (non-publicly-trusted) certs for internal services

> [!warning] Gotcha
> **TLS termination at ALB** means ALB → EC2 traffic is plaintext by default. For end-to-end encryption, configure HTTPS on the target group and install a cert (can be self-signed) on the instances.

> [!tip] Exam Tip
> The TLS handshake uses **asymmetric encryption (RSA/ECDH) only for key exchange** — the actual data is encrypted with symmetric AES, which is far faster. The server's public key (from the cert) is used to securely establish a shared secret; after that, RSA is done.

## CloudWatch Logs

See [CloudWatch](../services/cloudwatch.md) for full details.

- CloudWatch Logs is a **regional, public-endpoint** service — reachable from VPCs, on-prem, and other clouds (given network + IAM access)
- Hierarchy: **Log Group → Log Stream → Log Event**; configuration (retention, permissions, metric filters) lives at the **log group** level
- **Log event** = timestamp + raw message block; CloudWatch Logs does not parse or interpret the content
- Native AWS services (Lambda, VPC Flow Logs, CloudTrail, Route 53) send logs automatically via IAM service roles
- **Unified CloudWatch Agent** is required for OS-level logs from EC2 and on-prem servers — EC2 does **not** auto-send logs, only basic metrics
- **Metric filters** scan log content for patterns → increment a CloudWatch Metric → optionally trigger an alarm

> [!tip] Exam Tip
> Pattern-match: *"detect failed SSH attempts and alert"* → **metric filter on a log group + CloudWatch Alarm + SNS**. *"EC2 instance logs not appearing in CloudWatch"* → **CloudWatch Agent not installed or IAM role missing**.

> [!warning] Gotcha
> Log **streams** have no configuration of their own — retention, permissions, and metric filters all attach to the **log group**. If a question asks where to set retention, the answer is the log group.

## KMS

See [KMS](../services/kms.md) for full details.

- KMS is **regional + public-endpoint**; keys are scoped to a region and **never leave** the service (FIPS 140-2 Level 2)
- Direct `Encrypt` / `Decrypt` is capped at **4 KB** per call — for larger payloads use **`GenerateDataKey`** + envelope encryption
- `GenerateDataKey` returns **two copies of the same DEK**: a **plaintext** version (use immediately, then discard) and a **ciphertext** version (store alongside the encrypted data). KMS does **not** store DEKs
- For **symmetric** `Decrypt`, you do **not** pass `--key-id` — KMS reads it from metadata embedded in the ciphertext blob
- **Three key ownership tiers**: **AWS Owned** (invisible, no control), **AWS Managed** (visible like `aws/s3`, AWS controls policy & rotation), **Customer Managed** (full control — required if you need custom policy, rotation control, grants, or deletion)
- Rotation creates **new backing key material** but keeps the **same logical Key ID**; previous backing keys are retained so old ciphertexts stay decryptable
- **Aliases** (`alias/...`) decouple callers from the underlying Key UUID — repoint an alias to switch keys without touching code
- **Every KMS key has a key policy, and it is required.** Unlike S3, **IAM alone cannot grant access** — the key policy must explicitly trust the account root first
- Three access-control layers compose: **Key Policy + IAM Policies + Grants**. Grants are temporary, programmatic delegations used by AWS services (EBS attaching an encrypted volume, Auto Scaling launching from an encrypted AMI, …)
- **Cross-account KMS access**: Allow on the key policy (key's account) **and** Allow on the caller's IAM identity (calling account) — identical intersection rule to cross-account S3

> [!tip] Exam Tip
> Pattern-match: *"encrypt a 5 GB file with KMS"* → **`GenerateDataKey` + envelope encryption** (direct `Encrypt` would exceed 4 KB). *"need to edit the key policy / disable rotation / share with another account"* → **Customer Managed** key, not AWS Managed. *"S3 SSE-KMS / EBS encryption uses KMS — but how does KMS handle a 100 MB object?"* → the **service** does envelope encryption; KMS only wraps the DEK.

> [!warning] Gotcha
> Granting `kms:*` in an IAM policy is **not** enough — without a key policy statement trusting the account root, the call still fails. This is the opposite default from S3, where IAM permissions alone work within the same account.

> [!warning] Gotcha
> KMS Keys are **region-scoped**. Encrypting in `us-east-1` and trying to decrypt in `us-west-2` with the same Key ID fails — the key doesn't exist there. Use **multi-region keys** if cross-region decrypt is required.

### S3 Object Encryption — SSE & CSE

See [S3 — Object Encryption](../services/s3.md#object-encryption--sse-and-cse) for full details.

- **Buckets aren't encrypted — objects are.** "Bucket default encryption" only sets the default SSE flavour for new uploads; two objects in the same bucket can use different SSE methods
- **SSE is mandatory** on S3 (no opt-out since 2023) — the question is only *which* SSE method, not whether to encrypt
- **In transit (HTTPS)** is automatic and orthogonal to **at rest (SSE/CSE)** — both happen
- **CSE ≠ SSE-C.** SSE-C is still **server-side** — S3 does the crypto; you just provide the key per request. CSE means S3 never touches plaintext

| Method | Key management | Encryption | Notable property |
| --- | --- | --- | --- |
| **CSE** | YOU | YOU | S3 never sees plaintext |
| **SSE-C** | YOU | S3 | Key supplied per request; S3 stores only a one-way hash |
| **SSE-S3** | S3 (invisible) | S3 | AES-256; no key control; **no role separation** |
| **SSE-KMS** | You + [KMS](../services/kms.md) | S3 | Rotation control + **role separation** + CloudTrail audit |

- **SSE-S3** = the default low-overhead option. S3 generates a per-object DEK, encrypts with **AES-256**, and wraps the DEK with an invisible internal master key. You give up key control, rotation control, and role separation
- **SSE-KMS** = uses [`GenerateDataKey`](../services/kms.md#data-encryption-keys-deks) against a Customer Managed KMS key. The encrypted DEK is stored in object metadata. This is the only SSE flavour that supports **role separation**, **rotation control**, and **CloudTrail-audited key usage**
- **SSE-C** = client supplies key on every PUT/GET. S3 keeps a hash to verify but destroys the key. Lose the key and the data is gone — by design

> [!tip] Exam Tip
> Pattern-match: *"AES-256"* → **SSE-S3**. *"Role separation / S3 admin must not be able to read object data"* → **SSE-KMS**. *"Audit who decrypts what / control rotation"* → **SSE-KMS**. *"Customer provides key per request, S3 stores only a hash"* → **SSE-C**. *"AWS must never see plaintext, ever"* → **CSE** (encrypt before upload).

> [!warning] Gotcha
> "Buckets are encrypted" is a trap framing. Encryption is **per-object**. If a question implies you can flip a bucket-level switch and have all existing objects re-encrypted, that's wrong — bucket default encryption only affects **new** objects from that point on.

> [!warning] Gotcha
> SSE-KMS uses [KMS DEKs](../services/kms.md#data-encryption-keys-deks) because KMS itself can only directly encrypt **≤ 4 KB**. Don't fall for *"how does KMS encrypt a 1 GB object?"* — it doesn't; S3 does the bulk encryption with a DEK that KMS generated.

#### S3 Bucket Keys — SSE-KMS cost optimization

See [S3 Bucket Keys](../services/s3.md#s3-bucket-keys--sse-kms-cost-optimization) for full details.

- **What it solves**: at scale (millions/billions of objects), SSE-KMS calls KMS once per PUT and once per GET — those request charges add up. **S3 Bucket Keys** can reduce KMS request charges by **up to 99%**
- **How it works**: S3 fetches a short-lived **bucket-level key** from KMS, caches it in memory, and uses it to derive **per-object DEKs locally** — no KMS round trip per object. The bucket key is still fetched **at least once per requester** so access is captured in CloudTrail
- **SSE-KMS only** — Bucket Keys don't apply to SSE-S3 (no KMS calls anyway), SSE-C (you supply the key), CSE (S3 doesn't encrypt), or DSSE-KMS (unsupported)
- **Only affects new objects** after enabling — to migrate existing objects, `CopyObject` re-encrypts them
- **Trust model unchanged**: bucket key is short-lived, never persisted, still derived from your KMS key

> [!warning] Gotcha — encryption context flips from object ARN to bucket ARN
> If you have a [KMS key policy](../services/kms.md#key-policies) or IAM policy with `kms:EncryptionContext:aws:s3:arn` conditions scoped to specific **object ARNs**, those conditions stop matching once Bucket Keys are on — S3 passes the **bucket ARN** instead. Update the policy conditions to use bucket ARN **before** enabling, or decrypts fail.

> [!warning] Gotcha — replication ETag drift
> Replicating an **unencrypted source object** into a destination bucket that has Bucket Keys + SSE-KMS as its default encryption re-encrypts the object on arrival — and the replica's **`ETag` differs from the source's**. Applications that compare source/replica ETags will break.

> [!tip] Exam Tip
> Pattern-match: *"SSE-KMS works but the KMS bill is huge on a bucket with millions of objects"* → **enable S3 Bucket Keys**. *"Cheaper than SSE-KMS"* on the same data is a misleading framing — Bucket Keys reduce **KMS request charges**, not the price per byte stored.

## Key Service Properties to Memorize

| Service | Resilience level |
| --- | --- |
| [IAM](../services/iam.md) | Global |
| [VPC](../services/vpc.md) | Regional |
| IGW | Regional |
| Subnet | AZ-specific |
| NAT GW | Zonal (need one per AZ for HA) |
| [EC2](../services/ec2.md) instance | AZ-specific |
| [S3](../services/s3.md) bucket | Regional (globally unique name) |
| [KMS](../services/kms.md) key | Regional (single-region by default; multi-region keys are a separate feature) |

## Common Exam Traps

- [NACLs](../services/vpc.md#network-acl-nacl) apply to **subnet boundary** traffic — intra-subnet traffic is not filtered
- [Security Groups](../services/vpc.md#security-groups-sg) attach to **ENIs**, not instances or subnets
- NAT GW needs an **Elastic IP** (assigned automatically on creation)
- 1 VPC → at most 1 IGW; 1 IGW → at most 1 VPC
- DHCP Options Sets cannot be edited — create a new one and associate it
- [IAM](../services/iam.md) and network security ([SG](../services/vpc.md#security-groups-sg)/[NACL](../services/vpc.md#network-acl-nacl)) are independent layers — both must pass

## References

- [IAM](../services/iam.md)
- [VPC](../services/vpc.md)
- [EC2](../services/ec2.md)
- [S3](../services/s3.md)
- [Networking Concepts](../concepts/networking.md)
- [Identity and Access Concepts](../concepts/identity-and-access.md)
- [Resource Policies vs Identity Policies](../concepts/resource-vs-identity-policies.md)
- [Route 53](../services/route53.md)
- [DNS Concepts](../concepts/dns.md)
- [TLS/HTTPS Concepts](../concepts/tls-https.md)
- [Organizations](../services/organizations.md)
- [Cost Optimization](../patterns/cost-optimization.md)
- [KMS](../services/kms.md)
