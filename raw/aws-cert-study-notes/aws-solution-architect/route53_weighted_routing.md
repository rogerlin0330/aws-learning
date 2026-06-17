# Route 53 — Weighted Routing

## Problem / Context

You have a group of records sharing the same name (e.g. several EC2 instances behind `www`) and you want to control _how often_ each one is returned in response to queries. Two common drivers:

- A **simple form of load balancing** across multiple resources.
- **Testing new software versions** by sending a small slice of traffic (e.g. 5%) to a server running the new build.

## Solution / Mechanics

Weighted routing starts, like every routing policy, with a **hosted zone** containing multiple records of the same name. In the example below, three `www` A records each point at a different IP address (three EC2 instances).

![[R53-weighted-routing.png]]

Each record is assigned a **record weight** (the small dashed callout in the diagram points at the weight column). Here the top record has weight `40`, the middle `40`, and the bottom `20`.

How the weight resolves to a probability:

1. For a given name, Route 53 sums all the weights into a **total weight** — here `40 + 40 + 20 = 100` (shown on the left).
2. Each record is returned **based on its weight versus the total weight**.

With these values, the colour-matched chips at the bottom of the diagram show the result: the top record (`1.2.3.3`) is returned **40%** of the time, the middle (`1.2.3.4`) **40%**, and the bottom (`1.2.3.5`) **20%**. The weights happen to total 100 here so they read directly as percentages, but the _same formula_ (weight ÷ total weight) applies regardless of what the weights sum to.

### Zero weight

Setting a record's weight to `0` means it is **never returned** — useful for temporarily taking a record out of rotation. The exception (left-hand callout): if **all** records are set to `0`, then **all** are considered. This prevents a configuration where nothing could ever resolve.

### Combining with health checks

Weighted routing can be paired with health checks (right-hand callout). The interaction is important and a common trap:

- A record is first selected using the weight calculation above.
- If the selected record is **unhealthy**, it is **skipped** and the selection process **repeats** until a healthy record is chosen.
- Health checks **do not remove records from the calculation** and therefore **do not adjust the total weight**. An unhealthy record is simply skipped over when selected — the denominator stays the same.

## Exam Quick-Reference Recap

🎯 **Use weighted routing for:** simple load balancing, or controlled rollout / testing of new software versions (e.g. send X% of traffic to a new version).

🎯 **Formula:** an individual record is returned based on `record weight ÷ total weight`. Weights don't need to total 100.

⚠️ **Weight `0`** → never returned, **unless every record is `0`**, in which case all are returned.

⚠️ **Health checks don't change the total weight.** An unhealthy selected record is skipped and the process repeats; it is not removed from the denominator.

**Trigger keywords:** "percentage of traffic", "weight", "test new version", "simple load balancing", "control distribution across same-name records".