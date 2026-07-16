---

tags:

- aws
- saa-c03
- ec2
- auto-scaling
- asg
- cloudwatch
- scaling aliases:
- Auto Scaling Group Scaling Policies
- Simple vs Step Scaling

---

# ASG Scaling Policies

## Overview

An Auto Scaling Group has three capacity values: **min size**, **max size**, and **desired capacity**. Scaling policies exist to _adjust the desired capacity automatically_ in response to changing demand.

**Scaling policies are optional.** An ASG created with no scaling policies works perfectly well — it simply holds static min/max/desired values. This is a very common point of confusion.

Three broad modes:

|Mode|What it means|
|---|---|
|**Manual scaling**|You adjust min/max/desired yourself. Useful for testing, urgent situations, or pinning capacity at a fixed number as a **cost control** measure.|
|**Dynamic scaling**|Policies adjust desired capacity based on a metric/alarm.|
|_(Scheduled scaling)_|📚 Not covered in this lesson — time-based adjustments.|

Everything below is **dynamic scaling** — every type ultimately does one thing: **adjust the desired capacity based on some criteria**.

⚠️ **Min/max always win.** However extreme the policy's calculation, the resulting desired capacity is clamped between min and max. A policy that says "remove 3" when you're at min=1 removes nothing.

---

## The Four Dynamic Scaling Approaches

### 1. Simple Scaling

Define actions that fire when a **CloudWatch alarm** enters ALARM state. Static adjustment.

> _Example (illustrative, not a memorizable value):_ if `ASGAverageCPUUtilization > 50%` → add 2 instances; if `< 50%` → remove 2 instances.

**Problem:** inflexible. Whether the metric is 1% over your target or 50% over, you add the same 2 instances. Simple, but not efficient.

### 2. Step Scaling

Increases/decreases desired capacity based on a set of **step adjustments** that vary **based on the size of the alarm breach**. You define upper and lower bounds per step.

> _Example (illustrative):_ 50–59% → do nothing; 60–69% → +1; 70–79% → +2; 80–100% → +3. And in reverse for scale-in.

**Generally better than simple** — it adapts to how extreme the load change actually is.

### 3. Target Tracking

You pick a **predefined metric** and an **ideal target value** (e.g. "keep average CPU at 50%"). The ASG calculates the scaling adjustment itself, automatically, and keeps the metric at your target. The further the actual metric is from target, the more extreme the action.

<cite index="17-1">Target tracking supports these predefined metrics:</cite>

- `ASGAverageCPUUtilization` — average CPU utilization of the ASG
- `ASGAverageNetworkIn` — average bytes received across all network interfaces
- `ASGAverageNetworkOut` — average bytes sent across all network interfaces
- `ALBRequestCountPerTarget` — average ALB request count per target

📚 Cantrill describes this as being _like a thermostat_ — and so does AWS: <cite index="14-1">it works similar to the way your thermostat maintains the temperature of your home — you select a temperature and the thermostat does the rest.</cite>

📚 <cite index="17-1">To use `ALBRequestCountPerTarget` you must also specify the `ResourceLabel` parameter identifying the ALB target group.</cite> Also note it's **ALB only** — not NLB, not CLB. 🎯

### 4. Scaling Based on SQS

Scale on **`ApproximateNumberOfMessagesVisible`** in an SQS queue. This is the classic **worker pool** architecture: as messages pile up, the ASG scales out to process them; as the queue drains, the ASG scales back in to reduce cost.

🎯 "Worker pool" + "queue depth" + "decoupled" → SQS-based scaling.

---

## Diagram Walkthrough — Simple Scaling

![[asg-scaling-policy-simple.png]]

**Context.** Title reads _ASG - Simple Scaling_. The diagram shows **the same Auto Scaling Group at three points in time**, left to right.

**The footer rules** (bottom of the diagram, in bold) define the whole policy — read these first:

- `If ASGAverageCPUUtilization > 50% ADD 2 Instances` (the word ADD in orange/red)
- `If ASGAverageCPUUtilization < 50% REMOVE 2 Instances` (REMOVE in green)

Note the metric name is `ASGAverageCPUUtilization` — that's the real AWS metric identifier, worth recognizing.

**Main objects.** Each of the three panels is an orange dotted rounded rectangle = the ASG boundary, labelled with the orange ASG icon. Inside each:

- **Orange/solid CPU chip icons** = instances _actually provisioned and running_.
- **Pale/faded CPU chip icons** = _capacity headroom_ — slots the ASG could fill up to MAX, but hasn't.
- The **blue chip icon + gauge** on the left of each panel = the current average CPU metric reading for the group.

Under each panel is the capacity state line: `MIN=1, MAX=4, Desired = N`.

**Reading the three panels:**

|Panel|CPU reading|Gauge|State|
|---|---|---|---|
|**Left**|**10%** (green)|needle low, no warning|MIN=1, MAX=4, **Desired=1** → 1 solid chip, 3 faded|
|**Middle**|**60%** (red)|⚠️ warning triangle beside gauge = alarm breached|MIN=1, MAX=4, **Desired=3** → 3 solid chips, 1 faded|
|**Right**|**10%** (green)|needle low, warning gone|MIN=1, MAX=4, **Desired=1** → back to 1 solid chip|

**The arrows** carry the transitions and are colour-coded to match the footer rules:

- **Red arrow** (left→middle) labelled `Desired = Desired + 2 (MAX 4)` — CPU jumped to 60%, breaching the >50% alarm, so the ADD rule fires. Desired goes 1 → 3. Still within MAX of 4, so 2 more instances are provisioned, with room for a fourth. The `(MAX 4)` annotation is the reminder that the result is clamped.
- **Green arrow** (middle→right) labelled `Desired = Desired - 2 (MIN 1)` — CPU dropped to 10%, the <50% REMOVE rule fires. Desired goes 3 → 1. Two instances terminate.

**The lesson the diagram is teaching:** the adjustment is **always ±2**, regardless of severity. 1% over target or 50% over — same 2 instances. That's the inflexibility.

---

## Diagram Walkthrough — Step Scaling

![[asg-scaling-policy-step.png]]

**Context.** Title reads _ASG - Step Scaling_. Same ASG, now at **six points in time**, left to right. Same `MIN=1, MAX=4` as the simple scaling example.

**The footer rules** — split into two columns around a central metric-graph icon, with arrows pointing outward to a **green ⊕ (add)** button on the left and a **pink ⊖ (remove)** button on the right:

_Scale-out steps (green ⊕ side):_

- 50%–59% → DO NOTHING
- 60%–69% → ADD 1
- 70%–79% → ADD 2
- 80%–100% → ADD 3

_Scale-in steps (pink ⊖ side):_

- 40%–49% → DO NOTHING
- 30%–39% → REMOVE 1
- 20%–29% → REMOVE 2
- 0%–19% → REMOVE 3

📚 Note the two "do nothing" bands together form a **40%–59% dead zone** around the 50% target. That gap is deliberate — <cite index="11-1">AWS advises choosing an adequate margin between scale-out and scale-in thresholds to prevent flapping.</cite> 🎯 "flapping" / "thrashing" → the answer is usually a wider dead band.

**Main objects.** Six orange dotted columns = the ASG at each point in time. Same visual language as before: **solid orange chips = running instances**, **pale chips = unused headroom**. Below each column, a blue chip icon shows the CPU reading. Above each column is a **header label** showing _what the policy calculated_ vs _what actually happened_ — this is the key to reading the diagram.

**Reading the six panels:**

|#|Header|CPU|Running|What's happening|
|---|---|---|---|---|
|1|`-3 or MIN1`|5%|**1**|0–19% band says remove 3 — but MIN=1 clamps it. Group sits at 1, headroom for 3.|
|2|`+3 or MAX4`|100%|**4**|Extreme load. 80–100% band adds 3 → desired hits 4, which is also MAX. Fully provisioned.|
|3|`+0 or MAX4`|55%|**4**|The extra capacity has pulled average CPU down to 55%. That's in the 50–59% do-nothing band → **no change**.|
|4|`-3 or MIN1`|5%|**1**|Load collapses. 0–19% band removes 3 → desired drops 4 → 1.|
|5|`-3 or MIN1`|5%|**1**|Still 5%, policy still _wants_ to remove 3 — but MIN=1 holds. **No change.**|
|6|`+1 or MAX4`|60%|**2**|End-of-day load returns. 60–69% band adds 1 → desired 1 → 2.|

**Read the header labels literally:** `-3 or MIN1` means _"the policy calculated -3, but the MIN of 1 is what actually governs"_. Panels 1 and 5 are the same reading (5%) producing the same clamped result — the diagram repeats it deliberately to hammer home that **min/max always override the policy's arithmetic**.

**The lesson:** how extreme the change is determines how many units of compute move. It's **not static** like simple scaling.

---

## Simple vs Step — The Core Distinction

🎯 **This is the most commonly confused pair in this topic.**

- **Simple**: one alarm → one fixed adjustment. Same response regardless of breach magnitude.
- **Step**: one alarm → multiple step adjustments, selected by **how far past the threshold** the metric is.

> The main difference is _the ability to scale in different ways based on how extreme the load changes are._

---

## ⚠️ Course vs Current AWS Reality

This is the biggest gap in this lesson. **Cantrill says "AWS recommends step scaling versus simple."** That was true, but AWS's current guidance has moved further:

**1. AWS now strongly recommends _target tracking_ first — not step.** <cite index="14-1">AWS strongly recommends using target tracking scaling policies and choosing a metric that changes inversely proportional to a change in the capacity of your ASG.</cite> The step/simple docs echo this: <cite index="11-1">AWS strongly recommends target tracking scaling policies to scale on metrics like average CPU utilization or average request count per target.</cite>

Step scaling's positioned role today: <cite index="15-1">you still have the option to use step scaling as an additional policy for a more advanced configuration — for example, configuring a more aggressive response when demand reaches a certain level.</cite>

**2. Simple scaling is effectively legacy.** <cite index="13-1">AWS keeps simple scaling in the docs as a reference for customers who want it, but recommends target tracking or step scaling instead.</cite> The historical reason: <cite index="13-1">EC2 Auto Scaling originally supported only simple scaling policies; a policy created before target tracking and step scaling were introduced is treated as a simple scaling policy.</cite>

**3. The _real_ mechanical difference between simple and step (not mentioned in the lesson) is cooldown blocking.** 🎯 This is exam gold:

- <cite index="15-1">The main issue with simple scaling is that after a scaling activity starts, the policy must wait for that activity (or health check replacement) to complete **and the cooldown period to end** before responding to additional alarms.</cite>
- <cite index="15-1">With step scaling, the policy can continue to respond to additional alarms even while a scaling activity or health check replacement is in progress — all breached alarms are evaluated as the messages are received.</cite>
- Which is why: <cite index="13-1">AWS recommends step scaling policies instead, **even if you have only a single scaling adjustment**.</cite>

**4. Cooldowns belong to simple scaling; warmup belongs to the rest.** <cite index="12-1">Target tracking and step scaling policies can initiate a scale-out activity immediately without waiting for a cooldown period to end; instead, individual instances have a warmup period.</cite> <cite index="18-1">`DefaultCooldown` and `Cooldown` are only needed when you use simple scaling policies.</cite>

**Why keep the course framing anyway:** older exam questions may still test "step > simple" without mentioning target tracking. Know the whole hierarchy: **target tracking ≥ step > simple**, and pick based on what the question offers.

---

## 📚 Additional Exam-Relevant Detail Not in the Lesson

**Min/max clamping is an AWS-documented guarantee**, not just a diagram convention: <cite index="14-1">when a dynamic scaling policy is invoked, if the capacity calculation produces a number outside the min/max size range, EC2 Auto Scaling ensures the new capacity never goes outside those limits.</cite>

**The metric is an aggregate across the whole group**: <cite index="14-1">the metrics used to invoke the alarm state are an aggregation of metrics from all instances in the ASG — e.g. an ASG with one instance at 60% CPU and one at 40% averages to 50%.</cite> This is why the diagrams show a _single_ CPU figure per group.

**Multiple policies can coexist**: <cite index="14-1">an ASG can have more than one scaling policy — one or more target tracking policies, one or more step scaling policies, or both.</cite> ⚠️ But: <cite index="14-1">AWS recommends caution when mixing target tracking with step scaling, because conflicts can cause undesirable behavior.</cite>

**Target tracking prioritizes availability**: <cite index="17-1">a target tracking policy scales in more gradually when traffic is decreasing; if you want greater control, step scaling might be the better option.</cite> 🎯 "need precise/granular control over scaling behavior" → step, not target tracking.

⚠️ **Don't touch target tracking's alarms.** <cite index="17-1">AWS creates and manages the CloudWatch alarms associated with target tracking policies and may edit, replace, or delete them; do not create, edit, or delete them yourself.</cite> Contrast with simple/step: <cite index="11-1">with step scaling and simple scaling, _you_ create and manage the CloudWatch alarms that invoke the scaling process.</cite> 🎯 **This is a clean discriminator**: "who owns the alarm?" — you = simple/step; AWS = target tracking.

---

## Exam Traps

|⚠️ Trap|Reality|
|---|---|
|"Scaling policies are required on an ASG"|**False.** An ASG with no policies works fine — it just holds static min/max/desired.|
|"Manual scaling means clicking a button to launch instances"|No — it means **manually adjusting min/max/desired values**.|
|"Simple scaling and step scaling both need you to create the alarm"|**True** — and this is the distinguisher from target tracking, where AWS owns the alarm.|
|"Target tracking works with any CloudWatch metric out of the box"|Only 4 **predefined** metrics; anything else requires a **custom metric specification**.|
|"`ALBRequestCountPerTarget` works with NLB/CLB"|**ALB only.**|
|"Step scaling is AWS's top recommendation"|Course-era framing. **Target tracking** is the current top recommendation.|
|"Cooldown periods apply to all dynamic scaling"|Cooldowns are the **simple scaling** mechanism. Target tracking/step use **instance warmup** and can scale out immediately.|
|"A policy calculating -3 at MIN=1 will drop the group below 1"|Never. Min/max clamp the result.|
|Scale-out and scale-in thresholds set to the same value|Causes **flapping**. Keep a dead band (the 40–59% gap in the step diagram).|

### 🎯 Trigger Keywords

|Question says...|Answer|
|---|---|
|"maintain average CPU at 40%" / "set and forget" / "thermostat" / "keep metric at a value"|**Target tracking**|
|"respond differently depending on how large the breach is" / "graduated response" / "add more instances the worse it gets"|**Step scaling**|
|"granular control" / "aggressive response at a specific level" / "custom step adjustments"|**Step scaling**|
|"worker pool" / "process backlog" / "queue depth" / "`ApproximateNumberOfMessagesVisible`"|**SQS-based scaling**|
|"hold capacity fixed" / "cost control" / "urgent, temporary override"|**Manual scaling** (adjust desired)|
|"legacy" / "single adjustment" / "waits for cooldown before reacting to further alarms"|**Simple scaling**|
|"policy stopped responding while a scaling activity was in progress"|**Simple scaling** + cooldown is the culprit → recommend step|
|"AWS manages the alarm for you"|**Target tracking**|
|"requests per instance behind an ALB"|**Target tracking** with `ALBRequestCountPerTarget`|

---

## Quick Reference

||Simple|Step|Target Tracking|SQS-based|
|---|---|---|---|---|
|**Adjustment**|Single, fixed|Multiple steps by breach size|Calculated automatically|Based on queue depth|
|**Alarm owner**|You|You|**AWS**|You (custom metric)|
|**Flexibility**|Lowest|High / granular|High / automatic|Purpose-built|
|**Cooldown blocking**|⚠️ **Yes**|No (warmup)|No (warmup)|Depends on policy type used|
|**AWS stance today**|Legacy — not recommended|Recommended for advanced/granular|✅ **Strongly recommended default**|Standard worker-pool pattern|
|**Best for**|Nothing new|Variable load needing controlled response|Metric-proportional load|Decoupled batch/worker processing|

---

## Open Questions for You, Roger

1. **Do you want a separate `Scaling Cooldowns & Instance Warmup` note?** The cooldown-vs-warmup distinction is the single most exam-testable thing in this area and Cantrill's lesson doesn't touch it. It may get its own lesson later in the course — worth checking before I duplicate.
2. **Scheduled scaling** wasn't mentioned here either — is it covered in a separate Cantrill lesson, or should I fold it into this note?
3. **Predictive scaling** (ML-based, forecasts load) exists and is occasionally referenced in SAA question banks. Cantrill doesn't cover it in this lesson. Want a stub note?
4. Should I add the `[[EC2 Launch Templates]]` and `[[Auto Scaling Groups]]` wiki-links now, or wait until the ASG note exists to avoid dead links in your vault?

---

## Sources

- **Course**: Adrian Cantrill — AWS Solutions Architect Associate (SAA-C03), _ASG Scaling Policies_ lesson, [learn.cantrill.io](https://learn.cantrill.io/)
- **Diagrams**: Adrian Cantrill (`asg-scaling-policy-simple.png`, `asg-scaling-policy-step.png`)
- AWS Docs — [Dynamic scaling for Amazon EC2 Auto Scaling](https://docs.aws.amazon.com/autoscaling/ec2/userguide/as-scale-based-on-demand.html)
- AWS Docs — [Step and simple scaling policies](https://docs.aws.amazon.com/autoscaling/ec2/userguide/as-scaling-simple-step.html)
- AWS Docs — [Simple scaling policies](https://docs.aws.amazon.com/autoscaling/ec2/userguide/simple-scaling-policies.html)
- AWS Docs — [Target tracking scaling policies](https://docs.aws.amazon.com/autoscaling/ec2/userguide/as-scaling-target-tracking.html)
- AWS Docs — [Scaling cooldowns](https://docs.aws.amazon.com/autoscaling/ec2/userguide/ec2-auto-scaling-scaling-cooldowns.html)
- AWS Docs — [Available warm-up and cooldown settings](https://docs.aws.amazon.com/autoscaling/ec2/userguide/consolidated-view-of-warm-up-and-cooldown-settings.html)
- AWS API Reference — [PredefinedMetricSpecification](https://docs.aws.amazon.com/autoscaling/ec2/APIReference/API_PredefinedMetricSpecification.html)