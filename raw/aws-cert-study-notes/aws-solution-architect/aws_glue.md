---
tags:
  - aws
  - saa-c03
  - analytics
  - etl
  - glue
course: Adrian Cantrill SAA-C03
lesson: AWS Glue 101
---

# AWS Glue

## Overview

**AWS Glue** is a **serverless ETL** (Extract, Transform, Load) and **data catalog** service — part of AWS's data and analytics family.

Two halves, one product:

| Component | Job |
|---|---|
| **Data Catalog** (+ crawlers) | Discover data sources and store their *metadata* (schemas) centrally |
| **Glue Jobs** | Move data from a source to a target, transforming it in between |

- **Serverless** — AWS owns and manages the compute; you pay only for what jobs consume.
- Use a Glue job when you need to **restructure or enrich** source data on its way to a destination.

> [!note] Scope
> Cantrill covers Glue at the architectural-theory level only — that is the depth the exam expects.

### Supported locations (per course)

| Role | Examples |
|---|---|
| **Source data stores** | S3, RDS, JDBC-compatible databases (e.g. Redshift), DynamoDB |
| **Source streams** | Kinesis Data Streams, Apache Kafka |
| **Targets** | S3, RDS, JDBC-compatible databases |

> 🎯 Note the asymmetry: DynamoDB and streams appear as **sources**, not targets, in the course's list.

---

## Diagram Walkthrough

![[aws-glue-architecture.png]]

The diagram shows **two flows that share the Data Catalog**: a blue *discovery* flow (top) and a red→green *ETL* flow (middle). Read it in that order.

**1. Discovery flow (blue arrows, top-left → centre)**
- **Data stores (left)** — the cylinders represent the sources listed bottom-left (S3, RDS, JDBC, DynamoDB, Kinesis Data Streams, Kafka).
- **Crawler (bug-over-table icon)** — connects to each store, **determines its schema**, and creates **metadata**.
- **Data Catalog (blue box)** — the metadata lands here. The three search icons represent catalogued, *searchable* table definitions — not the data itself.

**2. ETL flow (red → green, middle row)**
- **Data Extract (red arrow)** — a Glue job pulls data from a source store.
- **Glue Job (pink box)** — the engine runs your **script**, which defines the transformation.
- **Data Load (green arrow)** — transformed output goes to a target (S3, RDS, JDBC).

**3. How the pieces connect (dark two-way arrows)**
- **Catalog ↔ Glue Job** — jobs *read* the catalog to know a source's schema (and can write updated metadata back). This is why the catalog is the hub of the diagram.
- **Glue Job ↔ AWS Managed Warm Resource Pool (orange box)** — when a job runs, Glue allocates compute from an AWS-owned warm pool and releases it after. This *is* the "serverless" claim made visual: no compute sits in your account.

**4. Control plane (red arrows, right side)**
- **Management console** — access to browse the Data Catalog and control jobs.
- **Job initiation** — jobs start **manually** or via **events / schedules in EventBridge**.

> **Key takeaway:** crawlers *describe* data → catalog *stores the description* → jobs *use the description* to move and transform data on borrowed compute.

---

## Mechanics

### Data Catalog

- A **data catalog** = persistent **metadata** about data sources, plus tools to manage and search it.
- **One Data Catalog per AWS account per region.** Confirmed: [AWS Glossary — Glue Data Catalog](https://awsglossary.org/terms/glue-data-catalog)
- **Purpose — break data silos:** data that one team gathers becomes visible and usable elsewhere in the org (Cantrill's example: finance using data collected by other teams).
- **Population:** configure **crawlers**, give them **credentials**, point them at sources.
- **Consumers of the catalog:**
  - **Athena** — SQL over S3 using catalog table definitions
  - **Redshift Spectrum** — queries S3 via an external schema in the catalog
  - **EMR** — can use it as a Hive-compatible metastore
  - **Lake Formation** — builds governance/access control on top of it

### Glue Jobs

- **Extract** from source → **transform** via a script you write → **load** to target.
- Compute comes from an AWS-managed **warm pool**; you never provision or manage it.
- **Billing:** only for resources consumed.
  - 📚 Billed in **DPU-hours, per second**, with a **1-minute minimum** for Spark jobs on Glue 2.0+ (older versions: 10-minute minimum). Crawlers are also DPU-billed (10-minute minimum per run). Catalog: first 1 million metadata objects free per month. [Cloudchipr — Glue pricing](https://cloudchipr.com/blog/aws-glue-pricing) — *beyond transcript; exam rarely tests figures, flag for review.*
- **Starting jobs:** manually, or event-driven / scheduled via **EventBridge**.
  - 📚 Glue's own trigger types: `SCHEDULED` (cron), `CONDITIONAL` (fires when other jobs/crawlers reach a state — used to chain jobs), `ON_DEMAND`, and `EVENT` (EventBridge events, with optional batching). [AWS docs — Glue Triggers](https://docs.aws.amazon.com/glue/latest/dg/aws-glue-api-jobs-trigger.html)

### 📚 Worked Example: S3 → Crawler → Catalog → Consumers

*Beyond the transcript — an illustrative scenario, not values to memorize.*

**1. Raw data** — lease records land in S3 as CSV, foldered by month:

```
s3://company-data/leases/year=2026/month=10/leases_01.csv
s3://company-data/leases/year=2026/month=11/leases_01.csv
```

```csv
lease_id,property,unit,monthly_rent,start_date
L1001,Sunset Apts,204,2450.00,2026-10-01
```

At this point no service knows the columns or types — it is just files.

**2. What you configure vs what the crawler infers**

| You configure | Crawler does automatically |
|---|---|
| Data source (S3 path, JDBC connection) | Detects format via a prioritized list of **built-in classifiers** (CSV, JSON, Parquet…) |
| IAM role (the "credentials") | Infers columns and types from sampled data |
| Target catalog database | Infers **partition keys** from the folder structure |
| Run mode: on demand / schedule / event | Groups same-schema files into one table |

- You do **not** define the schema. Write a **custom classifier** (e.g. grok pattern) only when inference is wrong for unusual formats; custom classifiers run before built-in ones.
- Inference is not perfect — e.g. a date column is often typed as `string`.

**3. Partition inference**

- **Hive-style paths** (`year=2026/month=10/`) → keys named `year`, `month`.
- **Plain paths** (`2026/10/`) → keys named `partition_0`, `partition_1` (renamable in the catalog afterwards).
- **Inconsistent folder structure or schemas** → the crawler may split the data into **multiple tables** instead of one partitioned table.
- ⚠️ **New partitions are not registered automatically.** A new `month=11/` folder is invisible to the catalog until the next crawl. Options:
  - Re-run the crawler on a schedule (can be limited to new folders)
  - Athena `MSCK REPAIR TABLE` (Hive-style paths only)
  - Athena **partition projection** — computes partitions from rules instead of reading them from the catalog

**4. What lands in the catalog** — table `raw.leases`:

| Property | Value |
|---|---|
| Columns | lease_id (string), property (string), unit (bigint), monthly_rent (double), start_date (string) |
| Partition keys | year, month |
| Location | `s3://company-data/leases/` |
| Classification / SerDe | csv / how engines parse the files |

> **Key point:** the catalog stores the *description* only. The data never leaves S3.

**5. How the catalog is consumed**

```
S3 CSV files ──crawler infers schema──▶ Catalog: raw.leases
                                             │
            ┌──────────────┬─────────────────┼──────────────┐
            ▼              ▼                 ▼              ▼
         Athena        Glue Job      Redshift Spectrum     EMR
       (SQL query)   (ETL transform)  (warehouse joins)  (Spark/Hive)
```

- **Athena** — query the files as a table, no loading:
  ```sql
  SELECT property, AVG(monthly_rent) FROM raw.leases
  WHERE year = '2026' AND month = '10' GROUP BY property;
  ```
  Filtering on partition keys means only `month=10` files are scanned → faster and cheaper (Athena bills by data scanned).
- **Glue job** — finds source and schema through the catalog, no hard-coded paths:
  ```python
  df = glueContext.create_dynamic_frame.from_catalog(database="raw", table_name="leases")
  ```
  Typical use: convert CSV → Parquet and fix types, write to a `curated/` prefix, then crawl that too → **raw → curated** data-lake layering.
- **Redshift Spectrum** — mount the catalog database as an external schema and JOIN S3 data with tables inside Redshift, without loading it.
- **EMR** — use the catalog as the Hive metastore (`spark.table("raw.leases")`); table definitions outlive the cluster.

This is the "break data silos" point in practice: operations produces the data, while finance (Athena), data engineering (Glue) and analytics (Redshift/EMR) all work from the **same schema definition**.

---

## Glue vs Data Pipeline

| | **AWS Glue** | **AWS Data Pipeline** |
|---|---|---|
| Compute | Serverless — AWS-managed warm pool | Runs **in your account** (see ⚠️ below) |
| Management | None | You own the provisioned resources |
| Exam keywords | **serverless, ad hoc, cost-effective** | (legacy) |
| Status | Active | ⚠️ Maintenance mode — closed to new customers |

Cantrill notes Data Pipeline offers some functionality Glue doesn't, but expected Glue to absorb its role over time.

---

## Course vs Reality

> [!warning] ⚠️ Data Pipeline is closed to new customers
> **Course:** Data Pipeline presented as a live alternative; Glue "expected" to replace it eventually.
> **Reality:** AWS Data Pipeline **stopped accepting new customers on 25 July 2024** and is in maintenance mode (no new features or regions). Existing customers can keep using it. [AWS docs — What is Data Pipeline](https://docs.aws.amazon.com/datapipeline/latest/DeveloperGuide/what-is-datapipeline.html), [AWS Big Data Blog — Migrate from Data Pipeline](https://aws.amazon.com/blogs/big-data/migrate-workloads-from-aws-data-pipeline/)
> AWS's named migration targets: **Glue** (ETL/serverless), **Step Functions** (orchestrating multiple AWS services), **Amazon MWAA** (managed Airflow).
> **Exam impact:** Data Pipeline may still appear as a distractor. Cantrill's keyword rule still holds.

> [!warning] ⚠️ Data Pipeline compute — EC2, not only EMR
> **Course:** Data Pipeline "creates EMR clusters" to perform tasks.
> **Reality:** Data Pipeline creates **EC2 instances** to run activities; **EMR clusters** are launched for EMR-type workloads. The underlying point — compute lives *in your account*, unlike Glue — is correct. [AWS docs — What is Data Pipeline](https://docs.aws.amazon.com/datapipeline/latest/DeveloperGuide/what-is-datapipeline.html)

---

## Exam Traps

| 🎯 Trigger / scenario | Answer |
|---|---|
| ETL, **serverless**, **ad hoc**, or **cost-effective** | **Glue** |
| Both Glue and Data Pipeline offered as options | Pick **Glue** when the keywords above appear |
| Need to discover schemas of data in S3/RDS/DynamoDB automatically | **Glue crawlers** → Data Catalog |
| Make data visible across teams / avoid data silos | **Glue Data Catalog** |
| What metadata store do Athena / Redshift Spectrum / EMR / Lake Formation use? | **Glue Data Catalog** |
| How many catalogs? | **One per account, per region** |
| Run ETL on a schedule or in response to events | Glue job triggered via **EventBridge** |
| 📚 New S3 data not showing up in Athena results | Partition not in catalog → re-run crawler, `MSCK REPAIR TABLE`, or partition projection |
| 📚 Reduce Athena query cost on partitioned S3 data | Filter on **partition keys** in `WHERE` |
| "Manage the compute that runs Glue transforms" | Distractor — Glue is serverless (AWS warm pool) |

---

## Quick Reference

| Term | Definition |
|---|---|
| **ETL** | Extract from source → Transform → Load into target |
| **Data Catalog** | Regional, persistent metadata store about data sources; searchable, shared across the account |
| **Crawler** | Connects to a store with credentials, infers schema, writes metadata to the catalog |
| **Classifier** | Rule a crawler uses to recognise a format and infer schema; built-in or custom (custom runs first) |
| **Partition key** | Column derived from the folder path (e.g. `year`, `month`), letting engines skip irrelevant files |
| **Glue Job** | Script-driven ETL run on AWS-managed compute |
| **Warm Resource Pool** | AWS-owned compute Glue borrows per job run — the basis of "serverless" |
| **Data Pipeline** | Legacy ETL/orchestration service using compute in your account; closed to new customers since Jul 2024 |

Related: [[Kinesis]] · [[EventBridge]] · [[Step Functions]] · [[Athena]] · [[Redshift]] · [[EMR]]

---

## Open Questions (for Roger)

1. **📚 additions** — keep the DPU billing detail and Glue trigger types, or trim to stay closer to exam scope?
2. **Cross-links** — I linked `[[Athena]]`, `[[Redshift]]`, `[[EMR]]`, `[[Step Functions]]`; which of these exist in your vault, and should Step Functions get a back-link noting it as a Data Pipeline replacement?
3. **Data Pipeline** — standalone stub note for the deprecation, or keep it folded into this note?
4. **📚 Worked Example** — the partition/Athena material leans toward [[Athena]] territory. Keep it here, or move the partitioning part into the Athena note and link back?
