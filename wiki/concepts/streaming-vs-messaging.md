---
title: Streaming vs Messaging — Kinesis, SQS, and Kafka
type: concept
tags: [streaming, messaging, kinesis, sqs, kafka, msk, queues, log, decoupling, ingestion, replay, ordering, partitions, shards]
certs: [aws-saa]
updated: 2026-08-01
---

# Streaming vs Messaging — Kinesis, SQS, and Kafka

Two comparisons come up constantly, both in real design work and on the exam: **[Kinesis](../services/kinesis.md) vs [SQS](../services/sqs.md)** (which AWS primitive fits this workload) and **Kinesis vs Kafka** (which streaming technology, with **Amazon MSK** as managed Kafka on AWS). They share one root question, so they share a page.

## The Root Distinction — A Log Is Not a Queue

Almost every difference below falls out of one structural choice:

| | **Queue** (SQS) | **Log / stream** (Kinesis, Kafka) |
| --- | --- | --- |
| Data structure | A pool of messages waiting to be worked | An **append-only ordered log** of records |
| What consuming does | **Removes** the message (after an explicit delete) | **Reads** the record and leaves it in place |
| Who tracks progress | The **service** — via the visibility timeout and the delete call | The **consumer** — via its own position (sequence number or offset) |
| Natural consumer count | Many workers **competing** for the same messages, each message handled once | Many consumers **each reading everything**, independently |
| Re-reading data | Impossible — a deleted message is gone | Normal — **replay** is just rewinding a position |
| Lifetime | Until processed (bounded by retention) | Until the **retention window** expires, processed or not |

> [!tip] Exam Tip — the one-line test
> Ask **"does the data have one job, or many audiences?"** Work that one worker should do once is a **queue**. A record that a real-time dashboard, an archiver, an ML pipeline, and a fraud detector all want to see is a **log**. That question answers Kinesis-vs-SQS more reliably than any keyword list.

## Kinesis vs SQS

The short version lives on both service pages ([Kinesis](../services/kinesis.md#kinesis-vs-sqs--the-classic-exam-decision), [SQS](../services/sqs.md#exam-traps)); this is the reasoning behind it.

| Dimension | **[SQS](../services/sqs.md)** | **[Kinesis Data Streams](../services/kinesis.md#kinesis-data-streams)** |
| --- | --- | --- |
| Model | Queue — receive, hide for the **visibility timeout**, delete | Log — read records at your own position within the **rolling window** |
| Producers | Typically one component or group (a web tier in an ASG) | Huge numbers (sensors, clickstreams, mobile apps) |
| Consumers | One worker pool competing for messages | Many independent consumers, each reading everything at its own cadence |
| Ordering | None on standard queues; strict **per message group** on FIFO | Strict **per shard**, always |
| Replay | No — processed means gone | Yes, anywhere inside the retention window |
| Retention | 1 minute to **14 days** (default 4 days), and the message disappears once processed | **24 hours** to **365 days**, regardless of who has read it |
| Capacity unit | None to manage — SQS scales invisibly | **Shards** (1 MB/s or 1,000 records/s in, 2 MB/s out), or on-demand mode |
| Consumer parallelism | Add workers freely; throughput follows | Bounded by shard count — one worker per shard per application |
| Poison-message handling | **Dead-letter queue** after `maxReceiveCount`, the rest of the queue keeps flowing | A failing record **blocks its shard**, because ordering must hold. Handle it with `BisectBatchOnFunctionError`, a max retry count, and an on-failure destination on the [Lambda](../services/lambda.md) event source mapping |
| Cost shape | Per **request** (64 KB chunks) — idle queues cost nothing | Per **shard-hour** plus payload, or per GB on-demand — an idle provisioned stream still bills |
| Typical trigger words | Decouple, worker pool, async, job, process once | Ingest, analytics, dashboard, clickstream, IoT, replay |

Three consequences worth internalising:

- **Fan-out is free in Kinesis and deliberate in SQS.** Five teams wanting the same events means five reads of one Kinesis stream, but in SQS it means [SNS fanout to five queues](../services/sqs.md#sns-fanout-to-multiple-sqs-queues) — one queue per consumer, because a message can only be consumed once.
- **Ordering costs parallelism in both, differently.** SQS FIFO trades throughput for order within a message group; Kinesis gives order per shard for free but caps how many workers can process that shard.
- **Idle cost is opposite.** A provisioned Kinesis stream bills for its shards whether or not data flows; an SQS queue with no traffic is free. For bursty, low-volume, work-shaped traffic that is a real argument for SQS even when the data is stream-shaped.

> [!warning] Gotcha — "it's a stream" is not the same as "the data must persist"
> Data Streams keeps records for the window and then discards them. If the requirement is durable landing of streaming data, the answer is Kinesis **plus [Firehose](../services/kinesis.md#kinesis-data-firehose-now-amazon-data-firehose)** into [S3](../services/s3.md), not a longer retention window.

## Kinesis vs Kafka (and Amazon MSK)

**Apache Kafka** is the open-source distributed log. On AWS you can self-manage it on [EC2](../services/ec2.md) or use **Amazon MSK** (Managed Streaming for Apache Kafka), where AWS runs the brokers and you keep the Kafka API. Conceptually Kinesis Data Streams and Kafka are the *same idea*; the differences are operational, ecosystem-related, and architectural.

### The concepts map almost one to one

| Kinesis Data Streams | Apache Kafka | Notes |
| --- | --- | --- |
| Stream | **Topic** | The named log |
| **Shard** | **Partition** | The unit of parallelism, ordering, and throughput |
| Partition key | Message key | Hashed to pick shard or partition |
| Sequence number | **Offset** | The consumer's position in the log |
| KCL application | **Consumer group** | A set of workers dividing shards or partitions between them |
| Enhanced fan-out | (not needed) | Kafka consumer groups already read independently at full speed |
| `PutRecord` / `GetRecords` / `SubscribeToShard` | Producer / consumer API | Kinesis is an AWS API; Kafka is a wire protocol with a huge client ecosystem |
| Split and merge shards | Add partitions | Kafka can only **increase** partition count, and doing so **changes key-to-partition mapping**, breaking per-key ordering for existing keys. Kinesis splits and merges without that trap |

### Where they genuinely differ

| Dimension | **Kinesis Data Streams** | **Kafka / Amazon MSK** |
| --- | --- | --- |
| Operational model | No servers, no version, no brokers — a pure API | A **cluster**: you choose the Kafka version, broker type and count, and storage (MSK Serverless removes the capacity planning) |
| Network position | **Public regional endpoint**, reached by IAM-authenticated API calls (or a VPC interface endpoint) | Brokers run **in your VPC** via ENIs in your account, **private by default**, with optional public access and multi-VPC [PrivateLink](../services/vpc.md) |
| Retention | 24 hours to **365 days** | Configurable, including **effectively unlimited** with **tiered storage** that offloads older log segments to S3; also supports **log compaction** (keep the latest value per key forever), which Kinesis has no equivalent for |
| Record size | 1 MB default, up to 10 MiB opt-in | Broker-configurable (Kafka's `message.max.bytes` defaults to ~1 MB); **MSK Serverless caps at 8 MiB** |
| Delivery semantics | At-least-once; de-duplication is the consumer's job | At-least-once, plus **idempotent producers and transactions** for exactly-once *within Kafka* |
| Ecosystem | Deep AWS integration: [Firehose](../services/kinesis.md#kinesis-data-firehose-now-amazon-data-firehose), [Lambda](../services/lambda.md) event source mappings, Managed Service for Apache Flink, [CloudWatch](../services/cloudwatch.md) | The Kafka ecosystem: **Kafka Connect** (MSK Connect), Kafka Streams, ksqlDB, Schema Registry, and every existing Kafka client and tool |
| Portability | AWS-only API | Runs anywhere — the standard argument against lock-in, and what makes a lift-and-shift possible |
| Cost shape | Shard-hours plus payload units, or per GB on-demand | Broker-hours plus storage (provisioned), or throughput plus storage plus partition-hours (Serverless) |
| Expertise needed | Minimal | Real Kafka knowledge for tuning, partitioning, and consumer-group behaviour — even on MSK |

### MSK deployment options

| Option | What it is | Notable characteristics |
| --- | --- | --- |
| **Provisioned, Standard brokers** | Classic MSK — you size instances and EBS storage | Most configuration control; optional **tiered storage**, storage auto-scaling, T3/M5/M7g instances |
| **Provisioned, Express brokers** | Managed-storage brokers (M7g only) | Up to **3x throughput per broker**, **20x faster scaling**, ~90% faster recovery, storage you never provision, effectively unlimited ([AWS — Express brokers GA](https://aws.amazon.com/about-aws/whats-new/2024/11/express-brokers-amazon-msk-generally-available)) |
| **Serverless** | No cluster sizing at all; pay for throughput and storage | Per cluster: **200 MBps ingress / 400 MBps egress**, 5 MBps ingress and 10 MBps egress **per partition**, 2,400 partitions (120 for compacted topics), **8 MiB max message**, unlimited retention, 3,000 connections, 500 consumer groups ([MSK quotas](https://docs.aws.amazon.com/msk/latest/developerguide/limits.html)) |

**Authentication** on MSK is IAM (recommended), SASL/SCRAM, or mTLS — IAM access control is the one that makes MSK feel AWS-native. Metadata management has moved from **ZooKeeper to KRaft**: KRaft mode is available for new clusters since May 2024, raises the brokers-per-cluster quota from 30 to 60, and is automatic for Express-broker clusters on Kafka 3.9 ([AWS — KRaft for Express brokers](https://aws.amazon.com/about-aws/whats-new/2025/12/aws-msk-express-brokers-support-kraft)).

### Choosing between them

Choose **MSK** when:

- You **already run Kafka** — on-premises or on EC2 — and want a migration with minimal application change. This is the single most common exam framing.
- You depend on the **Kafka ecosystem**: Kafka Connect connectors, Kafka Streams, ksqlDB, schema registries, or existing Kafka client tooling.
- You need something Kinesis does not offer: **log compaction**, **unlimited retention**, **transactional exactly-once within the broker**, or Kafka-specific tuning.
- **Portability** across clouds or back on-premises is an explicit requirement.
- Brokers must sit **inside your VPC** as a hard network requirement.

Choose **Kinesis Data Streams** when:

- The work is **net-new on AWS** and nobody on the team is a Kafka operator.
- Producers and consumers are **AWS services**: Lambda, Firehose into S3 or Redshift, Managed Flink, IoT.
- You want **no cluster at all** — no version upgrades, no broker sizing, no partition-count planning beyond shards.
- Traffic is **spiky or unpredictable**, which suits on-demand capacity mode.

> [!tip] Exam Tip — the Kafka trigger word
> On SAA-C03, the appearance of **"Apache Kafka"**, "existing Kafka applications", "Kafka APIs", or "our team already runs Kafka" is the signal for **Amazon MSK**. AWS positions MSK as the *migrate-your-Kafka* answer and Kinesis as the *AWS-native streaming* answer. A question that never mentions Kafka and stresses "fully managed, minimal operational overhead" is pointing at **Kinesis**.

> [!warning] Gotcha — MSK is managed, not serverless (unless it is)
> "Fully managed" does not mean "no capacity decisions". Provisioned MSK still has you choosing broker count, instance size, and often storage, and partition-count planning remains your problem. Only **MSK Serverless** removes that, and it comes with the hard per-cluster ceilings listed above.

## Deciding in Practice

1. **Is the data work to be done once, or a record many things care about?** Work means [SQS](../services/sqs.md). A record means a stream.
2. **If it's a stream — is there existing Kafka investment (apps, tooling, skills, on-premises clusters)?** Yes means **MSK**. No means **[Kinesis](../services/kinesis.md)**.
3. **If it's Kinesis — does the data need to land somewhere durable?** Add **[Firehose](../services/kinesis.md#kinesis-data-firehose-now-amazon-data-firehose)** to S3, Redshift, or OpenSearch.
4. **If it's a queue — does order or de-duplication matter?** Yes means an SQS **FIFO** queue; otherwise standard with idempotent consumers.

Also remember that these are not exclusive. A common shape is Kafka or MSK as the enterprise backbone, with Kinesis and Firehose used for AWS-native ingestion and delivery paths, or [SNS](../services/sns.md) fanning one event into several SQS queues alongside a stream that keeps the durable ordered record.

## Quick Reference

| Question | Answer |
| --- | --- |
| Many independent consumers of the same data | **Kinesis** (or Kafka), not SQS |
| Replay or re-read after a bug fix | **Kinesis** or **Kafka** |
| One worker pool doing jobs once | **SQS** |
| Strict ordering with parallel throughput | Kinesis **shards** or Kafka **partitions** (order is per shard or per partition) |
| Strict ordering in a queue | SQS **FIFO** with `MessageGroupId` |
| Poison record stalls processing | Kinesis shard head-of-line blocking — bisect batches and set an on-failure destination |
| Poison message quarantined, queue keeps flowing | SQS **dead-letter queue** |
| Idle workload, cost must go to zero | **SQS**, or Kinesis **on-demand** — not provisioned shards |
| Existing Kafka applications must keep working | **Amazon MSK** |
| Kafka Connect, Kafka Streams, ksqlDB, schema registry | **Amazon MSK** |
| Retention beyond 365 days, or log compaction | **Kafka / MSK** (tiered storage) |
| No cluster, no Kafka expertise, AWS-native producers and consumers | **Kinesis Data Streams** |
| Brokers must live inside the VPC | **MSK** (Kinesis is a public regional endpoint) |
| Kafka wanted, but no capacity planning | **MSK Serverless** (200 MBps in / 400 MBps out per cluster) |

## References

- [Kinesis](../services/kinesis.md) · [SQS](../services/sqs.md) · [SNS](../services/sns.md) · [Lambda](../services/lambda.md) · [S3](../services/s3.md)
- [Architecture Evolution — Coupling, Queues & Event-Driven](../patterns/architecture-evolution.md) · [Global & Regional AWS Architecture](../patterns/global-regional-architecture.md)
- [Amazon MSK FAQs](https://aws.amazon.com/msk/faqs/) · [Amazon MSK quotas](https://docs.aws.amazon.com/msk/latest/developerguide/limits.html)
- [AWS — Express brokers for Amazon MSK generally available](https://aws.amazon.com/about-aws/whats-new/2024/11/express-brokers-amazon-msk-generally-available) · [KRaft support for Express brokers](https://aws.amazon.com/about-aws/whats-new/2025/12/aws-msk-express-brokers-support-kraft) · [KRaft mode for new clusters](https://aws.amazon.com/about-aws/whats-new/2024/05/amazon-msk-kraft-mode-apache-kafka-clusters)
- [Amazon Kinesis Data Streams quotas and limits](https://docs.aws.amazon.com/streams/latest/dev/service-sizes-and-limits.html) · [Amazon SQS message quotas](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/quotas-messages.html)
