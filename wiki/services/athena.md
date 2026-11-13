---
title: Amazon Athena
type: service
tags: [athena, analytics, sql, serverless, s3, schema-on-read, glue-data-catalog, workgroups, capacity-reservations, federated-query, partitioning, parquet, logs]
certs: [aws-saa]
updated: 2026-11-12
---

# Amazon Athena

Athena is a **serverless, interactive analytics service**. It offers two engines: **SQL** (Trino/Presto-based) and **[Apache Spark](athena-spark.md)**. This page covers the SQL engine, which is the one the SAA exam tests. With it, you point SQL queries at data that already sits in [S3](s3.md). There is no cluster to size, no database to load, and no ETL pipeline you must build first. [AWS — What is Athena?](https://docs.aws.amazon.com/athena/latest/ug/what-is.html)

The idea in one sentence: **keep the raw data in S3, describe its shape with a table definition, and let Athena apply that shape while it reads.** This is called *schema-on-read*. It is why Athena suits ad hoc questions over data nobody wants to load into a database first.

## Two Analytics Engines

Both engines read source data from S3 and run serverless, but they are used differently. The SQL engine is for **asking questions** about data. The Spark engine is for **working on** data with code. [Athena for Apache Spark](athena-spark.md) covers the Spark engine in full.

|                  | SQL engine (this page)                          | Apache Spark engine                                              |
| ---------------- | ----------------------------------------------- | ---------------------------------------------------------------- |
| How you work     | SQL statements in the query editor              | PySpark code in notebook cells                                   |
| Typical use      | Ad hoc queries: filter, join, aggregate         | Multi-step data engineering: ETL transforms, ML data preparation |
| Pricing          | Per TB scanned, or Capacity Reservations        | Per DPU-hour of session compute only                             |
| Where results go | Query result location in S3, or managed storage | Calculation results in S3, and shown in notebook cells           |

Each [workgroup](#workgroups) is created for one engine, so using both means using two workgroups.

## When Athena Fits

| Situation                                                                                | Why Athena fits                                                                                                                                                                                                                                                                                                      |
| ---------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Ad hoc or occasional** SQL over data already in S3                                     | No infrastructure to run between queries, and no load or transform step before the first query                                                                                                                                                                                                                       |
| **Cost-conscious** analysis with sporadic use                                            | Per-query pricing has no base monthly charge; you pay for data scanned                                                                                                                                                                                                                                               |
| Querying **AWS service logs**                                                            | Documented table definitions exist for [CloudTrail](cloudtrail.md), [VPC Flow Logs](vpc-flow-logs.md), [ELB](elb.md) access logs, [CloudFront](cloudfront.md) logs, S3 access logs, and Cost and Usage Reports ([AWS service-log docs](https://docs.aws.amazon.com/athena/latest/ug/querying-aws-service-logs.html)) |
| Data already cataloged in the **[Glue Data Catalog](glue.md#data-catalog-and-crawlers)** | Athena reuses those table definitions directly                                                                                                                                                                                                                                                                       |
| Occasional queries that **join S3 data with another source**                             | [Federated query](#federated-query) connectors reach DynamoDB, RDS engines, CloudWatch Logs, and others                                                                                                                                                                                                              |

It is a weaker fit for constant, high-concurrency dashboards over heavily modeled data (consider [Redshift](redshift.md)), for millisecond lookups (use a database such as [DynamoDB](dynamodb.md)), and for heavy repeated transformation pipelines (use [Glue](glue.md) jobs or EMR, then query the output with Athena).

## Schema-on-Read Architecture

In a traditional database the table is the **final storage structure**. You create tables, often run an **ETL** (extract, transform, load) process to reshape the source data into them, and then load the rows before anyone can query. [Data Integration](../concepts/data-integration.md#etl-elt-and-streaming) describes that pipeline.

Athena reverses the order:

1. **Source data stays in S3, unchanged.** Athena reads it but never rewrites it. The original objects remain in their original format.
2. **Table definitions describe how to read it.** An Athena table holds no rows. It holds metadata: the S3 location, the columns and types, and the SerDe (serializer/deserializer) or format that turns bytes into rows. The course compares it to a recipe for turning ingredients into a meal. These definitions live in the **Glue Data Catalog**, which Athena uses as its metastore. [AWS — Tables, databases, and data catalogs](https://docs.aws.amazon.com/athena/latest/ug/understanding-tables-databases-and-the-data-catalog.html)
3. **The schema is applied at query time.** As Athena reads the objects, it projects them through the table definition, so standard SQL can treat semi-structured files as relational rows.
4. **Results are written out.** By default the result set goes to an S3 location, or to [Athena-managed storage](#query-results), and is shown in the console or retrieved by other tools and services.

![Athena architecture: source data in many formats (Ion, JSON, CSV/TSV, Avro, Parquet, ORC, Apache logs, CloudTrail, VPC Flow Logs) stored in an S3 bucket, read through a schema of table definitions held in the Glue Data Catalog, queried by Athena with SQL-like queries and billed on data consumed, with output written to S3 by default for other services to consume](../assets/athena-schema-on-read-architecture.png)

How to read the diagram: the top row lists source formats, all landing in one S3 bucket as the unchanged source data. The blue schema band with its tables sits between the bucket and Athena, because data passes through the table definitions only when it is read. The right-hand notes give the billing model (data consumed during the query) and the output (S3 by default, consumable by other services).

> [!warning] Gotcha: logs must reach S3 before Athena can read them
> Clarification on the diagram above: A standard Athena table can only point at an `s3://` location (other sources are reached through [federated query](#federated-query) instead). [ELB](elb.md) access logs go only to S3, and a [CloudTrail](cloudtrail.md) trail always writes to S3. [VPC Flow Logs](vpc-flow-logs.md#destinations-and-analysis) can go to CloudWatch Logs, S3, or Firehose, so choose S3 as the destination when you plan to query them with Athena. If the logs live only in CloudWatch Logs, use CloudWatch Logs Insights, or use Athena federated query with the CloudWatch Logs connector, which adds Lambda cost and gets no benefit from partitioning or Parquet ([AWS — Querying AWS service logs](https://docs.aws.amazon.com/athena/latest/ug/querying-aws-service-logs.html)).

For two hands-on ways to create the table definitions, writing SQL DDL in Athena or running a Glue crawler, see [Real-World Examples of How to Use Athena](#real-world-examples-of-how-to-use-athena) at the end of this page.

> [!tip] Exam Tip
> Pattern words for Athena: *"query data in S3 with SQL"*, *"no infrastructure"*, *"ad hoc"*, *"no need to load or transform the data"*, *"serverless"*, *"pay per query"*, *"analyze CloudTrail / VPC Flow Logs / ELB logs"*. Pattern words against Athena: *"must load into a data warehouse for complex BI with consistent sub-second dashboards"* (Redshift) or *"transform and clean before use"* (Glue ETL).

## Supported Data Formats

Athena reads structured, semi-structured, and text-based log data: **CSV/TSV**, **JSON**, **Amazon Ion**, **Avro**, **Parquet**, **ORC**, plus log formats such as **Apache web server logs**, CloudTrail, and VPC Flow Logs, which it parses through a matching SerDe (regex and Grok SerDes cover custom text logs). [AWS — Supported SerDes and data formats](https://docs.aws.amazon.com/athena/latest/ug/supported-serdes.html)

> [!warning] Gotcha: "unstructured" means parseable text
> The course says Athena handles *unstructured* data. In practice a table definition must be able to turn the bytes into rows and columns. Free-form text logs work through regex or Grok SerDes, but arbitrary binaries such as images or video are not queryable as columns. Read "unstructured" as "not already in a database schema", not as "any file at all".

## Pricing: Per-Query vs Capacity Reservations

The SQL engine has two pricing models. [AWS — Athena pricing](https://aws.amazon.com/athena/pricing/)

| Model                     | How you pay                                                                                                                                                                     | Choose it when                                                                              |
| ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| **Per query** (default)   | Per TB of data **scanned**, rounded up to the nearest MB, with a **10 MB minimum per query** (about $5 per TB in US Regions). DDL statements and failed queries are not charged | Sporadic or unpredictable use; you want no idle cost                                        |
| **Capacity Reservations** | A fixed rate per **DPU-hour** (data processing unit) for dedicated serverless capacity; no per-TB scan charge for queries that run on it                                        | You want predictable cost, guaranteed concurrency, or workload isolation and prioritization |

Since February 2026, Capacity Reservations can be as small as **4 DPUs** and as short as **1 minute**. Earlier course material and older blog posts describe the original **24-DPU, 1-hour** minimum. [AWS — 1-minute reservations and 4 DPU minimum](https://aws.amazon.com/about-aws/whats-new/2026/02/amazon-athena-one-minute-capacity-reservations/)

S3 storage and request charges, Glue Data Catalog charges, and federated-connector Lambda charges are billed separately from Athena itself. "No base cost" means no Athena charge between queries, not a free data lake.

### Reducing data scanned

Because per-query billing tracks bytes scanned, the main cost and performance levers are the ones that make Athena read less. [AWS — Performance tuning](https://docs.aws.amazon.com/athena/latest/ug/performance-tuning.html)

| Lever                                              | Effect                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| -------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Columnar formats** (Parquet, ORC)                | Athena reads only the columns the query references, instead of whole rows as with CSV or JSON                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| **Compression** (for example Snappy, GZIP, ZSTD)   | Fewer bytes on S3 means fewer bytes scanned                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| **Partitioning** (for example by `year/month/day`) | A `WHERE` clause on partition keys skips whole S3 prefixes; see the [Glue lease-record example](glue.md#worked-example--s3-lease-records-to-shared-catalog-consumers)                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| **CTAS** (`CREATE TABLE AS SELECT`)                | Rewrites query output as a new partitioned, compressed Parquet or ORC table, a common one-step conversion from raw CSV or JSON ([AWS — CTAS](https://docs.aws.amazon.com/athena/latest/ug/ctas.html)). For example:<pre>CREATE TABLE orders_optimized<br>WITH (<br>    format = 'PARQUET',<br>    parquet_compression = 'SNAPPY',<br>    external_location = 's3://my-optimized-bucket/orders/',<br>    partitioned_by = ARRAY['order_date'],<br>    bucketed_by = ARRAY['customer_id'],<br>    bucket_count = 16<br>) AS<br>SELECT<br>    order_id,<br>    customer_id,<br>    amount,<br>    status,<br>    order_date<br>FROM orders_raw;</pre> |
| **Query result reuse**                             | Returns a recent stored result for an identical query instead of scanning again ([AWS — Reuse query results](https://docs.aws.amazon.com/athena/latest/ug/reusing-query-results.html))                                                                                                                                                                                                                                                                                                                                                                                                                                                             |

> [!tip] Exam Tip
> *"Reduce Athena cost"* or *"improve Athena query performance"* almost always means **convert to Parquet or ORC, compress, and partition**. Choosing a larger instance is never the answer, because there is no instance.

## Workgroups

A **workgroup** is Athena's management unit: every query runs in one. Each account has a default workgroup named `primary`, configured for the SQL engine. [AWS — Workgroups](https://docs.aws.amazon.com/athena/latest/ug/workgroups-manage-queries-control-costs.html)

- **Engine choice.** A new workgroup is created for either SQL or [Apache Spark](athena-spark.md#workgroups), and this choice cannot be changed later. The engine *version* can be upgraded, but the engine *type* cannot ([AWS — Get started with Apache Spark on Athena](https://docs.aws.amazon.com/athena/latest/ug/notebooks-spark-getting-started.html)).
- **Isolation.** Separate teams, applications, or workloads, and control access with [IAM](iam.md) policies on the workgroup resource.
- **Cost controls.** A **per-query data usage limit** cancels any query that scans more than the threshold. **Per-workgroup** limits over a time window raise [CloudWatch](cloudwatch.md) alarms or SNS notifications. [AWS — Data usage controls](https://docs.aws.amazon.com/athena/latest/ug/workgroups-setting-control-limits-cloudwatch.html)
- **Result settings.** The query result location, encryption, and whether clients may override them.
- **Capacity.** A Capacity Reservation is assigned to one or more workgroups, whose queries then run on the reserved DPUs.

> [!tip] Exam Tip
> *"Separate query costs per team"*, *"stop a runaway query from scanning too much data"*, or *"enforce an encrypted result location"* all point to **Athena workgroups**.

## Query Results

The course says results go to S3 or managed storage. Both are current options, and a workgroup can use either one. [AWS — Managed query results](https://docs.aws.amazon.com/athena/latest/ug/managed-results.html)

| Option                                       | Behavior                                                                                                                                                                                        |
| -------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Customer-managed S3 location**             | Results and metadata are written to a bucket you own. You manage retention (for example with [S3 lifecycle rules](s3.md)), encryption, and access. Downstream tools can read the files directly |
| **Athena managed query results** (June 2025) | Athena stores, encrypts, and deletes the results after **24 hours**, at no extra storage cost. Access is governed per workgroup. Query result reuse is not supported in this mode               |

## Federated Query

By default Athena queries S3. **Federated query** extends it to non-S3 sources through **data source connectors**, so a single SQL statement can join S3 data with, for example, a DynamoDB table or an RDS database. Connectors run as [Lambda](lambda.md) functions in your account or, for newer Glue-connection-based connectors, through a Glue connection. Pre-built connectors cover CloudWatch Logs, DynamoDB, DocumentDB, MySQL, PostgreSQL, Oracle, SQL Server, Redshift, and many others. [AWS — Federated query](https://docs.aws.amazon.com/athena/latest/ug/federated-queries.html)

> [!warning] Gotcha: "Athena is S3-only" is outdated
> Athena's home is S3, but *"query data across S3 and a relational or NoSQL database without moving it"* is a valid Athena answer through federated query. Lambda-based connectors add Lambda cost and may spill large intermediate results to an S3 bucket. A common pattern is to run a federated **CTAS** that copies the external data into S3 as Parquet, then query the copy.

## Athena vs Neighboring Services

| Need                                                                              | Service                                                                                                                                        |
| --------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| Ad hoc SQL across **many** S3 objects, with joins and aggregations                | **Athena**                                                                                                                                     |
| Filter rows or columns inside **one** S3 object (legacy, closed to new customers) | [S3 Select](s3.md#server-side-filtering--s3-select--glacier-select)                                                                            |
| Discover schemas and **transform** data before analysis                           | [Glue](glue.md) crawlers and ETL jobs                                                                                                          |
| Persistent data warehouse for heavy, repeated BI queries                          | [Amazon Redshift](redshift.md); [Redshift Spectrum](redshift.md#querying-data-without-loading-it) queries S3 from an existing Redshift cluster |
| Large custom processing on Spark, Hadoop, or similar                              | Amazon EMR (or [Athena for Apache Spark](athena-spark.md) for interactive notebooks)                                                           |
| Interactive dashboards on top of Athena results                                   | Amazon QuickSight, using Athena as its data source                                                                                             |
| Operational log search and near-real-time analysis                                | [CloudWatch Logs Insights](cloudwatch.md) or OpenSearch                                                                                        |

> [!warning] Gotcha: Redshift Spectrum vs Athena
> Both query S3 in place with SQL. Spectrum **requires an existing Redshift cluster** and suits teams that already run Redshift and want to join warehouse tables with S3 data. If the scenario has **no warehouse** and wants **serverless, pay-per-query** access, choose Athena.

## Exam Quick Reference

| Concept               | Key point                                                                                               |
| --------------------- | ------------------------------------------------------------------------------------------------------- |
| What it is            | Serverless, interactive SQL (and Spark) analytics over data in S3                                       |
| Core idea             | **Schema-on-read**: table definitions are applied at query time; the source data in S3 is never changed |
| Table definitions     | Metadata in the Glue Data Catalog; Athena tables hold no rows                                           |
| Formats               | CSV/TSV, JSON, Ion, Avro, Parquet, ORC, Apache logs, AWS service logs                                   |
| Default pricing       | Per TB scanned, 10 MB minimum per query, no base cost                                                   |
| Predictable pricing   | Capacity Reservations per DPU-hour; 4 DPU and 1 minute minimum since February 2026                      |
| Cut cost or speed up  | Parquet or ORC, compression, partitioning, CTAS, result reuse                                           |
| Management unit       | Workgroup: engine choice, isolation, per-query and per-workgroup data limits, result location, capacity |
| Results               | Customer S3 location, or Athena managed storage for 24 hours                                            |
| Non-S3 sources        | Federated query with Lambda or Glue-connection connectors                                               |
| Classic exam scenario | Query CloudTrail, VPC Flow Logs, or ELB logs stored in S3 without building infrastructure               |

## Real-World Examples of How to Use Athena

Let's work through some real-world use cases about Athena. In this section, we will imagine the following scenario:

We are an e-commerce website builder. We have order data stored in AWS S3. The merchant storefronts on our platform export their orders to S3 as CSV files, and our finance team wants answers from those files, starting with revenue by order status. The data does not live in a database, and we would rather not build one just to answer a few questions. Athena lets us query the files where they already are.

Whatever the data looks like, every Athena query needs the same three things: the **raw data**, a **table definition** that tells Athena how to read it, and the **SQL** itself. Notice that the table definition is stored in the **[Glue Data Catalog](glue.md#data-catalog-and-crawlers)**, not inside Athena ([AWS — Tables, databases, and data catalogs](https://docs.aws.amazon.com/athena/latest/ug/understanding-tables-databases-and-the-data-catalog.html)). In this case, the raw data stays in S3 and the SQL stays simple.

### Stage 1: Simple orders, with the schema defined by hand

For now, let's assume that the orders structure is simple enough that we can easily define the schema manually. Each order is one line with five fields: an order ID, a customer ID, the amount, a status, and the order date. The files sit under `s3://my-raw-bucket/orders/` and look like this:

```csv
O-1001,C-17,42.50,SHIPPED,2026-11-01
O-1002,C-09,18.00,PENDING,2026-11-02
```

To tell Athena how to read these files, we run one statement in the Athena query editor:

```sql
CREATE EXTERNAL TABLE orders_raw (
    order_id STRING,
    customer_id STRING,
    amount DOUBLE,
    status STRING,
    order_date STRING
)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
LOCATION 's3://my-raw-bucket/orders/';
```

The statement has four parts. `EXTERNAL` tells Athena that the data lives outside it, so it registers metadata only; if we later run `DROP TABLE`, the definition disappears but the S3 objects stay. The column list is the schema Athena will apply on read, in the same order as the fields in each line. `ROW FORMAT DELIMITED FIELDS TERMINATED BY ','` says to split each line on commas, which Athena does with its default LazySimpleSerDe ([AWS — LazySimpleSerDe](https://docs.aws.amazon.com/athena/latest/ug/lazy-simple-serde.html)). Finally, `LOCATION` points at a **folder**, not a single file, so Athena reads every object beneath that prefix, including files that arrive later.

After that, the table definition lives in AWS Glue Data Catalog. Athena creates it in the catalog it calls `AwsDataCatalog`, inside the database selected in the query editor (`default` unless we choose another) ([AWS — Create tables in Athena](https://docs.aws.amazon.com/athena/latest/ug/creating-tables.html)). Creating the table takes seconds, costs nothing, and moves no data. No rows are loaded and no files are rewritten. This is schema-on-read in practice: `orders_raw` is only a recipe for reading the files, and any other service that uses the catalog can see it too.

Now finance can ask its question:

```sql
SELECT status, COUNT(*) AS orders, SUM(amount) AS revenue
FROM orders_raw
WHERE order_date >= '2026-11-01'
GROUP BY status;
```

Behind the scenes, Athena looks up `orders_raw` in the Data Catalog, lists the objects under `s3://my-raw-bucket/orders/`, reads each line through the column definitions, computes the totals, and writes the result to the workgroup's [result location](#query-results). We are billed for the bytes it read. Because this table has no partitions, that means the whole prefix every time, even though the query only asks about November.

`order_date` is declared as a `STRING`, so the comparison is textual. It works only because ISO dates (`YYYY-MM-DD`) sort correctly as text; if we later need date arithmetic, we should declare the column as `DATE` or use `CAST`.

> [!warning] Gotcha: header rows and quoted commas
> If the files begin with a header line such as `order_id,customer_id,...`, Athena reads it as a data row, and `amount` comes back `NULL` for it. Add `TBLPROPERTIES ('skip.header.line.count'='1')` to the statement. `FIELDS TERMINATED BY ','` also splits on every comma, including commas inside quoted values; use the OpenCSVSerDe for quoted CSV.

#### Adding partitions to `orders_raw`

The query above only asks about November, but Athena still reads every order file. As more orders arrive, every query reads more data and costs more. To avoid this, we partition the data: we store the files in one folder per month, and tell Athena which folder holds which month.

First, we save the files in Hive-style folders, where each folder name has the form `key=value`:

```text
s3://my-raw-bucket/orders/year=2026/month=10/orders-2026-10.csv
s3://my-raw-bucket/orders/year=2026/month=11/orders-2026-11.csv
```

Next, we add `PARTITIONED BY` to the table definition. Because `orders_raw` already exists, we first run `DROP TABLE orders_raw;`. This removes only the table definition; the files in S3 are not touched. Then we create the table again:

```sql
CREATE EXTERNAL TABLE orders_raw (
    order_id STRING,
    customer_id STRING,
    amount DOUBLE,
    status STRING,
    order_date STRING
)
PARTITIONED BY (year STRING, month STRING)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
LOCATION 's3://my-raw-bucket/orders/';
```

`year` and `month` are **partition keys**. They are not fields in the CSV files; Athena takes their values from the folder names. A partition key cannot have the same name as a column in the column list ([AWS — Partitioning data in Athena](https://docs.aws.amazon.com/athena/latest/ug/partitions.html)).

The new table starts with no partitions, and Athena only reads partitions that are registered in the Data Catalog. Until we register them, queries return no rows. Because our folders use the `key=value` naming, one command finds and registers them all:

```sql
MSCK REPAIR TABLE orders_raw;
```

If the folders do not follow this naming, for example `orders/2026/11/`, we register each partition explicitly and give its location:

```sql
ALTER TABLE orders_raw ADD PARTITION (year = '2026', month = '11')
LOCATION 's3://my-raw-bucket/orders/2026/11/';
```

Now finance can filter on the partition keys:

```sql
SELECT status, COUNT(*) AS orders, SUM(amount) AS revenue
FROM orders_raw
WHERE year = '2026' AND month = '11'
GROUP BY status;
```

Athena now reads only the `month=11` folder instead of every file. This is called partition pruning (see [Reducing data scanned](#reducing-data-scanned)). A filter on `order_date` alone does not do this: `order_date` is a column inside the files, so Athena has to open the files to check it. Only a filter on the partition keys lets Athena skip whole folders.

The downside is that we now maintain the partitions ourselves. Each new month folder must be registered before Athena can see it, which we come back to in Stage 2.

### Stage 2: Orders grow to hundreds of columns, so a crawler learns the schema

> See also: [A Demo on how to setup a AWS Glue Crawler](https://www.youtube.com/watch?v=gfUNDP0QiEQ)

As our platform iterates, the order data becomes much more complex. Merchants ask for line-item detail, payment and tax breakdowns, shipping and fulfillment status, discount codes, and marketing attribution, and each new feature adds columns to the export.

Before long, each file has hundreds of columns, and a few more appear with most releases. Order volume grows too, so the platform team starts writing the new export to a fresh prefix, one folder per month, with a header row naming every column:

```text
s3://my-raw-bucket/orders_v2/year=2026/month=10/part-0001.csv
s3://my-raw-bucket/orders_v2/year=2026/month=11/part-0001.csv

order_id,customer_id,order_total,currency,status,payment_method,shipping_country,discount_code, ... (hundreds more)
```

At this size, writing `CREATE EXTERNAL TABLE` by hand stops being practical. Listing every column is slow and error-prone, and the list would be out of date after the next release. So instead of manually listing all the columns, we use an **[AWS Glue crawler](glue.md#data-catalog-and-crawlers)** to learn the schema by crawling the raw CSV files in S3. The crawler writes the table definition into the same Data Catalog, and Athena queries it exactly as before ([AWS — Use a crawler to add a table](https://docs.aws.amazon.com/athena/latest/ug/schema-crawlers.html)). The trade-off is that the crawler is not free: it is billed per DPU-hour while it runs ([Glue pricing](https://aws.amazon.com/glue/pricing/)).

Setting it up takes a few steps:

1. **We keep one dataset per prefix.** Every file under `orders_v2/` shares the same format and compatible columns. Mixing different datasets in one folder is the most common reason a crawler produces several tables instead of one.
2. **We create a catalog database** called `raw`, either in the Glue console or with `CREATE DATABASE raw;` in Athena. It becomes the namespace Athena shows in its database list.
3. **We give the crawler permission to read the data.** The crawler runs with an [IAM role](iam.md#iam-roles) that Glue can assume. We attach the `AWSGlueServiceRole` managed policy and grant `s3:GetObject` and `s3:ListBucket` on the source prefix, plus `kms:Decrypt` if the objects use SSE-KMS ([AWS — Crawler IAM role](https://docs.aws.amazon.com/glue/latest/dg/create-an-iam-role.html)).
4. **We create the crawler.** We point it at `s3://my-raw-bucket/orders_v2/`, choose the role and the target database `raw`, and set a schedule (on demand or cron). In the output options we enable **Create a single schema for each S3 path**, so that compatible files are combined into one table ([AWS — Grouping behavior](https://docs.aws.amazon.com/glue/latest/dg/crawler-grouping-policy.html)).
5. **We run it.** The crawler's classifiers sample the files and recognize CSV, take the column names from the header row, infer a type for each column, and read the partition keys `year` and `month` from the folder names. The crawler then registers a table named after the folder, `raw.orders_v2`, together with its partitions ([AWS — Classifiers](https://docs.aws.amazon.com/glue/latest/dg/add-classifier.html)).
6. **We review the result** in the Glue console before anyone relies on it. Inference is a good guess, not a guarantee, so we check that IDs were not typed as numbers, that dates look right, and that the header row was recognized.

Now the finance team can query hundreds of columns without anyone having typed them. In Athena we choose the `AwsDataCatalog` data source and the `raw` database, and `orders_v2` is already in the table list:

```sql
SELECT payment_method, COUNT(*) AS orders, SUM(order_total) AS revenue
FROM raw.orders_v2
WHERE year = '2026' AND month = '11'
GROUP BY payment_method;
```

Because the filter is on the partition keys, Athena reads only the `month=11` folder instead of every file. This is the same partition pruning as in Stage 1, but this time the crawler found the partition keys and registered the partitions for us.

Athena never ran the crawler, and it does not care who wrote the table. To Athena, `raw.orders_v2` looks exactly like `orders_raw`: a catalog entry that points at an S3 prefix. The crawler handled discovery, the Data Catalog stored the result, and Athena only queried. This is the same division of labor described in [Data Integration](../concepts/data-integration.md).

When December's folder `month=12/` arrives, Athena does not see it yet. Athena only reads partitions that are registered in the Data Catalog, so queries return no December rows (without an error) until the new partition is registered ([AWS — Partitioning data in Athena](https://docs.aws.amazon.com/athena/latest/ug/partitions.html)). New files added to an existing partition, such as another file in `month=11/`, are read automatically. There are two common ways to run the crawler again automatically:

- **On a schedule.** Give the crawler a cron expression so that it scans S3 periodically, for example `cron(0 1 * * ? *)` to run every day at 01:00 UTC. Schedules use UTC and can run at most every 5 minutes. Setting the crawler to crawl new folders only keeps each run short ([AWS — Crawler schedules](https://docs.aws.amazon.com/glue/latest/dg/monitor-data-warehouse-schedule.html)).
- **When a new file is uploaded.** Turn on Amazon EventBridge notifications for the bucket, and create an [EventBridge](eventbridge.md) rule that matches `Object Created` events under `orders_v2/`. The rule invokes a [Lambda](lambda.md) function that calls the Glue `StartCrawler` API, so the crawler runs whenever a new file arrives. The function's role needs the `glue:StartCrawler` permission ([AWS — S3 events in EventBridge](https://docs.aws.amazon.com/AmazonS3/latest/userguide/EventBridge.html)).

Without the crawler, we can also run `MSCK REPAIR TABLE raw.orders_v2;` in Athena, or set up partition projection.

**Partition projection** replaces a lookup with a calculation. Without it, Athena looks up the partition list in the Glue Data Catalog. With it, Athena computes the partition locations from rules in the table properties ([AWS — Partition projection](https://docs.aws.amazon.com/athena/latest/ug/partition-projection.html)). The rule has three parts:

1. **Input: the partition columns.** The keys declared in `PARTITIONED BY`, such as `year` and `month`, are not stored inside the files. They are the inputs to the rule.
2. **Mapping rule: the table properties.** `TBLPROPERTIES` gives each partition column a projection type (`enum`, `integer`, `date`, or `injected`) and its allowed values. An optional `storage.location.template` describes the S3 path, for example `s3://my-raw-bucket/orders/${year}/${month}/` for folders like `orders/2026/11/`. Without a template, Athena assumes the Hive-style `key=value` layout ([AWS — Supported types for partition projection](https://docs.aws.amazon.com/athena/latest/ug/partition-projection-supported-types.html)).
3. **Output: the S3 prefixes to read.** Athena puts the values from the `WHERE` clause into the template, works out the matching S3 prefixes, and lists and reads only the files under them.

For example, suppose the Stage 1 order files sit in plain `year/month` folders such as `s3://my-raw-bucket/orders/2026/11/`. A table with one projected partition column, `order_month`, covers every month without any partition registration:

```sql
CREATE EXTERNAL TABLE orders_projected (
    order_id STRING,
    customer_id STRING,
    amount DOUBLE,
    status STRING,
    order_date STRING
)
PARTITIONED BY (order_month STRING)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
LOCATION 's3://my-raw-bucket/orders/'
TBLPROPERTIES (
    'projection.enabled' = 'true',
    'projection.order_month.type' = 'date',
    'projection.order_month.format' = 'yyyy/MM',
    'projection.order_month.range' = '2024/01,NOW',
    'projection.order_month.interval' = '1',
    'projection.order_month.interval.unit' = 'MONTHS',
    'storage.location.template' = 's3://my-raw-bucket/orders/${order_month}/'
);
```

Each property maps to one part of the rule:

- `projection.enabled` turns projection on for the table.
- `projection.order_month.type` = `date` says the column holds dates, and `format` = `yyyy/MM` says how those dates are written, both in queries and in the S3 path.
- `range` gives the first and last values Athena may generate. `NOW` is evaluated in UTC at query time, so each new month becomes queryable as soon as its folder exists, with no `MSCK REPAIR TABLE` or crawler run.
- `interval` and `interval.unit` set the step between values, here one month.
- `storage.location.template` places each value into the path. The template must end with `/` and contain a placeholder for every partition column ([AWS — Setting up partition projection](https://docs.aws.amazon.com/athena/latest/ug/partition-projection-setting-up.html)).

A query such as `WHERE order_month = '2026/11'` now becomes the single prefix `s3://my-raw-bucket/orders/2026/11/`. If a computed prefix holds no files, Athena returns no rows for it rather than an error.

If the query filters on a range (`BETWEEN`, `>=`) or does not filter on the partition columns at all, Athena enumerates every value the rule allows and maps each one to a path. `WHERE order_month BETWEEN '2026/01' AND '2026/06'`, for example, becomes six prefixes. For an `integer` column, the values come from its required `range` and its optional `interval` (default 1). For a `date` column, they come from `range` and `interval`; `interval` and `interval.unit` can be left out only at day or month precision, where the step defaults to one day or one month. For an `enum` column, they are the listed values. An `injected` column has no range, so every query must give its exact value in the `WHERE` clause.

Because our month folders are predictable, partition projection removes the registration step completely. The [Glue worked example](glue.md#4-keep-new-partitions-discoverable) compares these options.

> [!warning] Gotcha: projection skips the partition list, not the catalog
> Athena still reads the table definition (columns, SerDe, location) from the Glue Data Catalog. Only the partition lookup is replaced. Projected partitions are never registered in the catalog, so other engines that read it, such as Glue ETL jobs, EMR, or Redshift Spectrum, do not see them. When projection is enabled, Athena also ignores any partitions that are registered for that table.

> [!warning] Gotcha: triggering the crawler on every upload
> Every uploaded file creates its own event, but a crawler cannot start while it is already running: `StartCrawler` fails with `CrawlerRunningException`. The Lambda function should catch this error, because the running crawl or the next one will pick up the new files. A new file in an existing folder, such as another file in `month=11/`, does not need a crawl at all unless the schema changed. Only a new folder (partition) does. If uploads are frequent, a schedule is usually simpler. The crawler can also use S3 event notifications itself (**crawl based on events**), so that each scheduled run crawls only the folders that changed ([AWS — Crawls using S3 event notifications](https://docs.aws.amazon.com/glue/latest/dg/crawler-s3-event-notifications.html)).

> [!warning] Gotcha: crawler inference is a starting point
> The built-in CSV classifier detects a header only when that row looks different from the data rows. If every column is a string, the header can be missed, and you get columns named `col0`, `col1`, and so on, plus the header as a data row. Fix this with a custom CSV classifier set to `ContainsHeader = PRESENT`. Inconsistent layouts or incompatible schemas can still produce several tables, so verify the result with a sample query. Re-crawls can also update or log schema changes according to the crawler's **schema change policy** ([AWS — Crawler configuration](https://docs.aws.amazon.com/glue/latest/dg/crawler-configuration.html)).

### What the story shows

In both stages, the raw data stayed in S3 and the SQL stayed simple. The key points are:

1. **An Athena table is metadata, not storage.** Creating it moves no data, and dropping an external table deletes no data.
2. **Athena reads its table definitions from the Glue Data Catalog.** Whether we wrote the DDL or a crawler did, the definition is stored in AWS Glue Data Catalog.
3. **The way we write the schema follows the data.** Hand-written DDL suits a small, known schema. A crawler suits a wide or evolving one.
4. **Data layout drives cost more than SQL does.** Partitioning, and columnar formats such as Parquet, decide how many bytes Athena reads, and that is what we pay for.

> [!tip] Exam Tip
> *"Many columns"*, *"unknown or changing schema"*, or *"automatically discover the schema of data in S3"* points to a **Glue crawler** that populates the Data Catalog, then **Athena** to query it. *"Quickest way to query a small, known CSV"* points to `CREATE EXTERNAL TABLE` in Athena. Either way, the table lives in the Glue Data Catalog and the data stays in S3.

## References

- Adrian Cantrill, AWS SAA course — "Amazon Athena (SQL)" lesson (shared associate-level lecture)
- [Amazon Athena for Apache Spark](athena-spark.md) — the Spark engine page
- [AWS — What is Amazon Athena?](https://docs.aws.amazon.com/athena/latest/ug/what-is.html)
- [AWS — Athena pricing](https://aws.amazon.com/athena/pricing/)
- [AWS — Capacity management](https://docs.aws.amazon.com/athena/latest/ug/capacity-management.html)
- [AWS — Querying AWS service logs](https://docs.aws.amazon.com/athena/latest/ug/querying-aws-service-logs.html)
- [AWS — Performance tuning](https://docs.aws.amazon.com/athena/latest/ug/performance-tuning.html)
- [AWS — Workgroups](https://docs.aws.amazon.com/athena/latest/ug/workgroups-manage-queries-control-costs.html)
- [AWS — Managed query results](https://docs.aws.amazon.com/athena/latest/ug/managed-results.html)
- [AWS — Federated query](https://docs.aws.amazon.com/athena/latest/ug/federated-queries.html)
- [AWS — Query VPC flow logs](https://docs.aws.amazon.com/athena/latest/ug/vpc-flow-logs.html)
