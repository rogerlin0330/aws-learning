---
tags:
- aws
- saa
- sqs
- messaging
---

# SQS Standard vs FIFO Queues

> Companion to [[Simple Queue Service (SQS)]] — that note covers SQS mechanics broadly; this one is the queue-type comparison.

## The core analogy

The lesson frames the whole comparison around highways:

- **FIFO queue = single-lane highway.** No overtaking, so cars (messages) arrive in exactly the order they entered. Throughput is capped by the width of the road.
- **Standard queue = multi-lane highway.** Many messages travel in parallel, so throughput scales almost without limit — but cars can pass each other, so arrival order isn't guaranteed.

**What you're trading is performance for preserved order.** That single sentence is the lesson's thesis.

---

## Diagram walkthrough

![[sqs-standard-vs-fifo.png]]

**How to read it:** the slide is split into two horizontal bands, each drawn as a road. **Pink/magenta = standard** (top), **orange = FIFO** (bottom). The **flask icons** mark behavioural properties (the guarantees), while **green arrows** point to typical use cases. Message numbers on the box icons show delivery order.

**Top band — Multi Lane Highway (standard):**

- The road has **two lanes** of messages moving right. Note the numbering: the top lane carries **5** then **3**, the bottom carries **4**, **2**, **1** — deliberately scrambled to show messages overtaking each other.
- The **vertical double-headed arrow** on the left is capacity: _scalable, as wide as required, near unlimited TPS_. The road can gain lanes indefinitely.
- Flask properties: **best-effort ordering** (no rigid preservation of message order) and **at-least-once delivery** (there could on occasion be more than one copy of a message).
- Green arrow → use cases: **decoupling, worker pools, batch for future processing**.

**Bottom band — Single Lane Highway (FIFO):**

- One lane, messages in strict descending order **5, 4, 3, 2, 1** — no overtaking possible.
- The **vertical double-headed arrow** is much shorter than the standard one, encoding the capped throughput: **300 TPS without batching, 3,000 with**.
- Flask properties: **exactly-once processing** (duplicates are removed) and **message order is strictly preserved — first in, first out**.
- Also labelled: **must have `.fifo` suffix**.
- Green arrow → use cases: **workflow ordering, command ordering, price adjustments**.

📚 The relative lengths of the two vertical capacity arrows are doing real work in this diagram — the standard arrow spans the full band height, the FIFO one is compressed. That's the performance trade-off rendered spatially.

---

## Throughput mechanics

The lesson is precise about what the numbers mean, and it's a common point of confusion:

- FIFO handles **300 transactions per second to the SQS API**.
- **Each transaction is one message** — hence 300 messages/sec unbatched.
- **With batching, each transaction can contain 10 messages** — hence 3,000 messages/sec.

So the 10× is not a separate limit; it falls directly out of batching 10 messages into one API transaction.

Standard queues _"don't suffer from any real performance issues and can scale to a near infinite number of transactions per second."_

### ⚠️ Course-vs-current-AWS: high-throughput mode

The lesson notes that FIFO high-throughput mode exists but is **"only available in preview"** at time of recording.

That is now badly out of date:

- High-throughput mode for FIFO went **generally available in May 2021**, initially 3,000 TPS per API action. ([AWS What's New](https://aws.amazon.com/about-aws/whats-new/2021/05/amazon-sqs-now-supports-a-high-throughput-mode-for-fifo-queues))
- Current AWS message quotas list high-throughput FIFO as **region-dependent**: up to **70,000 TPS per API action** without batching, and up to **700,000 messages/sec** with batching in the highest-throughput Regions. Other Regions have lower high-throughput quotas. ([AWS SQS message quotas](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/quotas-messages.html))

🎯 **The 300 / 3,000 figures are still correct as the _default_ FIFO baseline**, and remain the expected exam answer. But "FIFO is slow" is no longer true in absolute terms — it's throughput-_limited relative to standard_, which is the framing that survives.

---

## The two delivery guarantees, and what they actually mean

The lesson states the guarantees but not the mechanisms behind them. Filling that in:

### FIFO guarantees — two mechanisms at two points

FIFO's "exactly-once" and "strict ordering" claims come from two related but distinct mechanisms:

**Ingest side — `MessageDeduplicationId` protects exactly-once into the queue.** A producer that hits a network or timeout error retries, which is the normal way a duplicate is born. Dedup IDs are tracked in a **5-minute window**; a repeat send is **accepted and acknowledged as successful, but the payload is discarded** rather than written to the queue. The success ack matters — the producer stops retrying, so the duplicate dies quietly. Supply the ID explicitly, or enable **content-based deduplication** (SHA-256 of the message _body_, not attributes). Provide neither and the send **fails with an error**. ([AWS SQS exactly-once processing](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/FIFO-queues-exactly-once-processing.html), [AWS `SendMessage` API reference](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/APIReference/API_SendMessage.html))

**Consuming side — `MessageGroupId` protects ordering and group advancement.** Dedup prevents a second _copy_ existing; it doesn't decide which later messages can be delivered while one is in flight. `MessageGroupId` is the ordering unit: messages in the same group are processed one at a time, and the group's continuation does not advance until the current in-flight messages are deleted or become visible again. AWS's definition:

> In FIFO queues, `MessageGroupId` is an attribute that organizes messages into distinct groups. Messages within the same message group are always processed one at a time, in strict order, ensuring that no two messages from the same group are processed simultaneously. In standard queues, using `MessageGroupId` enables fair queues. If strict ordering is required, use a FIFO queue. — [Using the message group ID](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/using-messagegroupid-property.html)

📚 Receive-side mechanics that fill this in (from [FIFO queue delivery logic](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/FIFO-queues-understanding-logic.html)) — these elaborate the definition rather than qualify it, since a batch is delivered to **one** consumer and still processed in order:

- **A single receive can return up to 10 messages from the same group** (`MaxNumberOfMessages`). SQS _attempts to return as many messages as possible with the same group ID in one call_, and they retain FIFO order within the batch.
- **Further delivery from that group is withheld** until the received messages are **deleted** _or_ **become visible again** (e.g. visibility timeout expires). Both exit conditions matter — a stuck consumer unblocks its group by timeout, not only by deletion.
- **Batches can span groups.** If fewer than 10 messages are available for one group, SQS may include messages from other groups in the same batch; each group retains its own FIFO order.
- **You cannot request a specific group.** There is no API parameter to ask for messages from a chosen `MessageGroupId`.
- **Parallelism comes from group count.** Different group IDs process concurrently; one group ID for the whole queue means strictly sequential processing.

📚 **Retry semantics** (same page): a producer may safely retry `SendMessage` with the same deduplication ID — as long as it receives **at least one acknowledgment before the deduplication interval expires**, retries introduce no duplicates and don't disrupt order. A consumer may retry `ReceiveMessage` using the same **receive request attempt ID** without disrupting order, provided it gets an acknowledgment before the visibility timeout expires.

### 🎯 Two layers of exclusivity — only one is FIFO-specific

Easy to credit FIFO with something both queue types do. Separate them:

| Layer                  | What it protects                                                                                                              | Applies to                 |
| ---------------------- | ----------------------------------------------------------------------------------------------------------------------------- | -------------------------- |
| **Visibility timeout** | The messages _you received_ — hidden from every other consumer while in flight                                                | **Both** standard and FIFO |
| **Message group hold** | The _group's continuation_ — no further messages from that group go to anyone until yours are deleted or become visible again | **FIFO only**              |

Receiving a batch of 10 from a **standard** queue already gives you exclusive possession of those 10 — that's the visibility timeout, not a FIFO feature. What FIFO adds is the second layer: the group stops advancing for everyone.

**This is precisely why standard can't preserve order.** With no group hold, the next messages flow straight to whichever consumer polls next, in parallel with your in-flight batch. Ordering can't survive that, regardless of how the messages were stored.

📚 Caveat on layer 1 for standard queues: "no one else handles it" is at-least-once-qualified, because of the unavailable-replica case below — a surviving copy can reach another consumer later. On FIFO that doesn't occur.

### Standard at-least-once — the absence of both

Standard queues have **neither mechanism**, which is why at-least-once is what remains:

- No dedup ID → a producer retry lands as a genuinely separate message.
- No group coordination → nothing serialises consumption, so nothing preserves order either.
- Additionally, SQS stores messages **redundantly across multiple servers**. AWS documents that if one of those servers is unavailable when the delete request is sent, the copy survives on that server and may be returned on a later receive. ([AWS SQS at-least-once delivery](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/standard-queues-at-least-once-delivery.html), [AWS `DeleteMessage` API reference](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/APIReference/API_DeleteMessage.html))

📚 That third point is documented as a **standard-queue implementation detail**. AWS describes FIFO at the level of _guarantees_ rather than replica behaviour, so don't try to build a like-for-like comparison from it.

### 🎯 What "exactly-once" does _not_ cover

The diagram's phrasing — _"exactly-once processing, duplicates are removed"_ — is easy to over-read. The guarantee is about **what reaches the queue**, not what your application executes.

If a consumer **overruns its visibility timeout**, or **commits a database write and then crashes before calling `DeleteMessage`**, the message becomes visible again and is **delivered a second time — on FIFO exactly as on standard**. No duplicate was ever introduced _into the queue_, so the guarantee is not violated.

**Therefore: consumers must be idempotent regardless of queue type.** The usual pattern is tracking processed message IDs in a low-latency store such as DynamoDB.

| Cause of duplicate                          | Standard           | FIFO                                   |
| ------------------------------------------- | ------------------ | -------------------------------------- |
| Producer resends after no ack               | Duplicate persists | **Prevented** — dedup ID, 5-min window |
| Consumer crash / visibility timeout overrun | **Redelivered**    | **Redelivered too**                    |

---

## ⚠️ Course-vs-current-AWS: `MessageGroupId` on standard queues

The course presents `MessageGroupId` as purely a FIFO concept. It is now **accepted on standard queues too**, where it does something entirely different — it enables **fair queues**, using the value as a **tenant identifier** to mitigate noisy neighbours.

AWS states the distinction explicitly: `MessageGroupId` on standard queues does not behave as it does on FIFO queues; it is used only as a tenant identifier and does not enforce message ordering.

How fair queues work: SQS monitors how in-flight messages are distributed across tenants. When one tenant is disproportionately represented, SQS prioritises returning _other_ tenants' messages, keeping their **dwell time** low while the noisy tenant's backlog drains. Fair queues preserve virtually unlimited throughput with no limit on tenant count — precisely because nothing is being locked. ([AWS fair queues docs](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-fair-queues.html))

🎯 **The trap:** seeing `MessageGroupId` on a standard queue and assuming ordering comes with it. It does not. If strict ordering is required, you need a FIFO queue.

---

## Use cases

|           | **Standard**                                                                                       | **FIFO**                                                                                                                       |
| --------- | -------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| Ideal for | Decoupling application components; worker pool architectures; batching items for future processing | Workflow-based order processing; command ordering; sequential/iterative price adjustment calculations in sales order workflows |
| Why       | Throughput and elasticity matter more than order                                                   | Order or non-duplication is functionally required                                                                              |

📚 The lesson's command-ordering example: a system administrator entering commands into a processing system where execution order must be maintained.

---

## 🎯 Exam Traps

| If the question mentions…                     | Think…                                                             |
| --------------------------------------------- | ------------------------------------------------------------------ |
| Order must be preserved / first-in-first-out  | **FIFO queue**                                                     |
| Maximum throughput, app tolerates duplicates  | **Standard queue**                                                 |
| Queue name ending requirement                 | **`.fifo` suffix** — flagged as appearing on the exam "many times" |
| Duplicates cannot be tolerated at ingest      | **FIFO** + `MessageDeduplicationId`                                |
| Consumer might process a message twice        | **Idempotency** — FIFO does not solve this                         |
| Dedup needed on a standard queue              | **Not supported** — build it yourself                              |
| `MessageGroupId` on a _standard_ queue        | **Fair queues** (tenant fairness), _not_ ordering                  |
| One stuck message blocks everything behind it | **Head-of-line blocking** within a message group                   |
| FIFO ordering _and_ parallelism required      | **Many distinct message group IDs**                                |

---

## Quick reference

|                             | **Standard**                 | **FIFO**                                                |
| --------------------------- | ---------------------------- | ------------------------------------------------------- |
| Analogy                     | Multi-lane highway           | Single-lane highway                                     |
| Ordering                    | Best-effort                  | Strictly preserved                                      |
| Delivery                    | At-least-once                | Exactly-once _into the queue_                           |
| Throughput                  | Near-unlimited TPS           | 300 TPS / 3,000 batched (default)                       |
| High-throughput mode        | n/a                          | Now GA; region-dependent, up to 70,000 TPS              |
| Name suffix                 | —                            | **`.fifo` required**                                    |
| `MessageDeduplicationId`    | Not supported                | 5-minute window                                         |
| `MessageGroupId`            | Fair queues (tenant ID only) | Ordering unit; processed one at a time, in strict order |
| Consumer idempotency needed | Yes                          | **Yes — still**                                         |

---

## Open Questions for Roger

1. **Overlap with the main note** — [[Simple Queue Service (SQS)]] already carries a Standard vs FIFO section that I expanded during our discussion. Do you want that section trimmed to a pointer at this note, or is some duplication fine for revision purposes?
2. **Fair queues** — worth a standalone sub-note? AWS has three diagrams (steady state / noisy neighbour / mitigation) plus dedicated CloudWatch metrics (`ApproximateNumberOfMessagesVisibleInQuietGroups`) that I did not pull in here.
3. **Cross-links to create/confirm:** `[[Simple Queue Service (SQS)]]`, `[[Simple Notification Service (SNS)]]`, `[[DynamoDB]]` (referenced for the idempotency pattern).
4. **Quota follow-up:** the FIFO in-flight quota conflict from the main note is settled by current AWS docs: FIFO queues support **120,000 in-flight messages**, same listed default as standard queues. ([AWS SQS FIFO queue quotas](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/quotas-fifo.html), [AWS SQS service quotas](https://docs.aws.amazon.com/general/latest/gr/sqs-service.html))

---

_Source: Adrian Cantrill SAA-C03 — "SQS Standard vs FIFO Queues" lesson. Mechanism detail and discrepancy flags added from current AWS documentation (2026), cited inline._
