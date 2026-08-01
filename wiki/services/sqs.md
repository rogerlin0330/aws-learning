---
title: Simple Queue Service (SQS)
type: service
tags: [sqs, messaging, queues, decoupling, fifo, dead-letter-queue, delay-queue, visibility-timeout, polling, fanout, integration]
certs: [aws-saa]
updated: 2026-07-31
---

# Simple Queue Service (SQS)

## Overview

SQS provides **fully managed message queues**. You create a queue and AWS delivers it as a service — no replication, resiliency, or capacity management on your side. Queues are **highly available and highly performant by design**, within a region.

- **Public service.** SQS runs from the AWS public zone. Reaching it needs network connectivity to the public AWS endpoints plus IAM permissions — including from inside a private VPC that has a route to public AWS services. It is **not** something you place inside a VPC subnet, exactly like [SNS](sns.md).
- **Regional and regionally resilient.** Messages are replicated across AZs within the region. See [Resource Scope](../concepts/resource-scope.md).
- **Two queue types:** **standard** and **FIFO** (full comparison in [Standard vs FIFO](#standard-vs-fifo-queues)).
- **Message size:** up to **1 MiB** per message (see the callout below — the course-era figure was 256 KB). For anything larger, store the payload in [S3](s3.md) and put a pointer to the S3 object in the message (the "claim-check" pattern), which is what the **SQS Extended Client Libraries** for Java and Python automate, up to **2 GB**. Architecturally, keep messages small regardless — they are easier to process and manage at scale.
- **Retention:** configurable from **1 minute to 14 days**, default **4 days** ([SQS message quotas](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/quotas-messages.html)).

> [!warning] Course-vs-current-AWS — the message size ceiling moved to 1 MiB
> The Cantrill course and most exam material state **256 KB**. AWS **raised the maximum SQS payload to 1 MiB (1,048,576 bytes) on 4 August 2025**, across all commercial Regions and GovCloud, applied automatically to new and existing queues; [Lambda](lambda.md) event source mappings for SQS were updated to match ([AWS — Amazon SQS increases maximum message payload size to 1 MiB](https://aws.amazon.com/about-aws/whats-new/2025/08/amazon-sqs-max-payload-size-1mib), [SQS message quotas](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/quotas-messages.html)). Two things to keep straight:
>
> - **256 KB is still the likely expected exam answer** for SAA-C03, and it remains correct for [SNS](sns.md), which did **not** get the increase.
> - AWS's own **general reference quota table still lists "Message Size: 256 Kilobytes"** while the Developer Guide and the pricing page both say 1 MiB. The Developer Guide is the authoritative one here.

Related services: [SNS](sns.md), [Lambda](lambda.md), [EC2 Auto Scaling](ec2-auto-scaling.md), [S3](s3.md), [EventBridge](eventbridge.md), [KMS](kms.md), [Step Functions](step-functions.md).

## Core Mechanic — Send, Poll, Visibility Timeout, Delete

The whole service rests on one loop:

1. **Producers send** messages to the queue.
2. **Consumers poll** the queue. *Polling* is simply checking for messages.
3. A polled message is **not deleted** — it is **hidden** for the **visibility timeout**.
4. The visibility timeout is the time the consumer has to do its work:
   - Finish processing, then **explicitly delete** the message, and it is gone forever.
   - Fail, crash, or otherwise never delete it, and once the visibility timeout expires the message **reappears** on the queue for another consumer.

Default visibility timeout is **30 seconds**, configurable from **0 seconds to 12 hours**. Set it longer than your maximum expected processing time; a consumer that needs more can call `ChangeMessageVisibility` to extend it mid-flight ([ChangeMessageVisibility API](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/APIReference/API_ChangeMessageVisibility.html)).

> [!tip] Exam Tip — the visibility timeout is fault tolerance for free
> If a consumer dies mid-processing, the queue's *default* action returns the message so another consumer picks it up. This is heavily tested. Be precise: the visibility timeout is **the time a message is hidden after being received**, and a message that is not explicitly deleted **comes back**.

Useful mechanics that hang off this:

- Setting `VisibilityTimeout` to **0** terminates the timeout immediately — the "I can't process this, let someone else try right now" lever ([SQS visibility timeout](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html)).
- Extending an in-flight message cannot push its total invisibility past the **12-hour ceiling measured from the original receive** — the clock does not reset. For work that genuinely needs longer, AWS points at [Step Functions](step-functions.md) or breaking the task into smaller steps.
- **The queue-level default** visibility timeout is a queue attribute set at creation or via `SetQueueAttributes`; `ChangeMessageVisibility` is the **per-message** lever for a message you have already received. The course phrasing "per queue or per message with `ChangeMessageVisibility`" blurs these two.
- Deletion is normally the explicit `DeleteMessage` action, though some AWS SDKs delete automatically on successful processing.
- A message that fails **repeatedly** should not loop forever — that is what a [Dead-Letter Queue](#dead-letter-queues-dlq) is for.

## Visibility Timeout vs Delay Queues

Both features hide messages, which is exactly why the exam pairs them. The distinction is **when the hiding starts**:

> A **delay queue** hides a message **the moment it is added**, before any consumer has seen it. A **visibility timeout** hides a message **only after a consumer receives it** — it was visible first.

![Two-band SQS timeline diagram: the top band shows the visibility timeout — SendMessage puts a visible message on the queue, ReceiveMessage starts processing and an eye icon marks the message going invisible for the Visibility Timeout while further receive calls return no messages, ending either in DeleteMessage on success or the message reappearing on the queue for reprocessing on failure; the bottom band shows a delay queue — SendMessage routes the message straight into a Delay (in Seconds) holding period during which receive calls return no messages, after which the message becomes visible and behaves normally](../assets/sqs-visibility-timeout-vs-delay-queues.png)

Read each band left to right as the life of one message along the pink "SQS Queue over Time" axis. The top band is the visibility timeout covered above, with its footer facts: default **30 s**, range **0 s to 12 h**. The bottom band is the delay queue: the message never appears on the visible queue at all until `DelaySeconds` elapses, so `ReceiveMessage` returns nothing even though nobody has consumed it.

### Delay queues and message timers

Delay queues exist to **build a delay into your processing** — give a downstream system time to catch up, or insert a gap between a customer action and the processing of the message representing it.

- Controlled by **`DelaySeconds`** on the queue; **default 0** (a queue is only a *delay* queue when this is non-zero); **maximum 15 minutes** (900 s) ([SQS delay queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-delay-queues.html)).
- **Message timers** set an initial invisibility period on an **individual message**, overriding the queue's `DelaySeconds` for that message. Same bounds, 0 to 15 minutes ([SQS message timers](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-message-timers.html)).
- **FIFO queues support delay queues but not message timers.** The per-message timer is the unsupported part; a queue-level `DelaySeconds` works fine on FIFO ([FIFO queue quotas](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/quotas-fifo.html)).
- Changing a queue's `DelaySeconds` is **not retroactive on standard queues** (messages already queued keep their original delay) but **is retroactive on FIFO queues**.

> [!warning] Gotcha — 15 minutes is a hard ceiling
> Delay queues and message timers both cap at 15 minutes. Anything longer is **EventBridge Scheduler**, not SQS — it schedules one-time or recurring actions with no 15-minute limit. See [EventBridge](eventbridge.md).

> [!tip] Exam Tip — anchor on when the hiding starts
> "Hidden the moment it is added / before any consumer sees it" is a **delay queue**. "Reappears for reprocessing after a consumer failed" is the **visibility timeout**. That single question resolves nearly every trap in this pair.

| | **Visibility Timeout** | **Delay Queue** |
| --- | --- | --- |
| When the message becomes invisible | After a consumer **receives** it | The moment it is **added** |
| Was it visible first? | Yes | No |
| Primary purpose | Error correction, automatic reprocessing | Build a processing delay into the app |
| Setting | `VisibilityTimeout` | `DelaySeconds` |
| Default | 30 seconds | 0 seconds |
| Range | 0 seconds to 12 hours | 0 to 15 minutes |
| Per-message override | `ChangeMessageVisibility` (in-flight) | Message timers (0 to 15 min) |
| FIFO per-message support | Supported | Message timers **not supported** |
| Need longer than the max? | [Step Functions](step-functions.md), or split the task | EventBridge Scheduler |

### In-flight message quotas

*In flight* means received but not yet deleted. Both queue types cap at **120,000 in-flight messages** — adjustable via Service Quotas for standard queues and by contacting AWS Support for FIFO ([SQS service quotas](https://docs.aws.amazon.com/general/latest/gr/sqs-service.html)). Behaviour at the cap depends on the polling mode: **short polling** returns an `OverLimit` error, while **long polling** simply returns nothing until the in-flight count drops.

## Standard vs FIFO Queues

The mental model the course builds the whole comparison on: **standard is a multi-lane highway, FIFO is a single-lane road with no overtaking.** What you trade is performance for preserved order.

![Two-band road diagram comparing SQS queue types: the top pink band, Multi Lane Highway (standard), shows two lanes of message boxes numbered out of sequence (5 and 3 in one lane, 4, 2 and 1 in the other) with a tall vertical capacity arrow labelled scalable and near-unlimited TPS, flask icons for best-effort ordering and at-least-once delivery, and use cases of decoupling, worker pools and batching; the bottom orange band, Single Lane Highway (FIFO), shows one lane in strict order 5, 4, 3, 2, 1 with a much shorter capacity arrow labelled 300 TPS without batching and 3,000 with, flask icons for exactly-once processing and strictly preserved order, the .fifo suffix requirement, and use cases of workflow ordering, command ordering and price adjustments](../assets/sqs-standard-vs-fifo.png)

The relative lengths of the two vertical capacity arrows are doing real work in the diagram — the standard arrow spans the full band, the FIFO one is compressed. That is the trade-off rendered spatially.

| | **Standard** | **FIFO** |
| --- | --- | --- |
| Ordering | Best-effort; messages can arrive out of order | Strictly preserved, first-in first-out |
| Delivery | **At-least-once** (duplicates possible) | **Exactly-once into the queue** (no duplicates introduced) |
| Scaling | Near-infinite, nearly unlimited TPS per API action | Throughput-limited by comparison |
| Name suffix | — | Must end in **`.fifo`** |
| Ideal for | Decoupling, worker pools, batching work for later | Workflow ordering, command ordering, sequential price adjustments |

### Throughput mechanics

The default FIFO numbers and what they mean:

- FIFO handles **300 transactions per second per API action**, and each transaction is one message, hence **300 messages/second unbatched**.
- **Batching packs 10 messages into one transaction**, hence **3,000 messages/second**. The 10× is not a separate limit — it falls straight out of batching.
- Standard queues support a **nearly unlimited number of API calls per second per action**.

**High throughput mode for FIFO** went generally available in May 2021 and is now region-tiered, well beyond the course-era figures ([SQS service quotas](https://docs.aws.amazon.com/general/latest/gr/sqs-service.html)):

| Region tier | Unbatched TPS per API action | Batched TPS |
| --- | --- | --- |
| us-east-1, us-west-2, eu-west-1 | 70,000 | 700,000 |
| us-east-2, eu-central-1 | 18,000 | 180,000 |
| ap-south-1, ap-southeast-1, ap-southeast-2, ap-northeast-1 | 9,000 | 90,000 |
| eu-west-2, sa-east-1 | 4,500 | 45,000 |
| All other Regions | 2,400 | 24,000 |

Under the hood, FIFO stores data in **partitions**, and the **message group ID is hashed to pick the partition**. Each partition supports about 300 messages/second unbatched or 3,000 batched, and SQS adds partitions as the request rate grows, up to the regional quota. That is why AWS's advice for more throughput is **more distinct message group IDs** ([high throughput for FIFO queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/high-throughput-fifo.html)).

> [!warning] Course-vs-current-AWS — "FIFO is slow" no longer holds absolutely
> The course records high throughput mode as being "in preview." It has been GA since 2021. The **300 / 3,000 figures remain correct as the default FIFO baseline** and remain the expected exam answer, but the framing that survives is **FIFO is throughput-limited *relative to standard***, not that FIFO is slow.

### The two delivery guarantees, and what makes them work

FIFO's exactly-once and strict-ordering claims come from two distinct mechanisms at two different points.

**Ingest side — `MessageDeduplicationId` protects exactly-once *into* the queue.** A producer that hits a network error or timeout retries, which is the normal way a duplicate is born. Dedup IDs are tracked in a **5-minute window**; a repeat send is **accepted and acknowledged as successful, but the payload is discarded** rather than written to the queue. The success acknowledgement matters — the producer stops retrying, so the duplicate dies quietly. Supply the ID explicitly, or enable **content-based deduplication** (SHA-256 of the message *body*, not its attributes). Provide neither and the send **fails** ([exactly-once processing](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/FIFO-queues-exactly-once-processing.html), [SendMessage API](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/APIReference/API_SendMessage.html)).

**Consuming side — `MessageGroupId` protects ordering.** Dedup prevents a second *copy* existing; it does not decide which later messages may be delivered while one is in flight. The message group is the ordering unit: messages in the same group are processed one at a time, in strict order, and the group does not advance until the in-flight messages are **deleted or become visible again** ([using the message group ID](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/using-messagegroupid-property.html)). Receive-side detail worth knowing ([FIFO queue delivery logic](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/FIFO-queues-understanding-logic.html)):

- A single receive can return **up to 10 messages from the same group**, in FIFO order within the batch, delivered to one consumer.
- Further delivery from that group is withheld until those messages are deleted **or** become visible again — so a stuck consumer unblocks its group by timeout, not only by deletion.
- **Batches can span groups.** If fewer than 10 messages are available for one group, SQS may fill the batch from other groups, each keeping its own order.
- **You cannot request a specific group** — there is no API parameter for it.
- **Parallelism comes from group count.** One group ID for the whole queue means strictly sequential processing.
- There is **no quota on the number of message groups** in a FIFO queue ([FIFO queue quotas](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/quotas-fifo.html)).

**Standard queues have neither mechanism**, which is exactly why at-least-once is what remains: a producer retry lands as a genuinely separate message, and nothing serialises consumption so nothing preserves order. On top of that, SQS stores messages **redundantly across multiple servers**; if one of those servers is unavailable when the delete request arrives, the surviving copy may be returned on a later receive ([at-least-once delivery](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/standard-queues-at-least-once-delivery.html)).

> [!warning] Gotcha — "exactly-once" does not mean your code runs once
> The guarantee is about **what reaches the queue**, not what your application executes. A consumer that overruns its visibility timeout, or commits a database write and then crashes before calling `DeleteMessage`, gets the message **delivered a second time — on FIFO exactly as on standard**. No duplicate was introduced into the queue, so the guarantee is intact. **Consumers must be idempotent regardless of queue type**; the usual pattern is tracking processed message IDs in a low-latency store such as DynamoDB.

| Cause of duplicate | Standard | FIFO |
| --- | --- | --- |
| Producer resends after no acknowledgement | Duplicate persists | **Prevented** by dedup ID, 5-minute window |
| Consumer crash or visibility-timeout overrun | Redelivered | **Redelivered too** |

### Two layers of exclusivity — only one is FIFO-specific

It is easy to credit FIFO with something both queue types do:

| Layer | What it protects | Applies to |
| --- | --- | --- |
| **Visibility timeout** | The messages *you received* — hidden from every other consumer while in flight | **Both** standard and FIFO |
| **Message group hold** | The *group's continuation* — no further messages from that group reach anyone until yours are deleted or visible again | **FIFO only** |

Receiving a batch of 10 from a **standard** queue already gives you exclusive possession of those 10; that is the visibility timeout, not a FIFO feature. What FIFO adds is the second layer, and its absence is precisely **why standard cannot preserve order** — with no group hold, the next messages flow to whichever consumer polls next, in parallel with your in-flight batch.

> [!warning] Course-vs-current-AWS — `MessageGroupId` now works on standard queues too
> The course presents `MessageGroupId` as purely a FIFO concept. It is now accepted on **standard** queues, where it does something entirely different: it enables **fair queues**, using the value as a **tenant identifier** to mitigate noisy neighbours. SQS watches how in-flight messages are distributed across tenants and prioritises returning *other* tenants' messages, keeping their dwell time low while the noisy tenant's backlog drains — with virtually unlimited throughput and no limit on tenant count, precisely because nothing is being locked ([fair queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-fair-queues.html)). **It does not enforce ordering.** If strict ordering is required, you need a FIFO queue.

## Dead-Letter Queues (DLQ)

A **dead-letter queue** handles *reoccurring* failures. Without one, a single "poison" message that keeps failing processing cycles through the queue forever: received, hidden for the visibility timeout, never deleted, reappears, received again, and so on — burning compute indefinitely. A DLQ breaks the loop by counting receives and, past a threshold, moving the problem message out of the **source queue** into a separate queue for isolated handling.

A DLQ is **not a special queue type**. It is an ordinary SQS queue that another queue has been configured to dump failed messages into.

![SQS dead-letter queue diagram: a redrive policy note at top-left defines the source queue, the dead-letter queue and the condition, with the trigger stated at top-right as ReceiveCount greater than maxReceiveCount and the message not deleted; the middle dashed box is the source queue, drawn as a left-to-right timeline alternating between a visible message icon and a grey Visibility Timeout block, with blue arrows underneath labelled ReceiveMessage where processing fails and a ReceiveCount incrementing 1, 2, 3, 4, 5; a pink arrow on the right curls down into the bottom dashed dead-letter queue box marked with a tombstone, which notes that the enqueue timestamp of the message is unchanged and that DLQ retention is generally longer than the source queue's](../assets/sqs-dead-letter-queues.png)

Read the diagram top to bottom. The **redrive policy** governs the flow, the **source queue** is the timeline in the middle showing one message's repeated failure, and the **dead-letter queue** is the box at the bottom.

### `ReceiveCount` and `maxReceiveCount`

- Every message carries a **`ReceiveCount`** attribute, incremented each time it is received by a consumer.
- The **redrive policy** is set on the **source** queue and names three things: the source queue, the target DLQ, and **`maxReceiveCount`**.
- When a message's `ReceiveCount` **exceeds** `maxReceiveCount` and it has not been explicitly deleted, SQS moves it to the DLQ. The **default `maxReceiveCount` is 10** if you configure a redrive policy without specifying one ([using dead-letter queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html)).
- A **single DLQ can serve multiple source queues** — you do not need one per queue.
- A DLQ must be the **same type** as its source (standard with standard, FIFO with FIFO) and live in the **same account and Region**.

Why bother: a [CloudWatch](cloudwatch.md) alarm on the DLQ notifies you the moment any message fails permanently; the quarantined messages can be inspected for root cause; and you can attach *different* processing logic to problem messages that the main pipeline cannot handle.

> [!warning] Gotcha — the enqueue timestamp is not reset on the move
> Every queue has a retention period, and a message that ages past it is dropped. When a message moves from a source queue to a DLQ, its **enqueue timestamp keeps the original source-queue value**, so the DLQ retention clock is measured from when the message first entered the *source* queue. Worked example: a message sits in the source queue for **1 day**, then moves to a DLQ with **2-day** retention — it survives **1 more day**, not a fresh 2. **Design rule: make the DLQ's retention period longer than the source queue's**, or failed messages expire before you can inspect them.

### Redrive allow policy and redrive to source

Two features the course predates:

- **Redrive allow policy** — set on the **DLQ**, controlling *which* source queues may target it. Three modes: **`allowAll`** (the default, any queue in the same account and Region), **`byQueue`** (only listed queues, up to **10** source queue ARNs), and **`denyAll`**. Do not confuse it with the redrive policy: the **redrive policy sits on the source queue** and points at the DLQ, while the **redrive allow policy sits on the DLQ** and controls who may use it ([configure a dead-letter queue redrive](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-configure-dead-letter-queue-redrive.html)).
- **DLQ redrive to source** — once the consumer bug is fixed, you can move messages **back out of the DLQ** to the source queue, or to any other queue of the same type. A redrive task runs up to 36 hours, with a maximum of 100 active tasks per account and a tunable redrive velocity.

> [!warning] Course-vs-current-AWS — DLQ redrive is no longer standard-only
> Redrive back to source was originally limited to standard queues. AWS **added FIFO dead-letter queue redrive on 27 November 2023**, so a FIFO DLQ can redrive to a FIFO source or a FIFO custom destination. The rule that survives is the **type-match** one: both ends must be the same queue type ([AWS — Amazon SQS announces support for FIFO dead-letter queue redrive](https://aws.amazon.com/about-aws/whats-new/2023/11/amazon-sqs-fifo-dead-letter-queue-redrive/)).

## Short Polling vs Long Polling

| | **Short polling** | **Long polling** |
| --- | --- | --- |
| Behaviour | Returns **immediately**, even with 0 messages | **Waits** up to `WaitTimeSeconds` for messages to arrive |
| Wait time | 0 | Up to **20 seconds** |
| Cost efficiency | Poor — empty receives still consume billable requests | Better — far fewer requests |
| Return | 0 or more messages | Up to 10 messages, counted as one request |

`ReceiveMessageWaitTimeSeconds` ranges from **0 to 20 seconds**; any non-zero value enables long polling, and the default of 0 means short polling ([configuring queue parameters](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-configure-queue-parameters.html)).

Keeping a queue near-empty with short polling requires an almost constant stream of empty, billable polls.

> [!tip] Exam Tip — long polling is the recommended default
> "Reduce cost", "stop empty receives", or "eliminate empty responses" all point at **long polling** with `WaitTimeSeconds` up to 20 seconds.

## Billing — Requests, Not Messages

**You are billed per request, and a request is not the same thing as a message.** Every SQS action counts as a request. A single request can carry **1 to 10 messages, up to a maximum total payload of 1 MiB**, and **each 64 KB chunk of a payload is billed as one request** — so a 1 MiB API action bills as **16 requests** ([SQS pricing](https://aws.amazon.com/sqs/pricing/)).

Two consequences:

- A request that returns **zero messages is still billable**, which is why frequent short polling is expensive and long polling is the fix.
- Larger payloads cost proportionally more even in a single call, because of the 64 KB chunking.

> [!warning] Gotcha — 64 KB is a billing unit, not a batch ceiling
> Course-era notes often say a request returns "up to 10 messages, up to 64 KB total." The **10 messages** part is right; the 64 KB is the **billing chunk size**, not a cap on what a request may carry. The batch payload cap is **1 MiB**.

## Scaling and Integration Patterns

Queues **decouple** application components: one component adds to the queue, another reads from it, and neither needs awareness of the other. See [Architecture Evolution](../patterns/architecture-evolution.md) for where this sits in the monolith-to-event-driven progression.

- **[Auto Scaling Groups](ec2-auto-scaling.md)** can scale on **queue length** (`ApproximateNumberOfMessagesVisible`), not CPU.
- **[Lambda](lambda.md)** can be invoked when messages appear, via an **event source mapping** that polls the queue on the function's behalf.
- Together these give you **worker-pool architectures** that scale **all the way to zero** when the queue empties.

**Worker-pool example (single queue):** a web-app ASG scaling on CPU uploads a master video to [S3](s3.md) and drops a message on SQS; a worker ASG scaling on queue length polls the queue, fetches the master from S3, transcodes it, writes to a second bucket, then deletes the message. If a worker fails, the message reappears after the visibility timeout and another worker picks it up.

### SNS fanout to multiple SQS queues

![SNS and SQS fanout architecture: on the right a user interacts with an EC2 web server instance inside a dashed Auto Scaling Group, which stores the uploaded master video in an S3 Master File Bucket and publishes a Message containing the video's S3 location to an SNS topic in the centre; three SQS queues are subscribed to that topic, and on the left each queue feeds its own dashed Auto Scaling Group and EC2 encoding instance for the 480p, 720p and 1080p outputs; blue arrows show each encoding instance pulling the master video and writing its transcoded result to the S3 Transcoded File Bucket at the bottom, from which the web tier serves the different sizes back to the user](../assets/sns-and-sqs-fanout-architecture.png)

Reading the diagram, pink arrows are the ingest and message path while blue arrows are the encoding worker paths:

- **Right, the producer tier.** The user interacts with an EC2 web server in its own ASG scaling on **CPU load**. The web server stores the uploaded master video in the S3 master bucket and drops a **message onto the SNS topic** containing a **link to the master video's S3 location**, never the video itself.
- **Centre, the fanout point.** One [SNS](sns.md) topic has **three SQS queues subscribed**. One published message is delivered into **every** subscribed queue, so one upload becomes three independent jobs.
- **Left, the consumer tiers.** Three independent pipelines, one per output size (480p, 720p, 1080p), each with its own queue, its own ASG, and its own encoding instance. **Each ASG scales on the length of its own queue**, so each size scales independently on its own backlog.
- **Bottom, the output path.** Each encoder pulls the master from S3, transcodes, and writes to the transcoded bucket, from which the web tier serves the sizes back to the user.

**Why fanout exists:** an [S3 event notification](s3.md#event-notifications) for an object upload is **one event** delivered to **one destination**, and S3 does not allow overlapping notification configurations for the same event type and prefix/suffix. To turn one event into multiple independent events, you publish to one SNS topic with multiple SQS subscribers. The message lands in every queue, so multiple jobs start per single upload, each scaling on its own queue.

> [!note] The modern alternative
> Routing the S3 event through **[EventBridge](eventbridge.md)** instead gets you multiple rules and multiple targets from one event without an SNS topic in the middle — see the [native events vs EventBridge comparison](s3.md#event-notifications). SNS fanout to SQS remains the pattern the course teaches and the one most exam questions are written against.

> [!tip] Exam Tip — heavily emphasised
> If a question describes **one upload or event that must trigger several independent parallel processing pipelines**, the answer is **SNS fanout to multiple SQS queues**. The course flags this as one of the most reliably tested patterns in the messaging domain.

## Encryption

- **At rest** — server-side encryption via **[KMS](kms.md)** (SSE-KMS) or **SSE-SQS** with SQS-managed keys. Relevant because messages can persist for up to 14 days.
- **In transit** — traffic between SQS and its clients is encrypted **by default**.

> [!tip] Exam Tip
> Know the difference between **at rest** (data on disk) and **in transit** (data on the wire). They are not the same thing, and questions mixing them are common.

## Access Control

Access to a queue is controlled by **identity policies** or a **queue policy**:

- **Identity policies or queue policies** control access from the **same account**.
- **Queue policies only** can grant access to **external accounts** — an identity policy in your account cannot grant anything to a principal in another one.
- A **queue policy is a resource policy**, the same family as an [S3 bucket policy](s3.md#bucket-policies) or an SNS topic policy. See [Resource Policies vs Identity Policies](../concepts/resource-vs-identity-policies.md).

## Exam Traps

> [!tip] SQS trigger keywords
>
> | If the question mentions... | Think... |
> | --- | --- |
> | One upload or event driving multiple independent parallel pipelines | **SNS fanout to multiple SQS queues** |
> | Message reappears and another worker picks it up after a failure | **Visibility timeout** (not deleted, so it comes back) |
> | Order must be preserved, or no duplicates at ingest | **FIFO queue** plus `MessageDeduplicationId` |
> | Maximum throughput, order not critical | **Standard queue** with idempotent consumers |
> | Reduce cost, stop empty polls, eliminate empty receives | **Long polling**, `WaitTimeSeconds` up to 20 s |
> | Payload larger than the message ceiling | **S3 plus a pointer** (Extended Client Library, up to 2 GB) |
> | Message keeps failing, poison message, infinite reprocessing | **Dead-letter queue** and `maxReceiveCount` |
> | Notify me when messages fail permanently | **CloudWatch alarm on the DLQ** |
> | Message expired from the DLQ sooner than expected | **Enqueue timestamp is not reset** — DLQ retention runs from the original enqueue time |
> | Which queues may use this DLQ | **Redrive allow policy** (`allowAll` / `byQueue` / `denyAll`), not the redrive policy |
> | Reprocess messages after fixing the bug | **DLQ redrive** back to the source queue |
> | Hidden the moment it is added, before any consumer sees it | **Delay queue** (`DelaySeconds`) |
> | Per-message delay on a FIFO queue | **Message timers are not supported on FIFO** |
> | Delay longer than 15 minutes | **Not a delay queue** — EventBridge Scheduler |
> | Make an in-flight message immediately available again | `ChangeMessageVisibility` with `VisibilityTimeout = 0` |
> | Scale the worker fleet on backlog | **ASG scaling on queue length** (`ApproximateNumberOfMessagesVisible`) |
> | Consumer might process a message twice | **Idempotency** — FIFO does not solve this |
> | `MessageGroupId` on a **standard** queue | **Fair queues** (tenant fairness), *not* ordering |
> | Cross-account access to a queue | **Queue policy** (resource policy) |
> | Decouple components so each scales independently | **SQS between the tiers** |

## Quick Reference

| Attribute | Value |
| --- | --- |
| Service type | Public, regional, fully managed |
| Max message size | **1 MiB** since Aug 2025 (256 KB in course/exam material); larger goes to S3 plus a pointer, up to 2 GB |
| Retention | 1 minute to 14 days, default **4 days** |
| Visibility timeout | 0 s to 12 h, default **30 s** |
| Delay queue / message timer | 0 to **15 minutes**, default 0; timers unsupported on FIFO |
| Long poll wait time | 0 to **20 s** (non-zero enables long polling) |
| Messages per batch | Up to **10**, max total payload 1 MiB |
| In-flight limit | **120,000** for both standard and FIFO |
| Standard queue | At-least-once, best-effort order, nearly unlimited TPS |
| FIFO queue | Exactly-once into the queue, strict order, **`.fifo`** suffix required |
| FIFO throughput (default) | 300 TPS unbatched, 3,000 batched |
| FIFO throughput (high throughput mode) | Region-tiered, up to **70,000 TPS** unbatched / 700,000 batched |
| DLQ threshold | `ReceiveCount` exceeding `maxReceiveCount` (**default 10**) |
| DLQ constraints | Same queue type, same account, same Region; one DLQ can serve many sources |
| Billing unit | **Requests**, each 64 KB chunk billed as one |
| Encryption | At rest via KMS or SSE-SQS; in transit by default |
| Access control | Identity policy or queue policy; **queue policy** for cross-account |

## References

- [SNS](sns.md) · [Lambda](lambda.md) · [EC2 Auto Scaling](ec2-auto-scaling.md) · [S3](s3.md) · [EventBridge](eventbridge.md) · [KMS](kms.md) · [Step Functions](step-functions.md)
- [Architecture Evolution — Coupling, Queues & Event-Driven](../patterns/architecture-evolution.md) · [Serverless Architecture](../patterns/serverless-architecture.md) · [Resource Policies vs Identity Policies](../concepts/resource-vs-identity-policies.md)
- [Amazon SQS message quotas](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/quotas-messages.html)
- [Amazon SQS FIFO queue quotas](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/quotas-fifo.html)
- [Amazon SQS endpoints and service quotas](https://docs.aws.amazon.com/general/latest/gr/sqs-service.html)
- [Amazon SQS pricing](https://aws.amazon.com/sqs/pricing/)
- [Amazon SQS visibility timeout](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html) · [delay queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-delay-queues.html) · [message timers](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-message-timers.html)
- [Using dead-letter queues in Amazon SQS](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html) · [Configure a dead-letter queue redrive](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-configure-dead-letter-queue-redrive.html)
- [FIFO queue exactly-once processing](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/FIFO-queues-exactly-once-processing.html) · [FIFO delivery logic](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/FIFO-queues-understanding-logic.html) · [high throughput for FIFO queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/high-throughput-fifo.html)
- [Standard queue at-least-once delivery](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/standard-queues-at-least-once-delivery.html) · [Fair queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-fair-queues.html)
- [AWS — Amazon SQS increases maximum message payload size to 1 MiB](https://aws.amazon.com/about-aws/whats-new/2025/08/amazon-sqs-max-payload-size-1mib)
- [AWS — Amazon SQS announces support for FIFO dead-letter queue redrive](https://aws.amazon.com/about-aws/whats-new/2023/11/amazon-sqs-fifo-dead-letter-queue-redrive/)
- Adrian Cantrill, AWS SAA course — Simple Queue Service, SQS Standard vs FIFO Queues, SQS Dead-Letter Queues, SQS Delay Queues
- Sources: `raw/aws-cert-study-notes/aws-solution-architect/sqs.md`, `sqs_standard_vs_fifo.md`, `sqs_dead-letter_queues.md`, `sqs_delay_queues.md`
