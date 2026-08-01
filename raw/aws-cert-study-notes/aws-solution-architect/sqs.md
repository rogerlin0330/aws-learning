---

tags:

- aws
- saa
- sqs
- messaging
- decoupling

---

# Simple Queue Service (SQS)

## Overview

**SQS provides managed message queues.** It is a **public service** — accessible anywhere with access to the AWS public space endpoints, including private VPCs that have connectivity to public AWS services. It is **fully managed**: you create a queue, and the service delivers that queue as a service.

Queues are **highly available and highly performant by design**, so you don't manage replication or resiliency. This happens **within a region** by default.

Two queue types exist: **standard** and **FIFO** (covered in detail below).

- **Message size:** up to **256 KB** per message. For larger payloads, store the data on S3 and put a **link/pointer** to the S3 object inside the message (the "claim-check" pattern). 📚 Architecturally, keep messages small — they're easier to process and manage at scale.
- **Retention:** messages can live in a queue for a period of time (up to **14 days**). 📚 The configurable range is **1 minute → 14 days**, with a **default of 4 days**. ([AWS SQS FAQs](https://aws.amazon.com/sqs/faqs/))

---

## Core mechanic: send, poll, visibility timeout, delete

The fundamental queue loop:

1. **Clients send** messages to the queue.
2. **Other clients poll** the queue. _Polling_ is the process of checking for any messages on a queue.
3. When a client polls and receives messages, **those messages are not deleted** — they are **hidden** for a period called the **visibility timeout**.
4. The **visibility timeout** is the amount of time a client has to process the message.
    - If the client **finishes processing**, it **explicitly deletes** the message → it's gone forever.
    - If the client **does not delete** it (e.g. it failed or crashed), then **after the visibility timeout the message reappears** in the queue, available for processing by a different client.

📚 Default visibility timeout is **30 seconds**; configurable **0 seconds → 12 hours**. It should be set **longer than your maximum expected processing time**. If a consumer needs more time, it can call `ChangeMessageVisibility` to extend it. ([AWS Glossary — SQS](https://awsglossary.org/terms/sqs))

> [!important] Why this matters architecturally The visibility timeout gives you **fault tolerance for free**. If a client fails or dies mid-processing, the queue's default action returns the message to the queue so another client can pick it up. **This is a heavily tested exam concept** — be clear that the visibility timeout is _the amount of time a message is hidden after being received_, and if it isn't explicitly deleted it comes back.

---

## Dead-Letter Queues (DLQ)

A **dead-letter queue** is a queue where **problem messages can be moved**. For example, if a message is **received 5 or more times and never successfully deleted**, one possible outcome is to move it to the DLQ.

DLQs let you run **different processing** on messages that are problematic — e.g. corrupt messages, or messages that need a different handling style. Separate workloads can watch the DLQ.

📚 The "5 receives" figure is illustrative — the threshold (`maxReceiveCount`) is **configurable**. Messages that exceed it are redirected to the DLQ. 📚 SQS also supports **DLQ redrive** to move messages back to the source queue once the consumer code is fixed. ([AWS Messaging Studies](https://jbcodeforce.github.io/aws-messaging-study/sqs/))

---

## Diagram walkthrough — SNS + SQS Fanout

![[sns-and-sqs-fanout-architecture.png]]

**How to read it:** The **user is on the far right**. Flow moves **right → left** for the upload/ingest path, then the encoded outputs loop back **left → right** to the transcoded bucket. Two arrow colors are in play:

- **Pink/red arrows** = the ingest and message path (user → web app → master bucket, and the SNS/SQS message flow).
- **Blue arrows** = the encoding worker paths (encoding instances pulling the master video and pushing results to the transcoded bucket).

**Right side — the web (producer) tier:**

- The **user** interacts with the **EC2 (Web Server Instance)**, which sits in its own **ASG** (dashed box, scales on **CPU load**).
- The web server stores the uploaded **master video** into **S3 (Master File Bucket)**.
- It also drops a **Message** onto the **SNS Topic** (the dotted line from the master bucket represents the object-upload event conceptually driving the message). The message contains a **link to the master video's S3 location** — not the video itself.

**Center — the fanout point:**

- The single **SNS (Topic)** has **three SQS queues subscribed** to it (the three SQS icons stacked in the middle).
- One message published to the topic is **delivered into every subscribed queue** → one upload becomes three independent jobs.

**Left side — the three worker (consumer) tiers:**

- Three independent pipelines, one per output size: **480p**, **720p**, **1080p**.
- Each has its **own SQS queue**, its **own ASG** (dashed box), and its **own EC2 (Encoding Instance)**.
- Each ASG **scales on the length of _its own_ queue**, so each size scales independently based on its own backlog.

**Bottom — the output path:**

- Each encoding instance pulls the master video from the master bucket, transcodes it, and writes the result to **S3 (Transcoded File Bucket)**.
- The web tier later retrieves the different sizes from the transcoded bucket to present to the user.

---

## Why fanout? The S3 single-event problem

📚 The whole reason for the SNS→SQS fanout pattern:

> S3 buckets can generate an event when an object is uploaded — **but only one event**.

To turn that **one event into multiple independent events**, you use fanout: **one SNS topic → multiple SQS subscribers**. The message lands in **every** subscribed queue, letting **multiple jobs start per single object upload**, each scaling on its own queue.

🎯 **The instructor emphasizes this pattern _very_ heavily for the exam** ("I'm winking as much as I can"). If a question describes _one upload/event needing to trigger multiple independent parallel processing pipelines_, the answer is **SNS fanout to multiple SQS queues**.

---

## Standard vs FIFO queues

Mental model: **standard = multi-lane highway**; **FIFO = single-lane road with no overtaking**.

||**Standard**|**FIFO**|
|---|---|---|
|**Ordering**|Best-effort; messages **can** arrive out of order|**Strict order guaranteed** (first-in, first-out)|
|**Delivery**|**At-least-once** (duplicates possible)|**Exactly-once** (no duplicates introduced)|
|**Scaling**|Near-**infinite**, linear/fluid ("add more lanes")|**Limited** (see throughput note below)|
|**Name suffix**|—|Must end in **`.fifo`** 📚|

With a **standard** queue you could get the **same message delivered twice** on two different polls, and order may differ. **FIFO** guarantees both **exactly-once delivery** and **order preservation**.

### ⚠️ FIFO throughput — course figures are now outdated

The course states FIFO performance as **300 messages/sec without batching, 3,000/sec with batching**.

That is still the **default (non-high-throughput) FIFO baseline today**, but AWS has since added **high-throughput mode for FIFO queues**, raising the ceiling dramatically:

- Default FIFO: **300 TPS/API action** (~3,000 msg/s with batching). ✅ matches course.
- **High-throughput mode FIFO** now supports **up to 70,000 TPS per API action** in selected regions (up to ~700,000 msg/s with batching), with lower high-throughput quotas in other regions. ([AWS SQS message quotas](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/quotas-messages.html))

🎯 For the exam, the safe framing remains: **standard scales near-infinitely; FIFO is throughput-limited by comparison.** But know that FIFO is no longer as slow as the raw 300/3,000 numbers suggest — high-throughput mode exists.

📚 In-flight message quotas: **120,000** for standard and **120,000** for FIFO (messages received but not yet deleted). ([AWS SQS service quotas](https://docs.aws.amazon.com/general/latest/gr/sqs-service.html), [AWS SQS FIFO queue quotas](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/quotas-fifo.html))

---

## Billing model: requests, not messages

**You are billed per _request_, and a request ≠ a message.**

- A **request** is a single call to SQS.
- One request can return **0 to 10 messages**, up to **64 KB total**.
- A request can return **zero messages** and **still be billable**.

**Consequence:** SQS is **less cost-effective the more frequently you poll**, because frequent polling burns billable requests that may return nothing.

---

## Short polling vs long polling

||**Short polling**|**Long polling**|
|---|---|---|
|Behaviour|Returns **immediately**, even with 0 messages|**Waits** (`WaitTimeSeconds`) for messages to arrive|
|Wait time|0|Up to **20 seconds**|
|Cost efficiency|**Poor** — empty polls still consume billable requests|**Better** — fewer requests|
|Return|0 or more messages|Up to 10 messages / 64 KB, counted as one request|

Keeping a queue near-empty with **short polling** requires an almost constant stream of empty (billable) polls.

🎯 **Long polling is the recommended way to poll SQS** — it uses fewer requests and simply waits for messages to arrive.

📚 `ReceiveMessageWaitTimeSeconds` range is **0–20 s**; a non-zero value enables long polling (default 0 = short polling). ([SQS console config docs](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-configure-queue-parameters.html))

---

## Scaling & integration patterns

Queues **decouple** application components — one component adds to the queue, another reads from it; neither needs awareness of the other. They're also great for **scaling**:

- **Auto Scaling Groups** can scale on **queue length** (number of messages).
- **Lambda** can be **invoked when messages appear** on a queue.
- Together these enable **worker-pool architectures** (see the diagram walkthrough).

**Worker-pool example (single queue):** a web-app ASG (scaling on **CPU**) uploads the master video to S3 and drops a message on SQS; a worker ASG (scaling on **queue length**) polls, retrieves the master from S3, transcodes, writes to a second bucket, then deletes the message. If a worker fails, the message reappears after the visibility timeout and another worker picks it up. As the queue empties, workers scale back **all the way to zero**.

---

## Encryption

- **Encryption at rest** — supported via **KMS** (server-side encryption / SSE). Relevant because messages can persist up to 14 days. 📚 Also available: **SSE-SQS** (SQS-managed keys). ([AWS Glossary](https://awsglossary.org/terms/sqs))
- **Encryption in transit** — data is encrypted **by default** between SQS and clients.

🎯 Know the difference: **at rest** (data on disk) vs **in transit** (data on the wire). They are **not the same thing**.

---

## Access control

- Access to a queue is controlled by **identity policies** _or_ a **queue policy**.
- **Identity policies or queue policies** → control access **from the same account**.
- **Queue policies only** → can allow access **from external accounts**.
- A **queue policy is a resource policy**, like the ones used on S3 buckets or SNS topics.

---

## 🎯 Exam Traps — trigger keyword table

|If the question mentions…|Think…|
|---|---|
|One upload/event → multiple independent parallel pipelines|**SNS fanout → multiple SQS queues**|
|Message reappears / another worker picks it up after failure|**Visibility timeout** (message not deleted → reappears)|
|Order must be preserved / no duplicates|**FIFO queue** (exactly-once + ordering)|
|Maximum throughput / near-infinite scale, order not critical|**Standard queue**|
|Reduce cost / stop empty polls / reduce empty receives|**Long polling** (`WaitTimeSeconds` up to 20 s)|
|Payload > 256 KB|Store in **S3**, put pointer in the message|
|Problematic / repeatedly-failing messages|**Dead-letter queue** (after `maxReceiveCount`)|
|Scale worker fleet on backlog|**ASG scaling on queue length**|
|Cross-account access to a queue|**Queue policy** (resource policy)|
|Decouple components so each scales independently|**SQS between the tiers**|

---

## Quick reference

|Attribute|Value|
|---|---|
|Service type|Public, regional, fully managed|
|Max message size|256 KB (larger → S3 + pointer)|
|Retention|1 min – 14 days (default 4 days)|
|Visibility timeout|0 s – 12 h (default 30 s)|
|Long poll wait time|0 – 20 s|
|Messages per request|0 – 10 (up to 64 KB)|
|Standard queue|At-least-once, best-effort order, near-infinite scale|
|FIFO queue|Exactly-once, strict order, `.fifo` suffix|
|FIFO throughput (default)|300 TPS / ~3,000 msg/s batched|
|FIFO throughput (high-throughput mode)|up to 70,000 TPS ⚠️ post-course|
|In-flight limit|120,000 (standard) / 120,000 (FIFO)|
|Billing unit|**Requests** (not messages)|
|Encryption|At rest (KMS/SSE-SQS) + in transit (default)|
|Access control|Identity policy or queue policy; queue policy for cross-account|

---

## Open Questions for Roger

1. **Cross-linking** — which of these already exist as notes in the vault vs. need creating? `[[Simple Notification Service (SNS)]]`, `[[Auto Scaling Groups]]`, `[[S3 Event Notifications]]`, `[[Lambda]]`, `[[KMS]]`, `[[Resource Policies vs Identity Policies]]`.
2. **Sub-note candidates** — do you want standalone deep-dives on any of: **SQS Delay Queues / message timers** (mentioned in current AWS but not the transcript), **DLQ redrive**, **FIFO high-throughput mode mechanics** (message-group-ID partitioning)?
3. **Depth on FIFO throughput** — I flagged the 70K TPS update with a `⚠️`. Do you want the full per-region high-throughput breakdown, or is the "course says 300/3,000; reality is much higher now" framing enough for exam purposes?
4. **SNS note coordination** — this fanout diagram spans SNS + SQS. Should the diagram walkthrough live here, in the SNS note, or be duplicated/cross-referenced in both?

---

_Source: Adrian Cantrill SAA-C03 — "Simple Queue Service" lesson. Fact-checks cited inline against current AWS documentation (2026)._
