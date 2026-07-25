---
title: Amazon EventBridge (CloudWatch Events)
type: service
tags: [eventbridge, cloudwatch-events, serverless, event-driven, event-bus, rules, scheduler, lambda, cron, rate, saa]
certs: [aws-saa]
updated: 2026-07-25
---

# Amazon EventBridge (CloudWatch Events)

**EventBridge** (formerly **CloudWatch Events**) is a serverless **event bus** that delivers a near real-time stream of **system events** describing changes in AWS products and services. When something happens — an [EC2](ec2.md) instance is terminated, started, or stopped — the supported service emits an **event**, and EventBridge lets you match that event and route it to a **target**.

The mental model for the whole service is: **"if X happens, or if it's a certain time (Y), then do Z."**

- **X** — a supported service that produces an event (the *producer*)
- **Y** — a certain time or time range, expressed as a schedule expression
- **Z** — a supported target service that receives the event (commonly [Lambda](lambda.md))

This is the AWS implementation of the generic **event router + event bus** stage of the [event-driven architecture](../patterns/architecture-evolution.md#stage-6--event-driven) progression.

## EventBridge is "CloudWatch Events v2"

EventBridge uses the **same underlying APIs** and the **same basic architecture** as CloudWatch Events, but adds a superset of functionality — most notably the ability to ingest events from **third-party / SaaS providers** and **custom applications**, which CloudWatch Events cannot do. AWS recommends EventBridge by default for any new deployment. The default event bus and the rules you created in CloudWatch Events also display in the EventBridge console, and [EventBridge uses the same CloudWatch Events API so existing code keeps working](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-what-is.html).

> [!warning] Course-vs-reality — naming & "deprecation"
> Older courses describe CloudWatch Events as *"the service which is replacing"* and say AWS *"are now starting to encourage a migration."* That framing is dated. AWS now describes EventBridge as **the evolution of Amazon CloudWatch Events**, and [new features are added only to EventBridge](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-what-is.html), not to CloudWatch Events. But AWS's own FAQ is explicit that [they are *not* deprecating the API or the service itself — EventBridge uses the same API and the CloudWatch Events name is being replaced over time](https://aws.amazon.com/eventbridge/faqs/). So "CloudWatch Events is deprecated" (a claim common in third-party blogs) is **stronger than AWS's official wording**. For the exam: treat them as **the same service, different name**, with EventBridge as the current and preferred name.

## Architecture

![EventBridge / CloudWatch Events architecture: an EC2 producer (Bob triggers a stopped-to-running state change) publishes an event onto the Default Event Bus; the EventBridge service icon observes the bus; an Event Pattern Rule and a Schedule Rule each route matched events as JSON to Lambda targets](../assets/eventbridge-architecture.png)

Read the diagram left to right, following the event's journey:

1. **EC2 box (far left, the producer).** Bob triggers a **State Change** on an instance (stopped to running). This is the "X happens" step.
2. **The EventBridge service icon (center).** EventBridge sits *over the top* of any event bus it has exposure to and **monitors every event** flowing through it.
3. **Arrows between the service icon and the Default Event Bus.** The **blue arrow** is the **producer publishing** the state-change event onto the bus. The two **purple arrows** are EventBridge **observing** the stream of events on the bus. The key visual point: EventBridge does not *contain* the bus; it **watches** the bus.
4. **The two rule stacks (right of center).** An **Event Pattern Rule** (checklist icon) matches on the *content* of events as they pass through the bus. A **Schedule Rule** (clock icon) matches a **time or range of times** rather than an event's content.
5. **Red arrows from each rule to the Lambda targets (far right).** When a rule matches, it **routes the matched event to one or more targets**. Lambda is the example target here.

The horizontal band packed with small coloured blocks at the bottom is the **Default Event Bus** — the stream of events from all supported services in the account. Each coloured block is one event. The **JSON callout** on the far right is the reminder that every event is just a JSON structure; the target (for example Lambda) receives this JSON, which carries *which* instance changed, *what* state it moved to, the *timestamp*, and other detail.

> [!note] Reading tip — arrow colours encode roles
> Blue is a **producer writing to the bus**, purple is **EventBridge observing the bus**, red is a **matched rule delivering to a target**. Tracking colour is the fastest way to answer "who is doing what" on this style of exam diagram.

## Mechanics

### The event bus

- A **bus** is a **stream of events** from any supported service in the account.
- Every account has a **default event bus**.
- **CloudWatch Events:** exactly **one** event bus exists, and it is **implicit** — not exposed in the UI. You just look for events and send them to targets.
- **EventBridge:** you can create **additional custom buses** — for your own applications or for third-party / SaaS products — and interact with them the same way as the default bus. **This ability to have multiple buses is EventBridge-only.**

### Rules

You create **rules**, each **linked to a specific event bus** (the default unless you say otherwise). There are two types:

| Rule type | Matches on | Trigger source |
| --- | --- | --- |
| **Event pattern rule** | The **pattern / content** of events on the bus | An event that matches the pattern |
| **Schedule rule** | A **schedule expression** (specific times or ranges) | The clock reaching the specified time |

When a rule matches, it **delivers the matched event to one or more targets** defined on that rule. A common target is **invoke a [Lambda](lambda.md) function** — non-exhaustive: [S3](s3.md), SQS, Step Functions, SNS, Kinesis, and many others are also valid targets.

### Events are JSON

Every event is a **JSON structure**, and the **target consumes the JSON**. For an EC2 state change, the Lambda receives JSON containing the instance ID, the new state, the timestamp, and related metadata — so targets can act on the event's contents.

## CloudTrail as an Event Source

There are **two independent relationships** between EventBridge and [CloudTrail](cloudtrail.md); keep them separate.

1. **CloudTrail as a source (CloudTrail feeds EventBridge).** Many AWS services surface their events onto the EventBridge bus **via CloudTrail**. When CloudTrail records an API call, that call can appear on the bus so a rule can react to it. These events carry the field [`detail-type: "AWS API Call via CloudTrail"`](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-service-event.html) — that exact string is the tell that the event reached EventBridge through CloudTrail.
2. **EventBridge as a subject (CloudTrail audits EventBridge).** Separately, CloudTrail audits API calls made **to EventBridge itself** (`PutRule`, `PutTargets`, `DeleteRule`, and so on) — who created or changed a rule. This is just the normal "CloudTrail audits every service" relationship. Don't blur it with the first.

### Native path vs CloudTrail path

The same underlying action can reach EventBridge by **two different routes**, carrying **different payloads**:

| | Native path | CloudTrail path |
| --- | --- | --- |
| How it arrives | Service pushes its own event straight to EventBridge | API call recorded by CloudTrail, surfaced onto the bus |
| Example | EC2 **instance state-change notification** | `StopInstances` API call via CloudTrail |
| `detail-type` | e.g. `EC2 Instance State-change Notification` | `AWS API Call via CloudTrail` |
| Includes caller identity? | **No** — won't say *which* IAM user/role | **Yes** — includes the requesting principal |

> [!tip] Exam Tip — who made the call?
> If a question hinges on **"which IAM user/role made the call,"** the answer routes through **CloudTrail** (`AWS API Call via CloudTrail`), not the native service event — the native event omits the IAM principal.

### Management vs data events (the opt-in gotcha)

Whether a CloudTrail-sourced event reaches EventBridge depends on whether CloudTrail is **capturing** it in the first place (see [CloudTrail event categories](cloudtrail.md#event-types)):

- **Management events** (config-level API calls — create IAM role, launch instance, modify RDS) are captured by default, so they reach EventBridge **without** needing a trail.
- **Data events** (high-volume, object-level — S3 `GetObject` / `PutObject`, Lambda `Invoke`) are **not** logged by default, so you must **configure a trail** that logs them before EventBridge rules can fire.

> [!warning] Gotcha — data events need a trail, and delivery is best-effort
> Once you opt in (a trail is configured to log those data events **for the relevant resources — right region, right bucket/prefix selectors**), CloudTrail records them and delivers them to EventBridge as `AWS API Call via CloudTrail` events that a rule can match. Two caveats: delivery is **best-effort**, not guaranteed, and data-event logging is **scoped to the resources you select**, not a blanket account-wide switch.

**One-line model:** an API call happens, CloudTrail records it (management by default; data events only if a trail selects them), it appears on the EventBridge bus tagged `AWS API Call via CloudTrail`, and a rule matches it. CloudTrail answers *what happened and who did it*; EventBridge answers *do something about it now*.

## Scheduling: rules vs EventBridge Scheduler

> [!warning] Course-vs-reality — scheduled rules are now "legacy"
> Older courses describe **scheduled rules** as the way to do time-based triggers. Current AWS docs label scheduled rules **"(legacy)"** and [recommend **EventBridge Scheduler** instead](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-create-rule-schedule.html) — a separate, more scalable service supporting cron and rate expressions, one-time invocations, flexible time windows, retry limits, time zones, and a wider set of targets. EventBridge Scheduler is the newer, preferred, standalone scheduling service and **does not require an event bus**. If an exam question distinguishes **Scheduler** from **scheduled rules**, that is the discriminator.

Both scheduled rules and EventBridge Scheduler accept **two** schedule-expression forms:

- [**Rate expressions**](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-scheduled-rule-pattern.html) run at a regular interval (for example every 20 minutes) — simpler.
- **Cron expressions** run at specific times or dates — finer control.

> [!tip] Exam Tip — rate vs cron
> "Every N minutes / hours" points to a **rate** expression; "at 8am on the first Monday" points to a **cron** expression.

## EventBridge vs CloudWatch Alarms

Do not blur these two CloudWatch-family capabilities:

- **EventBridge / CloudWatch Events** react to **state-change events** and **schedules**.
- **[CloudWatch Alarms](cloudwatch.md#metric-filters-and-alarms)** react to a **metric crossing a threshold**.

Different services, different triggers. (For the reverse relationship — reacting quickly to API activity — [CloudTrail is not real-time, so you put an EventBridge rule on management events](cloudtrail.md) to trigger fast responses.)

## Exam Traps

> [!tip] Exam Tip — trigger keywords
>
> - **Third-party / SaaS / custom-application events** points to **EventBridge**, never CloudWatch Events. Ingesting non-AWS events (Zendesk, Shopify, a custom app) is the flagship capability that distinguishes EventBridge.
> - **Multiple / custom event buses** points to **EventBridge**. CloudWatch Events has exactly one implicit bus.
> - **"If X happens then do Z"** (react to a resource state change, e.g. "run a Lambda when an EC2 instance enters `stopped`") points to an **EventBridge event pattern rule**.
> - **"Run something on a schedule / cron"** (nightly Lambda, scheduled snapshots, cron-style automation) points to an **EventBridge schedule rule** or **EventBridge Scheduler**.
> - **Events are JSON** and the **payload is delivered to the target** — targets can act on the event's contents.

> [!warning] Gotcha — don't confuse the players
> *Event bus* = the stream; *rule* = the matcher; *target* = the receiver. The EventBridge service **observes** the bus (the purple arrows) — it is **not** the bus itself.

## Quick Reference

| Concept | CloudWatch Events | EventBridge |
| --- | --- | --- |
| Relationship | Original service | Same API, superset of features ("v2") |
| Event buses | **One**, implicit, not in UI | **Default + custom** buses |
| AWS service events | Yes | Yes |
| Third-party / SaaS events | No | Yes |
| Custom application events | No | Yes |
| Rule types | Event pattern, Schedule | Event pattern, Schedule |
| Schedule expression forms | cron / rate | cron / rate (or **EventBridge Scheduler**) |
| Event format | JSON | JSON |
| Example target | Lambda | Lambda (and many others) |
| AWS recommendation | — | Use by default for new work |

| Core term | Meaning |
| --- | --- |
| **Event bus** | Stream of events from supported services in the account |
| **Event** | A JSON structure describing a change or occurrence |
| **Producer (X)** | Supported service that emits an event |
| **Event pattern rule** | Matches on event content, delivers to target |
| **Schedule rule (Y)** | Matches a cron / rate time, delivers to target |
| **Target (Z)** | Service that receives the matched event (e.g. Lambda) |

## References

- [Lambda](lambda.md) — the canonical EventBridge target and the "do Z" action
- [CloudWatch](cloudwatch.md) — the metrics/logs/alarms platform EventBridge split out from; contrast Alarms (metric threshold) with EventBridge (events + schedules)
- [CloudTrail](cloudtrail.md) — put an EventBridge rule on management events for fast reactions, since CloudTrail delivery is not real-time
- [Architecture Evolution — event-driven stage](../patterns/architecture-evolution.md#stage-6--event-driven) — the generic event router + bus pattern EventBridge implements
- [Amazon EventBridge — What is EventBridge](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-what-is.html)
- [Amazon EventBridge FAQs](https://aws.amazon.com/eventbridge/faqs/)
- [Creating an EventBridge rule that runs on a schedule](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-create-rule-schedule.html)
- Source: `raw/aws-cert-study-notes/aws-solution-architect/eventbridge_and_cloudwatch_events.md`
