---
title: Data Integration — Storage, Catalogs, ETL, and Queries
type: concept
tags: [data-integration, analytics, etl, elt, data-lake, metadata, glue, s3, appflow, saas, orchestration]
certs: [aws-saa]
updated: 2026-11-12
---

# Data Integration — Storage, Catalogs, ETL, and Queries

## Separate the Responsibilities

A data pipeline is easier to design when each service has a clear responsibility. [AWS Glue](../services/glue.md) covers discovery and transformation, while [S3](../services/s3.md) stores datasets and [Step Functions](../services/step-functions.md) can coordinate the work.

| Responsibility | Question it answers | Example |
| --- | --- | --- |
| Ingestion | How do records arrive? | [AppFlow](../services/appflow.md) for SaaS records; [DMS](../services/dms.md) for database migration and CDC |
| Storage | Where are the records? | S3 objects; see [storage fundamentals](storage.md#object-storage) |
| Catalog | Where is the dataset, and what structure does it have? | Glue Data Catalog table definitions |
| Discovery | How do we infer those definitions? | Glue crawlers |
| Transformation | How should the records change? | Glue jobs that clean, join, enrich, or convert formats; [Athena for Apache Spark](../services/athena-spark.md) notebooks for interactive PySpark work |
| Query | What answer can we compute from the records? | [Athena](../services/athena.md) SQL over S3, [Redshift Spectrum](../services/redshift.md#querying-data-without-loading-it), or an EMR processing engine |
| Orchestration | What runs next, and what happens on failure? | Glue workflows or Step Functions |
| Governance | Who may discover or use the data? | [IAM](../services/iam.md) and Lake Formation permissions |

The catalog is shared metadata, not a replacement for storage or a query engine. Cataloging data makes it discoverable but does not automatically authorize every consumer. [AWS catalog model and integrations](https://docs.aws.amazon.com/glue/latest/dg/catalog-and-crawler.html).

## ETL, ELT, and Streaming

**ETL** means extract, transform, then load into the target. **ELT** loads first and transforms in the destination environment. The distinction is where transformation occurs relative to the destination, not whether a system is serverless.

For example, a Glue job can read raw S3 JSON, normalize timestamps, join reference data, and write curated Parquet to a second S3 prefix. That is ETL relative to the curated destination, even though the raw input already lives in S3. A warehouse pipeline that loads raw rows and then transforms them inside the warehouse is ELT.

**Batch versus streaming is a separate decision.** A batch job processes a bounded input and finishes. A streaming job repeatedly consumes arriving records. [Kinesis Data Streams](../services/kinesis.md#kinesis-data-streams) and [Kafka](streaming-vs-messaging.md#kinesis-vs-kafka-and-amazon-msk) can supply Glue streaming ETL; [Firehose](../services/kinesis.md#kinesis-data-firehose-now-amazon-data-firehose) provides managed delivery with optional transformations. [AWS Glue workload types](https://docs.aws.amazon.com/glue/latest/dg/what-is-glue.html), [streaming ETL](https://docs.aws.amazon.com/glue/latest/dg/add-job-streaming.html).

## Choosing a Data Movement Service

Choose by the data source, transfer semantics, and processing required, rather than the broad label “integration.”

| Requirement | Starting point | Boundary to remember |
| --- | --- | --- |
| Transfer supported SaaS records with mapping, masking, and filters | [AppFlow](../services/appflow.md) | Check connector directions, triggers, and quotas; custom connectors require code |
| Migrate a database and keep applying source changes | [DMS](../services/dms.md) | Targets include analytics services and S3, not only databases |
| Retain an ordered stream for independent consumers and replay | [Kinesis Data Streams](../services/kinesis.md#kinesis-data-streams) | Consumers process records; it is not a SaaS connector catalog |
| Deliver incoming streaming data to a supported destination | [Data Firehose](../services/kinesis.md#kinesis-data-firehose-now-amazon-data-firehose) | Managed delivery with buffering and optional transformations |
| Match events and route them to targets | [EventBridge](../services/eventbridge.md) | Event routing is different from bulk record transfer |
| Discover datasets or perform flexible joins, cleanup, and format conversion | [Glue](../services/glue.md) | Separate catalog/crawler work from ETL jobs; connectors can also reach external sources |

These services can cooperate. In the [AppFlow SaaS-to-analytics example](../services/appflow.md#worked-example--saas-records-to-an-analytics-dataset), AppFlow transfers records to S3 and can populate Glue catalog metadata directly, then a Glue job performs further preparation. A crawler is one way to register metadata, not a mandatory step in every pipeline. [AWS AppFlow catalog integration](https://docs.aws.amazon.com/appflow/latest/userguide/flows-catalog.html).

## Worked Example — Raw Files to a Shared Dataset

The [lease-record walkthrough on the Glue page](../services/glue.md#worked-example--s3-lease-records-to-shared-catalog-consumers) makes this concrete with monthly CSV files, crawler configuration, the `raw.leases` catalog table, Athena SQL, a Glue read, and Redshift Spectrum/EMR consumers. It also distinguishes registering new partitions from Athena-only partition projection.

1. An application lands records in an S3 raw-data prefix.
2. A Glue crawler discovers the file schema, or the team defines the catalog table explicitly.
3. A Glue job reads the raw dataset and writes cleaned output to a curated prefix.
4. The output table's metadata is created or updated to match the new dataset.
5. Authorized analysts query the curated files with Athena using the catalog definition.

[EventBridge](../services/eventbridge.md) can initiate a Glue workflow when data arrives; Step Functions can coordinate the job with validation and downstream services. The catalog and transformation job are reusable independently: querying suitable raw files does not inherently require a preceding ETL job.

> [!warning] Gotcha
> Three different failures require different fixes: **wrong schema → fix metadata; wrong values → fix data or transformation logic; access denied → fix permissions/connectivity**. Re-running a crawler does not clean records or grant access.

## References

- [Amazon AppFlow study note](../../raw/aws-cert-study-notes/aws-solution-architect/amazon_appflow.md)
- [AWS — Cataloging AppFlow output](https://docs.aws.amazon.com/appflow/latest/userguide/flows-catalog.html)
- [AWS Glue study note](../../raw/aws-cert-study-notes/aws-solution-architect/aws_glue.md)
- [AWS — Data discovery and cataloging](https://docs.aws.amazon.com/glue/latest/dg/catalog-and-crawler.html)
- [AWS — Glue overview](https://docs.aws.amazon.com/glue/latest/dg/what-is-glue.html)
- [AWS — Glue workflows from EventBridge](https://docs.aws.amazon.com/glue/latest/dg/starting-workflow-eventbridge.html)
