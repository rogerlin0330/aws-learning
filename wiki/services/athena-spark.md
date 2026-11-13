---
title: Amazon Athena for Apache Spark
type: service
tags: [athena, spark, pyspark, analytics, serverless, s3, notebooks, dataframes, dpu, workgroups, glue-data-catalog, iceberg, delta-lake, data-engineering, etl, sagemaker-unified-studio]
certs: [aws-saa]
updated: 2026-11-12
---

# Amazon Athena for Apache Spark

[Athena](athena.md) has two analytics engines. The SQL engine answers questions about data that already exists. The **Apache Spark** engine lets you write **PySpark code** in a notebook to read, transform, and write data, with serverless compute that you do not provision ([AWS — Use Apache Spark in Amazon Athena](https://docs.aws.amazon.com/athena/latest/ug/notebooks-spark.html)). This page covers the Spark engine. For schema-on-read, SQL pricing, and the SQL walkthrough, see the [Athena page](athena.md).

Apache Spark is an open-source distributed processing framework. It is not an AWS product, and it runs the same way on AWS, other clouds, and on-premises. What Athena adds is a **serverless place to run it**: no cluster to create, no instances to pick, and no scaling to manage.

## SQL Engine or Spark Engine?

The course puts the difference simply: use SQL to **ask questions** about your data, and use Spark to **work on** your data.

| Need                                                                                                                | Engine    |
| ------------------------------------------------------------------------------------------------------------------- | --------- |
| Filter, join, aggregate, or group data that is already in S3                                                        | **SQL**   |
| Ad hoc questions, pay only for data scanned                                                                         | **SQL**   |
| Multi-step pipelines: clean, reshape, enrich, then write the output somewhere else                                  | **Spark** |
| Logic that is easier to write as a program than as one SQL statement (loops, Python libraries, ML data preparation) | **Spark** |
| Interactive, iterative exploration in notebook cells                                                                | **Spark** |

SQL can also do some transformation, for example a [CTAS](athena.md#reducing-data-scanned) statement that rewrites CSV as Parquet. Spark becomes the better fit when the work needs **several steps** or **code**, which the course calls a *code-driven* workflow rather than a SQL-driven one.

## Architecture

![Athena for Apache Spark architecture: source data in Parquet, ORC, JSON, CSV, Iceberg, or Delta Lake format stored in S3; PySpark code in a notebook runs inside a Spark session on serverless DPUs, where the data is held as in-memory DataFrames; calculation results are stored in S3 by default and displayed in notebook cells](../assets/athena-spark-architecture.png)

The diagram reads from top to bottom:

1. **Source data sits in S3.** Spark reads structured and semi-structured formats such as Parquet, ORC, JSON, and CSV, plus open table formats such as **Apache Iceberg** and **Delta Lake** (Apache Hudi is supported too) ([AWS — Use non-Hive table formats](https://docs.aws.amazon.com/athena/latest/ug/notebooks-spark-table-formats.html)). As with SQL, Athena reads the source data and does not change it on its own.
2. **You write PySpark code in a notebook.** A notebook is a Python editor split into cells. You write code in a cell, run it, see the output, and write the next cell. It feels more like writing a program than running a query.
3. **The code runs in a Spark session.** When you run a cell, Athena starts a Spark session behind the scenes. The session runs on **DPUs** (data processing units), which are serverless compute that Athena allocates for you.
4. **The data lives in DataFrames.** Inside the session, your code loads data into **DataFrames**: table-like structures held **in memory**. The code can filter them, transform them, join them with other DataFrames, and build new DataFrames from them.
5. **Results go to two places.** Each cell run is called a **calculation**. Its result, standard output, and errors are written to the workgroup's results location in S3, and the result is also shown inline in the notebook cell ([AWS — Get started with Apache Spark on Athena](https://docs.aws.amazon.com/athena/latest/ug/notebooks-spark-getting-started.html)). The notebook is the interactive view, and S3 is the durable copy.

> [!warning] Gotcha: DataFrames disappear when the session ends
> DataFrames exist only while the Spark session is alive. When the session ends, they are gone. The calculation output in the results location is not the same as saving your transformed dataset. If you want to keep the data, your code must write it, for example as Parquet files to S3 or into an Iceberg table.

> [!warning] Gotcha: "source data is never changed" is about Athena, not your code
> The slide says the S3 source data is never changed, which is true in the sense that reading it into a DataFrame does not modify it. But Spark code can write anywhere its execution role allows. Code that writes back to the source prefix, or runs `MERGE` or `DELETE` on an Iceberg table, does change the data. Write output to a separate prefix or bucket, and give the role write access only there.

## Example: Cleaning the Order Files

This continues the e-commerce scenario from the [Athena SQL walkthrough](athena.md#real-world-examples-of-how-to-use-athena). The raw order exports in `s3://my-raw-bucket/orders_v2/` have hundreds of columns, some duplicate orders, and test orders that merchants created while setting up their stores. The finance team wants a clean copy in Parquet that all its later queries can use.

In an Athena Spark notebook, the `spark` session object is already created, so the first cell can read the files directly:

```python
orders = (spark.read
    .option("header", "true")
    .csv("s3://my-raw-bucket/orders_v2/"))
```

Spark reads every CSV file under the prefix into one DataFrame. Because the folders are named `year=2026/month=11/`, Spark also adds `year` and `month` as columns. The next cell cleans the data:

```python
from pyspark.sql import functions as F

clean = (orders
    .dropDuplicates(["order_id"])
    .filter(F.col("status") != "TEST")
    .withColumn("order_total", F.col("order_total").cast("double")))

clean.groupBy("status").count().show()
```

The `show()` output appears in the notebook cell, so we can check the result before saving anything. Nothing in S3 has changed yet: `orders` and `clean` exist only in the session's memory. The last cell writes the clean data out:

```python
(clean.write
    .mode("overwrite")
    .partitionBy("year", "month")
    .parquet("s3://my-curated-bucket/orders/"))
```

Now the clean data is stored in S3 as partitioned Parquet, in a different bucket from the raw files. A Glue crawler or a `CREATE EXTERNAL TABLE` statement can register it in the [Glue Data Catalog](glue.md#data-catalog-and-crawlers), and the SQL engine can query it cheaply from then on.

Instead of reading the path directly, the first cell could also read the crawler's table with `spark.table("raw.orders_v2")`. The course points out both options: Spark can use tables defined in the Glue Data Catalog, or load files from S3 directly. Loading from S3 is the simplest, and the catalog is useful when the table already exists there.

## Pricing

The Spark engine has one pricing model: **per DPU-hour of compute**, billed per second (about $0.35 per DPU-hour in US Regions). The session's **driver** (coordinator) DPUs are billed for the whole session, and the **worker** (executor) DPUs are billed while calculations run. The notebook itself is not charged ([AWS — Athena pricing](https://aws.amazon.com/athena/pricing/), [Athena FAQs](https://aws.amazon.com/athena/faqs/)).

Compared with the SQL engine:

|                                  | SQL engine       | Spark engine                                                                                                                                    |
| -------------------------------- | ---------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| Per TB scanned                   | Yes, the default | **No**                                                                                                                                          |
| Capacity Reservations (DPU-hour) | Yes              | **No**; Spark workgroups are not supported ([AWS — Capacity management](https://docs.aws.amazon.com/athena/latest/ug/capacity-management.html)) |
| Per DPU-hour of session compute  | No               | **Yes**, the only option                                                                                                                        |

The longer a session runs, the more DPU-hours it uses. An idle notebook with an open session still pays for the driver, so end sessions when you are done, or keep the session idle timeout short.

> [!warning] Gotcha: "serverless" does not mean "no settings"
> The course says you do not provision DPUs or pick instance types. You do not pick instance types, but you can size a session. In the PySpark engine version 3 console, the coordinator and executor sizes default to 1 DPU each, and you can change them along with the maximum number of DPUs a session may use. These settings decide how fast the work runs and how much it costs.

## Workgroups

As with SQL, every Spark session runs in an Athena [workgroup](athena.md#workgroups). The engine type is chosen when the workgroup is created and **cannot be changed later**: a SQL workgroup cannot become a Spark workgroup, or the other way around ([AWS — Get started with Apache Spark on Athena](https://docs.aws.amazon.com/athena/latest/ug/notebooks-spark-getting-started.html)). The workgroup also holds the calculation results location and the [IAM](iam.md) execution role that the Spark code uses to reach S3 and other services.

> [!tip] Exam Tip
> If a scenario needs both SQL querying and Spark notebooks in Athena, plan for **two workgroups**, one for each engine.

## Current State: Apache Spark 3.5 (Since November 2025)

The course describes the original Athena Spark experience, now called **PySpark engine version 3**, where notebooks open in the Athena console's notebook editor. In late 2025 AWS released a new version, **Apache Spark version 3.5** (Spark 3.5.6, based on Amazon EMR 7.12), which changes where and how you work ([AWS — Release notes](https://docs.aws.amazon.com/athena/latest/ug/release-notes.html), [AWS — Migrating to Apache Spark version 3.5](https://docs.aws.amazon.com/athena/latest/ug/notebooks-spark-migration-v3-to-v35.html)):

|                               | PySpark engine version 3 (course)  | Apache Spark version 3.5                                                                                                                                                                |
| ----------------------------- | ---------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Where notebooks run           | Athena console notebook editor     | **Amazon SageMaker Unified Studio** notebooks ([AWS — announcement](https://aws.amazon.com/about-aws/whats-new/2025/11/amazon-athena-apache-spark-sagemaker-notebooks/))                |
| Other clients                 | Athena calculation APIs            | **Spark Connect** from your own client, through the `GetSessionEndpoint` API ([AWS — Spark Connect support](https://docs.aws.amazon.com/athena/latest/ug/notebooks-spark-connect.html)) |
| Athena magics such as `%%sql` | Supported                          | Not supported; use `spark.sql(...)` and `df.show()` instead                                                                                                                             |
| Monitoring                    | Calculation details in the console | Adds the Spark UI and new debugging tools                                                                                                                                               |
| Lifecycle                     | **End of life on April 1, 2027**   | Current                                                                                                                                                                                 |

A workgroup cannot be upgraded in place from version 3 to version 3.5. You create a new Spark 3.5 workgroup and move the notebooks and permissions to it.

## Athena Spark vs Glue ETL vs EMR

All three can run Spark code over S3 data. The difference is how you use them.

| Need                                                                                       | Service                                                 |
| ------------------------------------------------------------------------------------------ | ------------------------------------------------------- |
| Interactive, exploratory Spark in notebooks; no cluster                                    | **Athena for Apache Spark**                             |
| Scheduled or event-driven production ETL jobs, crawlers, and job bookmarks                 | **[Glue](glue.md#jobs-sources-and-targets)** ETL jobs   |
| Full control of Spark, Hadoop, or other frameworks and their configuration, at large scale | **Amazon EMR** (on clusters, on EKS, or EMR Serverless) |
| SQL questions over S3 without writing code                                                 | **[Athena SQL](athena.md)**                             |

## Exam Quick Reference

| Concept           | Key point                                                                                                                       |
| ----------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| What it is        | Serverless Apache Spark in Athena: PySpark code in notebooks, no cluster provisioning                                           |
| When to choose it | Code-driven data engineering: ETL transforms, multi-step pipelines, ML data preparation                                         |
| Data              | Source data in S3 (Parquet, ORC, JSON, CSV, Iceberg, Delta Lake, Hudi); read through the Glue Data Catalog or directly from S3  |
| Processing        | Code runs in a Spark session on DPUs; data is held in in-memory DataFrames that disappear when the session ends                 |
| Output            | Calculation results stored in the workgroup's S3 location and shown in notebook cells; code must write data out to keep it      |
| Pricing           | Per DPU-hour only; no per-TB scanned pricing and no Capacity Reservations                                                       |
| Workgroup         | Engine type (SQL or Spark) is fixed at creation                                                                                 |
| Current version   | Apache Spark 3.5 runs through SageMaker Unified Studio notebooks and Spark Connect; PySpark engine version 3 ends April 1, 2027 |

> [!tip] Exam Tip
> The SAA exam mostly tests Athena SQL. If Athena Spark appears, the key phrases are *"PySpark"*, *"notebook"*, *"interactive Spark without managing clusters"*, and *"complex transformations that SQL cannot express"*. A scheduled production ETL pipeline still points to **Glue**, and *"query S3 with SQL"* still points to **Athena SQL**.

## References

- Adrian Cantrill, AWS SAA course — "Amazon Athena (Apache Spark)" lesson (shared associate-level lecture)
- [AWS — Use Apache Spark in Amazon Athena](https://docs.aws.amazon.com/athena/latest/ug/notebooks-spark.html)
- [AWS — Get started with Apache Spark on Amazon Athena](https://docs.aws.amazon.com/athena/latest/ug/notebooks-spark-getting-started.html)
- [AWS — Use non-Hive table formats in Athena for Spark](https://docs.aws.amazon.com/athena/latest/ug/notebooks-spark-table-formats.html)
- [AWS — Migrating to Amazon Athena Apache Spark version 3.5](https://docs.aws.amazon.com/athena/latest/ug/notebooks-spark-migration-v3-to-v35.html)
- [AWS — Spark Connect support](https://docs.aws.amazon.com/athena/latest/ug/notebooks-spark-connect.html)
- [AWS — Athena for Apache Spark in SageMaker notebooks (November 2025)](https://aws.amazon.com/about-aws/whats-new/2025/11/amazon-athena-apache-spark-sagemaker-notebooks/)
- [AWS — Athena release notes](https://docs.aws.amazon.com/athena/latest/ug/release-notes.html)
- [AWS — Capacity management](https://docs.aws.amazon.com/athena/latest/ug/capacity-management.html)
- [AWS — Athena pricing](https://aws.amazon.com/athena/pricing/)
- [AWS — Athena FAQs](https://aws.amazon.com/athena/faqs/)
