# Database Migration Service (DMS)

## Overview

The **Database Migration Service (DMS)** is a _managed database migration service_. It's a topic that appears increasingly often on the exam, and the exam heuristic is simple: **any database migration scenario where at least one endpoint is inside AWS → default to DMS**. If the scenario stresses _no downtime_ / _minimal downtime_, that reinforces DMS as the answer.

Database migrations are inherently complex. Done manually, they involve either setting up replication (hard) or taking a point-in-time backup and restoring it to the destination — which then leaves the tricky problem of the changes that occur _between_ the backup and cutover. DMS solves both, and also handles migrations _between different database engines_.

---

## Diagram Walkthrough

![[dms-architecture.png]]

**Context bar (top):** The header names the service — _Database Migration Service (DMS)_. The top caption states the central idea: the **replication instance** performs the migration between **source and destination endpoints**, which store the **connection information** for the source and target databases.

**Left / Right anchors:** A **Source** database (left) and a **Target** database (right). The left caption lists **common DB engine support** — MySQL, Aurora, Microsoft SQL, MariaDB, MongoDB, PostgreSQL, Oracle, Azure SQL, "……" (📚 _the ellipsis matters — this is an illustrative list, not exhaustive; DMS supports 20+ engines_).

**Main objects (center, reading left → right):** Inside the blue **AWS DMS** boundary, data flows through three staged boxes:

1. **Source Endpoint** (yellow) — holds connection details for the source DB.
2. **Replication Instance** (orange box) — an EC2 instance running migration software. Stacked inside it are multiple **Replication Tasks** (one highlighted blue, two greyed) — a single replication instance can run _multiple_ tasks.
3. **Destination Endpoint** (green) — holds connection details for the target DB.

The red arrows trace the data path: Source DB → Source Endpoint → Replication Instance/Task → Destination Endpoint → Target DB.

**Right caption:** The **Schema Conversion Tool (SCT)** "can assist with Schema Conversion" — deliberately positioned _outside_ the DMS boundary, because schema conversion is **not** a native DMS capability.

**Footer (reading order):** The three job types — **Full Load** (one-off migration of all data), **Full Load + CDC** (Change Data Capture; ongoing replication that captures changes), and **CDC Only** (used when you bulk-transfer existing data by an alternative method such as **native tooling**).

---

## Architecture & Mechanics

The service is built from four conceptual pieces:

**Replication instance** — an EC2 instance running migration software, able to communicate with the DMS service. This is the engine that does the work.

**Replication task(s)** — each replication instance runs _one or more_ tasks. A task defines all options for the migration; architecturally its two most important attributes are the **source and destination endpoints**. A task moves data from the source (using source-endpoint details) to the target (using destination-endpoint details).

**Source & destination endpoints** — store the connection information (address, credentials, etc.) that lets the instance and task reach the source and target databases.

🎯 **The one hard restriction:** **one of the endpoints must be within AWS.** DMS _cannot_ migrate between two on-premises databases.

Compatible engines include a wide range — MySQL, Aurora, Microsoft SQL, MariaDB, MongoDB, PostgreSQL, Oracle, Azure SQL, and more.

### Migration (job) types

|Type|What it does|When to use|
|---|---|---|
|**Full Load**|Migrates existing data only; creates tables as required.|You can afford an outage long enough to copy the existing data.|
|**Full Load + CDC**|Full load _plus_ replication of ongoing changes. Runs the full load while capturing source changes; after full load completes, captured changes are applied to the target until a steady state is reached.|**No/minimal-downtime** migrations. At steady state you stop the apps, let remaining changes flow through, then repoint apps at the new target.|
|**CDC Only**|Replicates _only_ data changes.|When it's more efficient to move the bulk data by another method (e.g. Oracle's own export/import tools), then use DMS just to replicate changes from the point of that bulk load.|

🎯 **Exam trigger:** _"no downtime" / "minimal downtime" migration_ → **Full Load + CDC**.

---

## Schema Conversion Tool (SCT)

DMS does **not** natively perform any schema conversion. **SCT** is a _separate, dedicated tool_ whose sole purpose is schema modification/conversion between different database **versions** or different database **engines**. It frequently goes hand-in-hand with DMS migrations.

🎯 **The golden SCT rule:** **SCT is used only when the database _engine_ is changing** (a _heterogeneous_ migration).

⚠️ **SCT is NOT used for migrations between compatible engines.** Example: on-premises **MySQL** → AWS RDS **MySQL**. Even though the _products_ differ (self-managed MySQL vs. RDS), the _engine_ is the same, so **no SCT**.

**SCT works with:**

- **OLTP** databases — e.g. MySQL, Microsoft SQL, Oracle (📚 _illustrative list_).
- **OLAP** / data-warehouse databases — e.g. Teradata, Oracle, Vertica, Greenplum (📚 _illustrative list_).

**Example situations where SCT _is_ used (engine changes):**

- On-premises **Microsoft SQL** → AWS RDS **MySQL** (SQL Server → MySQL).
- On-premises **Oracle** → AWS **Aurora** (Oracle → Aurora).

> ⚠️ 📚 **Transcript vs. current AWS — important discrepancy.** The transcript describes SCT purely as a _standalone desktop application_ you download. That is still accurate and still exam-relevant, but AWS has since added a **fully managed, console-based alternative called _DMS Schema Conversion (DMS SC)_** built directly into the DMS console — no download required. AWS positions the two as **two schema-conversion options**: the downloadable **AWS SCT** (broader engine/feature support, including data warehouses, big-data frameworks, application SQL, and ETL) and the web-based **DMS SC** (fewer platforms, more limited functionality; supports Oracle/SQL Server sources → RDS/Aurora/Redshift targets). For the SAA exam, the classic "SCT = standalone tool for engine changes" framing is what's tested — but be aware the managed option exists in case a question references it. Sources: [DMS Schema Conversion docs](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_SchemaConversion.html); ["A Fully Managed Schema Conversion in AWS DMS"](https://aws.amazon.com/blogs/aws/new-a-fully-managed-schema-conversion-in-aws-database-migration-service/).

---

## DMS + SCT + Snowball (large-scale migrations)

DMS normally transfers data **over the network** — via Direct Connect, VPN, or a VPC peer. But for **multi-terabyte** migrations, moving data over the wire is slow and consumes network capacity needed for normal business operations. For these, DMS can be combined with the **Snowball** range for bulk physical transfer.

**How the combined DMS + SCT + Snowball flow works:**

1. Use **SCT** to extract the data from the source database and store it **locally** in a _generic file format_.
2. Move that data onto a **Snowball** device (ordered from AWS), then ship the device back to AWS.
3. AWS loads the data into an **S3 bucket**.
4. **DMS** migrates the data from **S3** into the target database.
5. If **CDC** is enabled, changes since the initial bulk transfer are also migrated — these likewise pass through **S3 as an intermediary** before DMS writes them to the target.

⚠️ **Why this doesn't break the "SCT only when engine changes" rule:** SCT here converts the source database into a **generic file format** for transport. You _are_ effectively changing the format/engine of the data (from the native source engine into a generic file representation), so the golden rule holds.

📚 _Verified against AWS: the SCT replication/extraction agent copies data to Snowball Edge; the device ships to AWS, data lands in S3, and DMS copies it to the target endpoint — matching the transcript. Source: [Integrating AWS DMS with AWS SCT](https://docs.aws.amazon.com/SchemaConversionTool/latest/userguide/CHAP_DMSIntegration.html)._

---

## Exam Traps / Gotchas

- ⚠️ **Both endpoints on-premises = not allowed.** At least one endpoint _must_ be inside AWS.
- ⚠️ **Same engine ≠ SCT.** Compatible-engine (homogeneous) migrations do **not** use SCT, even if the products differ (self-managed → RDS).
- ⚠️ **DMS has no native schema conversion** — that's exclusively SCT's job.
- ⚠️ **"No downtime" → Full Load + CDC**, not plain Full Load.
- ⚠️ **CDC Only** implies the bulk data is (or will be) moved by _something other than DMS_ (native export/import, etc.).
- ⚠️ **Big / multi-TB migration over constrained network → DMS + SCT + Snowball**, with **S3 as the intermediary**.

---

## Quick Reference

|Concept|Key fact|🎯 Trigger keywords|
|---|---|---|
|**DMS**|Managed DB migration service; one endpoint must be in AWS|"database migration", "migrate DB to AWS", "minimal downtime migration"|
|**Replication instance**|EC2 instance running the migration; runs multiple tasks|"replication instance"|
|**Endpoints**|Store connection info for source & target|"source/target endpoint"|
|**Full Load**|Existing data only; outage acceptable|"one-off", "can afford downtime"|
|**Full Load + CDC**|Existing data + ongoing changes|"no downtime", "minimal downtime", "keep source live"|
|**CDC Only**|Changes only; bulk moved by other means|"already loaded data", "native export/import"|
|**SCT**|Standalone (or managed DMS SC) schema conversion; **only when engine changes**|"different database engine", "Oracle → Aurora", "SQL Server → MySQL", "heterogeneous"|
|**SCT + Snowball**|Bulk multi-TB transfer via generic file → Snowball → S3 → DMS|"multi-terabyte", "too large for network", "Snowball"|
|**Homogeneous migration**|Same engine — **no SCT**|"MySQL → RDS MySQL", "same engine"|

---

## Source Attribution

Primary source: Adrian Cantrill AWS SAA course — _Database Migration Service (DMS)_ lesson transcript (an extension of the Associate Architect course coverage).

**Discrepancies flagged against current AWS documentation:**

- The transcript presents **SCT** solely as a downloadable standalone application. Current AWS additionally offers a **fully managed, console-based DMS Schema Conversion (DMS SC)** as an alternative (flagged inline above). The classic standalone-SCT framing remains exam-relevant; the managed option is noted for completeness.
- All other transcript facts (engine support, restriction that one endpoint be in AWS, the three job types, the SCT engine-change rule, and the DMS + SCT + Snowball → S3 flow) were **verified consistent** with current AWS documentation.