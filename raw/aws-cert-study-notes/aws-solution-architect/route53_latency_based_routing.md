# Route 53 — Latency-Based Routing

## Problem / Context

You run a **global application** and want Route 53 to return the record that will give a user the **best performance** — i.e. the lowest latency. Use latency-based routing when **optimising for performance and user experience**.

## Solution / Mechanics

Starts with a **hosted zone** and **records sharing the same name** (`www`). The example has three `www` A records pointing at three IPs.

![[R53-latency-based-routing.png]]

The extra piece compared with weighted/multi-value is the **record region** (the dashed callout points at the region column). For each record you specify the **AWS region** where that record's infrastructure lives — here `us-east-1`, `us-west-1`, and `ap-southeast-2`.

🎯 **Latency-based routing supports one record with the same name per AWS region.** You're declaring where the infrastructure for each record is located.

How resolution works (right-hand callout): **AWS maintains a database of latency** between the user's **general location** and the **regions tagged in records**.

1. When a user resolves the name, Route 53 uses an **IP lookup** to determine the user's general location (here, Australia — note the animals/kangaroo cluster on the left).
2. It consults its latency database. The table (top-right) shows the modelled latencies from `AUS`: `us-east-1` = 400, `us-west-1` = 280, **`ap-southeast-2` = 42** (highlighted).
3. The record offering the **lowest estimated latency** is returned. The blue arrow returns the `ap-southeast-2` record (`1.2.3.5`, Australian flag) to the user.

The record returned is the one with the **lowest estimated latency** _and_ that **is healthy**.

### Combining with health checks

If the lowest-latency record is **unhealthy**, the **next lowest latency** healthy record is returned instead.

### Important caveat

⚠️ The latency database is **not real-time**. It is updated in the background and **does not account for local networking issues**. It's an _estimate_ — better than nothing and a significant help for performance, but not a live measurement.

## Latency vs Geoproximity (preview)

Latency-based routing answers with the region of **lowest estimated latency**. Geoproximity (separate note) answers with the **lowest physical distance** and adds an adjustable **bias** — a distinct mechanism even though both feel location-aware.

## Exam Quick-Reference Recap

🎯 **Use for:** performance / user-experience optimisation for global apps.

🎯 Records are tagged with an **AWS region** (one record per name per region). AWS holds a **latency database** between user location and those regions.

🎯 Returns the record with the **lowest estimated latency that is also healthy**; if unhealthy, the **next lowest** is returned.

⚠️ Latency data is **estimated / not real-time** and ignores local network conditions.

**Trigger keywords:** "lowest latency", "performance", "global application", "record region", "latency database", "best estimated latency".