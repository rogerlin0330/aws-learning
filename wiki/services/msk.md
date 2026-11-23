---
title: Amazon MSK
type: service
tags: [msk, kafka, apache-kafka, streaming, event-driven, topics, partitions, offsets, consumer-groups, express-brokers, msk-serverless, msk-connect, firehose, data-delivery, public-access, iam, sasl-scram, mtls, privatelink]
certs: [aws-saa]
updated: 2026-11-23
---

# Amazon MSK

Amazon Managed Streaming for Apache Kafka (Amazon MSK) runs **open-source Apache Kafka** for you. Kafka is a widely used distributed event log: producers write records to named **topics**, and any number of consumers read them. MSK keeps the Kafka protocol and APIs unchanged, so existing Kafka producers, consumers, and tools keep working. What changes is who runs the brokers.

MSK is the AWS answer when a workload **already uses Kafka**, or a team is moving or extending Kafka-based applications to AWS and does not want to operate Kafka clusters itself. For new event-driven work built only from AWS services, [Kinesis Data Streams](kinesis.md#kinesis-data-streams) is usually the simpler choice. [Streaming vs Messaging](../concepts/streaming-vs-messaging.md#kinesis-vs-kafka-and-amazon-msk) covers that decision in depth.

## What AWS Manages and What You Manage

The lecture puts it as a split between the control plane and the data plane. The [MSK developer guide](https://docs.aws.amazon.com/msk/latest/developerguide/what-is-msk.html) describes it the same way: MSK provides the control-plane operations for creating, updating, and deleting clusters, and you use ordinary Apache Kafka data-plane operations to produce and consume data.

| AWS manages                                                                | You manage                                                             |
| -------------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| Creating the cluster and its brokers (and the metadata controllers)        | Topics, partitions, and topic configuration such as retention          |
| Patching and Kafka version upgrades you request                            | Producers and consumers, written with Kafka clients                    |
| Detecting and replacing failed brokers                                     | Consumer groups and their offsets                                      |
| Running the cluster in an MSK-managed network, with interfaces in your VPC | Network placement (subnets, security groups) and client authentication |

The lecture says AWS also scales the cluster for you. That is true only in part. On a provisioned cluster you still decide the broker count and size, and MSK carries out the change when you ask. Only **MSK Serverless** removes those decisions completely. See [Deployment Options](#deployment-options).

## How Kafka Stores and Serves Records

Kafka is a **log**, not a queue. [Streaming vs Messaging](../concepts/streaming-vs-messaging.md#the-root-distinction--a-log-is-not-a-queue) explains why this matters; the essentials for MSK are:

- **Record**: a key and a value, plus a timestamp and optional headers. A producer might send one record per placed order.
- **Topic**: a named log, such as `orders` or `shipping`. A topic is split into **partitions**, and the partitions are spread across brokers.
- **Partition**: an append-only, ordered sequence of records. Records with the same key go to the same partition, so **ordering is guaranteed within a partition, not across a whole topic**. The lecture slide says topics are "ordered"; that is accurate only per partition.
- **Retention**: records stay on the topic for the retention period you configure, whether or not anyone has read them. Reading does not delete. This is the opposite of [SQS](sqs.md), where a processed message is deleted.
- **Offset**: a record's position in a partition. Each consumer group stores its own offsets, which works like a bookmark.
- **Consumer group**: a set of consumers that share the work of reading a topic. Different consumer groups read the same topic **independently**, each at its own pace and position. A group can also **replay** records by moving its offsets back, as long as the records are still within retention.

Independent consumer groups are why one topic can feed several applications without copying the data. With SQS, the same fan-out needs [one queue per consumer](sqs.md#sns-fanout-to-multiple-sqs-queues).

## Architecture

![Amazon MSK architecture from the lecture: on the left, a Producers box holds a Storefront service and a Warehouse service, both connected over the Kafka protocol to an MSK cluster in the centre, labelled "Hosted on brokers within your VPC". The cluster contains two topics, "Topic: Orders" with six records and "Topic: Shipping" with three records, and a label notes that records are key plus value. On the right, a Consumers box holds a Fraud service and a Recommend service, each reading the Orders topic at a different position, labelled "Individual Offsets". At the bottom right, "MSK Connect or Data Delivery" sends data towards Athena in the Consumers box.](../assets/msk-architecture.png)

The lecture's example is an online store whose services already use Kafka:

1. **The cluster.** The store creates an MSK cluster instead of building its own Kafka cluster. The brokers provide the compute and storage for the topics, and clients reach them inside the VPC across Availability Zones. The cluster is private by default.
2. **Producers.** The storefront service writes a record to the `orders` topic every time an order is placed, and the warehouse service writes a record to the `shipping` topic for every shipment. Both use a Kafka client. They do not need AWS SDKs or AWS APIs to produce data, although they need AWS credentials when the cluster uses IAM authentication.
3. **Consumers with individual offsets.** A fraud service reads every record on `orders` to look for anomalies. A recommendation service reads the same topic to build suggestions from order history. Each is its own consumer group with its own offsets, so neither affects the other, and either can rewind and replay.
4. **Getting data to S3 for Athena.** [Athena](athena.md) queries data in [S3](s3.md). To run ad hoc queries over the `shipping` topic, MSK Connect or MSK data delivery can write the topic's records to an S3 bucket. See [Moving Data Out Without a Kafka Client](#moving-data-out-without-a-kafka-client).

The slide label "within your VPC" describes where clients connect, not where the broker hardware lives. According to the [MSK developer guide](https://docs.aws.amazon.com/msk/latest/developerguide/what-is-msk.html), MSK always runs in a VPC managed by the MSK service, and a provisioned cluster's brokers are reached through **elastic network interfaces (ENIs) in your subnets**. This is the same pattern as [CloudHSM](cloudhsm.md#architecture) and other services that place interfaces into customer subnets; see [VPC](vpc.md) and [Networking](../concepts/networking.md).

## Deployment Options

| Option                            | You choose                                                      | Storage                                                                     | Billing shape (pricing page, checked 2026-11-23)                                  |
| --------------------------------- | --------------------------------------------------------------- | --------------------------------------------------------------------------- | --------------------------------------------------------------------------------- |
| **Provisioned, Standard brokers** | Broker instance type and count, EBS storage size, Kafka version | EBS volumes you provision; optional storage auto-scaling and tiered storage | Per broker-hour plus GB-month of storage, paid whether busy or idle               |
| **Provisioned, Express brokers**  | Express instance type and count                                 | Managed by MSK, pay as you go, no sizing                                    | Per broker-hour plus per GB written plus GB-month of storage                      |
| **Serverless**                    | Nothing about brokers or storage                                | Managed by MSK                                                              | Per cluster-hour, per partition-hour, per GB in and out, plus GB-month of storage |

- **Standard brokers** give the most configuration control. A Standard-broker cluster uses two or three subnets in different Availability Zones (exactly two in US West (N. California)), and MSK spreads the brokers evenly across them ([BrokerNodeGroupInfo](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-properties-msk-cluster-brokernodegroupinfo.html)).
- **Express brokers** are an M7g-based broker type with managed storage. AWS states up to 3x throughput per broker, up to 20x faster scaling, and about 90% faster recovery than Standard brokers ([Express brokers launch](https://aws.amazon.com/about-aws/whats-new/2024/11/express-brokers-amazon-msk-generally-available)). Express clusters always replicate across **three Availability Zones** ([what is MSK](https://docs.aws.amazon.com/msk/latest/developerguide/what-is-msk.html)) and have no maintenance windows.
- **MSK Serverless** suits variable or unknown workloads because there is no broker or storage sizing. It has hard per-cluster quotas, such as 200 MBps ingress and 400 MBps egress, listed on [Streaming vs Messaging](../concepts/streaming-vs-messaging.md#msk-deployment-options) and in the [MSK quotas](https://docs.aws.amazon.com/msk/latest/developerguide/limits.html).

The lecture says Serverless charges only for throughput and storage. Current [MSK pricing](https://aws.amazon.com/msk/pricing/) also charges per cluster-hour and per partition-hour, so an idle Serverless cluster still costs money.

> [!tip] Exam Tip
> "Predictable, highly tuned workload" points to **provisioned** MSK. "Variable or unknown throughput, no capacity planning" points to **MSK Serverless**. "Kafka storage without provisioning or scaling disks" points to **Express brokers**.

Cluster metadata is now handled by **KRaft** controllers rather than Apache ZooKeeper on newer clusters. MSK includes the controllers at no extra cost, and clients need no changes ([MSK FAQs](https://aws.amazon.com/msk/faqs/)).

## Networking and Public Access

An MSK cluster is **private by default**. Traffic stays on the AWS network and clients must reach the brokers from inside the VPC, or through a connected network such as [VPC peering](vpc-peering.md), Transit Gateway, VPN, or Direct Connect.

**MSK multi-VPC private connectivity** uses AWS PrivateLink to let clients in other VPCs or accounts reach a provisioned cluster privately. MSK Serverless always works this way: AWS manages the cluster and gives your VPCs private connectivity through PrivateLink endpoints, for up to five client VPCs per cluster ([MSK quotas](https://docs.aws.amazon.com/msk/latest/developerguide/limits.html)).

### Turning on public access

[Public access](https://docs.aws.amazon.com/msk/latest/developerguide/public-access.html) is documented only for **MSK Provisioned** clusters. It cannot be turned on when the cluster is created; you turn it on afterwards with the console or `UpdateConnectivity`. The cluster must meet all of these conditions (checked 2026-11-23):

1. Apache Kafka **2.6.0 or later**.
2. The cluster's subnets are **public**, with a route to an **internet gateway**. MSK assigns public IPv4 addresses to the brokers.
3. **Encryption in transit** within the cluster was turned on at creation, and **plaintext** between clients and brokers is turned off.
4. **Unauthenticated access is off**, and at least one of **IAM**, **SASL/SCRAM**, or **mTLS** is on. With SASL/SCRAM or mTLS, you first set Kafka ACLs and then set `allow.everyone.if.no.acl.found` to `false`.
5. Security groups allow inbound TCP only from the addresses that need access.

Public access has no extra MSK charge, but normal data transfer charges apply. ZooKeeper and KRaft controller nodes are never made public.

> [!warning] Gotcha — public access does not need NAT gateways
> The lecture mentions NAT gateways when describing public access. A [NAT gateway](vpc.md) only lets private resources make **outbound** connections, so it does not help internet clients reach brokers. Public access needs public subnets routed to an internet gateway, plus the encryption and authentication settings above.

> [!tip] Exam Tip
> "Clients on the internet must reach the Kafka cluster" means **provisioned MSK with public access turned on after creation**, in public subnets, with TLS and authentication enforced. "Clients in another VPC or account" usually means **multi-VPC private connectivity** (PrivateLink) instead.

## Authentication, Authorization, and Encryption

MSK supports three authenticated client access methods, plus unauthenticated access for private provisioned clusters:

| Method                 | Identity                                                               | Authorization                                         |
| ---------------------- | ---------------------------------------------------------------------- | ----------------------------------------------------- |
| **IAM access control** | AWS credentials, through the MSK IAM client library                    | [IAM](iam.md) policies with `kafka-cluster:*` actions |
| **SASL/SCRAM**         | User name and password stored in [Secrets Manager](secrets-manager.md) | Kafka ACLs                                            |
| **mTLS**               | Client TLS certificates issued by AWS Private CA                       | Kafka ACLs                                            |

The lecture describes mTLS as the option to pick "if you want encryption". That mixes two things. **TLS encryption in transit** is a cluster setting, and it is required for public access whichever authentication method you use. **mTLS** is a way to authenticate clients with certificates. IAM and SASL/SCRAM connections are also encrypted with TLS.

**MSK Serverless requires IAM access control** and does not support Kafka ACLs ([MSK Serverless](https://docs.aws.amazon.com/msk/latest/developerguide/serverless.html)). Data at rest is encrypted with [KMS](kms.md). In-transit encryption defaults to TLS only for clusters created from the console or CLI ([MSK FAQs](https://aws.amazon.com/msk/faqs/)).

## Moving Data Out Without a Kafka Client

MSK's producers and consumers normally use Kafka clients. Three managed options move topic data to [S3](s3.md) and other systems without writing a consumer:

| Option                                                                                                   | What it is                                                          | Works with                                       | Notes (checked 2026-11-23)                                                                                                                                                      |
| -------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------- | ------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [**MSK Connect**](https://docs.aws.amazon.com/msk/latest/developerguide/msk-connect.html)                | Managed Kafka Connect workers that run source and sink connectors   | MSK or self-managed Kafka reachable from the VPC | You supply the connector plugin, for example an S3 sink. Billed per MSK Connect Unit (MCU, 1 vCPU and 4 GB) hour; can auto-scale workers                                        |
| [**Amazon Data Firehose**](https://docs.aws.amazon.com/firehose/latest/dev/writing-with-msk.html)        | Firehose stream with an MSK cluster as its source                   | Provisioned and Serverless                       | The cluster must use **IAM** access control. Private connectivity needs multi-VPC private connectivity; public brokers need public access                                       |
| [**MSK data delivery**](https://docs.aws.amazon.com/msk/latest/developerguide/msk-data-delivery-s3.html) | A built-in "channel" on the cluster that writes topic records to S3 | **Express brokers only**                         | S3 general purpose buckets ($8 per TB delivered) or Apache Iceberg tables in S3 Tables ($10 per TB). 5 to 15 minute freshness, same Region only, no backfill of earlier records |

[Kinesis Data Firehose (now Amazon Data Firehose)](kinesis.md#kinesis-data-firehose-now-amazon-data-firehose) is covered on the Kinesis page. Other AWS services also read MSK directly: [Lambda](lambda.md) event source mappings, [Redshift](redshift.md) streaming ingestion, [Glue](glue.md) streaming jobs, and [OpenSearch Ingestion](opensearch.md).

[Athena](athena.md) can query Kafka topics in place through the [Athena MSK connector](https://docs.aws.amazon.com/athena/latest/ug/connectors-msk.html), a Lambda-based federated query connector, but it reads only records still within the topic's retention. For repeated ad hoc analysis, the lecture's approach of landing the data in S3 first is usually simpler.

> [!tip] Exam Tip
> "Deliver Kafka topic data to S3 with no code and no connector to manage" points to **MSK data delivery** on Express brokers, or **Firehose** with an MSK source. "Use an existing Kafka Connect connector" points to **MSK Connect**.

## When to Choose MSK

The lecture's rule matches the exam: choose MSK when the scenario involves **Apache Kafka**, such as existing Kafka applications, Kafka clients and APIs, or a migration of an on-premises Kafka cluster, and you do not want to manage Kafka infrastructure. A new event-driven design that uses only AWS services and never mentions Kafka points to [Kinesis Data Streams](kinesis.md). Both are logs with independent consumers and replay; the [concept map](../concepts/streaming-vs-messaging.md#the-concepts-map-almost-one-to-one) shows how topics, partitions, and offsets correspond to streams, shards, and sequence numbers.

> [!tip] Exam Tip
> The trigger words are **"Apache Kafka"**, **"Kafka clients"**, **"Kafka APIs"**, or **"already runs Kafka"**. Add "many independent consumers that can replay the same events" and the answer is MSK. If the question only asks to decouple work so each message is processed once, the answer is [SQS](sqs.md).

> [!warning] Gotcha — managed is not serverless
> Provisioned MSK still bills every broker-hour, even when idle, and you still plan broker count and partitions. Only MSK Serverless removes broker sizing, and it has fixed throughput and partition quotas.

## Pricing

Example rates from [Amazon MSK pricing](https://aws.amazon.com/msk/pricing/), checked 2026-11-23. Rates vary by Region.

- **Standard brokers:** per broker-hour by instance type (for example `kafka.m7g.large` at $0.204 per hour) plus $0.10 per GB-month of storage.
- **Express brokers:** per broker-hour (for example `express.m7g.large` at $0.408 per hour), plus $0.01 per GB written and $0.10 per GB-month of storage.
- **Serverless:** $0.75 per cluster-hour, $0.0015 per partition-hour, $0.10 per GB in, $0.05 per GB out, and $0.10 per GB-month of storage.
- **MSK Connect:** about $0.11 per MCU-hour. **Data delivery:** $8 per TB to S3 buckets, $10 per TB to S3 Tables.
- Replication between brokers inside the cluster is free. Standard AWS data transfer charges apply to traffic into and out of the cluster.

## Related Services

- [Kinesis](kinesis.md): the AWS-native streaming service, and the usual alternative when Kafka is not a requirement.
- [SQS](sqs.md): a queue, where consuming deletes the message; contrast with Kafka's retained log.
- [Athena](athena.md): queries MSK data after delivery to S3, or in place through the MSK connector.
- [Lambda](lambda.md): can consume MSK topics through an event source mapping.
- [VPC](vpc.md) and [VPC Interface Endpoints](vpc-interface-endpoints.md): where clients connect, and the PrivateLink technology behind multi-VPC connectivity.
- [IAM](iam.md), [Secrets Manager](secrets-manager.md), and [KMS](kms.md): IAM authentication, SASL/SCRAM credentials, and encryption at rest.

## References

- Adrian Cantrill, AWS Certified Solutions Architect – Associate (SAA-C03): "Amazon MSK"
- [AWS: What is Amazon MSK?](https://docs.aws.amazon.com/msk/latest/developerguide/what-is-msk.html) — control plane and data plane, MSK-managed VPC, Express brokers in three AZs
- [Amazon MSK FAQs](https://aws.amazon.com/msk/faqs/) — KRaft controllers, in-transit encryption default, authorization with IAM or Kafka ACLs (checked 2026-11-23)
- [AWS: Turn on public access to an MSK Provisioned cluster](https://docs.aws.amazon.com/msk/latest/developerguide/public-access.html) — Kafka 2.6.0+, not at creation, public subnets, encryption, authentication, and ACL conditions (checked 2026-11-23)
- [AWS: What is MSK Serverless?](https://docs.aws.amazon.com/msk/latest/developerguide/serverless.html) — IAM access control required, PrivateLink connectivity
- [AWS: Amazon MSK quotas](https://docs.aws.amazon.com/msk/latest/developerguide/limits.html) — Serverless throughput, partition, connection, and client VPC quotas
- [AWS CloudFormation: AWS::MSK::Cluster BrokerNodeGroupInfo](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-properties-msk-cluster-brokernodegroupinfo.html) — two or three client subnets in different AZs, brokers spread evenly
- [AWS: MSK Connect](https://docs.aws.amazon.com/msk/latest/developerguide/msk-connect.html)
- [AWS: MSK data delivery to Amazon S3](https://docs.aws.amazon.com/msk/latest/developerguide/msk-data-delivery-s3.html) and [streaming tables to Apache Iceberg](https://docs.aws.amazon.com/msk/latest/developerguide/msk-data-delivery.html) — Express brokers only, pricing, freshness (checked 2026-11-23)
- [Amazon Data Firehose: Amazon MSK as a source](https://docs.aws.amazon.com/firehose/latest/dev/writing-with-msk.html) — IAM access control requirement, private and public bootstrap brokers
- [Amazon Athena MSK connector](https://docs.aws.amazon.com/athena/latest/ug/connectors-msk.html)
- [AWS: Express brokers for Amazon MSK generally available](https://aws.amazon.com/about-aws/whats-new/2024/11/express-brokers-amazon-msk-generally-available)
- [Amazon MSK pricing](https://aws.amazon.com/msk/pricing/) — example rates (checked 2026-11-23)
