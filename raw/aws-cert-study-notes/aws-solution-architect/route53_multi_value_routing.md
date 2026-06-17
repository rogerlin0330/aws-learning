# Route 53 — Multi-Value Routing

## Problem / Context

You have **many resources that can all service requests** and you want to:

- Return several of them at once,
- Have each one **independently health-checked**, and
- Let the client pick one at random.

Multi-value routing behaves like a **mixture of simple and failover routing** — it takes the multiple-records aspect of simple routing and the health-checking of failover, and merges them into an active-active approach to DNS.

## Solution / Mechanics

Like every policy, it starts with a **hosted zone** containing **multiple records of the same name**. In the example, three `www` A records each map to an IP address.

![[R53-multi-value-routing.png]]

Reading the diagram:

- **Right side:** each record is **independent** and can have an **associated health check** (the small monitor/heartbeat icons). The top two pass (green ticks); the bottom one fails (red cross), so its record (`1.2.3.5`) is struck through. **Any record that fails its health check won't be returned** when queried.
- **Centre / left:** the healthy records flow back to the client (laptop). **Up to 8 healthy records are returned** to the client. If more than 8 healthy records exist, **8 are selected at random**.
- **Top-left:** the **client chooses and uses 1 value** from the set it received, then connects to that resource directly.

Because every record is health-checked, an unhealthy record (bottom one here) is neither returned nor selected by the client — improving availability.

## Not a Load Balancer

⚠️ The top-right callout is the key exam trap: **Multi-value improves availability — it is NOT a replacement for load balancing.** Multi-value routing does **not** handle the actual connection process from a network perspective. It simply returns multiple health-checkable IP addresses; the client does the choosing and connecting. A load balancer manages connections at the network level; multi-value uses DNS to _improve availability_, not to balance load.

## Distinguishing From Simple and Failover

|Policy|Health checks?|Typical use|
|---|---|---|
|Simple|No|A single resource (e.g. one web server)|
|Failover|Yes|Active/backup architectures (commonly an S3 bucket as backup)|
|Multi-value|Yes (per record)|Many resources that can all service requests; health-checked and returned at random|

## Exam Quick-Reference Recap

🎯 **Up to 8 healthy records** returned per query; if more exist, **8 are chosen at random**.

🎯 Each record is **independent** with its **own health check**; failing records are excluded from responses.

🎯 The **client selects one value** and connects to it.

⚠️ **Multi-value ≠ load balancer.** It improves availability via DNS; it does not manage network-level connections.

**Trigger keywords:** "multiple resources all servicing requests", "return several healthy records", "active-active DNS", "health-checked records returned at random", "improve availability (not load balancing)".