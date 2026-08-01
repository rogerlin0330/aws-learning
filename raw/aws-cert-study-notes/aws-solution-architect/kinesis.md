---

tags:

- aws
- saa
- kinesis
- streaming
- analytics
- serverless

---

# Kinesis

## Overview

**Kinesis** is a family of AWS products built to handle **large-scale streaming data** — ingestion, delivery, real-time processing, and (separately) media streaming. It is a **public service**, **highly available within a region by design**, and fully managed: you don't handle replication, network placement, or scaling infrastructure yourself.

The family covered here, and the one-line job of each:

|Product|Job|Real-time?|
|---|---|---|
|**Kinesis Data Streams**|Ingest huge volumes of data; let many consumers read it independently, with a persistence window|✅ Real-time (~200 ms)|
|**Kinesis Data Firehose**|Managed **delivery** of streaming data to destinations (S3, Redshift, etc.); optional Lambda transform|⚠️ **Near** real-time (~60 s)|
|**Kinesis Data Analytics**|Real-time **SQL processing** of streaming data|✅ Real-time (unless output is Firehose)|
|**Kinesis Video Streams**|Ingest **live video/time-serialized media** from producers; integrate with analytics services|✅ Live streaming|

> [!warning] Two of these products have been renamed, and one has been discontinued in the form the course teaches The course names (Kinesis Data Firehose, Kinesis Data Analytics) are still the terms the exam is most likely to use, so learn them — but AWS reality has moved on. See [[#Course-vs-current-AWS reality]] before you rely on any of the naming or destination lists below. The changes are significant enough that they're flagged inline throughout, then collected in that section.

---

## Kinesis Data Streams

The **stream** is the basic entity of Kinesis. **Producers** put data in; **consumers** read data out. A stream scales from trivial throughput to near-infinite by adding **shards**.

### Architecture walkthrough

![[kinesis-architecture.png]]

**How to read this diagram:** producers on the **right** feed into the **Kinesis Stream** in the middle (the horizontal red bars), and consumers on the **left** read from it. The purple curved lines are the many-to-one / one-to-many flows — lots of things in, lots of things out.

- **Producers (right column):** EC2 instances, on-premises servers, mobile apps/devices, IoT sensors (cars, medical devices, retail/e-commerce). Anything designed to _put_ data into the stream.
- **Consumers (left column):** on-prem servers, EC2 instances, and **Lambda** functions (which can be invoked automatically when data lands on the stream). Each consumer can read at its own cadence — every second, every minute, every hour — which is what makes Kinesis great for analytics and dashboards.
- **The stream (center):** made of **shards**, shown as red bars labelled `1`, `2`, `N`. A stream starts with one shard and grows as scale is needed. The orange squares inside are **Kinesis Data Records** (max **1 MB** each). Records are spread across shards, so throughput scales **linearly** with shard count.
- **Top labels — the two things that cost money:**
    - **24-hour window, up to 365 days 💵** — the rolling persistence window (below).
    - **Kinesis Data Record (1 MB)** — the record size cap.
- **Bottom labels — per-shard throughput:** **1 MB/s ingestion**, **2 MB/s consumption**.
- **Bottom-center:** the Firehose icon feeding an S3 bucket — a preview of using [[#Kinesis Data Firehose]] to move stream data en masse into another service.

### The persistence window

Kinesis provides a level of persistence that SQS does not. Data ingested is retained in a **rolling window** — **24 hours by default** — during which any consumer can read it. Data older than the window (24 h + 1 s, by default) is discarded.

- The window can be extended up to a **maximum of 365 days**, for additional cost. <cite index="17-1">Retention is 24 hours by default and extendable up to 365 days.</cite>
- The window is **persistent storage**: a 365-day window means 365 days of data physically stored and included in the product. More window = more cost.

### Shards and throughput

Each shard provides a fixed slice of capacity:

- **Ingestion:** 1 MB/s **or** 1,000 records/s per shard. <cite index="25-1">A shard's throughput is limited to 1 MB per second or 1,000 messages per second.</cite>
- **Consumption:** 2 MB/s per shard (shared across all classic consumers — see the trap below).

More shards → more performance **and** more cost. Records have a **max size of 1 MB** and carry a **partition key** that is hashed to decide which shard they land on.

> [!note] 📚 Enhanced fan-out (not in the transcript, but a very common exam distractor) The 2 MB/s consumption figure is the **shared / classic fan-out** limit — _all_ classic consumers on a shard **share** that 2 MB/s. So three classic consumers on one shard get ~667 KB/s each. <cite index="26-1">Standard consumers share the read throughput of each shard.</cite>
> 
> **Enhanced fan-out** gives **each registered consumer its own dedicated 2 MB/s per shard**, pushed via `SubscribeToShard` over HTTP/2 (push, not poll), reducing latency. <cite index="26-1">Enhanced fan-out gives each registered consumer its own dedicated 2 MB/s read throughput per shard, and instead of polling, consumers receive data pushed via HTTP/2.</cite> <cite index="31-1">With On-demand Advantage mode you can register up to 50 consumers per stream for enhanced fan-out; with On-demand Standard and Provisioned streams, up to 20.</cite>
> 
> **Exam cue:** "multiple consumers need full-speed independent reads of the same stream" → **enhanced fan-out**.

### 🎯 SQS vs Kinesis — the classic exam decision

There is almost always a question asking you to choose between these two. Read every such question through this lens:

|Question is about…|Answer|
|---|---|
|**Ingestion** of data (huge scale, many producers)|**Kinesis**|
|Worker pools, **decoupling**, **asynchronous** communication|**SQS** (assume SQS first; only switch with strong reason)|

||**SQS**|**Kinesis Data Streams**|
|---|---|---|
|**Producers**|Typically one thing / one group (e.g. a web tier in an ASG)|Huge numbers (thousands of sensors, clickstreams)|
|**Consumers**|Usually one consumer / worker group|Many independent consumers at different rates|
|**Persistence**|❌ No window — messages deleted after processing, gone forever|✅ Rolling window (24 h → 365 days)|
|**Purpose**|Decoupling application components, async messaging|Ingestion, analytics, monitoring, click streams|

> [!tip] 🎯 Trigger keywords Ingestion · analytics · dashboards · **mobile/click streams** · IoT · "many producers" · "multiple consumers reading independently" · "replay" / re-read data → **Kinesis**. Decouple · worker tier · queue · async · "process once then delete" → **SQS**.

---

## Kinesis Data Firehose

> [!warning] Renamed to **Amazon Data Firehose** (Feb 2024) This product is now **Amazon Data Firehose**. The course teaches it as "Kinesis Data Firehose" and the exam may still use that name, so keep both in mind. Details in [[#Course-vs-current-AWS reality]].

**Firehose is a fully managed, serverless, resilient delivery service.** It accepts streaming data and delivers it to supported destinations in **near real time**, and can transform data in flight with **Lambda**. Its core value is giving Kinesis Data Streams something it lacks natively: **persistence and delivery** to other services (records that age out of the Data Streams window are otherwise gone forever).

### Architecture walkthrough

![[kinesis-data-firehose.png]]

**How to read this diagram:** sources on the **left** feed **Kinesis Data Firehose** (the funnel icon, center). Firehose optionally routes data through a **Lambda** transform (top), and delivers to destinations in the **green panel on the right**.

- **Sources (left):** CloudWatch (Logs and Events), **KPL** (Kinesis Producer Library), a **Kinesis Data Stream**, **Kinesis Agent**, and **IoT**. The key text: _"Producers can send records to data streams **or** send directly at firehose. Firehose can also read from a data stream as a consumer."_
    - ⚠️ Note the label **"Kinesis streams are realtime ~200 ms"** — that's the **Data Streams** side. Firehose itself is _not_ real-time.
- **Transform (top):** **Lambda Transformation Function (Blueprints)** — source records go up to Lambda, transformed records come back down (blue arrows). Blueprints are pre-built functions for common tasks (e.g. Apache/system logs → JSON/CSV). Transformation adds latency.
- **Backup Bucket (bottom):** if you transform, you can optionally keep the **unmodified source records** in a backup S3 bucket ("Source Records" arrow).
- **Buffering (center-right text):** _"delivery when buffer size in MB (1) fills **or** buffer interval in seconds (60) passes."_ This is the near-real-time mechanism (below).
- **Destinations (green panel):** HTTP endpoints, **Splunk**, **Redshift**, **Elasticsearch** (⚠️ now OpenSearch — see below), **S3** / Destination Bucket.
    - **Redshift is special:** delivery to Redshift uses **S3 as an intermediate** — Firehose writes to S3, then runs a **Redshift COPY** (green dotted arrow). Conceptually "direct," but mechanically it's S3 → COPY, handled end-to-end by Firehose.

### Near real-time: the buffer

Firehose _receives_ data in real time but _delivers_ on a buffer. It waits for whichever comes first:

- **1 MB of data**, **or**
- **60 seconds**

Both are adjustable, but these are the general minimums. Low-volume producers hit the 60-second timer; high-volume producers hit the 1 MB size. <cite index="21-1">Firehose buffers data and delivers to destinations; the older architecture used ~1 MB / 60 s buffering.</cite>

> [!tip] 🎯 The single most important Firehose exam fact **Firehose is NOT real-time — it is near real-time (~60 s).** AWS frames **~200 ms** as real-time and **~60 s** as near real-time. If a question demands real-time delivery, **Firehose cannot be the answer** — use Data Streams (with, e.g., a Lambda consumer) instead.

### Destinations (course list) — 🎯 memorize

The course stresses knowing valid destinations, because that's how you rule Firehose in or out:

- **S3** · **Redshift** (via S3 intermediate + COPY) · **Elasticsearch** · **Splunk** · **HTTP endpoints** (third-party providers)

> [!warning] ⚠️ This list has grown a lot — see [[#Course-vs-current-AWS reality]]. For the exam, the course five are the safe core; in reality Firehose now also delivers to OpenSearch, Snowflake, Apache Iceberg tables, S3 Tables, and many more HTTP partners.

### When to use Firehose

- Give **persistence** to data arriving on a Data Stream (so it survives the rolling window).
- Store data in a **different format** (transform via Lambda — Parquet/ORC, etc.).
- **Deliver** stream or direct-PUT data into one of the supported destinations, **without building the plumbing yourself**.

The trade-off: you give up the real-time nature of raw Data Streams in exchange for not having to build delivery.

---

## Kinesis Data Analytics

> [!warning] The SQL variant taught here was **discontinued (Jan 27, 2026)**; the Apache Flink variant was renamed **Amazon Managed Service for Apache Flink** This is the biggest course-vs-reality gap in the whole Kinesis set. The lesson teaches **SQL-based** stream processing — that specific product no longer exists. Full detail in [[#Course-vs-current-AWS reality]]. The **concept** (real-time processing between input and output streams) is still exam-relevant, so study the mechanics; just know the naming is dated.

**Kinesis Data Analytics provides real-time processing of streaming data using SQL.** Data comes in one side, SQL queries run against it in real time, results go out the other side.

- **Ingests from:** Kinesis Data Streams **or** Kinesis Firehose, and can optionally pull **static reference data from S3**.
- **Outputs to:** **Firehose** (and, indirectly, anything Firehose supports — but this makes it **near** real-time), **Lambda**, or **Kinesis Data Streams** (Lambda and Data Streams keep it **real-time**).

> [!tip] 🎯 Real-time vs near real-time depends on the destination Output → **Firehose** (or Firehose's downstream destinations) = **near** real-time. Output → **Lambda** or **Data Streams** = **real-time**. This is a classic distractor.

### Architecture walkthrough

![[kinesis-data-analytics.png]]

**How to read this diagram — three panels, left to right:** blue **Source Stream** → orange **Kinesis Analytics Application** → green **Destination Stream**. The key idea: sources and destinations live **outside** the application; inside, the app works on **virtual tables** that mirror those external streams. Analytics never modifies the sources themselves.

- **Source Stream (blue, left):** the external inputs — a **Kinesis Stream** or **Kinesis Firehose**.
- **Reference Data (teal, bottom-left):** an **S3 bucket** feeding **CSV/JSON** static data in.
- **Inside the Analytics Application (orange, center):**
    - **In-application input stream** (blue-topped table) — a constantly-updated virtual table mirroring the source stream. Think "database table, but a live feed."
    - **Reference Table** (teal-topped table) — mirrors the S3 static data; used to **enrich** the streaming input.
    - **SQL** (black box) — the **application code**. _"Application Code Processes Input, Produces Output."_ It reads the input stream + reference table and writes results out.
    - **In-application output streams** (green-topped tables) — results the SQL produces; these map onto the **real external** destination streams.
    - **In-application error stream** (red-topped table) — errors generated during processing.
- **Destination Stream (green, right):** the external outputs — a **Kinesis Stream** (→ **Consumers**) or **Kinesis Firehose** (→ **Destinations**).

**The enrichment example:** a popular online game streams raw **player scores/activity** on a Kinesis stream. The **reference table** (from S3) holds static **player metadata** — names, owned items, awards — used to enrich the raw stream data in real time.

### When to use it — 🎯

Anything needing **real-time, stream-based SQL processing**:

- **Time-series analytics** (e.g. election data, e-sports)
- **Real-time dashboards / leaderboards** (game high-score tables)
- **Real-time metrics** for security and response teams

> [!tip] 🎯 Data Analytics vs Firehose+Lambda Firehose can also transform with Lambda, but: Firehose is **near real-time** and Lambda transforms are **relatively simple**. Data Analytics supports **complex SQL** and stays **real-time** (with the right destination). **Complex real-time manipulation → Data Analytics.** Also note: it's **not cheap** — you pay per data processed, so it's for scenarios that truly need it.

---

## Kinesis Video Streams

**Ingests live streaming media from producers.** Producers include security cameras, smartphones, cars, drones — and **non-video, time-serialized data** like **audio, thermal, depth, or radar**. Once media is in AWS, consumers access it **frame by frame / as needed** for analysis. Data is encrypted **in transit and at rest** and persisted, as a managed service.

> [!tip] 🎯 You cannot access the ingested data directly on storage Video Streams does **not** store data in its original format on EBS/S3/EFS that you can reach directly — it's **indexed and stored in a structured way inside the product**. You must go **via the product**. Any exam answer claiming you can grab the raw video off S3/EBS/EFS is **wrong**.

**Common integrations:** **Rekognition** (deep-learning analytics on live video, e.g. facial recognition) and **Connect** (voicemail / audio streaming).

### Architecture walkthrough — smart-home face detection

![[kinesis-video-streams.png]]

> [!note] The NoteGPT transcript spells it "Recognition" throughout — the service (and the diagram) is **Rekognition** (with a k).

**How to read this diagram:** video flows **left → right** across the top (cameras → Video Streams → Rekognition), then analysis flows **right → left** across the bottom (Data Stream → Lambda → SNS → alert).

- **Left:** a house with a cat 🐱 and dog 🐶, and **three cameras**. Each camera's **Video Feed** (solid red arrows) goes into its **own Kinesis Video Stream — 1 per camera**. _No in-house processing hardware needed_, and video is stored off-property.
- **Top-center → right:** the three Video Streams feed **Rekognition Video** (dotted red "Video Streams" arrows).
- **Far right — Face Collection:** known faces Rekognition compares against — Bob (homeowner), Julie (friend), a Friend, and the Cat Sitters (blue arrow into Rekognition).
- **Rekognition output (green arrow down):** analysis goes to a **Kinesis Data Stream**. The analysis lists **Detected** faces and flags which ones **Match** a face in the collection.
- **Bottom, right → left:** **Kinesis Data Stream** → **Lambda (analysis, every record)** → decision logic → if an unknown face is detected, **Simple Notification Service (SNS)** sends an alert → Bob/Julie get **"HALP! Unknown human"** 🚨.

This is a simple **event-driven video analytics** pattern. The product can do much more, but at SAA level that's the depth expected.

> [!tip] 🎯 Trigger keywords **Live video streaming**, analytics on a video stream, **GStreamer**, or **RTSP** → default to **Kinesis Video Streams**. Not expected to feature in deep detail on the exam.

---

## Course-vs-current-AWS reality

The Cantrill lessons predate several AWS changes. Both framings are preserved because **exam questions may reflect either** — but you should know the current truth.

> [!warning] ⚠️ Kinesis Data Firehose → **Amazon Data Firehose** (Feb 9, 2024) AWS renamed the service. <cite index="23-1">AWS renamed Amazon Kinesis Data Firehose to Amazon Data Firehose; the change is effective in the console, docs, and web pages, with no changes to endpoints, APIs, CLI, IAM policies, or CloudWatch metrics.</cite> The exam is likely to still say "Kinesis Data Firehose," so recognize both.

> [!warning] ⚠️ Kinesis Data Analytics — renamed **and** the SQL variant discontinued
> 
> - The **Apache Flink** variant was renamed **Amazon Managed Service for Apache Flink** (Aug 30, 2023). <cite index="11-1">AWS renamed Amazon Kinesis Data Analytics to Amazon Managed Service for Apache Flink, effective in the console, documentation, and service webpages, with no changes to endpoints, APIs, CLI, IAM policies, or CloudWatch metrics.</cite>
> - The **SQL** variant — the exact product this lesson teaches — was **discontinued**. <cite index="6-1">AWS discontinued Amazon Kinesis Data Analytics for SQL applications effective January 27, 2026, recommending Amazon Managed Service for Apache Flink (or its Studio) instead.</cite> The wind-down ran over ~15 months: <cite index="8-1">no new SQL applications could be created from October 15, 2025, and existing applications began being deleted starting January 27, 2026.</cite>
> - **What this means for you:** study the _concept_ (real-time processing sitting between input and output streams, enriched by S3 reference data), because the exam question pattern still tests that. But the "SQL-based Kinesis Data Analytics" product framing is historical. Modern real-time stream processing on AWS = **Amazon Managed Service for Apache Flink**.

> [!warning] ⚠️ "Elasticsearch" destination → **Amazon OpenSearch Service** AWS renamed Amazon Elasticsearch Service to **Amazon OpenSearch Service** (2021). The course diagram/list says "Elasticsearch"; current Firehose docs say **OpenSearch Service**. <cite index="15-1">Amazon Data Firehose delivers to destinations such as Amazon S3, Amazon OpenSearch Service, Amazon Redshift, Splunk, and other supported destinations.</cite>

> [!warning] ⚠️ Firehose destinations have expanded well beyond the course five Current supported destinations include the course set **plus** Snowflake, Apache Iceberg tables, Amazon S3 Tables, and many more HTTP partners. <cite index="21-1">Firehose currently supports Amazon S3, Amazon Redshift, Amazon OpenSearch Service, Snowflake, Apache Iceberg tables, Splunk, Datadog, New Relic, Dynatrace, Sumo Logic, LogicMonitor, MongoDB, and HTTP endpoints as destinations.</cite> For the exam, the **course five (S3, Redshift, Elasticsearch/OpenSearch, Splunk, HTTP)** remain the reliable core to recognize.

> [!note] 📚 Capacity modes and enhanced fan-out (beyond the lesson) The lesson describes Data Streams in **provisioned** (shard-managed) terms. In reality, Data Streams also offers an **on-demand** capacity mode that auto-scales. <cite index="27-1">A new on-demand stream starts at 4 MB/s and 4,000 records/s for writes and automatically scales up to 200 MB/s and 200,000 records/s.</cite> Pair this with the [[#Kinesis Data Streams|enhanced fan-out]] note above.

---

## 🎯 Exam Traps — trigger keyword table

|You see…|Think…|Why|
|---|---|---|
|"ingest data from thousands of devices/sensors/clickstreams"|**Kinesis Data Streams**|Ingestion at scale = Kinesis, not SQS|
|"decouple components," "worker tier," "async," "process once"|**SQS**|No persistence/window; single consumer group|
|"multiple consumers must **replay** / re-read the same data"|**Kinesis Data Streams**|Rolling window + many independent consumers|
|"**real-time** delivery to S3"|❌ **not Firehose** → Data Streams + Lambda|Firehose is **near** real-time (~60 s)|
|"**near** real-time delivery to S3/Redshift/Splunk, no infra to build"|**Firehose**|Managed delivery, ~1 MB / 60 s buffer|
|"load streaming data into **Redshift**"|**Firehose** (S3 intermediate + COPY)|Redshift delivery is indirect via S3|
|"**complex** real-time transformation with SQL"|**Data Analytics** (now Managed Flink)|Firehose+Lambda = simple + near-real-time only|
|"enrich a stream with **static reference data**"|**Data Analytics** + S3 reference table|Core Analytics pattern|
|"persist stream data past the window / archive to a data lake"|**Firehose**|Data Streams alone doesn't persist beyond the window|
|"live **video**," "**RTSP**," "**GStreamer**," drone/camera feeds + analytics|**Kinesis Video Streams** (+ Rekognition)|Media ingestion; can't read raw off S3/EBS/EFS|
|"many consumers each need full 2 MB/s, no contention"|**Enhanced fan-out**|Classic consumers _share_ 2 MB/s per shard|

---

## Quick reference

|Attribute|Data Streams|Firehose (now _Amazon Data Firehose_)|Data Analytics (now _Managed Flink_)|Video Streams|
|---|---|---|---|---|
|**Purpose**|Ingest + persist + multi-consumer read|Deliver to destinations|Real-time SQL processing|Live media ingestion|
|**Latency**|Real-time (~200 ms)|**Near** real-time (~60 s)|Real-time (near real-time if → Firehose)|Live|
|**Persistence**|24 h default, up to 365 days|Buffers only (~1 MB / 60 s)|Transient (virtual tables)|Persisted internally|
|**Scaling unit**|Shards (1 MB/s in, 2 MB/s out each)|Fully serverless/auto|Serverless/auto|Managed|
|**Record size**|1 MB max|—|—|—|
|**Sources**|Producers (SDK/KPL/Agent)|Direct PUT, Data Streams, CloudWatch, IoT, KPL, Agent|Data Streams or Firehose (+ S3 reference)|Cameras, phones, cars, drones; audio/thermal/depth/radar|
|**Destinations**|Consumers (EC2, Lambda, on-prem)|S3, Redshift, Elasticsearch/OpenSearch, Splunk, HTTP (+ many more now)|Firehose, Lambda, Data Streams|Consumers via product API; Rekognition, Connect|
|**Transform**|No (consumer does it)|Lambda (simple)|SQL (complex)|n/a|
|**Real-time?**|✅|❌ (near)|✅ (dest-dependent)|✅|

---

## Open Questions for Roger

1. **Cross-linking — which of these already exist in your vault?** This note wikilinks to (conceptually) `[[SQS]]`, `[[Lambda]]`, `[[S3]]`, `[[Redshift]]`, `[[CloudWatch]]`, `[[Rekognition]]`, and `[[SNS]]`. I left the internal references as plain text or bare mentions rather than forcing `[[...]]` links — want me to wikilink any of these?
2. **Depth on the renames.** I kept the course names as primary (since the exam likely uses them) and flagged reality in callouts. Do you want a dedicated standalone **"AWS service renames"** note to collect these (Firehose, Managed Flink, OpenSearch) across topics, or is inline flagging enough?
3. **Managed Flink sub-note?** Since KDA-for-SQL is effectively gone, a short `[[Amazon Managed Service for Apache Flink]]` stub might be worth it for completeness — though it's arguably beyond SAA-C03 scope. Skip, or add a minimal one?
4. **Enhanced fan-out depth.** I added it as a 📚 note because it's a frequent distractor. Want it broken out with the `GetRecords` / `SubscribeToShard` API detail, or is the current level right for SAA?
5. **KPL / Kinesis Agent / KCL.** These appear on the Firehose diagram but the transcripts don't explain them. Want a short "producers/consumers tooling" aside, or leave as-is?

---

## Source attribution

Course content: Adrian Cantrill SAA-C03 — _Kinesis Data Streams_, _Kinesis Data Firehose_, _Kinesis Data Analytics_, _Kinesis Video Streams_ lessons (learn.cantrill.io). Diagrams © Adrian Cantrill. Current-AWS fact-checks cited inline from AWS What's New posts, AWS documentation, and the Amazon Data Firehose / Kinesis Data Streams FAQs (accessed Jul 2026).
