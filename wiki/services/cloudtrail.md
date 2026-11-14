---
title: AWS CloudTrail
type: service
tags: [cloudtrail, audit, governance, logging, security, observability, management-events, data-events, network-activity-events, insight-events, global-service-events, organizational-trail, log-file-integrity, eventbridge, s3, cloudwatch, kms]
certs: [aws-saa]
updated: 2026-11-13
---

# AWS CloudTrail

CloudTrail is AWS's **auditing and governance** service. It records **API actions and account activity** across an account: stop an instance, change a security group, create or delete an [S3](s3.md) bucket, log in to the console — almost everything done *to* an account is captured here as an event. Where [CloudWatch](cloudwatch.md) answers "is my workload healthy?", CloudTrail answers "**who did what, when, and from where?**".

## Core Concepts

### CloudTrail Event

A CloudTrail **event** is a record of a single activity in an account. The activity can be performed by an [IAM](iam.md) user, an IAM role, or an AWS service acting on your behalf. Each logged API call or account action becomes one event, recorded as a structured JSON record that includes the calling principal, the source IP, the timestamp, and the request/response parameters.

### Event History — the free default

Every account has **Event History** enabled **by default at no cost**:

- Stores the **last 90 days** of events, viewable and searchable in the console or via the API.
- Captures **management events only** (see below) — data and Insight events do not appear here.
- Requires no configuration.

To do anything beyond the rolling 90-day window — long-term retention, delivery to [S3](s3.md), data-event capture, or cross-account aggregation — you must create a **trail**.

> [!warning] Gotcha
> Event History is **not** a complete audit archive. It is management-events-only, capped at 90 days, and disappears off the back of that window. If a question asks for retention beyond 90 days, data-event capture, or log delivery to S3/CloudWatch, the answer is **create a trail** — Event History alone is insufficient.

## Event Types

CloudTrail events come in four types. Older notes that list only management, data, and Insights events are outdated; **network activity events** are now a separate CloudTrail event category. Knowing which are on by default — and which cost extra — is a recurring exam theme.

| Type                        | What it captures                                                                   | Examples                                                                          | Default?                            |
| --------------------------- | ---------------------------------------------------------------------------------- | --------------------------------------------------------------------------------- | ----------------------------------- |
| **Management events**       | Control-plane operations *on* resources                                            | Create/terminate EC2 instance, create a VPC, console logins, attach an IAM policy | Logged by default                   |
| **Data events**             | Data-plane operations performed *on or in* a resource                              | [S3](s3.md) object GET/PUT, Lambda function invocation, DynamoDB item activity    | Off by default — opt-in, extra cost |
| **Network activity events** | AWS API activity that traverses supported VPC endpoints                            | PrivateLink/VPC endpoint calls to supported AWS services                          | Off by default — opt-in, extra cost |
| **Insight events**          | Anomaly detection over management-event volume (unusual call rates / error spikes) | Sudden surge in `TerminateInstances`, spike in throttling errors                  | Off by default — opt-in, extra cost |

> [!tip] Why data events are off by default
> Data events are **far higher volume** than management events — logging every individual S3 object access on a busy bucket would generate enormous quantities of records and cost. You enable them deliberately, on the specific resources you need to audit.

CloudTrail **Insights** is a separate opt-in analysis layer: it establishes a baseline of normal management-event activity and emits an **Insight event** when it detects a deviation (for example a runaway script or a misconfigured service hammering an API). It complements — it does not replace — management/data event logging.

## Trails — the unit of configuration

A **trail** is how you configure CloudTrail's behaviour: it tells CloudTrail **what** to log and **where** to deliver it. Without a trail you only have the 90-day Event History.

### Region scope of a trail

CloudTrail is a **regional service** (see [Resource Scope](../concepts/resource-scope.md)). A trail is created in a region, and can be configured one of two ways:

- **Single-region trail** — only ever lives in, and logs events for, its one region. To cover every region this way you would create a separate single-region trail in each region.
- **All-regions trail** — behaves as a collection of trails across **every** AWS region, managed as **one logical trail**. It also **automatically picks up new regions** as AWS adds them. This is the recommended configuration and the console default.

### Global Service Events

A separate trail setting controls whether the trail also captures **global service events**.

- **Most services** log events to the **region where the activity occurred** (an EC2 instance created in `ap-southeast-2` logs to `ap-southeast-2`).
- A small number of **true global services** have no region of their own, so their events are delivered to **`us-east-1` (N. Virginia)** as global service events. The classic exam trio:
  - [IAM](iam.md)
  - STS
  - CloudFront
- [Route 53](route53.md) is another global service whose events route to `us-east-1`.
- A trail must have **global service event logging enabled** to capture these. The console **enables it by default** when you create a trail.

> [!tip] Exam Tip
> If a question describes **missing IAM / STS / CloudFront logs**, the answer involves **global service events** being delivered to **`us-east-1`** — global service event logging must be enabled on a trail there (or on an all-regions trail). Mnemonic: **"I.S.C."** = **I**AM, **S**TS, **C**loudFront, all funnel to `us-east-1`.

### Organizational trail

- Created from the **management account** of an [AWS Organization](organizations.md).
- Captures and stores events for **all member accounts** in the organization, in one place.
- Gives you a **single management and audit point** for API/account activity across every account — dramatically simplifying multi-account auditing. Member-account administrators can see the org trail but cannot modify or delete it.

> [!tip] Exam Tip
> "Audit all API activity across every account in the organization, centrally" → **organizational trail from the management account**, not a per-account trail in each member account. This pairs naturally with an [SCP](organizations.md#service-control-policies-scps) that forbids member accounts from disabling CloudTrail.

## Storage and Delivery Destinations

Once a trail captures events (management always, data/Insight when enabled), it delivers them to one or both of:

### S3 bucket (the default destination)

- Logs are written to a **specified [S3](s3.md) bucket** and retained **indefinitely** — you pay only for the storage used.
- Stored as **compressed JSON log files**, so the footprint is tiny and any JSON-aware tooling can parse them.
- Logs can be encrypted with [SSE-KMS](s3.md#sse-kms--kms-managed-keys) for an extra layer of control and a KMS audit trail on the encryption key.

### CloudWatch Logs (optional integration)

- A trail can **also** push events into [CloudWatch Logs](cloudwatch.md#cloudwatch-logs).
- This unlocks **searching, metric filters, and alarms** over the event data — far more powerful than raw S3 files. A canonical use is a metric filter that watches for a specific API call (for example a root-account login or a security-group change) and fires a CloudWatch alarm to SNS.

## Log File Integrity Validation

Because CloudTrail logs are evidence, a trail can be configured with **log file integrity validation** (a synthesis point beyond the raw note, but a recurring exam topic):

- CloudTrail computes a **SHA-256 hash** of each delivered log file and, once per hour, writes a **digest file** that references those hashes and is **signed with SHA-256 with RSA** using a CloudTrail-managed private key.
- This lets you **prove cryptographically** that log files were not modified, deleted, or forged after delivery — exactly what auditors and compliance frameworks demand.

> [!tip] Exam Tip
> "Prove the audit logs haven't been tampered with" → enable **CloudTrail log file integrity validation** (SHA-256 hashing + signed digest files). Combine with an S3 bucket policy / [Object Lock](s3.md#object-lock) and a restrictive [SCP](organizations.md#service-control-policies-scps) to make the logs effectively immutable.

## Timing — CloudTrail Is Not Real-Time

CloudTrail **log delivery is not real-time**. Log files are typically delivered to S3/CloudWatch Logs **within about 15 minutes** of the activity, published several times per hour in batches.

> [!warning] Gotcha
> Any question that requires **real-time** logging or instantaneous reaction to an API call → **CloudTrail's log delivery is the wrong answer.** Do not treat Event History or delivered log files as an up-to-the-second list.

There is an important nuance, though. The ~15-minute figure applies to **log file delivery** to S3 and CloudWatch Logs. CloudTrail **also surfaces API-call events from an enabled trail onto [Amazon EventBridge](eventbridge.md)**, where rules can match them with much lower latency. So the pattern "react automatically and quickly when a specific API call happens" is solved with **EventBridge rules on CloudTrail events**, not by polling delivered log files.

These CloudTrail-sourced events carry `detail-type: "AWS API Call via CloudTrail"` and — unlike a service's **native** EventBridge event — **include the requesting IAM principal**, so a rule can react based on *who* made the call. The CloudTrail path requires an **active trail with logging enabled**; management events are the trail default, while **data events** (S3 object-level, Lambda `Invoke`) and **network activity events** require explicit event selectors. An EventBridge rule in the ordinary `ENABLED` state matches **write / mutating** management events; matching **read-only** (`Describe` / `Get` / `List`) management events additionally needs the rule state `ENABLED_WITH_ALL_CLOUDTRAIL_MANAGEMENT_EVENTS`. Delivery remains best-effort and scoped to what the trail logs. See [EventBridge — CloudTrail as an Event Source](eventbridge.md#cloudtrail-as-an-event-source) for the native-path-vs-CloudTrail-path comparison.

## Reading the Diagram

![CloudTrail infographic: blue dotted arrows feed Management Events and Data Events into the central CloudTrail icon; a red double-headed arrow shows the One Region vs All Regions scope modes; captured events flow out to CloudWatch Logs and an S3 bucket; red dotted arrows from a Region (US flag) icon and a Global Services (globe) icon feed back into the trail, illustrating ingestion of both regional and global service events](../assets/cloudtrail-essentials.png)

Read it left to right:

- **Left (inputs):** Management Events and Data Events flow as blue dotted arrows *into* the central CloudTrail icon — management by default, data only when enabled.
- **Centre (the trail):** the red double-headed arrow shows the two scope modes — **One Region** vs **All Regions**.
- **Top / right (outputs):** captured events are delivered out to **CloudWatch Logs** and an **S3 bucket**.
- **Bottom-right (sources):** the Region (US flag) and Global Services (globe) icons feed back into the trail — a trail ingests both normal regional events and global service events (IAM/STS/CloudFront, delivered to `us-east-1`).

## Where CloudTrail Fits

CloudTrail is one of three complementary observability/governance services that appear together on the exam:

| Service                     | Answers                                                                                                               | Granularity                       |
| --------------------------- | --------------------------------------------------------------------------------------------------------------------- | --------------------------------- |
| **CloudTrail**              | *Who* made *which* API call, when, from where, and with what parameters                                               | Account/API activity (audit)      |
| [CloudWatch](cloudwatch.md) | *Is the workload healthy* (metrics) and *what did it log* (CloudWatch Logs)                                           | Operational telemetry             |
| [AWS Config](config.md)     | *What the resource looked like* before and after each change, how it changed over time, and *whether it is compliant* | Resource configuration/compliance |

> [!warning] Gotcha: CloudTrail records the API call activity, not the resource state
> A CloudTrail event shows what the API call asked for in `requestParameters`. For example, an `AuthorizeSecurityGroupIngress` event contains the security group ID and the new rule (TCP 22 from `0.0.0.0/0`). It **cannot** tell you what the resource's state was before the call or what its state is after it, such as the security group's full rule set. Rebuilding that from CloudTrail alone would mean replaying every call that ever touched the resource. That before-and-after view is what [AWS Config](config.md#configuration-items-and-history) records. Each Config configuration item links to the related CloudTrail event, so Config shows how the resource changed and CloudTrail shows the call that caused it.

CloudTrail records activity but does not judge it. [GuardDuty](guardduty.md) analyzes CloudTrail management events, and S3 data events with its S3 Protection plan, to detect suspicious activity. It reads its own independent copy of the events, so it does not need a trail.

CloudTrail also underpins finer-grained audit stories on other pages: it is the **structured, IAM-context-rich, near-real-time alternative** to [S3 server access logging](s3.md#access-logging) (via S3 **data events**), and it is what records every `Encrypt` / `Decrypt` / `GenerateDataKey` call when you use [SSE-KMS](s3.md#sse-kms--kms-managed-keys).

## Exam Recap

| Concept                     | Key fact                                                                                  |
| --------------------------- | ----------------------------------------------------------------------------------------- |
| **Default behaviour**       | 90-day **Event History**, free, **management events only**, no S3 storage                 |
| **Trail needed for**        | Long-term retention, S3/CloudWatch delivery, data events, customization                   |
| **Default event type**      | Management events (includes console logins)                                               |
| **Data events**             | Opt-in, extra cost, high volume (S3 objects, Lambda invokes)                              |
| **Network activity events** | Opt-in, extra cost, supported VPC endpoint activity                                       |
| **Insight events**          | Opt-in anomaly detection over management-event activity                                   |
| **Trail scope**             | Single-region **or** all-regions (auto-adds new regions)                                  |
| **Global services**         | IAM, STS, CloudFront (and Route 53) deliver to **`us-east-1`** as global service events   |
| **Storage format**          | Compressed **JSON** files in S3, retained indefinitely; optional SSE-KMS                  |
| **CloudWatch Logs**         | Optional; enables search + metric filters + alarms                                        |
| **Integrity**               | **Log file integrity validation** = SHA-256 hashes + signed digest files                  |
| **Org trail**               | From the management account; all accounts in one place                                    |
| **Real-time?**              | No — log delivery is ~15 min. For fast reactions use **EventBridge** on management events |

> [!warning] Gotcha
> **CloudTrail is not the only thing logging "logs."** Don't confuse it with [CloudWatch Logs](cloudwatch.md) (operational application/OS logs), VPC Flow Logs (network flow records), or S3 access logs (HTTP request records). CloudTrail = **AWS API/account activity audit**.

## References

- [CloudWatch](cloudwatch.md) — metrics, CloudWatch Logs, the alarm pipeline CloudTrail feeds into
- [EventBridge](eventbridge.md) — rules on CloudTrail management events for near real-time reactions (vs the ~15-min log delivery)
- [S3](s3.md) — default log destination; access logging vs CloudTrail data events
- [IAM](iam.md) — principals recorded in events; global service events
- [Organizations](organizations.md) — organizational trail; SCPs that protect CloudTrail
- [Resource Scope](../concepts/resource-scope.md) — CloudTrail as a regional service; global service events to `us-east-1`
- Source note: `raw/aws-cert-study-notes/aws-solution-architect/cloudtrail_101.md`
- [AWS CloudTrail User Guide](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-user-guide.html)
- [AWS CloudTrail events — event categories](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-events.html)
