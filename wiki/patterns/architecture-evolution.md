---
title: Architecture Evolution — Coupling, Queues & Event-Driven
type: pattern
tags: [architecture, decoupling, coupling, monolithic, tiered, microservices, event-driven, queue, sqs, eventbridge, worker-fleet, synchronous, asynchronous, serverless, scale-to-zero, producer, consumer, events]
certs: [aws-saa]
updated: 2026-07-24
---

# Architecture Evolution — Coupling, Queues & Event-Driven

The evolution of a system's architecture, from a single black-box application to a mature event-driven, serverless design. The exam skill is to recognise **which stage a scenario describes** and pick the architecture that fits a set of business requirements.

The running example is **CatTube**, a cat-video sharing platform (a thinly-veiled YouTube). Users upload a **4K master video** and the platform transcodes it into lower resolutions (1080p, 720p, 480p). Transcoding is the resource-intensive step, and it drives most of the architectural pressure.

The progression runs through six stages: **Monolithic**, then **Tiered**, then **Tiered + Load Balancers**, then **Queue-decoupled (worker fleet)**, then **Microservices**, then **Event-driven**. The through-line is a steady move from **tight coupling, synchronous communication, and always-on billing** toward **loose coupling, asynchronous communication, and consume-only-when-working**.

> [!tip] The one mental model
> Three properties change together as you move down the progression: **coupling** (tight to loose), **communication** (synchronous to asynchronous), and **billing** (always-on to only-while-working). Almost every exam question in this area is really asking you to place a scenario on that axis.

## Stage 1 — Monolithic

All functions live in **one code base on one entity**. For CatTube, that is Upload, Processing, and Store-and-Manage crammed into a single application.

![Architecture diagram: a single bordered box labelled Monolithic containing three stacked bands — Upload, Processing, Store and Manage — with a workflow on the left showing a 4K master video fanning out into 480p, 720p, and 1080p renditions, and three consequences on the right: fails together, scales together, bills together](../assets/architecture-monolithic.png)

The single outer border is the whole point: logically distinct functions are one indivisible unit. This produces the **monolith signature** — three consequences to memorise:

- **Fails together** — one component failing takes down the whole application end-to-end.
- **Scales together** — components are highly coupled, share the same server, and can generally only be **vertically scaled** as a unit (see [Scaling — Horizontal vs Vertical](scaling.md)).
- **Bills together** — everything is always running and always allocated, so it always incurs charges even when idle.

A monolith is one of the *least cost-effective* ways to architect: capacity must be provisioned for all components at once, and nothing can scale or bill independently.

> [!tip] Exam Tip — the monolith signature
> "Fails together / scales together / bills together" — any of those three phrases in a scenario points at a monolith and its cost and availability downsides.

## Stage 2 — Tiered

The same functions, now split into **distinct tiers** (Upload, Processing, Store-and-Manage) rather than bands in one box. Each tier connects directly to a single endpoint of the next, and tiers can live on the same server or different servers.

![Architecture diagram: three separated tiers — Upload, Processing, Store and Manage — connected by vertical double-headed arrows through small connector icons, with two dark-red callouts: Upload expects and requires Processing to respond, and each tier has to be running something for the app to function](../assets/architecture-tierd.png)

Tiering buys you **independent vertical scaling per tier** — make just the Processing server bigger — but two limitations remain, and they are the exam-relevant part:

- **The communication is synchronous.** Upload sends a job and *waits* for an answer. If Processing fails entirely, Upload fails too — a downstream failure still cascades upstream.
- **You cannot scale any tier to zero.** Because communication is synchronous, there must always be something running to answer.

So the coupling has not gone away; it has just been stretched across tiers.

## Stage 3 — Tiered + Load Balancers

Placing **internal load balancers** between the tiers abstracts each tier away from specific instances of the next. This unlocks **horizontal scaling** (add more Processing instances) and **high availability** (a failed instance is routed around). See [Global & Regional Architecture](global-regional-architecture.md#web-tier) for the "abstract customers from infrastructure" idea and [ELB](../services/elb.md) for the load-balancer mechanics — an **internal** load balancer between tiers is the concrete AWS implementation.

But a load balancer does **not** fix the two core problems: communication is still **synchronous** and the tiers are still **coupled**, so you still cannot scale to zero and a downstream failure still cascades.

> [!warning] Gotcha — load balancer ≠ decoupling
> A load balancer gives HA and horizontal scaling but keeps communication synchronous and coupled. A **queue** is what actually decouples a producer from a consumer. If a scenario needs scale-to-zero or failure isolation, a load balancer alone is the wrong answer.

## Stage 4 — Queue-Decoupled (Worker Fleet)

A **queue** placed *between* two tiers decouples them. The producer adds jobs and does not care about the consumer's health; the consumer reads jobs and does not care how they got there.

![Architecture diagram: Bob uploads a 4K video; two arrows branch from Upload — one writes the master to an S3 bucket, the other adds a message to a central queue; a dotted-border Auto Scaling Group of Processing workers pulls messages from the queue and fetches masters from S3; dotted Queue Length arrows feed from the queue back to the ASG as the scaling signal, with ASG parameters Min 0, Desired 0 to 2 to 1 to 0, Max 1337](../assets/architecture-event-driven-1.png)

How the CatTube worker-fleet flow runs:

1. Upload writes the master video to an [S3](../services/s3.md) bucket **and** adds a message to the queue describing *where* the video lives and *what* sizes are required. Once both are done, Upload's transaction is complete — it does **not** talk to Processing and does **not** wait. This is the decoupling, and the communication is now **asynchronous**.
2. Queue length rises above zero, so the [Auto Scaling Group](../services/ec2-auto-scaling.md) scales Desired from 0 to 2, provisioning two workers.
3. Workers **poll** the queue, receive front messages, fetch the masters from S3, and transcode.
4. Completed jobs are **deleted** from the queue. A shorter queue lets the ASG scale from 2 to 1.
5. The last worker processes the final message. With zero messages, the ASG scales from 1 to 0, terminating the last instance.

Both the ASG's **Min and Desired start at 0**, meaning **zero instances when the queue is empty**. This is the "worker fleet" pattern, and it is extremely common for upload-then-process workloads: the worker fleet can scale **from zero to near-infinite** based purely on queue length, and the consumer side **bills only while working**.

> [!tip] Exam Tip — the scaling metric is queue length
> "Scale the processing fleet based on the number of jobs waiting" means Auto Scaling on **queue length** (specifically the SQS `ApproximateNumberOfMessagesVisible` metric), with a worker fleet whose Min and Desired are 0 — **not** CPU-based scaling. See [EC2 Auto Scaling — SQS worker pool](../services/ec2-auto-scaling.md).

> [!warning] Gotcha — do not assume queues are FIFO
> In **Amazon SQS**, the *default* queue type is a **Standard queue**, which provides only **best-effort ordering** and **at-least-once delivery** — messages can arrive out of order and occasionally more than once. Strict ordering with exactly-once processing is a *separate* queue type you must explicitly choose (a **FIFO queue**, name ending `.fifo`). Do not carry "queues are FIFO by default" into the exam. Worker code on a Standard queue should be **idempotent** because a message may be delivered more than once. Source: [AWS SQS Features](https://aws.amazon.com/sqs/features/), current as of 2026.

The SQS pattern-match for the exam:

| Requirement | Queue type |
| --- | --- |
| Ordering matters / no duplicates / exactly-once | **SQS FIFO** |
| Maximum throughput / order does not matter / can tolerate duplicates | **SQS Standard** (make consumers idempotent) |

> [!warning] Verify before relying — SQS figures change
> Rough current SQS figures, for context rather than memorisation: FIFO throughput is about 300 msg/s without batching or about 3,000 with batching in base mode, with **High Throughput Mode** going far higher; Standard throughput is near-unlimited. The **source note also states the max payload was raised from 256 KB to 1 MB in January 2026** — treat that as unverified here and confirm against AWS docs before relying on it. Source: [AWS SQS Features](https://aws.amazon.com/sqs/features/).

## Stage 5 — Microservices

Keep breaking a monolith into smaller and smaller independent pieces and you arrive at a **microservices architecture** — a collection of small, self-sufficient applications, each with its own logic, its own data store, and its own I/O. CatTube might have hundreds or thousands of them.

In this model:

- A **producer** produces data or messages (for example, the Upload service).
- A **consumer** consumes data or messages (for example, the Processing service).
- A service can be **both** (for example, Store-and-Manage).

The things services produce and consume are **events**. You *can* wire microservices together with queues, but with many services exchanging data pairwise, you end up needing **a lot of queues**. It works, but it gets complex fast — and that complexity is exactly what event-driven architecture is designed to solve.

## Stage 6 — Event-Driven

An **event** is a signal that *something happened*: a customer clicks submit on an order, an error occurs packing an order or uploading a video, an upload completes or fails.

- **Producers** generate events *in reaction to something happening* — they are not looping or polling.
- **Consumers** sit ready and, when they see an event they care about, **take an action and then stop** (display something, dispatch a human, retry an upload).
- A component can be **both** — for example, it emits a "failed upload" event and also consumes events that trigger a retry.

![Architecture diagram: three zones left to right — Event Producer swim lane with service cogs emitting a single event rightward, a central Event Router (hexagon-network icon) sitting above an Event Bus of flowing messages with arrows placing events onto the bus and pulling them off, and an Event Consumer swim lane of waiting service cogs to which events are delivered; nothing is drawn as a loop](../assets/architecture-event-driven-2.png)

The architecture has three zones:

- **Event producers** emit events only when something happens.
- The **event router** is a highly available central exchange point. It sits above an **event bus** — a constant flow of information — placing producer events onto the bus and delivering them to the right consumers.
- **Event consumers** do not poll; events are **delivered** to them, they act, then return to rest.

The key visual message is that **nothing is drawn as a loop**. No component is spinning, waiting, or polling. Resources are consumed **only when an event is being handled**, then the system returns to a dormant, low-or-no-resource state.

> [!tip] AWS mapping — EventBridge
> The generic "event router with an event bus" maps directly to **[Amazon EventBridge](../services/eventbridge.md)**, a serverless event bus that ingests events from your apps, SaaS apps, and AWS services and routes them to targets (Lambda, SQS, [SNS](../services/sns.md), [Step Functions](../services/step-functions.md), and others) via rules. The lesson stays vendor-neutral on purpose, but EventBridge is the service this pattern is named after in practice. Source: [Amazon EventBridge](https://aws.amazon.com/eventbridge/).

## Why "Only Consume When Working" Matters

The billing contrast is the heart of the lesson:

- An **[EC2](../services/ec2.md) instance** running WordPress is a **consistent allocation of resources**. Whether load is low or high, you are billed for the instance — you are always consuming resources.
- A **mature event-driven architecture** springs into action *only* when events arrive: it allocates resources, scales components up, handles the events, then **returns to low-or-no resource usage** as its default state.

So an event-driven system **only consumes resources while handling events**. When nothing is happening, nothing is running and nothing is polling. This "consume only while working" property is a **key building block of serverless architecture** — and the compute primitive that embodies it is [AWS Lambda](../services/lambda.md), which runs your function only when invoked and bills only for its execution duration. For the concrete, service-by-service realisation of this end state, see the [Serverless Architecture](serverless-architecture.md) pattern.

The amazon.com analogy: clicking "order" generates an event and triggers action, but amazon.com is **not** checking your browser every second to see if you clicked. Producers emit on *occurrence*; consumers act on *delivery*.

## The Progression at a Glance

| Architecture | Coupling | Communication | Scaling | Scale to zero? | Billing when idle |
| --- | --- | --- | --- | --- | --- |
| **Monolithic** | Tightly coupled (one code base) | Internal / synchronous | Vertical, all together | No | Always billed |
| **Tiered** | Coupled (tier-to-tier endpoint) | Synchronous | Vertical, per tier | No | Always billed |
| **Tiered + Load Balancers** | Coupled (via LB) | Synchronous | Vertical or horizontal, per tier; HA | No (needs at least 1 downstream) | Always billed |
| **Queue-decoupled (worker fleet)** | **Decoupled** by queue | **Asynchronous** | Horizontal, 0 to near-infinite on queue length | Yes (consumer side) | Consumer bills only while working |
| **Microservices** | Loosely coupled (per service) | Async (events / queues) | Per service | Yes | Per service |
| **Event-driven** | **Loosely coupled** via router | **Asynchronous events** | Elastic, per component | Yes (dormant default) | **Only while handling events** |

## Exam Traps

- **"Synchronous" is the trap word for coupling.** Tiered — even with load balancers — is still synchronous: the upstream tier *waits* for a response, so it cannot scale to zero and a downstream failure cascades upstream. Scale-to-zero or failure isolation needs a **queue** or **events**, not just a load balancer.
- **Load balancer ≠ decoupling.** A load balancer gives HA and horizontal scaling but keeps communication synchronous and coupled. A **queue** is what decouples producer from consumer.
- **Worker-fleet scaling metric is queue length**, not CPU. "Scale on the number of jobs waiting" means Auto Scaling on queue length, with a Min/Desired-0 worker fleet.
- **"Fails together / scales together / bills together"** is the monolith signature.
- **Do not assume queues are FIFO.** SQS **Standard** is best-effort ordering and at-least-once (the default); **FIFO** is the explicit choice for strict ordering and exactly-once.
- **Event-driven = consumes resources only while handling events**, so it is strongly associated with **serverless** and cost efficiency for spiky, idle-heavy workloads.

## Key Takeaways

1. **The progression is one axis.** Monolithic to event-driven is a steady move along coupling (tight to loose), communication (sync to async), and billing (always-on to only-while-working).
2. **A monolith fails, scales, and bills together** — the least cost-effective shape, and its signature phrase on the exam.
3. **Tiering and load balancers do not decouple.** They add per-tier and horizontal scaling plus HA, but communication stays synchronous, so no scale-to-zero and failures still cascade.
4. **A queue is the decoupler.** It makes communication asynchronous and lets a worker fleet scale from zero to near-infinite on queue length, billing only while working.
5. **Event-driven is the mature end state** and a core serverless building block: producers emit on occurrence, a highly available router delivers to consumers, and resources are consumed only while handling events.

## References

- [Scaling — Horizontal vs Vertical](scaling.md)
- [Global & Regional AWS Architecture](global-regional-architecture.md)
- [EC2 Auto Scaling — Auto Scaling Groups](../services/ec2-auto-scaling.md)
- [ELB — Elastic Load Balancer](../services/elb.md)
- [S3](../services/s3.md)
- [Amazon SQS Features](https://aws.amazon.com/sqs/features/)
- [Amazon EventBridge](https://aws.amazon.com/eventbridge/)
- Adrian Cantrill, AWS SAA course — Architecture Deep Dive
- Source: `raw/aws-cert-study-notes/aws-solution-architect/architecture_deep_dive.md`
