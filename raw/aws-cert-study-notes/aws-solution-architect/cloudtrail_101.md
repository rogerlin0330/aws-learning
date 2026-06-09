# CloudTrail

CloudTrail is the AWS auditing and governance service that **logs API actions and account activity** affecting an AWS account. Stop an instance, change a security group, create or delete an S3 bucket — almost everything done to an account is recorded here.

---

## Core Concepts

### CloudTrail Event

- A **record of an activity** in an AWS account.
- The activity can be taken by a **user**, a **role**, or a **service**.
- Each logged API call / account activity becomes one event.

### Event History (the default)

- CloudTrail stores the **last 90 days** of events in **Event History**.
- Enabled **by default** in every account, at **no cost**.
- To do anything beyond this 90-day window, you must create a **trail**.

---

## Event Types

CloudTrail events come in three types. (Insight events are covered separately and may not apply to every course path.)

|Type|What it captures|Examples|Default?|
|---|---|---|---|
|**Management events**|Management operations on resources — the **control plane**|Create/terminate EC2 instance, create VPC, console logins|✅ Logged by default|
|**Data events**|Resource operations performed **on or in** a resource|S3 object uploads/access, Lambda function invocation|❌ Must be explicitly enabled (extra cost)|
|**Insight events**|(Separate topic)|—|—|

> 🎯 **Why data events are off by default:** they are **much higher volume**. Logging every single S3 object access would add up extremely quickly — and cost money.

---

## Trails — the unit of configuration

A **trail** is how you configure CloudTrail's behavior: it tells CloudTrail what to log and where to store it.

### Region scope of a trail

CloudTrail is a **regional service**. A trail logs events for the region it's created in, but can be configured one of two ways:

- **One region trail** — only ever lives in, and logs events for, its single region. To cover every region manually, you'd create a one-region trail _in each region_.
- **All regions trail** — behaves as a collection of trails across **every** AWS region, managed as **one logical trail**.
    - ✅ **Bonus:** automatically updated to include **new regions** as AWS adds them.

### Global Service Events ⭐ (high-value exam topic)

A separate trail configuration setting controls whether the trail also logs **global service events**.

- **Most services** log events to the **region where the event occurred** (e.g. an EC2 instance created in ap-southeast-2 logs to ap-southeast-2).
- A **small number of true global services** always log to **us-east-1 (N. Virginia)**:
    - **IAM**
    - **STS**
    - **CloudFront**
- These are **global service events**, and a trail must have global service event logging **enabled** to capture them.
- This setting is **normally enabled by default** when you create a trail in the console UI.

> 🎯 **Exam trigger:** If a question involves missing **IAM / STS / CloudFront** logs, the answer involves **global service events** logging to **us-east-1**, which must be enabled on the trail.

---

## Reading the Diagram

![[cloudtrail-essentials.png]]

The infographic maps directly onto the architecture above. Read it left → center → right:

- **Left side (blue, inputs):** **Management Events** and **Data Events** flow as **blue dotted arrows** _into_ the central CloudTrail icon — these are the event sources the trail captures (management by default, data only when enabled).
- **Center (the CloudTrail icon):** the trail itself, with a **red double-headed arrow** showing its two scope modes — **One Region** (left) vs. **All Regions** (right).
- **Top / right (blue, outputs):** captured events are delivered _out_ to two destinations:
    - **CloudWatch Logs** (center-top)
    - **S3** (top-right)
- **Bottom-right (red dotted arrows):** the **Region** (US flag) and **Global Services** (globe) icons feed _back up into_ the trail — illustrating that a trail ingests both normal regional events and the global service events (IAM/STS/CloudFront → us-east-1).

---

## Storage & Delivery Destinations

Once a trail is created, it captures management events (and data events if enabled) and delivers them to either or both:

### S3 Bucket (default destination)

- Logs stored in a **definable S3 bucket**, retained **indefinitely** (you pay only for storage used).
- Stored as a set of **compressed JSON log files** → tiny footprint, and parseable by **any tooling** that reads standard JSON.

### CloudWatch Logs (optional integration)

- CloudTrail can _also_ push data into **CloudWatch Logs**.
- Unlocks **searching** and **metric filters** — far more powerful and feature-rich than raw S3 files.

---

## Organizational Trail

- Created from the **management account** of an AWS Organization.
- Captures and stores events for **all accounts** in the organization.
- A **single management point** for all API/account activity across every member account → dramatically simplifies multi-account auditing.

---

## ⚠️ Key Traps & Gotchas

- **CloudTrail is NOT real-time.** Log files are typically delivered **within ~15 minutes** of the activity, published multiple times per hour. Do **not** assume Event History/logs are a complete, up-to-the-second list.
    - 🎯 **Exam trigger:** any question requiring **real-time** logging → **CloudTrail is the wrong answer.**
- Only the **90-day Event History** is free/default. **No S3 storage** happens unless you configure a trail.
- **Data events** are **not** logged by default — must be explicitly enabled, and they carry **extra cost**.
- **Management events** are the default; this includes console logins.
- **Global service events** (IAM/STS/CloudFront → us-east-1) require the trail setting to be enabled, or you'll miss them.

---

## Exam Recap

|Concept|Key fact|
|---|---|
|**Default behavior**|90-day **Event History** only, free, no S3 storage|
|**Trail needed for**|Long-term storage, S3/CloudWatch delivery, customization|
|**Default event type**|Management events|
|**Data events**|Opt-in, extra cost, high volume|
|**Trail scope**|One region **or** All regions (auto-adds new regions)|
|**Global services**|IAM, STS, CloudFront → log to **us-east-1** as global service events|
|**Storage format**|Compressed **JSON** files in S3, retained indefinitely|
|**CloudWatch Logs**|Optional; enables search + metric filters|
|**Org trail**|From management account; all accounts in one place|
|**Real-time?**|❌ No — ~15 min delivery delay|

**Mnemonic — global service trio:** **"I.S.C."** = **I**AM, **S**TS, **C**loudFront → all funnel to **us-east-1**.