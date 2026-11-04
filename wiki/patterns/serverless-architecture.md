---
title: Serverless Architecture
type: pattern
tags: [serverless, faas, lambda, event-driven, microservices, s3, dynamodb, cognito, mediaconvert, elastic-transcoder, s3-events, api-gateway, stateless, ephemeral, consumption-billing]
certs: [aws-saa]
updated: 2026-11-04
---

# Serverless Architecture

**Serverless** is a *software* architecture (not a hardware one) whose goal is to manage **few, if any, servers** — because servers carry overhead in the form of **cost, administration, and risk**. There *are* still servers running behind the scenes; the point is that *you* don't manage them. This is the concrete, service-by-service realisation of the abstract "event-driven, consume-only-when-working" end state from [Architecture Evolution](architecture-evolution.md#stage-6--event-driven).

Serverless blends the best parts of two other patterns:

- **Microservices** — break the application into small, independent pieces.
- **Event-driven architecture** — nothing runs until an event requires it.

It pushes decomposition even further than [microservices](architecture-evolution.md#stage-5--microservices): an application becomes a collection of small, specialised **functions** that start up, do one thing well, and stop. In AWS this is **[Lambda](../services/lambda.md)** (a Function-as-a-Service / FaaS product); other clouds have equivalents (e.g. Azure Functions), and the specific technology matters less than the *pattern*.

Core properties of these function environments:

- **Stateless** — the function assumes a clean, empty environment (see [Lambda statelessness](../services/lambda.md#statelessness)).
- **Ephemeral** — the environment exists only for the duration of execution.

Because each environment is identical, easy to provision, and short-lived, functions can run anywhere and scale cheaply. Each invocation typically obtains the data it needs, does one thing, and then optionally stores the result persistently or hands its output to something else.

Statelessness and the 15-minute ceiling are exactly what make a *single* function a bad home for a long, multi-step process. When a serverless workflow needs sequence, branching, waiting, or state carried across steps, the orchestration layer is **[Step Functions](../services/step-functions.md)** — a state machine that coordinates the short-lived functions rather than replacing them.

## Design Philosophy

The **event-driven cost model** is the heart of the pattern: nothing runs until an event (system-generated or a customer interaction) occurs. Lambda is billed on **execution duration**, so an idle serverless application is **close to zero cost** until an event fires. Serverless architectures generally have **no persistent compute usage**.

The design instinct is *consume-as-a-service wherever possible*: code as little as possible, use FaaS for general-purpose compute, and don't reinvent the wheel — lean on managed services:

- **[S3](../services/s3.md)** for persistent object storage.
- **DynamoDB** for persistent data storage.
- **Third-party identity providers** (Google, Facebook, corporate AD) instead of building your own identity store.
- **Managed media transcoding** (see the [discrepancy note](#course-vs-reality-discrepancies) — the course's Elastic Transcoder has since been retired).

## Diagram Walkthrough — PetTube Example

![Serverless architecture example: Julie on a laptop (bottom-left) downloads HTML and JavaScript from an S3 static website, authenticates with Google to get a token, swaps that token at Cognito for temporary AWS credentials, uploads a master video to the Originals S3 bucket, which emits an event that invokes a Processing Lambda; the Lambda creates transcoding jobs in AWS Elemental MediaConvert, each job reads the master and fans out resized renditions to the Transcode S3 bucket plus metadata to DynamoDB; a separate My Media Lambda reads DynamoDB and returns video URLs to Julie's browser. Nine numbered pink circles mark the sequence; a crossed-out server icon top-left reinforces the no-self-managed-servers theme](../assets/serverless-architecture-example.png)

**How to read it.** The **numbered circles (1–9)** are the sequence of steps, and the arrows show data/request flow. Read left to right, following Julie (the user, bottom-left) as she authenticates, uploads a video, that video gets processed, and finally she retrieves her media. The crossed-out server icon top-left reinforces the "no self-managed servers" theme.

Step by step:

1. **HTML + JS** — Julie's browser loads an [S3 bucket serving a static website](../services/s3.md) and downloads HTML with embedded client-side **JavaScript**. This JS *is* the PetTube front end, running in her browser. **No self-managed compute so far.**
2. **Authenticate with Google IdP, receive token** — the JS talks to a third-party identity provider (Google here). Google validates Julie's username/password and returns an **identity token** proving she authenticated.
3. **Swap Google token for AWS temp credentials** — S3 does not accept the Google login token as AWS credentials. The JS sends the Google token to **[Amazon Cognito](../services/cognito.md)**, which exchanges it for **temporary AWS credentials**.
4. **Upload the master video using temp credentials** — the JS uses those temporary credentials to upload the master video into the **Originals Bucket** (S3). Everything to here used only managed services (S3, Cognito, Google IdP) with **zero self-managed compute**.
5. **Event** — the Originals bucket is configured to emit an **event** on object upload; the event (containing the uploaded object's details) **invokes the Processing Lambda**. This is the canonical [S3 event notification](../services/s3.md#event-notifications) trigger, and on the Lambda side it is an [asynchronous invocation](../services/lambda.md#asynchronous-invocation).
6. **Jobs** — the Processing [Lambda](../services/lambda.md) creates **transcoding jobs** in the managed transcoding service (diagram: MediaConvert), one job per required output video size.
7. **Load original video** — each job reads the master video back from the **Originals Bucket** at the start of its processing cycle.
8. **Fan-out to outputs** — each job writes a resized rendition to the **Transcode Bucket** (S3) and writes metadata about each new video into **DynamoDB**. (The multiple parallel arrows depict one job per size.)
9. **Client calls My Media** — separately, Julie clicks in the client-side UI to invoke a second Lambda, **My Media**, which reads from **DynamoDB**, identifies which objects in the Transcode Bucket are hers, and returns **URLs** so her browser can display all the videos she's uploaded.

**Net result:** a master upload is transcoded into multiple sizes and then retrieved, with **no EC2 instances and no self-managed database servers anywhere**. Compute is consumed only when events fire (system-side at step 5 or client-side at step 9).

> [!note] API Gateway is omitted for simplicity
> The lesson notes this is a **simplified** diagram. In reality, **[API Gateway](../services/api-gateway.md)** would sit between the client-side JavaScript and the Lambda functions (steps 5 and 9), providing the managed HTTP front door to Lambda. API-Gateway-in-front-of-Lambda is an extremely common exam pattern, and it maps to the **web tier** of the [global/regional model](global-regional-architecture.md).

## Mechanics

### Why third-party identity instead of IAM users

- Lower admin overhead — you don't maintain a user store.
- **Hard limit: 5,000 [IAM](../services/iam.md) users per AWS account.** If PetTube used IAM users for auth, it would cap at 5,000 users, and each user would need an *additional* account (extra username + password) on top of one they already have. (Verified current — [AWS IAM quotas](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_iam-quotas.html); the 5,000 figure is unchanged as of 2026.)
- Users likely already have a Google/Facebook/etc. account, so you reduce the number of accounts they must maintain.

### The Cognito role (critical exam concept)

- A third-party (web) identity token goes to **Cognito**, which returns **temporary AWS credentials**.
- Cognito is the **broker/exchange** between an external identity provider and AWS. S3 and DynamoDB do not accept a Google/Facebook login token as an AWS request signature. The token-to-temporary-credentials exchange shown at step 3 is specifically an **Identity Pool** (federated identities) function, as distinct from a **User Pool** (a sign-up/sign-in identity store) — a common exam distinction. This ties into [identity federation](../concepts/identity-and-access.md) and the [temporary credentials from STS](../services/iam.md) that AWS roles hand out.

A user pool can be added when the app needs managed registration or a consistent token format across sign-in providers: [user pool → identity pool](../services/cognito.md#combining-user-and-identity-pools). An API-only design can instead validate user pool tokens at API Gateway and let the backend access AWS with its own role.

### Event-driven invocation

- [S3 event notifications](../services/s3.md#event-notifications) invoke [Lambda](../services/lambda.md) on object upload (step 5).
- The event payload carries the details of the object that triggered it, so the function knows *what* to process without polling.

### Consumption-based cost

- Base usage cost is approximately zero. You pay for S3 storage, DynamoDB, Lambda execution time, and transcoding job time — only when actually used. This is the "[only consume when working](architecture-evolution.md#why-only-consume-when-working-matters)" property made concrete.

## Course-vs-Reality Discrepancies

> [!warning] Elastic Transcoder has been discontinued
> The course transcript repeatedly names **AWS Elastic Transcoder** as the media service. **AWS discontinued Amazon Elastic Transcoder on November 13, 2025** — after that date the console and its resources are no longer accessible, and it was already unavailable to new accounts before then. The replacement is **AWS Elemental MediaConvert** (which is what the diagram now shows). For the SAA-C03 exam, questions may still reference Elastic Transcoder *or* MediaConvert. Learn the **role** — a managed, file-based video transcoding service that takes a master file and outputs multiple renditions — and treat the two names as interchangeable for that role. Don't memorise Elastic Transcoder as a "live" service. Sources: [AWS Elastic Transcoder details page](https://aws.amazon.com/elastictranscoder/details/); [AWS migration blog](https://aws.amazon.com/blogs/media/migrating-workflows-from-amazon-elastic-transcoder-to-aws-elemental-mediaconvert/).

> [!note] Diagram vs. transcript naming
> The narration calls the service **AWS Elastic Transcoder** throughout, but the diagram labels it **AWS Elemental MediaConvert** — Cantrill updating the *diagram* to reflect current AWS while the *audio* still says the old name. For the exam, understand the role ("managed transcoding service"), not the brand.

## Exam Traps

- **"Serverless" is not "no servers."** Servers exist; you just don't manage them. A question framing serverless as literally serverless is testing this.
- **Third-party token to AWS access = Cognito.** If a scenario logs users in via Google/Facebook and then needs AWS access, the exchange service is **Cognito** (returning **temporary** credentials). Don't pick IAM users.
- **5,000 IAM users per account** is the trigger figure that pushes you toward federated/third-party identity for consumer-facing apps. Large or unknown user counts mean **not** IAM users.
- **S3 static website + client-side JavaScript** is a fully serverless front end — no compute. Recognise "download HTML/JS from S3, JS runs in browser" as the front-end pattern.
- **S3 event, then Lambda** is the canonical event-driven trigger. "Object uploaded, then process it" means an S3 event notification invoking Lambda.
- **Idle serverless is approximately zero cost** (consumption-based billing). Any answer implying persistent/always-on compute cost for a serverless design is wrong.
- **Don't select Elastic Transcoder** as a current service in a "which service today" question — but recognise it if the exam still references it. The transcoding *role* answer is **MediaConvert**.

## Quick-Reference Table

| Component (diagram) | AWS service | Serverless role |
| --- | --- | --- |
| Static Website | **S3** (static website hosting) | Serves HTML + client-side JS front end |
| Google IdP | **Third-party IdP** (Google/Facebook/AD) | Authenticates the user, issues identity token |
| Cognito | **Amazon Cognito** | Exchanges third-party token for temp AWS credentials |
| Originals Bucket | **S3** | Stores uploaded master videos; emits upload event |
| Processing Lambda | **AWS Lambda** (FaaS) | Event-triggered; creates transcoding jobs |
| MediaConvert *(course says Elastic Transcoder)* | **MediaConvert** | Transcodes master into multiple sizes |
| Transcode Bucket | **S3** | Stores output renditions |
| DynamoDB | **Amazon DynamoDB** | Persistent metadata store for video records |
| My Media | **AWS Lambda** (FaaS) | Client-invoked; reads DynamoDB, returns video URLs |

**Managed services doing the heavy lifting:** S3, Cognito, Lambda, MediaConvert, DynamoDB — plus an external IdP. **Self-managed compute used: none.**

## References

- [Architecture Evolution — Coupling, Queues & Event-Driven](architecture-evolution.md) — the abstract progression this pattern concretises
- [AWS Lambda](../services/lambda.md) — the FaaS compute primitive
- [Step Functions](../services/step-functions.md) — the state-machine orchestration layer for multi-step serverless workflows
- [S3](../services/s3.md) — static hosting, object storage, and event notifications
- [API Gateway](../services/api-gateway.md) — the managed front door between clients and Lambda, omitted from the diagram for simplicity
- [Global & Regional AWS Architecture](global-regional-architecture.md) — where API Gateway/Lambda sit in the tier model
- [AWS IAM quotas](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_iam-quotas.html) — the 5,000-IAM-users limit
- [Migrating from Elastic Transcoder to MediaConvert](https://aws.amazon.com/blogs/media/migrating-workflows-from-amazon-elastic-transcoder-to-aws-elemental-mediaconvert/)
- Adrian Cantrill, AWS SAA course — Serverless Architecture (PetTube example)
- Source: `raw/aws-cert-study-notes/aws-solution-architect/serverless_architecture.md`
