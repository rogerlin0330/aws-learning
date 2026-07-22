---
title: EC2 Purchase Options
type: pattern
tags: [ec2, purchase-options, on-demand, spot, reserved-instances, standard-ri, convertible-ri, scheduled-reserved, capacity-reservations, zonal-ri, regional-ri, savings-plans, dedicated-hosts, dedicated-instances, tenancy, cost, byol]
certs: [aws-saa]
updated: 2026-07-21
---

# EC2 Purchase Options

AWS officially calls these **purchase options** — you'll often hear *launch types*, but **purchase options** is the canonical term and the one to recognize on the exam. They control three independent things at once: **how you pay** (per-second vs commitment vs $/hr), **how much you pay** (discount vs full price), and **what you get in return** (capacity guarantee? tenancy? specific hardware?).

The mental model: **start with On-Demand Instances as the default, then move to an alternative only if you can justify it** on cost, capacity, or licensing grounds. For the exam, the three to know deeply are **On-Demand Instances, Spot Instances, and Reserved Instances**. [Dedicated Hosts](#dedicated-hosts) and [Dedicated Instances](#dedicated-instances) matter for tenancy/licensing edge cases. [On-Demand Capacity Reservations](#capacity-reservations) and [Savings Plans](#savings-plans) are the modern flexibility layers on top.

> [!note] Scope
> This page is the EC2-focused deep dive. For the broader compute purchase landscape — including non-EC2 services like RDS, ElastiCache, and DynamoDB — see [Cost Optimization](cost-optimization.md). For the underlying tenancy primitives (Shared / Dedicated Instance / Dedicated Host as host-level concepts), see [EC2 — Hosts vs Instances](../services/ec2.md#hosts-vs-instances).

## All Options at a Glance

| Purchase option | Commitment | Discount | Capacity reserved? | Interruption risk | Tenancy | Primary use |
| --- | --- | --- | --- | --- | --- | --- |
| **On-Demand Instances** | None | None | No | None (barring HW failure) | Shared | Default; short or unknown duration |
| **Spot Instances** | None | Up to ~90% | No | **Yes** — AWS can interrupt for capacity or price; terminate/stop/hibernate depends on configuration | Shared | Interruption-tolerant, stateless, parallel work |
| **Standard Reserved Instances (Regional)** | 1 or 3 yr | Up to ~72% | No | None | Shared | Steady, long-term workloads; AZ flexibility |
| **Standard Reserved Instances (Zonal)** | 1 or 3 yr | Up to ~72% | **Yes — one AZ** | None | Shared | Steady, long-term workloads with a capacity guarantee |
| **Convertible Reserved Instances** | 1 or 3 yr | Up to ~54% | Optional (Regional = no, Zonal = yes — same scope choice as Standard) | None | Shared | Steady workload whose family/OS/tenancy may change; exchangeable for an RI of equal or greater value |
| **Scheduled Reserved Instances** *(retired 2024-09-03)* | 1 yr min | Modest vs On-Demand | Yes (in the window) | None (in the window) | Shared | Recurring time-window workloads — **discontinued; use On-Demand Capacity Reservations** |
| **On-Demand Capacity Reservations** | None (immediate-use) | None | **Yes — one AZ** | None | Shared | Pure capacity guarantee with no term lock-in |
| **Savings Plans** | 1 or 3 yr | Up to ~72% (EC2-only) / ~66% (Compute) | No | None | Shared | Flexible $/hour commitment across EC2/Fargate/Lambda |
| **Dedicated Instances** | None | Premium per-Region fee | No | None | Hardware not shared with other customers | Compliance: no co-tenant hardware, host opaque |
| **Dedicated Hosts** | Per host | Pay-per-host | Inherent (you own the host) | None | Whole host | Socket/core-based BYOL licensing |

> [!tip] The single decision question
> **"Can this workload tolerate interruption?"** — Yes → Spot Instances. No → keep reading. *"Is it steady and long-term?"* — Yes → Reserved Instances (Standard for max discount; Convertible if family/OS/tenancy may change) or Savings Plans. *"Specific recurring time window?"* — historically Scheduled Reserved Instances (retired); now an On-Demand Capacity Reservation. *"Need a per-socket license tied to physical hardware?"* — Dedicated Host. *"Need 'no other customer on my host' but don't care about owning it?"* — Dedicated Instance. *"None of the above?"* — On-Demand Instances.

## On-Demand Instances

The default. Deliberately unremarkable — **On-Demand Instances** have no strong pros or cons, which is exactly why they're the baseline you compare every other option against.

![EC2 On-Demand purchase option diagram: two EC2 hosts (Host A and Host B) each holding a mix of instance sizes (2xl, 4xl) shaded in two different colours representing different AWS customers coexisting on the same physical hosts; one 2xl is highlighted to illustrate the "isolated but shared" guarantee — instances are walled off even though the underlying metal is shared; blank space on each host represents unused/wasted capacity; a stopwatch on the left depicts per-second billing while the instance is running with a reminder that storage and other associated resources bill regardless of instance state](../assets/ec2-purchase-option-on-demand.png)

### How On-Demand Instances work

- Instances run on a **shared pool of EC2 hosts**. Different customers' instances coexist on the same hardware, isolated by the hypervisor — the model that lets AWS price the baseline reasonably.
- Instances of different sizes share the same hosts, each consuming a defined allocation of host resources.

### On-Demand Instance billing

- **Per-second compute billing while the instance is running.** Stop the instance and compute charges stop.
- **No capacity reservation by default.** Under a Regional or AZ capacity shortage, an On-Demand launch can fail unless you already hold reserved capacity through a [Zonal Reserved Instance](#scope-zonal-vs-regional-reserved-instances) or [On-Demand Capacity Reservation](#capacity-reservations).
- No discount, no upfront commitment.

> [!warning] Gotcha — stopped ≠ free
> Associated resources (notably attached [EBS](../services/ebs.md) volumes) keep billing *regardless of instance state*. Compute charges stop on shutdown; storage does not. See [EC2 — State-Based Costs](../services/ec2.md#what-changes-when-the-state-changes).

### Good for (On-Demand Instances)

- Short-term workloads (provision → run → terminate).
- Workloads of **unknown** duration or shape.
- Short-term/unknown workloads that **cannot tolerate interruption**.

## Spot Instances

**Spot Instances** are the **cheapest** way to get EC2 capacity — **up to ~90% off** On-Demand — by buying spare host capacity that would otherwise sit idle. The trade-off is the defining one: **Spot capacity is not reliable.**

![EC2 Spot purchase option diagram: two EC2 hosts with a vertical price ladder on the left; the bottom pink tier carries a banknote icon labelled ON-DEMAND representing the normal, reliable, higher price; grey tiers above are spare capacity sold as Spot, marked with coins instead of a banknote because they are cheaper; a red arrow points to the current published spot price at the top; two coin stacks below illustrate two customers' maximum prices (a 2-coin max and a 4-coin max); when capacity is available, both pay the same current Spot price, never their maximum; when AWS needs capacity back or price exceeds a configured maximum, a Spot Instance can be interrupted](../assets/ec2-purchase-option-spot-instances.png)

### How the Spot price works

- For each instance type, in each Region, there is some amount of unused host capacity at any moment.
- AWS publishes a **Spot price** for that spare capacity; the price rises and falls with how much spare capacity is available.
- You set a **maximum price** you are willing to pay. This is a *ceiling*, **not what you pay** — you always pay the **current Spot price**, never more than your max.
- **AWS can interrupt Spot Instances** when it needs the capacity back, or when the Spot price rises above a maximum price that you explicitly configured. Interruption behavior can be **terminate**, **stop**, or **hibernate**, depending on the Spot request/instance configuration.

### Key consequence of the pricing model

Two customers running the same instance type in the same Region pay the *same* current Spot price, regardless of their differing maximums. A **higher maximum buys you more price tolerance before interruption — not a different rate**. It does not protect you from capacity reclaim.

### Good for (Spot Instances — anything interruption-tolerant)

- Non-time-critical work.
- Highly parallel workloads splittable into many pieces (e.g. scientific analysis) where failed pieces can be rerun.
- Bursty capacity needs (e.g. media or image processing).
- Cost-sensitive workloads that wouldn't be economical on On-Demand Instances.
- **Stateless** workloads (no user/session state stored on the instance).

### Anti-patterns — never use Spot Instances for

- Anything that **cannot tolerate interruption**.
- Long-term, consistent, reliable compute.
- Business-critical systems — domain controllers, mail servers, traditional websites, flight-control-style systems.

> [!tip] Exam Tip
> Spot Instance pattern keywords: *"fault-tolerant"*, *"interruption-tolerant"*, *"checkpointing"*, *"batch"*, *"non-critical"*, *"flexible start/end time"*. Anti-pattern keywords: *"long-running stateful"*, *"24×7 critical"*, *"cannot tolerate interruption"*.

## Reserved Instances

**Reserved Instances** are for **long-term, consistent** EC2 usage. Where On-Demand Instances handle unknown/short-term workloads, **Reserved Instances handle the parts of your infrastructure that are always on and never change**.

![EC2 Reserved Instances diagram: top timeline shows the commitment — reservations run for 1- or 3-year terms with a crossed-out middle block indicating you cannot bail out partway; the middle row has four boxes showing the application logic — (1) no reservation means full per-second price; (2) a reservation matching the instance means reduced or zero per-second price; (3) an unused reservation is still billed (empty dashed box); (4) a reservation applied to a larger same-family instance gives partial coverage; the bottom row depicts the three payment structures ordered by increasing discount — No-Upfront (per-second fee only), Partial Upfront (some coins plus a reduced per-second fee), and All Upfront (a treasure chest, no per-second fee)](../assets/ec2-purchase-option-reserved-instances.png)

### How Reserved Instances work

- A **commitment** to AWS to consume EC2 resources over a fixed term. In return, you get a reduced (or eliminated) per-second cost.
- The reservation applies to a matching instance, reducing or removing its per-second price.
- **Partial application:** a reservation for a smaller instance (e.g. a `T3.large`-sized reservation) applied to a larger instance of the same family gives a *partial* discount covering an equivalent portion of the larger instance.

> [!warning] Gotcha — committed = billed even if unused
> Once committed, **you pay whether you use it or not.** An unused reservation is wasted money — plan reservations carefully. Within an [AWS Organization](../services/organizations.md), an unused Reserved Instance in one account automatically floats to a sibling account with matching usage by default, which is the main mitigation.

### Scope: Zonal vs Regional Reserved Instances

This is the orthogonal "scope" dimension — it determines whether the reservation also reserves capacity.

| Scope | Discount applies to | Reserves capacity? | Flexibility |
| --- | --- | --- | --- |
| **Zonal Reserved Instances** | Instances in **one specific AZ** | **Yes** — in that AZ | None — locked to AZ + instance type |
| **Regional Reserved Instances** | Instances in **any AZ in the Region** | **No** | AZ flexibility + instance-size flexibility within the family |

> [!tip] Mental model
> A Reserved Instance is fundamentally a **billing discount** applied to matching On-Demand usage — not a capacity object on its own. A **Zonal Reserved Instance** happens to bundle in a capacity reservation in its AZ; a **Regional Reserved Instance** does not. The dedicated [On-Demand Capacity Reservation](#on-demand-capacity-reservations) is the standalone way to reserve capacity with no term commitment.

### Term length

- **1 year** or **3 years.** Longer term = bigger discount, but more risk that your needs change.

### Payment structure (more upfront = bigger discount)

| Payment | Upfront cash | Per-second fee | Notes |
| --- | --- | --- | --- |
| **No Upfront** | None | Reduced, paid running or not | Least discount; no cash-flow hit |
| **Partial Upfront** | Smaller lump sum | Further reduced | Balanced middle ground |
| **All Upfront** | Entire term | **Zero** | Greatest discount |

> [!tip] The biggest possible Reserved Instance discount
> **3-year term + All Upfront** is the largest Reserved Instance discount AWS offers.

### Reserved Instance offering classes — Standard vs Convertible vs Scheduled

Reserved Instances come in three offering classes. **Standard** and **Convertible** are the two current offering classes and trade discount for exchange flexibility; **Scheduled** was a separate product that has been retired.

| Offering class | Max discount | Exchange / change rules | Best for |
| --- | --- | --- | --- |
| **Standard Reserved Instances** | up to **~72%** | Cannot exchange for a different family, OS, or tenancy. **Size flexibility** within the same family is permitted | Workload is **locked in** for the full 1- or 3-year term — cheapest option |
| **Convertible Reserved Instances** | up to **~54%** | Can be **exchanged** for one or more RIs with a different family / OS / tenancy / payment option, as long as the new RIs are of **equal or greater value** than the current one | Workload mix may shift — you trade ~18 percentage points of discount for the right to switch |
| ~~**Scheduled Reserved Instances**~~ *(retired 2024-09-03)* | ~5–10% (legacy) | Recurring time window (frequency + duration + window) | **Retired** — use [On-Demand Capacity Reservations](#on-demand-capacity-reservations) for new recurring-window designs |

Both **Standard** and **Convertible** can be purchased with either **Regional** or **Zonal** [scope](#scope-zonal-vs-regional-reserved-instances) — the offering-class axis (Standard vs Convertible) is independent of the scope axis (Regional vs Zonal).

> [!tip] When to pick Convertible over Standard
> A 1-year Standard at ~40% vs 1-year Convertible at ~31% is not a free lunch — you give up roughly an eighth of your discount. Reach for Convertible when there is a **real** chance the workload moves families (e.g. migrating from M-family to Graviton, switching OS, or moving from Shared to Dedicated tenancy mid-term). For workloads that are genuinely steady-state, Standard wins on cost.

> [!warning] Convertible exchange must be equal or greater value
> You cannot **downgrade** a Convertible RI via exchange — the new RI(s) must equal or exceed the current commitment's value. This means Convertible protects against *changes in workload shape*, not against shrinkage.

> [!warning] Scheduled Reserved Instances are discontinued
> AWS retired **Scheduled Reserved Instances** on **2024-09-03** — they cannot be purchased and AWS has no plans to bring them back. The replacement for recurring-window capacity guarantees is **On-Demand Capacity Reservations** (no term commitment, billed at full On-Demand); for discounting on top, layer in a **Savings Plan**. The section is kept here because older study material — including some SAA exam questions — still references the product.
>
> *Source: the AWS launch post [New – Scheduled Reserved Instances (2016-01-13)](https://aws.amazon.com/blogs/aws/new-scheduled-reserved-instances/) now carries an in-post 2024-09-03 update with the retirement notice. AWS gave no explicit reason, but the original launch shipped with only a **5–10% discount** and supported only the legacy **C3 / C4 / M4 / R3** families — narrow enough that newer mechanisms (Savings Plans in 2019, On-Demand Capacity Reservations) made the product redundant.*

![EC2 Scheduled Reserved Instances diagram: three illustrations of recurring patterns this option covers — daily (batch processing for 5 hours starting at 23:00), weekly (sales analysis every Friday for 24 hours), and monthly (100 hours of EC2 per month); shaded slivers represent the reserved time blocks within each repeating period; right-hand note restates the original constraints — not all instance types/Regions supported, minimum 1,200 hours/year, minimum 1-year commitment](../assets/ec2-purchase-option-scheduled-reserved-instances.png)

> [!note] Historical Scheduled Reserved Instance restrictions
> When it was available: didn't support all instance types/Regions; minimum **1,200 hours/year**; minimum commitment **1 year**. Useful if a question describes the legacy product.

### Good for (Reserved Instances)

- Components with **known, consistent, long-term** usage that need the lowest cost and **can't tolerate interruption**.

## Capacity Reservations

When a Region or AZ runs short on capacity (e.g. after a major failure), AWS hands out remaining EC2 capacity in a fixed **priority order** — so business-critical workloads that *must* be launchable need a capacity reservation, not just a discount.

![EC2 Capacity Reservations diagram: a single region with two AZs (AZA and AZB); a Regional Reservation arrow at the top gives a billing discount to valid instances in any AZ but reserves no capacity — flagged as risky during major faults when capacity is scarce; a Zonal Reservation arrow on the lower-left gives both the billing discount and reserved capacity but only in that one AZ; instances elsewhere get full price and no capacity reservation; treasure-chest + instance pairs in AZB illustrate On-Demand Capacity Reservations — capacity booked in a specific AZ that you pay for at full On-Demand price whether consumed or not, with one greyed-out instance representing reserved-but-unused capacity you're still billed for; battery icons (full vs near-empty) hint at how much guaranteed capacity each approach leaves you during a shortage](../assets/ec2-purchase-option-capacity-reservations.png)

### Capacity assurance under shortage

The old shortcut "Reserved Instances first, then On-Demand, then Spot" is too broad. **Discount alone is not capacity.** What matters is whether capacity is actually reserved in the AZ where you need to run.

| Situation | Capacity effect |
| --- | --- |
| **Zonal Reserved Instance** | Discount + capacity reservation in one AZ |
| **Regional Reserved Instance** | Discount only; no launch guarantee |
| **On-Demand Capacity Reservation** | Capacity reservation in one AZ; no discount by itself |
| **On-Demand launch without a reservation** | Uses available On-Demand capacity; can fail during shortage |
| **Spot Instance** | Uses spare capacity only; can be interrupted |

This is *the* reason capacity reservations exist: a discount alone gives you no claim on scarce capacity.

### The two separate components

A reservation has a **billing** component and a **capacity** component — and they can be used together or independently. Recognising this is the key to the whole section.

|                                     | Billed if idle? | Reserves capacity? | Term lock | Discount |
| ----------------------------------- | --------------- | ------------------ | --------- | -------- |
| **Regional Reserved Instance**      | Yes             | No                 | 1 or 3 yr | Yes      |
| **Zonal Reserved Instance**         | Yes             | Yes (one AZ)       | 1 or 3 yr | Yes      |
| **On-Demand Capacity Reservation**  | Yes             | Yes (one AZ)       | None      | No       |

The common thread: **all three are billed when idle**. What differs is what you get for it.

### On-Demand Capacity Reservations

A standalone capacity booking with **no term commitment** and **no built-in discount**.

- Books capacity in a **specific AZ**.
- **Pay full On-Demand price** for the booked capacity whether you use it or not.
- For **immediate-use** reservations: create, modify, and cancel freely.
- For **future-dated** reservations: a commitment duration applies before you can cancel.
- **Active or unused, the capacity counts toward your per-Region On-Demand Instance limits.**

### Combining discount + capacity

Because *discount* and *capacity* sit on separate axes, you can mix and match. The four practical combinations:

| Setup | Discount? | Capacity reserved? | Notes |
| --- | --- | --- | --- |
| **Regional Reserved Instance** alone | Yes | No | Flexible across AZs/sizes; no capacity guarantee |
| **On-Demand Capacity Reservation** alone | No | Yes (one AZ) | Capacity guaranteed; billed at full On-Demand |
| **Regional Reserved Instance + On-Demand Capacity Reservation** | Yes | Yes (one AZ) | The recommended combo to get *both* |
| **Zonal Reserved Instance** | Yes | Yes (one AZ) | Both in one purchase; locked to one AZ |

Key rules for the combined case:

- A **Regional Reserved Instance discount automatically applies to a matching On-Demand Capacity Reservation** — this is a supported, AWS-recommended pattern. Savings Plans work the same way.
- **Discounts hit running instances first**, then cover unused reserved capacity. Either way you pay the discounted rate, not full On-Demand.

> [!warning] Trap — discount-stacking only works with Regional Reserved Instances
> A **Zonal Reserved Instance's discount does not apply to a separate On-Demand Capacity Reservation**, because the Zonal Reserved Instance already carries its own bundled capacity reservation. The combine-discount-with-capacity pattern is **Regional Reserved Instance + On-Demand Capacity Reservation** (or Savings Plan + On-Demand Capacity Reservation).

### AWS's own four-way comparison

AWS publishes a table on the [On-Demand Capacity Reservations page](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-capacity-reservations.html) that adds **term** and **instance-limit** dimensions on top of the discount/capacity split:

| | On-Demand Capacity Reservation | Zonal Reserved Instance | Regional Reserved Instance | Savings Plan |
| --- | --- | --- | --- | --- |
| **Term** | None for immediate-use (create/modify/cancel freely); future-dated ones carry a commitment duration | Fixed 1- or 3-year | Fixed 1- or 3-year | Fixed 1- or 3-year |
| **Capacity benefit** | Capacity reserved in one AZ | Capacity reserved in one AZ | No capacity reserved | No capacity reserved |
| **Billing discount** | None (†) | Provides a discount | Provides a discount | Provides a discount |
| **Instance limits** | Your per-Region On-Demand Instance limits apply | Default 20 per AZ (increasable) | Default 20 per Region (increasable) | No limit |

† An On-Demand Capacity Reservation can be combined with a Savings Plan or Regional Reserved Instance to pick up a discount.

![AWS official comparison table: On-Demand Capacity Reservations vs Zonal Reserved Instances vs Regional Reserved Instances vs Savings Plans — rows for term, capacity benefit, billing discount, instance limits; same data as the table above, sourced from the AWS On-Demand Capacity Reservations docs page](../assets/ec2-purchase-option-comparison.png)

> [!warning] Capacity reservations still need a planning exercise
> Booked-but-unused capacity is billed. If usage turns out to be consistent and long-term, re-evaluate whether a [Reserved Instance](#reserved-instances) purchase or [Savings Plan](#savings-plans) would be cheaper.

## Savings Plans

**Savings Plans** are a **commitment based on hourly spend** rather than a specific instance type/AZ/Region. You commit to a minimum **$/hour for 1 or 3 years** and get a discounted rate up to that committed spend; usage beyond the commitment falls back to normal On-Demand rates.

Two types:

| Plan type | Max discount | Applies to |
| --- | --- | --- |
| **Compute Savings Plans** | up to **~66%** | EC2 (any family/Region/OS/tenancy), Fargate, Lambda |
| **EC2 Instance Savings Plans** | up to **~72%** | Specific **instance family in a specific Region**; flexible across size/OS/tenancy |

**Choosing:** if you consume only EC2 and your steady-state is one family in one Region, the EC2 Instance Savings Plan saves more. If you're adopting containers or serverless — or plan to migrate EC2 → Fargate → Lambda over time — Compute Savings Plans let you commit once and keep the benefit through the migration.

Unlike Reserved Instances, Savings Plans **do not have a capacity component** — to get capacity guarantees you must pair them with an [On-Demand Capacity Reservation](#on-demand-capacity-reservations).

> [!note] Exam scope
> The SAA exam only requires you to know Savings Plans exist and roughly how they work. The decision tree, Reserved Instance alternatives for non-EC2 services, and pooling rules across an AWS Organization are covered in [Cost Optimization](cost-optimization.md).

## Dedicated Hosts

**Amazon EC2 Dedicated Hosts** allocate an **entire physical EC2 host to you**, designed for a specific instance family (e.g. A, C, R).

![EC2 Dedicated Hosts diagram: the host is described in physical terms (e.g. R5, 2 sockets, 48 cores) — the point being that licensing is tied to the physical sockets/cores, not the instance's allocation; a red X shows that you cannot launch beyond the host's physical capacity so you must manage that capacity yourself; a green refresh icon represents Host Affinity, the feature that keeps instances linked to the same host across a stop/start; bottom legend reads "pay for the host, no per-instance charges"](../assets/ec2-purchase-option-dedicated-hosts.png)

### How Dedicated Hosts work

- You **pay for the host.** Instances launched on it have **no per-second charge**.
- You can run various instance sizes up to the host's full physical resource capacity.
- You **manage capacity yourself** — once the host is full, you cannot launch more instances on it.
- **Host Affinity** can link instances to a specific host so they return to the same host after a stop/start.

### Why use Dedicated Hosts

- **Primarily for socket- or core-based software licensing** (BYOL) tied to physical machine resources rather than instance allocation — Windows Server, Oracle, SQL Server. **This is the main exam reason.**
- Also gives hardware isolation (only your instances run on the host).

## Dedicated Instances

**Amazon EC2 Dedicated Instances** are a **middle ground** between Shared tenancy and Dedicated Hosts.

![EC2 Dedicated Instances comparison diagram: three tenancy models side by side — Default/Shared shows your instances (coloured blocks) sitting alongside other customers' on the same host with no host exposure and no capacity to manage; Dedicated Host shows the whole host as yours (one large block) where you pay for the host and manage its capacity; Dedicated Instances shows hardware running only your instances (no other customers) but you don't own or manage the host, accepting extra charges in exchange for dedicated hardware](../assets/ec2-purchase-option-dedicated-instances.png)

### How Dedicated Instances work

- Your instances run on hardware that **no other customer shares**, but you **don't pay for or manage the host** itself.
- AWS commits to never placing another customer's instances on that hardware.
- **Fees:** a one-off **per-Region hourly fee** while you're using Dedicated Instances (regardless of count), plus a fee for the instances themselves.

### Why use Dedicated Instances

- Strict requirements that **prohibit hardware sharing**, where you want EC2's usual benefits and guaranteed hardware isolation **but don't want to manage a host**.

## Tenancy / Hosting Model Summary

This is the host-level view that underpins On-Demand Instances, Spot Instances, and Reserved Instances — and contrasts them against the two dedicated-tenancy options. See [EC2 — Hosts vs Instances](../services/ec2.md#hosts-vs-instances) for the deeper treatment.

| Model                   | Hardware sharing                        | You pay for                | Capacity to manage   | Typical reason                                                |
| ----------------------- | --------------------------------------- | -------------------------- | -------------------- | ------------------------------------------------------------- |
| **Shared (default)**    | Shared with other customers             | Instances (per-second)     | None                 | On-Demand Instances, Spot Instances, Reserved Instances       |
| **Dedicated Instances** | Your instances only, no other customers | Instances + per-Region fee | None                 | Can't share hardware, don't want to manage a host             |
| **Dedicated Hosts**     | Your instances only (whole host)        | The host                   | Yes (host is finite) | Socket/core-based licensing                                   |

The Shared model underpins ~99% of usage. Dedicated Instances and Dedicated Hosts exist for the exceptions — compliance and BYOL licensing.

## Quick Decision Recap

- **Default everything to On-Demand Instances**, then justify a move away.
- **Spot Instances** → cheapest, but interruptible. Only for interruption-tolerant, re-runnable, stateless, or cost-sensitive work.
- **Standard Reserved Instances** → known, consistent, long-term, can't-be-interrupted workloads needing the lowest cost.
- **Convertible Reserved Instances** → same shape as Standard but the workload's family/OS/tenancy may shift mid-term — accept a smaller discount in exchange for the right to swap.
- ~~**Scheduled Reserved Instances**~~ → **retired 2024-09-03**; for recurring-window capacity guarantees use an On-Demand Capacity Reservation instead.
- **Capacity Reservations** → guarantee launchable capacity in an AZ. Zonal Reserved Instances bundle both; Regional Reserved Instances only discount; On-Demand Capacity Reservations skip the term commitment (no discount).
- **Savings Plans** → commit to an hourly spend (EC2 Instance Savings Plans or Compute Savings Plans across EC2/Fargate/Lambda) for broad, flexible discounts.
- **Dedicated Hosts** → physical-machine (socket/core) BYOL licensing.
- **Dedicated Instances** → no-shared-hardware requirement without managing a host.

> [!tip] Reminder
> Capacity under shortage: **reserved capacity first**, then ordinary On-Demand if capacity is available, then Spot only from spare capacity. A Regional Reserved Instance is only a discount, not a launch guarantee. To guarantee a launch, you need a Zonal Reserved Instance or an On-Demand Capacity Reservation.

## Key Takeaways

1. **"Purchase options" is the official term** — *launch types* is the common but informal alternative.
2. **On-Demand Instances are the baseline.** No commitment, no discount, per-second compute billing while running, storage bills regardless. **No capacity priority** under shortage.
3. **Spot Instances ≠ a discount tier; Spot ≠ reliable.** You always pay the *current* Spot price (not your max), and AWS can interrupt the instance when capacity is reclaimed or when price exceeds a configured max. The max sets price tolerance, not your rate.
4. **Reserved Instances are a billing discount, not a capacity object.** Zonal Reserved Instances happen to bundle a capacity reservation; Regional Reserved Instances do not. Reserved Instances are billed whether you use them or not.
5. **3-year + All Upfront = the biggest Reserved Instance discount.**
6. **Three Reserved Instance offering classes:**
   - **Standard Reserved Instances** (~72% max) — biggest discount; locked to family/OS/tenancy with size-flexibility within the family
   - **Convertible Reserved Instances** (~54% max) — exchangeable for an RI of **equal or greater value** with different family/OS/tenancy; trade ~18 points of discount for the right to switch
   - **Scheduled Reserved Instances** — **retired 2024-09-03**; use On-Demand Capacity Reservations for new recurring-window capacity guarantees
   Both Standard and Convertible can be Regional or Zonal in scope (orthogonal axes).
7. **Discount and capacity are independent axes.** Regional Reserved Instance = discount only; On-Demand Capacity Reservation = capacity only; Regional Reserved Instance + On-Demand Capacity Reservation = both; Zonal Reserved Instance = both bundled. **Discount stacking only works with Regional Reserved Instances**, not Zonal.
8. **Capacity assurance under shortage:** Zonal Reserved Instances and On-Demand Capacity Reservations reserve capacity in an AZ. Regional Reserved Instances and Savings Plans are discounts only.
9. **Savings Plans** — commit on $/hour. **Compute Savings Plans** (~66%) cover EC2/Fargate/Lambda; **EC2 Instance Savings Plans** (~72%) are locked to a family+Region. No capacity component — pair with an On-Demand Capacity Reservation if you need both.
10. **Dedicated Hosts** = pay per host, host targeting, BYOL per-socket/per-core licensing. **Dedicated Instances** = no-shared-hardware guarantee, host opaque, per-Region hourly fee. Easy to confuse — see [the "Dedicated is two things" gotcha](../services/ec2.md#tenancy-shared-vs-dedicated).

## Quick Reference

| Concept | Detail |
| --- | --- |
| Official term | **Purchase options** (informally "launch types") |
| Default option | **On-Demand Instances** — no commitment, no discount, no reserved capacity |
| Cheapest option | **Spot Instances** (~90% off) — interruptible for capacity reclaim or price, with terminate/stop/hibernate behavior depending on configuration |
| Biggest Reserved Instance discount | **3-year, All Upfront Standard Reserved Instance** |
| Reserved Instance offering class | **Standard** (~72%, no family/OS/tenancy changes) or **Convertible** (~54%, exchange for equal/greater value) |
| Reserved Instance scope | **Zonal Reserved Instance** (one AZ, includes capacity) or **Regional Reserved Instance** (any AZ, no capacity) — orthogonal to offering class |
| Scheduled Reserved Instances | **Retired 2024-09-03** — use On-Demand Capacity Reservations for recurring-window capacity guarantees |
| Capacity assurance under shortage | **Zonal RI or ODCR reserves capacity**; Regional RI/Savings Plan discounts do not |
| Pure capacity, no commitment | **On-Demand Capacity Reservation** — billed even idle, no discount |
| Discount stacking with capacity | **Regional Reserved Instance + On-Demand Capacity Reservation** (Zonal Reserved Instance cannot stack) |
| Compute Savings Plans | ~66%; EC2 any family/Region + Fargate + Lambda |
| EC2 Instance Savings Plans | ~72%; specific family + Region, size/OS-flexible |
| Dedicated Hosts | Pay per host; **BYOL per-socket/per-core licensing**; Host Affinity |
| Dedicated Instances | No co-tenant hardware; host opaque; per-Region hourly fee |

## References

- [EC2 — Hosts vs Instances, Tenancy](../services/ec2.md#hosts-vs-instances)
- [EC2 — Billing Model](../services/ec2.md#billing-model)
- [Cost Optimization — broader purchase-plan landscape (incl. RDS / ElastiCache / DynamoDB)](cost-optimization.md)
- [AWS Organizations — RI/SP pooling across accounts](../services/organizations.md#consolidated-billing)
- [AWS SAA Exam Notes — Cost Optimization](../certifications/aws-saa.md#cost-optimization)
- Source: `raw/aws-cert-study-notes/aws-solution-architect/ec2_purchase_options.md`
- [Reserve compute capacity with On-Demand Capacity Reservations — AWS docs](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-capacity-reservations.html)
- [Capacity Reservation pricing and billing — AWS docs](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/capacity-reservations-pricing-billing.html)
- [Regional and zonal Reserved Instances (scope) — AWS docs](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/reserved-instances-scope.html)
- [How Reserved Instance discounts are applied — AWS docs](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/apply_ri.html)
- [EC2 purchasing options — AWS docs](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instance-purchasing-options.html)
- [Spot Instance interruptions — AWS docs](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-interruptions.html)
