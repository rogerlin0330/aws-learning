---
title: Step Functions
type: service
tags: [step-functions, serverless, orchestration, state-machine, application-integration, asl]
certs: [aws-saa]
updated: 2026-07-29
---

# Step Functions

## Overview

Step Functions is a **serverless orchestration service**: it lets you build **state machines** — long-running workflows with a start, an end, and **states** in between.

It exists because of two deliberate design decisions in [Lambda](lambda.md) that make Lambda a poor fit for long-running, stateful, multi-step processes:

- **15-minute execution limit** — a Lambda function [cannot run past 900 seconds](lambda.md#timeout).
- **Stateless runtime environments** — each Lambda [execution context](lambda.md#statelessness) is isolated and eventually torn down, so state must be explicitly handed between invocations. You cannot naturally hold state across functions or across invocations.

You *can* chain Lambda functions (each one invoking the next) to stretch past 15 minutes, but this gets messy at scale and is not what Lambda is designed for. Step Functions is the right tool when a workflow is long-running, has decision points, may involve human intervention, and needs to persist data across steps.

Vocabulary:

- A **state machine** is a workflow.
- A **state** is a thing that occurs inside that workflow. States can *do* things and *decide* things, and they all take in data, modify it, and output it.

Maximum execution duration for a **Standard** workflow is **1 year**.

Step Functions is one of AWS's application-integration and [decoupling](../patterns/architecture-evolution.md) primitives, alongside SQS (queues), [SNS](sns.md) (pub/sub), and [EventBridge](eventbridge.md) (event bus). Where those move *messages*, Step Functions coordinates *sequence and state*, which makes it the orchestration layer of a [serverless architecture](../patterns/serverless-architecture.md).

## Architecture

![Step Functions architecture diagram for the Pet Cuddle-O-Tron demo: on the left a user (Bob) with a laptop downloads a static website from an S3 bucket (step 1), interacts with it and clicks a button (step 2) that calls API Gateway, which triggers API_Lambda (step 3), which starts the state machine execution (step 4); a dotted pink box labelled Pet Cuddle-O-Tron encloses the state machine, containing a Timer wait state that waits for a period of time (step 5) feeding a Choice state that evaluates conditions (step 6) and branches to three orange task states, EmailOnly (7a), EmailAndSMS (7b), and SMSOnly (7c); the two Lambda functions email_reminder_lambda (top right) and sms_reminder_lambda (bottom right) sit outside the dotted box, with a green line joining EmailOnly to the email function and on to Simple Email Service which sends Bob an email reminder (step 8), and a blue line joining SMSOnly to the SMS function (step 9) and on to Simple Notification Service which sends Bob an SMS reminder (step 10); pink lines carry the main request flow and the EmailAndSMS branch, which touches both Lambda functions](../assets/step-functions-architecture.png)

This is the **Pet Cuddle-O-Tron** demo scenario: Whiskers the cat reminds his human Bob when a cuddle is needed. Read it in three passes — the **serverless front end** (steps 1 to 4, left and top), the **state machine internals** (steps 5 to 7, the dotted box on the right), and the **notification back end** (steps 8 to 10, returning to Bob).

Step by step:

1. **Download the website** — Bob's laptop downloads the client-side website (HTML plus JavaScript) from an [S3 static-website bucket](s3.md).
2. **Click a button** — Bob interacts with the page; the JavaScript calls a managed API hosted by **API Gateway**.
3. **Invoke API_Lambda** — the button triggers the backing **API_Lambda** function.
4. **Start the execution** — **API_Lambda** starts the **state machine execution**, passing through all the input Bob entered (seconds until the cuddle, message, email address or phone number).
5. **Timer** — a **Wait** state inside the state machine waits for the configured period of time.
6. **Choice** — a **Choice** state evaluates conditions on the input and selects one of three branches.
7. **Three Task branches** — **7a EmailOnly** invokes only `email_reminder_lambda`; **7b EmailAndSMS** invokes *both* Lambda functions; **7c SMSOnly** invokes only `sms_reminder_lambda`.
8. **Email delivery** — `email_reminder_lambda` uses **SES (Simple Email Service)** to send Bob an email reminder.
9. **SMS invocation** — the state machine triggers `sms_reminder_lambda` to send the SMS.
10. **SMS delivery** — `sms_reminder_lambda` uses **[SNS](sns.md)** to send Bob an SMS reminder.

**Reading the line colours** (the diagram encodes routing this way even though the lesson does not call it out):

- The **green** path runs from **EmailOnly (7a)** across to `email_reminder_lambda` and on to SES — the email-only route.
- The **blue** path runs from **SMSOnly (7c)** down to `sms_reminder_lambda` and out to SNS — the SMS-only route.
- The **pink** lines are the primary request flow (Bob, then API Gateway, then API_Lambda, then the state machine), the **EmailAndSMS** branch touching both Lambda functions, and the final delivery legs back to Bob.

> [!warning] Gotcha — the Lambda functions live *outside* the dotted box
> The stateless Lambda functions are drawn outside the state-machine boundary on purpose. The state machine **coordinates** them, invoking them as Task integrations; it does **not** perform the email or SMS work itself. Attributing the work to Step Functions rather than the integrated service is a classic exam misread.

## Mechanics

### Workflow types — chosen at creation and unchangeable

You must pick **Standard** or **Express** when you create the state machine. The choice influences features, speed, and maximum duration, and [the type of state machine cannot be changed after it has been created](https://docs.aws.amazon.com/step-functions/latest/dg/choosing-workflow-type.html).

| | **Standard** (default) | **Express** |
| --- | --- | --- |
| Max duration | **1 year** | **5 minutes** |
| Execution model | **Exactly-once** | **At-least-once** (async) or at-most-once (sync) |
| Ideal for | Long-running, durable, auditable workflows | High-volume event processing: IoT, streaming data processing and transformation, mobile app back ends |
| Billing | Per state transition | Per execution plus duration and memory consumed |

> [!tip] Exam Tip — workflow type triggers
> "Long-running" / "up to one year" / "auditable" means **Standard**. "High-volume" / "IoT" / "streaming" / "5 minutes" means **Express**.

> [!note] Depth beyond the exam
> Cantrill says you only need the high-level split for the exam: Standard is the default with a 1-year limit, Express is for high-throughput short workloads up to 5 minutes. The exactly-once versus at-least-once execution model and the two billing models are current AWS detail beyond the lesson — worth knowing, likely deeper than SAA needs. Source: [AWS — Choosing workflow type](https://docs.aws.amazon.com/step-functions/latest/dg/choosing-workflow-type.html).

### Starting a state machine

State machines can be started many ways — **API Gateway**, **IoT rules**, **[EventBridge](eventbridge.md)** (for event-driven architectures), **[Lambda](lambda.md)**, or **manually**. Step Functions is generally used for back-end processing, so something inside your application initiates the execution. Using EventBridge as the trigger widens this to the broad set of AWS services that emit events onto an event bus, which is why EventBridge lists Step Functions as one of its standard [rule targets](eventbridge.md).

### Amazon States Language (ASL)

State machines are defined and exported using a template format called **Amazon States Language (ASL)**, which is **JSON-based**. You can author a state machine in the console and then export the ASL once it is configured to your liking.

### Permissions — IAM roles

Like other AWS services, a state machine interacts with other AWS services via an **[IAM role](iam.md#iam-roles)**. The state machine **assumes the role while it runs** and gets temporary credentials for whatever services it needs — the same pattern as a [Lambda execution role](lambda.md#security) or an [EC2 instance role](ec2.md#instance-roles--instance-profiles).

## States

States fall into two groups: **flow-control states** and the **work-performing Task state**.

**Flow-control states:**

- **Succeed** and **Fail** — terminal states. If the workflow reaches one, the execution succeeds or fails accordingly.
- **Wait** — pauses the workflow for a **period of time** *or* **until a specific date and time**, supplied as input. This is the "Timer" in the diagram.
- **Choice** — takes a different path depending on an input; branching logic. This is the diagram's Choice state feeding EmailOnly, EmailAndSMS, and SMSOnly.
- **Parallel** — creates **parallel branches** that run concurrently. A single Choice branch might lead into a Parallel state to do several things at once.
- **Map** — accepts a **list** of items and performs an action (or set of actions) **for each item** in the list. Example: an order with 10 items means the Map state runs its actions 10 times, once per item.
- **Pass** — passes its input straight to its output, optionally injecting fixed data, without performing work. Useful when constructing and debugging state machines.

**Work-performing state:**

- **Task** — a **single unit of work** performed by the state machine; this is how the state machine actually *does* things. Task states integrate with many services: **Lambda, AWS Batch, DynamoDB, ECS, SNS, SQS, Glue, SageMaker, EMR**, and more. The state machine **coordinates** the work; it does not perform the work itself.

> [!note] Course-vs-reality — the lesson omits `Pass`
> Current ASL defines **eight** state types: [Task, Choice, Parallel, Map, Pass, Wait, Succeed, Fail](https://docs.aws.amazon.com/step-functions/latest/dg/statemachine-structure.html). Cantrill's lesson lists seven, leaving out **Pass**. It is included above because it is one of the eight documented types and is cheap to remember; the raw note flags the omission rather than quietly folding it in. Source: [AWS — State machine structure](https://docs.aws.amazon.com/step-functions/latest/dg/statemachine-structure.html).

## Exam Traps

> [!tip] Step Functions exam triggers
>
> - **"Lambda can't run longer than 15 minutes"** or **"needs to maintain state across steps"** means the scenario wants **Step Functions**, not chained Lambdas.
> - **"Long-running workflow up to a year"** means a **Standard** workflow.
> - **"High-volume / IoT / streaming, under 5 minutes"** means an **Express** workflow.
> - The workflow **type cannot be changed after creation** — a distractor offering to "switch a Standard state machine to Express" is wrong; you would recreate it.
> - Step Functions **coordinates** services; the actual compute is done by the integrated service (for example Lambda) through a **Task** state. Don't attribute the work to Step Functions itself.
> - **ASL is JSON-based**, not YAML.
> - Permissions come from an **IAM role the state machine assumes** — the same pattern as every other AWS service.

## Quick Reference

| Item | Value |
| --- | --- |
| Base entity | State machine (a workflow) |
| Definition language | Amazon States Language (ASL), JSON-based |
| Workflow types | Standard (default), Express — fixed at creation |
| Standard max duration | 1 year |
| Express max duration | 5 minutes |
| State types (current, 8) | Task, Choice, Parallel, Map, Pass, Wait, Succeed, Fail |
| State types (lesson, 7) | Task, Choice, Parallel, Map, Wait, Succeed, Fail — omits Pass |
| Work performed by | Task states (integrating Lambda, ECS, DynamoDB, SNS, SQS, Glue, and others) |
| Permissions | IAM role assumed at runtime |
| Example trigger sources | API Gateway, IoT rules, EventBridge, Lambda, manual |

## References

- [Lambda](lambda.md) — the 15-minute timeout and statelessness that Step Functions works around
- [Serverless Architecture](../patterns/serverless-architecture.md), [Architecture Evolution](../patterns/architecture-evolution.md) — where orchestration sits in the pattern
- [EventBridge](eventbridge.md), [SNS](sns.md) — the sibling application-integration services
- [AWS — Choosing workflow type](https://docs.aws.amazon.com/step-functions/latest/dg/choosing-workflow-type.html)
- [AWS — State machine structure](https://docs.aws.amazon.com/step-functions/latest/dg/statemachine-structure.html)
- Adrian Cantrill, AWS SAA course — Step Functions
- Source: `raw/aws-cert-study-notes/aws-solution-architect/step_functions.md`
