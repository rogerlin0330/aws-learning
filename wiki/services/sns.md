---
title: Simple Notification Service (SNS)
type: service
tags: [sns, messaging, integration, pubsub, fanout, decoupling]
certs: [aws-saa]
updated: 2026-07-29
---

# Simple Notification Service (SNS)

## Overview

SNS is a **highly available, durable, secure, fully managed pub/sub messaging service**. It coordinates the sending and delivery of **messages** (payloads up to **256 KB**) between **publishers** and **subscribers**. It is one of AWS's core application-integration and [decoupling](../patterns/architecture-evolution.md) primitives, sitting alongside SQS (queues) and [EventBridge](eventbridge.md) (event bus).

- **Public AWS service** — SNS runs from the AWS public zone. To reach it you need network connectivity to the public AWS endpoints plus IAM permissions. It is accessible from the public internet or from a VPC configured to reach public AWS endpoints. It is **not** something you place inside a VPC subnet.
- **Base entity is the SNS topic.** Topics are where permissions are controlled (via a resource/topic policy) and where most configuration lives.
- **Pub/sub model:** publishers send messages *into* a topic; subscribers *receive* from the topic. By default every subscriber receives every message sent to the topic.
- **Message size:** up to 256 KB. SNS is designed for small notification payloads, not bulk or binary data — you don't need the exact byte count for the exam, but the *concept* matters.

> [!note] Beyond 256 KB — the Extended Client Libraries
> The 256 KB ceiling isn't absolute. The **SNS Extended Client Libraries (Java and Python)** let you publish payloads up to **2 GB** by storing the actual payload in [S3](s3.md) and publishing only an S3 reference to the topic; subscribers de-reference the payload from S3. This is the same pattern SQS uses, and it's the standard answer to "how do I send a message larger than 256 KB." Source: [AWS — Publishing large messages with SNS and S3](https://docs.aws.amazon.com/sns/latest/dg/large-message-payloads.html).

Related services: SQS, [Lambda](lambda.md), [CloudWatch](cloudwatch.md), [CloudFormation](cloudformation.md), [EC2 Auto Scaling](ec2-auto-scaling.md), [S3](s3.md), [EventBridge](eventbridge.md).

## Architecture

![SNS architecture diagram: three horizontal zones — a red Public Internet band at top with a third-party API and Mobile Push targets, a tan Public AWS Zone in the middle where the SNS service and its Topic run alongside CloudWatch, and a green Private Zone (VPC) at the bottom with EC2, an Auto Scaling Group, and a CloudFormation stack; a dotted red boundary marks the SNS service edge; orange arrows show publishers writing into the topic and blue arrows show the topic delivering to subscribers (API, Mobile Push, Email, SQS queues, Lambda); a Filter label sits on the delivery line toward Lambda and a Fanout label with a curved arrow loops one message into a stack of SQS queues](../assets/sns-architecture.png)

Read the diagram in three bands.

**The three horizontal zones (top to bottom)** illustrate that SNS is a public service reachable from anywhere with connectivity and permissions:

- **Public Internet** (top) — external entities like a third-party **API** and **Mobile Push** targets.
- **Public AWS Zone** (middle) — where the **SNS service and its Topic** actually run. CloudWatch also sits here as an AWS-service publisher.
- **Private Zone (VPC)** (bottom) — your own resources (**EC2**, **ASG**, **CloudFormation stack**) publishing into the topic from inside a VPC.

The **dotted red boundary** around the Topic marks the SNS service edge: everything inside it is SNS, and arrows crossing it are publish (in) or deliver (out) flows.

**The two arrow colours** are the key to reasoning about the diagram:

- **Orange arrows are publishers writing to the topic** (inbound). Sources shown include CloudWatch, an external API, EC2, the ASG, and a CloudFormation stack. They originate from all three zones, showing that anything with connectivity plus permissions can publish.
- **Blue arrows are the topic delivering to subscribers** (outbound). Targets shown include the API, Mobile Push, Email, **SQS queues**, and **Lambda**.

The API appears on *both* an orange and a blue arrow, so it is simultaneously a **publisher and a subscriber** — the same entity in two roles. The two labelled patterns in the diagram, **Filter** (on the delivery line toward Lambda) and **Fanout** (the curved arrow looping into the stack of SQS queues), are covered under [Filtering](#filtering) and [Fanout](#fanout) below.

## Publishers

A publisher is anything that sends messages to a topic — an external API on the public internet, a custom app, or an AWS service. SNS is used *across* AWS as the notification backbone:

- **[CloudWatch](cloudwatch.md)** — publishes when an **alarm changes state** (the canonical "alarm notifies you" path).
- **[CloudFormation](cloudformation.md)** — publishes when a **stack changes state**.
- **[Auto Scaling Groups](ec2-auto-scaling.md)** — can publish when a **scaling event** occurs (and lifecycle hooks can target SNS).
- Plus [EC2](ec2.md), [S3 event notifications](s3.md#event-notifications), custom apps, and many other AWS services.

## Subscribers

A subscriber is an endpoint that receives messages from the topic. By default a subscriber receives **all** messages sent to the topic. Endpoint types:

- **HTTP / HTTPS endpoints**
- **Email** and **email-JSON**
- **SQS queues** — each message is enqueued as it arrives at the topic
- **Mobile push notifications**
- **SMS** text messages
- **[Lambda](lambda.md) functions** — the function is *invoked* per message

> [!note] A2A vs A2P
> AWS groups subscriber types into **A2A** (application-to-application: SQS, Lambda, HTTP/S, Kinesis Data Firehose) and **A2P** (application-to-person: SMS, mobile push, email). Source: [AWS SNS features](https://aws.amazon.com/sns/features).

### Filtering

A **subscription filter policy** lets a subscriber receive only the messages relevant to its function, instead of the default (all messages). The filter is set **per subscription** — it is the subscriber opting into a subset, not routing logic in the publisher. In the diagram, the filter sits on the delivery line heading toward Lambda.

### Fanout

**Fanout** is one SNS topic with **multiple SQS queues subscribed to it**. A single message (representing one workload) is delivered into every subscribed queue, letting multiple related-but-isolated workers each process it their own way.

- Course example: a video-processing job published once, fanned out to several queues that each transcode a **different bitrate or resolution** variant.
- This is the concrete decoupling primitive behind the [worker-fleet / event-driven progression](../patterns/architecture-evolution.md#stage-4--queue-decoupled-worker-fleet): SNS fans one event out to many queues, and each queue drives its own scale-to-zero worker fleet.

> [!tip] Exam Tip — fanout means SNS to SQS
> If a question describes delivering a **single message to multiple queues**, or decoupling **one event into several parallel workloads**, the answer is **SNS fanout to SQS** — not multiple direct publishes and not a load balancer.

## Reliability, Encryption, and Access

### Delivery status and retries

- **Delivery status** is confirmable for **HTTP/HTTPS, Lambda, and SQS** subscribers (per the course). This is a common "which endpoints can confirm delivery" distractor.
- **Delivery retries:** SNS retries failed deliveries according to a delivery policy.

> [!note] Retry and DLQ detail (production nuance, deeper than the exam)
> For **customer-managed HTTP/S endpoints** the default policy is **50 total attempts** before the message is discarded, and HTTP/S is the **only** protocol whose delivery policy you can customise. For **AWS-managed endpoints (SQS, Lambda)**, SNS retries far more aggressively — up to roughly **100,015 attempts over 23 days**. A message discarded after all retries is **lost by default** unless the subscription has a **dead-letter queue (DLQ)** attached — and an SNS DLQ is just a standard **SQS queue**. Sources: [AWS — Message delivery retries](https://docs.aws.amazon.com/sns/latest/dg/sns-message-delivery-retries.html), [AWS — SNS dead-letter queues](https://docs.aws.amazon.com/sns/latest/dg/sns-dead-letter-queues.html).

### Resilience, encryption, cross-account

- **Regionally resilient and highly available:** data sent to SNS is replicated within the region, so a topic keeps functioning if individual AZs fail. It scales from near-zero to highly transactional workloads within the region.
- **Server-Side Encryption (SSE):** data persisted to disk can be stored encrypted — relevant when on-disk encryption is mandated.
- **Cross-account access:** apply a **topic policy** (a resource policy, exactly like an [S3 bucket policy](s3.md#bucket-policies)) to control which identities can access the topic. See [Resource Policies vs Identity Policies](../concepts/resource-vs-identity-policies.md).

## Standard vs FIFO Topics

SNS has **two topic types**. The Cantrill course treats "topic" as one thing, but the distinction is testable on current SAA-C03.

| | **Standard topic** | **FIFO topic** |
| --- | --- | --- |
| Ordering | Best-effort (may arrive out of order) | Strict, first-in-first-out within a message group |
| Deduplication | Best-effort (at-least-once) | Strict (5-minute dedup window) |
| Throughput | Nearly unlimited | Up to **3,000 msg/s or 20 MB/s** per topic (whichever comes first), plus **300 msg/s per message group** |
| Subscriber types | All (SQS, Lambda, HTTP/S, email, SMS, push, Firehose) | **SQS queues only** — cannot deliver to email/HTTP/SMS/push directly |

> [!warning] Gotcha — FIFO topics deliver to SQS only
> If a scenario needs strict ordering **and** email/SMS/HTTP delivery, that is a mismatch — a FIFO topic can only deliver to SQS queues (and those must be SQS FIFO queues to preserve ordering end to end). SNS FIFO pairs with SQS FIFO.

> [!note] The 300-vs-3,000 throughput numbers
> Both figures are real and describe different scopes. A FIFO topic allows up to **3,000 messages per second (or 20 MB/s), whichever comes first, per topic** when `FifoThroughputScope` is set to `Topic`, and up to **300 messages per second per message group**. Older secondary sources cite only the 300 number as if it were the topic limit — it is the *per-message-group* limit. Trust the AWS Service Quotas page over blog aggregators. Source: [AWS — SNS endpoints and quotas](https://docs.aws.amazon.com/general/latest/gr/sns.html).

## Exam Traps

> [!tip] SNS exam triggers
>
> - **"Public service"** — SNS needs connectivity to public AWS endpoints plus IAM permissions; it is not placed inside a VPC subnet. Reachable from the internet or from a VPC with public-endpoint access.
> - **Fanout** is one topic to many **SQS** queues. A scenario duplicating one event across several parallel processors is SNS fanout to SQS.
> - **Filtering** happens at the **subscription** level (filter policy) — a subscriber opts into a subset. Don't confuse it with publisher-side routing.
> - **Cross-account** access to a topic is a **topic policy** (resource policy) — the S3-bucket-policy analogy.
> - **FIFO topic means SQS-only subscribers.** Strict ordering plus email/SMS/HTTP delivery is a mismatch.
> - **256 KB** message ceiling — SNS is for notifications, not bulk/binary data. Large payloads need the S3-backed Extended Client Library (up to 2 GB).
> - **Delivery status is supported for HTTP/S, Lambda, and SQS** — a common "which endpoints can confirm delivery" distractor.

## Quick Reference

| Concept | Key fact |
| --- | --- |
| Service type | Public, regional, fully managed pub/sub |
| Base entity | Topic (holds permissions and config) |
| Publisher | Sends messages into a topic (CloudWatch, CloudFormation, ASG, EC2, APIs) |
| Subscriber | Receives from topic (HTTP/S, email, SQS, Lambda, SMS, mobile push) |
| Default subscriber behaviour | Receives **all** topic messages |
| Filter | Per-subscription filter policy — receive only the relevant subset |
| Fanout | One topic to many SQS queues |
| Message size | 256 KB native; up to 2 GB via the S3-backed Extended Client Library |
| Delivery status | HTTP/S, Lambda, SQS |
| Retries / DLQ | Retries per delivery policy; discarded unless a DLQ (an SQS queue) is attached |
| Resilience | Regionally resilient (replicated in-region, AZ-fault tolerant) |
| Encryption | SSE for data at rest |
| Cross-account | Topic policy (resource policy) |
| Topic types | Standard (best-effort order/dedup) vs FIFO (strict, SQS-only subscribers) |

## References

- [Architecture Evolution — Coupling, Queues & Event-Driven](../patterns/architecture-evolution.md)
- [EventBridge](eventbridge.md), [Lambda](lambda.md), [S3](s3.md), [CloudWatch](cloudwatch.md), [CloudFormation](cloudformation.md), [EC2 Auto Scaling](ec2-auto-scaling.md)
- [AWS SNS features](https://aws.amazon.com/sns/features)
- [AWS — SNS endpoints and quotas](https://docs.aws.amazon.com/general/latest/gr/sns.html)
- [AWS — Publishing large messages with SNS and S3](https://docs.aws.amazon.com/sns/latest/dg/large-message-payloads.html)
- [AWS — SNS message delivery retries](https://docs.aws.amazon.com/sns/latest/dg/sns-message-delivery-retries.html)
- [AWS — SNS dead-letter queues](https://docs.aws.amazon.com/sns/latest/dg/sns-dead-letter-queues.html)
- Adrian Cantrill, AWS SAA course — Simple Notification Service (SNS)
- Source: `raw/aws-cert-study-notes/aws-solution-architect/sns.md`
