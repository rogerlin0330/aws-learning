---
title: Amazon Redshift
type: service
tags: [redshift, analytics, data-warehouse, olap, oltp, columnar, mpp, provisioned, serverless, leader-node, compute-node, slices, rpu, namespace, workgroup, multi-az, snapshots, spectrum, federated-query, copy, unload, ra3, rg]
certs: [aws-saa]
updated: 2026-11-13
---

# Amazon Redshift

Redshift is a **petabyte-scale, massively parallel processing (MPP) data warehouse**. A data warehouse is the place where many operational databases across a business send their data for **long-term reporting and analytics**. It is not where an application stores data it reads and writes while serving users. MPP means Redshift splits both the data and the query work across many compute resources that run in parallel, which is what lets it handle petabytes. [AWS — What is Amazon Redshift?](https://docs.aws.amazon.com/redshift/latest/mgmt/welcome.html)

The course frames Redshift around a few questions the exam asks: is the workload OLTP or OLAP, should it run provisioned or serverless, how does the cluster spread work, how is data protected, and how does data get in.

## OLTP vs OLAP

Redshift is an **OLAP** (online analytical processing) platform. [RDS](rds.md) and [Aurora](aurora.md) are **OLTP** (online transaction processing) platforms. The difference is what each one is built to do.

|                    | OLTP (RDS, Aurora)                                  | OLAP (Redshift)                                                     |
| ------------------ | --------------------------------------------------- | ------------------------------------------------------------------- |
| Purpose            | Capture and process transactions in real time       | Complex queries and analytics over aggregated historical data       |
| Typical operations | Inserts, updates, and deletes of individual records | Large reads that scan, join, and aggregate many records             |
| Example            | A web store adds an order to a cart                 | The business asks which products were most profitable last year     |
| Data sources       | The application itself                              | Many OLTP systems (sales, manufacturing, marketing) loaded together |
| Storage layout     | Row-based                                           | Column-based                                                        |

The course example is a retailer with separate OLTP databases for manufacturing, sales, and marketing. Loading all three into Redshift lets analysts ask which products were most successful and most cost-effective over the year, using information from manufacturing through to marketing.

Redshift still supports writes to its own warehouse tables: `COPY` loads data, and SQL operations such as `INSERT`, `UPDATE`, and `DELETE` change stored records. Loading a copy from RDS or S3 does not change that original source. These persistent warehouse writes are distinct from temporary intermediate results during query processing; the Multi-AZ description below includes warehouse writes. [AWS — INSERT](https://docs.aws.amazon.com/redshift/latest/dg/r_INSERT_30.html)

### Why column storage suits analytics

In a row-based orders table, each row is one order, holding the order ID, customer, product, price, date, and so on. Rows are created and updated one at a time, which suits transactions. But a report such as "how much did we make on one product last year" only needs the product, price, and date. A full table scan in a row store reads blocks containing the other columns too; a suitable covering index can avoid reading the full rows.

Redshift stores data **by column**: all products together, all prices together, all dates together. A query reads only the columns it references, so reporting queries read far less data. Columns of one type also compress well, which reduces I/O further. [AWS — Columnar storage](https://docs.aws.amazon.com/redshift/latest/dg/c_columnar_storage_disk_mem_mgmnt.html)

> [!tip] Exam Tip
> *"Data warehouse"*, *"OLAP"*, *"analytics across data from many databases"*, *"BI and reporting over petabytes"*, or *"columnar"* point to **Redshift**. *"Transactions"*, *"orders"*, or *"application database"* point to RDS, Aurora, or DynamoDB. Redshift is not an application's operational database.

## Two Deployment Modes

Redshift runs as **provisioned** clusters or as **Redshift Serverless**. Both use the same Redshift engine and the same SQL, and the data flow in and out is the same. What differs is who manages capacity and how you pay.

### Provisioned

You choose the **node type** and the **number of nodes**, and the cluster runs until you pause or delete it. [AWS — Provisioned clusters](https://docs.aws.amazon.com/redshift/latest/mgmt/working-with-clusters.html)

- **Leader node.** Handles all communication with client programs and with the compute nodes. It parses queries, builds execution plans, hands the work to the compute nodes, and aggregates their results. On a multi-node cluster you **pay only for compute nodes**; the leader node is not billed. On a single-node cluster, one node does both jobs.
- **Compute nodes.** Run the query work the leader node assigns.
- **Slices.** Each compute node is partitioned into **slices**. Each slice gets a portion of the node's memory and disk and processes a portion of the workload. The leader node distributes data and work across all slices, and the slices run in parallel. This is the MPP part of Redshift. The course says a node has 2, 4, 8, or 16 slices depending on its size; current AWS node tables list values such as 2 for `ra3.xlplus`, 4 for `ra3.4xlarge`, and 16 for `ra3.16xlarge`.
- **Networking.** Nodes in a cluster use high-speed networking between them.
- **Single-AZ by default.** A default cluster runs in one Availability Zone, so it is not highly available across AZs. For that, **Multi-AZ** is optional on **RA3 and RG** node types. It places an equal number of compute nodes in two AZs behind **a single endpoint**, and all nodes serve reads and writes during normal operation (there is no idle standby). Each individual query runs on the compute in one AZ only. If an AZ fails, Redshift continues in the remaining AZ with **zero data loss (RPO)**, but open connections may drop and queries running in the failed AZ must be retried. AWS says Multi-AZ reduces recovery time but publishes no RTO figure. The "under 60 seconds" figure comes from AWS's pre-launch tests in the [launch blog](https://aws.amazon.com/blogs/big-data/enable-multi-az-deployments-for-your-amazon-redshift-data-warehouse/), not from a commitment. The SLA rises from 99.9% to **99.99%**. The subnet group must cover at least three AZs, Multi-AZ clusters must be encrypted, and converting a Single-AZ cluster can double the node count, with nodes in both AZs billed at the normal per-node rate. [AWS — Multi-AZ deployment](https://docs.aws.amazon.com/redshift/latest/mgmt/managing-cluster-multi-az.html), [AWS — Setting up Multi-AZ](https://docs.aws.amazon.com/redshift/latest/mgmt/overview-multi-az.html)
- **Billing.** You pay **per node-hour while the cluster runs**, whether anyone is querying it or not. **Reserved nodes** lower the rate for steady use ([Cost Optimization](../patterns/cost-optimization.md)).
- **Pause.** Pausing a cluster stops compute billing. The course says you then pay only for backup storage; AWS states more precisely that on-demand billing is suspended and only the cluster's storage incurs charges, which for RA3 and RG includes managed storage as well as backups. [AWS — Pausing and resuming a cluster](https://docs.aws.amazon.com/redshift/latest/mgmt/rs-mgmt-pause-resume-cluster.html)

> [!warning] Gotcha: Multi-AZ clusters cannot be paused
> A Multi-AZ deployment can't be paused or resumed, can't be single-node, and can't be relocated to another AZ. Multi-AZ also protects against an AZ failure only, not a Region failure; for that, copy snapshots to another Region ([AWS — Multi-AZ setup](https://docs.aws.amazon.com/redshift/latest/mgmt/overview-multi-az.html)).

#### Node types today

The course names only RG and RA3. For context:

| Node type | Status                                                                                                                                                                                                                                                                           |
| --------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **RG**    | Graviton-based provisioned nodes, generally available since **12 May 2026**; AWS's recommended choice for new clusters and for upgrades from DC2 or RA3 ([AWS — RG launch](https://aws.amazon.com/about-aws/whats-new/2026/05/amazon-redshift-rg-instances-powered-by-graviton)) |
| **RA3**   | Separates compute from **Redshift Managed Storage** (RMS), which is backed by S3, so storage scales independently of nodes ([AWS — RA3 node type](https://repost.aws/knowledge-center/redshift-ra3-node-type))                                                                   |
| **DC2**   | Older dense-compute nodes with local SSD storage; deprecated in April 2025 in favor of RA3, RG, or Serverless ([AWS — DC2 migration](https://aws.amazon.com/blogs/big-data/amazon-redshift-dc2-migration-approach-with-a-customer-case-study))                                   |

### Serverless

There are no nodes to choose. Capacity is measured in **RPUs** (Redshift Processing Units). You set a **base capacity**, Redshift scales automatically with the workload, and you can set a **maximum** so costs stay bounded. Base capacity currently ranges from **4 to 1024 RPUs**; the 4-RPU minimum arrived in 2025. Available capacity limits depend on Region; the 1024-RPU base maximum is supported only in selected Regions. [AWS — Serverless considerations](https://docs.aws.amazon.com/redshift/latest/mgmt/serverless-usage-considerations.html)

Serverless splits what a cluster does into two resources:

| Resource      | Holds                                                                             |
| ------------- | --------------------------------------------------------------------------------- |
| **Namespace** | Data objects and users: databases, schemas, tables, users, and snapshots          |
| **Workgroup** | Compute: the RPUs, network settings, and the **endpoint** applications connect to |

- **VPC placement.** Serverless still runs in your VPC. A workgroup needs **at least two subnets in two AZs**, or three subnets in three AZs if **Enhanced VPC Routing** is enabled (the 2-AZ option arrived in July 2025). This is a **placement requirement**, not the provisioned Multi-AZ feature.
- **Billing.** You pay **per RPU-second** for compute actually used, with a 60-second minimum charge, and **no compute charge while the warehouse is idle**. Managed storage remains billable while idle. There is nothing to pause. [AWS — Serverless billing](https://docs.aws.amazon.com/redshift/latest/mgmt/serverless-billing-on-demand.html)

### Choosing a mode

| Choose          | When                                                                                                                            |
| --------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| **Provisioned** | Steady, heavy, predictable workloads, where reserved nodes make it the cheapest option; or when you need Multi-AZ for a cluster |
| **Serverless**  | Variable or intermittent workloads, or when you don't want to manage capacity                                                   |

> [!tip] Exam Tip
> *"Unpredictable"*, *"intermittent"*, *"no capacity management"*, or *"pay only when queries run"* point to **Redshift Serverless**. *"Steady 24/7 reporting"* plus *"lowest cost"* points to **provisioned with reserved nodes**. A provisioned cluster that sits idle overnight can be **paused** to stop compute charges.

## Querying Data Without Loading It

Data is normally loaded into Redshift before it is queried. Two features avoid that step. Both still need a Redshift warehouse (a provisioned cluster or a serverless workgroup).

- **Redshift Spectrum** queries data in [S3](s3.md) directly through external tables, so large datasets don't have to be loaded first. External table definitions typically live in the [Glue Data Catalog](glue.md#data-catalog-and-crawlers), and Spectrum can join that S3 data with tables inside the warehouse. [AWS — Redshift Spectrum](https://docs.aws.amazon.com/redshift/latest/dg/c-using-spectrum.html)
- **Federated query** queries live data in remote databases, specifically RDS and Aurora **PostgreSQL** and **MySQL**, directly from Redshift. [AWS — Federated queries](https://docs.aws.amazon.com/redshift/latest/dg/federated-overview.html)

> [!warning] Gotcha: Spectrum vs Athena
> Both query S3 with SQL without loading. Spectrum **requires a Redshift warehouse** and fits teams already running Redshift who want to join warehouse tables with S3 data. With no warehouse and a need for serverless, pay-per-query SQL over S3, choose [Athena](athena.md).

## Access, Security, and Monitoring

- **SQL access over JDBC or ODBC.** Any SQL tool or application that supports either standard can connect, for example SQL Workbench/J. AWS services also integrate natively, such as **Amazon Quick Sight** for dashboards (formerly Amazon QuickSight, now part of Amazon Quick).
- **Private by default.** Redshift runs inside a [VPC](vpc.md). Since January 2025, new provisioned clusters have **public access disabled** by default and require SSL. [AWS — Default security changes](https://aws.amazon.com/about-aws/whats-new/2025/01/amazon-redshift-default-security-configurations-new-warehouses)
- **Permissions.** [IAM](iam.md) controls AWS API access and authorizes access to AWS resources such as S3. Inside the database, users and database roles need SQL privileges on schemas and tables, managed with `GRANT` and `REVOKE`; IAM access alone does not grant table access. [AWS — GRANT](https://docs.aws.amazon.com/redshift/latest/dg/r_GRANT.html)
- **Encryption at rest.** New data warehouses are encrypted by default. If you specify no [KMS](kms.md) key, AWS uses an AWS-owned key; you can choose your own KMS key instead.
- **Monitoring** through [CloudWatch](cloudwatch.md).

## Architecture

![Amazon Redshift architecture: a provisioned Redshift cluster inside a private subnet of a VPC. A leader node connects to applications, SQL tools such as SQL Workbench/J, and visualization suites over JDBC or ODBC. Compute nodes are divided into slices that each hold a range of the data (A-D through X-Z), and data is mirrored to another node. S3 is used for LOAD and UNLOAD and for automatic snapshots (every 8 hours or 5 GB) and manual snapshots, which can be copied to another AWS Region. Data enters through COPY from DynamoDB, Firehose streaming, DMS migration, and Glue transform-and-load](../assets/redshift-architecture.png)

How to read the diagram:

1. **The cluster sits in one subnet in one AZ.** The course draws a provisioned cluster. A serverless workgroup would span at least two subnets in two AZs instead, but data flows in and out the same way.
2. **Everything connects to the leader node** (right side). SQL tools such as SQL Workbench/J, applications, and visualization suites connect over JDBC or ODBC, and AWS services such as Quick Sight connect natively. With Serverless, clients connect to the workgroup endpoint instead.
3. **Compute nodes are divided into slices** (center). The leader node spreads data across slices, shown as ranges A-D through X-Z, and each slice works on its share in parallel.
4. **Data protection** (bottom). Data is mirrored to another node, automatic snapshots go to S3, and snapshots can be copied to another Region. See [Data Protection](#data-protection).
5. **Data in and out** (left and top right). S3 `COPY` and `UNLOAD`, `COPY` from DynamoDB, Firehose, DMS, and Glue. See [Getting Data In and Out](#getting-data-in-and-out).

## Data Protection

Because a default provisioned cluster lives in one AZ, Redshift protects data in several layers.

- **Within the cluster.** The course says that when a cluster has two or more compute nodes, the data on each node is mirrored to another node, which reduces the risk of data loss if a node fails. That describes clusters whose data lives on node storage (the older DC2 model). On RA3 and RG, the durable copy of the data is in **Redshift Managed Storage**, backed by S3, and the nodes cache it locally. A single-node cluster has no other node to mirror to.
- **Automatic snapshots to S3.** Taken about **every 8 hours or after every 5 GB per node of data changes**, whichever comes first. They are incremental and deleted at the end of their **retention period**: 1 day by default, configurable from 1 to 35 days. Automatic snapshots can't be disabled on RA3 and RG. Because S3 stores data across AZs, the snapshots survive the failure of the cluster's AZ. [AWS — Snapshots and backups](https://docs.aws.amazon.com/redshift/latest/mgmt/working-with-snapshots.html)
- **Manual snapshots to S3.** Taken by an administrator and retained indefinitely by default; you can set a retention period so they expire automatically. Managing their retention is the administrator's job.
- **Cross-Region snapshot copy.** Redshift can be configured to copy snapshots automatically to another AWS Region. This gives global resilience and a way to bring up a cluster in another Region after a disaster.
- **Restore** creates a new cluster from a snapshot.

Serverless works the same way, except its automatic backups are called **recovery points**. They are taken about every 30 minutes or after every 5 GB of data changes per node, and kept for **24 hours**. To keep one longer, convert it to a snapshot. [AWS — Serverless snapshots and recovery points](https://docs.aws.amazon.com/redshift/latest/mgmt/serverless-snapshots-recovery-points.html)

> [!tip] Exam Tip
> *"Redshift disaster recovery in another Region"* means **cross-Region snapshot copy**. *"Survive an AZ failure"* for a provisioned cluster means **Multi-AZ** (RA3 or RG). A default cluster is single-AZ.

## Getting Data In and Out

| Path                                                                      | What it does                                                                                                                                          |
| ------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| **`COPY` from S3**                                                        | The main bulk-load path: loads files from S3 in parallel across the slices ([AWS — COPY](https://docs.aws.amazon.com/redshift/latest/dg/r_COPY.html)) |
| **`UNLOAD` to S3**                                                        | Exports query results to S3 files, for example as Parquet                                                                                             |
| **`COPY` from DynamoDB**                                                  | Loads a [DynamoDB](dynamodb.md) table into Redshift                                                                                                   |
| **[Firehose](kinesis.md#kinesis-data-firehose-now-amazon-data-firehose)** | Streams data into Redshift. Mechanically, Firehose writes to S3 first and then issues a `COPY`                                                        |
| **[DMS](dms.md)**                                                         | Migrates data from databases into Redshift, including ongoing changes                                                                                 |
| **[Glue](glue.md)**                                                       | Transforms data and loads it into Redshift                                                                                                            |

Beyond the course, Redshift can also ingest directly from **Kinesis Data Streams** and **Amazon MSK** with **streaming ingestion** (no S3 staging), and **zero-ETL integrations** replicate data from sources such as Aurora, RDS for MySQL, and DynamoDB without building a pipeline. [AWS — Redshift data ingestion options](https://aws.amazon.com/blogs/big-data/amazon-redshift-data-ingestion-options)

[Data Integration](../concepts/data-integration.md) covers how these load, transform, and query roles fit together.

## Enhanced VPC Routing

By default, Redshift sends `COPY` and `UNLOAD` traffic over the internet, even when the other end is an AWS service such as S3. That traffic does not pass through your VPC, so VPC controls don't apply to it.

**Enhanced VPC Routing** forces all `COPY` and `UNLOAD` traffic between the warehouse and its data sources through your VPC. Once it is on, the traffic behaves like any other VPC traffic: [security groups](vpc.md#security-groups-sg), [NACLs](vpc.md#network-acl-nacl), route tables, and [VPC endpoint](vpc-gateway-endpoints.md) policies control it, and [VPC Flow Logs](vpc-flow-logs.md) can record it. [AWS — Enhanced VPC routing](https://docs.aws.amazon.com/redshift/latest/mgmt/enhanced-vpc-routing.html)

- The VPC must provide a path to the data. For S3 that is usually an **S3 gateway endpoint**; otherwise traffic needs a NAT gateway or an internet gateway. If no path exists, `COPY` and `UNLOAD` fail.
- For Serverless, enabling it raises the placement requirement to **three subnets in three AZs**.

> [!tip] Exam Tip
> *"Control how Redshift traffic moves through the network"*, *"keep COPY and UNLOAD traffic inside the VPC"*, or *"monitor Redshift data-load traffic with VPC Flow Logs"* point to **Enhanced VPC Routing**.

## Redshift vs Neighboring Services

| Need                                                                                         | Service                                        |
| -------------------------------------------------------------------------------------------- | ---------------------------------------------- |
| Transactional application database                                                           | [RDS](rds.md) or [Aurora](aurora.md) (OLTP)    |
| Key-value access at any scale, millisecond reads                                             | [DynamoDB](dynamodb.md)                        |
| Persistent warehouse for heavy, repeated BI and reporting over joined data from many sources | **Redshift**                                   |
| Ad hoc SQL over S3, no warehouse, pay per query                                              | [Athena](athena.md)                            |
| Already on Redshift; join warehouse tables with S3 data                                      | **Redshift Spectrum**                          |
| Large custom Spark or Hadoop processing                                                      | Amazon EMR, or [Glue](glue.md) for managed ETL |

The course's rule for the common Athena comparison: Athena runs SQL directly against data in S3 with **no warehouse at all**, so it suits occasional ad hoc queries. If data needs to be loaded into a warehouse for heavy analytics, that is Redshift; if that warehouse should be serverless, that is Redshift Serverless.

## Exam Quick Reference

| Concept             | Key point                                                                                                                 |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| What it is          | Petabyte-scale MPP data warehouse; OLAP, column-based                                                                     |
| Not                 | An OLTP database for application transactions                                                                             |
| Modes               | Provisioned (nodes) or Serverless (RPUs); same engine and SQL                                                             |
| Leader node         | Client connections, query plans, result aggregation; not billed on multi-node clusters                                    |
| Compute nodes       | Run queries in parallel across **slices**                                                                                 |
| Availability        | Provisioned is single-AZ by default; Multi-AZ optional on RA3 and RG                                                      |
| Provisioned billing | Per node-hour while running; pause to stop compute charges; reserved nodes for steady use                                 |
| Serverless          | Namespace (data, users) plus workgroup (compute, endpoint); 2+ subnets in 2+ AZs                                          |
| Serverless billing  | Per RPU-second used; no idle compute charge, storage still billed; base 4–1024 RPUs (Region-dependent)                    |
| Query in place      | Spectrum (S3), federated query (RDS and Aurora PostgreSQL or MySQL)                                                       |
| Access              | JDBC or ODBC; Quick Sight natively                                                                                        |
| Snapshots           | Automatic every 8 hours or 5 GB per node, 1 to 35 days retention; manual retention configurable; cross-Region copy for DR |
| Serverless backups  | Recovery points about every 30 minutes, kept 24 hours                                                                     |
| Loading             | `COPY` from S3 or DynamoDB, `UNLOAD` to S3, Firehose (S3 plus `COPY`), DMS, Glue                                          |
| Network control     | Enhanced VPC Routing forces `COPY` and `UNLOAD` through the VPC (default is the internet)                                 |

## References

- Adrian Cantrill, AWS SAA course — "Redshift Architecture" lesson
- [AWS — What is Amazon Redshift?](https://docs.aws.amazon.com/redshift/latest/mgmt/welcome.html)
- [AWS — Provisioned clusters](https://docs.aws.amazon.com/redshift/latest/mgmt/working-with-clusters.html)
- [AWS — Columnar storage](https://docs.aws.amazon.com/redshift/latest/dg/c_columnar_storage_disk_mem_mgmnt.html)
- [AWS — Multi-AZ deployment](https://docs.aws.amazon.com/redshift/latest/mgmt/managing-cluster-multi-az.html)
- [AWS — Setting up Multi-AZ deployment](https://docs.aws.amazon.com/redshift/latest/mgmt/overview-multi-az.html)
- [AWS — Pausing and resuming a cluster](https://docs.aws.amazon.com/redshift/latest/mgmt/rs-mgmt-pause-resume-cluster.html)
- [AWS — Redshift RG instances powered by Graviton (May 2026)](https://aws.amazon.com/about-aws/whats-new/2026/05/amazon-redshift-rg-instances-powered-by-graviton)
- [AWS — DC2 migration approach](https://aws.amazon.com/blogs/big-data/amazon-redshift-dc2-migration-approach-with-a-customer-case-study)
- [AWS — Considerations when using Redshift Serverless](https://docs.aws.amazon.com/redshift/latest/mgmt/serverless-usage-considerations.html)
- [AWS — Serverless 2-AZ subnet configurations (July 2025)](https://aws.amazon.com/about-aws/whats-new/2025/07/amazon-redshift-serverless)
- [AWS — Billing for Serverless compute capacity](https://docs.aws.amazon.com/redshift/latest/mgmt/serverless-billing-on-demand.html)
- [AWS — Redshift Spectrum](https://docs.aws.amazon.com/redshift/latest/dg/c-using-spectrum.html)
- [AWS — Federated queries](https://docs.aws.amazon.com/redshift/latest/dg/federated-overview.html)
- [AWS — Default security configurations for new warehouses (January 2025)](https://aws.amazon.com/about-aws/whats-new/2025/01/amazon-redshift-default-security-configurations-new-warehouses)
- [AWS — Snapshots and backups](https://docs.aws.amazon.com/redshift/latest/mgmt/working-with-snapshots.html)
- [AWS — Serverless snapshots and recovery points](https://docs.aws.amazon.com/redshift/latest/mgmt/serverless-snapshots-recovery-points.html)
- [AWS — Enhanced VPC routing](https://docs.aws.amazon.com/redshift/latest/mgmt/enhanced-vpc-routing.html)
- [AWS — Database permissions (GRANT)](https://docs.aws.amazon.com/redshift/latest/dg/r_GRANT.html)
- [AWS — INSERT](https://docs.aws.amazon.com/redshift/latest/dg/r_INSERT_30.html)
- [AWS — COPY](https://docs.aws.amazon.com/redshift/latest/dg/r_COPY.html)
- [AWS — Redshift data ingestion options](https://aws.amazon.com/blogs/big-data/amazon-redshift-data-ingestion-options)
- [AWS — QuickSight evolves to Amazon Quick Suite](https://aws.amazon.com/blogs/business-intelligence/reimagine-business-intelligence-amazon-quicksight-evolves-to-amazon-quick-suite)
