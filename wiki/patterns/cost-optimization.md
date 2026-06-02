---
title: Cost Optimization — Purchase Plans
type: pattern
tags: [cost, savings-plans, reserved-instances, spot, organizations]
certs: [aws-saa]
updated: 2026-05-15
---

# Cost Optimization — Compute Purchase Plans

AWS offers three layered mechanisms for paying less than On-Demand: **Savings Plans**, **Reserved Instances**, and **Spot**. They are not mutually exclusive — a well-tuned account typically uses all three plus On-Demand for unpredictable bursts. The trade-off is uniform: **more commitment = bigger discount**.

> [!note] Scope of this page
> Compute purchase plans only (EC2, Fargate, Lambda) plus the database reservation equivalents. Storage-tier, data-transfer, and architecture-level cost patterns are out of scope here. For the **EC2-specific deep dive** — including Scheduled Reserved Instances, Capacity Reservations (Regional / Zonal / On-Demand), the discount-vs-capacity combination matrix, and Dedicated Hosts vs Dedicated Instances — see [EC2 Purchase Options](ec2-purchase-options.md).

## The Three Mechanisms at a Glance

| Mechanism | Commit on | Term | Typical discount | Capacity reserved? | Best for |
| --- | --- | --- | --- | --- | --- |
| **Savings Plans** | $/hour of compute spend | 1 or 3 yr | Up to **~72 %** | No | Steady compute baseline (EC2/Fargate/Lambda) |
| **Reserved Instances** | Specific instance attributes | 1 or 3 yr | Up to **~72 %** | Optional (Zonal RI / Capacity Reservation) | Predictable EC2 + databases SPs don't cover |
| **Spot** | Nothing | None | Up to **~90 %** | No | Interruption-tolerant batch, CI, stateless workers |

Discount also scales with **payment**: All Upfront > Partial Upfront > No Upfront.

## Savings Plans (the modern default for compute)

Two flavors:

### Compute Savings Plan

- **Most flexible.** Applies automatically to **EC2 (any family, size, region, OS, tenancy)**, **Fargate**, and **Lambda**.
- Up to **~66 %** off On-Demand.
- Use when workload mix may change — switching instance families, moving regions, or migrating EC2 → Fargate keeps the SP active.

### EC2 Instance Savings Plan

- Locked to a specific **instance family in a specific region** (e.g. `m5` in `us-east-1`).
- Flexible across **size** (size-flexibility within the family) and **OS / tenancy**.
- Up to **~72 %** off — biggest SP discount.
- Use when you know your steady-state will stay in one family + region.

You commit to an hourly $ amount; AWS applies the SP to whatever eligible usage gives the **biggest savings each hour**. Anything over the commitment falls back to On-Demand (or your RIs).

> [!tip] Exam Tip
> The exam-pattern between the two SP types: question pins **family/region** → EC2 Instance SP for the deeper discount. Question emphasizes **flexibility across services/families** → Compute SP.

## Reserved Instances (still essential for non-EC2 compute)

Although Savings Plans replaced most EC2 RI usage, RIs remain the **only purchase plan** for several services:

| Service | RI option name |
| --- | --- |
| RDS | Reserved DB Instances |
| ElastiCache | Reserved Cache Nodes |
| Redshift | Reserved Nodes |
| OpenSearch (formerly Elasticsearch Service) | Reserved Instances |
| DynamoDB (provisioned capacity) | Reserved Capacity |

> [!warning] Gotcha
> Savings Plans do **not** apply to RDS, ElastiCache, Redshift, OpenSearch, or DynamoDB. If a question describes a long-running database and asks for cost reduction, the answer is **Reserved Instance / Reserved Capacity**, not a Savings Plan.

### RI Types

| RI type | Discount | Flexibility |
| --- | --- | --- |
| **Standard RI** | Highest (~72 %) | Only **size flexibility** within same family/OS/region — can't change family, OS, tenancy |
| **Convertible RI** | Lower (~54 %) | Can be **exchanged** for an RI of equal or greater value with different family / OS / tenancy |

### Scope dimension (orthogonal to type)

- **Regional RI** (default) — applies to any AZ in the region; no capacity reservation
- **Zonal RI** — pinned to a specific AZ; **does reserve capacity**

For pure capacity reservation without a price commitment, use a stand-alone **On-Demand Capacity Reservation** instead. The discount/capacity combination matrix and the four-way comparison (Regional RI / Zonal RI / On-Demand Capacity Reservation / Savings Plan) live on the [EC2 Purchase Options](ec2-purchase-options.md#capacity-reservations) page.

## Spot Instances

- Spare EC2 capacity at **up to 90 % off**.
- AWS can reclaim with a **2-minute interruption warning**.
- Ideal for: batch processing, ML training, CI runners, rendering, big-data (EMR/Spark), stateless web workers behind a load balancer.
- Pair with **EC2 Auto Scaling mixed-instances policy** or **Spot Fleet** to diversify across pools — spreading across instance types and AZs dramatically lowers interruption frequency.
- Avoid for: stateful single-node services, anything that can't checkpoint, anything with strict per-request SLAs.

> [!tip] Exam Tip
> Pattern keywords for Spot: *"fault-tolerant"*, *"interruption-tolerant"*, *"checkpointing"*, *"batch"*, *"non-critical"*, *"flexible start/end time"*. Pattern keywords *against* Spot: *"long-running stateful"*, *"24×7 critical"*, *"cannot tolerate interruption"*.

## Decision Tree

```text
Is the workload interruption-tolerant?
├── Yes → Spot (often combined with On-Demand baseline)
└── No
    ├── Is it a database/cache/data-warehouse?
    │   └── Reserved Instance / Reserved Capacity (RDS, ElastiCache,
    │       Redshift, OpenSearch, DynamoDB)
    └── Is it general compute (EC2, Fargate, Lambda)?
        ├── Steady baseline?
        │   ├── Locked to one family+region → EC2 Instance Savings Plan
        │   └── Flexible across families/services → Compute Savings Plan
        └── Spiky/unpredictable → On-Demand
```

Most production accounts end up layering:

1. Savings Plan covering the 24×7 floor
2. RIs covering predictable database/cache workloads
3. Spot for batch and dev/test
4. On-Demand for everything else

## Pooling Across Accounts

When accounts share a single payer via [AWS Organizations consolidated billing](../services/organizations.md#consolidated-billing):

- **Savings Plans and Reserved Instances bought in any account apply org-wide.** The hour-by-hour benefit goes to whichever account has the matching usage that maximizes total savings.
- Sharing is **on by default**; the payer can disable RI/SP sharing per account if a business unit must keep its own benefits.
- **Volume-tier discounts** (S3 storage tiers, data transfer) also aggregate — 10 accounts each pushing 10 TB are billed as one 100 TB customer.

For "multiple AWS accounts, maximize cost efficiency" exam questions, this pooling behavior is the entire reason to consolidate billing.

> [!tip] Exam Tip
> An RI bought in a member account that no longer needs it is **not wasted** — by default it floats to whichever sister account has matching usage. This is a frequent right answer to "how do we get more value from underused RIs?"

## Common Exam Traps

- **"Reduce RDS cost long-term"** → Reserved DB Instance, **not** Savings Plan (SPs don't apply to RDS).
- **"Locked to m5.large in us-east-1"** → EC2 Instance Savings Plan (bigger discount) beats Compute SP.
- **"Workload may move from EC2 to Fargate next year"** → Compute SP (survives the migration); EC2 SP wouldn't.
- **"Batch job, can resume from checkpoint"** → Spot.
- **"Three AWS accounts under one company"** → consolidate billing via [Organizations](../services/organizations.md) before buying any RIs/SPs, so benefits pool automatically.

## References

- [EC2 Purchase Options](ec2-purchase-options.md) — EC2-focused deep dive on all 7 options + capacity reservations + tenancy
- [AWS Organizations](../services/organizations.md)
- [EC2](../services/ec2.md)
- [Savings Plans Documentation](https://docs.aws.amazon.com/savingsplans/)
- [Reserved Instances Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-reserved-instances.html)
- [Spot Instances Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-spot-instances.html)
