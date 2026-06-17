# Route 53 Failover Routing

## Context / Problem

You have a single DNS name (e.g. `www`) that should normally resolve to a primary resource, but you need an automatic fallback to a backup resource when the primary becomes unavailable. This is the classic **active/passive failover** requirement: send users to the working resource, and divert them to a standby only when the original fails.

## Solution / Mechanics

Failover routing lets you create **multiple records of the same name** within a hosted zone, each assigned a failover role:

- A **Primary** record pointing at the main resource (e.g. an EC2 instance running the application).
- A **Secondary** record of the same name pointing at a different resource (e.g. an S3 bucket).

The defining element is a **health check**, which is generally associated with the **primary** record:

- If the health check target is **healthy**, queries to the name resolve to the **primary** record's value.
- If the primary **fails its health check**, queries return the **secondary** record's value of the same name.

Because the primary and secondary point at _different services_ (EC2 vs S3 in this example), failover is commonly used to build an **"out-of-band" failure/maintenance architecture** — the backup is intentionally a separate, independent service so it can serve a maintenance or static fallback page even when the primary's underlying platform is the thing that's broken.

## Diagram Walkthrough

![[R53-failover-routing-architecture.png]]

The diagram is bounded by the dotted box representing the **`animals4life.org` Hosted Zone** (labelled bottom-center), with the client (user + laptop) on the left and the two target resources at the top and bottom.

**Inside the hosted zone (center)**, two records share the name `www`:

- `*Primary` — the orange `WWW` record
- `*Secondary` — the pink `WWW` record

A **health check** (heart icon, top-right) is attached to the **primary** `www` record — the pink dotted "www health check" arrow runs from the health check back to the orange primary record, and the health check itself probes the EC2 instance (pink dotted arrow up to the chip/Catagram icon).

**Healthy path (orange):** When the health check reports **Healthy**, the primary record is used. Follow the orange arrows — the client's `www` query resolves to the orange primary record, which points up to the **EC2 instance** (top, the chip icon running the Catagram app). This matches the top-left callout: _"If the target of the health check is 'Healthy' the Primary record is used."_

**Unhealthy path (pink):** When the health check reports **Unhealthy**, the secondary record of the same name is returned instead. Follow the pink arrows — the query now resolves via the pink secondary record down to the **S3 bucket** (bottom). This matches the right-side callout: _"If the target of the health check is Unhealthy .. any queries return the secondary record of the same name."_

The color coding is the key to reading this diagram: **orange = primary/healthy flow**, **pink = secondary/failover flow**, and the **heart icon = the health check that decides between them**.

## Use Case

Use failover routing when you need **active/passive failover** — route traffic to a resource while it is healthy, and to a _different_ resource when the original is failing its health check. The canonical example is serving an "out-of-band" failure or maintenance page (e.g. EC2 primary → S3 static fallback).

## 🎯 Exam Quick-Reference

|Aspect|Detail|
|---|---|
|**Policy purpose**|Active/passive **failover**|
|**Records**|Multiple records of the **same name**: one **Primary**, one **Secondary**|
|**Decision mechanism**|**Health check** (typically on the primary record)|
|**Healthy primary**|Resolves to **primary** record value|
|**Unhealthy primary**|Resolves to **secondary** record value|
|**Common pattern**|"Out-of-band" failure/maintenance page (e.g. EC2 → S3)|

**Trigger keywords:** "active/passive failover", "primary and secondary", "failover to a backup/maintenance page when the resource is unhealthy", "different resource when the original fails its health check".

⚠️ Don't confuse with **multi-value** or **weighted** routing: failover is specifically _active/passive_ (one resource at a time based on health), not load distribution across multiple healthy resources.