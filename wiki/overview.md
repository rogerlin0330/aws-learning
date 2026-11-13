---
title: AWS Learning — Overview
type: overview
tags: [aws, architecture, athena, elasticache, caching, valkey, redis, memcached, iam, vpc, organizations, control-tower, cost, s3, ec2, ecs, eks, kubernetes, containers, cloudwatch, cloudtrail, cloudformation, bootstrapping, serverless, lambda, step-functions, api-gateway, orchestration, dns, dnssec, rds, rds-custom, aurora, dms, database, database-migration, multi-az, high-availability, acid, base, cap-theorem, global, regional, tiers, cloudfront, elb, asg, auto-scaling, sqs, sns, messaging, queues, decoupling, fifo, dead-letter-queue, kinesis, streaming, ingestion, firehose, analytics, video-streams, kafka, msk, log-vs-queue]
updated: 2026-11-12
---

# AWS Learning Overview

A synthesis of the AWS knowledge base, tracking the big picture across services, concepts, and patterns.

## Goals

- Understand how AWS services work — deeply, not just exam-surface level
- Build mental models for designing resilient, secure, cost-effective architectures
- Prepare for AWS Solutions Architect Associate (SAA-C03) and AI Practitioner (AIF-C01)

## Core Mental Model

AWS infrastructure is built in layers:

```text
Account
└── Region
    └── VPC (isolated network)
        ├── Subnet (AZ-scoped)
        │   ├── EC2 Instance
        │   │   └── ENI (network card) ← Security Group attached here
        │   └── NAT Gateway (if public subnet)
        └── Internet Gateway (regional, attaches to VPC)
```

[IAM](services/iam.md) is a **global** layer sitting above all of this — it controls who can do what to which resources, independent of region or network topology. Network security ([Security Groups](services/vpc.md#security-groups-sg), [NACLs](services/vpc.md#network-acl-nacl)) controls traffic flows *within* this infrastructure. The two layers are complementary.

[Amazon Cognito](services/cognito.md) connects application users to this identity layer: **user pools** manage sign-in and issue tokens for [API Gateway](services/api-gateway.md) or application backends; **identity pools** provide temporary IAM-role credentials when clients need direct AWS access. These are separate choices in a [serverless architecture](patterns/serverless-architecture.md), not two mandatory stages of every login.

[VPC gateway endpoints](services/vpc-gateway-endpoints.md) connect private VPC workloads to same-Region [S3](services/s3.md) and [DynamoDB](services/dynamodb.md) through service-prefix-list routes. They illustrate the separation between **network path** (gateway endpoint), **permitted use of that path** (endpoint policy), and **resource authorization** (IAM/bucket policies). [PrivateLink interface endpoints](services/vpc-interface-endpoints.md) instead present private ENI addresses. For [SNS](services/sns.md), a [Route 53 private hosted zone](services/route53.md#private-hosted-zone) can make the usual service hostname resolve to those ENIs, preserving the application URL while changing its network destination. The [comparison and SNS walkthrough](services/vpc-interface-endpoints.md#gateway-endpoint-vs-interface-endpoint) connects endpoint routing, DNS, security groups, and multi-AZ placement.

For analytics, [AppFlow](services/appflow.md) can bring supported SaaS records into AWS, [S3](services/s3.md) holds datasets, [Glue](services/glue.md) discovers their schemas and runs transformations, and query engines such as [Athena](services/athena.md) read the data in place with SQL using shared catalog definitions, applying the schema only at query time. When the work needs code rather than a query, [Athena for Apache Spark](services/athena-spark.md) runs PySpark notebooks over the same S3 data without a cluster. AppFlow can also register eligible S3 output directly in the catalog. [Kinesis](services/kinesis.md) can feed continuous processing, while [EventBridge](services/eventbridge.md) and [Step Functions](services/step-functions.md) initiate and coordinate workflows. [Data Integration](concepts/data-integration.md) explains these responsibilities and service choices.

[VPC peering](services/vpc-peering.md) connects two VPC networks through explicit routes and firewall permissions. It is non-transitive: connecting A–B and B–C does not connect A–C. This separates **direct connectivity** from **transit routing**, while its DNS options illustrate another independent concern: reaching a private IP does not automatically expose a [Route 53 private hosted zone](services/route53.md#private-hosted-zone).

[DynamoDB](services/dynamodb.md) provides persistent key-value/document state for serverless applications. [NoSQL access patterns](concepts/nosql-access-patterns.md) connect key design to query cost and traffic distribution; [transaction models](concepts/db-transaction-model.md) distinguish fresh reads from atomic changes. Streams connect item updates to Lambda, DAX caches repeated reads, and global tables extend the data layer across Regions.

[ElastiCache](services/elasticache.md) is the general-purpose caching tier. It sits in front of [RDS](services/rds.md), [Aurora](services/aurora.md), or other data sources so read-heavy workloads scale without a proportional rise in database reads, and it holds session state so instances behind a [load balancer](services/elb.md) stay [stateless](patterns/scaling.md#the-critical-consideration--sessions). Both uses require application changes. The engine choice is a resilience choice: Valkey/Redis OSS replicate, fail over, and back up, while Memcached does not.

## Key Cross-Cutting Themes

### Security is Defense in Depth

AWS enforces security at multiple independent layers — an attacker must defeat all of them:

- **[IAM](services/iam.md) policies** — API-level: what actions are allowed on which resources (global)
- **[Security Groups](services/vpc.md#security-groups-sg)** — L3/4 stateful firewall on each ENI (instance level)
- **[NACLs](services/vpc.md#network-acl-nacl)** — L3/4 stateless firewall at the subnet boundary
- **WAF** — L7 HTTP content inspection (future page)

See [Networking Concepts](concepts/networking.md) for OSI layer breakdown and [Identity and Access Concepts](concepts/identity-and-access.md) for how IAM evaluation works.

### Stateful vs Stateless is Everywhere

This distinction recurs across AWS — understanding it once unlocks many services:

- [NACLs](services/vpc.md#network-acl-nacl) are **stateless** → must explicitly allow return traffic (ephemeral ports)
- [Security Groups](services/vpc.md#security-groups-sg) are **stateful** → return traffic automatic
- [NAT GW](services/vpc.md#nat--nat-gateway) is **stateful** → tracks connections to map return traffic

### Global Components vs Regional Tiers

[CloudFront](services/cloudfront.md) serves cached HTTP objects and routes web paths; [Lambda@Edge](services/lambda-edge.md) transforms requests/responses at cache-aware trigger positions. [Global Accelerator](services/global-accelerator.md) supplies stable anycast entry IPs for TCP/UDP traffic over the AWS backbone. [ACM](services/acm.md) supplies TLS certificates, with us-east-1 required for CloudFront viewer certificates. [Edge Delivery](concepts/edge-delivery.md) connects these roles through service comparisons and architecture explanations.

Large systems split into **global components** that get the customer to a healthy region (Route 53 for discovery and health-check failover, CloudFront for content delivery) and **regional applications** built from standard tiers (web, compute, caching, database, storage, app services). What looks like one global application — Netflix, say — is really a collection of regional applications duplicated across regions. See [Global & Regional AWS Architecture](patterns/global-regional-architecture.md) for the full model, including the three architecture types (single-region, single-region + DR, multi-region).

### Resilience Tiers

Every AWS service operates at a specific resilience scope. Knowing this is critical for HA design; [Resource Scope](concepts/resource-scope.md) is the detailed reference.

| Tier | Examples |
| --- | --- |
| Global | [IAM](services/iam.md), Route 53 |
| Regional | [VPC](services/vpc.md), IGW, [S3](services/s3.md) |
| AZ | Subnet, NAT GW, [EC2](services/ec2.md) instance (and its host, ENI, EBS volume, instance store) |

### The ENI is the Networking Atom

The [ENI (Elastic Network Interface)](concepts/networking.md) is the fundamental networking unit in AWS. Security Groups attach to ENIs, not to instances or subnets. Any resource that has an ENI in your VPC can have a Security Group — including [EC2](services/ec2.md), RDS, [ALB](services/elb.md), Fargate, and Lambda-in-VPC.

[VPC Flow Logs](services/vpc-flow-logs.md) observes traffic at these interfaces. It connects the network-control model to [CloudWatch Logs](services/cloudwatch.md) for operations and [S3](services/s3.md) plus [Athena](services/athena.md) for analysis. Records describe delayed traffic metadata; they are separate from [CloudTrail](services/cloudtrail.md) API activity and application logs.

### Compute Packaging Changes the Management Line

[EC2](services/ec2.md) gives you the full OS and runtime surface. [Containers](concepts/containers.md) package the application and dependencies into portable images that run as isolated processes on a host OS. Two managed orchestrators run those containers: [ECS](services/ecs.md) (AWS-proprietary) and [EKS](services/eks.md) (AWS-managed [Kubernetes](concepts/kubernetes.md), the cloud-agnostic open standard). Both run tasks/pods on EC2 when you want host control and pricing flexibility, or on Fargate when you want AWS to own the host layer — so launch type is orthogonal to the orchestrator choice. Both orchestrators pull their images from a registry — on AWS that is [ECR](services/ecr.md), the managed container image registry that stores, scans, and distributes images for ECS, EKS, and Fargate. Managed compute such as Fargate or Lambda shifts more infrastructure responsibility away from you, but the exam still tests the basic boundary: VM means full guest OS; container means shared host kernel plus packaged user-space dependencies.

## Pages in This Wiki

### Services

- [DynamoDB](services/dynamodb.md) — Managed tables, key design, capacity and consistency, indexes, Streams, global replication, caching, and expiry
- [ElastiCache](services/elasticache.md) — In-memory caching and session state with Valkey, Redis OSS, or Memcached

- [CloudFront](services/cloudfront.md) — HTTP delivery, cache behaviors, freshness, private content, origin protection, and TLS
- [Global Accelerator](services/global-accelerator.md) — Anycast entry IPs, regional endpoint routing, and TCP/UDP acceleration
- [Lambda@Edge](services/lambda-edge.md) — CloudFront trigger functions and cache-aware customization
- [ACM](services/acm.md) — TLS certificate lifecycle, regional associations, and exportable certificates

- [AppFlow](services/appflow.md) — SaaS connectors, reusable connections, flow processing and triggers, PrivateLink limits, and an S3/Glue analytics example
- [Athena](services/athena.md) — serverless schema-on-read SQL over S3, per-query versus capacity pricing, workgroups, and federated query
- [Athena for Apache Spark](services/athena-spark.md) — serverless PySpark notebooks over S3, in-memory DataFrames, per-DPU-hour pricing, and the Spark 3.5 move to SageMaker Unified Studio
- [Glue](services/glue.md) — crawlers, shared Data Catalog metadata, serverless batch/streaming ETL, triggers and orchestration, billing, and the Data Pipeline comparison
- [Cognito](services/cognito.md) — user directories and JWTs, identity federation and temporary AWS credentials, optional guest roles, and combined application sign-in/direct-access architectures
- [IAM](services/iam.md) — users, groups, roles (STS / federation), policies, ARNs
- [VPC](services/vpc.md) — subnets, routing, IGW, NACL, SG, NAT GW, gateway endpoints versus PrivateLink
- [VPC Interface Endpoints](services/vpc-interface-endpoints.md) — PrivateLink ENIs, gateway comparison, security and HA, plus SNS private hosted zone and alias walkthrough
- [VPC Gateway Endpoints](services/vpc-gateway-endpoints.md) — private S3/DynamoDB routing, built-in multi-AZ availability, endpoint/bucket policy distinction, and before/after diagrams
- [VPC Flow Logs](services/vpc-flow-logs.md) — network metadata, capture scopes, destinations, record interpretation, and exclusions
- [EC2](services/ec2.md) — IaaS foundations, hosts vs instances (tenancy + AZ-resilience), instance lifecycle & state-based costs (incl. stop/start vs restart), AMIs, bootstrapping (user data), SSH/RDP access, when-to-use-EC2 decision framing
- [ELB](services/elb.md) — Elastic Load Balancer family (CLB legacy, ALB for HTTP/S, NLB for TCP/TLS/UDP); a load balancer is really auto-scaling **nodes** behind one DNS A record; internet-facing vs internal (node addressing only), listeners, subnet rules (one per AZ, 2+ AZs, /27 recommended), cross-zone load balancing; the abstraction layer that decouples tiers and enables [horizontal scaling](patterns/scaling.md)
- [EC2 Auto Scaling](services/ec2-auto-scaling.md) — Auto Scaling groups; min/desired/max with the ASG's one job (keep running count equal to desired), Multi-AZ instance placement, self-healing (including the min = desired = max = 1 cheap-HA pattern), scaling policies (scheduled; dynamic: simple / step / target tracking / SQS worker pool) with the target-tracking-first recommendation hierarchy, cooldown vs instance warmup, ELB integration (auto target registration, application-aware health checks), and lifecycle hooks (pause launch/terminate transitions for custom actions, SNS/SQS/EventBridge integration); completes the [horizontal scaling](patterns/scaling.md) story alongside [ELB](services/elb.md)
- [RDS](services/rds.md) — managed relational database **server** (one instance, many databases); engines include IBM Db2, MariaDB, MySQL, Oracle, PostgreSQL, and SQL Server; databases-on-EC2 decision (single-AZ by nature; right only for unsupported engine/version, specific OS+DB combo, or OS/root access), **RDS Custom** as the middle ground (SQL Server/Oracle only, OS/engine access via SSH/RDP/Session Manager, resources visible in your account), DB subnet groups, dedicated EBS per instance, **Multi-AZ standby = synchronous** vs **read replica = asynchronous** (Db2 standby replicas and Oracle mounted replicas are DR-only), **Multi-AZ instance** (passive standby, DNS-repoint failover) vs **Multi-AZ cluster** (2 readable readers, semi-synchronous quorum, faster failover, MySQL/PostgreSQL only), per-second billing; [Aurora](services/aurora.md) is a separate product
- [Aurora](services/aurora.md) — AWS-built relational engine on a **shared cluster volume** (6 copies / 3 AZs, self-healing, synchronous at the storage level); up to 15 replicas serving reads **and** failover at once, multiple endpoints (cluster / reader / custom / instance), Standard vs I/O-Optimized storage billing, Backtrack + fast clones, **Aurora serverless** (formerly Serverless v2; ACUs, scale-to-zero), **Global Database** (async storage-layer cross-region replication for DR + global reads, RTO on the order of minutes), retired multi-master
- [DMS](services/dms.md) — Database Migration Service; managed database and analytics data migration with DMS Standard replication instances or DMS Serverless/DCUs, Full Load / Full Load + CDC / CDC Only task modes, DMS Schema Conversion or AWS SCT for heterogeneous engine changes, and Snowball/S3 bulk-transfer patterns for very large migrations
- [CloudFormation](services/cloudformation.md) — infrastructure as code (templates → stacks); cfn-init desired-state bootstrapping, configSets, cfn-hup re-runs, CreationPolicy + cfn-signal completion handshake
- [ECS](services/ecs.md) — managed container orchestration (AWS-proprietary); task definitions, services, task roles, EC2 launch type vs Fargate
- [EKS](services/eks.md) — AWS-managed Kubernetes; control plane + etcd (multi-AZ), node options (EC2 / managed node groups / Fargate), two-VPC architecture, ECS-vs-EKS choice
- [ECR](services/ecr.md) — managed container image registry; registry/repository/image/tag hierarchy, public vs private permission model, IAM + repository policies, Basic vs Enhanced (Inspector) scanning, cross-region + cross-account replication
- [Lambda](services/lambda.md) — Function as a Service; the serverless compute primitive that bills only for execution duration; resource model (1769 MB = 1 vCPU, 900 s max timeout), the "Docker = not Lambda" exam signal, public vs VPC networking (execution role needs EC2/ENI permissions), the two-part security model (resource policy = who invokes, execution role = what it does), three invocation types (sync / async-with-idempotency / event source mapping), versions vs aliases, and cold vs warm starts + provisioned concurrency
- [S3](services/s3.md) — buckets, bucket policies, ACLs (legacy), Block Public Access, pre-signed URLs, versioning, SSE/CSE encryption, multipart upload + Transfer Acceleration, storage classes + lifecycle, replication (CRR/SRR), S3 Select / Glacier Select, event notifications + EventBridge, access logging
- [Systems Manager (SSM)](services/ssm.md) — ops-management suite; **Parameter Store** covered in depth — config/secrets store, the proper alternative to EC2 user-data secrets; String/StringList/SecureString, hierarchical names, regional + no cross-Region replication, standard vs advanced (KMS direct-encrypt vs envelope) tiers, and Parameter Store vs Secrets Manager
- [Route 53](services/route53.md) — hosted zones, routing policies, health checks
- [Organizations](services/organizations.md) — multi-account management, consolidated billing, SCPs
- [Control Tower](services/control-tower.md) — "Organizations with superpowers"; orchestrates a well-architected landing zone (foundational OUs + Audit/Log Archive accounts), guardrails (preventative SCP / detective Config), and Account Factory self-service provisioning
- [CloudWatch](services/cloudwatch.md) — monitoring & observability; log groups, metric filters, alarms, and the **CloudWatch Agent** for in-guest EC2 metrics + OS/application logs
- [CloudTrail](services/cloudtrail.md) — audit & governance; API/account-activity events, Event History vs trails, management/data/Insight events, global service events, organizational trail, log file integrity
- [EventBridge](services/eventbridge.md) — serverless event bus (formerly CloudWatch Events); "if X happens, or at time Y, then do Z" — producers emit events, rules match on content or schedule, targets (Lambda and others) receive the JSON; the AWS implementation of the [event-driven](patterns/architecture-evolution.md#stage-6--event-driven) router/bus pattern, distinct from CloudWatch Alarms (metric thresholds)
- [SNS](services/sns.md) — Simple Notification Service; fully managed pub/sub messaging (topic = base entity); publishers (CloudWatch, CloudFormation, ASG, S3, custom apps) write into a topic and subscribers (HTTP/S, email, SQS, Lambda, SMS, mobile push) receive; subscription **filter policies** (opt into a subset), **fanout** (one topic to many SQS queues), 256 KB messages (2 GB via S3-backed Extended Client), retries + optional DLQ, cross-account via topic policy, and Standard vs FIFO (FIFO = strict ordering, SQS-only subscribers); a core [decoupling / event-driven](patterns/architecture-evolution.md) primitive alongside [SQS](services/sqs.md) and [EventBridge](services/eventbridge.md)
- [SQS](services/sqs.md) — Simple Queue Service; managed message queues and the canonical **decoupling** primitive; the send/poll/**visibility timeout**/delete loop that gives fault tolerance for free, **delay queues** and message timers (hidden on *add* rather than on *receive*, 15-minute cap), **Standard** (at-least-once, best-effort order, nearly unlimited throughput) vs **FIFO** (`.fifo` suffix, strict ordering via `MessageGroupId`, dedup via `MessageDeduplicationId`, 300/3,000 TPS default with region-tiered high throughput mode), **dead-letter queues** (`ReceiveCount` past `maxReceiveCount`, the enqueue-timestamp retention gotcha, redrive policy vs redrive allow policy), **long polling**, per-request billing in 64 KB chunks, and the **[SNS](services/sns.md) fanout** and **worker-pool** ([ASG](services/ec2-auto-scaling.md) on queue length) architectures
- [Step Functions](services/step-functions.md) — serverless **orchestration**; state machines coordinate short-lived [Lambda](services/lambda.md) functions into long-running, stateful, branching workflows, working around Lambda's 15-minute timeout and stateless environments; Standard (1 year) vs Express (5 minutes) chosen at creation and unchangeable, eight ASL state types with **Task** the only one that performs work, definitions in JSON-based **ASL**, permissions via an assumed IAM role; the orchestration layer of a [serverless architecture](patterns/serverless-architecture.md)
- [API Gateway](services/api-gateway.md) — managed APIs; an **intermediary between clients and integrations** with a request phase (authorize, validate, transform), the integration itself, and a response phase (transform, prepare, return); HTTP/REST/WebSocket types, three endpoint types (edge-optimized, regional, private), Cognito / Lambda-authorizer / IAM authentication, per-stage settings including **caching** and **canary deployments**, and direct AWS service integrations that need **no compute**; the front door of a [serverless architecture](patterns/serverless-architecture.md) and the regional entry point alongside [ELB](services/elb.md)
- [Kinesis](services/kinesis.md) — the streaming-data family; **Data Streams** ingests at scale and lets many consumers read independently from a **24-hour-to-365-day rolling window** (the replay property [SQS](services/sqs.md) lacks), scaling through **shards** with **enhanced fan-out** for contention-free per-consumer throughput; **Firehose** (now Amazon Data Firehose) adds managed **near real-time delivery** to S3, Redshift, OpenSearch, Splunk, and HTTP with an optional [Lambda](services/lambda.md) transform; **Data Analytics** (now Managed Service for Apache Flink) does real-time processing between an input and an output stream, enriched with S3 reference data; **Video Streams** ingests live media for analytics via Rekognition
- [KMS](services/kms.md) — key management; envelope encryption, key policies + IAM + grants

### Concepts

- [NoSQL Access Patterns and Key Design](concepts/nosql-access-patterns.md) — Design direct read paths, spread traffic, and choose indexes with explicit freshness requirements

- [Edge Delivery](concepts/edge-delivery.md) — Caching, global network acceleration, security boundaries, and service selection

- [Data Integration](concepts/data-integration.md) — storage vs metadata vs transformation vs query vs orchestration; ETL/ELT and batch/streaming choices
- [Networking](concepts/networking.md) — OSI layers, firewall layers, routing, internet connectivity patterns
- [Identity and Access](concepts/identity-and-access.md) — auth vs authz, identity types, role assumption, federation
- [Resource Policies vs Identity Policies](concepts/resource-vs-identity-policies.md) — the two policy families; cross-account & anonymous access
- [Resource Scope](concepts/resource-scope.md) — global / regional / AZ / edge tiers; blast radius and movement cost
- [DNS](concepts/dns.md) — resolution, record types, TTL
- [DNSSEC](concepts/dnssec.md) — securing DNS; origin auth + integrity, chain of trust root→record (RRSIG/DNSKEY/ZSK/KSK/DS), trust anchor + root signing ceremony, Route 53 signing + validation
- [Database Transaction Models](concepts/db-transaction-model.md) — ACID, BASE, and CAP; distinguish transaction guarantees, read freshness, and cross-Region replication
- [TLS/HTTPS](concepts/tls-https.md) — certificate trust, TLS handshake, ACM
- [Streaming vs Messaging](concepts/streaming-vs-messaging.md) — choosing between [Kinesis](services/kinesis.md), [SQS](services/sqs.md), and Kafka/**Amazon MSK**; the log-vs-queue root distinction (consuming a queue *removes* the message, consuming a log only *reads* it), why fan-out and replay are free in a stream and deliberate in a queue, the Kinesis-to-Kafka concept map, and when an existing Kafka investment makes MSK the answer
- [Containers](concepts/containers.md) — container vs VM mental model, Docker image layers, runtime R/W layer, registries, ECS as orchestration
- [Kubernetes](concepts/kubernetes.md) — cloud-agnostic orchestration; control plane vs nodes, pods/services/ingress, ephemeral vs Persistent Volume storage, mapped to AWS via EKS

### Patterns

- [Global & Regional AWS Architecture](patterns/global-regional-architecture.md) — the foundational mental model: global components (Route 53, CloudFront) deliver customers to regional applications built from standard tiers (web / compute / caching / database / storage / app services); three architecture types (single-region, +DR, multi-region)
- [Architecture Evolution — Coupling, Queues & Event-Driven](patterns/architecture-evolution.md) — the monolithic-to-event-driven progression as one axis (coupling, communication, billing); monolith signature (fails/scales/bills together), load-balancer ≠ decoupling, the queue as the decoupler, the worker-fleet ASG scaling on queue length, SQS Standard vs FIFO, and the event router/bus (EventBridge) as a serverless building block
- [Cost Optimization](patterns/cost-optimization.md) — Savings Plans, RIs, Spot, decision tree, pooling via Organizations
- [Scaling — Horizontal vs Vertical](patterns/scaling.md) — bigger box vs more boxes; the session problem; stateless design; how horizontal scaling combines with ELB + Auto Scaling for HA
- [Boot Time to Service Time](patterns/boot-time-optimization.md) — AMI baking vs bootstrapping vs combined; minimizing post-launch time while keeping flexibility

### Certifications

- [AWS Solutions Architect Associate](certifications/aws-saa.md) — exam tips, comparison tables, common traps

## Multi-Account Architecture

Most real-world AWS deployments use **many accounts** — a per-team / per-environment / per-business-unit split provides hard blast-radius isolation that VPCs alone cannot. [AWS Organizations](services/organizations.md) is the glue: one bill, OU-based grouping, [SCP](services/organizations.md#service-control-policies-scps) guardrails, and a single login point that uses [cross-account IAM roles](services/iam.md#cross-account-access) to move between accounts. The cost-side benefit is that [Savings Plans and Reserved Instances pool org-wide](patterns/cost-optimization.md#pooling-across-accounts) — so multi-account doesn't multiply purchase commitments.

[AWS Control Tower](services/control-tower.md) sits one layer above Organizations: instead of assembling the governance yourself, it **orchestrates** Organizations, IAM Identity Center, CloudFormation, and Config into a well-architected **landing zone** — default OUs, isolated **Log Archive** and **Audit** foundational accounts, SSO, **guardrails** (preventative SCP vs detective Config rules), and an **Account Factory** for self-service, baselined account provisioning.

## Topics To Cover Next

- EC2: placement groups (user-data & bootstrapping now covered on the [EC2 page](services/ec2.md#bootstrapping); IMDS / IMDSv2 on the [IMDS section](services/ec2.md#instance-metadata-service-imds))
- CloudFormation: the full surface beyond cfn-init — parameters, mappings, outputs, intrinsic functions, change sets, nested stacks / StackSets, drift detection (the [CloudFormation page](services/cloudformation.md) currently covers only the bootstrapping features)
- ELB deep dives: listeners in depth, target groups in detail — the [ELB architecture page](services/elb.md) and the [EC2 Auto Scaling page](services/ec2-auto-scaling.md) (scaling policies, health checks, lifecycle hooks) now cover the rest; remaining ASG gaps: warm pools, predictive scaling, instance refresh, termination policies
- Container services: ELB integration patterns and App Mesh / service discovery — builds on [ECS](services/ecs.md), [EKS](services/eks.md), [ECR](services/ecr.md), and the [Containers](concepts/containers.md) / [Kubernetes](concepts/kubernetes.md) concept pages
- AWS AI services (for AI Practitioner cert)
- Architecture patterns: HA, multi-AZ, disaster recovery tiers
