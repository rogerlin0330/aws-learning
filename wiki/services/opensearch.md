---
title: Amazon OpenSearch Service
type: service
tags: [opensearch, elasticsearch, kibana, search, full-text-search, log-analytics, near-real-time, domain, serverless, ocu, opensearch-ingestion, opensearch-dashboards, multi-az, vector-search, rag, siem, clickstream]
certs: [aws-saa, aws-aip]
updated: 2026-11-13
---

# Amazon OpenSearch Service

Amazon OpenSearch Service is a managed **search and analytics** service. It runs the open-source **OpenSearch** engine and **OpenSearch Dashboards**, which were forked from **Elasticsearch** and **Kibana**. AWS renamed the service from Amazon Elasticsearch Service in 2021, and it can still run legacy Elasticsearch versions up to 7.10 ([AWS: What is Amazon OpenSearch Service?](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/what-is.html)). Older exam questions and documentation may use the Elasticsearch and Kibana names.

The service does two jobs, both in **near real time**:

1. **Full-text search**, for example over a product catalog or a document library.
2. **Log analytics**, for example over application logs, [VPC Flow Logs](vpc-flow-logs.md), or [CloudTrail](cloudtrail.md) events.

For the SAA exam, OpenSearch is a **selection** topic. Know what it is, how it is laid out, and which scenario wording points to it. The deciding words are "search", "log analytics", and "near real time".

## How Data Becomes Searchable

Data arrives as **JSON documents** and is stored in an **index**. Every field in a document is indexed on arrival, so you can search on any field without designing a schema first. On a managed domain, a new document becomes searchable after the index refreshes, which by default happens every second for indexes that are receiving searches ([OpenSearch: refresh index](https://docs.opensearch.org/latest/api-reference/index-apis/refresh/)). This is why the lecture repeats "near real time": data is indexed as it lands and can be queried a moment later, not after a batch load.

> [!warning] Gotcha: "no schema" means the schema is inferred
> OpenSearch uses **dynamic mapping**: the first document that contains a new field decides that field's type ([OpenSearch: mappings](https://docs.opensearch.org/latest/field-types/)). If a later document sends a different type for the same field, for example text where a number was first seen, that document can be rejected. Production log pipelines usually define explicit mappings or index templates for important fields.

## Deployment Options

There are two ways to run it:

| Option                        | What you manage                                               | Scaling                                                                         | Billing                                                                                        |
| ----------------------------- | ------------------------------------------------------------- | ------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| **Domain** (the classic mode) | A cluster with instance types and node counts that you choose | No automatic scaling of node count. To grow, add nodes or change instance types | Per instance-hour while the domain exists, whether or not it is queried, plus attached storage |
| **Serverless** (collections)  | No nodes or instance types; you set OCU capacity limits       | Scales automatically between the minimum and maximum you set                    | Per OCU allocated, plus data stored in S3                                                      |

### Domains

A **domain** is an OpenSearch cluster. Clients connect to one domain endpoint. Inside the domain are **data nodes**, the instances with storage that hold the indexes and run queries. AWS also recommends three **dedicated master nodes** (now called cluster manager nodes) to keep the cluster stable, and Multi-AZ with Standby provisions them automatically ([AWS: dedicated master nodes](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/managedomains-dedicatedmasternodes.html)).

For older data that is searched less often, a domain can move indexes to cheaper storage tiers, **UltraWarm** and **cold storage**, which are backed by S3 ([AWS: UltraWarm storage](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/ultrawarm.html)). This keeps months of logs searchable without paying for hot nodes to hold them.

### Serverless

With **OpenSearch Serverless** you create **collections** instead of domains. Capacity is measured in **OpenSearch Compute Units (OCUs)**. One OCU is [6 GiB of memory with matching vCPU](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/serverless-scaling.html), and indexing and search scale separately. Indexes are stored in S3, so compute and storage are separate ([AWS: Serverless overview](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/serverless-overview.html)).

A collection has one type, chosen at creation and not changeable later: **search** (full-text), **time series** (logs and metrics), or **vector search** (embeddings).

The lecture says you set a minimum and maximum OCU count and that Serverless can **scale to zero** when idle. This is accurate for the **next generation** of OpenSearch Serverless, which became [generally available in May 2026](https://aws.amazon.com/about-aws/whats-new/2026/05/amazon-opensearch-serverless-next-generation-generally-available/):

- Collections are placed in a **collection group**, which sets minimum and maximum OCUs for indexing and for search. The maximum is [1,700 OCUs each](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/serverless-scaling.html).
- The minimum defaults to **0**. After [10 minutes with no requests](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/serverless-scale-to-zero.html), indexing and search capacity drop to zero OCUs. The first request after that waits about **10 to 30 seconds** while capacity returns.
- To avoid that cold start, set a non-zero minimum, as the lecture suggests with a minimum of 1.
- Scale to zero applies only to next-generation collections in a collection group, not to older (Classic) collections.

> [!warning] Gotcha: Serverless is near real time, but slower than one second
> On Serverless, the refresh interval for search and time series collections is about [10 seconds](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/serverless-overview.html) and cannot be changed. The lecture's "searchable within about a second" describes the default on a managed domain.

## Networking and Resilience

**Connectivity:** a domain is either **inside a VPC** or has a **public endpoint**. You choose at creation. Serverless collections use network policies to allow VPC endpoint access, public access, or both. See [Networking](../concepts/networking.md).

**Multi-AZ (domains):** you can place a domain's data nodes across **two or three AZs** ([AWS: multi-AZ domains](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/managedomains-multiaz.html)):

- **Without Standby:** nodes are spread across two or three AZs, with 99.9% availability.
- **Multi-AZ with Standby:** three AZs, with the nodes in one AZ held as standby. Standby nodes do not serve searches, and the service activates them in **under a minute** when it detects a failure. This option gives 99.99% availability and requires at least two replicas per index, three dedicated master nodes, and a data node count that is a multiple of three.

> [!warning] Gotcha: spreading nodes is not enough; you also need replica shards
> The lecture says data is copied across AZs so the domain survives losing a node or an AZ. That depends on **replica shards**. An index is split into primary shards, and each replica shard is a copy of a primary that OpenSearch places on a node in another AZ. AWS says that without replicas it "can't distribute copies of your data to other Availability Zones" ([multi-AZ domains](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/managedomains-multiaz.html)). The default is one replica per index, and Multi-AZ with Standby requires two.

**Serverless:** there are no AZs or replicas to choose. AWS provides high availability by default and keeps the indexes in S3.

## Getting Data In

The lecture groups ingestion paths into three kinds:

| Path                     | Sources                                                                                                                                                                                                                           | Notes                                                                                                                        |
| ------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| **Direct integrations**  | [Amazon Data Firehose](kinesis.md#kinesis-data-firehose-now-amazon-data-firehose), [CloudWatch Logs](cloudwatch.md) subscription filters                                                                                          | Firehose buffers, batches, and can transform with Lambda before delivery. CloudWatch Logs can stream a log group to a domain |
| **Lambda in between**    | [Kinesis Data Streams](kinesis.md#kinesis-data-streams), [S3](s3.md) events, [DynamoDB Streams](dynamodb.md)                                                                                                                      | Your [Lambda](lambda.md) function reads the source and writes documents to OpenSearch                                        |
| **OpenSearch Ingestion** | Many sources, including S3, DynamoDB, DocumentDB, Kinesis Data Streams, Kafka, HTTP, and OpenTelemetry ([supported sources](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/pipeline-config-reference.html)) | Managed pipeline that collects, filters, and reshapes data before delivering it, with no Lambda code or servers to run       |

**OpenSearch Ingestion** is built on the open-source **Data Prepper** collector. It writes to both domains and Serverless collections, and it is billed in **Ingestion OCUs** allocated to a pipeline whether or not data is flowing ([AWS: OpenSearch Ingestion](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/ingestion.html)). You do not have to use it: the direct and Lambda paths still work without it.

> [!warning] Gotcha: the CloudWatch Logs path uses Lambda behind the scenes
> The lecture calls CloudWatch Logs delivery "native". The console does set it up for you, but the subscription delivers through a Lambda function, and you must supply a Lambda execution role that can write to the domain ([AWS: streaming CloudWatch Logs to OpenSearch Service](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_OpenSearch_Stream.html)). It works only for log groups in the Standard log class, and the page describes OpenSearch Service domains, not Serverless collections. For Serverless, a common route is a subscription to Firehose or to an OpenSearch Ingestion pipeline.

## Getting Insight Out

Once the data is indexed, there are three common ways to use it:

- **OpenSearch Dashboards** is the built-in UI. You can search, build visualizations, and set alerts.
- **Amazon Quick Sight** (formerly QuickSight) can use OpenSearch as a data source for business intelligence dashboards.
- **Custom applications** can query the OpenSearch API directly.

None of these receive the data. They all query the index of JSON documents held in OpenSearch, so their results are as current as the index.

## Architecture Example

![OpenSearch architecture from the lecture: on the IN side, a CloudWatch Logs log group feeding OpenSearch, and Kinesis Data Streams feeding Firehose into OpenSearch; in the middle, an OpenSearch domain with data nodes in AZ A and AZ B, labelled near real time, storing JSON documents in an index where every field is searchable; on the OUT side, OpenSearch Dashboards, Amazon Quick Sight, and custom applications querying the domain](../assets/opensearch-architecture.png)

The lecture's example company has two teams sending data to one domain spread across two AZs:

1. **Operations team.** [VPC Flow Logs](vpc-flow-logs.md) already go to a CloudWatch Logs log group. A subscription filter on that log group streams the events into OpenSearch.
2. **Web team.** Clickstream events from the web store already go to Kinesis Data Streams. Firehose reads the stream, buffers and batches the events, optionally transforms them, and delivers them to OpenSearch.

The operations team could also send its CloudWatch Logs subscription to Firehose, so both feeds arrive through Firehose. Both designs are valid, and the exam uses both. A source that has no direct integration, or that needs reshaping, can go through OpenSearch Ingestion.

On the output side, the operations team uses OpenSearch Dashboards to search the flow logs, build an operational dashboard, and alert on rejected traffic. The web team uses Quick Sight to analyze how customers use the store. Both views are near real time because they read the index directly.

The exam pattern to remember is **Kinesis in, OpenSearch in the middle, Quick Sight out** for real-time clickstream search and visualization.

## Use Cases

| Use case                   | Examples                                                                                                                                      |
| -------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| **Log analytics**          | Application and infrastructure logs, VPC Flow Logs, CloudTrail                                                                                |
| **Application monitoring** | Logs, metrics, and traces in one place                                                                                                        |
| **Full-text search**       | Product catalogs, content, and documents; relevance-ranked text search that [RDS](rds.md) and [DynamoDB](dynamodb.md) do not provide at scale |
| **Clickstream analytics**  | Kinesis feeding OpenSearch, with Quick Sight for business intelligence                                                                        |
| **Security analytics**     | SIEM-style investigation across authentication, [AWS WAF](waf.md), and [GuardDuty](guardduty.md) findings                                     |
| **Vector search**          | Semantic search and retrieval-augmented generation (RAG)                                                                                      |

**Vector search** stores **embeddings**, lists of numbers that represent the meaning of text or images, and finds the stored items closest to a query's embedding. This gives **semantic search** ("find things like this") instead of exact word matching. In **RAG**, an application retrieves the most relevant documents from OpenSearch and passes them to a model so its answer is grounded in your data. An OpenSearch Serverless vector collection is one of the vector stores that [Amazon Bedrock Knowledge Bases](https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base-setup.html) can use. [Neptune Analytics](neptune.md#deployment-options-and-capabilities) fills the same role when the retrieval also needs graph relationships (GraphRAG).

## Choosing OpenSearch

| Need                                                                                             | Service                                   |
| ------------------------------------------------------------------------------------------------ | ----------------------------------------- |
| Search or analyze logs and documents in **near real time**, on any field, with a managed service | **OpenSearch**                            |
| Store large amounts of data cheaply and query it **occasionally** with SQL                       | [S3](s3.md) plus [Athena](athena.md)      |
| Interactive queries over logs that are already in CloudWatch Logs, without another service       | [CloudWatch Logs Insights](cloudwatch.md) |
| Heavy, repeated BI reporting over structured data from many sources                              | [Redshift](redshift.md)                   |
| Look up items by known keys at any scale                                                         | [DynamoDB](dynamodb.md)                   |
| Queries that follow relationships between connected items                                        | [Neptune](neptune.md)                     |

The trade-off with Athena is cost against freshness. OpenSearch keeps compute running (except a Serverless group scaled to zero) so data is indexed and searchable as it arrives. Athena runs nothing between queries and charges per query, but it scans raw files in S3 each time.

## Exam Quick Reference

| Scenario wording                                                         | Answer                                                                       |
| ------------------------------------------------------------------------ | ---------------------------------------------------------------------------- |
| "Search", "full-text search", "log analytics" in "near real time"        | OpenSearch                                                                   |
| Elasticsearch, Kibana, or the ELK stack (as a managed AWS service)       | OpenSearch                                                                   |
| Real-time clickstream search and dashboards                              | Kinesis, then OpenSearch, then Quick Sight                                   |
| Search VPC Flow Logs or CloudTrail logs with dashboards and alerts       | CloudWatch Logs or Firehose into OpenSearch, viewed in OpenSearch Dashboards |
| Ingest from many sources, filter or transform, no servers or Lambda code | OpenSearch Ingestion                                                         |
| Search capacity that scales automatically, or to zero when idle          | OpenSearch Serverless                                                        |
| Keep older logs searchable at lower cost on a domain                     | UltraWarm or cold storage                                                    |
| Semantic search or a vector store for RAG                                | OpenSearch vector search                                                     |
| Store cheaply, query only occasionally                                   | S3 plus Athena, not OpenSearch                                               |

> [!tip] Exam Tip
> The giveaway is **near real time** combined with **search** or **log analytics**. OpenSearch indexes each JSON document as it arrives and makes it searchable within about a second on a domain. If the scenario instead stresses low cost and rare queries, prefer S3 with Athena.

## References

- Adrian Cantrill, *AWS Certified Solutions Architect – Associate* course: Amazon OpenSearch Service lecture
- [AWS: What is Amazon OpenSearch Service?](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/what-is.html)
- [AWS: Configuring a multi-AZ domain](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/managedomains-multiaz.html)
- [AWS: Dedicated master nodes](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/managedomains-dedicatedmasternodes.html)
- [AWS: UltraWarm storage](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/ultrawarm.html)
- [AWS: Amazon OpenSearch Serverless overview](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/serverless-overview.html)
- [AWS: Managing capacity limits for OpenSearch Serverless](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/serverless-scaling.html)
- [AWS: Scale to zero for OpenSearch Serverless](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/serverless-scale-to-zero.html)
- [AWS What's New: Next generation of OpenSearch Serverless is generally available (May 2026)](https://aws.amazon.com/about-aws/whats-new/2026/05/amazon-opensearch-serverless-next-generation-generally-available/)
- [AWS: Amazon OpenSearch Ingestion](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/ingestion.html) and [supported plugins and sources](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/pipeline-config-reference.html)
- [AWS: Streaming CloudWatch Logs data to Amazon OpenSearch Service](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_OpenSearch_Stream.html)
- [AWS: CloudWatch Logs subscription filters](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/SubscriptionFilters.html)
- [OpenSearch: Refresh index API](https://docs.opensearch.org/latest/api-reference/index-apis/refresh/)
- [OpenSearch: Mappings and field types](https://docs.opensearch.org/latest/field-types/)
- [AWS: Set up a vector store for Bedrock Knowledge Bases](https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base-setup.html)
