---
tags: [aws, saa-c03, appflow, integration, saas, data-transfer]
course: Cantrill SAA-C03
---

# Amazon AppFlow

## Overview

**Amazon AppFlow** is a **fully managed integration service** — think of it as **middleware** — that exchanges data between applications (SaaS apps and AWS services) using **flows**.

- **Connectors** — how AppFlow talks to an application. Many popular apps are supported out of the box; anything else can be built with the **Custom Connector SDK**.
- **Flow** — the main unit of configuration. A flow = **source connector + destination connector + optional components** (mapping, transformation, filtering, validation).
- **Purpose** — sync data across applications, or aggregate data from different sources to **break down data silos**.

**Example use cases (from the course):**
- Sync **Salesforce** contact records → **Redshift** for analysis
- Copy **Zendesk** support tickets → **S3** for storage/analysis

> [!tip] Exam level
> Cantrill states a **basic architectural understanding** is sufficient for most AWS exams — know *what it is* and *when to pick it*, not configuration detail.

---

## Diagram Walkthrough

![[appflow-architecture.png]]

Read the diagram as a **container (the flow) holding three stages, left → right**, with a footer explaining the building blocks.

1. **Outer pink box — the Flow.** Everything happens *inside* a flow; it's the unit you create and run.
2. **Left — Source Connection (Slack).** Tells AppFlow *how* to reach the source and *which credentials* to use.
3. **Middle dotted box — what the flow does to data in transit:**
   - **Source → Destination field mapping** (squares → pentagons icon): which source fields you want and where each lands in the destination. *Required.*
   - **(Optional) Data transformation**: reshape values en route.
   - **(Optional) Filters and validation** (funnel icon): the many green dashed lines (raw source data) narrow through the funnel into a single blue dotted line — only records that pass filters/checks continue.
4. **Right — Destination Connection (Redshift).** Same role as the source connection, for the target.
5. **Footer — two facts about connections:**
   - Connections store **configuration & credentials** to access applications.
   - Connections are **defined separately from flows**, so one connection can be **reused across many flows**.

> [!note] Concept vs example
> Slack → Redshift is just *an example* pairing. The concept is **connection (reusable, holds auth) → flow (mapping + optional transform/filter/validate) → connection**.

---

## Mechanics

| Component | What it does | Required? |
|---|---|---|
| **Connection** | Stores config + credentials for one app; reusable across flows | Yes (source + destination) |
| **Field mapping** | Maps source fields → destination fields | Yes |
| **Transformation** | Modifies data en route | Optional |
| **Filtering** | Controls *which* records are copied | Optional |
| **Validation** | Checks performed on data en route | Optional |

### Networking
- **Default:** uses **public endpoints** → can interact with public SaaS applications.
- **Private:** can use **AWS PrivateLink** to access private sources, keeping flow data on the AWS network.

📚 **Beyond the transcript** (from AWS docs — review):
- **Flow triggers** — three types: **on demand**, **on a schedule**, or **on an event** from the source SaaS app (only apps that emit change events). Scheduled flows can do a **full** snapshot or an **incremental** transfer (only new/changed records since the last run). ([AWS docs – Flow triggers](https://docs.aws.amazon.com/appflow/latest/userguide/flow-triggers.html))
- **Transform options** include field mapping, merging, **masking**, filtering, and validation; data is encrypted at rest and in transit with AWS-managed or customer-managed keys; up to **100 GB per flow**. ([AWS AppFlow features](https://aws.amazon.com/appflow/features/))
- **Custom Connector SDK** is available in **Java and Python**. ([AWS AppFlow features](https://aws.amazon.com/appflow/features/))
- API terminology: a "connection" is a **connector profile** (e.g. `CreateConnectorProfile`).

---

## Course vs Reality

⚠️ **PrivateLink is not universal.** The course says AppFlow "can work using PrivateLink to access private sources." In practice, private flows are supported only for specific SaaS apps (docs list **Salesforce, Singular, Snowflake, Trend Micro**), and the SaaS account itself must be enabled for PrivateLink. Even in private flows, **metadata calls** (listing objects/fields in the console) still use public endpoints — only the actual flow-run data transfer goes over the VPC endpoints. ([AWS docs – Private flows](https://docs.aws.amazon.com/appflow/latest/userguide/private-flows.html))

No other discrepancies found — the course content is high-level and still accurate.

---

## 🎯 Exam Traps

| Scenario wording | Answer | Why |
|---|---|---|
| "Transfer data from **Salesforce / Zendesk / Slack / SAP / Google Analytics** to **S3 / Redshift** with **no custom code**" | **AppFlow** | Managed SaaS ↔ AWS integration is its core purpose |
| "Data from SaaS must **not traverse the public internet**" | **AppFlow + PrivateLink** (private flow) | Default is public endpoints |
| "Need to connect a SaaS app with **no built-in connector**" | **Custom Connector SDK** | Build your own connector |
| "Reuse the **same credentials** across multiple integrations" | **Connections** | Defined separately from flows, reusable |
| "Mask PII / filter records during transfer" | Flow **transformation / filtering** | Built into the flow, no Lambda needed |

---

## Quick Reference — AppFlow vs Look-alikes

| Service | Best fit | Key distinguisher |
|---|---|---|
| **AppFlow** | **SaaS ↔ AWS** data exchange | Pre-built SaaS connectors, no-code |
| [[DMS]] | **Database** migration/replication | Source/target are databases |
| [[Kinesis]] / [[Amazon Data Firehose]] | **Real-time streaming** ingestion | Continuous high-volume streams |
| [[EventBridge]] | **Event routing** (incl. SaaS partner events) | Routes events, not bulk data sync |
| AWS Glue | **ETL** on data already in AWS / databases | Heavier transformation, Spark-based |

---

## Open Questions (for Roger)

1. Should this link into an **integration/data-movement hub note** alongside [[DMS]], [[Kinesis]], and [[EventBridge]], or stand alone?
2. Are the 📚 additions (triggers, 100 GB limit, SDK languages) worth keeping given Cantrill says high-level is enough, or trim to the course scope?
3. The AWS Glue row in the comparison table has no existing note to link — add a placeholder `[[AWS Glue]]` link or leave as plain text?
