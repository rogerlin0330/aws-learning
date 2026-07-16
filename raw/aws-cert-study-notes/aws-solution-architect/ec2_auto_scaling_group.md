---

## tags: [aws, saa-c03, ec2, auto-scaling-groups, elb, scaling-policies, cloudwatch]

# Auto Scaling Groups (ASG)

## Overview

An **Auto Scaling group (ASG)** does one core thing: it provides automatic scaling for EC2 instances. It can also be used — as part of scaling or entirely in isolation — to implement a **self-healing architecture**.

An ASG doesn't know how to build instances itself. It delegates that to a **launch template** (or, on older exam questions, a **launch configuration**) — one specific configuration, one specific version, linked to the group at any given time. You can change which template/version/configuration is linked, but only one is active at a time, so every instance the ASG launches comes from that single definition.

📚 A useful mental model for the exam: **the ASG controls the _when_ and the _where_** (when instances launch, and which subnets/AZs they land in), while **the launch template/configuration controls the _what_** (which AMI, instance type, and bootstrap configuration is used).

---

## Diagram Walkthrough — Auto Scaling Groups Architecture

![[asg-architecture.png]]

- **Context**: A VPC (`10.16.0.0/16`, `us-east-1`) spans three Availability Zones — AZ‑A, AZ‑B, and AZ‑C — each with its own private subnet (`10.16.32.0/20`, `10.16.96.0/20`, `10.16.160.0/20`).
- **Main objects**: A single **Auto Scaling group** (orange dotted boundary) stretches across all three subnets/AZs. Inside each subnet sits one EC2 instance (the "cat picture" icon), each behind a lock icon representing the subnet's security boundary.
- **What it teaches**: The ASG is not tied to a single subnet or AZ — it is configured with a _set_ of subnets (typically one per AZ) and provisions instances across them. This is the mechanism that gives ASGs Multi-AZ resilience "for free."
- **Footer takeaway**: When the ASG provisions instances, it attempts to **keep the instance count balanced across AZs** — in this diagram, three subnets and a desired capacity of three would probably (not guaranteed) result in one instance per AZ.

---

## The Three Core Values: Min, Desired, Max

Every ASG has three defining values, often shorthanded as **X, Y, Z** (e.g., "1, 2, 4" = min 1, desired 2, max 4):

|Value|Meaning|
|---|---|
|**Minimum size**|The floor — the ASG will never let running instance count drop below this|
|**Desired capacity**|The target — the ASG actively works to keep the _actual_ running instance count equal to this|
|**Maximum size**|The ceiling — the ASG will never scale above this|

**The ASG's one foundational job**: keep the number of running instances equal to desired capacity, by provisioning or terminating instances as needed.

- Desired capacity must always be **≥ minimum** and **≤ maximum**.
- Desired capacity 2, but only 1 running → ASG launches a new instance.
- Desired capacity 2, but 3 running → ASG terminates one instance.

You can run an ASG in a **fully manual** mode (no automation, you just edit the values yourself and the ASG reconciles), or pair it with **scaling policies** that adjust desired capacity automatically based on criteria like CPU load.

🎯 **Trigger keywords**: "min/desired/max," "keep instance count matching," "X, Y, Z notation" all point straight at ASG fundamentals.

---

## Diagram Walkthrough — ASG + Load Balancers

![[asg-and-load-balancers.png]]

- **Context**: Bob (far right) browses to `animals4life.org`, hitting a load balancer, which routes via a listener to the "blog" target, which is served by instances inside **Target Group 1**.
- **Main objects**: The **Auto Scaling group** (orange dotted box) contains two running instances. A **green arrow** points from the ASG's instances into Target Group 1, showing instances being automatically registered as targets. The **pink curved arrow** between the load balancer and the ASG represents the ASG _consuming the load balancer's health check results_ instead of relying purely on EC2 status checks.
- **Key annotations**:
    - _"ASG Instances are automatically added to or removed from the target group"_ — this is the ELB integration in a nutshell; you never manually manage target group membership when an ASG is attached.
    - _"ASG can use the Load Balancer health checks rather than EC2 status checks — Application Awareness"_ — this is the callout for **ELB health checks**, which are richer/more application-aware than basic EC2 status checks.

---

## Networking & Placement

- ASGs are linked to a **VPC**, and you configure **which subnets** the ASG can use.
- Instances are only ever launched into the configured subnets.
- The ASG attempts to **keep instance counts even across AZs** where possible (not a hard guarantee — it's a best-effort leveling behavior).

---

## Scaling Methods

There are three broad ways to scale an ASG:

### 1. Manual scaling

Not really a "policy" — you directly edit min/desired/max, and the ASG reconciles.

### 2. Scheduled scaling

Adjusts desired capacity on a **time-based schedule**. Ideal for predictable demand patterns — sales events, or scaling in outside business hours.

🎯 **Exam trigger**: "known periods of high/low usage," "predictable traffic pattern," "every day at 9am" → **scheduled scaling**.

### 3. Dynamic scaling

Rules that **react** to a metric and adjust the ASG's values. Three subtypes:

#### a) Simple scaling

A pair of rules — one to scale out, one to scale in — reacting to a single metric threshold (e.g., "CPU > 50% → add 1 instance"; "CPU < 50% → remove 1 instance"). The metric isn't limited to CPU — memory, disk I/O (with the **CloudWatch agent** installed for some of these), SQS queue length, or a custom application metric like response time can all be used.

📚 This is the _legacy_ policy type and is the one still associated with the classic **cooldown period** concept (see below).

#### b) Step scaling

Similar to simple scaling but with **graduated responses** based on how far outside normal the metric is — e.g., add 1 instance if CPU > 50%, but add 3 if CPU spikes above 80%. Reacts more aggressively the more extreme the deviation. 📚 Step scaling is generally preferable to simple scaling except when simplicity is the only priority.

#### c) Target tracking

A different model entirely: you define an **ideal target value** for a metric (e.g., 40% aggregate CPU), and the ASG scales in/out automatically to hold that target — similar to a thermostat.

Not all metrics support target tracking. Confirmed **predefined metrics** include:

- `ASGAverageCPUUtilization` (average CPU utilization)
- `ASGAverageNetworkIn` / `ASGAverageNetworkOut`
- `ALBRequestCountPerTarget` (relevant when the ASG sits behind an **Application Load Balancer**)

You can also target-track on **custom CloudWatch metrics**. <cite index="1-1">Predefined metrics provide easier access to the most commonly used metrics for scaling, while custom metrics allow scaling on other available CloudWatch metrics, including high-resolution metrics published at finer intervals</cite>.

🎯 **Exam trigger**: "maintain 40% CPU," "keep utilization at a target level" → **target tracking**. "React more aggressively as load increases" → **step scaling**. "Simple threshold, add/remove one" → **simple scaling**.

---

## ⚠️ Course-vs-Current-AWS Discrepancy: Cooldown vs. Instance Warmup

The course frames the **cooldown period** as a single, universal ASG setting: _"a value in seconds that controls how long to wait at the end of a scaling action before doing another,"_ used to avoid chaotic/rapid scaling and the extra cost of minimum billable periods.

This was accurate historically, but **current AWS documentation narrows this significantly**:

- **Default cooldown** now applies **specifically to simple scaling policies** — <cite index="20-1">it's the amount of time, in seconds, between one scaling activity ending and another one starting due to simple scaling policies, defaulting to 300 seconds</cite>.
- For **target tracking and step scaling policies**, the modern equivalent is the **default instance warmup** (or a policy-specific instance warmup override) — <cite index="11-1">the amount of time, in seconds, until a new instance is considered to have finished initializing and resource consumption to become stable after entering the InService state</cite>. This value is also used to delay new instances from skewing CloudWatch metrics used for scaling decisions.
- A widely cited practitioner summary puts it plainly: <cite index="14-1">EC2 Auto Scaling supports cooldown periods when using simple scaling policies, but not when using target tracking policies, step scaling policies, or scheduled scaling</cite>.
- If default instance warmup isn't explicitly set, AWS docs note the system falls back sensibly — <cite index="13-1">target tracking and step scaling policies use the value set for the default cooldown as the default warm-up time if no default instance warmup is configured, while predictive scaling policies have no default warm-up time at all</cite>.

📚 **What this means for the exam**: If a question is old/legacy-styled and just says "cooldown period" in a generic sense, treat it as the course does — a delay-between-scaling-actions concept. But if a question specifically involves **target tracking or step scaling** and asks about a "warm-up" delay before an instance contributes to metrics, the correct term/concept is **default instance warmup**, not cooldown.

---

## Health Checks & Self-Healing

By default, ASGs monitor instance health using **EC2 status checks**. If EC2 reports a failed instance, the ASG:

1. Terminates the failed instance.
2. Provisions a replacement.

This is **self-healing** — it also applies if you manually terminate an instance; the ASG simply replaces it to restore desired capacity.

### 📚 The "cheap HA trick": min=1, desired=1, max=1

If you configure:

- A launch template that can fully bootstrap an instance,
- An ASG using that template across **multiple subnets in different AZs**,
- Min = Desired = Max = **1**,

...you get simple, low-cost **instance recovery / high availability**. If the instance fails or is terminated, the ASG relaunches it — potentially in a _different_ AZ if the original one is impaired. This pattern gives HA without needing a load balancer or multiple concurrent instances.

🎯 **Exam trigger**: "cheapest way to add high availability to a single EC2 instance" → ASG with min/desired/max all = 1 across multiple AZs.

### Integrating with load balancer health checks

When an ASG is attached to a load balancer's target group, it can be configured to use **ELB health checks** instead of (or alongside) EC2 status checks. ELB health checks are **application-aware** — they can validate HTTP/HTTPS response codes, not just infrastructure-level status.

⚠️ **Exam trap**: Health check choice matters a lot here, in both directions:

- If your health check only pings a static page while the app underneath has failed logic, the check can report healthy when the app is actually broken.
- If your health check depends on a downstream dependency (e.g., a database call), a **database outage** can cause **every instance** to fail health checks — leading the ASG to terminate and replace a fleet of otherwise-healthy instances. The real problem (the DB) goes unaddressed while healthy compute is churned.

---

## ASG + Load Balancer Integration

When an ASG is linked to a load balancer's target group:

- Instances the ASG provisions are **automatically registered** as targets.
- Instances the ASG terminates are **automatically deregistered**.
- Users connect via the load balancer and are abstracted from the individual instances — this fluid, automatic add/remove behavior is what makes the architecture **elastic**.

This ASG + ALB combination is the standard pattern for elastic, self-scaling web application architectures.

---

## Scaling Processes (Suspend / Resume)

An ASG performs several distinct internal processes, each of which can be individually **suspended** or **resumed**:

|Process|What suspending it does|
|---|---|
|**Launch**|ASG will not scale out, even if alarms/scheduled actions fire|
|**Terminate**|ASG will not terminate any instances|
|**AddToLoadBalancer**|Controls whether new instances are registered to the load balancer|
|**AlarmNotification**|Controls whether the ASG reacts to CloudWatch alarms|
|**AZRebalance**|Controls whether the ASG attempts to redistribute instances evenly across AZs|
|**HealthCheck**|Turns instance health checking on/off across the whole group|
|**ReplaceUnhealthy**|Controls whether unhealthy instances get replaced|
|**ScheduledActions**|Controls whether scheduled scaling actions run|

Additionally, you can set an **individual instance** to **Standby** (temporarily exempt from all ASG activity — useful for performing maintenance without the ASG interfering) or back to **InService**.

🎯 **Exam trigger**: "perform maintenance on one instance in an ASG without it being terminated or affecting scaling" → set that instance to **Standby**.

---

## Cost Considerations

- **ASGs themselves are free** — you only pay for the resources (EC2 instances, etc.) that the ASG provisions.
- Use **cooldowns** (simple scaling) or **instance warmup** (target tracking/step scaling) to avoid rapid, chaotic scaling that racks up cost — especially relevant since instances have a **minimum billable period** regardless of actual usage duration.
- 📚 **Favor more, smaller instances over fewer, larger ones.** Smaller instances give more granular scaling steps — adding 1 of 20 small instances is a much smaller cost/capacity jump than adding 1 of 2 large instances. This granularity is a recurring SAA exam cost-optimization theme.

---

## Quick Reference

|Concept|Summary|
|---|---|
|ASG's one job|Keep running instance count == desired capacity|
|Min / Desired / Max|Floor / target / ceiling for instance count|
|Manual scaling|You edit the values directly|
|Scheduled scaling|Time-based desired capacity changes|
|Simple scaling|Single threshold, add/remove N instances; uses **cooldown**|
|Step scaling|Graduated thresholds, faster response to bigger deviations; uses **instance warmup**|
|Target tracking|Maintain a target metric value (e.g., 40% CPU); uses **instance warmup**|
|Cooldown period|Legacy concept, now specific to **simple scaling** (default 300s)|
|Default instance warmup|Modern equivalent for target tracking/step scaling — delay before a new instance counts toward metrics|
|Self-healing|Failed/terminated instances are automatically replaced|
|Min=Desired=Max=1 across AZs|Cheap single-instance HA/recovery pattern|
|ELB health checks vs. EC2 status checks|ELB checks are application-aware (HTTP/HTTPS); EC2 checks are infrastructure-only|
|ASG + Target Group|Instances auto-registered/deregistered as they're provisioned/terminated|
|Standby|Removes one instance from ASG's active management (e.g., for maintenance)|
|ASG cost|ASG itself is free; you pay only for provisioned resources|
|ASG controls|**When** and **where** instances launch|
|Launch template/config controls|**What** gets launched (AMI, instance type, bootstrap)|

---

## Open Questions

- Do you want a dedicated note on **Launch Templates vs. instance warmup interactions** (e.g., how `InstanceWarmup` overrides at the policy level interact with `DefaultInstanceWarmup` at the group level), or is the Quick Reference summary sufficient for now?
- Want a follow-up note on **lifecycle hooks** (mentioned briefly in AWS docs as a way to extend/reduce the warm-up period) since they weren't covered in this transcript but are exam-relevant and closely related?
- Should the **Standby / suspend-resume processes** table be cross-linked into a future "ASG Instance Refresh & Lifecycle" note if that topic appears later in the course?

---

## Source

- Adrian Cantrill SAA-C03 course — "Auto Scaling Groups" lesson (learn.cantrill.io)
- Diagrams: `asg-architecture.png`, `asg-and-load-balancers.png` (Cantrill course materials)
- AWS Documentation: [Target tracking scaling policies for Amazon EC2 Auto Scaling](https://docs.aws.amazon.com/autoscaling/ec2/userguide/as-scaling-target-tracking.html)
- AWS Documentation: [Available warm-up and cooldown settings](https://docs.aws.amazon.com/autoscaling/ec2/userguide/consolidated-view-of-warm-up-and-cooldown-settings.html)
- AWS Documentation: [Set default values for instance warmup or scaling cooldown](https://docs.aws.amazon.com/autoscaling/ec2/userguide/set-default-values-for-instance-warmup-or-scaling-cooldown.html)