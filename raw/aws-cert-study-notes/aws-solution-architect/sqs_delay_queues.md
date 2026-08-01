---

tags:

- aws
- saa
- sqs
- messaging
- decoupling

---

# SQS Delay Queues

## Overview

**Delay queues** let you _postpone the delivery of new messages to consumers_ for a configured period of time. This is a small, self-contained exam topic — but the exam loves to test it _against_ visibility timeouts, because superficially the two features look similar (both hide messages from consumers) while mechanically they are completely different.

The one-line distinction to lock in:

> **Delay queue** → a message is hidden **the moment it's added** to the queue (before anyone ever sees it). **Visibility timeout** → a message is hidden **only after a consumer receives it** (it was already visible first).

Everything else in this note hangs off that difference.

---

## Diagram Walkthrough

![[sqs-visibility-timeout-vs-delay-queues.png]]

The diagram is split into two horizontal bands, each with the SQS queue drawn as a **left-to-right time axis** (the pink arrow labelled _"SQS Queue >> Time"_). Read each band left → right as the life of a message.

### Top band — Visibility Timeout (refresher)

Reading along the timeline:

- **`SendMessage`** (far left, pink down-arrow) — a message (the box icon) is _added to the queue_. At this point it is **immediately visible**.
- **`ReceiveMessage` → Processing** — a consumer polls and receives the message. The **eye icon** marks the point where the message becomes invisible, and the blue **`ReceiveMessage → No Messages`** arrow shows that _further_ receive calls now return nothing.
- The blue **Visibility Timeout (In Seconds)** block is the invisibility window. Below it, the green arrows show the intended happy path: **`Processing` → `DeleteMessage`** (message consumed successfully and explicitly removed).
- **"Or not ….." (trash icon)** — if the consumer _fails_ / crashes and never deletes, the timeout expires and the message **reappears on the queue** (red arrow, right side) for **automatic reprocessing**.
- Footer facts (right): **Default 30s**, valid range **0 seconds → 12 hours**, set **per-queue or per-message**, and `ChangeMessageVisibility` changes the value **on a per-message basis** for an in-flight message.

📚 **Precision point on "per-queue vs per-message":** The transcript says the value "can be changed on a per queue or per message basis … with the change message visibility operation." Slightly more precisely: the _queue-level default_ visibility timeout is set as a **queue attribute** (`SetQueueAttributes` / at creation). `ChangeMessageVisibility` specifically adjusts the timeout of a **single in-flight message** you've already received — it's the per-message lever, not the per-queue one. ([SQS visibility timeout docs](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html), [ChangeMessageVisibility API](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/APIReference/API_ChangeMessageVisibility.html))

### Bottom band — Delay Queue

Reading along the timeline:

- **`SendMessage`** (left) — the message is added, but instead of landing on the visible queue it is immediately routed (dotted red arrow) into a **`Delay (in Seconds)`** holding period (the pink block with the two clock icons). During this window the message is **parked / invisible**.
- **`ReceiveMessage → No Messages`** (blue up-arrow, below the delay block) — while the delay is running, receive calls return nothing, _even though nobody has consumed the message yet_. This is the key contrast with the top band.
- Once the delay expires, the message becomes **visible on the queue** and behaves like a normal message from there.
- Footer facts (right): a delay queue has a **`DelaySeconds`** set; messages are invisible for `DelaySeconds`; **min = 0, max = 15 minutes**; and **message timers** allow a **per-message invisibility** that overrides the queue setting (also min 0 / max 15 min) — **not supported on FIFO queues**.

---

## Mechanics — Deep Dive

### Visibility timeout (what it's actually for)

Visibility timeout exists for **error correction and automatic reprocessing**. The flow:

1. Consumer calls `ReceiveMessage` and starts work.
2. The message goes invisible for the visibility-timeout duration (default **30s**).
3. **Success** → the message is deleted. Two deletion modes: 📚 **manual** via the `DeleteMessage` action, or **automatic** on successful processing (supported in certain AWS SDKs).
4. **Failure / crash** → the consumer may be in no state to tell the queue anything. When the timeout expires, the message **reappears automatically** and another consumer can retry.

- Configurable **per-queue** (default) or **per-message** (in-flight, via `ChangeMessageVisibility`).
- Range **0 seconds → 12 hours**; default **30 seconds**. ([ChangeMessageVisibility API](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/APIReference/API_ChangeMessageVisibility.html))
- 📚 Setting `VisibilityTimeout` to **0** _terminates_ the timeout immediately — handy for "I can't process this, let someone else try right now." ([SQS visibility timeout docs](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html))
- 📚 Extending an in-flight message can't push its total invisibility past the **12-hour** ceiling measured from the original receive — the clock doesn't reset. For work that needs longer, AWS docs specifically recommend **AWS Step Functions** or breaking the task into smaller steps.
- 📚 Persistent failures (messages that fail repeatedly and keep reappearing) are the textbook use case for a **Dead-Letter Queue** — pair visibility timeout with a DLQ + `maxReceiveCount` so poison messages get pulled off the main queue instead of looping forever.

### Delay queues (what they're actually for)

Delay queues exist to **build a delay into your processing** — e.g. give a downstream system time to catch up, or insert a gap between a customer action and the processing of the message representing it.

- Controlled by **`DelaySeconds`** on the queue.
- **Default is 0** (a queue is only a _delay_ queue when `DelaySeconds` is non-zero).
- **Maximum is 15 minutes** (900 seconds). ([SQS delay queues docs](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-delay-queues.html))
- Messages are invisible for `DelaySeconds` **from the moment they're added** — no consumer involvement required.

### Message timers (per-message delay)

- Set an initial invisibility period on an **individual message** instead of the whole queue.
- Same bounds: **min 0 / max 15 minutes**.
- A message timer **overrides** the queue's `DelaySeconds` for that message. ([SQS message timers docs](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-message-timers.html))
- 🎯 **Not supported on FIFO queues** — FIFO queues do **not** support message timers on individual messages. (Delay _queues_ themselves still work on FIFO; it's the per-message timer that's unsupported.)

---

## Enrichment / Beyond the Transcript

📚 **EventBridge Scheduler is now AWS's recommended answer for delays > 15 minutes.** Both delay queues and message timers cap out at 15 minutes. Current AWS docs explicitly point you to **EventBridge Scheduler** when you need longer or more flexible scheduling — it can schedule one-time or recurring API actions with no 15-minute ceiling. If an exam question describes needing a delay _longer than 15 minutes_, SQS delay queues are the **wrong** answer. ([SQS delay queues docs](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-delay-queues.html)) _(Cross-links to the deferred EventBridge Scheduler sub-note flagged in earlier sessions.)_

📚 **Per-queue `DelaySeconds` retroactivity differs by queue type** — a subtle detail the course doesn't mention:

- **Standard queues:** changing the queue's delay setting is **not retroactive** — messages already in the queue keep their original delay.
- **FIFO queues:** changing the queue's delay setting **is retroactive** — it affects messages already in the queue. ([SQS delay queues docs](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-delay-queues.html))

📚 **In-flight message quotas (visibility-timeout side).** _In-flight_ = received but not yet deleted. Standard queues cap at **~120,000** in-flight messages (soft limit, quota-increase eligible); FIFO limits depend on active message groups rather than a flat number. Behaviour at the cap depends on polling mode:

- **Short polling** → SQS returns an **`OverLimit`** error; no new messages until some in-flight ones are deleted.
- **Long polling** → no error; SQS simply returns nothing until the in-flight count drops below the limit. ([SQS visibility timeout docs](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html))

📚 **Visibility timeout is not a dedup guarantee.** Because SQS standard queues use an **at-least-once** delivery model, the visibility timeout reduces — but does **not** absolutely prevent — the same message being delivered more than once. Build consumers to be **idempotent**. For **FIFO** queues, messages sharing a **message group ID** are processed in strict sequence: while one is in-flight, later messages in that group wait until it's deleted or its timeout expires (the group isn't locked indefinitely — it advances as each message clears). ([SQS visibility timeout docs](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html))

_(No ⚠️ course-vs-reality discrepancies on the core numbers this lesson — the 30s / 0–12h / 0–15min figures and the FIFO message-timer exclusion all match current AWS documentation as of 2026.)_

---

## Exam Traps 🎯

| Trigger phrase in the question                                       | What they're testing              | Correct read                                           |
| -------------------------------------------------------------------- | --------------------------------- | ------------------------------------------------------ |
| "hidden **the moment it's added** / before any consumer sees it"     | Delay queue vs visibility timeout | **Delay queue**                                        |
| "message **reappears** for **reprocessing** after a consumer failed" | Visibility timeout vs delay queue | **Visibility timeout**                                 |
| "need a delay of **X, where X > 15 minutes**"                        | Delay queue limit                 | **Not** a delay queue → **EventBridge Scheduler**      |
| "per-**message** delay on a **FIFO** queue"                          | Message timer support             | **Not supported** on FIFO                              |
| "default invisibility period of a newly received message"            | Visibility timeout default        | **30 seconds**                                         |
| "maximum time a received message can stay invisible"                 | Visibility timeout max            | **12 hours**                                           |
| "maximum delay before a new message becomes visible"                 | Delay queue / message timer max   | **15 minutes**                                         |
| "make an in-flight message immediately available again"              | Terminating visibility timeout    | `ChangeMessageVisibility` with `VisibilityTimeout = 0` |

🎯 The single most common trap: conflating the two features because both "hide messages." Anchor on **when** the hiding starts — _on add_ (delay) vs _on receive_ (visibility timeout).

---

## Quick-Reference Comparison

|                                    | **Visibility Timeout**                        | **Delay Queue**                               |
| ---------------------------------- | --------------------------------------------- | --------------------------------------------- |
| **When message becomes invisible** | After a consumer **receives** it              | The moment it's **added** to the queue        |
| **Was it visible first?**          | Yes                                           | No — invisible from the start                 |
| **Primary purpose**                | Error correction / **automatic reprocessing** | Build a **processing delay** into the app     |
| **Setting**                        | `VisibilityTimeout`                           | `DelaySeconds`                                |
| **Default**                        | **30 seconds**                                | **0 seconds** (non-zero → it's a delay queue) |
| **Range**                          | **0 seconds → 12 hours**                      | **0 → 15 minutes**                            |
| **Per-message override**           | `ChangeMessageVisibility` (in-flight)         | **Message timers** (0–15 min)                 |
| **FIFO per-message support**       | n/a                                           | Message timers **not supported**              |
| **Need longer than the max?**      | **Step Functions** / split into smaller steps | **EventBridge Scheduler**                     |

---

## Open Questions for Roger

1. **Cross-links** — do the following notes already exist in the vault, or should they be flagged as to-create? `[[SQS]]` (parent/overview), `[[SQS FIFO Queues]]`, `[[SQS Dead-Letter Queues]]`, `[[EventBridge Scheduler]]`. I referenced EventBridge Scheduler and FIFO but left them unbracketed pending your call.
2. **Sub-note depth** — want a standalone `[[SQS Visibility Timeout]]` note (heartbeat/renewal patterns, in-flight message limits, the 12-hour extension clock), or is the refresher-level treatment here sufficient for exam purposes?
3. **DLQ tie-in** — visibility timeout + failed reprocessing is the natural lead-in to Dead-Letter Queues and `maxReceiveCount`. Should I stub that connection here, or keep this note tightly scoped to delay queues as the lesson was?

---

## Source

Adrian Cantrill SAA-C03 — _SQS Delay Queues_ lesson (transcript + course diagram). Figures fact-checked against current AWS documentation, 2026.

## References

- [Amazon SQS message timers — SQS Developer Guide](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-message-timers.html)
- [Amazon SQS delay queues — SQS Developer Guide](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-delay-queues.html)
- [Amazon SQS visibility timeout — SQS Developer Guide](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html)
- [ChangeMessageVisibility — SQS API Reference](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/APIReference/API_ChangeMessageVisibility.html)