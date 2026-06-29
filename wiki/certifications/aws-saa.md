---
title: AWS Solutions Architect Associate — Exam Notes
type: certification
tags: [iam, vpc, networking, ec2, ami, ami-baking, ami-permissions, ami-marketplace, ami-lifecycle, block-device-mapping, instance-lifecycle, ssh, rdp, key-pair, iaas, shared-responsibility, host, tenancy, dedicated-host, dedicated-instance, instance-store, ephemeral, az-resilience, restart-vs-stop-start, instance-types, general-purpose, compute-optimized, memory-optimized, accelerated-computing, storage-optimized, graviton, burstable, gpu, fpga, eni, elastic-network-interface, primary-eni, secondary-eni, mac-address, private-dns, public-dns, split-horizon-dns, elastic-ip, eip, ipv6, source-destination-check, imds, instance-metadata, imdsv2, link-local, user-data, ssrf, status-checks, system-status-check, instance-status-check, auto-recovery, cloudwatch-alarm, sns, route53, organizations, cost, purchase-options, on-demand, spot, reserved-instances, scheduled-reserved, capacity-reservations, zonal-ri, regional-ri, savings-plans, byol, s3, foundations, object-storage, file-storage, block-storage, efs, fsx, ebs, gp2, gp3, io1, io2, block-express, provisioned-iops, per-instance-ceiling, iops-ratio, raid0, hdd, st1, sc1, throughput-optimized-hdd, cold-hdd, ebs-encryption, dek, envelope-encryption, aes-256, at-rest-encryption, dek-reuse, default-encryption, bucket-naming, bucket-limits, prefix, flat-namespace, regional-resilience, data-sovereignty, cloudwatch, multipart, transfer-acceleration, kms, encryption, sse, sse-kms, sse-s3, sse-c, cse, storage-classes, glacier, intelligent-tiering, lifecycle, replication, crr, srr, rtc, presigned-urls, s3-select, glacier-select, athena, events, event-notifications, eventbridge, lambda-trigger, access-logs, audit, observability, object-lock, worm, retention, legal-hold, compliance-mode, governance-mode, immutability, access-points, mrap, multi-region, vpc-endpoint, failover, dr, scaling, horizontal-scaling, vertical-scaling, stateless, sticky-sessions, elasticity, load-balancer, containers, docker, container-image, container-registry, ecs, eks, kubernetes, k8s, control-plane, etcd, kubelet, pod, ingress, persistent-volume, node-group, fargate-profile, ecr, fargate, task-definition, task-role, cloudtrail, audit, governance, management-events, data-events, insight-events, global-service-events, organizational-trail, log-file-integrity, event-history, eventbridge, control-tower, landing-zone, guardrails, account-factory, iam-identity-center, bootstrapping, cloud-init, boot-time-to-service-time, cloudformation, cfn-init, configsets, cfn-hup, cfn-signal, creation-policy, desired-state, infrastructure-as-code, cloudwatch-agent, cwagent, parameter-store, log-group, log-stream, placement-groups, cluster-placement-group, spread-placement-group, partition-placement-group, hpc, enhanced-networking, topology-aware, walking-the-tree, resolver, hosts-file, root-servers, nslookup, dig, dnssec, rrset, rrsig, dnskey, zsk, ksk, ds-record, trust-anchor, root-signing-ceremony, chain-of-trust, nsec3, cache-poisoning, rds, database, relational, subnet-group, multi-az, multi-az-instance, multi-az-cluster, read-replica, synchronous-replication, semi-synchronous-replication, asynchronous-replication, quorum, failover, reader-endpoint, cluster-endpoint, graviton, nvme, aurora, acid, base, cap-theorem, consistency, availability, eventual-consistency, dynamodb, nosql, backup, snapshot, automated-backup, manual-snapshot, transaction-logs, pitr, point-in-time-recovery, restore, rpo, rto, cross-region-backup, rds-security, encryption-in-transit, encryption-at-rest, ssl, tls, kms-key, cmk, dek, envelope-encryption, aes-256, tde, transparent-data-encryption, cloudhsm, iam-authentication, generate-db-auth-token, authentication, authorization]
certs: [aws-saa]
updated: 2026-06-27
---

# AWS Solutions Architect Associate (SAA-C03)

A running collection of exam tips, gotchas, and comparison tables. Deep-dive pages: [IAM](../services/iam.md) · [VPC](../services/vpc.md) · [EC2](../services/ec2.md) · [ECS](../services/ecs.md) · [ECR](../services/ecr.md) · [S3](../services/s3.md) · [RDS](../services/rds.md) · [Route 53](../services/route53.md) · [Organizations](../services/organizations.md) · [Control Tower](../services/control-tower.md) · [CloudWatch](../services/cloudwatch.md) · [KMS](../services/kms.md) · [EKS](../services/eks.md) · [Networking](../concepts/networking.md) · [Identity & Access](../concepts/identity-and-access.md) · [Resource vs Identity Policies](../concepts/resource-vs-identity-policies.md) · [DNS](../concepts/dns.md) · [DNSSEC](../concepts/dnssec.md) · [Database Models](../concepts/db-transaction-model.md) · [TLS/HTTPS](../concepts/tls-https.md) · [Containers](../concepts/containers.md) · [Kubernetes](../concepts/kubernetes.md) · [CloudFormation](../services/cloudformation.md) · [Cost Optimization](../patterns/cost-optimization.md) · [Scaling](../patterns/scaling.md) · [Boot Time to Service Time](../patterns/boot-time-optimization.md).

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

## S3 Foundations

See [S3 — Foundations](../services/s3.md#foundations) for full details.

- S3 is **object storage** — distinct from **file** (EFS / FSx) and **block** ([EBS](../services/ec2.md)). The access verb in the question picks the family: *mount* / *drive* / *volume* → not S3; *shared file share between EC2s* → EFS or FSx; *whole-object HTTP* / *data lake* / *land service output* → S3
- Buckets are **regional** but bucket names are **globally unique** across all AWS accounts and all regions
- Bucket name rules: **3–63 chars**, lowercase letters + digits + hyphens only, must start with a letter or digit, **cannot be IP-address-shaped** (`10.0.0.1` is invalid), **no underscores**
- **Object size: 0 B – 5 TB.** Single PUT cap is 5 GB → use [multipart upload](../services/s3.md#upload-performance) above that
- **Bucket count is per account, not per region:** soft limit **100**, hard limit **1,000** via Support. This is why "one bucket per user" doesn't scale — use **one bucket + per-user prefixes** instead
- **Objects per bucket: unlimited.** Object count is never a reason to need more buckets
- **Flat namespace.** What looks like a folder in the console is a **prefix** on the object key. There is no rename-a-directory; "moving a folder" rewrites every key
- **Regional resilience.** S3 replicates across **AZs within the home region** automatically; an AZ failure is transparent. A **region-wide failure is not** — that's what [CRR](../services/s3.md#replication) and [MRAP](../services/s3.md#multi-region-access-points-mrap--single-global-endpoint-over-a-replicated-bucket-fleet) are for
- **Data never leaves the home region** unless you explicitly configure replication. Region choice = data-sovereignty decision

### S3 foundations cheat sheet

| Keyword in the question | Answer |
| --- | --- |
| "Globally unique" bucket name | ✅ S3 — names collide across **all accounts and regions** |
| Bucket name with `_`, uppercase, or `1.1.1.1` shape | ❌ Invalid bucket name |
| Max object size | **5 TB** (multipart above 5 GB) |
| Min object size | **0 bytes** |
| Max objects per bucket | **Unlimited** |
| Default bucket limit per account | **100** (soft) |
| Hard bucket limit per account | **1,000** (via Support) |
| "Thousands of users, one bucket each" | ❌ Use **one bucket + per-user prefix** |
| "Mount as a drive / volume" | ❌ Not S3 — use **EBS** (block) or **EFS / FSx** (file) |
| "Network file share for Windows / Linux" | ❌ Not S3 — use **FSx** or **EFS** |
| "Default destination for an AWS service's output" | ✅ S3 |
| "Large-scale distribution to many users" | ✅ S3 |
| "Offload static assets from EC2" | ✅ S3 |
| "Folder" in S3 | Actually a **prefix** — flat structure underneath |
| Data leaves the home region automatically? | ❌ No — only via explicit **Replication** or **MRAP** |
| Blast radius of an S3 bucket | **The region** it was created in |
| Resilience model | Replicated **across AZs within the region** |

> [!tip] Exam Tip
> When multiple storage options appear next to each other (S3, EBS, EFS, FSx), **the access verb is the discriminator**, not the data type. *"Mount", "format", "drive letter", "block device"* → not S3. *"Shared file share between EC2 instances"* → EFS (Linux) / FSx (Windows). *"Whole-object HTTP", "static asset hosting", "data lake", "service input/output"* → S3.

> [!warning] Gotcha
> The bucket limit is **per account**, not per region — moving buckets between regions does not free up headroom. If a scenario describes "needs millions of isolated stores per tenant", **bucket-per-tenant is the wrong design**: use prefixes (and optionally [access points](../services/s3.md#access-points-and-multi-region-access-points)) for tenant isolation.

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

### S3 Object Lock

See [S3 — Object Lock](../services/s3.md#object-lock) for full details.

- **Object Lock = WORM (Write-Once-Read-Many)** layered on top of [versioning](../services/s3.md#versioning-and-mfa-delete). Protects object **versions** from delete/overwrite for a defined retention period
- **Must be enabled at bucket creation** by default; to enable on an **existing bucket**, **contact AWS Support**
- Enabling Object Lock **auto-enables versioning**; once enabled, neither Object Lock nor the auto-enabled versioning can be disabled — both are **one-way doors**
- **Two retention methods** that compose as an OR (either restriction blocks the action):
  - **Retention period** (days and/or years) with one of two modes — **Governance** or **Compliance**
  - **Legal Hold** — binary ON/OFF flag, no time bound
- Can be set per object version **or** as a **bucket default** (same pattern as default encryption)

#### Compliance vs Governance vs Legal Hold

| Mode | Time-bound? | Bypass possible? | Use when |
| --- | --- | --- | --- |
| **Compliance retention** | Yes (days/years) | **No — not even by root** | Regulatory mandate to retain unmodified for N years (SEC 17a-4, HIPAA, FINRA) |
| **Governance retention** | Yes (days/years) | Yes — `s3:BypassGovernanceRetention` + `x-amz-bypass-governance-retention: true` header | Internal anti-tampering with an authorized admin escape hatch |
| **Legal Hold** | No (binary flag) | Yes — `s3:PutObjectLegalHold` to toggle OFF | Active legal case / indefinite preservation with no predefined end date |

#### Required permissions

| Action | Permission |
| --- | --- |
| Configure Object Lock on a bucket | `s3:PutBucketObjectLockConfiguration` |
| Set/change retention on a version | `s3:PutObjectRetention` |
| Bypass Governance mode | `s3:BypassGovernanceRetention` + `x-amz-bypass-governance-retention: true` header |
| Add or remove a legal hold | `s3:PutObjectLegalHold` |

> [!tip] Exam Tip
> Pattern-match: *"WORM" / "immutable storage" / "regulatory retention"* → **Object Lock**. *"no one, including the root user, can delete or modify for N years"* → **Compliance mode** (the "even root" wording is the giveaway — MFA Delete won't satisfy this; only Compliance does). *"admins must be able to override the lock for emergencies"* → **Governance mode** + scoped `s3:BypassGovernanceRetention`. *"indefinite hold with no defined end date — active legal case"* → **Legal Hold**. *"enable Object Lock on an existing bucket"* → **Contact AWS Support**. *"Object Lock applies to objects or to versions?"* → **versions**. *"object lock + MFA Delete"* — these solve different problems: MFA Delete gates the bucket-versioning toggle and per-version permanent delete with a second factor (compromised-IAM-admin scenario); Object Lock is the WORM/regulatory-retention answer (no-one-can-touch-this-version-period scenario).

> [!warning] Common Object Lock Gotchas
>
> - **Compliance mode is irreversible.** Misconfigured retention of 10 years means 10 years of storage cost; AWS Support cannot override it
> - **Locks are on versions, not keys.** A new PUT to the same key creates a fresh, unlocked version unless the bucket-default Object Lock config catches it
> - **Retention and legal hold are independent** — removing one does not unlock the other
> - **The bypass header is default in the AWS Console** — console users with `s3:BypassGovernanceRetention` can override Governance locks without manually adding any header. Scope this permission tightly
> - **Versioning cannot be suspended** on an Object-Lock-enabled bucket — versioning is permanent for the life of the bucket

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

### S3 Event Notifications

See [S3 — Event Notifications](../services/s3.md#event-notifications) for full details.

- S3 emits notifications when objects are **created, deleted, restored (from Glacier), or replicated** — the trigger for event-driven workflows around the bucket
- **Three native destinations only**: **Lambda**, **SQS**, **SNS** — nothing else. (EventBridge is a separate, broader integration.)
- The destination needs a **resource policy** trusting the **`s3.amazonaws.com`** service principal — Lambda function policy / SQS queue policy / SNS topic policy. Without it, **delivery silently fails**
- Events are delivered as **JSON**; Lambda receives it inside the `event` argument
- **Object Created** fires on `PUT`, `POST`, `COPY`, **completed multipart upload**
- **Object Removed** fires on standard deletes + **delete-marker creation** (on versioned buckets)
- **Object Restore** fires when Glacier restore **starts** AND **completes**
- **Object Replication** fires on **missed 15-min RTC SLA**, replicated-after-threshold, no-longer-tracked, and **failed** replication
- **Filtering**: native event notifications support only **prefix/suffix** on the object key — anything richer needs EventBridge

#### Native S3 Events vs EventBridge

| Need | Pick |
| --- | --- |
| Trigger Lambda on object upload, simple bucket-local config | **S3 Event Notifications** (or EventBridge) |
| Destinations beyond Lambda/SQS/SNS (Step Functions, Kinesis, custom buses, SaaS) | **EventBridge** |
| Rich event-pattern filtering (any field, not just key prefix/suffix) | **EventBridge** |
| Lifecycle / intelligent-tiering / fine-grained replication events | **EventBridge** |
| Cross-account fan-out | **EventBridge** |
| Default for new workflows | **EventBridge** unless a specific reason to use the native feature |

> [!tip] Exam Tip
> Pattern-match: *"trigger Lambda when a file lands in S3"* → **S3 Event Notifications → Lambda** (also acceptable: EventBridge → Lambda). *"three destinations / Lambda, SQS, SNS only"* → native S3 Event Notifications. *"events delivered as JSON"* → yes. *"what permits S3 to invoke / send / publish"* → **resource policy on the destination** trusting **`s3.amazonaws.com`**. *"notify when Glacier restore completes"* → `s3:ObjectRestore:Completed`. *"detect failed cross-region replication"* → `s3:ObjectReplication:*` (or EventBridge). *"events not arriving, no error in CloudTrail on the bucket side"* → check the **destination's resource policy** — bucket isn't the requester, so IAM errors surface on the destination, not the bucket.

> [!warning] Gotcha — silent failure mode
> S3 Events fail silently if the destination's resource policy is missing or wrong. The S3 console wires up Lambda permissions for you when you add the destination, but if you build the configuration via CloudFormation / Terraform / CLI, the function/queue/topic policy is a **separate resource you must declare**. EventBridge has the same constraint on its targets.

### S3 Access Logging

See [S3 — Access Logging](../services/s3.md#access-logging) for full details.

- Async, best-effort request-level audit log of every operation against a **source bucket**, delivered as **Apache-style** text files into a **target bucket** (same region and same account)
- Configured on the **source bucket** via the console or **`PUT Bucket logging`** API; the XML config names the target bucket and an optional `TargetPrefix`
- Log delivery is performed by the AWS-managed S3 logging delivery path — *not* the source-bucket owner or the requester
- The target bucket grants write access to the **`logging.s3.amazonaws.com`** service principal. Use a **bucket policy** for modern buckets; an ACL grant to the legacy S3 Log Delivery Group is possible only when ACLs are enabled and is not recommended
- **Best-efforts, hours of latency** — enabling logging or changing config can take a few hours; delivery is not guaranteed real-time. Don't build alerting on top of it
- **Format**: log files contain many records; **records are newline-delimited**, **attributes within a record are space-delimited**. Conceptually identical to an Apache access log
- **One target bucket, many sources** — use distinct `TargetPrefix` values per source bucket
- **No built-in retention** — attach a **Lifecycle rule** to the target bucket (transition to Glacier, expire after retention window) or logs accumulate forever

#### Access Logs vs CloudTrail Data Events

| Need | Pick |
| --- | --- |
| Bulk, cheap, request-level audit log; no per-event charge | **S3 Access Logs** |
| Near-real-time, structured JSON, full IAM context (principal, session, MFA, source IP) | **CloudTrail data events** |
| Native integration with CloudWatch / EventBridge / Detective for alerting | **CloudTrail data events** |
| Compliance evidence of every read/write at minimal cost | **S3 Access Logs** + lifecycle to Glacier |

> [!tip] Exam Tip
> Pattern-match: *"who accessed which objects in my bucket, cheapest option"* → **S3 Access Logs**. *"where do I enable logging / where do logs go"* → enable on **source bucket**, delivered to **target bucket**. *"target bucket constraints"* → **same region and same account**, no Object Lock default retention, no Requester Pays. *"how does S3 get permission to write into the target bucket"* → **bucket policy** granting `s3:PutObject` to **`logging.s3.amazonaws.com`**. *"why is logging not showing up immediately"* → **best-efforts; can take hours**. *"record / attribute delimiter"* → **newline / space**. *"log retention"* → **your responsibility — use Lifecycle**. *"need real-time / structured / IAM-context-rich logs for security automation"* → **CloudTrail data events**, not access logs. *"need to trigger Lambda on object upload"* → **Event Notifications**, not access logs (access logs are batch/delayed).

> [!warning] Gotcha — ACLs disabled changes the answer
> Modern S3 defaults to ACLs disabled (Object Ownership = Bucket owner enforced). In that mode, an ACL grant to the S3 Log Delivery Group cannot work. The fix is a target-bucket policy granting `s3:PutObject` to `logging.s3.amazonaws.com`, not re-enabling ACLs.

### S3 Access Points & Multi-Region Access Points

See [S3 — Access Points and MRAP](../services/s3.md#access-points-and-multi-region-access-points) for full details.

- **Regular access points** are named DNS endpoints attached to a **single** bucket — each with its own resource policy (≤ 20 KB), network origin (`Internet` or `VPC`), and BPA settings. Solve the "one bucket policy is too small / too complex for many teams" problem
- **Multi-Region Access Points (MRAP)** are a **single global endpoint** over **2+ buckets in different regions**, with latency-based routing — solve the "single endpoint for multi-region S3" and "automatic regional failover" problems
- Both are **naming/routing layers, not permissions systems** — like a DNS CNAME with a policy attached. The underlying **bucket policy must still allow** the request
- Both live in the **`aws s3control`** CLI namespace (not `aws s3` / `aws s3api`)
- Both have **account-unique** names (not globally unique like bucket names)
- **Two-layer permissions rule**: matching permissions in both policies, **or** delegation pattern (recommended) — bucket policy uses `s3:DataAccessPointAccount` to allow any AP in the account; AP policies hold the real granularity
- AP-specific condition keys: `s3:DataAccessPointArn`, `s3:DataAccessPointAccount`, `s3:AccessPointNetworkOrigin` (`Internet` or `VPC`)

#### Regular Access Points — quick recall

- **Network origin** (`Internet` / `VPC`) and **BPA settings** are **immutable** after creation
- **Bucket association** is immutable — recreate the AP to change it
- AP must be in the **same region as the bucket**; regional ARN format `arn:aws:s3:<region>:<account>:accesspoint/<name>`
- **VPC-origin pattern** requires three pieces to line up: VPC-origin AP, VPC endpoint policy naming **both** the bucket ARN **and** the AP ARN, and (typically) a bucket policy denying any request whose `s3:AccessPointNetworkOrigin != VPC`
- **SCP guardrail** for org-wide VPC-only enforcement: `Deny s3:CreateAccessPoint` when `s3:AccessPointNetworkOrigin != VPC`
- **Cannot be a [Replication](../services/s3.md#replication) destination** — beware when locking a bucket to VPC-only APs (replication writes directly to the bucket, not through an AP)
- **Limit**: 10,000 access points per account per region (soft); 20 KB AP policy size

#### MRAP — quick recall

- Backing buckets are **fixed at creation** — cannot be added or removed later; recreate the MRAP to change
- **Versioning** must be enabled on every backing bucket (CRR requires it)
- **Replication is separate** — configure mesh **CRR** between the backing buckets, typically via the "Replicate among all specified buckets" template
- ARN format `arn:aws:s3::<account>:accesspoint/<alias>` — note the **empty region segment** (regionless)
- **Internet origin only** — MRAP does **not** support VPC origin (regular APs do)
- **Failover mode**: `active–active` (default; any active bucket can be picked) or `active–passive` (passives only used when no actives are healthy) — the active/passive variant is the **DR** pattern
- **Creation latency** — typically 10–30 minutes, AWS quotes up to **24 hours** worst-case
- **Consistency lag** — MRAP routing is regional/latency-based, async CRR replicates *after* the write. A read from a different region than the write *can* return `NoSuchKey` before replication finishes. **Mitigate** with [Replication Time Control (RTC)](../services/s3.md#replication-time-control-rtc) (99.9% within 15 min) or read-from-write-region in the app
- **Cost** — per-GB data routing fee on top of S3 storage, CRR transfer, and (optional) RTC

#### Access Points & MRAP — decision lookup

| Keyword in question | Answer |
| --- | --- |
| "Bucket policy too large / too complex / 20 KB limit" | **Regular S3 Access Points** |
| "Many teams need scoped views into one bucket" | **Regular Access Points** + **delegation** (`s3:DataAccessPointAccount`) |
| "Restrict bucket access to one specific VPC" | **VPC-origin Access Point** + VPC endpoint policy naming both ARNs |
| "Single global S3 endpoint" / "lowest-latency S3 bucket from anywhere" | **MRAP** |
| "Active/active S3 in multiple regions behind one endpoint" | **MRAP** + **mesh CRR** |
| "Automatic regional failover for S3" | **MRAP** with **active/passive** failover configuration |
| "Add another bucket to the MRAP later" | **Not supported** — backing buckets fixed at creation; recreate the MRAP |
| "Object not found right after writing through the MRAP" | **Consistency lag** — async CRR hasn't propagated; mitigate with **RTC** or read-from-write-region |
| "Guaranteed 15-minute replication SLA" | **[Replication Time Control (RTC)](../services/s3.md#replication-time-control-rtc)** |
| "MRAP / AP name uniqueness" | **Account-scoped** (bucket names are globally unique; APs and MRAPs are not) |
| "CLI to use an MRAP" | `s3://<MRAP-ARN>/key` — **ARN**, not bucket name |
| "Access point cannot be a…" | **[Replication](../services/s3.md#replication) destination** |
| "SCP to force VPC-only access points" | `Deny s3:CreateAccessPoint` when `s3:AccessPointNetworkOrigin != VPC` |
| "Bucket policy delegation to access points" | Use `s3:DataAccessPointAccount` to grant `*` to any AP in the account |

> [!tip] Exam Tip
> Pattern-match: *"bucket policy too large / many teams"* → **regular Access Points** with delegation. *"VPC-only S3"* → **VPC-origin AP** + VPC endpoint policy + (optionally) deny-non-VPC bucket policy. *"single global S3 endpoint / latency-route to closest bucket / multi-region S3 DR"* → **MRAP** + **mesh CRR** (+ **RTC** if the question stresses consistency or 15-minute SLA). *"object not found right after writing through MRAP"* → **consistency lag**; RTC narrows but doesn't eliminate it. *"can I add a bucket to the MRAP later?"* → **No** — fixed at creation. *"who grants the permission in a cross-account access point?"* → the **bucket owner** (the AP is just a doorway; the bucket policy is the real authority).

> [!warning] Common Gotchas
>
> - **Access points are not a permissions system.** The bucket policy is still the final authority. Forgetting this is the most common AP exam trap.
> - **Access points cannot be replication destinations.** A "VPC-only access points" bucket policy will silently break [Replication](../services/s3.md#replication) unless the replication role is allow-listed as an exception.
> - **MRAP gives up strong read-after-write consistency.** Regular S3 has it; MRAP does not. RTC narrows the window; only same-region read-from-write eliminates it.
> - **MRAP backing buckets are fixed.** Plan the bucket set up front — there's no add/remove API.
> - **MRAP has no VPC origin.** Only regular APs do. If you need both VPC scoping and multi-region routing, you must layer it at the application level.
> - **CLI namespace is `aws s3control`** for both AP types — not `aws s3` or `aws s3api`. Console wizards hide this distinction.

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

## EC2

See [EC2](../services/ec2.md) for full details.

- **EC2 = IaaS.** You manage OS + above; AWS manages hypervisor + below. Patching the guest OS is **your** responsibility — managed compute (Lambda, Fargate) is what shifts that line
- **EC2 is the default compute service** in AWS — pattern-match "general-purpose VM / I want an OS / long-running server" → EC2
- **Host vs Instance.** Host = physical AWS-managed hardware in one AZ. Instance = VM on a host, AZ-bound through it. Host, subnet, [ENI](../concepts/networking.md), [EBS](../services/ec2.md#storage-options) volume, and instance store are **all AZ-scoped** — no cross-AZ attachment
- **Tenancy:** **Shared** (default, hypervisor-isolated) / **Dedicated Instance** (no hardware co-tenancy, host opaque) / **Dedicated Host** (per-host allocation + per-host billing + host targeting + BYOL per-socket licensing)
- **Private by default.** An instance launches into one [VPC](../services/vpc.md) subnet; public reachability needs public subnet + [IGW](../services/vpc.md#internet-gateway-igw) + public IP + [SG](../services/vpc.md#security-groups-sg) rule. The **default VPC** wires this up automatically; **custom VPCs** require it explicitly
- **AZ-resilient, not AZ-fault-tolerant.** Instance lives in a subnet → subnet lives in an AZ → AZ failure kills the instance. HA = **fleet across multiple AZs + load balancer**, not "more durable instance"
- **Cross-AZ relocation is not native** — snapshot the EBS volume(s) or capture an AMI, then launch in the target AZ
- **State machine: Running ↔ Stopped → Terminated.** Stop ↔ Start is bidirectional; **Terminate is one-way** and irreversible
- **Stop/Start ≠ Restart.** Restart stays on the **same host** (instance store + auto public IP preserved). Stop/Start can **land on a new host** in the same AZ — instance-store data is **lost** and the auto-public-IPv4 is **released**
- **Billing granularity:** per second (Linux + most Windows AMIs, 60-second minimum) or per hour (some commercial-license AMIs)
- **AMI is region-specific** — to use it in another region, you must **copy** it (and pay the snapshot data-transfer cost). Stored in the region but **deployable to any AZ within that region** because the backing EBS snapshots live on regional S3. See [AMI — Amazon Machine Image](#ami--amazon-machine-image) for the full treatment
- **AMI permission modes:** **Private** (default, owner-only) / **Public** (everyone — e.g. AWS base images) / **Explicit** (specific account IDs granted launch permission)
- **AMI is a logical container, not a data carrier** — it holds an AMI ID + permissions + a block device mapping pointing at EBS snapshots. The real volume data lives in the snapshots; deleting the snapshots breaks the AMI
- **AMIs cannot be edited** — to change one, launch from it, modify, and create a brand-new AMI
- **Linux access: SSH/22 + private key** (direct public-key auth, no password). **Windows access: RDP/3389 + private key to decrypt the Administrator password**, then RDP with that password — the SSH key never touches the RDP session itself
- **Private key downloads once.** Lose it → create a new key pair (no recovery from AWS)

### EC2 state-based costs

| Component | Running | Stopped | Terminated |
| --- | :-: | :-: | :-: |
| CPU / Memory / Network | 💰 Charged | ✅ Free | ✅ Free |
| **EBS volumes** | 💰 Charged | 💰 **Still charged** | ✅ Free (deleted) |
| Public IPv4 (auto-assigned) | Held | Released | Released |
| Instance-store volumes | Present | **Lost** | Lost |
| Elastic IP (attached) | Free | Small hourly charge while idle | Charge continues until **released** |

### EC2 cheat sheet

| Keyword in the question | Answer |
| --- | --- |
| "Stopped the instance to save money, still being billed" | **[EBS](../services/ec2.md#storage-options) storage** still charged — only **Terminate** ends all costs |
| "Restore terminated instance" | ❌ Not possible — Terminate is irreversible; snapshot/AMI **before** terminating |
| "Single instance survives AZ failure" | ❌ Impossible — instance is AZ-scoped; need multi-AZ fleet |
| "Move instance to another AZ" | Snapshot EBS / capture AMI → launch new instance in target AZ; instances don't natively cross AZs |
| "Attach EBS volume from another AZ" | ❌ Impossible — EBS is AZ-scoped; snapshot → create volume in target AZ |
| "Multiple ENIs on different subnets" | OK only if the subnets are in the **same AZ** |
| "Rebooted, lost scratch disk" | Was it really a reboot? **Stop/Start** can land on a new host → **instance-store lost**. Restart keeps the same host |
| "Public IP changed after I cycled the instance" | Stop/Start releases auto public IPv4; use an **Elastic IP** for stability |
| "Use the same AMI in another region" | **Copy AMI** to the destination region first (`aws ec2 copy-image`); copied AMIs default back to **Private** permissions |
| "Use the same AMI in another AZ" | **No copy needed** — an AMI deploys to any AZ in its home region |
| "Edit an existing AMI to add a package" | **Cannot edit** — launch from it, configure, capture a new AMI |
| "Cost of an AMI sitting in an account" | Billed on the **EBS snapshots' used data**, not the AMI container or the volume's allocated size |
| "Marketplace AMI billing" | Normal instance hourly rate **plus** a per-hour software-license charge bundled into the AMI |
| "Build many identical web servers from a configured master" | **AMI baking** — configure one, capture as AMI, launch the fleet |
| "Per-socket Windows / Oracle / SQL Server licensing" | **Dedicated Host** — pay per host, target the host, satisfy BYOL per-socket licensing |
| "No hardware co-tenancy but I don't care about the host" | **Dedicated Instance** — single-tenant hardware, host stays opaque |
| "Instance has no internet despite being in public subnet" | Check public IP assignment, SG inbound, route to [IGW](../services/vpc.md#internet-gateway-igw), NACL ephemeral return |
| "Connect to Windows EC2" | RDP/3389 + use **private key to decrypt admin password** |
| "Connect to Linux EC2" | SSH/22 + private key (direct auth, no password) |
| "Lost the SSH key" | Create a new key pair; detach root volume → attach to recovery instance to re-key (no AWS recovery) |
| "Launch many instances from a configured image" | Create an **AMI** from the configured instance, then launch from it |
| "Share an AMI with another AWS account" | Set permission to **Explicit** with that account's ID |
| "Instance-level firewall" | [Security Group](../services/vpc.md#security-groups-sg) on the [ENI](../concepts/networking.md) — not a subnet-level control |
| "Service role for EC2 to call S3 / KMS / etc." | [IAM role](../services/iam.md#iam-roles) attached to the instance — **never embed access keys** |
| "Per-second vs per-hour billing" | Per-second (60s minimum) on most Linux + Windows; per-hour on certain commercial AMIs |
| "Long-running 24/7 server with a specific OS" | **EC2** (Lambda has a 15-min runtime cap; containers / Beanstalk are deviations that require a reason) |

> [!tip] Exam Tip
> Pattern-match: *"reduce cost of an idle instance overnight"* → **stop** it (saves compute) — but note **storage still bills**. *"reduce cost permanently"* → **terminate** + snapshot first if data matters. *"instance fails, AZ failed"* → expected behaviour; HA is a **multi-AZ fleet** concern, not an instance property. *"AMI not available in region X"* → **copy** AMI from source region. *"cannot SSH but security group + route are correct"* → check **NACL ephemeral return ports** (NACLs are stateless). *"scratch disk gone after cycling the instance"* → it was a **stop/start**, not a restart; the instance may have moved to a new host. *"BYOL with per-socket licensing"* → **Dedicated Host**.

> [!warning] Common EC2 Gotchas
>
> - **Stopped ≠ free.** EBS volumes keep billing while stopped — the most-tested cost trap
> - **Terminate deletes EBS root volume by default** (`DeleteOnTermination=true`). Snapshot first if you need the data
> - **Instance-store data is ephemeral** — lost on stop, lost on terminate, lost on host failure. Don't use it for anything you can't lose
> - **Stop/Start ≠ Restart.** Stop/Start can move the instance to a new host (same AZ) → loses instance-store data and releases the auto public IPv4. Restart keeps the same host
> - **Everything is AZ-scoped** — host, subnet, ENI, EBS, instance store. You cannot attach an EBS volume or ENI from one AZ to an instance in another AZ
> - **Dedicated Instance vs Dedicated Host** look similar but are not the same: Dedicated Host adds **per-host billing**, **host targeting**, and **BYOL per-socket licensing**; Dedicated Instance only guarantees no hardware co-tenancy
> - **AMI ≠ snapshot.** An AMI references one or more EBS snapshots plus metadata (permissions, block-device mapping); deleting just the underlying snapshots breaks the AMI
> - **Region-locked AMIs.** Multi-region launch requires `aws ec2 copy-image` first; otherwise the AMI ID won't exist in the target region
> - **AMIs cannot be edited.** The only path is (1) launch from the AMI, (2) modify the instance, (3) capture a new AMI; the old AMI is immutable
> - **Marketplace AMIs add a per-hour license charge** on top of the standard EC2 instance rate — easy to miss when sizing cost for vendor-bundled stacks
> - **Private key is download-once.** A lost private key on a single-key-pair instance means a painful recovery via root-volume detach

### Containers Fundamentals

See [Containers](../concepts/containers.md) for the full concept page.

- **Container vs VM:** a VM boots a full guest OS on a hypervisor; a container runs as an isolated process environment on a host OS. The container packages app code, libraries, and user-space dependencies, but shares the host kernel
- **Container engine:** Docker is the common name, but Docker is one implementation. "Docker container" is not synonymous with every possible container
- **Image vs container:** an image is the template; a container is a running instance of that image with a runtime read/write layer
- **Image layers are read-only.** A Dockerfile builds an image layer-by-layer. Each step stores only the filesystem change it introduced
- **Runtime R/W layer:** the per-container layer holds runtime changes such as logs and temporary files. Containers from the same image share the read-only layers and differ mostly by this layer
- **Registry:** images are pushed to and pulled from registries. Docker Hub is a public registry; AWS container service designs use the same registry/image/host/run vocabulary
- **Benefits:** portability, consistency, density, fast start/stop, and less duplicated OS overhead than VMs
- **Ports:** exposing a container port is only the container-level side. Real reachability still depends on host/task networking, load balancers, [VPC](../services/vpc.md) routes, and [Security Groups](../services/vpc.md#security-groups-sg)

#### Containers cheat sheet

| Keyword in the question | Answer |
| --- | --- |
| "Runs as a process, not a full OS" | **Container** |
| "Full guest OS, kernel, and boot lifecycle" | **VM / EC2 instance** |
| "Package app plus dependencies for portability" | **Container image** |
| "Build instructions for the image" | **Dockerfile** |
| "Read-only filesystem layers" | **Image layers** |
| "Per-container runtime filesystem changes" | **Read/write layer** |
| "Where images are pushed and pulled" | **Container registry** |
| "Docker Hub" | A public registry, not the whole container ecosystem |
| "Same image, many running copies" | Many containers sharing the image layers |

> [!tip] Exam Tip
> Pattern-match: portability, consistency, high density, and fast startup point to containers. Full OS control, per-socket licensing, or traditional VM migration points back to [EC2](../services/ec2.md#when-to-use-ec2).

#### Container gotchas

> [!warning] Common Container Gotchas
> A base image is not a full OS kernel; it contains user-space files and libraries, while the running container still uses the host OS kernel. Image layers are read-only, runtime changes belong in the per-container read/write layer, and that layer is not durable application state. Exposed ports also do not equal reachable services; AWS networking controls still decide whether traffic can arrive.

### ECS

See [ECS](../services/ecs.md) for the full service page and [Containers](../concepts/containers.md) for container fundamentals.

- **ECS is the managed orchestrator** for containers. It runs tasks from task definitions and keeps production workloads healthy through services
- **Hierarchy:** cluster contains services; services manage tasks; tasks contain one or more containers
- **Container definition:** image, ports, and container-level behavior
- **Task definition:** application template containing one or more container definitions plus CPU, memory, networking, compatibility, and roles
- **Task role:** [IAM role](../services/iam.md#iam-roles) assumed by the task. This is the best-practice way for application containers to access AWS services
- **Task execution role:** platform/setup role used by ECS or Fargate to pull images and write logs. Do not confuse it with the task role
- **Service:** long-running distributed layer that keeps a desired number of task copies running, replaces failed tasks, supports rolling deployments, and can attach to a load balancer
- **ECR:** AWS-managed container registry (see the dedicated [ECR section](#ecr) below); ECS task definitions can point at images in [ECR](../services/ecr.md) or another registry
- **EC2 launch type:** ECS places tasks on EC2 container instances in your account. You manage and pay for the host fleet even when it is idle, but you get EC2 pricing flexibility such as Spot and Reserved Instances
- **Fargate launch type:** no EC2 host fleet to manage. Tasks run on AWS-managed infrastructure and attach to your [VPC](../services/vpc.md) through [ENIs](../concepts/networking.md#elastic-network-interfaces-enis)
- **Fargate billing correction:** billed for configured task resources while tasks run, not for live CPU utilization only

#### ECS cheat sheet

| Keyword in the question | Answer |
| --- | --- |
| "Containerized app on AWS, managed orchestration" | **ECS** |
| "Where does ECS get the image?" | **Container registry**, commonly ECR |
| "One running copy of an application" | **Task** |
| "Application template with one or more containers" | **Task definition** |
| "One container inside a task definition" | **Container definition** |
| "Keep 5 copies running and replace failed copies" | **ECS service** |
| "Container app needs S3 / DynamoDB / KMS permissions" | **ECS task role** |
| "Pull image from ECR and write logs" | **Task execution role** |
| "Large, steady, price-sensitive container workload" | **ECS on EC2** |
| "Small / bursty / periodic / no host management" | **ECS on Fargate** |
| "Serverless but still has VPC networking" | **Fargate task with ENI** |

> [!tip] Exam Tip
> Pattern-match ECS launch type questions by the management line. If the company wants host control, Spot / Reserved economics, or dense packing for steady load, use ECS on EC2. If the company wants no host management or has small, bursty, or periodic container work, use ECS on Fargate.

> [!warning] Common ECS Gotchas
> A standalone task is not highly available by itself; use a service for production desired-count management. Fargate removes EC2 host management, but it does not remove VPC design: subnets, security groups, routes, public IP assignment, and load balancers still matter. The task role is for app permissions; the execution role is for ECS platform setup.

### ECR

See [ECR](../services/ecr.md) for the full service page.

- **ECR is the managed container image registry** — "Docker Hub for AWS." It stores, secures, scans, and distributes images; it does not build or run them
- **Hierarchy:** registry → repository → image → tag. **Tags are unique within a repository**
- **One private registry per account, per region** (endpoint `<account>.dkr.ecr.<region>.amazonaws.com`); plus access to a public registry. The private registry is a **regional** resource
- **Public registry:** anyone can pull, push needs permissions. **Private registry:** permissions required for **all** operations
- **[IAM](../services/iam.md) controls all access**; **repository policies** (resource policies) enable cross-account pulls
- **Enhanced scanning = Amazon Inspector**, OS + software packages, layer-by-layer; **Basic** is the lighter built-in scan
- **Observability:** CloudWatch (metrics), CloudTrail (API audit), EventBridge (image-change events)
- **Replication is cross-region AND cross-account**
- **Who pulls the private image?** On ECS it's the **[task execution role](../services/ecs.md#task-definitions-and-task-roles)**, not the task role

#### ECR cheat sheet

| Keyword in the question | Answer |
| --- | --- |
| "Managed container image registry on AWS" | **ECR** |
| "Tags must be unique within…" | The **repository** |
| "Anyone can pull, no credentials" | **Public** registry |
| "Permissions required for pull and push" | **Private** registry |
| "Grant another AWS account pull access" | **Repository (resource) policy** |
| "CVE / vulnerability scan, OS + language packages, layer-by-layer" | **Enhanced scanning (Inspector)** |
| "Lighter, default image scan" | **Basic scanning** |
| "Metrics on push / pull / auth" | **CloudWatch** |
| "Audit who called which ECR API" | **CloudTrail** |
| "Trigger a workflow when a new image is pushed" | **EventBridge** |
| "Copy images to another region or account" | **Replication** (cross-region and cross-account) |
| "ECS can't pull the private image" | Check the **task execution role** permissions |

> [!tip] Exam Tip
> The two highest-value ECR facts: **Enhanced scanning is the Inspector tier** (OS + software packages, layer-by-layer) — Basic is the lighter option; and **ECR replication is both cross-region and cross-account**. For "who pulls the image on ECS," the answer is the **task execution role**.

> [!warning] Common ECR Gotchas
> The private registry is **regional** — an image pushed to one region isn't in another unless you configure replication. Public vs private only changes the **read** default (anonymous pull on public); push always needs permissions. Don't credit **Inspector** to Basic scanning — only **Enhanced** uses it.

### Kubernetes & EKS

See [Kubernetes](../concepts/kubernetes.md) for the cloud-agnostic concept and [EKS](../services/eks.md) for the AWS service.

- **Kubernetes** is an open-source, **cloud-agnostic** container orchestrator. Control plane does the thinking; **nodes** run the workloads
- **Control-plane components:** **kube-apiserver** (front door, everything talks to it), **etcd** (key/value state store), **kube-scheduler** (assigns pods to nodes), **kube-controller-manager** (controllers, e.g. Node Controller responds to node outages), **cloud-controller-manager** (optional cloud integration)
- **Node software:** container runtime, **kubelet** (talks to control plane via the K8s API), and **kube-proxy** (runs on every node, implements services) — kube-proxy is a **node** component, not control-plane
- **Pod** = smallest unit; **think pods, not containers**; one-container-one-pod is the common pattern. **Pods are non-permanent and not highly available** — services provide the stable abstraction
- **Service** = consistent abstraction over pods (the "application"). **Job** = run-to-completion task. **Ingress** = external access to a service; the **Ingress Controller** provisions the hardware (on AWS, the AWS Load Balancer Controller uses ALB/NLB)
- **Storage:** default is **ephemeral and node-local** (like [EC2 instance store](../services/ec2.md#instance-store), lost on pod relocation); **Persistent Volumes** outlive pods
- **EKS = AWS-managed Kubernetes** — same Kubernetes, AWS runs it. Pick it for **cloud-agnostic / avoid lock-in / existing Kubernetes** workloads
- **Control plane AND etcd** are both AWS-managed and **multi-AZ**
- **Node options:** self-managed EC2, **managed node groups** (EC2), and **Fargate** (serverless, via **Fargate profiles**)
- **Integrations:** ECR (images), ELB (load balancing/ingress), IAM (security), VPC (networking)
- **Persistent storage providers:** EBS, EFS, FSx
- **Architecture:** control plane lives in an **AWS-managed VPC** (multi-AZ); EC2 workers live in **your VPC**; connectivity via **injected ENIs** or the **public endpoint**

#### Kubernetes & EKS cheat sheet

| Keyword in the question | Answer |
| --- | --- |
| "Container orchestration," open standard, avoid lock-in | **Kubernetes / EKS** |
| "Front end for the control plane" | **kube-apiserver** |
| "Cluster key/value backing store" | **etcd** |
| "Assigns pods to nodes" | **kube-scheduler** |
| "Responds to node outages" | **Node Controller** (in kube-controller-manager) |
| "Network proxy on every node" | **kube-proxy** (node component) |
| "Consistent abstraction over pods / the app" | **Service** |
| "External access to a service" | **Ingress** (+ Ingress Controller) |
| "Run to completion, then finish" | **Job** |
| "Storage that outlives a pod" | **Persistent Volume** (EBS/EFS/FSx on EKS) |
| "Managed Kubernetes on AWS" | **EKS** |
| "Already run Kubernetes elsewhere, want managed AWS" | **EKS**, not ECS |
| "Serverless EKS nodes" | **Fargate** via a **Fargate profile** |
| "EKS worker nodes — where do they run?" | **Your (customer) VPC** |
| "EKS control plane — where does it run?" | **AWS-managed VPC**, multi-AZ |

> [!tip] Exam Tip — ECS vs EKS
> Both run on EC2 or Fargate, so launch type is **not** the differentiator. Choose **EKS** when the scenario wants Kubernetes specifically (cloud-agnostic, open standard, existing K8s workload). Choose **ECS** when an AWS-native orchestrator is fine and you want to avoid EKS's **per-cluster control-plane hourly fee** (ECS has none).

> [!warning] Common Kubernetes & EKS Gotchas
> Pods are **not** durable or highly available — never design around a specific pod surviving. **kube-proxy and kubelet are node components**, not control-plane. Default pod storage is **ephemeral/node-local** (instance-store-like), not EBS-durable. On EKS, **both the control plane and etcd** are multi-AZ — don't credit only the control plane. EKS worker nodes sit in **your VPC**, while the control plane sits in an **AWS-managed VPC**. EKS bills an **hourly per-cluster** control-plane fee that ECS does not — a real cost reason to prefer ECS when Kubernetes isn't required.

### EC2 Status Checks & Auto Recovery

See [EC2 — Status Checks and Auto Recovery](../services/ec2.md#status-checks-and-auto-recovery) for full details.

- **Every EC2 instance has two independent status checks**: **System** (the EC2 service and host underneath) and **Instance** (the OS layer). A healthy, fully booted instance shows **2/2 checks passed** — anything else after boot completes is a real problem
- **System check fails** → AWS-side issue (host power, host network, host hardware/software). Common manual fixes that work: **Stop/Start** (lands on a new host) or **Recover** (auto recovery)
- **Instance check fails** → OS-side issue (corrupt filesystem, OS networking misconfig, kernel/boot problem). Reboot, console-based recovery, or rebuild from AMI
- **The static-public-IPv4 trap** — setting a public IPv4 on an OS interface is a classic instance-check failure pattern. The public IP lives on the [IGW](../services/vpc.md#internet-gateway-igw) as 1:1 NAT, never inside the guest
- **Status Check Alarm** = a [CloudWatch](../services/cloudwatch.md) alarm created via the EC2 console (*Actions → Create Status Check Alarm*). Fires when **either** check fails for **≥ 1 consecutive 5-minute period**
- **Alarm destinations**: SNS topic (notification) **plus** one optional EC2 action — **Reboot**, **Stop**, **Terminate**, or **Recover** (auto recovery)
- **Terminate as an alarm action** only "self-heals" if paired with an [Auto Scaling group](../patterns/scaling.md) that re-provisions a replacement — on a lone instance, Terminate is just deletion
- **Auto recovery (the Recover action)** brings the instance back to life by anything from a restart to migrating it to a **brand-new EC2 host**. **All IP addressing is preserved** — primary private IPv4, the auto-assigned public IPv4, and any Elastic IP. With software set to auto-start, the heal is fully unattended
- **Auto recovery requirements**: (1) **same AZ only** — does not protect against AZ failure, only isolated host/instance failures; (2) **supported instance type** (older docs cite A1/C4/C5/M4/M5/R3/R4/R5 — the modern list is much broader); (3) **EBS-only** — instance store would be lost on host migration; (4) **spare host capacity** must exist in the AZ
- **Simplified Automatic Recovery** is the modern default — most current-generation types are auto-recovered without you setting up a CloudWatch alarm. The mechanism and limits are identical; the difference is opt-out vs opt-in

#### Auto Recovery vs Stop/Start vs Restart cheat sheet

| Operation | Host migration | Instance-store data | Auto public IPv4 | Elastic IP |
| --- | --- | :-: | --- | :-: |
| **Restart (reboot)** | ❌ Never — same host | Preserved | Preserved | Preserved |
| **Stop → Start** | Possible (new host, same AZ) | **Lost** | **Released — new IP on Start** | Preserved |
| **Auto recovery** | Possible (new host, same AZ) | **Lost** (EBS-only requirement) | **Preserved** | Preserved |

#### Status Checks & Auto Recovery cheat sheet

| Keyword in the question | Answer |
| --- | --- |
| "Healthy instance shows X/Y checks" | **2/2 checks passed** |
| "The check that failed indicates a host hardware fault" | **System status check** |
| "The check that failed indicates an OS-level issue / corrupt FS / kernel problem" | **Instance status check** |
| "Static public IPv4 set on the OS interface" | ❌ Instance check failure pattern — public IPv4 is NAT'd at the IGW, never in the guest |
| "How do I get notified when a check fails" | **CloudWatch Status Check Alarm → SNS topic** |
| "Alarm triggers when…" | Either check fails for **at least one consecutive 5-minute period** |
| "Optional EC2 actions on the alarm" | **Reboot / Stop / Terminate / Recover** |
| "Self-heal a failed instance, keep its IP, no manual action" | **Recover** (auto recovery) |
| "Auto recovery preserves which IPs?" | **All of them** — primary private IPv4, auto-assigned public IPv4, **and** Elastic IP |
| "Auto recovery and AZ failure" | ❌ Same-AZ only — does not survive AZ-wide failure. Use **multi-AZ + ELB + Auto Scaling** for HA |
| "Auto recovery on an instance with instance-store volumes" | ❌ Not supported — **EBS-only** requirement |
| "Auto recovery fails during a major regional event" | Likely **no spare host capacity** in the AZ — recovery depends on AWS having a target host to migrate to |
| "Auto recovery isn't working on this old instance type" | Verify the type is supported — older list **A1, C4, C5, M4, M5, R3, R4, R5**; modern types are broader and most are recovered automatically (Simplified Automatic Recovery) |
| "Terminate action on a single instance — does it replace?" | ❌ No — Terminate **deletes**; replacement requires pairing with [Auto Scaling](../patterns/scaling.md) |

> [!tip] Exam Tip
> Pattern-match: *"check the host underneath"* → **System status check**. *"check the OS inside"* → **Instance status check**. *"automated heal that preserves the same IP"* → **Recover** (auto recovery), not Stop/Start. *"automated heal that survives AZ failure"* → ❌ not auto recovery — you need a **multi-AZ fleet behind a load balancer**. *"alarm fires after 5 minutes of failed checks"* → standard Status Check Alarm trigger. *"notify the team when an instance becomes unhealthy"* → **CloudWatch Status Check Alarm + SNS**.

> [!warning] Common Status Check / Auto Recovery Gotchas
>
> - **Auto recovery is single-AZ, not HA.** It saves you from an isolated host failure, not from an AZ outage. The exam trap is selecting "enable auto recovery" as the answer to "design for AZ failure"
> - **Auto recovery requires EBS-only.** Instance-store volumes block recovery — by definition, local disks cannot follow the instance to a new host
> - **Auto recovery requires spare capacity.** In a major regional event, the underlying mechanism can silently fail because AWS has no free host to migrate to
> - **Terminate as an alarm action is not a replacement mechanism on its own** — only when paired with Auto Scaling does termination self-heal. On a lone instance, Terminate ends the instance and that's it
> - **5 minutes of consecutive failure** is the trigger — shorter blips don't fire the alarm. Use richer CloudWatch metric thresholds if you need a tighter SLO
> - **Auto recovery preserves IPs, including the auto-assigned public IPv4** — this is the one case where a host-migration event does *not* release the dynamic public IP. The contrast with Stop/Start is exam-testable

### EC2 Instance Types

See [EC2 — Instance Types](../services/ec2.md#instance-types) for full details.

- The whole string (e.g. `R5dn.8xlarge`) is the **instance type** — use it in full; sub-components: family letter + generation + optional capability letters + size
- **Category = workload bottleneck**: C = CPU, R = RAM, I/D = local storage throughput, G/P = GPU/parallel processing, T/M = general purpose (burstable vs steady-state)
- **Additional capability letters** between the generation and size: `a` = AMD, `d` = NVMe local disk, `n` = network optimised, `e` = extra RAM/storage, `g` = Graviton (ARM)
- **Prefer the newest generation** within a family — better price-to-performance. Only deviate if the region lacks it or approvals require otherwise
- EBS-heavy workloads: the **instance's storage network bandwidth cap** is an independent ceiling from the EBS volume's provisioned IOPS — check both

#### Instance category quick-pick

| Category | Keyword in the question | Typical families |
| --- | --- | --- |
| **General Purpose** | "default", "balanced", "burst", "low average load" | T3, T3a, M5, M5a, M5n, A1, M6g |
| **Compute Optimized** | "CPU-intensive", "HPC", "media encoding", "gaming server", "scientific modelling" | C5, C5n |
| **Memory Optimized** | "in-memory database", "cache", "Redis", "real-time analytics", "large dataset in RAM" | R5, R5a, X1, z1d, High Memory |
| **Accelerated Computing** | "GPU", "ML training/inference", "FPGA", "parallel processing", "genomics" | P3, G4, F1, Inf1 |
| **Storage Optimized** | "NoSQL", "Elasticsearch", "data warehouse", "high IOPS local storage", "Hadoop/HDFS" | I3/I3en, D2, H1 |

> [!tip] Exam Tip
> Pattern-match: *"T3 performance degraded over time under consistent load"* → **CPU credit exhaustion** — switch to **M5** for sustained CPU workloads. *"max RAM per dollar / in-memory workload"* → **R5 / X1**. *"GPU / ML training"* → **P3 / G4**. *"programmable hardware / genomics / financial analysis"* → **F1 (FPGA)**. *"large NVMe SSD, NoSQL or analytics"* → **I3/I3en**. *"Hadoop / HDD-heavy throughput"* → **D2 / H1**. *"ARM-native, cost-optimised"* → **A1 / M6g (Graviton)**.

> [!warning] Gotcha — T3 burstable vs M5 steady-state
> **T3** instances accrue CPU credits at idle and spend them under load. Once credits run out, CPU is throttled to baseline. The trap: a T3 performs well initially then slows down. This is **not** an infrastructure failure — it is the burstable billing model working as designed. **M5** (non-burstable) is the correct fix for sustained steady-state CPU workloads.

### Instance Store

See [EC2 — Instance Store](../services/ec2.md#instance-store) and [EBS vs Instance Store](../services/ebs.md#ebs-vs-instance-store--decision-framework) for full details.

- Instance store volumes are **physically attached to the EC2 host** — no network hop → **highest storage performance in AWS**
- **Ephemeral** — data is lost when the instance moves to a new host: stop/start, resize, host failure, or volume hardware failure
- **Reboot preserves instance store data** — a restart stays on the same host. The trap: *"rebooted and lost scratch data"* is almost always a **stop/start in disguise**
- **Launch time only** — instance store cannot be added after the instance is running
- **Included in instance price** — no cost reason to avoid using already-allocated volumes
- **Best-in-class performance**: D3 → ~4.6 GB/s HDD; i3 → ~16 GB/s NVMe SSD, ~2 million read IOPS

#### Instance store vs EBS — decision table

| If the question emphasizes… | Pick |
| --- | --- |
| Persistence / durability | **EBS** |
| Resilience within an AZ + snapshots | **EBS** |
| Detach & reattach to another instance | **EBS** |
| Resilience **with** app-level replication | **Instance store** (across many instances) |
| Lowest cost HDD | **ST1 / SC1** |
| Throughput / streaming | **ST1** |
| Boot volume | **Not ST1/SC1** |
| 16,000–64,000 IOPS per volume | **IO1 / IO2** |
| Up to 256,000 IOPS per volume | **IO2 Block Express** |
| Max per instance (RAID 0 over EBS) | **~260,000 IOPS** |
| More than 260,000 IOPS, non-persistent OK | **Instance store** (i3: ~2 million IOPS) |

> [!tip] Exam Tip
> Pattern-match: *"highest possible storage performance, data loss acceptable"* → **instance store**. *"app replicates its own data (Cassandra, HDFS)"* → **instance store** across many instances — let the app provide durability, keep the speed and the cost saving. *"need more than ~260,000 IOPS per instance"* → **instance store** (RAID 0 over EBS is capped at ~260,000; i3 reaches millions). *"add volumes after launch"* → only **EBS** — instance store is **launch-time only**. *"D3 for sequential throughput"* → instance store (~4.6 GB/s) beats any EBS HDD. *"i3 for NVMe IOPS"* → instance store (~2 million IOPS) vs IO2 Block Express (256,000/volume, ~260,000/instance).

> [!warning] Common Instance Store Gotchas
>
> - **Reboot preserves, stop/start does not.** "Lost scratch data after a reboot" = almost always a stop/start; the instance landed on a new host
> - **Launch time only** — cannot attach instance store after launch; any question about adding post-launch is testing this
> - **Not all instance types include instance store** — verify instance type specs before assuming
> - **Included in price** — no reason to leave allocated volumes unused; the cost is already sunk

### EC2 Network Interfaces (ENIs) & Instance DNS

See [EC2 — Network Interfaces (ENIs)](../services/ec2.md#network-interfaces-enis) for full details.

- **Network identity lives on the ENI, not the instance.** Primary private IPv4, secondary private IPv4s, optional public IPv4, public/private DNS names, MAC, EIPs, IPv6, Security Groups, and source/destination check are **all per-ENI**
- **Every instance has exactly 1 primary ENI + 0 or more secondary ENIs.** The primary ENI **cannot be detached**; secondary ENIs **can** be detached and re-attached to another instance — the MAC, IPs, EIPs, and SGs travel with the ENI
- **All ENIs on one instance must be in the same AZ.** They can sit in **different subnets**, but those subnets must all belong to the same AZ — there is no cross-AZ ENI
- **Internal DNS name format**: `ip-<private-ipv4-dashed>.ec2.internal` (e.g. `ip-10-16-0-10.ec2.internal`) — resolvable only **inside the VPC**, always to the **private** IP
- **Public DNS name format**: `ec2-<public-ipv4-dashed>.compute-1.amazonaws.com` — and resolves with **split-horizon** behaviour
- **The auto-assigned public IPv4 is NOT visible inside the OS** — 1:1 NAT applied at the [IGW](../services/vpc.md#internet-gateway-igw). Configure only the **private** IPv4 in the guest. **IPv6 is OS-visible** (and all IPv6 in AWS is publicly routable)
- **The auto public IPv4 is dynamic** — released on **Stop**, a **new one** assigned on **Start**, preserved across **Reboot**. *"My public IP changed after I rebooted"* → the action was a **Stop/Start**, not a reboot
- **Attaching an EIP removes the dynamic public IPv4**; detaching the EIP later gives a **new, different** dynamic public IP — the original is **lost for good**
- **Source/destination check** is enabled by default — **must be disabled** for [NAT instances](../services/vpc.md#nat-gw-vs-nat-instance), software routers, or any in-line forwarding role
- **Security Groups attach to ENIs.** A SG governs **every IP on the ENI**. Need different SGs for different IPs on the same instance → split them across **multiple ENIs**
- **Movable secondary ENI = MAC-licensing & IP-failover pattern.** Bind the licence (or service IP + EIP) to a secondary ENI; re-attach it to a replacement instance instead of re-issuing or re-DNS-ing

#### Split-horizon DNS — the most exam-tested ENI behaviour

The EC2 public DNS name resolves **differently depending on caller location**:

| Caller | Public DNS name resolves to |
| --- | --- |
| Inside the VPC | **Private IPv4** |
| Outside the VPC | **Public IPv4** |

One hostname works everywhere; **intra-VPC traffic stays private** (never hairpins through the IGW). This is what makes VPC peering and hybrid networking work cleanly when apps connect by public DNS name. See [DNS — Split-Horizon DNS](../concepts/dns.md#split-horizon-dns).

#### ENI / DNS / EIP cheat sheet

| Keyword in the question | Answer |
| --- | --- |
| "Security Group attaches to…" | **ENI**, not the instance or subnet |
| "Different SGs for different IPs on one instance" | Split IPs across **multiple ENIs** |
| "Multiple ENIs on different subnets" | Allowed only if those subnets are in the **same AZ** |
| "Move the licence with the server when I replace it" | **Secondary ENI** with the MAC-bound licence + re-attach on the replacement |
| "Stable public IPv4 across stop/start" | **Elastic IP** (EIP) |
| "Detached my EIP and the original public IP came back" | ❌ Wrong — a **new, different** dynamic public IP is assigned; the original is lost |
| "Public IP changed after rebooting" | Likely a **Stop/Start in disguise** — reboot preserves the auto public IPv4 |
| "Internal DNS name doesn't resolve from on-prem" | `ip-…ec2.internal` is **VPC-only**; use Route 53 Resolver inbound endpoints or split-view PHZ for hybrid |
| "Public DNS name resolves to a private IP from inside the VPC" | **Split-horizon DNS** — intended behaviour |
| "Apps in two peered VPCs connect via public DNS — does traffic leave AWS?" | **No** — split-horizon DNS resolves to the **private IP** on both sides |
| "EC2 acting as a NAT instance drops packets" | **Source/destination check** is still enabled — disable it on the NAT instance's ENI |
| "Configure the public IPv4 inside the guest OS" | ❌ Not possible — public IPv4 is invisible to the OS; only IPv6 is OS-visible |
| "Cannot SSH after stop/start, OS firewall unchanged" | New auto public IPv4 was assigned — update your SSH target (or attach an **EIP** for stability) |
| "Need every existing public IP to remain forever" | **Allocate an EIP** *before* detaching anything — otherwise the dynamic IP is permanently lost on first EIP swap |

> [!tip] Exam Tip
> Pattern-match by ownership: *"X is attached to the instance"* — read it as *"X is attached to the instance's primary ENI."* The handful of exceptions matter: things that live on the **EC2 host** (instance store, host-affinity, dedicated-host BYOL) or in the **subnet/AZ** ([NACLs](../services/vpc.md#network-acl-nacl), [route tables](../services/vpc.md#route-tables)) are *not* on the ENI. Everything else — IPs, DNS names, MAC, EIPs, IPv6, SGs, source/dest check — is per-ENI.

> [!warning] Common ENI / DNS / EIP Gotchas
>
> - **EIP detach is a one-way IP swap.** Disassociate an EIP → a new dynamic public IPv4 is assigned, not the original. If you need the original, keep the EIP attached
> - **Public IPv4 invisibility in the OS** — only the private IPv4 is seen by the guest; the IGW does 1:1 NAT. IPv6 is the lone exception (OS-visible, all publicly routable)
> - **Reboot ≠ Stop/Start** for the auto public IPv4 — reboot preserves it; Stop/Start releases it and assigns a new one
> - **Source/destination check defaults on** — silently breaks NAT instances and any in-line forwarder until you turn it off
> - **MAC is per-ENI, not per-instance** — re-attaching a secondary ENI carries the MAC. Useful for MAC-locked licensing; also why the *primary* ENI's MAC is fixed for the life of the instance
> - **Internal DNS is VPC-scoped** — `ip-…ec2.internal` does not resolve from outside the VPC; use Route 53 private hosted zones + resolver endpoints for hybrid resolution

### EC2 Instance Metadata Service (IMDS)

See [EC2 — Instance Metadata Service (IMDS)](../services/ec2.md#instance-metadata-service-imds) for full details.

- **The endpoint is a memorizable URL**: `http://169.254.169.254/latest/meta-data/<attribute>` — link-local IPv4, **same address on every EC2 instance**. Memorize as "169.254 repeated" + "latest meta-data"
- **`169.254.0.0/16` is the link-local range** — the request never leaves the instance, so IMDS is reachable from inside even with no internet route or public IP
- **IMDS is the only way for code on the instance to discover its own public IPv4** — the OS configures only the private IPv4; the public IPv4 lives on the [IGW](../services/vpc.md#internet-gateway-igw) as 1:1 NAT. *"How does the app on the instance learn its public address?"* → **IMDS**, not `ifconfig` / `ipconfig`
- **IMDS is how the SDK gets IAM role credentials** — when an instance has an [IAM role](../services/iam.md#iam-roles) (instance profile) attached, the temporary creds (access key + secret + **session token** + expiry) live at `iam/security-credentials/<role-name>` and are refreshed automatically. This is the mechanism that makes *"never bake access keys into AMIs or user data"* possible
- **User data is exposed via IMDS** at `/latest/user-data/` (separate from `/meta-data/`) — bootstrap scripts can read what the launch passed them
- **EC2 Instance Connect injects short-lived SSH public keys via IMDS** — the SSH daemon's helper script pulls them out at login time
- **IMDSv1 has no authentication and no encryption** — anything with a shell on the instance, or any SSRF vulnerability in a web app on the instance, can read everything IMDS exposes including the **role credentials**. This is the Capital One 2019 breach pattern
- **IMDSv2 fixes this with session tokens** — clients first `PUT` to `/latest/api/token` (with `X-aws-ec2-metadata-token-ttl-seconds`), then send the returned token in `X-aws-ec2-metadata-token` on every subsequent `GET`. SSRF cannot obtain the token because most HTTP libraries forced into making a `GET` cannot pivot to a `PUT`
- **Enforce IMDSv2 with `HttpTokens=required`** — per-instance setting; modern hardening baseline. `optional` accepts both v1 and v2
- **The default version is set by the AMI's `imds-support` parameter, not the OS** — **Amazon Linux 2** leaves it unset, so instances default to `optional` (IMDSv1 + IMDSv2 both enabled, v2 not enforced). **Amazon Linux 2023** sets `imds-support=v2.0`, so instances default to `required` (IMDSv2 only, IMDSv1 disabled) with hop limit 2

#### IMDS cheat sheet

| Keyword in the question | Answer |
| --- | --- |
| The metadata service URL | **`http://169.254.169.254/latest/meta-data/`** |
| "Code on the instance needs its public IPv4" | **IMDS** — `meta-data/public-ipv4` |
| "App on EC2 needs to call S3 without storing access keys" | **IAM role on the instance + SDK reads creds from IMDS** at `iam/security-credentials/<role>` |
| "Bootstrap script needs to read the user-data passed at launch" | `http://169.254.169.254/latest/user-data/` |
| "Pulled credentials from the metadata service via SSRF" | The **Capital One** pattern — IMDSv1 vulnerability; fix is **enforce IMDSv2** |
| "Mitigate SSRF against the metadata service" | **`HttpTokens=required`** (forces IMDSv2 session-token handshake) |
| "Block all metadata access from inside the OS" | Local OS firewall rule blocking `169.254.169.254` — per-instance, high-overhead; IMDSv2 is the real answer |
| "Metadata service uses TLS" | ❌ No — IMDS is **plain HTTP** over link-local. Confidentiality comes from the local-only addressing, not from encryption |
| "169.254.169.254 reachable from outside the instance" | ❌ No — link-local; never leaves the instance |
| "Same IP address for IMDS in `us-east-1` and `ap-southeast-2`" | ✅ Yes — link-local, identical on every instance everywhere |
| "Which Amazon Linux requires IMDSv2 by default" | **AL2023** (`imds-support=v2.0`, IMDSv1 disabled). **AL2** defaults to `optional` (both enabled) |

> [!tip] Exam Tip
> Pattern-match: *"discover own public IPv4 from inside the instance"* → **IMDS** at `meta-data/public-ipv4`. *"SDK on EC2 obtains credentials"* → **IAM role + IMDS** (no static keys). *"SSRF stole the role credentials from metadata"* → **enforce IMDSv2** (`HttpTokens=required`). *"the metadata endpoint URL"* → **`http://169.254.169.254/latest/meta-data/`** — memorize verbatim.

> [!warning] Common IMDS Gotchas
>
> - **IMDSv1 is unauthenticated** — any local process or SSRF-vulnerable web app reads everything including the **role's temporary credentials**. Treat the role's policy as production-secret-equivalent
> - **IMDS is plain HTTP**, not HTTPS. The link-local addressing is the only confidentiality boundary
> - **IMDS works without internet access** — link-local addressing routes inside the hypervisor; an instance in a fully private subnet still reads metadata fine
> - **Stop/Start changes the public IPv4** — and IMDS will return the new one immediately. Apps that cache `public-ipv4` need to re-read after a host migration
> - **OS firewall blocking 169.254.169.254 breaks SDK credential resolution** — the SDK has no other path to the role's temporary keys. Don't blanket-block; enforce IMDSv2 instead

### AMI — Amazon Machine Image

See [EC2 — AMI](../services/ec2.md#ami--amazon-machine-image) for full details.

- **AMI = a logical container, not a data carrier.** Holds an AMI ID, permissions, and a **block device mapping** pointing at EBS snapshots. The volume data lives in the snapshots — the AMI itself stores no bytes
- **Three sources**: **AWS-provided** (Amazon Linux, Windows base), **Community** (Red Hat, CentOS, Ubuntu), **Marketplace** (commercial software bundled). Marketplace billing = **instance hourly rate + per-hour software-license charge**
- **Three permission modes**: **Private** (default — owner-only) / **Public** (everyone) / **Explicit** (specific named AWS account IDs)
- **Regional** — one AMI lives in exactly one region. Same OS has a **different AMI ID per region**. **Deployable to any AZ within the region** (snapshots live on regional S3)
- **Four-phase lifecycle**: (1) Launch from an AMI, (2) Configure the instance, (3) Create Image (captures snapshots and writes the block device mapping), (4) Launch new instances from the new AMI. The flow runs in **both directions** — using an AMI to launch an instance, and capturing an instance as an AMI
- **AMIs cannot be edited.** To change an AMI, launch from it, modify the instance, capture a **brand-new** AMI
- **AMIs can be copied across regions** with `aws ec2 copy-image`. Copies the underlying snapshots; copied AMIs **default back to Private** — sharing does not travel with the copy
- **Encrypted AMIs** require [KMS](../services/kms.md) grants for services like Auto Scaling to launch from them. Cross-region copy re-wraps the snapshot DEK under a destination-region KMS key
- **Billing**: pay for the **EBS snapshot used data** the AMI references, not for the AMI container, and not for the source volume's full allocated size. A mostly-empty large volume produces a small, cheap snapshot
- **AMI baking** = configure an instance, install all software, capture as AMI, launch many identical instances. The canonical pattern for fleet provisioning and golden-image pipelines (EC2 Image Builder, Packer)

#### AMI cheat sheet

| Keyword in the question | Answer |
| --- | --- |
| "Same OS, different AMI IDs across regions" | Yes — AMIs are **regional**; same OS gets a distinct AMI ID in each region |
| "Move an AMI to another region" | **`aws ec2 copy-image`** — copies the underlying snapshots; permissions reset to Private |
| "Move an AMI to another AZ" | **No action needed** — an AMI deploys to any AZ in its home region |
| "Modify an existing AMI" | ❌ Not possible — launch from it, configure, capture a **new** AMI |
| "Share AMI with a specific AWS account" | **Explicit** permission with that account's ID |
| "Share AMI with everyone" | **Public** permission — also the model for AWS-provided base images |
| "Default AMI permission" | **Private** (owner-only) |
| "Marketplace AMI cost model" | Normal instance hourly rate **plus** a per-hour software-license charge |
| "What does the AMI actually store?" | AMI ID + permissions + **block device mapping** pointing at EBS snapshots. **No volume bytes inside the AMI** |
| "Block device mapping" | A table: snapshot ID + original device ID (e.g. `/dev/xvda` for boot, `/dev/xvdf` for data). Used at launch to recreate the volume layout |
| "How much does an idle AMI cost?" | **Snapshot used data × $/GB-month** — not the volume's allocated size, not the AMI itself |
| "Configure once, launch many" | **AMI baking** — capture the configured instance as an AMI |
| "AMIs created from one AZ" | Deployable to **all AZs in the region** — the AMI's snapshots are stored on S3, which is regional |
| "Encrypted AMI fails to launch via Auto Scaling" | Auto Scaling needs a **KMS grant** on the AMI's encryption key |

> [!tip] Exam Tip
> Pattern-match: *"create many identical instances from a customized server"* → **AMI baking** (capture as AMI, then launch). *"AMI not available in region X"* → **copy AMI** to that region first. *"edit an AMI to update the package version"* → ❌ not possible; (1) launch from the AMI, (2) modify the instance, (3) capture a new AMI. *"AMI created in AZ-a — can it launch in AZ-b?"* → ✅ yes, same region is enough. *"Marketplace AMI total cost"* → instance rate **plus** per-hour license. *"Who owns the data in an AMI?"* → the EBS snapshots underneath; the AMI is just a pointer.

> [!warning] Common AMI Gotchas
>
> - **AMIs are immutable.** Every change is a new AMI. The exam trap framing is *"edit the AMI to upgrade the OS"* — always wrong
> - **Region-locked.** Multi-region launch requires `copy-image` first. Permissions reset to Private after copy
> - **Logical container, not data.** Deleting the underlying snapshots breaks the AMI; AMI lifecycle and snapshot lifecycle must be managed together
> - **Snapshot used-data billing.** A 100 GB volume with 5 GB written produces an AMI billed for ~5 GB, not 100 GB. The cost is in the *snapshot*, not the AMI object
> - **Marketplace AMIs carry a license charge** bundled into the per-hour bill — not just the instance rate
> - **Encrypted AMI + Auto Scaling** silently fails without a KMS **grant** on the encryption key — common cross-account / multi-region trap

### EC2 Bootstrapping — User Data & cfn-init

See [EC2 — Bootstrapping](../services/ec2.md#bootstrapping), [CloudFormation — cfn-init](../services/cloudformation.md), and [Boot Time to Service Time](../patterns/boot-time-optimization.md) for full details.

- **Bootstrapping** = self-configure at launch. In EC2 it's delivered via **user data**, read from [IMDS](../services/ec2.md#instance-metadata-service-imds) at **`http://169.254.169.254/latest/user-data/`** (same service as metadata, different path)
- User data **runs once, at first launch only**, **as root**, and is **opaque to EC2** (not validated/interpreted)
- **Edit + restart does NOT re-run user data** — the new data is visible but never re-executed. One-shot
- **Failed bootstrap does not fail the instance** — it stays running and passes **2/2 status checks** → silent **bad config**. EC2 can't tell good config from bad
- **Not a secret store** — readable by anyone with code/shell on the box; **enforcing IMDSv2** (default on Amazon Linux 2023) closes the *remote SSRF* path but does **not** make user data secret. **No long-term credentials** — use an [IAM role](../services/iam.md#iam-roles)
- **16 KB limit** — for bigger payloads, bootstrap a small script that **downloads** the rest
- **Base64**: console **auto-encodes**; **CloudFormation requires `Fn::Base64`**
- Diagnose with **`/var/log/cloud-init-output.log`** (user data) and **`/var/log/cfn-init-cmd.log`** (cfn-init)
- **cfn-init** = desired-state config management (version-aware packages/files/services/commands), defined in **`AWS::CloudFormation::Init`** metadata, **invoked from user data**. Procedural *or* desired-state, vs user data's procedural-only
- **cfn-hup** lets cfn-init **re-run on stack-metadata updates** — user data can't (it's one-shot)
- **CreationPolicy + cfn-signal** (`cfn-signal -e $?`) is the only way CloudFormation **knows** in-instance bootstrapping succeeded: it **holds** the resource at a non-complete state and **waits for a signal** within a **timeout** (e.g. `PT15M`); no signal = assumed failure. Used on **EC2 instances** and **Auto Scaling Groups**
- **Boot time to service time** = provisioning time + post-launch config time. Three approaches: **bootstrap** (flexible, slow each launch), **AMI baking** (fast, frozen config), **combine both** (bake slow/stable part + bootstrap fast-changing part = optimal)

> [!tip] Exam Tip
> Trigger phrases: *"run a script at launch / configure on first boot"* → **user data**. *"failed bootstrap but instance still healthy / passes status checks"* → user data is **opaque**; detect failure with **cfn-signal**. *"re-apply config on stack update, not just first launch"* → **cfn-init + cfn-hup** (NOT user data). *"ensure a specific package version / complex desired-state bootstrapping"* → **cfn-init**. *"don't store credentials in user data"* → use an **IAM role**. *"minimize boot time to service time while keeping flexibility"* → **combine AMI baking + bootstrapping** (bake the 90%, bootstrap the 10%).

> [!warning] Common Bootstrapping Gotchas
>
> - **User data is one-shot.** Edit + stop/start does not re-run it. Re-runnable config = **cfn-init + cfn-hup**
> - **A broken bootstrap looks healthy.** Running + 2/2 status checks ≠ correctly configured. CloudFormation only knows via **CreationPolicy + cfn-signal**
> - **Console auto-base64s, CloudFormation does not** — forget `Fn::Base64` in a template and the launch breaks
> - **`-e $?`** in cfn-signal forwards the **exit code of the preceding cfn-init run** — that's what becomes the success/failure signal
> - **User data is not encrypted/secret** — IMDSv2 stops remote SSRF but a shell user still reads it; never put long-term secrets there

### EC2 Instance Roles & Instance Profiles

See [EC2 — Instance Roles & Instance Profiles](../services/ec2.md#instance-roles--instance-profiles) and [IAM Roles](../services/iam.md#iam-roles) for full details.

- **EC2 instance role** = an [IAM role](../services/iam.md#iam-roles) whose **trust policy trusts `ec2.amazonaws.com`**. It's how an app on EC2 reaches other AWS services with **no long-term access keys on the instance**
- **Instance profile** = a wrapper around **one** IAM role. A role **cannot attach to an instance directly** — the **profile is what attaches**
- **Console**: creating the role **auto-creates a same-name instance profile** (the two-object reality is hidden). **CLI / CloudFormation / API**: you create **role and profile separately**, and they **can have different names** (`AWS::IAM::Role` + `AWS::IAM::InstanceProfile` are two resources)
- **Credential delivery**: temporary creds (Access Key + Secret + session token + expiry) are delivered via **[IMDS](../services/ec2.md#instance-metadata-service-imds)** at **`iam/security-credentials/<role-name>`**, and **EC2 + STS auto-renew them before expiry** → always valid while attached. Apps should re-read metadata, not over-cache
- **Credential provider chain**: instance profile credentials are the **last** source the AWS CLI checks → an instance with a role needs **zero credentials configured**. But **anything higher silently shadows the role** (env vars `AWS_ACCESS_KEY_ID`, `~/.aws/credentials`, CLI flags)

> [!tip] Exam Tip
> *"App on EC2 needs to access S3 / DynamoDB / etc."* → **instance role** (not access keys). *"Access keys / hardcoded credentials on an instance"* → **bad practice**, replace with an instance role. *"What is attached to the instance — role or profile?"* → the **instance profile**. *"How do role credentials reach the app without keys?"* → **temporary creds via IMDS, auto-renewed by EC2 + STS**. *"Role attached but app uses wrong/expired permissions"* → **higher-precedence credentials (env vars / credentials file) shadowing the role** — remove them so the chain falls through to IMDS. Mnemonic **RPM**: **R**ole holds permissions, **P**rofile attaches, **M**etadata delivers.

### EC2 Placement Groups

See [EC2 — Placement Groups](../services/ec2.md#placement-groups) for full details.

- Placement groups influence the **physical placement** of instances on EC2 hosts: **Cluster** packs them close together for performance; **Spread** and **Partition** separate them for resilience
- **Cluster**: same rack, often same host; **10 Gbps single-stream** throughput (normal cap is 5 Gbps); lowest latency and max PPS in AWS; **single AZ only** (locked to the AZ of the first instance launched, not chosen at creation); little to no resilience
- **Spread**: each instance on its own isolated rack with its own network and power; **hard limit of 7 running instances per AZ *per group*** (per placement group, **not** account-wide — for more, AWS recommends multiple spread groups, but there's no guarantee of spread *between* groups); can span multiple AZs
- **Partition**: **max 7 partitions per AZ**, each on isolated racks with no sharing between partitions; **unlimited instances**; you pick the partition or EC2 decides; exposes partition topology to applications

#### Placement group decision table

| If the question emphasises… | Pick |
| --- | --- |
| Highest performance, lowest latency, fastest node-to-node (HPC, scientific) | **Cluster** |
| Small number of critical instances, max availability, separate blast radius | **Spread** |
| Large-scale distributed/replicated app, more than 7 instances, topology-aware (HDFS / HBase / Cassandra) | **Partition** |

#### Numbers to memorise

| Fact | Value |
| --- | --- |
| Cluster single-stream throughput | **10 Gbps** (vs **5 Gbps** normal) |
| Spread instances per AZ | **7** (hard limit, **per group** not per account) |
| Partition *partitions* per AZ | **7** (max) |
| Partition instances per partition | Unlimited (your choice) |

> [!tip] Exam Tip
> The cluster performance numbers are conditional: you need a **supported instance type** with high-performance networking **and enhanced networking enabled** on all members. Same instance type and launching all members at the same time are **recommended, not mandatory**. And the mnemonic: **C**luster is **C**lose together for **C**ompute performance; **S**pread is **S**eparated, **S**even per AZ; **P**artition gives 7 **P**artitions per AZ for **P**arallel-processing apps at scale.

> [!warning] Common Placement Group Gotchas
>
> - **Cluster cannot span AZs** — it can span **VPC peers**, but that significantly degrades performance
> - **Cluster has low or no resilience** — co-location means one hardware failure can take out the whole group
> - **Spread caps at 7 running instances per AZ *per group*** (not account-wide) — a scenario needing more separated instances is pointing at **Partition**, not Spread (the most common trap: separation **and** lots of instances means Partition). You *can* run multiple spread groups for more than 7, but AWS gives **no spread guarantee between** groups — so it's not a substitute for Partition at scale
> - **Spread does not support Dedicated Instances or Dedicated Hosts**
> - **Partition isolation is per partition, not per instance** — 10 instances in 1 partition all share that partition's fate

### EC2 Enhanced Networking

See [EC2 — Enhanced Networking](../services/ec2.md#enhanced-networking) for full details.

- **What it is:** a feature that boosts EC2 network performance via **SR-IOV** (Single Root I/O Virtualization), which makes the host's **physical NIC virtualization-aware**
- **How it works:** instead of one shared physical card the host arbitrates **in software**, the card presents **per-instance logical cards**. Each instance gets exclusive access to its own, and the **host CPU is taken out of the packet path** (offloaded to hardware, the **ENA** on AWS)
- **What it delivers:** more bandwidth, **higher packets per second (PPS)**, and — the key phrase — **consistent low latency**
- **Availability:** enabled by default or free on **most modern instance types**; SAA does not test the enablement mechanics
- **Why it matters:** it's the **prerequisite for [cluster placement groups](../services/ec2.md#cluster-placement-groups)** — the 10 Gbps / low-latency cluster numbers require it on every member

> [!tip] Exam Tip
> Map these triggers straight to **enhanced networking**: **"consistent low latency," "high packets per second / PPS," "small packets over a small isolated network,"** and **"required for cluster placement groups."** Don't overthink it — the SAA exam cares about *what it gives you* (PPS + consistent latency) and *what depends on it* (cluster placement), not how to turn it on.

### EC2 EBS-Optimized Instances

See [EC2 — EBS-Optimized Instances](../services/ec2.md#ebs-optimized-instances) for full details.

- **What it is:** a per-instance **on/off** flag that gives **[EBS](../services/ebs.md) traffic a dedicated network lane**, separate from regular data networking
- **Problem it solves:** historically one **shared network stack** carried both data and EBS storage traffic, so the two **contended** and capped each other's performance. EBS is **network-attached**, so its I/O always crosses the network
- **Effect:** faster EBS throughput / IOPS, and storage vs data traffic **no longer compete**
- **Required for:** instances driving **high throughput / IOPS** volumes (e.g. **gp2 / io1**); it's the instance-side half of the [EBS per-instance ceiling](../services/ebs.md#the-two-ceiling-rule)
- **Availability:** **default-on and free on most modern instance types** (disabling does nothing); paid opt-in on some older types

> [!tip] Exam Tip
> Two distinct "dedicated capacity" features that are easy to confuse: **enhanced networking** dedicates the **data**-network path (PPS, consistent latency, cluster placement); **EBS optimization** dedicates the **storage**-network path (EBS throughput/IOPS, no contention with data traffic). If the scenario is about **disk performance or EBS/network contention**, it's **EBS-optimized**; if it's about **packet rate / node-to-node latency**, it's **enhanced networking**.

### EBS Service Architecture

See [EBS — Service Architecture](../services/ebs.md#service-architecture) for full details.

- **EBS is AZ-scoped.** A volume lives in exactly one AZ; an instance must be in the same AZ to attach it.
- **File system comes from the OS.** EBS provides the raw block device; the EC2 instance's OS (Linux/Windows) formats and mounts it. EBS has no concept of EXT4 or NTFS — that's entirely the instance's responsibility.
- **Snapshots are stored in S3** (AWS-managed, regionally resilient — 3+ AZs). They are the only way to move volume data across AZs or regions.
- **Cross-region DR path:** snapshot the volume → copy snapshot to target region → restore volume there.
- **Multi-attach conditions (all three must hold):** (1) supported volume type, (2) same AZ for all instances, (3) cluster-aware application that coordinates writes.

> [!tip] Exam Tip
> Pattern-match: *"instance needs access to a volume in another AZ"* → **snapshot → restore in target AZ** (direct cross-AZ attachment is impossible). *"cross-region disaster recovery for EBS data"* → **snapshot → copy snapshot to target region → create volume from copy**. *"two instances share an EBS volume"* → **multi-attach**, but only with supported volume types, same AZ, and cluster-aware software. *"where does the EXT4 / NTFS file system on an EBS volume come from?"* → **the EC2 instance's OS** — EBS only provides the raw block device.

> [!warning] Gotcha — EBS replication is within an AZ only
> EBS replicates data across multiple physical disks *inside* the AZ — protecting against single-disk failure. It does **not** protect against an AZ-wide failure. The only protection against AZ failure is to snapshot the volume (which lands in S3 across 3+ AZs) and be prepared to restore in a different AZ.

### EBS Snapshots

See [EBS — Snapshots](../services/ebs.md#snapshots--crossing-the-az-and-region-boundaries) for full details.

- **First snapshot = full copy of used data only** (not allocated size). A 40 GB volume with 10 GB written → first snapshot bills for 10 GB.
- **Subsequent snapshots = incremental** — only changed/new data since the previous snapshot.
- **Incrementals are self-sufficient** — deleting any snapshot is safe; AWS relocates referenced data so later snapshots remain restorable.
- **Billing: GB-month on used data**, not allocated size. Frequent snapshots stay cheap because only deltas accumulate.
- **Restore is lazy** — blocks pulled from S3 on first access; unrestored blocks read at lower performance until warmed.
- **`dd`**: reads every block to force EBS to pull all data from S3. Free, manual warm-up.
- **FSR (Fast Snapshot Restore)**: makes restores instant. Extra cost. **Limit: 50 sets per region**, counted **per snapshot + AZ combination** (one snapshot × 4 AZs = 4 sets used).

> [!tip] Exam Tip
> Pattern-match: *"first snapshot size"* → **used data, not allocated volume size**. *"delete an incremental snapshot"* → **safe** — AWS preserves later snapshots automatically. *"slow I/O right after restoring from a snapshot"* → **lazy restore** — pre-warm with `dd` (free) or enable FSR (instant, costs extra). *"FSR limit"* → **50 per region**, counted per **snapshot + AZ**. *"billing for frequent snapshots"* → cheap — only deltas are stored; aggressive schedules don't scale cost the way you'd expect.

> [!warning] Common Snapshot Gotchas
>
> - **Volume bills on allocated size; snapshot bills on used data.** Same 40 GB volume, two different billing bases.
> - **FSR limit is per snapshot + AZ.** 1 snapshot × 4 AZs = 4 FSR sets consumed from your 50-set budget.
> - **Restore creates a new volume — not an in-place recovery.** The original volume and the restored copy coexist until you delete one.
> - **Snapshot copy is needed before cross-region restore.** You cannot restore a snapshot directly into a different region — copy it first.

### EBS Volume Types — GP2 vs GP3

See [EBS — Elastic Block Store](../services/ebs.md) for full details.

**The single biggest conceptual difference:** GP2 is credit-based (performance depends on volume size); GP3 is flat-rate (same baseline regardless of size).

**GP2 key numbers:**

- Baseline: **3 IOPS/GB**, minimum floor **100 IOPS**, ceiling **16,000 IOPS**
- Crossover points: **33.33 GB** (formula beats floor) → **1 TB** (baseline = burst; credit system irrelevant) → **5.33 TB** (max IOPS reached)
- Burst: **3,000 IOPS** via a **5.4 million credit bucket** — new volumes start full (30 min @ 3,000 IOPS)
- Max throughput: **250 MB/s**

**GP3 key numbers:**

- Standard baseline (every volume, every size): **3,000 IOPS + 125 MB/s** — no formula, no bucket
- Provisioned extras (extra cost, independently configurable): up to **16,000 IOPS** + up to **1,000 MB/s**
- Base price: **~20% cheaper than GP2**
- **4× the max throughput** of GP2

**Migration:** use **Elastic Volumes** to switch GP2 ↔ GP3 with no downtime — but after moving to GP3, explicitly provision the IOPS/throughput you need above the 3,000/125 MB/s standard; GP3 does not auto-scale with volume size.

> [!tip] Exam Tip
> Pattern-match: *"small EBS volume has inconsistent, degrading IOPS over time"* → **GP2 credit exhaustion** — switch to GP3 or size the volume larger for a higher baseline. *"need to set IOPS independently of volume size"* → **GP3** (or Provisioned IOPS). *"cost-optimise existing GP2 volumes"* → migrate to GP3 (~20% cheaper base). *"4× throughput compared to GP2"* → GP3 max is 1,000 MB/s vs GP2's 250 MB/s. *"boot volume / general workload / no special requirement"* → GP3 (current AWS default).

> [!warning] Gotcha — GP2 credit trap
> A workload that **consistently exceeds the GP2 baseline** (e.g. 3,000 IOPS on a 100 GB volume whose baseline is only 300 IOPS) will eventually drain the credit bucket to zero. Once empty, performance collapses to the 300 IOPS baseline — not the 3,000 burst rate — and stays there until credits rebuild. Monitor `BurstBalance` CloudWatch metric for GP2 volumes.

> [!warning] Gotcha — GP2 → GP3 migration
> After migrating from GP2 to GP3, you lose the automatic size-based IOPS scaling. A 1 TB GP2 volume had a 3,000 IOPS baseline by formula; a 1 TB GP3 volume has a 3,000 IOPS baseline by standard — but a 2 TB GP3 volume is still only 3,000 IOPS unless you explicitly provision more. Always confirm IOPS requirements before migrating large volumes.

### EBS Volume Types — Provisioned IOPS SSD (IO1, IO2, IO2 Block Express)

See [EBS — Provisioned IOPS SSD](../services/ebs.md#provisioned-iops-ssd--io1-io2-io2-block-express) for full details.

The Provisioned IOPS SSD family is for workloads that need **extreme, consistent, low-jitter performance** — I/O-intensive relational and NoSQL databases where average latency isn't enough; you need it *predictable*.

**Key numbers to know:**

| Attribute | IO1 | IO2 | IO2 Block Express |
| --- | --- | --- | --- |
| Max IOPS per volume | 64,000 | 64,000 | **256,000** |
| Max throughput per volume | 1,000 MB/s | 1,000 MB/s | **4,000 MB/s** |
| Max IOPS/GB ratio | 50 | 500 | 1,000 |
| Per-instance max IOPS | 260,000 | 160,000 | 260,000 |
| Size range | 4 GB – 16 TB | 4 GB – 16 TB | 4 GB – **64 TB** |

**The two-ceiling rule (most exam-tested concept):**

- **Per-volume ceiling**: what a single volume can deliver (IO1/IO2 = 64,000 IOPS).
- **Per-instance ceiling**: what can flow into one EC2 instance from all attached volumes combined (IO1 = 260,000 IOPS). These are separate ceilings — to saturate the per-instance ceiling you need multiple volumes in parallel.
- The per-instance ceiling is further capped by **EC2 instance type and size** — only the newest, largest instances reach these peaks. **All three must align: volume type + instance type + instance size.**

**IOPS/GB ratio caps:** IOPS are decoupled from size, but bounded by the ratio. Example: a 10 GB IO2 volume caps at 5,000 IOPS (500 × 10), not 64,000. To reach 64,000 IOPS you need at least 128 GB.

> [!tip] Exam Tip
> Pattern-match: *"need more than 16,000 IOPS"* → **IO1/IO2** (or Block Express). *"consistent, predictable sub-millisecond latency for a database"* → **IO1/IO2** (GP2/GP3 burst model doesn't guarantee consistency). *"high IOPS on a small volume"* → **IO2** (500 IOPS/GB ratio vs GP2's 3 IOPS/GB). *"per-instance performance ceiling"* → remember it exists separately from per-volume; multiple volumes needed to saturate it. *"256,000 IOPS / 64 TB volume"* → **IO2 Block Express**. *"extreme performance but I don't need persistence"* → [**Instance Store**](../services/ec2.md#storage-options) beats even Block Express, but data is ephemeral.

> [!warning] Gotcha — IO2 per-instance ceiling is lower than IO1
> IO2's per-instance ceiling is **160,000 IOPS** — lower than IO1's **260,000 IOPS**. This seems counterintuitive (IO2 is the newer type). AWS split the headroom: IO2 standard sits at 160k, while the premium capability went into Block Express (which matches IO1's 260k). Don't assume "newer = higher per-instance ceiling" for IO1 vs IO2 standard.

> [!warning] Gotcha — minimum size is 4 GB, not 1 GB
> GP2 and GP3 start at **1 GB**. IO1, IO2, and Block Express all start at **4 GB**. A question about a very small volume with extreme IOPS requirements needs both the ratio math and the 4 GB floor check.

### EBS Volume Types — HDD (ST1, SC1)

See [EBS — HDD-Based Volumes](../services/ebs.md#hdd-based-volumes--st1-and-sc1) for full details.

The HDD family is for **sequential, throughput-oriented workloads where cost matters more than IOPS**. HDDs have physical moving parts — appropriate only when access patterns play to their strength (large sequential I/O).

**Key numbers:**

| Attribute | ST1 (Throughput Optimized) | SC1 (Cold HDD) |
| --- | --- | --- |
| Relative cost | Cheap | **Cheapest EBS storage of all** |
| Size range | 125 GB – 16 TB | 125 GB – 16 TB |
| I/O block size | **1 MB** (vs 16 KB on SSD) | **1 MB** |
| Max IOPS | 500 | 250 |
| Max throughput | 500 MB/s | 250 MB/s |
| Baseline | 40 MB/s per TB | 12 MB/s per TB |
| Burst | 250 MB/s per TB | 80 MB/s per TB |
| Access pattern | Frequent, sequential | Infrequent, cold, archival |
| Typical workloads | Big data, data warehouses, log processing | Cold data, archives, few scans/day |

**1 MB I/O block rule:** on HDD, 1 IOPS = 1 MB — so the IOPS and MB/s ceilings are always the same number (500 IOPS = 500 MB/s). The credit bucket scales in **MB/s per TB** (not IOPS per GB like GP2).

> [!tip] Exam Tip
> Two HDD discriminators: (1) **random vs sequential** — the moment the question mentions IOPS, random access, databases, or low latency, HDD is ruled out. (2) **frequent vs infrequent** — ST1 for frequently accessed sequential data (big data, logs, warehouses); SC1 for cold/rarely accessed archival data. SC1 = the lowest-cost EBS storage overall.

> [!warning] Gotcha — 125 GB minimum and no boot volumes
> Both ST1 and SC1 have a **125 GB minimum** — much larger than GP2/GP3's 1 GB minimum. Neither can be used as a boot volume. A question describing a small volume, a boot volume, or a random-access workload cannot use HDD even if cost is the primary driver.

### EBS Encryption

See [EBS — Encryption](../services/ebs.md#ebs-encryption) for full details.

- **EBS encryption is envelope encryption built on [KMS](../services/kms.md)** — the KMS key wraps a per-volume DEK; the DEK does the bulk crypto on the EC2 host with **AES-256**. The KMS key itself **never encrypts volume data**
- **API used at volume create**: **`GenerateDataKeyWithoutPlaintext`** — EBS receives only the encrypted DEK and stores it in the volume's metadata. The plaintext DEK is fetched via a separate **`Decrypt`** call at first attach and lives **only in EC2 host memory**, never on disk
- **Transparent to the guest OS** — the file system sees plaintext bytes. If a question requires the **OS to hold the keys**, that's **software disk encryption** (LUKS / BitLocker), not EBS encryption
- **No cost for encryption itself, no measurable performance penalty** — enable by default. Only [KMS request charges](../services/kms.md) for the create-time and attach-time API calls
- **AWS-managed default key** `aws/ebs` works for most cases; a **[Customer Managed Key](../services/kms.md#key-ownership-and-management)** is required for custom key policy, rotation control, [grants](../services/kms.md#key-policy--grants), or cross-account sharing of encrypted snapshots
- **Account-level default encryption** is a per-region switch — turn it on in every region to enforce "no unencrypted EBS volumes"
- **Encryption is irreversible** — no console / CLI / API removes it. Going *from* encrypted *to* unencrypted requires an OS-level copy onto a separate unencrypted volume. Going the other direction (encrypting an existing unencrypted volume) uses **`CopySnapshot` with encryption enabled**

#### DEK reuse — the chain rule

| Resource | DEK |
| --- | --- |
| Two volumes created **from scratch** | **Different** DEKs |
| Snapshot of an encrypted volume | **Same DEK** as source |
| Volume restored from that snapshot | **Same DEK** as snapshot (and source) |

A single DEK threads through: **original volume → its snapshots → restored volumes**. Re-keying requires `CopySnapshot` with a different KMS key.

> [!tip] Exam Tip
> Pattern-match: *"how does KMS encrypt a 1 TB volume given the 4 KB ceiling"* → **it doesn't** — EBS does envelope encryption with a DEK; KMS only wraps the DEK. *"encrypt all new volumes in an account"* → **account-level default encryption** (per region). *"add encryption to an existing volume"* → snapshot → `CopySnapshot` with KMS key → restore. *"remove encryption"* → only via OS-level data copy onto an unencrypted volume — there is no API for it. *"OS must hold the keys / hypervisor not trusted"* → **software disk encryption** (LUKS/BitLocker) inside the guest, *not* EBS encryption. *"how many DEKs after N snapshots + M restored volumes of one source"* → **one**.

> [!warning] Gotcha — the KMS key does not encrypt the data
> Common exam confusion: the KMS key encrypts the **per-volume DEK**, not the volume contents. AES-256 happens on the EC2 host using the DEK. This is exactly why the [4 KB direct-encrypt limit](../services/kms.md#direct-encrypt--decrypt--4-kb) doesn't apply to multi-TB EBS volumes.

> [!warning] Gotcha — irreversibility
> Encryption is a **one-way door** like [versioning enable](../services/s3.md#state-machine--one-way-door) and [Object Lock](../services/s3.md#object-lock). Once on, no `DisableEncryption` API exists. Confirm KMS key choice and account-level defaults before creating production volumes.

> [!warning] Gotcha — KMS key region scope flows to EBS encryption
> KMS keys are **regional**. The account-level default-encryption setting and any customer-managed key referenced by EBS must exist in **every region** where you want encrypted volumes. Copying an encrypted snapshot to another region requires the destination region to have access to a KMS key (the source key's region won't decrypt it).

## RDS

See [RDS](../services/rds.md) for full details.

### Databases on EC2

- Running a DB on **[EC2](../services/ec2.md) is always architecturally worse** — the only question is whether a specific need outweighs it. It's **single-AZ by nature** (instance + EBS volume in one AZ), so it loses every HA/resilience scenario
- The **valid** reasons "DB on EC2" is the right exam answer: **engine or version AWS doesn't offer**, a **specific OS + DB version combo**, an **AWS-unsupported replication architecture**, or a genuine **OS/root-access** requirement (and even that is often met by **RDS Custom**)
- Same-AZ **private-IP** traffic is free; **cross-AZ** traffic is charged — relevant to split app/DB designs and Multi-AZ

### RDS core facts

- RDS = **"database server as a service"**, not "database as a service" — **one instance hosts multiple databases**. This phrasing is a deliberate exam trap
- **No SSH / no OS access** — exception: **RDS Custom**
- **Engines**: MySQL, MariaDB, PostgreSQL, Oracle, SQL Server (the last two carry licensing cost)
- RDS runs **inside a [VPC](../services/vpc.md)** (not a public-zone service); you **must pick a DB subnet group** at launch — a list of subnets RDS may use. Multi-AZ places primary + standby in **different AZs**, chosen at random unless specified
- **Each instance has its own dedicated [EBS](../services/ebs.md) storage** (primary and standby each separate) — relational DBs here use the **[ACID model](../concepts/db-transaction-model.md)**
- Billing is **resource-allocation** (like EC2), hourly rate **billed per second** — not usage-based. Snapshot storage **equal to DB storage is free**

### Multi-AZ vs Read Replicas

| | Standby (Multi-AZ) | Read Replica |
| --- | --- | --- |
| Replication | **Synchronous** | **Asynchronous** |
| Scope | Same region, different AZ | Same- **or cross-region** |
| Purpose | **HA / failover** | **Read scaling + DR** |
| App-readable? | **No** | **Yes** (own endpoint) |
| Automatic failover? | **Yes** (CNAME repoint) | **No** — manual **promotion** only |
| App changes needed? | No (same CNAME) | **Yes** — point reads at the replica endpoint |

- **Count:** **5** read replicas per instance (the long-standing/course number) or **up to 15** in current AWS docs — know both. **Cascading** replicas (replicas of replicas) are allowed but **lag compounds** at each hop
- **RPO/RTO:** read replicas give **near-zero RPO and RTO** via fast **promotion** to a standalone read/write instance — but only for **failure recovery**. For **data corruption** the replica holds a copy of the corrupted data, so it's **no defence** (use point-in-time backups instead). Backups improve **RPO**; read replicas improve **RTO**
- Async means a small **replication lag** (seconds); cross-region replication is **AWS-managed and encrypted in transit**

> [!tip] Exam Tip
> Pattern-match: *"engine/version AWS doesn't support"* or *"OS/root access required"* → **DB on EC2** (one of the rare cases it's correct). *"automatic failover to an identical copy"* → **Multi-AZ standby (synchronous)**. *"offload read traffic"* or *"read scaling across regions"* → **read replica (asynchronous)**. *"promote to a standalone instance"* / *"low RTO recovery"* → **read replica**, but *"recover from data corruption / deleted data"* → **NOT a replica** (it replicated the bad data) → **PITR backup**. *"app has no knowledge of the replica"* / *"separate endpoint"* → read replicas need **application changes**. *"multiple databases on one server"* → RDS (it's a server, not a single DB). *"backups without impacting the primary"* → Multi-AZ runs backups **from the standby**.

### Multi-AZ Instance vs Cluster

There are **two** Multi-AZ architectures — both **in-region HA only** (not DR), both **not** in the free tier:

| | Multi-AZ **Instance** | Multi-AZ **Cluster** |
| --- | --- | --- |
| Topology | 1 primary + **1 standby** (2 AZs) | 1 writer + **2 readers** (3 AZs) |
| Standby readable? | **No** — passive failover target | **Yes** — readers serve reads |
| Read scaling? | No (need a read replica) | **Yes** via the reader endpoint |
| Replication | **Synchronous**, storage-level | **Semi-synchronous** quorum (≥ 1 reader ack), transaction-log |
| Failover | **60–120 s** (DNS repoint) | **~35 s** floor / **25–75 s** typical (no crash recovery) |
| Storage / HW | EBS, standard | **Local NVMe SSD flushed to EBS**, **Graviton** |
| Engines | All five (incl. **SQL Server / Oracle**) | **MySQL & PostgreSQL only** |
| Endpoints | Single **database CNAME** | **Cluster** (writer) / **Reader** / **Instance** (test-only) |

- Instance-mode failover is a **DNS repoint**, not a data-recovery op — the standby already holds a synchronous copy, so the 60–120 s is DNS propagation + client caching. **Disable app-side DNS caching** to shorten it
- Cluster-mode replication is **semi-synchronous** in AWS docs (commit on **≥ 1 of 2** standbys), even though some material says "synchronous" — know both framings
- Cluster readers behave like **read replicas** (serve reads, exhibit per-reader replica lag) but replicate **semi-synchronously** rather than asynchronously — so a cluster delivers read scaling **and** a quorum-backed commit in one HA deployment, where a Multi-AZ *instance* would need separate async read replicas bolted on
- Cluster's **Instance endpoint** points at one specific instance and **does not fail over** — testing/fault-finding only, never production

> [!tip] Exam Tip
> Trigger words: *"readable standby"* / *"faster failover"* / *"lower commit (write) latency"* → **Multi-AZ cluster**. *"SQL Server or Oracle + Multi-AZ"* → must be **instance** mode (cluster is MySQL/PostgreSQL only). *"read scaling"* → **cluster or read replica, never instance mode**. *"Multi-AZ for cross-region / DR"* → wrong — Multi-AZ is in-region; use a **cross-region read replica**.

> [!warning] Gotcha — Aurora ≠ RDS
> **Amazon Aurora** is a separate AWS-built engine with a **shared-cluster storage** architecture — *not* the dedicated-EBS-per-instance model RDS uses (cluster mode still keeps **per-instance local storage**, and caps at exactly **2 readers** where Aurora allows more). Don't carry RDS storage facts over to Aurora questions.

### Backups, Snapshots & Restore

Two mechanisms, both stored in **AWS-managed [S3](../services/s3.md)** (regionally resilient; **invisible in your own S3 console** — exam trap):

| | Manual snapshot | Automated backup |
| --- | --- | --- |
| Trigger | **You** (or a script) | **RDS, once/day** in a backup window |
| Extra stream | None | **Transaction logs every 5 min** |
| Retention | **Never expires** — manual delete only; **survives instance deletion** | **0–35 days** (0 = disabled) |
| PITR | ❌ restores only to its **fixed moment** | ✅ **~5-min Recovery Point Objective (RPO)** point-in-time recovery |
| Scope | **Instance-level** (all DBs) | Instance-level |

- **First snapshot is full, the rest incremental** — true for **both** manual snapshots **and** automated backups (AWS treats an automated backup as an automated snapshot)
- **PITR works** by restoring the closest snapshot then **replaying transaction logs** to the chosen time — so it requires automated backups; a manual snapshot alone can't do it
- **Keep data past the 35-day max on deletion** → take a **final manual snapshot**. "Retain automated backups on delete" **still expires** them
- **Cross-region backup replication** (snapshots + logs) is **opt-in, not default**, and costs extra
- A **restore always creates a new instance with a new endpoint** — re-point apps; it never overwrites the source. Restore **time is your Recovery Time Objective (RTO)** (slow for large DBs); backups improve **RPO**, **read replicas** improve **RTO**
- On **Multi-AZ instance** mode backups run from the **standby** (no primary I/O hit); on **Single-AZ** the sole instance takes a brief I/O pause

> [!tip] Exam Tip
> Pattern-match: *"backups not in my S3 buckets"* → **AWS-managed, RDS console only**. *"restore to the exact moment before failure"* → **automated-backup PITR**, not a manual snapshot. *"keep the data after deleting the instance / past 35 days"* → **final manual snapshot**. *"first backup full, then incremental"* → **both** snapshot types. *"5 minutes"* → **transaction-log interval** / achievable PITR RPO. *"replicate backups to another region"* → opt-in, extra cost. *"restore is slow / improve recovery time (RTO)"* → that's the **restore window**; use a **read replica**, not bigger/faster backups.

### Data Security

Four separate concerns — keep **authentication vs authorization** and **in-transit vs at-rest** apart (see [RDS — Data Security](../services/rds.md#data-security)):

- **Encryption in transit** — **SSL/TLS**, all engines, can be made **mandatory per user**. Trigger: *"force/require SSL"*
- **Encryption at rest, default** — **KMS + EBS**, done **by the host** below the engine (the engine writes plaintext, unaware). A **KMS key** (customer- or AWS-managed `aws/rds`) generates **DEKs** ([envelope encryption](../services/kms.md), **AES-256**). Storage, logs, snapshots, and replicas all use the **same key**
- **Encryption at rest, engine-native** — **TDE**, **SQL Server and Oracle only**. **MySQL / MariaDB / PostgreSQL cannot use TDE** (they still get at-rest protection via the host-side KMS/EBS path)
- **No AWS in the chain of trust** — **RDS Oracle + TDE + CloudHSM** (you hold the keys; AWS never sees them). Oracle-specific
- **IAM authentication** — log in with a **15-minute token** from **`GenerateDBAuthToken`** instead of a DB password. **Authentication only — never authorization**: once in, the **local DB user** still governs permissions. Must be explicitly enabled; **TLS is mandatory** (token is sent as a cleartext password); mint a **fresh token per new connection**

| At-rest mechanism | Who encrypts | Engines |
| --- | --- | --- |
| **KMS + EBS** (default) | **The host**, below the engine | **All** engines |
| **TDE** | **The DB engine** itself | **SQL Server & Oracle only** |
| **TDE + CloudHSM** | The engine, with **your** keys | **Oracle only** |

> [!tip] Exam Tip
> Pattern-match: *"force/require SSL"* → **per-user mandatory TLS**. *"default at-rest, host does the encryption"* → **KMS + EBS** (KMS key → DEKs, AES-256). *"engine-native encryption"* → **TDE** (SQL Server/Oracle only — never MySQL/MariaDB/PostgreSQL). *"AWS must have no access to the keys"* → **Oracle + TDE + CloudHSM**. *"log in without a stored DB password using an AWS identity"* → **IAM authentication** (`GenerateDBAuthToken`, 15-min token). *"does IAM decide what the user can do in the DB?"* → **No, the local DB user does**.

> [!warning] Gotchas
>
> - **IAM auth ≠ authorization** — it only authenticates; the local DB user still controls permissions. The single most-tested point here
> - **TDE is restricted to SQL Server and Oracle** — "MySQL is encrypted" does **not** mean MySQL supports TDE (it uses host-side KMS/EBS instead)
> - **Encryption can't be removed, and can't be added in place** after creation — only via **snapshot → copy-with-encryption → restore**; a read replica's encryption state must **match its source**
> - **CloudHSM "no AWS trust" option is RDS Oracle-specific** — don't generalize it to other engines
> - **Don't confuse the KMS key (CMK) with the DEK** — the KMS key *generates* DEKs; the DEKs do the actual bulk AES-256 encryption

## Database Transaction Models — ACID & BASE

See [Database Transaction Models (ACID/BASE/CAP)](../concepts/db-transaction-model.md) for full details.

- **CAP theorem**: Consistency, Availability, Partition tolerance — a DB can deliver **two of three**. Once a network partition is in play, the real choice is **consistency vs availability**
- **ACID → consistency** (atomic, consistent, isolated, durable). Rigid rules **limit scalability**. Exam default: **ACID = [RDS](../services/rds.md) / relational** (financial systems)
- **BASE → availability** (basically available, soft state, eventually consistent). Highly scalable; pushes consistency handling to the application. Exam default: **BASE = NoSQL / DynamoDB**
- **DynamoDB straddles both**: eventually consistent **and** strongly consistent reads (app chooses), plus **DynamoDB Transactions** for ACID guarantees

> [!tip] Exam Tip
> Trigger words: *"ACID"* → relational / RDS. *"BASE" / "eventually consistent" / "soft state"* → NoSQL / DynamoDB. *"DynamoDB"* **with** *"ACID" or "transactions"* → **DynamoDB Transactions**. Watch the wording trap: **ACID's "C" (valid-state rules) is not CAP's "C" (read sees latest write)**.

## Scaling — Horizontal vs Vertical

See [Scaling — Horizontal vs Vertical](../patterns/scaling.md) for the full treatment.

- **Vertical scaling = resize one instance.** Simple, works for monoliths, no app changes. But requires **stop / modify / start** (disruption), only happens in maintenance windows, and is **capped at the largest available instance size**.
- **Horizontal scaling = add more instances behind a load balancer.** No upper limit, no disruption, finer granularity, and usually cheaper per unit of capacity. Requires a **load balancer** and **stateless application design**.
- **Sessions are the critical constraint.** With horizontal scaling, a customer's consecutive requests can hit different instances. If session state lives on the instance, the user appears logged out, the cart empties, the wizard restarts. **Move sessions off-host** to ElastiCache (Redis or Memcached), DynamoDB, or RDS.
- **ALB sticky sessions are NOT the proper fix.** They pin a user to one instance; if that instance is replaced or scaled in, the session is lost anyway. Sticky sessions are a migration aid for legacy apps. Exam-correct long-term answer: **off-host sessions, stateless instances**.
- **Resizing EC2 is a stop/start in disguise** — instance-store data destroyed, auto-public-IPv4 changes. See [EC2 — Stop/Start vs Restart](../services/ec2.md#stopstart-vs-restart--these-are-not-the-same).
- **Horizontal scaling is the foundation for HA.** Vertical scaling cannot reach fault tolerance — a single instance always inherits the resilience tier of its AZ. Horizontal + ELB + Auto Scaling Groups across multiple AZs is the HA pattern.

### Scaling — pattern-match cheat-sheet

| Question pattern | Answer |
| --- | --- |
| "App is a monolith on one instance, traffic doubled" | **Vertical scaling** — resize to a larger instance type |
| "Cannot tolerate downtime, need to scale during business hours" | **Horizontal scaling** — adding instances doesn't disrupt existing connections |
| "Users get logged out at random after deploying behind ALB" | **Move sessions off-host** to ElastiCache / DynamoDB, not sticky sessions |
| "Shopping cart empties intermittently across instances" | **Stateless app + external session store** |
| "Legacy app, on-host sessions, no time to refactor" | **ALB sticky sessions** — migration aid only |
| "Need infinite scale" | **Horizontal** — vertical has a hard cap |
| "Need fault tolerance, not just scalability" | **Horizontal scaling across multiple AZs** — load balancer + Auto Scaling Group; vertical can't reach this |
| "Cheapest at large total capacity" | **Many small commodity instances (horizontal)** — large instances carry a price premium |

> [!tip] Mnemonic
> **Many Bobs = horizontal. One giant Bob = vertical.** Adding workers = scale out; growing one worker = scale up.

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
- **PHZ namespace replacement (gotcha)**: a private zone is **not** an overlay — once it matches a query name (identical or **parent** match), the VPC Resolver answers **only** from that zone. A missing record returns **NXDOMAIN**, *not* the public answer. So an apex PHZ for `example.com` must duplicate **every** public name still needed inside the VPC (e.g. `customer.example.com`), or those names go dark internally. Public fall-through happens **only when no associated PHZ owns the namespace**, never per-record. (Escape hatches: a Resolver rule for the same domain **overrides** the PHZ; NS records in the PHZ can delegate a subdomain.) See [How association and namespace resolution work](../services/route53.md#how-association-and-namespace-resolution-work).
- TTL = how long resolvers cache a record; **lower TTL before migrations**, raise after

### How DNS Resolves (Walking the Tree)

See [DNS — Resolution Chain](../concepts/dns.md#resolution-chain) for the full walk and diagrams.

- DNS's whole job is to **find the one zone** that can give an authoritative answer; the climb from root down to the domain is called **walking the tree**, and **every query hands you the next step** (a referral *is* the NS records for the tier below)
- The recursive **resolver** (home router or ISP) does the walk on your behalf and **caches** the result; the client only ever talks to the resolver

| Trigger phrase | Answer |
| --- | --- |
| "static name-to-IP mapping that overrides DNS, checked first" | **hosts file** (`/etc/hosts`) |
| "queries on your behalf, runs on a router or ISP, caches" | **recursive resolver** |
| "hard-coded IPs maintained by the OS vendor, not looked up" | **root servers** |
| "points `.com` at VeriSign / points a domain at its nameservers" | **NS records = delegation** |
| "the definitive answer from the source-of-truth nameserver" | **authoritative** |
| "a cached copy at a resolver, may be stale" | **non-authoritative** (resolvers always label answers this way) |

> [!warning] Gotcha
> An authoritative answer is **not always an IP** — it can be a **CNAME** (another name), forcing the resolver to **walk the tree again** for that name. Chained CNAMEs stack latency and are a real cause of slow apps. And the **hosts file overrides DNS**: it's checked before any query, so a stale entry there is the classic *"why isn't my DNS change working?"* trap.

### Registrar vs DNS Hosting (Interoperability)

See [Registrar vs DNS Hosting](../services/route53.md#registrar-vs-dns-hosting-interoperability) for full details.

- "Registering a domain" is **two independent roles**, each can live with a different provider:
  - **Registrar** = registers the domain with the TLD **registry**; **one-off per-year** (or per-3-year) fee; console **Registered Domains**
  - **DNS Hosting** = the **public hosted zone** (4 name servers + zone file); **monthly** fee; console **Public Hosted Zone**
- **Registry ≠ Registrar**: the registry *owns* the TLD zone (`.org` = **PIR**); the registrar *registers with* it on your behalf
- **NS delegation records** are the handshake — the registry always points the domain at its 4 name servers; present in every configuration
- Three valid arrangements:
  - **Both via R53** — the default
  - **R53 registrar only** (3rd party hosts) — valid but **rare/discouraged**; you lose R53's main value (its DNS hosting)
  - **R53 hosting only** (3rd party registers) — **common**; create a public hosted zone in R53, give the 4 NS to the external registrar

> [!tip] Exam Tip
> **Migrating an old domain registered elsewhere to R53 DNS** = R53 **hosting only**, done *after the fact*: create a public hosted zone in R53 and update the **NS records at the existing registrar** to the 4 R53 name servers. No re-registration. Triggers: "domain registered with GoDaddy/Hover but we want R53 to manage records", "keep the existing registrar".

> [!warning] Gotcha
> Registration fee (per-year, to the registrar) and hosting fee (monthly, per hosted zone, to the host) are **separate fees**. And R53 registrar-only is the trap answer when a question implies it delivers R53's core benefit — it doesn't; the value is in **hosting**.

### DNS Record Types

See [DNS — Record Types](../concepts/dns.md#dns-record-types) for full details and diagrams.

| Record | Maps / does | Trigger keywords |
| --- | --- | --- |
| **NS** | Delegation — parent zone points at the child zone's nameservers | "delegation", "points at the servers hosting the zone" |
| **A** | Hostname to **IPv4** | "hostname to IPv4" |
| **AAAA** | Hostname to **IPv6** | "hostname to IPv6" |
| **CNAME** | Host to **another host (name)** | "alias", "shortcut", "reduce admin overhead" — ⚠️ never an IP |
| **MX** | Find the **mail server (SMTP)** for a domain | "mail server", "email delivery" |
| **TXT** | Arbitrary text on a domain | "prove domain ownership", "verification", "SPF / anti-spam" |

- **CNAME → name only, never an IP**, and never at the **zone apex** — use a Route 53 **Alias** for the apex case
- **MX: lower priority number = higher priority** (tried first). A value with **no trailing dot** is a host inside the same zone (`mail` means `mail.example.com`); a value **with a trailing dot** is an FQDN that can point to an external provider (Office 365, etc.). The host name itself is arbitrary
- **TXT** is the answer for **domain-ownership verification** (the external service hands you a string to publish) and **SPF/anti-spam** sender authorization
- **NS records are delegation** at every level (root → TLD → domain) — the answer whenever a question describes a parent zone pointing at the servers that host a child zone
- **Authoritative** answer = from the domain's source-of-truth nameserver (slower, always current); **non-authoritative** = a cached copy at a resolver (faster, can be stale until TTL expires)

> [!warning] Gotcha
> Two record-type traps to lock in: **(1)** a CNAME can only point at another name — if an answer has a CNAME pointing straight at an IP, it's wrong. **(2)** MX priority is "lowest number wins", not highest. And the TTL trap: migrating email with a **high MX TTL** leaves old mail servers cached and breaks delivery — **lower the TTL days in advance** of any DNS migration.

### Route 53 Routing Policies

| Policy | When to use | Health checks? |
| --- | --- | --- |
| Simple | Single endpoint | No |
| Weighted | A/B testing, gradual traffic shifts, simple load balancing | Optional |
| Latency | Route to lowest-latency AWS region (performance) | Optional |
| Failover | Active/passive HA; primary + secondary | Required on primary |
| Geolocation | Restrict/route by user's state/country/continent | Optional |
| Geoproximity | Closest resource by physical distance, tunable with bias | Optional |
| Multi-value | Return multiple IPs; improve availability (not a load balancer) | Optional, per record |

- **Weighted**: each same-name record gets a weight; returned with probability `weight ÷ total weight` — weights **need not sum to 100**. Weight **0** = never returned, **unless all are 0** (then all eligible). Health checks **don't change the denominator**: an unhealthy selected record is skipped and the draw repeats; its share is **not** redistributed. Triggers: "percentage of traffic", "test a new version", "send 5 % to the new build".
- **Latency**: each record tagged with an **AWS region** (one record per name per region); AWS holds a **latency database** between user location and regions. Returns the **lowest-estimated-latency healthy** record; if unhealthy, the next-lowest. Latency data is **estimated / not real-time**. Triggers: "lowest latency", "best performance for a global app".
- **Geolocation vs Geoproximity** — the most-tested distinction:
  - **Geolocation** = match by **tagged location** (US state, country, continent, default). Specificity order State → Country → Continent → Default, stopping at the first match; no match and no default → **"no answer"**. It returns **relevant records only, NEVER the closest** as a fallback. Triggers: "restrict content by region", "block/allow by country", "language-specific content".
  - **Geoproximity** = **physical distance** to the resource (AWS region for AWS resources, **lat/long** for external), plus a **bias**: `+` grows a resource's service area (shrinking neighbours), `-` shrinks it. Only geoproximity lets you **shift traffic away from the geographically closest** resource. Triggers: "closest resource", "bias", "route more/less traffic to a region".
- **Multi-value**: returns **up to 8 healthy** records (random 8 if more exist), each with its **own health check**; client picks one. **Improves availability, is NOT a load balancer** — it doesn't manage connections.

> [!tip] Exam Tip
> **Failover routing** requires a Route 53 **health check** on the primary record. If the primary fails the health check, Route 53 automatically returns the secondary record. It uses **two records of the same name** (one Primary, one Secondary). Trigger phrases: "active/passive", "primary and secondary", "fail over to a backup/maintenance page". The classic pattern is **out-of-band failover** — primary and secondary point at *different* services (e.g. an EC2 primary with an S3 static-page secondary) so the backup survives a platform-level failure of the primary. Don't confuse with **weighted/multi-value**, which *distribute* across healthy resources rather than active/passive switching.

> [!warning] Gotcha
> Two perennial routing traps: (1) **geolocation never returns the nearest record** — only a location-match, the default, or "no answer"; if a question says "closest", that's **geoproximity** or **latency**, not geolocation. (2) **Multi-value is not a load balancer** — it improves availability via DNS but performs no connection management; true load balancing means an **ELB**.

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

### DNSSEC

See [DNSSEC](../concepts/dnssec.md) for the full mechanics, worked example, and `dig` commands.

- **DNSSEC** is a *secure add-on* to DNS (not a replacement): it adds **origin authentication** + **data integrity** via digital signatures, building a **chain of trust** from the root down. It is **additive and backward compatible** — DNS-only clients never see the extra records. The problem it solves is **cache poisoning**
- It **validates**, it does **not correct** — it only tells you whether data is authentic/unmodified, never what the right answer should be
- **DNSSEC validates RRSETs** (all records with the *same name + same type*), **not individual records** — a classic trap
- Key/record vocabulary: **RRSIG** = signature of an RRSET (made by the private **ZSK**); **DNSKEY** = publishes the public keys (**flag 256 = ZSK, 257 = KSK**); **ZSK** signs all zone RRSETs; **KSK** signs *only the DNSKEY RRSET* and is the zone's point of trust; **DS** record in the **parent** holds a **hash of the child's KSK** (the cross-zone trust link)
- The **root KSK has no parent**, so it is a **trust anchor** — explicitly trusted by every validating resolver, distributed out-of-band by IANA, and re-signed at the quarterly **root signing ceremony** (HSMs at El Segundo / Culpeper, Crypto Officers, Verisign as root-zone maintainer)
- Four validation outcomes: **Secure** (chain verified, sets the **AD** flag) · **Insecure** (proven gap, e.g. unsigned zone with no DS) · **Bogus** (signatures fail → **SERVFAIL**) · **Indeterminate** (no trust anchor reachable)
- **On AWS**: [Route 53](../services/route53.md) does **DNSSEC signing** for public hosted zones (you bring an asymmetric **[KMS](../services/kms.md)** key as the KSK in `us-east-1`; Route 53 manages the ZSK; you must add the **DS record to the parent** to complete the chain), and **Route 53 Resolver** can do **DNSSEC validation** per-VPC

Route 53 signing **implementation** specifics (see [Implementing DNSSEC](../services/route53.md#dnssec)):

- The enabling sequence **starts with KMS** (asymmetric key pair backing the KSK), then Route 53 makes the ZSK, publishes the DNSKEY, signs it with the private KSK (`RRSIG DNSKEY`), then you add the parent DS and the CloudWatch alarms
- The KSK's KMS key **must be in `us-east-1`** — a hard requirement regardless of where the zone serves traffic
- **ZSK creation *and rotation* are handled internally by Route 53 — KMS is not involved in the ZSK** (KMS backs only the KSK). This KSK↔KMS / ZSK↔R53-internal split is a frequent exam distinction
- **DS record to the parent**: if the domain is **registered with Route 53**, add it from the **Registered Domains** console (R53 liaises with the TLD); if registered **elsewhere**, you add it **manually** at that registrar
- Create CloudWatch alarms **`DNSSECInternalFailure`** and **`DNSSECKeySigningKeysNeedingAction`** — both flag an urgent KSK/KMS-level problem with the zone
- Optional **per-VPC validation**: on a DNSSEC-enabled zone, a record that fails validation is **not returned** (non-DNSSEC zones are unaffected)

| Trigger phrase | Answer |
| --- | --- |
| "validate DNS data / detect tampered DNS answers" | **DNSSEC** |
| "protect against cache poisoning / spoofed DNS" | **DNSSEC** |
| "all records with the same name and type" | **RRSET** (what DNSSEC signs) |
| "signature of a record set" | **RRSIG** (made by the private ZSK) |
| "parent holds a hash of the child's key" | **DS record** (hash of the child **KSK**) |
| "root key, no parent, explicitly trusted" | **trust anchor** (root KSK) |
| "sign a public hosted zone in AWS" | **Route 53 DNSSEC signing** + a **KMS** KSK + a parent **DS** record |
| "where must the KMS key for the KSK live?" | **`us-east-1`** |
| "who creates/rotates the ZSK in Route 53?" | **Route 53 internally** (KMS not involved) |
| "alarms to set when enabling DNSSEC" | **`DNSSECInternalFailure`**, **`DNSSECKeySigningKeysNeedingAction`** |

> [!warning] DNSSEC Gotchas
>
> - It **validates, never corrects** — and it signs **RRSETs, not individual records**.
> - The **private ZSK is never in the zone**; only the **public** ZSK (via DNSKEY) is. The **DS** record holds a **hash of the KSK**, not the ZSK and not the full key.
> - **Bogus ≠ Insecure**: Bogus means signatures are present but *fail* (returns SERVFAIL); Insecure means a *proven* absence of signing (no DS). "No RRSIG" can mean an unsigned zone (Insecure), a signed **NSEC3** proof of non-existence (still **Secure**), or a CNAME into an unsigned zone (Insecure leaf) — not necessarily a failure.
> - Route 53: enable/disable has a strict **order** (sign before adding the parent DS; remove the DS and wait its TTL before unsigning) or validating resolvers SERVFAIL the whole zone.

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

### Control Tower

See [Control Tower](../services/control-tower.md) for full details.

- **Control Tower = "Organizations with superpowers"** — it **orchestrates** other services (Organizations, IAM Identity Center, CloudFormation, Config, CloudWatch, CloudTrail, SNS, Service Catalog) to stand up a well-architected multi-account environment. It does not reinvent them
- **Four parts:** **Landing Zone** (the environment), **Guardrails/Controls** (governance rules), **Account Factory** (automated account provisioning), **Dashboard** (oversight)
- **Landing Zone** has a **home region** that is **always available and cannot be denied**; other regions can be allowed/denied. Default OUs: **Foundational (Security)** holding the **Audit** + **Log Archive** accounts, and **Custom (Sandbox)**
- **Log Archive account** = isolated, **read-only**, explicitly-granted store of Config + CloudTrail logs from **all enrolled accounts**. **Audit account** = SNS governance/security notifications + CloudWatch monitoring + home for third-party audit tools
- **Guardrails, two independent axes:** enforcement level (**mandatory / strongly recommended / elective**) and functional type (**preventative = SCP, stops the action**; **detective = Config rule, only flags it**). Optional modern third type: **proactive = CloudFormation hooks**, block at deploy time
- **Account Factory** = self-service automated account creation with guardrails auto-applied and cookie-cutter **account + network baselines** (non-overlapping VPC IPs), via console / Service Catalog / API
- **IAM Identity Center** = formerly **AWS SSO**; both federated and internal identities funnel through it for SSO

> [!tip] Exam Tip
> "Quickly set up a **secure, well-architected, multi-account** environment with governance baked in" → **Control Tower** (not raw Organizations). "Self-service standardized account provisioning with guardrails pre-applied" → **Account Factory**. "Isolated read-only log store from every account" → **Log Archive**. "Governance-change notifications / third-party audit tooling" → **Audit account**.

> [!warning] Gotcha
> **Preventative ≠ detective.** Preventative guardrails *stop* actions (SCP); detective guardrails only *identify* non-compliance after the fact (Config rules). Detective does not block. Also: **IAM Identity Center and AWS SSO are the same service**; **guardrails and controls are the same thing** (console rename).

## Cost Optimization

See [Cost Optimization](../patterns/cost-optimization.md) for the decision tree and pooling rules, and [EC2 Purchase Options](../patterns/ec2-purchase-options.md) for the EC2-specific deep dive.

### Purchase Plans

| Plan | Term | Discount | What it covers |
| --- | --- | --- | --- |
| **On-Demand Instances** | None | None | Per-second EC2 compute; no commitment, **no capacity priority** |
| **Compute Savings Plans** | 1 / 3 yr | up to ~66 % | EC2 (any family/Region/OS), Fargate, Lambda |
| **EC2 Instance Savings Plans** | 1 / 3 yr | up to ~72 % | Specific family in a specific Region (size+OS flex) |
| **Standard Reserved Instances** | 1 / 3 yr | up to ~72 % | Family/OS/Region locked; size flexibility only |
| **Convertible Reserved Instances** | 1 / 3 yr | up to ~54 % | Exchangeable for different family/OS/tenancy |
| **Scheduled Reserved Instances** *(retired 2024-09-03)* | 1 yr min | Modest vs On-Demand | Recurring time-window EC2 — **discontinued; use On-Demand Capacity Reservations** |
| **On-Demand Capacity Reservations** | None (immediate-use) | None | Pure capacity guarantee in one AZ; billed even idle |
| **Reserved Capacity** (RDS / ElastiCache / Redshift / OpenSearch / DynamoDB) | 1 / 3 yr | up to ~72 % | The only purchase plan for these services |
| **Spot Instances** | None | up to ~90 % | Spare EC2 capacity, 2-minute interruption warning |
| **Dedicated Instances** | None | Premium (per-Region hourly fee) | No co-tenant hardware; host opaque |
| **Dedicated Hosts** | None or per-host | Pay-per-host | Socket/core-based **BYOL** licensing; Host Affinity |

> [!warning] Gotcha
> **Savings Plans do NOT cover RDS, ElastiCache, Redshift, OpenSearch, or DynamoDB.** For long-running databases, the answer is a **Reserved Instance / Reserved Capacity**, not a Savings Plan.

> [!warning] Gotcha — capacity priority under shortage
> If a Region or AZ runs short on EC2 capacity (e.g. after a major fault), AWS allocates remaining capacity in this order: **Reserved Instances → On-Demand Instances → Spot Instances.** A *discount* (Regional Reserved Instance, Savings Plan) gives you **no** capacity priority — only a Zonal Reserved Instance or an On-Demand Capacity Reservation does.

> [!tip] Exam Tip
> Pattern-match purchase plans: workload moves between EC2/Fargate/Lambda → **Compute Savings Plans**. Pinned `m5` in `us-east-1` → **EC2 Instance Savings Plans** (deeper discount). Long-running RDS → **Reserved DB Instance**. Batch / CI / checkpointable → **Spot Instances**. Many accounts → consolidate via **AWS Organizations** so Savings Plans / Reserved Instances pool automatically. **BYOL per-socket / per-core licensing** → **Dedicated Host** (pay per host, host targeting). **"No other customer's hardware" but don't need to target the host** → **Dedicated Instance**. **Need cheapest 24/7/365 EC2 long-term** → **3-year All-Upfront Standard Reserved Instance**. **Need a Reserved Instance but workload family/OS/tenancy may change** → **Convertible Reserved Instance** (smaller ~54% discount but exchangeable for equal/greater value). **Recurring time window, not 24/7** → historically **Scheduled Reserved Instances** (retired 2024-09-03); for current designs use **On-Demand Capacity Reservations** in the window.

### EC2 Capacity Reservations — discount vs capacity

A reservation has a **billing component** and a **capacity component**, and they are independent.

| Setup | Discount? | Capacity reserved? | Notes |
| --- | --- | --- | --- |
| **Regional Reserved Instance** alone | Yes | No | Flexible across AZs/sizes; no capacity guarantee |
| **On-Demand Capacity Reservation** alone | No | **Yes (one AZ)** | Capacity guaranteed; billed at full On-Demand |
| **Regional Reserved Instance + On-Demand Capacity Reservation** | Yes | **Yes (one AZ)** | The AWS-recommended combo for both benefits |
| **Zonal Reserved Instance** | Yes | **Yes (one AZ)** | Both in one purchase; locked to one AZ |
| **Savings Plan + On-Demand Capacity Reservation** | Yes | **Yes (one AZ)** | Same stacking trick as Regional Reserved Instance |

> [!warning] Trap — discount stacking only works with Regional Reserved Instances (or Savings Plans)
> A **Zonal Reserved Instance's discount does not apply to a separate On-Demand Capacity Reservation** — the Zonal Reserved Instance already includes its own bundled capacity reservation. The "get both discount and capacity" pattern is **Regional Reserved Instance + On-Demand Capacity Reservation** or **Savings Plan + On-Demand Capacity Reservation**.

> [!tip] Exam Tip
> When a question wants **both** a billing discount *and* a capacity guarantee, the canonical answers are: (1) buy a **Zonal Reserved Instance** if you're committed long-term and AZ-locked is fine, or (2) pair a **Regional Reserved Instance** (or Savings Plan) with an **On-Demand Capacity Reservation** if you need AZ/size flexibility plus capacity in one specific AZ.

### EC2 Purchase Options — pattern-match cheat-sheet

| Question pattern | Answer |
| --- | --- |
| "Default — short or unknown duration, cannot tolerate interruption" | **On-Demand Instances** |
| "Cheapest possible — can resume from checkpoint / stateless / batch" | **Spot Instances** |
| "Spot terminates when ___" | Spot price rises **above your maximum** |
| "Two customers both running Spot Instances at different max prices — what do they pay?" | **Same current Spot price** — max only sets termination tolerance |
| "Long-running 24/7/365, lowest possible cost" | **3-year All-Upfront Standard Reserved Instance** |
| "Long-running, but workload family/OS/tenancy might change mid-term" | **Convertible Reserved Instance** (~54% discount but exchangeable) |
| "Convertible RI exchange — can I downgrade?" | **No** — exchange requires equal or greater value |
| "Need EC2 only every Friday for 24 hours" / "recurring time window" | Historically Scheduled Reserved Instances — **retired 2024-09-03**; new answer is **On-Demand Capacity Reservations** in the window |
| "Guaranteed capacity in one AZ, no long-term commitment" | **On-Demand Capacity Reservations** |
| "Discount across families and from EC2 to Fargate/Lambda" | **Compute Savings Plans** |
| "Discount locked to `m5` in `us-east-1`, deepest possible Savings Plan" | **EC2 Instance Savings Plans** |
| "BYOL Oracle / SQL Server / Windows per-socket licensing" | **Dedicated Hosts** |
| "No co-tenant hardware, don't need to target the host" | **Dedicated Instances** |
| "Region-wide capacity shortage — who gets capacity first?" | **Reserved Instances → On-Demand Instances → Spot Instances** |
| "Reserved Instance bought in account A, account A doesn't use it" | Floats automatically to a sibling account with matching usage (default org behaviour) |
| "Unused Reserved Instance or Capacity Reservation costs anything?" | **Yes — billed whether used or not** |

> [!warning] Common purchase-option gotchas
>
> - **On-Demand Instances have no capacity priority.** Discount and capacity are separate — On-Demand Instances and Spot Instances both get whatever Reserved Instances didn't claim
> - **"You always pay the current Spot price, not your maximum."** Higher max = more termination tolerance, *not* a higher price
> - **Reserved Instances are billed if idle.** Plan reservations carefully; an unused 3-year All-Upfront Reserved Instance is wasted money (mitigated within an AWS Organization by automatic floating)
> - **Convertible Reserved Instance exchanges go up, not down.** The new RI(s) must be of equal or greater value than the current one — you cannot downgrade via exchange. Useful for *shape changes* (M-family → Graviton, OS swap, Shared → Dedicated tenancy mid-term), not for shrinking commitments
> - **Scheduled Reserved Instances were retired on 2024-09-03** — no longer purchasable; AWS recommends On-Demand Capacity Reservations instead. Historical minimums were 1,200 hr/year + 1-year commitment. Source: the in-post 2024-09-03 update on the [original launch announcement](https://aws.amazon.com/blogs/aws/new-scheduled-reserved-instances/) (the product shipped with only a 5–10% discount on legacy C3/C4/M4/R3 families — narrow enough that Savings Plans and On-Demand Capacity Reservations made it redundant)
> - **On-Demand Capacity Reservations count toward your per-Region On-Demand Instance limits** even when unused
> - **Zonal Reserved Instance ≠ Regional Reserved Instance + Capacity Reservation** for discount stacking — a Zonal Reserved Instance's discount does NOT apply to a separately-purchased On-Demand Capacity Reservation
> - **"Dedicated" is two things.** Dedicated Host (own the host, BYOL per-socket) vs Dedicated Instance (no co-tenant hardware, host opaque) — see [the tenancy gotcha](../services/ec2.md#tenancy-shared-vs-dedicated)

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

### CloudWatch Agent

- Standard CloudWatch sees an instance **from the outside** (hypervisor CPU, network, disk-ops). It does **not** report in-guest **memory usage**, **disk-space-used**, per-process, or per-core detail, and it captures **no OS or application logs**
- The **CloudWatch Agent** (unified agent) runs **inside** the instance and injects OS-level **metrics** into CloudWatch and **logs** into CloudWatch Logs
- It needs **(1) a config** (which metrics + log files) and **(2) permissions via an [instance role](../services/ec2.md#instance-roles--instance-profiles)** (`CloudWatchAgentServerPolicy`) — never baked-in access keys
- **Log mapping:** one **log group per log file**, one **log stream per instance** (stream defaults to the instance ID) — *group by file, stream by instance*
- Agent OS-level metrics show up under a new **`CWAgent`** namespace — its presence is the fingerprint the agent is installed
- Store the agent config in **[SSM Parameter Store](../services/ssm.md#parameter-store)** once and reuse it across a fleet (pairs with CloudFormation + Auto Scaling); other deploy paths: Systems Manager, AMI baking, user-data bootstrapping

> [!tip] Exam Tip
> Pattern-match: *"alarm on EC2 memory / disk-space used"* → **CloudWatch Agent** (custom metric); standard CloudWatch can't. *"collect /var/log/… or application logs from EC2"* → **CloudWatch Agent**. *"new `CWAgent` namespace appeared"* → the agent is installed. *"agent installed but a specific log never appears"* → **file-permission mismatch** between the log and the agent's run-as user (`cwagent`), not IAM.

> [!warning] Gotcha — run-as user vs IAM
> IAM (the instance role) decides whether the agent can **send** to CloudWatch. **OS file permissions** decide whether its **run-as user** (`cwagent` by default; older setups use `root`) can **read** the source log. Root-only files like `/var/log/secure` are silently skipped under `cwagent`. Missing logs with everything else working → check file permissions, not the role.

## CloudTrail

See [CloudTrail](../services/cloudtrail.md) for full details.

- CloudTrail records **API / account activity** as **events** — who did what, when, from where. This is the **audit** service; [CloudWatch](../services/cloudwatch.md) is the **health/telemetry** service, and **AWS Config** is the **resource-configuration/compliance** service
- **Event History** is on **by default, free, 90 days, management events only** — no S3 storage. For retention beyond 90 days, data events, or log delivery you must create a **trail**
- **Three event types**: **Management** (control-plane, on by default), **Data** (S3 object access, Lambda invokes — **opt-in, extra cost, high volume**), **Insight** (anomaly detection over management-event activity — opt-in)
- **Trail scope**: **single-region** or **all-regions** (the latter auto-adds new regions). All-regions is the recommended default
- **Global service events** — **IAM, STS, CloudFront** (and **Route 53**) have no region, so their events deliver to **`us-east-1`**. The trail setting must be enabled (console default)
- **Destinations**: an **[S3](../services/s3.md) bucket** (compressed JSON, retained indefinitely, optional **SSE-KMS**) and/or **[CloudWatch Logs](../services/cloudwatch.md)** (unlocks search + metric filters + alarms)
- **Organizational trail** — created from the **management account** of an [Organization](../services/organizations.md); captures all member accounts centrally
- **Log file integrity validation** — SHA-256 hashes + hourly **signed digest files** prove logs weren't tampered with
- **Not real-time** — log files are delivered ~every **15 minutes**. For fast automated reactions, match **management events in EventBridge** rather than polling delivered files

> [!tip] Exam Tip
> Pattern-match: *"who made this API call / who deleted the bucket"* → **CloudTrail**. *"keep audit logs longer than 90 days / deliver to S3"* → **create a trail** (Event History isn't enough). *"audit S3 object-level access / Lambda invokes"* → **data events** (opt-in, extra cost). *"missing IAM / STS / CloudFront logs"* → **global service events → `us-east-1`**. *"audit every account in the org centrally"* → **organizational trail from the management account**. *"prove logs weren't altered"* → **log file integrity validation**. *"react in real time to an API call"* → **EventBridge on CloudTrail events**, not the ~15-min log files. *"detect unusual API activity automatically"* → **CloudTrail Insights**.

> [!warning] Common CloudTrail Gotchas
>
> - **Event History ≠ a trail.** It's free but capped at **90 days** and **management events only** — no data events, no S3 archive.
> - **Data events are off by default and cost extra** — management events are the free default.
> - **CloudTrail is not real-time** for log delivery (~15 min). Any "instant" requirement points to EventBridge, not delivered log files.
> - **Global service events go to `us-east-1`** — if IAM/STS/CloudFront logs are "missing," check that setting / that region.
> - **Don't confuse CloudTrail with CloudWatch Logs, VPC Flow Logs, or S3 access logs** — CloudTrail = AWS API/account audit, not application/OS/network logs.

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

## SSM Parameter Store

See [Systems Manager — Parameter Store](../services/ssm.md#parameter-store) for full details.

- **Purpose** — central store for config and secrets; the **correct replacement for putting secrets in [EC2 user data](../services/ec2.md#bootstrapping)** (user data is readable by anyone who reaches the instance)
- **Types** — **String**, **StringList**, **SecureString** (SecureString = [KMS](../services/kms.md)-encrypted)
- **Hierarchical names** (`/app/db/password`) — retrieve **individually** by full name (`get-parameters`) or a **whole branch** at once (`get-parameters-by-path`); IAM permissions scope to a **single parameter or an entire tree**
- **Public AWS-published parameters** — classic use: latest **AMI ID** per OS/Region, referenced from [CloudFormation](../services/cloudformation.md) so templates don't hard-code an image
- **Regional + multi-AZ**, but **no cross-Region replication** — recreate parameters per Region
- **SecureString needs two permission layers** — `ssm:GetParameter*` **and** KMS permission on the key (`--with-decryption` triggers `kms:Decrypt`). Enables **role separation**: list-without-read. **Symmetric KMS keys only**
- **Standard vs advanced tier** — standard (4 KB, direct KMS `Encrypt`, **free**, 10K/Region) vs advanced (8 KB, **envelope encryption** via the Encryption SDK, **parameter policies**, **billable**, 100K/Region). Upgrade is **one-way** — no downgrade
- **Permission split** — standard encrypt `kms:Encrypt`; advanced encrypt `kms:GenerateDataKey`; decrypt (both) `kms:Decrypt`

> [!tip] Exam Tip
> Pattern-match: *"store config/secrets for EC2/Lambda, integrate with IAM, cheap, no rotation"* → **Parameter Store**. *"automatically rotate a DB password"* or *"native RDS secret integration"* → **Secrets Manager**. *"stop hard-coding the AMI ID in CloudFormation"* → **public parameter** (latest AMI ID). *"someone can list the secret name but must not read its value"* → SecureString with **KMS permission withheld** (role separation).

> [!warning] Gotcha
> Granting only `ssm:GetParameter` is **not enough** to read a SecureString — without `kms:Decrypt` on the key, the decrypt fails. The reverse trap: a parameter created in `us-east-1` is **not** visible in `us-west-2` (no cross-Region replication).

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
| [SSM Parameter Store](../services/ssm.md) parameter | Regional (multi-AZ within Region; no cross-Region replication) |
| [CloudTrail](../services/cloudtrail.md) trail | Regional (all-regions trail spans every region; global service events → `us-east-1`) |

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
- [Control Tower](../services/control-tower.md)
- [Systems Manager (SSM)](../services/ssm.md)
- [Cost Optimization](../patterns/cost-optimization.md)
- [KMS](../services/kms.md)
- [CloudWatch](../services/cloudwatch.md)
- [CloudTrail](../services/cloudtrail.md)
- [CloudFormation](../services/cloudformation.md)
- [Boot Time to Service Time](../patterns/boot-time-optimization.md)
