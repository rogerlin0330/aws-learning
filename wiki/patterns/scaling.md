---
title: Scaling — Horizontal vs Vertical
type: pattern
tags: [scaling, horizontal-scaling, vertical-scaling, scale-out, scale-in, elasticity, load-balancer, sessions, stateless, sticky-sessions, ha]
certs: [aws-saa]
updated: 2026-07-09
---

# Scaling — Horizontal vs Vertical

**Scaling** is how a system grows or shrinks in response to changes in customer load. Technically, it means **adding or removing resources from a system** — where a "system" can be a single [EC2](../services/ec2.md) instance, or a fleet of hundreds of thousands working together.

There are two fundamentally different ways to do it, and the choice cascades into every other architectural decision: load balancing, session storage, availability tier, cost shape, and how quickly you can react to load changes.

| Dimension | **Vertical** | **Horizontal** |
| --- | --- | --- |
| Slogan | "Bigger box" | "More boxes" |
| What changes | Size of one instance | Number of instances |
| Cap | Largest available instance | Effectively unlimited |
| Disruption to scale | Restart and downtime | None |
| App design needed | Anything (incl. monoliths) | Stateless + load balancer |

## Vertical Scaling — "Bigger Server"

Increase capacity by **resizing the individual instance**. In [EC2](../services/ec2.md) terms: stop the instance, change the instance type, start it again.

**Example.** An app on a `T3.large` (2 vCPU / 8 GiB) starts to struggle. Resize to `T3.xlarge` (4 vCPU / 16 GiB); if needed, then `T3.2xlarge` (8 vCPU / 32 GiB). Each step roughly doubles resources.

### Pros

| Benefit | Why it matters |
| --- | --- |
| **Simple** | No special architecture required |
| **No application modification** | If an app runs on an instance, it runs on a bigger instance |
| **Works for any app — including monoliths** | A single larger instance scales the entire codebase cleanly |

### Cons

| Limitation | Why it matters |
| --- | --- |
| **Downtime during resize** | Changing the EC2 instance type requires a **stop, modify, start** sequence — customers see disruption |
| **Outage windows only** | Disruption forces scaling into **pre-agreed maintenance windows**; slow to react to rapidly changing load |
| **Non-linear cost** | Larger instances carry a **price premium**; cost increases faster than capacity at the top end |
| **Hard upper cap** | There is always a **maximum instance size**. Vertical scaling can never exceed the largest available instance |

> [!tip] Exam Tip
> Vertical scaling = resize one instance = restart and downtime = scale only in outage windows. The keyword association to remember is *single instance has a size cap*.

> [!warning] Gotcha — resize is a stop/start in disguise
> An EC2 instance-type change is a stop, modify, start operation. The instance may land on a **new host in the same AZ**, which means **instance-store data is destroyed** and the **auto-assigned public IPv4 changes**. See [EC2 — Stop/Start vs Restart](../services/ec2.md#stopstart-vs-restart--these-are-not-the-same).

## Horizontal Scaling — "More Servers"

Instead of growing one instance, **add more instances** (1 becomes 2, then 8, then more), each typically a smaller commodity-sized instance. AWS terminology: **scale out** when adding, **scale in** when removing.

Because there are now many running copies of the application, traffic must be distributed across them, which requires a **load balancer**.

### The Load Balancer

A load balancer is an appliance that sits **between customers and your instances**, distributing incoming requests across all available instances. For a single customer, consecutive requests may land on the **same instance or a different one** depending on the load-balancing algorithm.

In AWS, this is delivered by **[Elastic Load Balancing (ELB)](../services/elb.md)** — Application Load Balancer for HTTP / HTTPS, Network Load Balancer for TCP / UDP. See the [ELB page](../services/elb.md) for the architecture: nodes, listeners, internet-facing vs internal, and cross-zone load balancing.

### The Critical Consideration — Sessions

> *"With horizontal scaling, sessions are everything."*

A **session** is the state of a customer's interaction with the app: login status, video playback position, shopping-cart contents, current step in a multi-page form.

- On a **single server**, sessions live on that server and that's fine.
- With **horizontal scaling**, a customer's consecutive requests can shift between instances. If sessions are stored on the instance, then a request that lands on Instance 1 followed by one on Instance 10 means **a different session or no session at all** — the user appears logged out, the cart looks empty, the wizard restarts. The app becomes **unusable**.

**The fix — off-host sessions.** Store session data **externally** so any instance can serve any user:

| Where to put sessions | AWS service |
| --- | --- |
| In-memory cache (fast, ephemeral) | **ElastiCache** (Redis or Memcached) |
| Durable key-value | **DynamoDB** |
| Shared relational state | **RDS** |

Once sessions are off-host, the instances become **stateless** — interchangeable "dumb" copies of the app. It no longer matters which instance a customer hits because their state lives elsewhere.

This requires **deliberate design and application support**. It is the price of admission for horizontal scaling; once paid, you unlock all of its benefits.

> [!tip] Exam Tip — "stateless" is the keyword
> When a question describes "users get logged out after deploying behind an ALB" or "shopping cart empties intermittently across instances", the answer is almost always **move session state to ElastiCache or DynamoDB** — not "enable sticky sessions" (see the gotcha below).

> [!warning] Gotcha — sticky sessions are not the same fix
> **ALB sticky sessions** (cookie-based session affinity) pin a user to one instance. They sound like a horizontal-scaling fix, but they aren't — if that instance fails or gets scaled in, the user loses state anyway. Sticky sessions are a **migration aid** for legacy apps with on-host sessions; the proper long-term answer is off-host sessions and stateless instances.

### Pros (once sessions are off-host)

| Benefit | Why it matters |
| --- | --- |
| **No disruption while scaling out** | Adding instances doesn't touch existing ones — current customer connections are unaffected |
| **No disruption while scaling in** | Because sessions are external, connections can be drained from a removed instance without logging anyone out |
| **No real upper limit** | Lots of small commodity instances, just keep adding — no single-instance size cap |
| **Often cheaper** | Smaller commodity instances avoid the premium charged for large instances |
| **Granular scaling** | Adding 1 instance to a fleet of 5 is roughly a 20 % increase. Compare to vertical, where large to xlarge is roughly a 100 % jump. The smaller the unit, the finer the granularity |

## Side-by-Side Summary

| Dimension | Vertical Scaling | Horizontal Scaling |
| --- | --- | --- |
| Method | Resize one instance | Add more instances |
| Disruption | Restart and downtime | None (add or remove freely) |
| When you can scale | Outage windows only | Anytime |
| Upper limit | Capped at max instance size | Effectively unlimited |
| Cost profile | Premium at large sizes (non-linear) | Cheaper commodity instances |
| Granularity | Coarse (steps roughly doubling) | Fine (one small instance at a time) |
| App requirements | None — works for monoliths | Load balancer + off-host, stateless sessions |
| Complexity | Simple | Requires design effort |

## Mnemonic — the "Bob" Trick

Picture a worker named Bob:

- **Horizontal scaling = add or remove Bobs.** One Bob, two Bobs, four Bobs, near-infinite Bobs. (Adds *more things*, no real limit.)
- **Vertical scaling = one bigger Bob.** Small Bob, medium Bob, large Bob. (One thing *grows*, until it can't.)

**Many Bobs = horizontal. One giant Bob = vertical.**

## Where This Leads — Elasticity, ELB, Auto Scaling

Horizontal scaling is the foundation that AWS's **elasticity** primitives sit on top of:

1. **[Elastic Load Balancing (ELB)](../services/elb.md)** distributes traffic across the fleet (Application Load Balancer for HTTP / HTTPS, Network Load Balancer for TCP / UDP).
2. **Auto Scaling Groups (ASG)** add and remove instances **automatically** in response to metrics (CPU, request count, custom [CloudWatch](../services/cloudwatch.md) alarms) or schedules.
3. **Stateless application design** is the precondition that makes both useful.

Combine these and the architecture is not only **scalable** but **highly available** (multi-AZ) and **fault-tolerant** (a failed instance is replaced without human intervention). Vertical scaling cannot reach this property — a single instance always inherits the resilience tier of its AZ. See [Overview — Resilience Tiers](../overview.md#resilience-tiers).

## Key Takeaways

1. **Vertical = bigger box; horizontal = more boxes.** The choice cascades into load balancing, session storage, availability, and cost.
2. **Vertical is simple but capped.** No app changes required; works for monoliths. But every resize is a stop, modify, start operation, so scaling happens only in maintenance windows and never beyond the largest instance size.
3. **Horizontal is unlimited but demands stateless design.** Add and remove instances anytime with no disruption, but only if sessions live off-host (ElastiCache, DynamoDB, RDS).
4. **Sticky sessions are not the proper fix** — they are a migration aid for legacy apps. The exam-correct answer for "users logged out across instances" is off-host sessions.
5. **Horizontal scaling unlocks ELB + Auto Scaling Groups**, which together give multi-AZ high availability and fault tolerance — properties vertical scaling cannot provide.

## References

- [EC2 — Stop/Start vs Restart](../services/ec2.md#stopstart-vs-restart--these-are-not-the-same)
- [Overview — Resilience Tiers](../overview.md#resilience-tiers)
- [AWS SAA Exam Notes — Scaling](../certifications/aws-saa.md#scaling--horizontal-vs-vertical)
- Source: `raw/aws-cert-study-notes/aws-solution-architect/ec2_horizontal_and_vertical_scaling.md`
