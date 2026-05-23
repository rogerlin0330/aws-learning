# S3 Event Notifications

## Overview

**S3 Event Notifications** is a feature that lets you configure an S3 bucket to emit notifications when specific things happen to objects inside it (creation, deletion, restoration, replication). These notifications are delivered to a small set of AWS destinations and form the basis for **event-driven workflows** triggered by activity in S3.

It is a relatively old feature, predating EventBridge integration. It supports a limited set of event types and destinations, but it remains useful and exam-relevant.

---

## Architecture

![[s3-events-notification.png]]
The flow has four moving parts:

1. **S3 Bucket** — the source where events originate.
2. **Event Notification Configuration** — attached to the bucket, this defines _which_ events to match and _where_ to send them.
3. **Event Types** — CREATE, DELETE, RESTORE, REPLICATE operations on objects.
4. **Destinations** — one of three AWS services that receive the event:
    - **Lambda function** (invoked with the event as payload)
    - **SQS queue** (event placed on the queue)
    - **SNS topic** (event published, fans out to subscribers)

Each destination must have a **resource policy** allowing the **S3 service principal** to interact with it (`SQS queue policy`, `SNS topic policy`, or `Lambda resource policy`). Without this, S3 cannot deliver the event.

> ⚠️ **CLI/API only for Lambda**: At the time of the lesson, Lambda resource policies for S3 event notifications can only be modified via the CLI or API — not the console.

---

## Supported Event Types

### Object Created

Triggered on `PUT`, `POST`, `COPY`, and completed **multi-part upload** operations.

**Typical use case**: Process an uploaded image — add a watermark, generate thumbnails, run ML inference, or apply a transformation (e.g., pixel-art conversion).

### Object Removed

Matches deletion operations using the `*` wildcard, including:

- Standard delete operations
- Delete marker creation (for versioned buckets)

**Typical use case**: Security automation — alert or auto-remediate when objects are removed from sensitive buckets.

### Object Restore

Triggered when restore operations from **S3 Glacier** or **Glacier Deep Archive** _start_ and _complete_.

**Typical use case**: Notify users/staff when their archived data has begun or finished restoring (long restore windows make this valuable).

### Object Replication

Fires on replication conditions, including:

- Operations that **miss** the 15-minute replication threshold
- Operations **replicated after** the threshold
- Objects that are **no longer tracked**
- **Failed** replication

**Typical use case**: Monitor cross-region replication health for compliance or DR posture.

---

## Event Payload Structure

Events are delivered as **JSON objects**. A simplified example:

```json
{
  "s3": {
    "configurationId": "ID found in the bucket notification configuration",
    "bucket": {
      "name": "bucket-name",
      "ownerIdentity": {
        "principalId": "Amazon-customer-ID-of-the-bucket-owner"
      },
      "arn": "bucket-ARN"
    },
    "object": {
      "key": "object-key"
    }
  }
}
```

When using **Lambda** as a destination, the function receives this structure inside the `event` parameter. The function must **parse** the JSON, extract the bucket name and object key (and other fields as needed), and then act on it.

---

## Resource Policy Requirement

S3 is the **principal** that invokes/delivers to the destination. Therefore:

|Destination|Required Policy|
|---|---|
|Lambda|Lambda resource (function) policy granting `s3.amazonaws.com` permission to invoke|
|SQS|SQS queue policy granting `s3.amazonaws.com` permission to `SendMessage`|
|SNS|SNS topic policy granting `s3.amazonaws.com` permission to `Publish`|

Without the resource-side permission, event delivery silently fails.

---

## S3 Event Notifications vs. EventBridge

S3 also integrates with **Amazon EventBridge**, which is a more modern and capable alternative:

|Aspect|S3 Event Notifications|EventBridge|
|---|---|---|
|Age|Older feature|Newer integration|
|Event types supported|Limited (create/delete/restore/replicate)|Broader set|
|Destination services|Lambda, SQS, SNS only|Many AWS services + SaaS partners + custom buses|
|Filtering|Basic (prefix/suffix on object key)|Rich event-pattern matching|
|Recommended default|When you specifically need this feature|**General-purpose default**|

> 💡 **Rule of thumb**: Default to **EventBridge** unless you have a specific reason to use S3 Event Notifications. Still, understand the native S3 feature — it appears on the exam.

---

## Exam Cheat Sheet

|Keyword / Scenario|Answer|
|---|---|
|"Notify when object uploaded"|S3 Event Notifications (or EventBridge)|
|"Trigger Lambda when file lands in S3"|S3 Event Notifications → Lambda|
|Supported destinations (native S3 events)|**Lambda, SQS, SNS** (only these three)|
|"Event types supported"|Create, Delete, Restore, Replicate|
|Create-event sub-operations|PUT, POST, COPY, multi-part upload completion|
|Restore events fire when...|Restore **starts** AND **completes** (from Glacier / Glacier Deep Archive)|
|Replication events fire on...|Miss threshold, replicated after threshold, no-longer-tracked, failed replication|
|What allows S3 to deliver to destination?|**Resource policy** on the destination (S3 service principal)|
|How to modify Lambda resource policy for this?|**CLI or API** (not console, at time of lesson)|
|Event payload format|**JSON**|
|Need richer events or more destinations?|Use **EventBridge** instead|
|Default recommendation|**EventBridge**, unless specific reason for S3 Event Notifications|

---

## Quick Decision Guide

**Use S3 Event Notifications when:**

- You only need Lambda, SQS, or SNS as a destination
- The event types (create/delete/restore/replicate) cover your needs
- You want the simplest possible bucket-local configuration

**Use EventBridge when:**

- You need additional event types
- You want to fan out to services other than Lambda/SQS/SNS
- You need advanced event-pattern filtering
- You want a centralized event bus across multiple sources