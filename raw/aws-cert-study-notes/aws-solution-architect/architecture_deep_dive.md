---

tags:

- aws
- saa
- architecture
- event-driven
- decoupling

---

# Architecture Deep Dive

## Overview

This lesson walks the **evolution of systems architecture** — from a single black-box application through to a mature, event-driven, serverless design. The worked example throughout is **CatTube**, a cat-video sharing platform (a thinly-veiled YouTube) where users upload a 4K master video and the platform transcodes it into multiple lower resolutions (1080p, 720p, 480p). Transcoding is the resource-intensive part, and it's the piece that drives most of the architectural pressure.

The progression is:

**Monolithic → Tiered → Tiered + Load Balancers → Queue-decoupled (worker fleet) → Microservices → Event-driven**

The through-line is a steady move from **tight coupling + synchronous communication + always-on billing** toward **loose coupling + asynchronous communication + consume-only-when-working**. As a solutions architect, the exam expects you to recognise which stage a scenario describes and to pick the architecture that fits a set of business requirements — so the whole lesson is conceptual scaffolding for later, more AWS-specific topics.

> 📚 _Added beyond transcript:_ The lesson is deliberately vendor-neutral in places (e.g. it says "queue" and "event router" rather than naming SQS or EventBridge). Where an AWS service is the natural mapping, this note flags it with 📚 so you can cross-link, but the exam concepts themselves are the coupling/communication/billing properties, not the service names.

---

## Diagram Walkthrough — Monolithic

![[architecture-monolithic.png]]

**Context bar (top):** Title "Example Architecture" with the CatTube ("Cat~~You~~Tube") branding top-left — this is the running example, not a real service.

**Left column — the workflow:** Bob (the user) uploads a **4K** master video. An arrow flows down into **Processing**, which fans out into three downstream outputs: **480p, 720p, 1080p**. This is showing you _what the application does_ — ingest one master, emit several renditions.

**Centre — the monolith itself:** A single bordered box labelled **Monolithic** containing three stacked bands: **Upload** (red), **Processing** (blue), **Store and Manage** (orange). The single outer border is the whole point — these are logically distinct functions crammed into **one code base on one entity**.

**Right column — the three consequences** (read these as the three things to memorise):

- 🎯 **Fails Together** — one component failing takes down the whole application end-to-end.
- 🎯 **Scales Together** — components are highly coupled, expect to share the same server, and can generally only be **vertically scaled** as a unit.
- 🎯 **Bills Together** — everything is always running and always allocated, so it always incurs charges even when idle.

**Takeaway:** A monolith is one of the _least cost-effective_ ways to architect, because capacity must be provisioned for all components at once and nothing can scale — or bill — independently.

---

## Diagram Walkthrough — Tiered Architecture

![[architecture-tierd.png]]

**Reading order:** The left stack is the tiered application; the right-hand text callouts are the two limitations.

**Left — the tiers, now separated:** **Upload** (red), **Processing** (blue), **Store and Manage** (orange) are now _distinct tiers_ rather than bands in one box. Between them sit small purple **connector icons** with **double-headed vertical arrows** — each tier connects directly to a single endpoint of the next. Tiers can live on the same server or different servers.

**The duplicated "Processing" box + horizontal arrows:** The diagram shows Processing twice, connected by horizontal arrows looping back up to the connector. This is illustrating that Upload talks to a _specific_ Processing endpoint and waits for it — the coupling hasn't gone away, it's just been stretched across tiers.

**Right — the two limitations (the exam-relevant part):**

- 🔴 (dark-red arrow, pointing at the Upload→Processing link): **"Upload EXPECTS and REQUIRES Processing to respond."** The communication is **synchronous** — Upload sends a job and _waits_ for an answer. If Processing fails entirely, Upload fails too.
- 🔴 (dark-red arrow, pointing at the Processing tier): **"Each tier has to be running something for the app to function."** Because communication is synchronous, **you cannot scale any tier back to zero** — there must always be something to answer.

**Takeaway:** Tiering buys you **independent vertical scaling** per tier (make just the Processing server bigger), but the tiers stay **coupled** and **synchronous**, so you still can't scale to zero and a downstream failure still cascades upstream.

> 📚 The transcript describes a _further_ evolution not shown in this diagram: putting **internal load balancers** between the tiers. That abstracts each tier away from specific instances of the next, which unlocks **horizontal scaling** (add more Processing instances) and **high availability** (a failed instance is routed around). But it still doesn't fix the two core problems below.

---

## Diagram Walkthrough — Evolving with Queues

![[architecture-event-driven-1.png]]

**Context bar (top-right):** The Auto Scaling Group parameters — **Min 0, Desired 0→2→1→0, Max 1337.** The struck-through "2" and "1" over "Desired" are showing the value _changing over the lifecycle of the job_ (0 → 2 → 1 → 0), not three separate settings. Min and Desired both start at **0**, meaning **zero instances when the queue is empty**.

**Left — the producer side:** Bob uploads his 4K video. Two blue arrows branch from Upload:

1. The **master 4K video** is written to an **S3 bucket** (green bucket icon).
2. A **message** is added to the **Queue** (the horizontal red rails in the centre), describing _where_ the video lives in S3 and _what_ sizes are required.

Once both are done, **Upload's transaction is complete** — it does not talk to Processing and does not wait. This is the decoupling.

**Centre — the queue:** The red rails with an orange block represent messages waiting in order. The FIRST message is architecturally at the **front** of the queue; later uploads queue up behind it.

**Right — the consumer side (worker fleet):** The dotted orange border is the **Auto Scaling Group**. Inside it, a **CPU/instance icon** feeds the blue **Processing** box. The blue arrow from S3 curving down into the ASG shows workers retrieving the master video from S3 _after_ pulling the job from the queue.

**Footer — the feedback loop:** The dotted red arrows labelled **Queue Length** run from the queue down to the **Auto Scaling Group** icon. This is the control signal: the ASG scales **on the number of messages in the queue**, provisioning instances when messages appear and terminating them when the queue drains.

### The queue lifecycle (as narrated)

1. Upload writes master to S3 + adds message → **transaction done** (asynchronous).
2. Queue length > 0 → ASG scales **Desired 0 → 2**, provisioning two workers.
3. Workers **poll** the queue, receive front messages, fetch masters from S3, transcode.
4. Completed jobs are **deleted** from the queue → shorter queue → ASG scales **2 → 1**, terminating one worker.
5. Last worker processes the final message → **0 messages** → ASG scales **1 → 0**, terminating the last instance.

**Takeaway:** A queue placed _between_ two tiers **decouples** them. The producer adds jobs and doesn't care about the consumer's health; the consumer reads jobs and doesn't care how they got there. Communication is now **asynchronous**, and the **worker fleet can scale from zero to near-infinite** based purely on queue length. This is the "worker fleet" pattern and it's extremely common for upload-then-process workloads.

> ⚠️ **Course-vs-reality discrepancy — queue ordering.** The transcript says messages are received "in most cases" in FIFO order, and this diagram treats the queue as ordered. In **Amazon SQS**, the _default_ queue type is a **Standard queue**, which provides only **best-effort ordering** and **at-least-once delivery** — messages can arrive out of order and occasionally more than once. Strict FIFO + exactly-once processing is a _separate_ queue type you must explicitly choose (a **FIFO queue**, name ending `.fifo`). Don't carry "queues are FIFO by default" into the exam. Source: [AWS SQS Features](https://aws.amazon.com/sqs/features/), current as of 2026.
> 
> 🎯 Exam trigger mapping: _"ordering matters / no duplicates / exactly-once"_ → **SQS FIFO**. _"maximum throughput / order doesn't matter / can tolerate duplicates"_ → **SQS Standard**. Worker code on a Standard queue should be **idempotent** because a message may be delivered more than once.

> 📚 _Non-exhaustive current SQS figures (for context, not memorisation at this depth):_ FIFO throughput is ~300 msg/s without batching or ~3,000 with batching in base mode, with **High Throughput Mode** going far higher; Standard throughput is nearly unlimited. Max payload was **raised from 256 KB to 1 MB in January 2026**. These change over time — verify against AWS docs before relying on a specific number. Source: [AWS "What's New" / SQS docs](https://aws.amazon.com/sqs/features/).

---

## From Queues to Microservices to Event-Driven

Keep breaking a monolith into smaller and smaller independent pieces and you arrive at a **microservices architecture** — a collection of small, self-sufficient applications, each with its own logic, its own data store, and its own I/O. CatTube might have hundreds or thousands of them (either different services, or many copies of the same service).

In this model:

- A **producer** produces data/messages (e.g. the Upload service).
- A **consumer** consumes data/messages (e.g. the Processing service).
- A service can be **both** (e.g. Store and Manage).

The things services produce and consume are **events**. You _can_ wire microservices together with queues — but with many services exchanging data pairwise, you end up needing **a lot of queues**. It works, but it gets complex fast. That complexity is what event-driven architecture is designed to solve.

### What an event actually is

An **event** is a signal that _something happened_:

- A customer clicks **submit** on an order.
- An **error** occurs packing an order or uploading a video.
- An **upload completes** (or fails).

**Producers** generate events _in reaction to something happening_. **Consumers** sit ready and, when they see an event they care about, **take an action and then stop** (display something, dispatch a human, retry an upload). A component can be both — e.g. it emits a "failed upload" event and also consumes events that trigger a retry.

---

## Diagram Walkthrough — Event-Driven Architecture

![[architecture-event-driven-2.png]]

**Overall layout:** Three zones, left to right — **Event Producer** (red-bordered column), **Event Router** (centre), **Event Consumer** (blue-bordered column). The producer and consumer are drawn as tall bordered "swim lanes" containing gear/cog icons (the services); the router sits between them.

**Left — Event Producer:** Cog icons representing producer services. A single **red horizontal arrow** flows _right_ from the producer zone into the Event Router. Producers only emit _when something happens_ — they are not looping or polling.

**Centre — Event Router + Event Bus:** The magenta hexagon-network icon is the **Event Router** — a **highly available central exchange point** for events. Below it, the horizontal rails with scattered coloured blocks are the **Event Bus** — think of it as a constant flow of information. The **two vertical purple arrows** between router and bus (one down, one up) show events being placed onto the bus and pulled from it — the router adds producer events to the bus and delivers them to the right consumers.

**Right — Event Consumer:** More cog icons representing consumer services, waiting. They don't poll; events are **delivered** to them, they act, then return to rest.

**The key visual message:** _Nothing is drawn as a loop._ No component is spinning, waiting, or polling. Resources are consumed **only when an event is being handled**, then the system returns to a dormant, low/no-resource state.

> 📚 _AWS mapping:_ The generic **"event router" with an "event bus"** maps directly to **Amazon EventBridge**, a serverless event bus that ingests events from your apps, SaaS apps, and AWS services and routes them to targets (Lambda, SQS, SNS, Step Functions, etc.) via rules. The transcript keeps it vendor-neutral on purpose, but EventBridge is the service you'll see this pattern named after on the exam and in practice. Source: [Amazon EventBridge](https://aws.amazon.com/eventbridge/).

---

## Mechanics — Why "Only Consume When Working" Matters

The billing contrast is the heart of the lesson. Compare:

- An **EC2 instance** running WordPress is a **consistent allocation of resources**. Whether load is low or high, you're billed for the instance — you're always consuming resources.
- A **mature event-driven architecture** springs into action _only_ when events arrive: it allocates resources, scales components up, handles the events, then **returns to low/no resource usage** as the default state.

So an event-driven system **only consumes resources while handling events**. When nothing is happening, nothing is running, nothing is polling. This "consume only while working" property is a **key component of a serverless architecture** — which the course covers in depth later.

🎯 The amazon.com analogy from the transcript: clicking "order" generates an event and triggers action — but amazon.com is **not** checking your browser every second to see if you clicked. Producers emit on _occurrence_; consumers act on _delivery_.

---

## Comparison Table — The Architecture Progression

> 📚 _Editorial addition for review_ — this consolidates the transcript's progression into one table. Flagged so you can decide whether it belongs in the vault or should stay as a study aid.

|Architecture|Coupling|Comms|Scaling|Scale to zero?|Billing when idle|
|---|---|---|---|---|---|
|**Monolithic**|Tightly coupled (one code base)|Internal / synchronous|Vertical, all-together|❌|Always billed|
|**Tiered**|Coupled (tier→tier endpoint)|Synchronous|Vertical, per-tier|❌|Always billed|
|**Tiered + Load Balancers**|Coupled (via LB)|Synchronous|Vertical **or** horizontal, per-tier; HA|❌ (still needs ≥1 downstream)|Always billed|
|**Queue-decoupled (worker fleet)**|**Decoupled** by queue|**Asynchronous**|Horizontal, 0 → near-∞ on queue length|✅ (consumer side)|Consumer bills only while working|
|**Microservices**|Loosely coupled (per service)|Async (events/queues)|Per service|✅|Per service|
|**Event-driven**|**Loosely coupled** via router|**Asynchronous events**|Elastic, per component|✅ (dormant default)|**Only while handling events**|

---

## Exam Traps

- 🎯 **"Synchronous" is the trap word for coupling.** Tiered (even with load balancers) is still synchronous — the upstream tier _waits_ for a response, so it **cannot scale to zero** and a downstream failure **cascades upstream**. If a scenario needs scale-to-zero or failure isolation, you need a **queue** or **events**, not just a load balancer.
- 🎯 **Load balancer ≠ decoupling.** A load balancer gives HA and horizontal scaling but keeps communication synchronous and coupled. A **queue** is what actually decouples producer from consumer.
- 🎯 **Worker fleet scaling metric is queue length**, not CPU. "Scale the processing fleet based on the number of jobs waiting" → Auto Scaling on **queue length** (Min/Desired 0, worker fleet).
- 🎯 **"Fails together / scales together / bills together"** is the monolith signature. Any of those three phrases in a scenario points at a monolith and its cost/availability downsides.
- ⚠️ **Don't assume queues are FIFO.** SQS **Standard** is best-effort ordering + at-least-once (default); **FIFO** is the explicit choice for strict ordering + exactly-once. See the discrepancy callout above.
- 🎯 **Event-driven = consumes resources only while handling events** → strongly associated with **serverless** and cost efficiency for spiky/idle-heavy workloads.

---

## Quick-Reference Table

|Concept|Definition|Key property|
|---|---|---|
|**Monolithic**|All components in one code base / entity|Fails, scales, bills together|
|**Tiered**|Components split into tiers, direct endpoints|Independent vertical scaling; still coupled + synchronous|
|**Internal load balancer**|LB placed between tiers|Abstracts instances → horizontal scaling + HA; still synchronous|
|**Queue**|System that accepts, stores, and delivers messages|Decouples producer/consumer; enables **async**|
|**Synchronous comms**|Sender waits for an immediate response|Coupling; can't scale to zero|
|**Asynchronous comms**|Sender fires and continues; no waiting|Decoupling; scale to zero possible|
|**Worker fleet**|Consumer ASG polling a queue|Scales 0 → near-∞ on **queue length**|
|**Microservice**|Small self-sufficient app (own logic/data/IO)|Producer, consumer, or both|
|**Event**|Signal that something happened|Click, error, upload complete, etc.|
|**Producer**|Emits events on occurrence|Not polling/waiting|
|**Consumer**|Acts on delivered events, then stops|Not polling/waiting|
|**Event router**|HA central exchange for events (📚 EventBridge)|Delivers events to the right consumers|
|**Event bus**|The flow of events the router reads/writes|Producers add, router delivers|
|**Event-driven architecture**|Producers + router + consumers|Consumes resources **only while handling events** → serverless building block|

---

## Open Questions (for Roger)

1. **Queue depth as its own note?** SQS Standard vs FIFO, at-least-once vs exactly-once, visibility timeout, DLQs, and queue-length-based Auto Scaling are all exam-relevant and go beyond this conceptual lesson. Worth a dedicated `[[SQS]]` note that this note links to? I've kept the discrepancy callout minimal here on purpose.
2. **EventBridge note / cross-link?** This lesson is the conceptual precursor to `[[EventBridge]]` (and arguably `[[SNS]]` and `[[Lambda]]` for the serverless/event-driven pairing). Want me to stub `[[EventBridge]]` and add "see also" links, or wait until you hit that lesson in the course?
3. **Auto Scaling cross-link.** The worker-fleet section leans on Min/Desired/Max and queue-length scaling policies — this connects to your existing Auto Scaling material. Should I add a `[[Auto Scaling Groups]]` wikilink at the worker-fleet lifecycle, or is that note named differently in your vault?
4. **Serverless forward-link.** The lesson explicitly teases serverless ("covered later"). Do you want a placeholder `[[Serverless]]` link now so the graph connects when you reach that section?
5. **Editorial table.** The "Architecture Progression" comparison table is my synthesis, not from the transcript — keep it in the vault, trim it, or drop it?