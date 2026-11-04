---
title: Kinesis
type: service
tags: [kinesis, data-streams, firehose, data-analytics, managed-flink, video-streams, streaming, ingestion, analytics, shards, enhanced-fan-out, serverless, real-time]
certs: [aws-saa]
updated: 2026-11-04
---

# Kinesis

## Overview

**Kinesis** is a family of AWS products built for **large-scale streaming data** — ingestion, delivery, real-time processing, and (separately) live media. Every member is a **public service** (reached over public AWS endpoints with IAM permissions, not placed inside a VPC subnet, exactly like [SQS](sqs.md) and [SNS](sns.md)), **regional**, and **highly available within the region by design**. You never manage replication, network placement, or scaling infrastructure. See [Resource Scope](../concepts/resource-scope.md).

Four products, one line each:

| Product | Job | Real-time? |
| --- | --- | --- |
| **Kinesis Data Streams** | Ingest huge volumes of data; let many consumers read it independently, with a persistence window | Yes, real-time (~200 ms) |
| **Kinesis Data Firehose** (now *Amazon Data Firehose*) | Managed **delivery** of streaming data to destinations (S3, Redshift, and more); optional Lambda transform | **Near** real-time (~60 s in course/exam terms) |
| **Kinesis Data Analytics** (now *Amazon Managed Service for Apache Flink*) | Real-time **stream processing**, taught as SQL | Yes, unless the output is Firehose |
| **Kinesis Video Streams** | Ingest **live video and time-serialized media** from producers; integrate with analytics services | Yes, live |

> [!warning] Two of these products were renamed and one was discontinued in the form the course teaches
> The course names — Kinesis Data Firehose and Kinesis Data Analytics — are still the terms the exam is most likely to use, so learn them. AWS reality has moved on: Firehose is now **Amazon Data Firehose**, the Flink variant of Data Analytics is **Amazon Managed Service for Apache Flink**, and the **SQL** variant of Data Analytics was **discontinued on 27 January 2026**. Changes are flagged inline and collected in [Course vs current AWS reality](#course-vs-current-aws-reality).

Related pages: [SQS](sqs.md) · [SNS](sns.md) · [Lambda](lambda.md) · [S3](s3.md) · [CloudWatch](cloudwatch.md) · [EventBridge](eventbridge.md) · [Streaming vs Messaging](../concepts/streaming-vs-messaging.md) · [Architecture Evolution](../patterns/architecture-evolution.md) · [Global & Regional Architecture](../patterns/global-regional-architecture.md).

For how Data Streams compares with **Apache Kafka** and **Amazon MSK**, see [Streaming vs Messaging](../concepts/streaming-vs-messaging.md#kinesis-vs-kafka-and-amazon-msk).

## Kinesis Data Streams

The **stream** is the basic entity of Kinesis. **Producers** put data in; **consumers** read data out. A stream scales from trivial throughput to effectively unlimited by adding **shards**.

### Stream architecture

![Kinesis architecture diagram: producers on the right (EC2 instances, on-premises servers, mobile devices, connected cars, medical devices, and retail sensors) feed many purple flow lines into a central Kinesis Stream drawn as three red horizontal shard bars labelled 1, 2 and N, each holding orange Kinesis Data Record blocks capped at 1 MB; consumers on the left (on-premises servers, EC2 instances, and a Lambda function) read back out along the same many-to-one flows; labels call out the 24-hour window extendable up to 365 days at extra cost, 1 MB per second ingestion and 2 MB per second consumption per shard, and a Firehose icon below the stream delivering into an S3 bucket](../assets/kinesis-architecture.png)

Reading the diagram:

- **Producers (right):** [EC2](ec2.md) instances, on-premises servers, mobile apps and devices, IoT sensors (cars, medical devices, retail and e-commerce). Anything designed to *put* data into the stream.
- **Consumers (left):** on-premises servers, EC2 instances, and **[Lambda](lambda.md)** functions, which can be invoked automatically as data lands on the stream (via an [event source mapping](lambda.md#event-source-mapping-esm) that polls the stream). Each consumer reads at its own cadence — every second, every minute, every hour — which is what makes Kinesis suit analytics and dashboards.
- **The stream (center):** made of **shards**, the red bars labelled `1`, `2`, `N`. A stream starts with one shard and grows as scale demands. The orange squares inside are **Kinesis Data Records**. Records spread across shards, so throughput scales **linearly** with shard count.
- **Top labels — the two things that cost money:** the rolling persistence window (24 hours by default, up to 365 days) and the record size.
- **Bottom labels — per-shard throughput:** 1 MB/s ingestion, 2 MB/s consumption.
- **Bottom-center:** the Firehose icon feeding an [S3](s3.md) bucket, previewing [Firehose](#kinesis-data-firehose-now-amazon-data-firehose) as the way to move stream data en masse into another service.

[AWS Glue streaming ETL](glue.md#jobs-sources-and-targets) is another managed consumer: it can read Kinesis Data Streams or Kafka/MSK, transform records, and write prepared datasets. It runs continuously and uses checkpoints. Glue crawlers do not discover stream schemas; configure the streaming source separately. This separates **ingestion** from **transformation**, as described in [Data Integration](../concepts/data-integration.md#etl-elt-and-streaming).

### The persistence window

Kinesis provides a level of persistence that [SQS](sqs.md) does not. Ingested data is retained in a **rolling window**, **24 hours by default**, during which any consumer can read it. Anything older than the window is discarded.

- The window can be extended to a **maximum of 365 days** for additional cost — the `IncreaseStreamRetentionPeriod` API caps at 8,760 hours ([Kinesis Data Streams quotas and limits](https://docs.aws.amazon.com/streams/latest/dev/service-sizes-and-limits.html)).
- The window is **persistent storage**: a 365-day window means 365 days of data physically stored. More window means more cost.

This is the property that makes **replay** possible, and replay is the single most reliable Kinesis-vs-SQS exam signal.

### Shards and throughput

Each shard provides a fixed slice of capacity ([quotas and limits](https://docs.aws.amazon.com/streams/latest/dev/service-sizes-and-limits.html)):

- **Ingestion:** 1 MB/s **or** 1,000 records/s per shard.
- **Consumption:** 2 MB/s **or** 2,000 records/s per shard, shared across all classic consumers.

More shards means more performance **and** more cost. Records carry a **partition key** that is hashed to decide which shard they land on, so a badly chosen partition key creates a hot shard that throttles while the rest of the stream idles.

> [!note] Record size: 1 MB on the exam, up to 10 MiB in current AWS
> The course and exam figure is a **1 MB maximum record size** — learn that one. Since **28 October 2025** a stream's maximum record size can be raised to **10 MiB** with the `UpdateMaxRecordSize` API or the console, with the `PutRecords` request ceiling raised from 5 MiB to 10 MiB and [Lambda](lambda.md) event source mappings supporting payloads up to 6 MiB ([AWS — Kinesis Data Streams now supports 10x larger record sizes](https://aws.amazon.com/about-aws/whats-new/2025/10/amazon-kinesis-data-streams-10x-larger-record-sizes)). The default is still 1 MiB, and large records still consume the shard's 1 MB/s budget — they are for *intermittent* large payloads, served from burst capacity.

> [!note] Enhanced fan-out — a very common exam distractor
> The 2 MB/s consumption figure is the **shared (classic) fan-out** limit: *all* classic consumers on a shard **share** that 2 MB/s, so three classic consumers on one shard get roughly 667 KB/s each. Classic consumers also poll with `GetRecords`, which is capped at 5 transactions per second per shard.
>
> **Enhanced fan-out** gives **each registered consumer its own dedicated 2 MB/s per shard**, pushed via `SubscribeToShard` over HTTP/2 rather than polled, which also cuts latency ([Develop enhanced fan-out consumers](https://docs.aws.amazon.com/streams/latest/dev/enhanced-consumers.html)). You can register **20 enhanced fan-out consumers per stream** on provisioned and On-demand Standard streams, and **50** on On-demand Advantage streams ([quotas and limits](https://docs.aws.amazon.com/streams/latest/dev/service-sizes-and-limits.html)).
>
> **Exam cue:** "multiple consumers each need full-speed independent reads of the same stream" means **enhanced fan-out**.

### Capacity modes

The course describes Data Streams purely in **provisioned** terms, where you choose and manage the shard count. Current AWS offers three modes ([Choose the right mode to stream in](https://docs.aws.amazon.com/streams/latest/dev/how-do-i-size-a-stream.html)):

| Mode | How capacity works | Notes |
| --- | --- | --- |
| **Provisioned** | You set the shard count and split or merge shards yourself | Predictable traffic; fine-grained control over distribution |
| **On-demand Standard** | AWS manages shards and scales automatically | New streams start at **4 MB/s write and 8 MB/s read**; a stream absorbs up to **double its peak write throughput of the previous 30 days**, and splits a shard within 15 minutes once it sustains more than 500 KB/s |
| **On-demand Advantage** | Account-level setting on top of on-demand | Adds configurable **warm throughput**, **50** enhanced fan-out consumers, and simpler pricing at least 60% below On-demand Standard; commits the account to 25 MiB/s ingest and 25 MiB/s retrieval per Region |

On-demand streams scale to **10 GB/s write and 20 GB/s read** in us-east-1, us-west-2, and eu-west-1, and to **200 MB/s write and 400 MB/s read** in other Regions (raisable by support request). A stream can switch between on-demand and provisioned **twice in 24 hours**.

> [!warning] Gotcha — the old on-demand ceiling figure
> Widely repeated study material still says on-demand "scales up to 200 MB/s and 200,000 records/s." That was the 2021 launch figure and now applies only to Regions outside the big three, and AWS now expresses on-demand capacity in **throughput** rather than records per second. The per-shard 1 MB/s and 1,000 records/s numbers are the ones that have not moved.

### Kinesis vs SQS — the classic exam decision

There is almost always a question asking you to choose between these two. The reasoning behind the answer — and the Kafka comparison — is on [Streaming vs Messaging](../concepts/streaming-vs-messaging.md). Read every such question through this lens:

| The question is about… | Answer |
| --- | --- |
| **Ingestion** of data at huge scale, from many producers | **Kinesis** |
| Worker pools, **decoupling**, **asynchronous** communication | **[SQS](sqs.md)** (assume SQS first; switch only with a strong reason) |

| | **[SQS](sqs.md)** | **Kinesis Data Streams** |
| --- | --- | --- |
| **Producers** | Typically one thing or one group (a web tier in an ASG) | Huge numbers (thousands of sensors, clickstreams) |
| **Consumers** | Usually one consumer or worker group | Many independent consumers reading at different rates |
| **Persistence** | No window — a message is deleted after processing and is gone | Rolling window, 24 hours up to 365 days |
| **Purpose** | Decoupling application components, async messaging | Ingestion, analytics, monitoring, clickstreams |

> [!tip] Exam Tip — trigger keywords
> Ingestion, analytics, dashboards, **mobile or click streams**, IoT, "many producers", "multiple consumers reading independently", "replay" or re-read the data: **Kinesis**. Decouple, worker tier, queue, async, "process once then delete": **[SQS](sqs.md)**.

## Kinesis Data Firehose (now Amazon Data Firehose)

> [!warning] Renamed to Amazon Data Firehose (February 2024)
> The product is now **Amazon Data Firehose**. Endpoints, APIs, CLI, IAM policies, and CloudWatch metrics were unchanged. The course teaches it as "Kinesis Data Firehose" and the exam may still use that name, so recognize both. See [Course vs current AWS reality](#course-vs-current-aws-reality).

Firehose is a **fully managed, serverless, resilient delivery service**. It accepts streaming data, optionally transforms it in flight with [Lambda](lambda.md), and delivers it to supported destinations in near real time. Its core value is giving Data Streams something it lacks natively: **persistence and delivery** into other services, since records that age out of the rolling window are otherwise gone forever.

### Firehose architecture

![Kinesis Data Firehose diagram: sources on the left (CloudWatch Logs and Events, the Kinesis Producer Library, a Kinesis Data Stream, the Kinesis Agent, and IoT) feed the central Firehose funnel, with a note that producers can send records to data streams or directly at Firehose and that Firehose can also read from a data stream as a consumer; above the funnel a Lambda transformation function using blueprints receives source records and returns transformed records; below it a backup bucket stores unmodified source records; text explains delivery happens when the buffer size in MB fills or the buffer interval in seconds passes; on the right a green destination panel shows HTTP endpoints, Splunk, Redshift reached through an S3 intermediate plus a Redshift COPY, Elasticsearch, and the destination S3 bucket](../assets/kinesis-data-firehose.png)

Reading the diagram:

- **Sources (left):** [CloudWatch](cloudwatch.md) Logs and Events, the **KPL** (Kinesis Producer Library), a **Kinesis Data Stream**, the **Kinesis Agent**, and **IoT**. The key sentence on the diagram: producers can send records to a data stream **or** send directly at Firehose, and Firehose can also read from a data stream as a consumer.
- The label "Kinesis streams are realtime ~200 ms" describes the **Data Streams** side. Firehose itself is not real-time.
- **Transform (top):** a **Lambda transformation function**, optionally built from **blueprints** — pre-built functions for common tasks such as converting Apache or system logs to JSON or CSV. Source records go up, transformed records come back down. Transformation adds latency.
- **Backup bucket (bottom):** if you transform, you can optionally keep the **unmodified source records** in a backup [S3](s3.md) bucket.
- **Destinations (right):** HTTP endpoints, **Splunk**, **Redshift**, **Elasticsearch** (now OpenSearch Service), and **S3**.
- **Redshift is special:** delivery to Redshift uses **S3 as an intermediate**. Firehose writes to S3 and then issues a **Redshift COPY**. Conceptually direct, mechanically S3 plus COPY, handled end to end by Firehose.

### Near real-time: the buffer

Firehose *receives* data continuously but *delivers* on a buffer, flushing on whichever comes first:

- **buffer size**, 1 MB in the course material, **or**
- **buffer interval**, 60 seconds in the course material.

Low-volume producers hit the timer; high-volume producers hit the size threshold.

> [!warning] Course vs current AWS — 60 seconds is no longer the minimum interval
> Since **26 December 2023**, Firehose supports **zero buffering**: the buffer interval can be set as low as **0 seconds**, against the previous minimum of 60, and streams with no additional processing typically deliver **within about five seconds** to S3, OpenSearch Service, Redshift, and HTTP endpoints ([AWS — Amazon Kinesis Data Firehose now supports zero buffering](https://aws.amazon.com/about-aws/whats-new/2023/12/amazon-kinesis-data-firehose-zero-buffering)). Buffer interval now ranges **0 to 900 seconds** for most destinations (Splunk caps at 60), and zero buffering is **not available with dynamic partitioning** ([buffering hints](https://docs.aws.amazon.com/firehose/latest/dev/buffering.html)).
>
> For the exam, keep answering **"Firehose is near real-time, roughly 60 seconds"** — that is the framing SAA-C03 tests. In practice the latency gap between Firehose and Data Streams is now seconds, not a minute.

> [!tip] Exam Tip — the single most important Firehose fact
> **Firehose is not real-time; it is near real-time.** AWS frames ~200 ms as real-time and ~60 s as near real-time. If a question demands genuinely real-time delivery, **Firehose cannot be the answer** — use Data Streams with, for example, a [Lambda](lambda.md) consumer.

### Destinations

The course stresses knowing the valid destinations, because that is how you rule Firehose in or out of an answer:

- **S3** · **Redshift** (via an S3 intermediate plus COPY) · **Elasticsearch** (now **OpenSearch Service**) · **Splunk** · **HTTP endpoints** (third-party providers)

Those five remain the reliable exam core. Current Firehose also delivers to **Snowflake**, **Apache Iceberg tables**, **Amazon S3 Tables**, and many more HTTP partners including Datadog, New Relic, Dynatrace, Sumo Logic, LogicMonitor, and MongoDB ([Amazon Data Firehose FAQs](https://aws.amazon.com/firehose/faqs/)).

### When to use Firehose

- Give **persistence** to data arriving on a Data Stream, so it survives past the rolling window.
- Store data in a **different format**, transformed via Lambda (Parquet, ORC, and similar).
- **Deliver** stream or direct-PUT data into a supported destination **without building the plumbing yourself**.

The trade-off: you give up the real-time nature of raw Data Streams in exchange for not having to build delivery.

## Kinesis Data Analytics (now Amazon Managed Service for Apache Flink)

> [!warning] The SQL variant taught here was discontinued on 27 January 2026
> This is the biggest course-versus-reality gap in the Kinesis set. The lesson teaches **SQL-based** stream processing, and that specific product no longer exists — the Apache Flink variant was renamed **Amazon Managed Service for Apache Flink**. The **concept** (real-time processing sitting between an input and an output stream, enriched by static reference data) is still exam-relevant, so learn the mechanics and treat the naming as dated. Detail in [Course vs current AWS reality](#course-vs-current-aws-reality).

Kinesis Data Analytics provides **real-time processing of streaming data using SQL**. Data comes in one side, SQL queries run against it in real time, results go out the other side.

- **Ingests from:** Kinesis Data Streams **or** Firehose, and can optionally pull **static reference data from [S3](s3.md)**.
- **Outputs to:** **Firehose** (and, indirectly, anything Firehose supports), **[Lambda](lambda.md)**, or **Kinesis Data Streams**.

> [!tip] Exam Tip — real-time vs near real-time depends on the destination
> Output to **Firehose**, or to any of Firehose's downstream destinations, makes the pipeline **near** real-time. Output to **Lambda** or **Data Streams** keeps it **real-time**. This is a classic distractor.

### Analytics application architecture

![Kinesis Data Analytics diagram in three panels: on the left a blue Source Stream panel holding a Kinesis Stream and a Kinesis Firehose plus a teal Reference Data panel with an S3 bucket feeding CSV or JSON; in the center an orange Kinesis Analytics Application containing a blue-topped in-application input stream table, a teal-topped reference table of static data used to enrich the streaming input, a black SQL box labelled application code processes input and produces output, two green-topped in-application output stream tables, and a red-topped in-application error stream for errors generated during processing; on the right a green Destination Stream panel with a Kinesis Stream feeding consumers and a Kinesis Firehose feeding destinations](../assets/kinesis-data-analytics.png)

The key idea: sources and destinations live **outside** the application. Inside, the application works on **virtual tables** that mirror those external streams — Analytics never modifies the sources themselves.

- **Source stream (blue, left):** the external inputs, a Kinesis Stream or Firehose.
- **Reference data (teal, bottom-left):** an S3 bucket supplying static CSV or JSON.
- **Inside the application (orange, center):**
  - **In-application input stream** — a constantly updated virtual table mirroring the source stream. Think "database table, but a live feed."
  - **Reference table** — mirrors the S3 static data, used to **enrich** the streaming input.
  - **SQL** — the application code. It reads the input stream and the reference table and writes results out.
  - **In-application output streams** — the results, mapped onto the real external destination streams.
  - **In-application error stream** — errors generated during processing.
- **Destination stream (green, right):** the external outputs, a Kinesis Stream feeding consumers or Firehose feeding destinations.

**The enrichment example:** a popular online game streams raw player scores and activity onto a Kinesis stream, and the reference table (from S3) holds static player metadata — names, owned items, awards — used to enrich the raw stream in real time.

### When to use it

Anything needing **real-time, stream-based processing**:

- **Time-series analytics** (election results, e-sports)
- **Real-time dashboards and leaderboards**
- **Real-time metrics** for security and response teams

> [!tip] Exam Tip — Data Analytics vs Firehose plus Lambda
> Firehose can also transform with Lambda, but Firehose is near real-time and Lambda transforms are relatively simple. Data Analytics supports **complex** processing and stays **real-time** with the right destination. **Complex real-time manipulation means Data Analytics.** It is also not cheap — you pay for data processed — so it is for cases that genuinely need it.

## Kinesis Video Streams

Kinesis Video Streams **ingests live streaming media from producers**: security cameras, smartphones, cars, drones — and **non-video, time-serialized data** such as audio, thermal, depth, or radar. Once media is in AWS, consumers access it **frame by frame or as needed** for analysis. Data is encrypted in transit and at rest and persisted, as a managed service.

**Common integrations:** **Rekognition** (deep-learning analytics on live video, such as facial recognition) and **Amazon Connect** (voicemail and audio streaming).

> [!warning] Gotcha — you cannot reach the ingested data directly on storage
> Video Streams does **not** store data in its original format on [EBS](ebs.md), [S3](s3.md), or [EFS](efs.md) where you can reach it directly. It is indexed and stored in a structured way **inside the product**, and you must go via the product's APIs. Any exam answer claiming you can grab the raw video off S3, EBS, or EFS is wrong.

### Architecture — smart-home face detection

![Kinesis Video Streams diagram: on the left a house with a cat and a dog and three cameras, each camera's video feed going into its own Kinesis Video Stream, one per camera; the three video streams feed Rekognition Video at the top right, which compares against a Face Collection of known people labelled Bob, Julie, Friend and Cat Sitters; Rekognition's analysis flows down into a Kinesis Data Stream, then right to left through a Lambda function analysing every record and on to Simple Notification Service, which sends an alert reading HALP, unknown human](../assets/kinesis-video-streams.png)

Video flows left to right across the top (cameras, Video Streams, Rekognition), then analysis flows right to left across the bottom (Data Stream, Lambda, SNS, alert):

1. Three **cameras** each send a **video feed** into **its own Kinesis Video Stream** — one stream per camera. No in-house processing hardware is needed, and video is stored off-property.
2. The three Video Streams feed **Rekognition Video**.
3. A **Face Collection** holds the known faces Rekognition compares against — the homeowner, a friend, and the cat sitters.
4. Rekognition's analysis goes to a **Kinesis Data Stream**, listing **detected** faces and flagging which ones **match** a face in the collection.
5. A **[Lambda](lambda.md)** function analyses every record and applies the decision logic.
6. If an unknown face is detected, **[SNS](sns.md)** sends the alert.

This is a simple **event-driven video analytics** pattern (see [Architecture Evolution](../patterns/architecture-evolution.md)). The product can do far more, but that is the depth expected at SAA level.

> [!tip] Exam Tip — trigger keywords
> Live video streaming, analytics on a video stream, **GStreamer**, or **RTSP** means **Kinesis Video Streams**. It is not expected to feature in deep detail on the exam.

## Course vs current AWS reality

The course lessons predate several AWS changes. Both framings are kept because exam questions may reflect either.

> [!warning] Kinesis Data Firehose is now Amazon Data Firehose (9 February 2024)
> AWS renamed the service in the console, documentation, and web pages, with **no changes to endpoints, APIs, CLI, IAM policies, or CloudWatch metrics** ([AWS — Amazon Kinesis Data Firehose is now Amazon Data Firehose](https://aws.amazon.com/about-aws/whats-new/2024/02/amazon-kinesis-data-firehose-amazon-data-firehose/)). The exam is likely to still say "Kinesis Data Firehose," so recognize both.

> [!warning] Kinesis Data Analytics was renamed, and the SQL variant was discontinued
>
> - The **Apache Flink** variant was renamed **Amazon Managed Service for Apache Flink** on 30 August 2023, again with no changes to endpoints, APIs, CLI, IAM policies, or CloudWatch metrics.
> - The **SQL** variant — the exact product the lesson teaches — was **discontinued**. No new SQL applications could be created from **15 October 2025**, and existing applications began being deleted on **27 January 2026**, after which they cannot be started or operated and support is no longer available ([AWS — Kinesis Data Analytics for SQL Applications discontinuation](https://docs.aws.amazon.com/kinesisanalytics/latest/dev/discontinuation.html)). AWS had labelled it a legacy offering since 2021.
> - **What this means:** study the *concept*, because the exam question pattern still tests it. The "SQL-based Kinesis Data Analytics" product framing is historical; modern real-time stream processing on AWS is **Amazon Managed Service for Apache Flink**.

> [!warning] "Elasticsearch" as a destination is now Amazon OpenSearch Service
> AWS renamed Amazon Elasticsearch Service to **Amazon OpenSearch Service** in 2021. The course diagram and destination list say "Elasticsearch"; current Firehose documentation says OpenSearch Service. Expect either name in a question.

> [!note] Firehose destinations have grown well beyond the course five
> Current supported destinations include the course set **plus** Snowflake, Apache Iceberg tables, Amazon S3 Tables, and many more HTTP partners ([Amazon Data Firehose FAQs](https://aws.amazon.com/firehose/faqs/)). For the exam, the course five remain the reliable core.

> [!note] Capacity modes, record size, and buffering have all moved
> Three changes documented in the sections above, collected here: on-demand capacity mode (including **On-demand Advantage** with 50 enhanced fan-out consumers) sits alongside the provisioned/shard model the course teaches; the maximum record size can be raised from 1 MiB to **10 MiB** since October 2025; and Firehose's minimum buffer interval is now **0 seconds** rather than 60.

## Exam Traps

> [!tip] Exam Tip — trigger keyword table
>
> | You see… | Think… | Why |
> | --- | --- | --- |
> | "Ingest data from thousands of devices, sensors, or clickstreams" | **Kinesis Data Streams** | Ingestion at scale is Kinesis, not SQS |
> | "Decouple components", "worker tier", "async", "process once" | **[SQS](sqs.md)** | No persistence window; single consumer group |
> | "Multiple consumers must **replay** or re-read the same data" | **Kinesis Data Streams** | Rolling window plus many independent consumers |
> | "**Real-time** delivery to S3" | **Not Firehose** — Data Streams plus Lambda | Firehose is near real-time |
> | "**Near** real-time delivery to S3, Redshift, or Splunk, with no infrastructure to build" | **Firehose** | Managed delivery on a buffer |
> | "Load streaming data into **Redshift**" | **Firehose** | Redshift delivery is indirect, via S3 plus COPY |
> | "**Complex** real-time transformation with SQL" | **Data Analytics** (now Managed Flink) | Firehose plus Lambda is simple and near real-time only |
> | "Enrich a stream with **static reference data**" | **Data Analytics** with an S3 reference table | The core Analytics pattern |
> | "Persist stream data past the window, archive to a data lake" | **Firehose** | Data Streams alone does not persist beyond the window |
> | "Live **video**", "**RTSP**", "**GStreamer**", drone or camera feeds with analytics | **Kinesis Video Streams** plus Rekognition | Media ingestion; raw media is not readable off S3, EBS, or EFS |
> | "Many consumers each need the full 2 MB/s with no contention" | **Enhanced fan-out** | Classic consumers *share* 2 MB/s per shard |
> | "One shard throttles while the others idle" | **Partition key** design | The key is hashed to pick the shard |

## Quick Reference

| Attribute | Data Streams | Firehose (*Amazon Data Firehose*) | Data Analytics (*Managed Flink*) | Video Streams |
| --- | --- | --- | --- | --- |
| Purpose | Ingest, persist, multi-consumer read | Deliver to destinations | Real-time stream processing | Live media ingestion |
| Latency | Real-time (~200 ms) | **Near** real-time (~60 s on the exam; 0 s buffer possible today) | Real-time, near real-time if the output is Firehose | Live |
| Persistence | 24 h default, up to 365 days | Buffers only | Transient (virtual tables) | Persisted inside the product |
| Scaling unit | Shards (1 MB/s in, 2 MB/s out each), or on-demand | Fully serverless, automatic | Serverless, automatic | Managed |
| Record size | 1 MB default, up to 10 MiB opt-in | — | — | — |
| Sources | Producers via SDK, KPL, or Agent | Direct PUT, Data Streams, CloudWatch, IoT, KPL, Agent | Data Streams or Firehose, plus S3 reference data | Cameras, phones, cars, drones; audio, thermal, depth, radar |
| Destinations | Consumers (EC2, Lambda, on-premises) | S3, Redshift, OpenSearch, Splunk, HTTP, and more | Firehose, Lambda, Data Streams | Consumers via the product API; Rekognition, Connect |
| Transform | No, the consumer does it | Lambda (simple) | SQL or Flink (complex) | Not applicable |
| Real-time? | Yes | No, near | Yes, destination-dependent | Yes |

## References

- [SQS](sqs.md) · [SNS](sns.md) · [Lambda](lambda.md) · [S3](s3.md) · [CloudWatch](cloudwatch.md) · [EventBridge](eventbridge.md) · [EC2](ec2.md) · [DMS](dms.md)
- [Streaming vs Messaging — Kinesis, SQS, and Kafka](../concepts/streaming-vs-messaging.md) · [Architecture Evolution — Coupling, Queues & Event-Driven](../patterns/architecture-evolution.md) · [Global & Regional AWS Architecture](../patterns/global-regional-architecture.md) · [Resource Scope](../concepts/resource-scope.md)
- [Amazon Kinesis Data Streams — quotas and limits](https://docs.aws.amazon.com/streams/latest/dev/service-sizes-and-limits.html) · [choose the right mode to stream in](https://docs.aws.amazon.com/streams/latest/dev/how-do-i-size-a-stream.html) · [enhanced fan-out consumers](https://docs.aws.amazon.com/streams/latest/dev/enhanced-consumers.html)
- [AWS — Amazon Kinesis Data Streams now supports 10x larger record sizes](https://aws.amazon.com/about-aws/whats-new/2025/10/amazon-kinesis-data-streams-10x-larger-record-sizes) · [up to 50 enhanced fan-out consumers](https://aws.amazon.com/about-aws/whats-new/2025/11/amazon-kinesis-data-streams-enhanced-fan-out-consumers) · [On-demand Advantage mode](https://aws.amazon.com/about-aws/whats-new/2025/11/amazon-kinesis-data-streams-ondemand-advantage)
- [Amazon Data Firehose FAQs](https://aws.amazon.com/firehose/faqs/) · [buffering hints](https://docs.aws.amazon.com/firehose/latest/dev/buffering.html) · [zero buffering announcement](https://aws.amazon.com/about-aws/whats-new/2023/12/amazon-kinesis-data-firehose-zero-buffering) · [rename to Amazon Data Firehose](https://aws.amazon.com/about-aws/whats-new/2024/02/amazon-kinesis-data-firehose-amazon-data-firehose/)
- [Amazon Kinesis Data Analytics for SQL Applications discontinuation](https://docs.aws.amazon.com/kinesisanalytics/latest/dev/discontinuation.html) · [Amazon Managed Service for Apache Flink](https://docs.aws.amazon.com/managed-flink/latest/java/what-is.html)
- Adrian Cantrill, AWS SAA course — Kinesis Data Streams, Kinesis Data Firehose, Kinesis Data Analytics, Kinesis Video Streams. Diagrams © Adrian Cantrill.
- Source: `raw/aws-cert-study-notes/aws-solution-architect/kinesis.md`
