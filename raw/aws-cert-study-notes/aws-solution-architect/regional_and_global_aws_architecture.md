---

tags:

- aws
- saa
- architecture
- foundations
- global-infrastructure course: Adrian Cantrill AWS SAA section: Architecture Fundamentals

---

# Regional and Global AWS Architecture

> [!info] Introduction lesson Cantrill is explicit that this is an **introduction / mental-model lesson**, not a deep-dive. The goal is to get you thinking about any architecture in terms of its **global** components and its **regional** components, and to see that most applications are assembled from the same set of **tiers**. Later lessons deep-dive each service; here you just build the scaffold.

## Overview

The running example is **Netflix**. The key reframing: Netflix looks like one _global_ application, but it's really **a collection of smaller regional applications** — discrete, independent blocks of infrastructure that are **duplicated across multiple regions** worldwide and together form the global platform.

Cantrill says that as a solutions architect, you'll design **three main types of architecture**:

1. **Single region / single country** — small-scale, only ever exists in one region.
2. **Single region with DR** — normally runs in one region, but if that region fails it **fails over to a secondary region**.
3. **Multi-region** — operates across multiple regions simultaneously and must **keep working through the failure** of one or more of them.

> [!tip] 🎯 The core mental model Think of every system as **a collection of regions making up a whole**. Very few AWS products are truly global — **most run inside a region**, and many regions together make up AWS. Thinking this way makes designing large platforms much easier. For any architecture, keep asking: _which parts give global resilience, and which parts are local/regional only?_

---

## The Two Layers: Global vs Regional Components

### Global components (get the customer to a healthy region)

At the **global** level there are three architectural jobs:

1. **Global service location & discovery** — when a user types `netflix.com`, how does their machine discover _where_ to point? → **DNS**, implemented in AWS by **Route 53**.
2. **Content delivery** — how does content/data reach users globally? Is it cached in pockets around the world, or pulled from one central location? → a **Content Delivery Network (CDN)**, implemented by **CloudFront**.
3. **Global health checks & failover** — detect whether infrastructure in one location is healthy, and move customers to another location/region when needed. → **Route 53 health checks**.

The whole purpose of the global layer is to **get customers to a suitable regional infrastructure location**, isolate any regional failures, move sessions to a working region, ideally direct customers to their _nearest_ region, and improve delivery via caching (CloudFront).

### Regional components (once traffic has entered a region)

Once the global layer has delivered the customer into a specific region, you "zoom in" and think regionally. A regional application is built from **tiers** — high-level groupings of functionality / zones of the application: the **web tier**, **compute tier**, **caching**, **storage**, **database tier**, and **application services**.

---

## Diagram Walkthrough

![[aws-global-and-regional-architecture.png]]

**How to read this diagram (context → main objects → footer):**

**Context (the frame).** The dotted box with the 🚩 flag labelled **"AWS Region"** encloses the regional components. The services drawn **outside** the box on the right are the **global** components. The US map is decorative — it just stands for "one region."

**Global components (right side, outside the box):**

- **Route 53** — shield icon marked **"53"**. DNS service discovery + health checks. Users hit this _first_ to resolve the app's name and be directed to a healthy region.
- **CloudFront** — the globe/network icon at the bottom-right. The global **CDN**; caches content close to users and pulls from an **origin** (e.g. **S3**) as needed. Note the double-headed arrow linking it directly to the **S3 bucket** in the Storage tier — CloudFront uses S3 as an origin for media.
- **The three people + clouds** = the **users / public internet**, entirely outside AWS.

**Regional tiers (vertical colored bands inside the box).** The bands group services by tier; the pink arrows show the request/data flow _between_ tiers:

|Band (color)|Tier|Services shown|
|---|---|---|
|Purple/grey (right, inside box)|**Web Tier**|Application Load Balancer / API Gateway — the regional entry point|
|Orange/tan|**Compute Tier**|EC2 (chip icon), Lambda (λ), Containers/ECS (cube icon)|
|Light blue|**Caching**|ElastiCache / DAX (the ⚡ in-memory cache icon)|
|Blue|**DB Tier**|RDS, Aurora, DynamoDB, Redshift|
|Green|**Storage**|EBS, EFS (cloud/file icons), S3 (bucket, bottom)|
|Pink (far left)|**App Services**|Kinesis, Step Functions, SQS, SNS|

**Flow along the pink arrows:** Users → Route 53 (discovery) + CloudFront (delivery) → **Web Tier** (load balancer, entry point) → **Compute Tier** → out to **Caching → DB**, **Storage**, and **App Services**. CloudFront ↔ S3 directly for content.

> [!warning] ⚠️ Don't over-read the icons This is a deliberately "everything, everywhere" diagram — one representative service per tier. The icons illustrate a tier's _role_; a real design picks one option per tier, not all of them. The exam-relevant takeaway is **which tier** and **whether it's global or regional**, not the specific icon.

---

## The Tiers in Detail

**Web Tier** — the **regional entry point**. Typically a regional AWS service such as **Application Load Balancer** or **API Gateway**. Its job is to **abstract customers away from the underlying infrastructure**, so that infrastructure behind it can **scale, fail, or change without impacting customers**.

**Compute Tier** — provides the actual functionality the customer receives via the web tier. Uses **EC2**, **Lambda**, or **Containers (ECS)**. In the Netflix example the load balancer sends traffic to **EC2**. (Later lessons cover when to pick each compute type.)

**Storage Tier** — consumed by the compute tier. Uses **EBS** (Elastic Block Store), **EFS** (Elastic File System), and **S3** (e.g. media storage). **CloudFront can use S3 as an origin** — Netflix stores movies/shows on S3 and CloudFront caches them for a global audience. Tiers are separate components that consume services from each other, so CloudFront reaching directly into S3 is normal.

**Database Tier** — most environments also need structured data storage: **RDS**, **Aurora**, **DynamoDB**, and **Redshift** (data warehousing).

**Caching Tier** — sits **in front of** the database. Most applications **don't hit the database directly**; they consult a cache first — **ElastiCache** (general) or **DAX / DynamoDB Accelerator** (for DynamoDB). Only on a cache miss is the DB consulted, and the cache is then updated.

> [!tip] 🎯 Why cache in front of the DB Caching is generally **in-memory → cheap and fast**. Databases are **expensive relative to the data volume**. So offload **reads** from the DB into the cache wherever possible to **improve performance and reduce cost**. "Minimize reads to the database" is the trigger phrase for a caching-tier answer.

**Application Services Tier** — a suite of products that provide functionality _to_ applications: **Kinesis**, **Step Functions**, **SQS**, **SNS**. These range from simple functions (email, notifications) to services that **change an application's architecture** — e.g. **decoupling components using queues (SQS)**.

---

## Exam Quick-Reference

|Concept|Key point|
|---|---|
|Three architecture types|Single-region · Single-region + DR failover · Multi-region|
|Global job #1: service discovery|**DNS → Route 53** (resolve name to a healthy region)|
|Global job #2: content delivery|**CDN → CloudFront** (cache near users; origin can be S3)|
|Global job #3: health check / failover|**Route 53 health checks** move sessions to a healthy region|
|Regional entry point|**Web Tier** — ALB / API Gateway (abstracts customers from infra)|
|Compute options|EC2, Lambda, Containers (ECS)|
|Storage options|EBS, EFS, S3|
|Database options|RDS, Aurora, DynamoDB, Redshift|
|Caching options|ElastiCache (general), DAX (DynamoDB)|
|App services|Kinesis, Step Functions, SQS, SNS|
|Purpose of caching tier|Offload DB **reads** → faster + cheaper|
|Purpose of app-services tier|Add functionality / **decouple** components (e.g. SQS queues)|
|Core lesson takeaway|Think **global components → regional components**; most services are **regional**, few are global|

### Mnemonic 📚 _(editorial addition — not from the transcript)_

Regional tiers, front to back: **W-C-C-D-S-A** → **W**eb, **C**ompute, **C**aching, **D**atabase, **S**torage, **A**pp services. (The transcript lists them as: web, compute, caching, storage, database, app services — order is presentational, not a request sequence.)

---

## Reconciliation notes (what changed vs the diagram-only draft)

- ✅ **CloudFront confirmed** as the bottom-right global icon (was previously a 📚 guess) — Cantrill names it directly and pairs it with S3 as origin.
- ✅ **Storage tier** now includes **EBS** (previously omitted); **App Services** named exactly as **Kinesis / Step Functions / SQS / SNS**; **DAX** confirmed for caching.
- ⚠️ **Removed** the "three levels of resilience (Global / Region / AZ)" section and the "I Really Care" (IAM/Route 53/CloudFront) mnemonic — that framing belongs to a _different_ Cantrill lesson and is **not** in this transcript. This lesson's model is **global-vs-regional components** plus the **three architecture types** (single-region / +DR / multi-region). IAM and AZ-level resilience are not discussed here.
- The only remaining editorial (non-transcript) item is the W-C-C-D-S-A mnemonic, flagged 📚 above.