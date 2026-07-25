---

tags:

- aws
- saa
- eventbridge
- cloudwatch
- serverless
- eventdriven

---
---

tags:

- aws
- saa
- eventbridge
- cloudwatch
- serverless
- eventdriven

---

# EventBridge and CloudWatch Events

## Overview

**EventBridge** (formerly **CloudWatch Events**) is a serverless event bus that delivers a near real-time stream of **system events** describing changes in AWS products and services. When something happens — an EC2 instance is terminated, started, or stopped — the supported service emits an **event**, and EventBridge lets you match that event and route it to a **target**.

The mental model for the whole service is: **"if X happens, or if it's a certain time (Y), then do Z."**

- **X** = a supported service that produces an event (the _producer_)
- **Y** = a certain time or time range, expressed in Unix cron format
- **Z** = a supported target service that receives the event

EventBridge is effectively **"CloudWatch Events v2"**: it uses the **same underlying APIs** and the **same basic architecture**, but adds a superset of functionality — most notably the ability to ingest events from **third-party / SaaS providers** and **custom applications**, which CloudWatch Events cannot do. AWS recommends EventBridge by default for any new deployment. <cite index="6-1">The default event bus and the rules you created in CloudWatch Events also display in the EventBridge console, and EventBridge uses the same CloudWatch Events API so existing code stays the same.</cite>

> ⚠️ **Course-vs-reality (naming & deprecation):** The transcript says CloudWatch Events is _"the service which is replacing"_ and that AWS _"are now starting to encourage a migration."_ That framing is dated. As of now, EventBridge is described by AWS as **the evolution of CloudWatch Events**, and <cite index="6-1">new features are added only to EventBridge, not to CloudWatch Events.</cite> However, AWS's own FAQ is explicit that <cite index="3-1">they are _not_ deprecating the API or the service itself — EventBridge uses the same API and the CloudWatch Events name is being replaced over time.</cite> So "CloudWatch Events is deprecated" (a claim you'll see in third-party blogs) is **stronger than AWS's official wording**. For the exam: treat them as **the same service, different name**, with EventBridge as the current/preferred name. (Sources: AWS EventBridge docs "EventBridge is the evolution of Amazon CloudWatch Events"; AWS EventBridge FAQs.)

---

## Diagram Walkthrough

![[eventbridge-architecture.png]]

**How to read this diagram — context bar → main objects → footer:**

**Top (title/context):** "EventBridge / CloudWatch Events" — the slash reinforces that these are one architecture under two names.

**Main objects, left to right (the event's journey):**

1. **EC2 box (far left, dashed border):** The **producer**. Inside, the red instance icon (stopped) has a pink arrow pointing down to the green "play" instance icon (running). The person icon is **Bob**, who triggers the **"State Change"** (stopped → running). This is the "X happens" step.
    
2. **The pink/magenta EventBridge service icon (center):** EventBridge sits _over the top_ of any event bus it has exposure to and **monitors every event** flowing through it.
    
3. **Arrows between the EventBridge icon and the Default Event Bus (center-bottom) — read the arrow directions carefully:**
    
    - The **blue arrow** runs from the EC2 box down into the bus: this is the **producer publishing** the state-change event onto the bus.
    - The two **purple arrows** run _between_ the EventBridge icon and the bus (one down, one up): EventBridge **observing** the stream of events on the bus. This is the key visual point — EventBridge doesn't _contain_ the bus; it **watches** the bus.
4. **The two rule stacks (right of center):**
    
    - **Event Pattern Rule** (top, with the ✓/✗ checklist icon): matches on the _content/pattern_ of events as they pass through the bus.
    - **Schedule Rule** (bottom, with the green **"24"-clock** icon): matches a **cron-formatted time or range of times** rather than an event's content.
5. **Red arrows from each rule → the orange Lambda icons (far right, labelled "Target"):** when a rule matches, it **routes the matched event to one or more targets**. Lambda is the example target here.
    

**Footer / bottom band — "Default Event Bus":** the horizontal band packed with small coloured blocks is the **stream of events** from all supported services in the account. Each coloured block is one event.

**Far-right "JSON" callout (pink) with the black arrow pointing into the bus:** every event is just a **JSON structure**. The target (e.g. Lambda) receives this JSON, which carries _which_ instance changed, _what_ state it moved to, the _timestamp_, and other detail.

> 📚 **Reading tip (beyond the transcript):** the arrow _colours_ encode roles — **blue = a producer writing to the bus**, **purple = EventBridge observing the bus**, **red = a matched rule delivering to a target**. Tracking colour is the fastest way to answer "who is doing what" on this style of exam diagram.

---

## Mechanics

### The event bus

- A **bus** is a **stream of events** from any supported service in the account.
- Every account has a **default event bus**.
- **CloudWatch Events:** exactly **one** event bus exists, and it is **implicit** — not exposed in the UI. You just look for events and send them to targets.
- **EventBridge:** you can create **additional custom buses** — for your own applications or for third-party/SaaS products — and interact with them the same way as the default bus. **This ability to have multiple buses is EventBridge-only.**

### Rules

You create **rules**, each **linked to a specific event bus** (default unless you say otherwise). Two types:

|Rule type|Matches on|Trigger source|
|---|---|---|
|**Event pattern rule**|The **pattern/content** of events on the bus|An event that matches the pattern|
|**Schedule rule**|A **cron expression** (specific times or ranges)|Clock reaching the specified time|

When a rule matches, it **delivers the matched event to one or more targets** defined on that rule. A common target is **invoke a Lambda function** (non-exhaustive — S3, SQS, Step Functions, SNS, Kinesis, and many others are also valid targets).

### Events are JSON

- Every event is a **JSON structure**.
- The **target consumes the JSON**. For an EC2 state change, the Lambda receives JSON containing the instance ID, the new state, the timestamp, and related metadata.

---

## CloudTrail as an Event Source

> 📚 **Added beyond the transcript** — this relationship is a common exam target and pairs with your **[[CloudTrail]]** note.

There are **two independent relationships** between EventBridge and CloudTrail; keep them separate:

**1. CloudTrail → EventBridge (CloudTrail as a source).** Many AWS services surface their events onto the EventBridge bus **via CloudTrail**. When CloudTrail records an API call, that call can appear on the bus so a rule can react to it. These events carry the field **`detail-type: "AWS API Call via CloudTrail"`** — that exact string is the 🎯 tell that the event reached EventBridge through CloudTrail. (Source: AWS EventBridge docs, per-service event references e.g. IAM/Secrets Manager: _"sends service events to EventBridge via AWS CloudTrail."_)

**2. EventBridge → CloudTrail (EventBridge as a subject).** Separately, CloudTrail audits API calls made **to EventBridge itself** (`PutRule`, `PutTargets`, `DeleteRule`, etc.) — i.e. who created or changed a rule. This is just the normal "CloudTrail audits every service" relationship. Don't blur it with #1.

### Native path vs CloudTrail path

The same underlying action can reach EventBridge by **two different routes**, carrying **different payloads**:

||**Native path**|**CloudTrail path**|
|---|---|---|
|How it arrives|Service pushes its own event straight to EventBridge|API call recorded by CloudTrail, surfaced onto the bus|
|Example|EC2 **instance state-change notification**|`StopInstances` API call via CloudTrail|
|`detail-type`|e.g. `EC2 Instance State-change Notification`|`AWS API Call via CloudTrail`|
|Includes caller identity?|❌ No — won't say _which_ IAM user/role|✅ Yes — includes the requesting principal|

🎯 **If a question hinges on "which IAM user/role made the call," the answer routes through CloudTrail**, not the native service event. (Source: AWS re:Post — the native EC2 event omits the IAM principal; the CloudTrail event contains it.)

### Management vs data events (the opt-in gotcha)

Whether a CloudTrail-sourced event reaches EventBridge depends on whether CloudTrail is **capturing** it in the first place:

- **Management events** (config-level API calls — create IAM role, launch instance, modify RDS): captured by default → reach EventBridge **without** needing a trail.
- **Data events** (high-volume, object-level — S3 `GetObject`/`PutObject`, Lambda `Invoke`): **not** logged by default → you must **configure a trail** that logs them before EventBridge rules can fire.

> ⚠️ Once you opt in (a trail is configured to log those data events **for the relevant resources — right region, right bucket/prefix selectors**), CloudTrail records them and delivers them to EventBridge as `AWS API Call via CloudTrail` events that a rule can match. Two caveats: delivery is **best-effort**, not guaranteed (per AWS docs), and data-event logging is **scoped to the resources you select**, not a blanket account-wide switch. (Source: AWS re:Post on minimal CloudTrail setup for `AWS API Call via CloudTrail` events.)

**One-line model:** API call happens → CloudTrail records it (management by default; data events only if a trail selects them) → it appears on the EventBridge bus tagged `AWS API Call via CloudTrail` → a rule matches it. CloudTrail answers _what happened and who did it_; EventBridge answers _do something about it now_.

---

## Course-vs-Reality Discrepancies

> ⚠️ **1. "Deprecating / replacing" language.** See the Overview callout. AWS's position: EventBridge is the **evolution/successor**; the CloudWatch Events **API and service are not being deprecated** (<cite index="3-1">same API, name being phased over time</cite>), but **new features land only in EventBridge** (<cite index="6-1">per AWS docs</cite>). Exam-safe framing: _same service, EventBridge is the current name and gets all new features._

> ⚠️ **2. Schedule rules are now "legacy"; EventBridge Scheduler is the recommended tool.** The transcript describes **scheduled rules** as the way to do time-based triggers. Current AWS docs label scheduled rules **"(legacy)"** and <cite index="14-1">recommend **EventBridge Scheduler** instead — a separate, more scalable service supporting cron and rate expressions, one-time invocations, flexible time windows, retry limits, time zones, and a wider set of targets.</cite> 🎯 If an exam question distinguishes **Scheduler** from **scheduled rules**, remember Scheduler is the newer, preferred, standalone scheduling service and **does not require an event bus**. (Source: AWS EventBridge docs, "Creating a scheduled rule (legacy)".)

> 📚 **3. Cron _and_ rate expressions.** The transcript only mentions **cron** for time-based rules. AWS supports **two** schedule-expression forms: <cite index="20-1">**rate expressions** (run at a regular interval, e.g. every 20 minutes — simpler) and **cron expressions** (run at specific times/dates — finer control).</cite> 🎯 "Every N minutes/hours" → think **rate**; "at 8am on the first Monday" → think **cron**.

---

## Exam Traps

- 🎯 **"Third-party / SaaS / custom-application events" → EventBridge, never CloudWatch Events.** Ingesting non-AWS events is the flagship capability that distinguishes EventBridge. If the question mentions a SaaS partner (Zendesk, Shopify, etc.) or custom app events, the answer is EventBridge.
- 🎯 **"Multiple / custom event buses" → EventBridge.** CloudWatch Events has exactly one implicit bus.
- 🎯 **"If X happens then do Z" (react to a resource state change) → EventBridge event pattern rule.** e.g., "run a Lambda when an EC2 instance enters `stopped`."
- 🎯 **"Run something on a schedule / cron" → EventBridge schedule rule (or EventBridge Scheduler).** Classic use: nightly Lambda, scheduled snapshots, cron-style automation.
- ⚠️ **Don't confuse the players:** _event bus_ = the stream; _rule_ = the matcher; _target_ = the receiver. The EventBridge service **observes** the bus (purple arrows) — it is not the bus itself.
- 🎯 **Events are JSON** and the **payload is delivered to the target** — targets can act on the event's contents.
- ⚠️ **CloudWatch Events vs CloudWatch Alarms** (easy to blur): **Events/EventBridge** react to _state-change events_ and _schedules_; **Alarms** react to a _metric crossing a threshold_. Different services, different triggers.
- 🎯 **CloudTrail vs EventBridge:** CloudTrail = _audit record_ of what happened and **who did it**; EventBridge = _react now_. If you need the **IAM caller identity**, the event must come via **CloudTrail** (`AWS API Call via CloudTrail`), not the native service event.
- ⚠️ **Management vs data events:** management events reach EventBridge **by default**; **data events** (S3 object-level, Lambda `Invoke`) require a **configured trail** first, and even then delivery is **best-effort** and **scoped to selected resources**.

---

## Quick-Reference Table

|Concept|CloudWatch Events|EventBridge|
|---|---|---|
|Relationship|Original service|Same API, superset of features ("v2")|
|Event buses|**One**, implicit, not in UI|**Default + custom** buses|
|AWS service events|✅|✅|
|Third-party / SaaS events|❌|✅|
|Custom application events|❌|✅|
|Rule types|Event pattern, Schedule|Event pattern, Schedule|
|Schedule expression forms|cron / rate|cron / rate (or **EventBridge Scheduler**)|
|Event format|JSON|JSON|
|Example target|Lambda|Lambda (+ many others)|
|AWS recommendation|—|Use by default for new work|

|Core term|Meaning|
|---|---|
|**Event bus**|Stream of events from supported services in the account|
|**Event**|A JSON structure describing a change / occurrence|
|**Producer (X)**|Supported service that emits an event|
|**Event pattern rule**|Matches on event content, delivers to target|
|**Schedule rule (Y)**|Matches a cron/rate time, delivers to target|
|**Target (Z)**|Service that receives the matched event (e.g. Lambda)|

---

## Open Questions (for Roger)

1. **Cross-links:** This note naturally connects to **[[Lambda]]** (the canonical target and the "do Z" action) and now **[[CloudTrail]]** (event source — a strong bidirectional link, since your CloudTrail note should probably point back here for the `AWS API Call via CloudTrail` reaction path). Want me to add wikilinks to any existing **[[CloudWatch]]**, **[[CloudWatch Logs]]**, or **[[CloudWatch Agent]]** notes so the CloudWatch family is interconnected? I've referenced the Events-vs-Alarms distinction — if you have a CloudWatch Alarms note, that's a good bidirectional link too.
2. **New sub-note?** **EventBridge Scheduler** is now a distinct, exam-relevant service (legacy scheduled rules vs Scheduler). Do you want a separate **[[EventBridge Scheduler]]** note, or is the ⚠️ callout here enough for SAA depth?
3. **Depth check — cron vs rate:** I added rate expressions (📚) since the transcript omitted them. Keep the brief mention, or would you like a small cron/rate syntax cheat-sheet (they show up occasionally as distractor answers)?
4. **Diagram colour note:** I documented the blue/purple/red arrow encoding in the walkthrough (📚). If you'd rather I keep walkthroughs strictly to what's labelled on the slide, tell me and I'll trim the inference.