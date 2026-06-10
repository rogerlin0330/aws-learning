---
title: Boot Time to Service Time — AMI Baking vs Bootstrapping
type: pattern
tags: [boot-time-to-service-time, ami-baking, bootstrapping, golden-image, ec2, user-data, cfn-init, image-builder, packer, scaling, ha]
certs: [aws-saa]
updated: 2026-06-10
---

# Boot Time to Service Time — AMI Baking vs Bootstrapping

When you launch an [EC2](../services/ec2.md) instance, it is rarely useful the instant the hypervisor reports "running." There is almost always extra work — installing software, applying configuration, pulling in application code — before the instance can actually accept customer connections. **Boot time to service time** is the metric for that whole gap, and minimizing it is a recurring architecture decision.

## What "Boot Time to Service Time" Measures

**Boot time to service time** is the total elapsed time from the moment you launch an instance until the moment it can serve real traffic. It has two parts:

1. **Provisioning time** — how long AWS takes to allocate the host, attach storage, and bring the OS to a running state. For a plain AWS-provided AMI this is measured in **minutes** and is largely outside your control.
2. **Post-launch time** — everything that happens *after* the OS is up: package installs, OS tuning, application deployment, and configuration. This is the part you design around. Left manual, it ranges from minutes to **hours** for a complex stack.

The whole game is shrinking part 2, because it is the slow, variable tail that delays an instance from entering service — and in an Auto Scaling fleet it directly affects how fast you can [scale out](scaling.md) to meet load.

## Three Ways to Handle Post-Launch Work

| Approach | What it does | Trade-off |
| --- | --- | --- |
| **Bootstrapping** | Automates installs and config *after* launch ([user data](../services/ec2.md#bootstrapping) or [cfn-init](../services/cloudformation.md#cfn-init--desired-state-bootstrapping)) | Flexible and repeatable, but the work still happens post-launch, so the time is still spent on every launch |
| **AMI baking** | Front-loads the work into a pre-configured ([golden](../services/ec2.md#ami-baking)) AMI | Removes most post-launch time, but the config is **frozen** into the image — less flexible, and a change means baking a new AMI |
| **Both combined** *(optimal)* | Bake the **time-intensive, slow-changing** part into the AMI; bootstrap the **final, fast-changing** config | Low boot time to service time **and** retained flexibility |

### The diagram

![Three stacked rows comparing approaches. Row 1, plain AMI no automation: an AMI tile points to an Instance in "minutes", followed by an open-ended pink "Post Launch Time" arrow with a clock and no endpoint, representing slow variable manual configuration. Row 2, bootstrapping: AMI to Instance in minutes, then a "Bootstrapping" step does post-launch config automatically — faster and repeatable but still after launch. Row 3, AMI baking: time is spent up front at an "AMI Bake" clock to produce a pre-configured AMI, which then launches to an Instance in minutes with little post-launch work. A dotted box around rows 2 and 3 is labelled "Optimal", indicating that combining baking with bootstrapping gives both low boot time to service time and flexibility](../assets/boot-time-to-service-time-ami-baking-vs-ec2-bootstrapping.png)

Reading the rows top to bottom:

1. **Plain AMI, no automation** — the instance launches in minutes, but then there is an open-ended manual configuration tail. That tail is the slow, variable, error-prone part.
2. **Bootstrapping** — the post-launch config is automated, so it is faster and repeatable than the manual tail, but the work still runs *after* launch on every instance.
3. **AMI baking** — the configuration time is paid **once, up front** to produce a golden image. New instances launch from it in minutes with little or no post-launch work, but the baked-in config is frozen.

The dotted "Optimal" box around rows 2 and 3 is the takeaway: **combine them**. Bake the heavy, slow-changing layers; bootstrap the last-mile config.

## The Combined Pattern in Practice

The canonical example is a "90/10" application: 90% of the time-to-ready is installing a large, stable software stack, and 10% is environment-specific configuration that changes per deploy.

- **Bake the 90%** into an AMI — the OS hardening, the runtime, the heavy dependencies — using a pipeline tool like **EC2 Image Builder** or **Packer**.
- **Bootstrap the 10%** at launch — pull the current app version, write environment-specific config, register with discovery — using [user data](../services/ec2.md#bootstrapping) or [cfn-init](../services/cloudformation.md#cfn-init--desired-state-bootstrapping).

This keeps launches fast (most work is pre-baked) while staying flexible (the volatile bits are still applied fresh each launch, so you are not rebaking an AMI for every config change).

> [!tip] Exam Tip
> Whenever a question pairs **"minimize boot time to service time"** with **"keep flexibility / still apply per-launch config,"** the answer is **combine AMI baking with bootstrapping** — bake the slow, stable part; bootstrap the fast-changing part. "Bake everything" loses flexibility; "bootstrap everything" stays slow on every launch.

> [!tip] Why this matters for scaling
> In an Auto Scaling Group, boot time to service time *is* your responsiveness to load. A 20-minute bootstrap means new capacity is 20 minutes late to every traffic spike; a baked AMI that is ready in 2 minutes lets the group track demand far more tightly. This is a frequent reason to favour AMI baking for fleet members. See [Scaling](scaling.md).

## References

- [EC2 — Bootstrapping](../services/ec2.md#bootstrapping)
- [EC2 — AMI Baking](../services/ec2.md#ami-baking)
- [CloudFormation — cfn-init](../services/cloudformation.md)
- Source: Adrian Cantrill SAA course — Advanced EC2 (Bootstrapping; Boot-time-to-service-time)
