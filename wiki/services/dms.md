---
title: DMS — Database Migration Service
type: service
tags: [dms, database-migration, migration, replication, cdc, full-load, replication-instance, replication-task, endpoint, dms-serverless, dcu, schema-conversion, dms-schema-conversion, sct, homogeneous-migration, heterogeneous-migration, snowball, s3, rds, aurora, redshift, dynamodb, vpc, kms, cloudwatch, cloudtrail]
certs: [aws-saa]
updated: 2026-11-04
---

# DMS — Database Migration Service

AWS Database Migration Service is the managed service for moving and replicating database and analytics data. The strongest exam heuristic is still simple: if a scenario describes a database migration to AWS, from AWS, or between AWS-hosted databases, start with DMS. If it adds "minimal downtime" or "keep the source database online," the answer is usually DMS with change data capture (CDC).

Modern DMS is broader than the older "replication instance" mental model. The classic model still exists as **DMS Standard**: source endpoint, target endpoint, replication instance, and replication task. Current DMS also includes **DMS Serverless**, **DMS Schema Conversion**, and serverless homogeneous migration paths. Older notes that describe only the EC2-backed replication instance are accurate for DMS Standard, but not for the full current service.

## Where DMS Fits

DMS sits between source and target data stores. It moves table data, can keep applying source changes with CDC, and can stream into AWS analytics targets such as [S3](s3.md), [Redshift](redshift.md), [Kinesis](kinesis.md), [OpenSearch](opensearch.md), DynamoDB, DocumentDB, Neptune, and relational engines such as [RDS](rds.md) and [Aurora](aurora.md).

![DMS architecture: a source database feeds a source endpoint, a DMS replication instance runs one or more replication tasks, a destination endpoint points at the target database, and the supported task modes are Full Load, Full Load plus CDC, and CDC Only. The side note names SCT as the schema-conversion companion for engine changes.](../assets/dms-architecture.png)

The diagram is the classic **DMS Standard** shape:

1. A **source endpoint** stores the connection details for the source.
2. A **replication instance** runs the migration engine.
3. One or more **replication tasks** define what to move and how.
4. A **target endpoint** stores the connection details for the destination.

The schema-conversion note on the right needs a modern update: **AWS SCT** still exists, but AWS also offers **DMS Schema Conversion** in the DMS console.

For SaaS application records exposed through connectors rather than a database migration endpoint, consider [Amazon AppFlow](appflow.md). The [data integration comparison](../concepts/data-integration.md#choosing-a-data-movement-service) distinguishes these from stream delivery and event routing.

## DMS Standard

DMS Standard is the provisioned model. You create a replication instance in a [VPC](vpc.md), connect it to source and target endpoints, then run replication tasks on it.

| Component                    | Role                                                                                                                                                    |
| ---------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Replication instance**     | The compute engine for DMS Standard; AWS creates it on EC2 in a VPC. It handles reads from the source, buffering, formatting, and writes to the target. |
| **Source endpoint**          | Connection information, credentials, network settings, and engine-specific options for the source data store.                                           |
| **Target endpoint**          | Connection information and engine-specific write options for the target data store.                                                                     |
| **Replication task**         | The unit of work: selected schemas/tables, table mappings, transformation rules, load mode, logging, error handling, CDC settings, and validation.      |
| **Replication subnet group** | The subnets where DMS can place the replication instance. This is the same placement idea as an RDS DB subnet group.                                    |

A replication instance can run multiple tasks, but it must be sized for the aggregate workload. Most processing happens in memory, while large transactions, cached changes, and log files can buffer to disk. For HA, DMS Standard supports Multi-AZ replication instances, where DMS maintains a synchronous standby in another AZ.

> [!warning] Gotcha
> DMS is **not** an on-premises-to-on-premises migration tool. Current AWS FAQ wording is that either the source or target database, or both, must reside in RDS or on EC2; on-premises to on-premises replication is not supported. In exam shorthand: at least one side should be AWS-hosted.

## DMS Serverless

DMS Serverless removes replication-instance management. You still define endpoints and a replication configuration, but DMS estimates capacity, provisions it, scales it, handles engine-version management, and charges based on DMS Capacity Units (DCUs). One DCU is 2 GiB of RAM.

Use DMS Serverless when migration load is variable, when you do not want to right-size a replication instance manually, or when a heterogeneous migration needs elastic capacity. For stable, predictable workloads, a provisioned replication instance can still be appropriate because it can be deliberately right-sized for cost and performance.

> [!tip] Exam Tip
> Older questions often equate DMS with a **replication instance**. Newer wording can mention **DMS Serverless**, **serverless replications**, or **DCUs**. Those point to the managed capacity model, not a provisioned replication instance.

## Migration Task Modes

| Mode                   | What it does                                                                                                                              | Use when                                                                                                              |
| ---------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| **Full load**          | Copies existing data from source to target, then stops.                                                                                   | You can afford downtime or the source will not keep changing during the migration.                                    |
| **Full load plus CDC** | Copies existing data while capturing source changes, then applies the cached changes and keeps replicating ongoing changes until cutover. | You need minimal downtime and the source must stay online during the load.                                            |
| **CDC only**           | Replicates changes only, starting from a known log position or time.                                                                      | Existing data was bulk-loaded by another method, such as native engine tooling, SCT extraction, or physical transfer. |

> [!tip] Exam Tip
> "No downtime" or "minimal downtime" database migration means **full load plus CDC**. The application keeps writing to the source while DMS loads old data and captures new changes; cutover happens after the target catches up.

> [!warning] Gotcha
> **CDC only** does not move the historical baseline. It assumes the target already has the existing data and DMS only needs to replay subsequent changes.

## Homogeneous vs Heterogeneous

| Migration type    | Meaning                                                                        | Schema tooling                                                                                                                   |
| ----------------- | ------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------- |
| **Homogeneous**   | Same or compatible engine, such as self-managed MySQL to Amazon RDS for MySQL. | Usually use native engine tooling or DMS homogeneous migration features; no cross-engine conversion is needed.                   |
| **Heterogeneous** | Engine changes, such as Oracle to Aurora PostgreSQL or SQL Server to MySQL.    | Use **DMS Schema Conversion** or **AWS SCT** to assess and convert schema/code, then use DMS to move data and replicate changes. |

DMS can create basic target tables and primary keys when they do not exist, but serious production migrations should not treat data movement and schema conversion as the same job. Tables, indexes, stored procedures, functions, data types, views, triggers, and engine-specific SQL often need assessment and conversion before the data load.

### DMS Schema Conversion vs AWS SCT

**DMS Schema Conversion** is the managed, console-based schema conversion feature inside DMS. It assesses migration complexity, converts schemas and many code objects, marks objects that need manual work, and can save assessment reports or converted SQL to [S3](s3.md). AWS has also added a generative-AI-assisted conversion feature to DMS Schema Conversion.

**AWS Schema Conversion Tool (AWS SCT)** is the downloadable desktop tool. It remains important because it has broader coverage than the web-based DMS Schema Conversion feature, especially for data warehouse schemas, big-data frameworks, application SQL, and ETL processes.

> [!warning] Gotcha
> The old rule "SCT only when the engine changes" is still a good SAA shortcut, but modern AWS wording may say **DMS Schema Conversion** instead of SCT. The underlying discriminator is the same: **same engine = no cross-engine schema conversion; different engine = assess/convert schema and code before moving data**.

## Large Migrations with Snowball and S3

For very large migrations, the network can be the bottleneck. DMS can be paired with AWS SCT agents and AWS Snowball Edge:

1. SCT extraction or replication agents copy source data to local files or to a Snowball Edge device.
2. The Snowball Edge device is shipped to AWS.
3. AWS loads the data into [S3](s3.md).
4. DMS loads from S3 into the target endpoint.
5. If CDC is part of the plan, DMS applies changes that occurred after the initial bulk extract.

This pattern is for multi-TB or larger migrations where over-the-network bulk copy would take too long or consume too much production bandwidth. It does not replace CDC; it gives CDC a smaller catch-up window after the baseline data lands.

## Networking and Security

DMS works only if the replication compute can reach both endpoints. For DMS Standard, that means putting the replication instance in subnets with routes and security group rules to reach the source and target. Sources and targets can be in [RDS](rds.md), [Aurora](aurora.md), [EC2](ec2.md), another AWS service, or a connected external network over VPN or Direct Connect.

Security facts to keep separate:

- **Network reachability** is VPC routing plus security groups, NACLs, VPN, Direct Connect, or peering/transit connectivity.
- **Database authentication** is stored in DMS endpoints or data providers, typically backed by Secrets Manager in newer DMS workflows.
- **Encryption at rest** for DMS resources uses [KMS](kms.md).
- **Encryption in transit** depends on endpoint SSL/TLS support and endpoint settings.
- **Monitoring** integrates with [CloudWatch](cloudwatch.md) logs/metrics and [CloudTrail](cloudtrail.md) API audit events.

## Current-Service Notes

- **DMS Fleet Advisor is end of support.** AWS ended support on May 20, 2026. If older material describes Fleet Advisor as the planning front door, treat that as historical and prefer AWS Application Discovery Service, Migration Evaluator, native assessment reports, or current DMS Schema Conversion assessment workflows depending on the scenario.
- **DMS homogeneous data migrations are serverless.** Current AWS FAQ describes the built-in homogeneous migration path as serverless and not based on replication instances.
- **DMS supports Database Savings Plans.** This is cost-awareness, not usually an SAA discriminator.
- **DMS does not make unsupported conversions magical.** Unsupported data types, incompatible SQL, triggers, stored procedures, collations, and target-engine feature gaps still require assessment and manual remediation.

## Exam Quick Reference

| Trigger                                                             | Answer                                                             |     |
| ------------------------------------------------------------------- | ------------------------------------------------------------------ | --- |
| Database migration to AWS / from AWS / between AWS-hosted databases | **DMS**                                                            |     |
| Minimal downtime, source remains online                             | **Full load plus CDC**                                             |     |
| Existing baseline already loaded, only changes needed               | **CDC only**                                                       |     |
| Same engine, such as MySQL to RDS MySQL                             | Homogeneous migration; **no SCT/schema conversion** needed         |     |
| Different engines, such as Oracle to Aurora PostgreSQL              | Heterogeneous migration; **DMS Schema Conversion or SCT** plus DMS |     |
| "Replication instance"                                              | DMS Standard provisioned model                                     |     |
| "Serverless replications" or "DCUs"                                 | DMS Serverless                                                     |     |
| On-premises to on-premises only                                     | Not supported                                                      |     |
| Multi-TB migration constrained by bandwidth                         | SCT/Snowball/S3 bulk load plus DMS CDC                             |     |
| Build an S3 data lake from databases                                | DMS can continuously replicate supported sources into S3           |     |

## References

- Cantrill SAA-C03 study notes — *Database Migration Service (DMS)*
- [AWS — What is AWS Database Migration Service?](https://docs.aws.amazon.com/dms/latest/userguide/Welcome.html)
- [AWS — Working with an AWS DMS replication instance](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_ReplicationInstance.html)
- [AWS — Working with AWS DMS Serverless](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Serverless.html)
- [AWS — AWS DMS Serverless components](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Serverless.Components.html)
- [AWS — Working with AWS DMS tasks](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Tasks.html)
- [AWS — DMS Schema Conversion](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_SchemaConversion.html)
- [AWS — DMS Fleet Advisor end-of-support notice](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_FleetAdvisor.html)
- [AWS — DMS FAQ](https://aws.amazon.com/dms/faqs/)
- [AWS SCT — Integrating AWS DMS with AWS SCT](https://docs.aws.amazon.com/SchemaConversionTool/latest/userguide/CHAP_DMSIntegration.html)
- Related: [RDS](rds.md) · [Aurora](aurora.md) · [S3](s3.md) · [VPC](vpc.md) · [KMS](kms.md) · [CloudWatch](cloudwatch.md) · [CloudTrail](cloudtrail.md)
