---

tags:

- aws
- saa
- compute
- serverless

---

# AWS Lambda

## Overview

Lambda is a **Function as a Service (FaaS)** product: you supply short-running, focused code, and Lambda runs it and bills you only for the duration it executes. A **Lambda function** is a piece of code that Lambda runs using a supported **runtime** (e.g. Python 3.8). Think of a Lambda function as a _unit of configuration_ — the code plus its wrappings: the runtime language, the deployment package, resource allocations, and security configuration.

When invoked, the deployment package is loaded into and executed within a **runtime environment** (also called an **execution context**) that is purpose-built for that runtime. You directly allocate memory; virtual CPU is allocated _indirectly_ in proportion to memory. Billing is based on allocated resource × execution duration per invocation.

Lambda is a core building block of **serverless and event-driven architectures** on AWS.

🎯 Exam trigger keywords: _serverless_, _event-driven_, _function as a service / FaaS_, _only pay for what you use_, _short-running focused compute_.

---

## Diagram Walkthrough — Architecture 101

![[lambda-architecture-101.png]]

**How to read it:** the center box is the **runtime environment**. The Python logo on its left feeds into it (the runtime), and the memory stick on its right represents allocated resource. **Function Code** sits inside, and the hard-hat icon underneath is the **execution role**. Everything flowing _into_ the box (deployment package from top-left, timeout, memory scaling from the right) are the configuration levers you set.

- **Top-left — Deployment package:** 50 MB zipped / 250 MB unzipped. 📚 (These limits are not stated in the transcript; see the Quick-Reference table for the current AWS figures and the discrepancy note.)
- **Top-center — 900s (15M) function timeout:** the hard maximum for a single invocation.
- **Left column — Common runtimes:** Python, Ruby, Java, Go, C# (transcript also names **Node.js**, which the diagram omits). Custom runtimes (e.g. **Rust**, shown as the crab) are possible via **layers**.
- **Right — memory → vCPU scaling:** 128 MB up to 10240 MB in 1 MB steps; **1769 MB = 1 vCPU**, scaling linearly.
- **Bottom-right — /tmp storage:** 512 MB default, up to 10240 MB.

---

## Mechanics

### Runtimes

- Supported runtimes include multiple versions of **Python, Node.js, Ruby, Java, Go, and C#**.
- **Custom runtimes** (e.g. Rust) can be added via **Lambda layers**, many community-built.
- 🎯 **Docker as an exam signal:** if you see the bare term **"Docker"**, read it as **NOT Lambda** — it points to traditional containerized compute (e.g. ECS). This is distinct from Lambda's _container image_ support: you can build **container images for Lambda** using your existing Docker build processes, but those images are purpose-built to run _inside the Lambda environment_, not as standalone Docker containers. Don't conflate the two.

### Resource allocation

- **Memory:** directly set, 128 MB → 10240 MB in 1 MB steps.
- **vCPU:** allocated indirectly; **1769 MB yields the equivalent of one vCPU**, scaling linearly (less memory = fractional vCPU; more memory = multiple vCPUs, up to ~6 vCPUs at 10240 MB 📚).
- **/tmp disk:** 512 MB mounted at `/tmp` by default, scalable to 10240 MB. Treat as **temporary scratch space** — assume it may be blank on any invocation.

### Timeout

- 🎯 Maximum **900 seconds / 15 minutes** per invocation — the **function timeout**. This is one of the few figures worth memorizing for the exam.
- For workloads exceeding 15 minutes, Lambda can't be used directly; use **Step Functions** to orchestrate longer workflows across multiple functions.

### Statelessness

- By default, assume **every invocation runs in a brand-new, clean runtime environment** with no leftover state. Your code must work 100% of the time on the assumption of a fresh environment.
- ⚠️ Execution contexts _can_ be reused (warm starts — see below), but you must never _depend_ on reuse.

### Common use cases

- **Serverless applications:** S3 + API Gateway + Lambda.
- **File processing:** S3 upload → S3 event → Lambda (classic example: watermarking images).
- **Database triggers:** DynamoDB + DynamoDB Streams + Lambda (fires on insert/modify/delete).
- **Serverless cron:** EventBridge / CloudWatch Events → Lambda on a schedule.
- **Real-time stream processing:** Kinesis stream → Lambda, scaling with data volume.

---

## Networking

Lambda has **two networking modes**. Know both for the exam.

### Public networking (default)

![[lambda-public-lambda-architecture.png]]

**Diagram Walkthrough:** the Lambda symbol sits inside the `aws` public-zone box. Green arrows (allowed paths) reach out to **public AWS services** (SQS, DynamoDB) and to the **public internet** (IMDb, for fetching cat-themed movie data). The red dotted arrow with the "no entry" icon points at the VPC bubble (Aurora, EC2, EFS) — this is the _blocked_ path: by default the function **cannot** reach VPC-based resources.

- By default, Lambda runs in the **public AWS network** and can reach **public-space AWS services** and the **public internet** (assuming valid auth).
- 🎯 It has **no access to VPC-based services** (Aurora, EC2, EFS) _unless_ those services are given public addressing and security rules allowing external access.
- **Best performance** — no customer-specific networking is required, so functions run on shared hardware/networking.

### VPC networking

![[lambda-private-lambda-architecture.png]]

**Diagram Walkthrough — left panel (behavior):** the Lambda symbol now sits _inside_ a private subnet. Orange arrows show it reaching VPC resources (EFS, EC2, Aurora) directly, and using a **VPCE** (VPC endpoint) to reach DynamoDB, or a **NATGW → IGW** path to reach the internet (IMDb). The red arrow with the block icon reminds you that without this networking scaffolding, external paths are closed. **Right panel ("NEW WAY"):** shows the modern shared-ENI architecture (see below) — three functions in the **AWS Lambda Service VPC** all fan into a single connection point into the customer VPC (**A4LVPC**), with **90s initial setup** and **no invocation delay**.

- 🎯 **Golden rule:** a Lambda function in a VPC **obeys all normal VPC networking rules** — treat it exactly like any other VPC resource.
- It can **freely reach other VPC resources** (subject to NACLs and security groups).
- It **cannot reach outside the VPC** (public AWS services, internet) unless the VPC provides the path: a **gateway endpoint** for DynamoDB, or a **NAT gateway + internet gateway** for public/internet access.
- 🎯 The function's **execution role needs EC2 network permissions**, because the Lambda service must create **network interfaces** in your VPC. A common exam scenario: VPC Lambda failing because it lacks these EC2 networking permissions.

#### Old vs. new ENI architecture

VPC Lambdas don't literally run in your VPC — the model resembles Fargate, with a **Lambda service VPC** connecting into your **customer VPC** via elastic network interfaces (ENIs).

- ⚠️ **Old way:** each function created **one ENI per function, per invocation**. This added invocation delay and scaled badly — high concurrency meant many ENIs and VPC capacity pressure. This made VPC Lambdas a performance liability.
- **New way:** AWS analyzes all functions in a region/account and builds **one ENI per unique combination of security groups + subnets**. Functions sharing the same SG+subnet combo share a single ENI. These ENIs are created **once**, when you configure/update the function (typically ~90 seconds), **not per invocation** — so there's **no per-invocation delay**, and private networking now scales without ENI sprawl.

> 📚 **Editorial note (flag for review):** the diagram labels the new architecture "A4LVPC" (Adrian's _Animals4Life_ demo VPC name) and "AWS Lambda Service VPC." These are illustrative container names, not exam terms — the exam-relevant concept is _shared ENIs per unique SG+subnet combination_.

---

## Security

### Two-part permissions model

![[lambda-security-architecture.png]]

**Diagram Walkthrough:** the dotted box is the function's permission boundary. Two red arrows come _in_ from the top (one from each `aws` logo) — these represent the **resource policy** controlling _who can invoke_. The blue arrows across the bottom go _out_ to DynamoDB (in) and S3 (out) — these represent what the **execution role** lets the function _do_. Read top-down: inbound = resource policy (invoke control); outbound = execution role (action permissions).

**1. Execution role (well-known, associate-level):**

- An **IAM role assumed by Lambda**, providing the temporary credentials the function uses to interact with other AWS services.
- Has a **trust policy** trusting Lambda and a **permissions policy** defining what the function can do (e.g. read DynamoDB, write S3).
- Conceptually **identical to an EC2 instance role**.

**2. Resource policy (less commonly known):**

- 🎯 Analogous to an **S3 bucket policy** — controls **who can invoke** a specific function (external accounts, or services like SNS / S3).
- Automatically modified when you integrate other services with Lambda.
- ⚠️ **Course-vs-reality discrepancy — resource policy editability:** the transcript states the resource policy can _only_ be changed via CLI/API and **cannot** be edited in the console UI. **This is now outdated.** Per current AWS docs, you can view and edit a function's resource-based policy in the console under **Configuration → Permissions → Resource-based policy**. ([AWS Lambda docs — Viewing resource-based IAM policies](https://docs.aws.amazon.com/lambda/latest/dg/access-control-resource-based.html); the console-view capability dates to a [2017 access-control update](https://aws.amazon.com/about-aws/whats-new/2017/03/aws-lambda-introduces-access-control-enhancements).) CLI/API editing remains fully valid; the "console-only-view / no-edit" framing does not.

### Logging & monitoring

- **CloudWatch Logs:** function output, log messages, errors, and execution duration details.
- **CloudWatch (metrics):** invocation counts, successes/failures, retries, latency.
- **X-Ray:** distributed tracing — trace a user/session path through a serverless app. 🎯 Remember the pairing _X-Ray = distributed tracing_.
- 🎯 **Exam trap:** for Lambda to write to CloudWatch Logs, the **execution role must grant the permissions** (there's a prebuilt basic-execution policy for this — `AWSLambdaBasicExecutionRole` 📚). A classic diagnostic scenario: "function isn't working, nothing in CloudWatch Logs" → likely a missing execution-role permission.

---

## Invocation (Part 3)

Three ways to invoke a Lambda function: **synchronous**, **asynchronous**, and **event source mapping**.

### Synchronous invocation

![[lambda-invocation-synchronous.png]]

**Diagram Walkthrough:** two request/response pairs. Top — **CLI/API** sends a red "invoke" arrow to Lambda and gets a green response arrow back. Bottom — **Clients** talk to **API Gateway** (red arrow), which proxies to Lambda (red), and responses flow back green through the gateway to the client. In both, the caller _waits_ the whole time.

- Caller (CLI/API, or clients via **API Gateway**) invokes the function and **waits** for the result during the request.
- The function runs once and returns **success or failure** synchronously.
- 🎯 **Errors and retries must be handled by the client** — Lambda does not retry.
- Typically used when a **human**, directly or indirectly, invokes the function.

### Asynchronous invocation

![[lambda-invocation-asynchronous.png]]

**Diagram Walkthrough:** the S3 bucket (left) fires a green "S3 Event .. PUT" arrow at the central Lambda symbol and then **stops tracking** — it doesn't wait. From Lambda, three outbound paths: blue up to **DynamoDB** (normal processing output), a red self-loop underneath (**retry logic, 0–2 times**), red up-right to the **DLQ**, and green down-right to a **Destination**. The right-side captions explain idempotency, retries, and destinations.

- 🎯 Typically used when **AWS services invoke Lambda** on your behalf (e.g. S3 events).
- The source (e.g. S3) generates the event and **stops tracking** — it doesn't wait for a response.
- **Lambda owns reprocessing** on failure: retries are **configurable between 0 and 2 times**.
- 🎯 **Idempotency requirement:** because Lambda re-runs the _same_ event with no extra context, the function must be **idempotent** — reprocessing should yield the same end state. (Transcript analogy: _set_ the balance to $20, don't _add_ $10. A "set" is safe to re-run; an "add" is not.) Design functions to make a desired state _true_ rather than performing additive/subtractive operations.
- **Dead-letter queue (DLQ):** events that still fail after retries can be routed to a DLQ for diagnostics.
- **Destinations (newer feature):** processed events can be delivered to **SQS, SNS, another Lambda, or EventBridge**, with **separate destinations for success vs. failure**.

### Event source mapping (ESM)

![[lambda-invocation-event-source-mapping.png]]

**Diagram Walkthrough:** top-left, **producer vans (telemetry)** push data via red arrows into a **Kinesis Data Stream** (top-center). A purple double-headed arrow connects the stream down to the **Event Source Mapping** (the Lambda-badge box, bottom-center) — this is the **polling** relationship, pulling **source batches**. From the ESM, an orange double-headed arrow sends **event batches** right into the actual **Lambda function** (bottom-right). A red dotted arrow loops from the Lambda function's **execution role** back up to the stream — signaling that **the ESM borrows the execution role's permissions** to read the source. Bottom-left, a purple arrow sends failed batches to a **DLQ** (SQS/SNS).

- 🎯 Used for **streams/queues that don't natively generate events** and require **polling**: **Kinesis, DynamoDB Streams, SQS**, and **Amazon MSK** (Managed Streaming for Apache Kafka).
- A hidden **event source mapping** component polls the source, retrieves **source batches**, splits them by configured **batch size**, and delivers **event batches** to the function.
- A single invocation may receive **hundreds of events** in a batch — 🎯 tune batch size carefully so processing completes within the **15-minute timeout**.
- 🎯 **Permissions exam trap — the key ESM fact:** the source does **not** deliver the event; the ESM **reads** from the source using the **function's execution role permissions**. So even though the function code never directly calls Kinesis, **the execution role still needs Kinesis (source) permissions** because the ESM uses them on the function's behalf.
    - Contrast with async: in async invocation the source _delivers_ the event, so the function only needs source permissions if it wants to _read additional data_ (e.g. fetching the actual S3 object).
- Persistently failing batches can be sent to an **SQS queue or SNS topic**.

---

## Versions & Aliases

- A **version** = the function's **code + configuration** (resources, environment variables, all config).
- 🎯 A **published version is immutable** and gets its **own ARN** — it never changes once published.
- **`$LATEST`** points at the most recent version and is **mutable** (changes as you publish).
- **Aliases** (e.g. `dev`, `stage`, `prod`) point at specific versions and are **mutable** (can be repointed).
- Typical pattern: publish immutable versions for major changes; use aliases so app components target specific version numbers.

---

## Startup: Cold Starts, Warm Starts & Provisioned Concurrency

![[lambda-invocation-cold-start-vs-warm-start.png]]

**Diagram Walkthrough:** left side = **cold start** (blue function-code box). An event arrives; the full environment must be _created and configured_ — including function code download — taking **~100ms** (the iceberg icon, signaling hidden cost). The big orange **"Future Invocations"** arrow crosses to the right side = **warm start** (orange box). The same execution context is **reused**; a new event is passed in but context creation is skipped, so it's **~1–2ms** (the campfire = kept warm). The `/tmp` + animal icons on both sides show data persisting in the reused context. Center captions cover context-reuse caveats and **provisioned concurrency**.

- **Execution context** = the environment (a small container with allocated resource) the function runs in.
- 🎯 **Cold start:** first invocation must **create + configure** the environment, download/install the runtime, and download/install the deployment package — **~100ms or more** before code runs.
- 🎯 **Warm start:** a subsequent invocation soon after can **reuse** the existing context, skipping setup — code runs in **~1–2ms**.
- Contexts are deleted after too long between invocations (→ another cold start). **One invocation runs per context at a time**, so 20 simultaneous invocations can mean 20 cold starts.

### Provisioned concurrency

- 🎯 Tell AWS in advance to **pre-create and keep X contexts warm**, improving start speed. Use for known high-load periods or ahead of a production release.

### Code-level optimization

- ⚠️ Always assume a fresh, clean context — never _require_ reuse — but you _can_ optimize for it:
    - Pre-download reusable assets to `/tmp`; a later invocation in the same context can reuse them.
    - Declare reusable objects (e.g. **database connections**) **outside the function handler** — these persist for future invocations in the same context. Anything **inside the handler** is scoped to that single invocation only.

---

## Course-vs-Reality Discrepancies (quick index)

|#|Topic|Course says|Current AWS reality|Source|
|---|---|---|---|---|
|⚠️ 1|Resource policy editing|CLI/API only; **not** editable in console UI|Viewable **and editable** in console (Configuration → Permissions)|[AWS docs](https://docs.aws.amazon.com/lambda/latest/dg/access-control-resource-based.html)|
|⚠️ 2|Old VPC ENI model|Presented as current-ish "old way"|Superseded by shared-ENI (per SG+subnet) model — already reflected in the "NEW WAY" panel|AWS Lambda VPC docs|

_(Discrepancies are also flagged inline near the relevant content above.)_

---

## Quick-Reference Table

|Attribute|Value|Notes|
|---|---|---|
|Runtimes|Python, Node.js, Ruby, Java, Go, C#|Custom via layers (e.g. Rust)|
|Memory|128 MB – 10240 MB, 1 MB steps|Directly set|
|vCPU|Indirect; **1769 MB = 1 vCPU**|Linear; ~6 vCPUs at 10240 MB 📚|
|`/tmp` storage|512 MB default → 10240 MB|Ephemeral scratch only|
|Timeout|**900s / 15 min** max|Use Step Functions beyond this|
|Deployment package|50 MB zipped / 250 MB unzipped 📚|Not in transcript; per AWS quotas — container images support up to 10 GB uncompressed|
|Cold start|~100 ms 📚|Full context creation|
|Warm start|~1–2 ms 📚|Context reused|
|Async retries|0–2 (configurable)|Lambda handles retry|
|Invocation types|Sync / Async / ESM|See sections above|

> 📚 Figures marked with 📚 (deployment package limits, cold/warm-start timings, 6-vCPU ceiling) appear on the diagrams but not in the spoken transcript. The 128/10240 MB range, 1769 MB vCPU threshold, /tmp sizes, and 15-min timeout are confirmed against current [AWS Lambda docs](https://docs.aws.amazon.com/lambda/latest/dg/configuration-memory.html).

---

## Exam Traps (consolidated)

- 🎯 **"Docker"** in a question = **NOT Lambda** (traditional containers → ECS). Lambda _container images_ are a separate thing.
- 🎯 **>15 minutes** of work → not a single Lambda invocation; use **Step Functions**.
- 🎯 **VPC Lambda = normal VPC resource** — needs endpoints/NAT/IGW for outside access, plus **EC2 network permissions** on the execution role.
- 🎯 **No CloudWatch Logs output** → suspect a **missing execution-role permission**.
- 🎯 **ESM permissions:** execution role needs **source-service permissions** (e.g. Kinesis) even though the code doesn't directly read the source.
- 🎯 **Async invocation → idempotency required** (retries re-run the same event).
- 🎯 **Published versions are immutable**; `$LATEST` and **aliases are mutable**.
- 🎯 **Sync = client handles retries; Async = Lambda handles retries.**

---

## Open Questions (for Roger)

1. **Cross-linking:** this note references several existing vault topics — do you want wikilinks added for [[Aurora]], [[EFS]], [[DynamoDB Streams]], [[Kinesis]], [[API Gateway]], [[Step Functions]], [[VPC Endpoints]], [[NAT Gateway]], and [[EC2 Instance Roles]] (for the execution-role comparison)? I left them as plain text pending your confirmation on which notes already exist.
2. **Sub-note candidates:** would you like dedicated sub-notes for **Provisioned Concurrency**, **Lambda Layers**, or **Lambda + VPC networking (shared-ENI architecture)**? Each has enough exam surface to stand alone, and the VPC-networking one could be shared with your existing VPC notes.
3. **Deployment package limits:** I added the 50 MB/250 MB figures from the diagram and flagged them 📚 since they're not in the transcript. Want me to verify the _exact_ current AWS quota values (direct-upload vs. S3 vs. container-image paths differ) and cite them, or leave the diagram figures as-is?
4. **Resource-policy discrepancy:** I documented the console-editing correction inline and in the discrepancy table. Do you want me to keep both the course claim and the current reality (as I've done), or collapse it to just the current fact with a one-line historical note?