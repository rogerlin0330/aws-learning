---

tags:

- aws
- saa
- sns
- messaging
- integration

---

# Simple Notification Service (SNS)

## Overview

SNS is a **highly available, durable, secure, fully managed pub/sub messaging service**. It coordinates the sending and delivery of **messages** (payloads up to **256 KB**) between **publishers** and **subscribers**.

- **Public AWS service** — runs from the AWS public zone. To reach it you need network connectivity to the public AWS endpoints (plus IAM permissions). Accessible from the public internet or from a VPC configured to reach public AWS endpoints.
- **Base entity = the SNS topic.** Topics are where permissions are controlled (via a resource/topic policy) and where most configuration lives.
- **Pub/sub model:** publishers send messages _into_ a topic; subscribers _receive_ from the topic. By default every subscriber receives every message sent to the topic.
- **Message size:** up to 256 KB. 🎯 The course frames this as "you can't send an entire cat movie" — SNS is not designed for large binary payloads. You don't need the exact number for the exam, but the _concept_ (small notification payloads, not bulk data) matters.

> 📚 **Beyond the transcript — the 256 KB ceiling isn't absolute.** The **SNS Extended Client Libraries (Java and Python)** let you publish payloads up to **2 GB** by storing the actual payload in S3 and publishing only an S3 reference to the topic; subscribers de-reference from S3. This is the same pattern SQS uses. Source: [AWS — Publishing large messages with SNS and S3](https://docs.aws.amazon.com/sns/latest/dg/large-message-payloads.html). _Flagged for your review — mention if you want this promoted or dropped._

Related notes: [[SQS]] · [[Lambda]] · [[CloudWatch]] · [[CloudFormation]] · [[Auto Scaling Groups]]

## Diagram Walkthrough

![[sns-architecture.png]]

**Read the diagram in three bands (context bar → main objects → footer):**

**Context bar — the three horizontal zones (top to bottom):**

- **Public Internet** (red band, top) — external entities like a third-party **API** and **Mobile Push** targets live here, reachable because SNS is a public service.
- **Public AWS Zone** (tan band, middle) — this is where the **SNS service and its Topic** actually run. CloudWatch also sits here as an AWS-service publisher.
- **Private Zone (VPC)** (green band, bottom) — your own resources (**EC2**, **ASG**, **CFN Stack**) that publish into the topic from inside a VPC.

The **dotted red boundary box** around the Topic marks the SNS service edge — everything inside it is SNS; arrows crossing it are publish (in) or deliver (out) flows.

**Main objects — the Topic and its two arrow colors:**

- **Orange arrows = publishers → topic** (inbound). Sources shown: CloudWatch, an external API, EC2, ASG, and a CloudFormation stack. Note these originate from all three zones — public internet, public AWS zone, and the VPC — illustrating that anything with connectivity + permissions can publish.
- **Blue arrows = topic → subscribers** (outbound delivery). Targets shown: the API (so the API is _both_ a publisher and a subscriber — same entity, two roles), Mobile Push, Email, **SQS Queues**, and **Lambda**.
- The Topic is drawn as **two stacked shapes** (a solid bar and a dashed bar). Read this as the topic holding both an unfiltered path and a filtered subscription — it sets up the Filter callout below.

**Footer / annotations — the two labeled patterns:**

- **"Filter"** (pink label, with the dotted circle on one blue delivery line) — a **subscription filter policy**. A subscriber can attach a filter so it receives only the subset of messages relevant to it, rather than the default (all messages). In the diagram the filter sits on the line heading toward Lambda.
- **"Fanout"** (pink label, bottom right, with the curved pink arrow pointing up into the SQS Queues block) — the fanout pattern: **one topic → multiple SQS queues** as subscribers. The pink arrow deliberately loops back up to the _stack_ of SQS queues to emphasize "one message, many queues."

🎯 Cross-reference the arrow colors when reasoning about the diagram: **orange = in (publish), blue = out (deliver).** An entity touched by both (the API) is simultaneously a producer and a subscriber.

## Mechanics

### Publishers

Anything that sends messages to a topic. Can be external APIs on the public internet, or AWS services. SNS is used _across_ AWS:

- **CloudWatch** — publishes when an **alarm changes state**.
- **CloudFormation** — publishes when a **stack changes state**.
- **Auto Scaling Groups** — can publish when a **scaling event** occurs.
- Plus EC2, custom apps, and many other AWS services.

### Subscribers (endpoint types)

By default a subscriber receives all topic messages. Types shown/mentioned:

- **HTTP / HTTPS endpoints**
- **Email** (and email-JSON)
- **SQS queues** — each message is enqueued as it arrives at the topic
- **Mobile push notifications**
- **SMS** (text messages)
- **Lambda functions** — the function is _invoked_ per message

> 📚 AWS groups these as **A2A** (application-to-application: SQS, Lambda, HTTP/S, Data Firehose) and **A2P** (application-to-person: SMS, mobile push, email). Source: [AWS SNS features](https://aws.amazon.com/sns/features). _Editorial framing — flag if you want it in._

### Filtering

A **subscription filter policy** lets a subscriber receive only messages relevant to its function, instead of the default (all messages). Set per subscription.

### Fanout

**One SNS topic → multiple SQS queues** subscribed to it. A single message (representing one workload) is delivered into every subscribed queue, letting multiple related-but-isolated workers each process it their own way.

- Course example: a video-processing job published once, fanned out to several queues that each transcode a **different bitrate / resolution** variant.
- 🎯 Exam trigger: _"deliver a single message to multiple queues / decouple one event into several parallel workloads"_ → **SNS fanout to SQS**.
- See [[SQS]] for queue mechanics.

### Delivery status & retries (reliable delivery)

- **Delivery status:** confirmable for certain subscriber types — **HTTP/HTTPS, Lambda, and SQS** (per transcript).
- **Delivery retries:** SNS retries failed deliveries per a delivery policy.

> 📚 **Retry / DLQ detail beyond the transcript (production nuance, not exam-depth):**
> 
> - For **customer-managed HTTP/S endpoints**, the default policy is **50 total attempts** before the message is discarded. HTTP/S is the _only_ protocol whose delivery policy you can customize.
> - For **AWS-managed endpoints (SQS, Lambda)**, SNS retries far more aggressively — up to ~**100,015 attempts over 23 days**.
> - A message discarded after retries is **lost by default** unless the subscription has a **dead-letter queue (DLQ)** attached. An SNS DLQ is just a standard **SQS queue**.
> - Sources: [AWS — Message delivery retries](https://docs.aws.amazon.com/sns/latest/dg/sns-message-delivery-retries.html), [AWS — SNS dead-letter queues](https://docs.aws.amazon.com/sns/latest/dg/sns-dead-letter-queues.html). _Flagged — deeper than the course goes; keep or trim as you prefer._

### Resilience, encryption, access

- **Regionally resilient / highly available:** data sent to SNS is replicated within the region; a topic keeps functioning if individual AZs fail. Scales from near-zero to highly transactional workloads within the region.
- **Server-Side Encryption (SSE):** data persisted to disk can be stored encrypted — relevant when on-disk encryption is mandated.
- **Cross-account access:** apply a **topic policy** (a resource policy, exactly like an [[S3]] bucket policy) to control which identities can access the topic.

## Course-vs-Reality Discrepancies

⚠️ **Standard vs. FIFO topics — the course doesn't distinguish them.** The transcript treats "topic" as one thing, but SNS has **two topic types**, and this is testable on current SAA-C03:

|                  | **Standard topic**                                    | **FIFO topic**                                                       |
| ---------------- | ----------------------------------------------------- | -------------------------------------------------------------------- |
| Ordering         | Best-effort (may arrive out of order)                 | Strict, first-in-first-out (within a message group)                  |
| Deduplication    | Best-effort (at-least-once)                           | Strict (5-minute dedup window)                                       |
| Throughput       | Nearly unlimited                                      | Up to **3,000 msg/s or 10 MB/s** per topic                           |
| Subscriber types | All (SQS, Lambda, HTTP/S, email, SMS, push, Firehose) | **SQS queues only** — cannot deliver to email/HTTP/SMS/push directly |

Source: [AWS SNS features](https://aws.amazon.com/sns/features), [AWS — FIFO message delivery](https://docs.aws.amazon.com/sns/latest/dg/fifo-message-delivery.html).

⚠️ **Older FIFO throughput figure.** Some secondary sources still cite **300 msg/s** for FIFO topics; the current AWS-published limit is **3,000 msg/s (or 10 MB/s), whichever comes first**. Trust the AWS features page over blog aggregators.

## Exam Traps

- 🎯 **"Public service"** → SNS needs connectivity to public AWS endpoints + IAM perms; it is _not_ something you place inside a VPC subnet. Reachable from internet or from a VPC with public-endpoint access.
- 🎯 **Fanout** = one topic → many **SQS** queues. If a question describes duplicating one event across several parallel processors, the answer is SNS → SQS fanout, not multiple direct publishes.
- 🎯 **Filtering** happens at the **subscription** level (filter policy), letting a subscriber opt into a subset — don't confuse with routing logic in the publisher.
- 🎯 **Cross-account** access to a topic = **topic policy** (resource policy), the S3-bucket-policy analogy.
- 🎯 **FIFO topic ⇒ SQS-only subscribers.** If a scenario needs strict ordering _and_ email/SMS/HTTP delivery, that's a mismatch — FIFO can't deliver to those endpoints.
- 🎯 **256 KB** message ceiling → SNS is for notifications, not bulk/binary data. (Large payloads need the S3-backed Extended Client Library.)
- 🎯 **Delivery status supported for HTTP/S, Lambda, SQS** — a common "which endpoints can confirm delivery" distractor.

## Quick-Reference Table

| Concept                     | Key fact                                                                  |
| --------------------------- | ------------------------------------------------------------------------- |
| Service type                | Public, regional, fully managed pub/sub                                   |
| Base entity                 | Topic (holds permissions + config)                                        |
| Publisher                   | Sends messages into a topic (CloudWatch, CFN, ASG, EC2, APIs, …)          |
| Subscriber                  | Receives from topic (HTTP/S, email, SQS, Lambda, SMS, mobile push)        |
| Default subscriber behavior | Receives **all** topic messages                                           |
| Filter                      | Per-subscription filter policy → receive only relevant subset             |
| Fanout                      | 1 topic → many SQS queues                                                 |
| Message size                | 256 KB native; up to 2 GB via S3-backed Extended Client Library           |
| Delivery status             | HTTP/S, Lambda, SQS                                                       |
| Retries / DLQ               | Retries per delivery policy; discarded unless DLQ (an SQS queue) attached |
| Resilience                  | Regionally resilient (replicated in-region, AZ-fault tolerant)            |
| Encryption                  | SSE for data at rest                                                      |
| Cross-account               | Topic policy (resource policy)                                            |
| Topic types                 | Standard (best-effort order/dedup) vs FIFO (strict, SQS-only subscribers) |

## Open Questions (for Roger)

1. **FIFO depth.** The course omits FIFO topics entirely, but they're on current SAA-C03. Want a dedicated **[[SNS FIFO Topics]]** sub-note (message groups, dedup ID, SNS-FIFO + SQS-FIFO pairing), or is the discrepancy table here enough for exam purposes?
2. **DLQ / retry mechanics.** I flagged the 50-attempt (HTTP/S) vs ~100,015-attempt (AWS-managed) split and DLQ behavior as 📚 production nuance. Keep it, or trim to just "retries + optional DLQ" for exam-depth?
3. **Cross-links.** I've stubbed wikilinks to [[SQS]], [[Lambda]], [[CloudWatch]], [[CloudFormation]], [[Auto Scaling Groups]], and [[S3]]. Do [[SQS]] and [[S3]] already exist in your vault, or should the SQS one wait until you reach that lesson (it's later in this section)?
4. **Extended Client Library (2 GB via S3).** Promoted this to a callout since it's a common "how do I exceed 256 KB" answer. Worth keeping at exam level, or is it deeper than SAA-C03 needs?