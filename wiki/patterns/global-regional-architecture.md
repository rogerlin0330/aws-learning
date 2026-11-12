---
title: Global & Regional AWS Architecture
type: pattern
tags: [architecture, foundations, global, regional, tiers, route53, cloudfront, cdn, dns, health-checks, failover, multi-region, dr, web-tier, compute-tier, caching, database-tier, storage-tier, app-services, decoupling]
certs: [aws-saa]
updated: 2026-11-11
---

# Global & Regional AWS Architecture

The foundational mental model for designing on AWS: every large system splits into **global components** (whose job is to get the customer to a healthy region) and **regional components** (the application itself, built from a standard set of tiers). This is a scaffold, not a deep-dive — each service mentioned here has (or will have) its own page.

The running example is Netflix. It looks like one *global* application, but it is really **a collection of smaller regional applications** — discrete, independent blocks of infrastructure duplicated across multiple regions, which together form the global platform.

> [!tip] The core mental model
> Think of every system as **a collection of regions making up a whole**. Very few AWS products are truly global — most run inside a region, and many regions together make up AWS (see [Resource Scope](../concepts/resource-scope.md) for the per-service breakdown). For any architecture, keep asking: *which parts give global resilience, and which parts are regional only?*

## The Three Architecture Types

As a solutions architect you will design three main types of architecture:

1. **Single region / single country** — small-scale; only ever exists in one region.
2. **Single region with DR** — normally runs in one region, but if that region fails it **fails over to a secondary region**.
3. **Multi-region** — operates across multiple regions simultaneously and must **keep working through the failure** of one or more of them.

## Global Components — Get the Customer to a Healthy Region

At the global level there are three architectural jobs:

1. **Global service location and discovery** — when a user types `netflix.com`, how does their machine discover where to point? The answer is [DNS](../concepts/dns.md), implemented in AWS by [Route 53](../services/route53.md).
2. **Content delivery** — how does content reach users globally: cached in pockets around the world, or pulled from one central location? The answer is a **Content Delivery Network (CDN)**, implemented by [CloudFront](../services/cloudfront.md).
3. **Global health checks and failover** — detect whether infrastructure in one location is healthy, and move customers to another region when needed. The answer is [Route 53 health checks](../services/route53.md#health-checks).

The whole purpose of the global layer is to deliver customers to a suitable regional infrastructure location: isolate regional failures, move sessions to a working region, ideally direct customers to their *nearest* region, and improve delivery via caching (CloudFront).

[Global Accelerator](../services/global-accelerator.md) adds a global network entry option: static anycast IPs bring TCP/UDP traffic onto AWS close to users, then select configured regional endpoints. This complements DNS discovery and differs from CloudFront caching. [Lambda@Edge](../services/lambda-edge.md) adds request/response customization to the CDN path; [Edge Delivery](../concepts/edge-delivery.md) separates routing, cache correctness, viewer authorization, origin protection, and TLS.

## Regional Components — The Application Tiers

Once the global layer has delivered the customer into a specific region, you "zoom in" and think regionally. A regional application is built from **tiers** — high-level groupings of functionality: the **web tier**, **compute tier**, **caching**, **storage**, **database tier**, and **application services**.

![Architecture diagram: a dotted AWS Region box containing vertical tier bands (web, compute, caching, database, storage, app services) with request flow arrows between them; Route 53 and CloudFront sit outside the box as global components, with CloudFront linked directly to an S3 bucket in the storage tier as its origin](../assets/aws-global-and-regional-architecture.png)

How to read the diagram: the dotted box labelled "AWS Region" encloses the regional components; the services drawn outside it on the right (Route 53, CloudFront, the users on the public internet) are the global components. The colored vertical bands group services by tier, and the pink arrows show request/data flow between tiers: users reach Route 53 (discovery) and CloudFront (delivery), then enter the **web tier** (the regional entry point), which passes traffic to the **compute tier**, which in turn consumes **caching, database, storage, and app services**. CloudFront also reaches **directly into [S3](../services/s3.md)** as its origin for media.

> [!warning] Gotcha: don't over-read the icons
> This is a deliberately "everything, everywhere" diagram — one representative service per tier. The icons illustrate a tier's *role*; a real design picks one option per tier, not all of them. The exam-relevant takeaway is **which tier** a service belongs to and **whether it is global or regional**, not the specific icon.

### Web Tier

The **regional entry point** — typically a regional AWS service such as an **[Application Load Balancer](../services/elb.md)** or **[API Gateway](../services/api-gateway.md)**. Its job is to **abstract customers away from the underlying infrastructure**, so that the infrastructure behind it can scale, fail, or change without impacting customers. This abstraction is what makes [horizontal scaling](scaling.md) possible.

### Compute Tier

Provides the actual functionality the customer receives via the web tier: [EC2](../services/ec2.md), Lambda, or containers ([ECS](../services/ecs.md)). In the Netflix example the load balancer sends traffic to EC2.

### Storage Tier

Consumed by the compute tier: [EBS](../services/ebs.md) (block), [EFS](../services/efs.md) (file), and [S3](../services/s3.md) (object) — see [Storage Fundamentals](../concepts/storage.md) for the block/file/object split. **CloudFront can use S3 as an origin**: Netflix stores movies and shows on S3, and CloudFront caches them for a global audience. Tiers are separate components that consume services from each other, so CloudFront reaching directly into S3 is normal.

### Database Tier

Structured data storage: [RDS](../services/rds.md), [Aurora](../services/aurora.md), DynamoDB, and Redshift (data warehousing).

### Caching Tier

Sits **in front of** the database. Most applications don't hit the database directly; they consult a cache first — **[ElastiCache](../services/elasticache.md)** (general-purpose) or **[DAX](../services/dynamodb.md#dynamodb-accelerator-dax)** (DynamoDB Accelerator). Only on a cache miss is the database consulted, and the cache is then updated. This requires the application to implement the caching logic; see [ElastiCache read caching](../services/elasticache.md#read-caching-architecture).

> [!tip] Exam Tip: why cache in front of the DB
> Caching is in-memory, so it is cheap and fast; databases are expensive relative to the data volume they hold. Offload **reads** from the database into the cache wherever possible to improve performance and reduce cost. "Minimize reads to the database" is the trigger phrase for a caching-tier answer.

### Application Services Tier

A suite of products that provide functionality *to* applications: **[Kinesis](../services/kinesis.md)**, **[Step Functions](../services/step-functions.md)**, **[SQS](../services/sqs.md)**, **[SNS](../services/sns.md)**. These range from simple functions (email, notifications) to services that change an application's architecture — the classic example being **decoupling components with [SQS queues](../services/sqs.md)**. See [Architecture Evolution](architecture-evolution.md) for how decoupling and event-driven design move a system from tight, synchronous, always-on coupling to loose, asynchronous, consume-only-when-working.

## Exam Quick-Reference

| Concept | Key point |
| --- | --- |
| Three architecture types | Single-region · single-region + DR failover · multi-region |
| Global job 1: service discovery | DNS, via [Route 53](../services/route53.md) — resolve the name and direct to a healthy region |
| Global job 2: content delivery | CDN, via CloudFront — cache near users; origin can be [S3](../services/s3.md) |
| Global job 3: health check / failover | [Route 53 health checks](../services/route53.md#health-checks) move sessions to a healthy region |
| Regional entry point | Web tier — [ALB](../services/elb.md) / [API Gateway](../services/api-gateway.md) (abstracts customers from infrastructure) |
| Compute options | [EC2](../services/ec2.md), Lambda, containers ([ECS](../services/ecs.md)) |
| Storage options | [EBS](../services/ebs.md), [EFS](../services/efs.md), [S3](../services/s3.md) |
| Database options | [RDS](../services/rds.md), [Aurora](../services/aurora.md), DynamoDB, Redshift |
| Caching options | [ElastiCache](../services/elasticache.md) (general), DAX (DynamoDB) |
| App services | [Kinesis](../services/kinesis.md), Step Functions, SQS, SNS |
| Purpose of caching tier | Offload database **reads** — faster and cheaper |
| Purpose of app-services tier | Add functionality / **decouple** components (e.g. SQS queues) |
| Core takeaway | Think global components first, then regional components; most services are **regional**, few are global |

> [!tip] Memory aid (editorial, not from the course)
> Regional tiers, front to back: **W-C-C-D-S-A** — **W**eb, **C**ompute, **C**aching, **D**atabase, **S**torage, **A**pp services. The order is presentational, not a strict request sequence — the compute tier consumes the other tiers as needed.

## References

- Adrian Cantrill, AWS SAA course — Architecture Fundamentals: "Regional and Global AWS Architecture" (introduction / mental-model lesson)
- [AWS Global Infrastructure](https://aws.amazon.com/about-aws/global-infrastructure/)
- [Multi-Region fundamentals whitepaper](https://docs.aws.amazon.com/whitepapers/latest/aws-multi-region-fundamentals/aws-multi-region-fundamentals.html)
