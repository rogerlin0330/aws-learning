---

tags:

- aws
- saa
- serverless
- application-integration

---

# Step Functions

## Overview

**Step Functions** is a serverless orchestration service that lets you build **state machines** — long-running workflows with a start, an end, and **states** in between. It exists to address design decisions in [[Lambda]] that make Lambda a poor fit for long-running, stateful, multi-step processes:

- **15-minute execution limit** — a Lambda function cannot run past 15 minutes.
- **Stateless runtime environments** — each Lambda invocation environment is isolated and cleaned; state must be explicitly transferred between invocations. You can't naturally hold state across functions or invocations.

You _can_ chain Lambda functions (one invokes the next) to stretch past 15 minutes, but this gets messy at scale and isn't what Lambda is designed for. Step Functions is the right tool for workflows that are long-running, have decision points, may involve human intervention, and need to persist state (data) across steps.

A **state machine** is a workflow. A **state** is a thing that occurs inside the workflow — states can _do_ things, _decide_ things, and they all take in data, modify it, and output it.

Maximum execution duration for a Standard workflow is **1 year**.

## Diagram Walkthrough

![[step-functions-architecture.png]]

This diagram illustrates the **Pet Cuddle-O-Tron** demo scenario (Whiskers the cat reminds his human Bob when a cuddle is needed). Read it in three passes: the **serverless front-end** (steps 1–4, left/top), the **state machine internals** (steps 5–7, the dotted box on the right), and the **notification back-end** (steps 8–10, returning to Bob).

**Front-end / initiation (steps 1–4):**

1. Bob's laptop downloads the client-side website (HTML + JavaScript) from an **S3** static-website bucket.
2. Bob interacts with the page and clicks a button; the JavaScript calls a managed API hosted by **API Gateway**.
3. The button triggers the backing **API_Lambda** function.
4. **API_Lambda** starts the **state machine execution**, passing through all the input Bob entered (seconds until cuddle, message, email/phone).

**State machine internals — the dotted "Pet Cuddle-O-Tron" box (steps 5–7):** 5. **Timer** (a **Wait** state) waits for the configured period of time. 6. **Choice** state evaluates conditions on the input and selects one of three branches. 7. The three branches are **Task** states:

- **7a EmailOnly** → invokes only `email_reminder_lambda`
- **7b EmailAndSMS** → invokes _both_ Lambda functions
- **7c SMSOnly** → invokes only `sms_reminder_lambda`

**Back-end / notification (steps 8–10):** 8. `email_reminder_lambda` uses **SES (Simple Email Service)** to send Bob an email reminder. 9. The state machine triggers `sms_reminder_lambda` to send SMS. 10. `sms_reminder_lambda` uses **SNS (Simple Notification Service)** to send Bob an SMS reminder.

**Reading the arrow colours** (📚 not called out in the transcript, but the diagram encodes routing this way):

- The **green** path runs from `email_reminder_lambda` down to **EmailOnly (7a)** and across to SES — the email-only route.
- The **blue** path runs from `sms_reminder_lambda` up to **SMSOnly (7c)** and out to SNS — the SMS-only route.
- The **pink/magenta** lines are the primary request flow (Bob → API Gateway → API_Lambda → state machine) and the EmailAndSMS branch touching both Lambdas.

The **stateless Lambda functions live _outside_ the dotted state-machine box** — the state machine _coordinates_ them (invokes them as Task integrations); it does not perform the email/SMS work itself.

## Mechanics

### Workflow types — choose at creation, cannot be changed later ⚠️

You must pick **Standard** or **Express** when you create the state machine. This choice influences features, speed, and maximum duration, and <cite index="3-1">the type of state machine cannot be changed after it has been created.</cite>

|                 | **Standard** (default)                     | **Express**                                                                                      |
| --------------- | ------------------------------------------ | ------------------------------------------------------------------------------------------------ |
| Max duration    | **1 year**                                 | **5 minutes**                                                                                    |
| Execution model | **Exactly-once**                           | **At-least-once** (async) / at-most-once (sync) 📚                                                |
| Ideal for       | Long-running, durable, auditable workflows | High-volume event processing: IoT, streaming data processing/transformation, mobile app backends |
| Billing         | Per state transition 📚                     | Per execution + duration + memory consumed 📚                                                     |

🎯 **Exam triggers:** "long-running" / "up to one year" / "auditable" → **Standard**. "high-volume" / "IoT" / "streaming" / "5 minutes" → **Express**.

For the exam, Cantrill says you only need the high-level split: Standard is the default with a 1-year limit; Express is for high-throughput short workloads up to 5 minutes. 📚 The exactly-once vs. at-least-once distinction and billing model above are current AWS depth beyond the transcript — worth knowing but likely deeper than SAA needs. Source: [AWS — Choosing workflow type](https://docs.aws.amazon.com/step-functions/latest/dg/choosing-workflow-type.html).

### Starting a state machine

State machines can be started many ways — examples: **API Gateway**, **IoT rules**, **EventBridge** (for event-driven architectures), **Lambda**, or **manually**. Generally they're used for backend processing, so something in your application initiates the execution. 📚 AWS documents 140+ EventBridge event sources as possible triggers ([source](https://docs.aws.amazon.com/step-functions/latest/dg/concepts-standard-vs-express.html)).

### Amazon States Language (ASL)

State machines are defined/exported using a template format called **Amazon States Language (ASL)**, which is **JSON-based**. You can use ASL to create and export a state machine once configured to your liking.

### Permissions — IAM roles

Like other AWS services, a state machine interacts with other AWS services via an **IAM role**. The state machine **assumes the role while running** and gets temporary credentials to interact with whatever services it needs. See [[IAM Roles]].

## States

States fall into two groups: **flow-control states** and **work-performing (Task) states**.

**Flow-control states:**

- **Succeed** / **Fail** — terminal states. If the workflow reaches one, the execution succeeds or fails accordingly.
- **Wait** — pauses the workflow for a **period of time** _or_ **until a specific date/time**. Provided this info as input. (This is the "Timer" in the diagram.)
- **Choice** — takes a different path depending on an input; branching logic. (The diagram's Choice state, feeding EmailOnly / EmailAndSMS / SMSOnly.)
- **Parallel** — creates **parallel branches** that run concurrently. A single Choice branch might lead into a Parallel state to do several things at once.
- **Map** — accepts a **list** of items and performs an action (or set of actions) **for each item** in the list. Example: 10 items on an order → the Map state runs its actions 10 times, once per item.

**Work-performing state:**

- **Task** — a **single unit of work** performed by the state machine; this is how the state machine actually _does_ things. Task states integrate with many services: **Lambda, AWS Batch, DynamoDB, ECS, SNS, SQS, Glue, SageMaker, EMR**, and more (non-exhaustive). The state machine **coordinates** the work; it does not perform the work itself.

⚠️ **Discrepancy — the transcript omits the `Pass` state.** Current ASL defines **eight** state types: <cite index="12-1">Task, Choice, Parallel, Map, Pass, Wait, Succeed, Fail.</cite> The `Pass` state passes its input to its output (or injects fixed data) without performing work — useful for constructing and debugging state machines. Cantrill's lesson lists Succeed, Fail, Wait, Choice, Parallel, Map, and Task but does **not** mention Pass. Source: [AWS — State machine structure](https://docs.aws.amazon.com/step-functions/latest/dg/statemachine-structure.html).

## Exam Traps

- 🎯 **"Lambda can't run longer than 15 minutes / needs to maintain state across steps"** → the scenario wants **Step Functions**, not chained Lambdas.
- 🎯 **"Long-running workflow up to a year"** → **Standard** workflow.
- 🎯 **"High-volume / IoT / streaming, under 5 minutes"** → **Express** workflow.
- The workflow **type cannot be changed after creation** — a distractor answer suggesting you can "switch a Standard state machine to Express" is wrong; you'd recreate it.
- Step Functions **coordinates** services; the actual compute/work is done by the integrated service (e.g. Lambda) via a **Task** state — don't attribute the work to Step Functions itself.
- **ASL is JSON-based** (not YAML).
- Permissions come from an **IAM role** the state machine assumes — same pattern as other AWS services.

## Quick-Reference Table

| Item                        | Value                                                                    |
| --------------------------- | ------------------------------------------------------------------------ |
| Base entity                 | State machine (a workflow)                                               |
| Definition language         | Amazon States Language (ASL), JSON-based                                 |
| Workflow types              | Standard (default), Express                                              |
| Standard max duration       | 1 year                                                                   |
| Express max duration        | 5 minutes                                                                |
| State types (current, 8)    | Task, Choice, Parallel, Map, Pass, Wait, Succeed, Fail                   |
| State types (transcript, 7) | Task, Choice, Parallel, Map, Wait, Succeed, Fail (omits Pass) ⚠️          |
| Work performed by           | Task states (integrate with Lambda, ECS, DynamoDB, SNS, SQS, Glue, etc.) |
| Permissions                 | IAM role assumed at runtime                                              |
| Example trigger sources     | API Gateway, IoT rules, EventBridge, Lambda, manual                      |

## Open Questions (for Roger)

- **Cross-links:** I've stubbed wikilinks to [[Lambda]], [[IAM Roles]]. Do you have (or want) notes for **API Gateway**, **SES**, **SNS**, **EventBridge**, and **S3 static website hosting** to link from the diagram walkthrough? Several of these appear in the Pet Cuddle-O-Tron architecture and could anchor future notes.
- **Pass state (⚠️):** I flagged the transcript's omission rather than silently adding it as a peer of the other states. Do you want the `Pass` state promoted into the main **States** list (it's exam-relevant as one of the eight ASL types), or kept as the discrepancy callout it is now?
- **Editorial additions (📚 flagged for your review):** I added a billing-model column and the exactly-once/at-least-once row to the workflow-type table, plus the arrow-colour reading of the diagram. These go beyond the transcript — keep, trim to exam depth, or cut?
- **Depth scope:** Cantrill explicitly says "for the exam, high-level only" on Standard vs. Express. Want me to collapse the deeper current-AWS detail into a single "beyond exam" callout so the note stays exam-focused?