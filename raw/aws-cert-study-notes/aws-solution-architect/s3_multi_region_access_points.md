# S3 Multi-Region Access Points (MRAP)

> **Source:** Adrian Cantrill — DEMO: Multi-Region Access Points (MRAP) **Type:** Mini-project / hands-on demo **Service:** Amazon S3

---

## 1. Foundational concept

A **Multi-Region Access Point (MRAP)** is a single S3 global endpoint that fronts multiple S3 buckets located in different AWS regions. Clients connect to one endpoint, and S3 transparently routes the request to the closest bucket (lowest-latency region) that backs the access point.

Pair MRAP with **S3 Cross-Region Replication (CRR)** so that all backing buckets contain the same data. The access point itself does not copy data between buckets — replication rules do.

### Anatomy

- **MRAP name**: unique within an AWS account (not globally unique like bucket names).
- **ARN**: used by SDK/CLI clients (`s3://<arn>`).
- **Alias**: shorter S3-style identifier for the access point.
- **Backing buckets**: 2+ S3 buckets in different regions, fixed at creation (cannot be added or removed later).
- **Failover configuration**: each backing bucket is `active` or `passive`.
    - `active–active` (default): traffic can be routed to any active bucket.
    - `active–passive`: passive buckets only used when no active buckets are available.

### Routing behavior

- A client request from region X is directed to the backing bucket closest to X (latency-based, similar in spirit to Route 53 latency routing).
- For writes: the object lands directly in the closest bucket, then replicates outward.
- For reads: served from the closest bucket. If the object hasn't replicated there yet, the read fails — this is the **consistency lag** problem.

---

## 2. Demo walkthrough (mechanics)

### Prerequisites

- AWS account with admin permissions (or IAM admin user in the management account if using Cantrill's general account setup).
- Two AWS regions chosen for backing buckets. Demo uses **ap-southeast-2 (Sydney)** and **ca-central-1 (Canada)**.

### Step 1 — Create two S3 buckets

For each region:

- Name pattern: `multi-region-demo-<region>-<random>` (e.g., `multi-region-demo-sydney-1337`). Names must be **globally unique**.
- Set the correct region for each bucket.
- **Enable bucket versioning** — required because replication depends on versioning.
- Leave everything else at default.

### Step 2 — Create the MRAP

- S3 console → **Multi-Region Access Points** → **Create**.
- Name: any string, unique within the account (e.g., `really-really-critical-cat-data`).
- **Add buckets**: select both buckets created above.
- **Note:** the set of backing buckets is locked at creation — they cannot be added, removed, or edited afterward.
- Click **Create**.

Provisioning can take anywhere from ~10 minutes to (worst-case) 24 hours. Wait for status = `ready` before proceeding.

### Step 3 — Inspect failover / replication state

Click into the MRAP → **Replication and failover** tab:

- **Failover tab**: both buckets shown as `active–active` by default. Each bucket has an **Edit routing status** option to flip to `passive`. Leave as active–active for this demo.
- **Replication tab**: nothing configured yet — needs to be created.

### Step 4 — Configure replication

Scroll to **Replication rules** → **Create replication rule**.

Two templates:

|Template|Use case|
|---|---|
|Replicate objects among all specified buckets|Active–active, every bucket holds the same data (mesh replication).|
|Replicate from source bucket(s) to destination bucket(s)|One-way / hub-and-spoke patterns.|

For active–active MRAP → choose **"Replicate objects among all specified buckets"**. This creates two-way replication between every pair of selected buckets.

Settings used in the demo:

- **Buckets**: select both Sydney and Canada.
- **Status**: Enabled.
- **Scope**: Apply to all objects in the bucket.
- **IAM role**: auto-generated (no action needed — S3 creates the role with the right permissions).
- **Encryption / destination storage class**: locked by the mesh template, leave as-is.
- **Additional replication options**:
    - ❌ **Replication Time Control (RTC)** — skipped (paid SLA feature; not needed for the demo).
    - ✅ **Replication metrics and notifications**.
    - ❌ **Replicate delete markers**.
    - ✅ **Replica modification sync** — keeps metadata changes in sync.

Click **Create replication rules**. The graphical view now shows two-way replication between the buckets.

### Step 5 — Test from a region near one bucket

- Switch console region to a region **close to but not the same as** one of the backing buckets.
- Demo uses **Tokyo (ap-northeast-1)**, which is geographically close to Sydney.
- Open **CloudShell**.
- Generate a 10 MB test file:
    
    ```bash
    dd if=/dev/urandom of=test1.file bs=1M count=10
    ```
    
- Copy the **MRAP ARN** from the console.
- Upload via the MRAP:
    
    ```bash
    aws s3 cp test1.file s3://<MRAP-ARN>
    ```
    
- Verify in S3 console: the file appears in the **Sydney** bucket first (closest to Tokyo). After a short delay, the replicated copy appears in **Canada**.

### Step 6 — Test from a region near the other bucket

- Switch CloudShell to **US East 2 (Ohio)** — close to Canada.
- Generate `test2.file` the same way.
- Upload to the MRAP. Result: file lands directly in **Canada**, then replicates to Sydney after a delay.

### Step 7 — Test from a "middle" region

- Switch to **Mumbai (ap-south-1)** — roughly equidistant between Sydney and Canada.
- Generate `test3.file`, upload via MRAP.
- Whichever bucket MRAP picks (in the demo: Sydney) receives the direct copy; the other receives the replicated copy after the lag.

### Step 8 — Demonstrate the consistency lag failure mode

This is the key teaching moment.

Open **two CloudShells**:

- Shell A in **Canada** (ca-central-1).
- Shell B in **Sydney** (ap-southeast-2).

In **Shell A (Canada)**: write to the MRAP.

```bash
dd if=/dev/urandom of=test4.file bs=1M count=10
aws s3 cp test4.file s3://<MRAP-ARN>
```

→ MRAP routes to the **Canada bucket** (same region as the caller). Object exists in Canada immediately; replication to Sydney is pending.

In **Shell B (Sydney)** _immediately after_: try to read the same object back via the MRAP.

```bash
aws s3 cp s3://<MRAP-ARN>/test4.file .
```

→ MRAP routes to the **Sydney bucket** (same region as the caller). The object hasn't replicated there yet → error: **object does not exist**.

If you retry the GET, it keeps failing until replication finishes. Once replicated, the GET succeeds.

This is the **read-after-write consistency lag** inherent to MRAP + async replication. It exists because:

- MRAP routing is regional/latency-based, not content-aware — it doesn't know which bucket actually holds the latest copy of a given object.
- Cross-Region Replication is asynchronous by default.

**Mitigation**: enable **Replication Time Control (RTC)** when creating the replication rule. RTC provides a 99.99% SLA that 99.9% of objects replicate within 15 minutes. RTC adds cost.

### Step 9 — Clean up

1. S3 → Multi-Region Access Points → select the MRAP → **Delete** (confirm by typing the name).
2. S3 → Buckets → for each bucket:
    - **Empty** the bucket (type "permanently delete" to confirm).
    - **Delete** the bucket (type the bucket name to confirm).

---

## 3. Key learnings & decision guidance

|Situation|Choice|
|---|---|
|Want a single global S3 endpoint with low-latency routing|Use MRAP|
|Adding/removing buckets after MRAP creation|**Not possible** — plan the bucket set up front|
|Active–active multi-region writes/reads|Active–active failover + mesh replication template|
|Cold standby region|Active–primary, passive–DR; failover flips on outage|
|Strict read-after-write across regions|MRAP alone is **not enough** — pair with **RTC** and tolerate the SLA window, or rewrite the application to read from the same region it wrote to|
|Bucket versioning|**Required** on every backing bucket — replication won't work without it|
|Replication IAM role|Auto-generated by the console; no manual setup needed in this flow|
|`Replicate delete markers`|Usually **off** — protects against accidental cross-region deletes|
|`Replica modification sync`|Usually **on** — keeps object metadata aligned both ways|

### Gotchas

- **MRAP creation latency**: can take up to 24h (typically 10–30 min). Cannot use the endpoint until status = `ready`.
- **Bucket set is immutable** post-creation. Recreate the MRAP to change membership.
- **MRAP ARN**, not the bucket name, is what CLI/SDK calls use: `s3://arn:aws:s3::<account>:accesspoint/<alias>`.
- **Async replication = consistency lag**. RTC narrows the window but doesn't eliminate it.
- **Cost**: MRAP charges a per-GB data routing fee on top of standard S3 + replication + (optional) RTC charges.

---

## 4. Exam cheat sheet

|Keyword in question|Likely answer|
|---|---|
|"Single global S3 endpoint" / "lowest-latency S3 bucket"|**Multi-Region Access Point**|
|"Active–active S3 across regions"|MRAP + mesh CRR|
|"Failover S3 region" / "DR for S3"|MRAP with active/passive failover configuration|
|"Add another bucket to the MRAP later"|**Not supported** — buckets are fixed at MRAP creation|
|"Cross-region replication requires…"|**Versioning enabled** on source and destination|
|"Guaranteed replication SLA" / "15-minute replication"|**Replication Time Control (RTC)** — paid feature, 99.99% SLA, 99.9% within 15 min|
|"Object not found right after writing through MRAP"|**Consistency lag** — replication hasn't propagated yet; fix with RTC or app-level read-from-write-region|
|"Which IAM role replicates objects?"|**Auto-generated S3 replication role** (the console creates it; trust policy = `s3.amazonaws.com`)|
|"Replicate every bucket to every other bucket"|"Replicate among all specified buckets" template (mesh)|
|"One-way replication"|"Source to destination" template|
|MRAP name uniqueness|**Account-scoped**, not global (unlike bucket names which are global)|
|Bucket name uniqueness|**Globally unique** across all AWS accounts|
|CLI syntax to use MRAP|`s3://<MRAP-ARN>` — use the ARN, not a bucket name|

---

## 5. Architecture recap

```
                       ┌─────────────────────────┐
        Client A ───►  │   MRAP global endpoint  │  ◄─── Client B
        (in TYO)       │   (latency-based route) │       (in IAD)
                       └────────────┬────────────┘
                                    │
                ┌───────────────────┴───────────────────┐
                ▼                                       ▼
        ┌───────────────┐   ◄── async CRR (mesh) ──►   ┌───────────────┐
        │ bucket-sydney │                              │ bucket-canada │
        │  (ap-se-2)    │   ◄────────────────────►     │ (ca-central-1)│
        └───────────────┘                              └───────────────┘
              versioning ON                                  versioning ON
              active                                         active
```

- Client A in Tokyo → MRAP → Sydney bucket (direct write), Canada bucket gets replicated copy.
- Client B in Virginia → MRAP → Canada bucket (direct write), Sydney bucket gets replicated copy.
- The consistency lag failure mode triggers when Client B reads an object Client A just wrote, before replication finishes.

---

## 6. Relationship to S3 Access Points

> See companion note: **S3 Access Points** (`s3_access_points.md`).

MRAP and regular S3 Access Points are **sibling features** in the same product family (both live under the `aws s3control` API namespace), but they solve different problems and are **not nested** — an MRAP does not sit on top of regular access points.

### Side-by-side

||**S3 Access Points**|**S3 Multi-Region Access Points**|
|---|---|---|
|**Purpose**|Manageability — split one bucket's policy into many scoped views|Routing + DR — single global endpoint over many regional buckets|
|**Backing storage**|**One** bucket|**2+** buckets in **different regions**, fixed at creation|
|**Endpoint scope**|Regional (same region as the bucket)|**Global** (latency-routed)|
|**CLI**|`aws s3control create-access-point`|`aws s3control create-multi-region-access-point`|
|**ARN format**|`arn:aws:s3:<region>:<account>:accesspoint/<name>`|`arn:aws:s3::<account>:accesspoint/<alias>` (regionless)|
|**Name uniqueness**|Account + region|Account-wide (global)|
|**Network origin (Internet / VPC)**|✅ Configurable, immutable|❌ Internet only|
|**Failover routing config**|❌|✅ Active/active or active/passive per backing bucket|
|**Replication relationship**|Cannot be a replication **destination**|Almost always paired with CRR; mesh CRR between backing buckets|
|**Consistency model**|Inherits S3 strong read-after-write|Subject to **cross-region replication lag**|
|**Per-AP policy**|✅ Up to 20 KB|✅ Up to 20 KB (same idea)|
|**Resource hierarchy mental model**|A "view" or "doorway" into one bucket|A "router" sitting in front of a set of buckets|

### Shared concepts (carry-over from the access points note)

- **Naming/routing layer, not a permissions system** — neither one grants access on its own. The underlying bucket policies must still allow the request. The DNS-CNAME mental model applies to both.
- **CLI namespace** — both are `aws s3control`, not `aws s3` or `aws s3api`.
- **Account-unique names** — neither is globally unique like bucket names.
- **Two-layer permissions rule** — same `s3:DataAccessPointAccount` / `s3:DataAccessPointArn` condition keys delegate trust from bucket policy to AP policy in both cases.
- **Immutable post-creation properties** — regular APs lock network origin and BPA; MRAPs lock the backing bucket set.

### Where they diverge sharply

- **Single-region vs. cross-region**: regular access points are bound to one bucket in one region; MRAP exists specifically to span regions.
- **Replication relationship is inverted**:
    - Regular access points **cannot be replication destinations** — a key gotcha when locking a bucket down to VPC-only access points.
    - MRAP **requires** replication between its backing buckets to actually be useful (otherwise the regional routing returns inconsistent data).
- **Consistency**: regular APs inherit S3's strong read-after-write. MRAP introduces a **replication lag window** because routing is regional but writes are asynchronously replicated. RTC narrows but doesn't eliminate this.
- **No VPC origin for MRAP** — that's a regular-access-point-only feature. If you need VPC-restricted access _and_ multi-region routing, you currently have to layer the application differently (e.g., regional VPC-origin APs per region, and handle routing in the app).
- **Failover semantics** — only MRAP has the active/passive routing concept. Regular access points have no analogue.

### Exam disambiguation

| Keyword in question                                                               | Answer                                                                                                     |
| --------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| "Single global S3 endpoint" / "lowest-latency S3" / "failover S3 across regions"  | **MRAP**                                                                                                   |
| "Many teams need different access to the same bucket" / "bucket policy too large" | **S3 Access Points**                                                                                       |
| "Restrict bucket access to one specific VPC"                                      | **S3 Access Points** with VPC origin (MRAP can't do this)                                                  |
| "Active/active S3 in two regions behind one endpoint"                             | **MRAP** + mesh CRR                                                                                        |
| "Disaster recovery for S3 with automatic regional failover"                       | **MRAP** active/passive                                                                                    |
| Cannot be a replication destination                                               | **Regular access points** (not MRAP — MRAP's backing buckets _are_ replicated, just not via the AP itself) |