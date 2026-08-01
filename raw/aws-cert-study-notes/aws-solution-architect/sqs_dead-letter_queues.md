---

tags:

- aws
- saa
- sqs
- dead-letter-queue
- messaging
- integration

---

# SQS Dead-Letter Queues

## Overview

A **dead-letter queue (DLQ)** is a mechanism for handling _reoccurring failures_ when processing messages in an SQS queue. It solves a specific problem: a single "poison" message that keeps failing processing can otherwise cycle through a queue **forever**, wasting compute and blocking clean operation.

The core loop it fixes:

1. A message is received → it becomes **invisible** for the duration of the **visibility timeout**.
2. If it's **not** successfully processed and explicitly deleted before the timeout expires, it **reappears** in the queue.
3. It gets received again… fails again… reappears again… potentially **indefinitely**.

A DLQ breaks this loop by counting how many times a message has been received and, past a defined threshold, **moving** the problematic message out of the main ("source") queue into a separate queue for isolated handling.

📚 _Conceptually, a DLQ isn't a special queue type — it's just an ordinary SQS queue that another queue has been configured to dump failed messages into via a redrive policy._

---

## Diagram Walkthrough

![[sqs-dead-letter-queues.png]]

Read the diagram top-to-bottom: the **redrive policy** (top-left) governs the whole flow, the **source queue** is the large dashed box in the middle, and the **dead-letter queue** is the dashed box at the bottom (marked with a 🪦 tombstone — the "RIP" theme for messages that couldn't be processed).

**Top — the redrive policy (the rules):**

- The note icon (top-left) defines the **redrive policy**, which specifies three things: the **source queue**, the **dead-letter queue**, and the **conditions** under which messages move from one to the other.
- The key variable it defines is **`maxReceiveCount`**.
- The red arrow drops down from the policy into the source queue — the policy _governs_ that queue.
- Top-right states the trigger condition: **when `ReceiveCount` > `maxReceiveCount` AND the message isn't deleted → it's moved to the dead-letter queue.**

**Middle — the source queue (the failure loop):**

- The red horizontal arrow is the timeline of a single message's life in the queue, read left → right.
- The timeline alternates between two states:
    - **Message icon (envelope/parcel)** = the message is **visible** and available to be received.
    - **Grey block with an eye-slash icon** labeled "**Visibility Timeout (In Seconds)**" = the message has been received and is now **invisible**, hidden for the visibility-timeout duration.
- The **blue arrows** rising from the bottom are each labeled "**ReceiveMessage where processing fails**" — every one represents a consumer picking up the message, failing to process it, and _not_ deleting it. When each visibility timeout expires, the message pops back into view (the next message icon).
- Below the timeline, **`ReceiveCount`** increments on each receive: **1 → 2 → 3 → 4 → 5 …** This attribute is bumped _every time_ the message is received.

**Right + Bottom — the move to the DLQ:**

- Once `ReceiveCount` exceeds `maxReceiveCount`, the pink arrow on the far right curls **down** out of the source queue and into the dead-letter queue box.
- Inside the DLQ box: **"Enqueue timestamp of message is unchanged"** and **"Retention period of a dead-letter queue is generally longer than the source queue(s)"** — the two "gotchas" covered below.

---

## Mechanics / Deep-Dive

### `ReceiveCount` and `maxReceiveCount`

- Every message carries a **`ReceiveCount`** attribute, incremented each time the message is received by a consumer.
- The redrive policy defines **`maxReceiveCount`**.
- **Rule:** when a message's `ReceiveCount` **exceeds** `maxReceiveCount` _and_ the message hasn't been explicitly deleted, SQS moves it to the DLQ. <cite index="6-1">When the ReceiveCount for a message exceeds the maxReceiveCount for a queue, Amazon SQS moves the message to the dead-letter-queue.</cite>
- 📚 The **default `maxReceiveCount` is 10** if you configure a redrive policy without specifying one. <cite index="6-1">maxReceiveCount – The number of times a message is delivered to the source queue before being moved to the dead-letter queue. Default: 10.</cite>

### Why a DLQ is useful

Setting up a DLQ unlocks several operational benefits:

- **Alarming** — configure a CloudWatch alarm that fires when _any_ message lands in the DLQ, so you're automatically notified of problematic messages.
- **Isolated diagnostics** — a separate, quarantined area where you can examine logs and figure out _why_ a specific message kept failing.
- **Content analysis** — inspect the actual message payloads that failed to diagnose the root cause.
- **Separate/isolated processing** — apply _different_ processing logic to problematic messages (e.g., a special consumer that handles edge cases the main pipeline can't).

### ⚠️ Retention period + enqueue timestamp (the critical real-world gotcha)

All SQS queues have a **message retention period** — if a message ages past it without being processed, the message is **dropped**.

- When a message is first sent to a queue, it's stamped with an **enqueue timestamp** (the time it entered the _original_ queue).
- **When a message is moved from a source queue to a DLQ, the enqueue timestamp is NOT reset** — it keeps the original timestamp from the source queue.
- **Consequence:** the DLQ retention clock is measured from the _original_ enqueue time, not the moment it arrived in the DLQ.

**Worked example (from the lesson):** a message sits in the source queue for **1 day**, then moves to a DLQ whose retention period is **2 days**. It will only survive **1 more day** in the DLQ (2-day retention − 1 day already elapsed), _not_ a fresh 2 days.

- **Design rule of thumb:** a DLQ's retention period should generally be **longer** than the source queue's, precisely because the enqueue timestamp isn't refreshed on the move — otherwise messages can expire out of the DLQ before you get a chance to inspect them.

### One DLQ, many sources

- A **single DLQ can serve multiple source queues** — you don't need a dedicated DLQ per queue.

---

## Course-vs-Current-AWS Discrepancies

The lesson's core model (redrive policy, `maxReceiveCount`, `ReceiveCount`, enqueue-timestamp behavior) is **accurate and current**. A few things the course doesn't mention have been added by AWS since — worth knowing they exist, though the exam leans heavily on the fundamentals above.

> ⚠️ **DLQ must match the source queue type.** A dead-letter queue must be the **same type (Standard or FIFO)** as its source queue, and must live in the **same AWS account and Region**. <cite index="9-1">A dead letter queue must be the same type of queue (FIFO or standard) as the source queue. A dead letter queue must be created using the same AWS account and region as the source queue.</cite>

> ⚠️ **Redrive allow policy** (governs _which_ source queues may target a DLQ) — not covered in the lesson. It has three modes: **`allowAll`** (default — any queue in the same account/Region), **`byQueue`** (only listed queues; **up to 10** source queue ARNs), and **`denyAll`** (no source queues). <cite index="5-1">The redrive allow policy specifies which source queues can access the dead-letter queue. You can choose whether to allow all source queues, allow specific source queues, or deny all source queues use of the dead-letter queue. The default allows all source queues to use the dead-letter queue.</cite> Don't confuse the **redrive policy** (on the _source_ queue: points at the DLQ + sets `maxReceiveCount`) with the **redrive allow policy** (on the _DLQ_: controls who's allowed to use it).

> ⚠️ **DLQ redrive to source** — since this feature launched (AWS Compute Blog, "Introducing Amazon SQS dead-letter queue redrive to source queues"), you can **move messages back out of a DLQ** to the source queue (or another same-type queue) for reprocessing once you've fixed the underlying issue. <cite index="12-1">By default, dead-letter queue redrive moves messages from a dead-letter queue to a source queue. However, you can also configure any other queue as the redrive destination if both queues are the same type.</cite> A redrive task can run up to 36 hours, with a max of 100 active tasks per account, and you can tune the redrive velocity. _(Standard queues only for the redrive-back operation.)_

---

## Exam Traps

|🎯 Trap / trigger keyword|What to remember|
|---|---|
|"Message keeps failing / infinite reprocessing / poison message"|→ **Dead-Letter Queue** + **redrive policy**.|
|"`ReceiveCount` > `maxReceiveCount`"|Message is **moved to the DLQ** (if not already deleted).|
|"Notify me when messages fail"|Put a **CloudWatch alarm** on the DLQ.|
|"Message expired / disappeared from the DLQ sooner than expected"|**Enqueue timestamp is NOT reset** on the move — DLQ retention counts from the _original_ enqueue time.|
|"How to keep failed messages available for inspection?"|Set the **DLQ retention period longer** than the source queue's.|
|"Multiple queues, failed-message handling"|**One DLQ can serve many source queues.**|
|"DLQ for a FIFO queue"|DLQ must **also be FIFO** (same type / account / Region).|
|"Reprocess messages after fixing the bug"|**DLQ redrive** moves messages back to the source queue.|
|"Which queues can use this DLQ?"|That's the **redrive allow policy** (`allowAll` / `byQueue` / `denyAll`), not the redrive policy.|

---

## Quick-Reference

|Concept|Detail|
|---|---|
|**Purpose of DLQ**|Isolate messages that repeatedly fail processing; break the infinite-reprocessing loop.|
|**DLQ itself**|Just a normal SQS queue, designated as a failure target by another queue's redrive policy.|
|**Redrive policy**|Set on the **source** queue. Defines source queue, target DLQ, and `maxReceiveCount`.|
|**`ReceiveCount`**|Per-message attribute, +1 on each receive.|
|**`maxReceiveCount`**|Threshold; exceeding it (and not deleting) → move to DLQ. **Default: 10.**|
|**Enqueue timestamp**|**Unchanged** when moved to DLQ (original source-queue time).|
|**Retention**|DLQ retention should be **longer** than source (because timestamp isn't reset). Max retention 14 days.|
|**Fan-in**|One DLQ can back **multiple** source queues.|
|**Queue type match** 📚|DLQ must be **same type** (Standard/FIFO), same account, same Region.|
|**Redrive allow policy** ⚠️|On the **DLQ**; controls which sources may use it (`allowAll` default / `byQueue` ≤10 / `denyAll`).|
|**DLQ redrive** ⚠️|Move messages **back** from DLQ → source for reprocessing.|

---

## Open Questions for Roger

1. **Cross-linking** — which of these already exist in the vault to wikilink? `[[Amazon SQS]]` (parent/standard vs FIFO overview), `[[SQS Visibility Timeout]]`, `[[CloudWatch Alarms]]`. Want me to add the links (and flag any that need creating as stubs)?
2. **Depth on redrive-back / redrive allow policy** — I've kept these as ⚠️ callouts since they're post-course and lightly examined. Happy to spin either out into its own sub-note, or trim them here if you'd rather keep the note tight to what Cantrill covered.
3. **Visibility timeout** — the DLQ mechanic leans entirely on visibility-timeout behavior. If there isn't a dedicated `[[SQS Visibility Timeout]]` note yet, this is a natural candidate to break out.

---

_Source: Adrian Cantrill SAA-C03 — "SQS Dead-Letter Queues." Fact-checks against AWS SQS Developer Guide (Using dead-letter queues; DLQ redrive), SetQueueAttributes API Reference, and AWS Compute Blog (DLQ redrive to source queues), current as of 2026._
