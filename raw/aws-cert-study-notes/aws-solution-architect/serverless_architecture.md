---

tags:

- aws
- saa
- architecture
- serverless

---

# Serverless Architecture

## Overview

Serverless is a **software** architecture (not a hardware one) whose goal is to manage **few, if any, servers** — because servers carry overhead in the form of **cost, administration, and risk**. There _are_ still servers running behind the scenes; the point is that _you_ don't manage them.

It blends the best parts of two other patterns:

- **Microservices** — break the application into small, independent pieces.
- **Event-driven architecture** — nothing runs until an event requires it.

Serverless pushes decomposition even further than microservices: an application becomes a collection of small, specialized **functions** that start up, do one thing well, and stop. In AWS this is **Lambda** (a Function-as-a-Service / FaaS product); other clouds have equivalents (e.g. Azure Functions), and the specific technology matters less than the _pattern_.

Core properties of these function environments:

- **Stateless** — the function assumes a clean, empty environment.
- **Ephemeral** — the environment exists only for the duration of execution.

Because each environment is identical, easy to provision, and short-lived, functions can run anywhere and scale cheaply. Each invocation typically: obtains the data it needs → does something → optionally stores the result persistently or hands output to something else.

**Event-driven cost model:** nothing runs until an event (system-generated or customer interaction) occurs. Lambda is billed on **execution duration**, so an idle serverless application is **close to zero cost** until an event fires. Serverless architectures generally have **no persistent compute usage**.

**Design philosophy:** consume-as-a-service wherever possible, code as little as possible, use FaaS for general-purpose compute, and don't reinvent the wheel — lean on managed services:

- **S3** for persistent object storage
- **DynamoDB** for persistent data storage
- **Third-party identity providers** (Google, Facebook, Twitter/X, corporate AD) instead of building your own identity store
- Media manipulation via a managed transcoding service _(see the ⚠️ discrepancy below — the course's Elastic Transcoder has since been retired)_

---

## Diagram Walkthrough

![[serverless-architecture-example.png]]

**How to read this diagram.** Top context bar: the title "Serverless Architecture Example" and the crossed-out server icon (top-left) reinforce the "no self-managed servers" theme; branding is Cantrill/"PetTube." The **numbered pink circles (1–9) are the sequence of steps**, and pink arrows show data/request flow. Read left→right, following Julie (the user, bottom-left) as she authenticates, uploads a video, that video gets processed, and finally she retrieves her media.

> [!note] 📚 Diagram vs. transcript naming The narration calls the service **AWS Elastic Transcoder** throughout, but the diagram labels the transcoding service **AWS Elemental MediaConvert**. This is Cantrill updating the _diagram_ to reflect current AWS while the _audio_ still says the old name. See the ⚠️ discrepancy section — for the exam, understand the role ("managed transcoding service"), not the brand.

Step-by-step (matching the numbered circles):

1. **HTML + JS** — Julie's laptop browses to an **S3 bucket serving a static website** and downloads HTML with embedded client-side **JavaScript**. This JS _is_ the PetTube front end, running in her browser. **No self-managed compute so far.**
2. **Authenticate with Google IDP → receive Token** — the JS talks to a third-party identity provider (Google here). Google validates Julie's username/password and returns an **identity token** proving she authenticated.
3. **Swap Google token for AWS Temp Credentials** — AWS can't consume third-party identities directly. The JS sends the Google token to **Amazon Cognito**, which exchanges it for **temporary AWS credentials**.
4. **Upload Woofie holiday video using temp credentials** — the JS uses those temporary credentials to upload the master video into the **Originals Bucket** (S3). _Everything to here used only managed services: S3, Cognito, Google IDP — zero self-managed compute._
5. **Event** — the Originals bucket is configured to emit an **event** on object upload; the event (containing the uploaded object's details) **invokes the Processing Lambda**.
6. **Jobs** — the Processing Lambda creates **transcoding jobs** in the managed transcoding service (diagram: MediaConvert), one job per required output video size.
7. **Load Original Video** — each job reads the master video back from the **Originals Bucket** at the start of its processing cycle.
8. **Fan-out to outputs** — each job writes a resized rendition to the **Transcode Bucket** (S3), and writes metadata about each new video into **DynamoDB**. (The multiple parallel arrows depict one job per size.)
9. **Client calls My Media** — separately, Julie clicks in the client-side UI to invoke a second Lambda, **My Media**, which reads from **DynamoDB**, identifies which objects in the Transcode Bucket are hers, and returns **URLs** so her browser can display all the videos she's uploaded.

**Net result:** master upload → transcode into multiple sizes → retrieval, with no EC2 instances and no self-managed database servers anywhere. Compute is consumed only when events fire (system-side or client-side).

---

## Mechanics

**Why third-party identity instead of IAM users**

- Lower admin overhead — you don't maintain a user store.
- 🎯 **Hard limit: 5,000 IAM users per AWS account.** If PetTube used IAM users for auth, it would cap at 5,000 users, and each user would need an _additional_ account (extra username + password) on top of one they already have. _(Verified current — [AWS IAM quotas](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_iam-quotas.html); the 5,000 figure is unchanged as of 2026.)_
- Users likely already have a Google/Facebook/etc. account, so you reduce the number of accounts they must maintain.

**The Cognito role (critical exam concept)**

- Third-party (web) identity token → Cognito → **temporary AWS credentials**.
- 🎯 Cognito is the _broker/exchange_ between an external IdP and AWS. AWS cannot use a Google/Facebook token directly.

**Event-driven invocation**

- S3 event notifications invoke Lambda on object upload (step 5).
- The event payload carries the details of the object that triggered it.

**Consumption-based cost**

- Base usage cost ≈ zero. You pay for S3 storage, DynamoDB, Lambda execution time, and transcoding job time — only when actually used.

---

## Course-vs-Reality Discrepancies

> [!warning] ⚠️ Elastic Transcoder has been **discontinued** The transcript repeatedly names **AWS Elastic Transcoder** as the media service. **AWS discontinued Amazon Elastic Transcoder on November 13, 2025** — after that date the console and its resources are no longer accessible, and it was already unavailable to new accounts before then. The replacement is **AWS Elemental MediaConvert** (which is what the diagram now shows). _Sources: [AWS Elastic Transcoder details page](https://aws.amazon.com/elastictranscoder/details/); [AWS migration blog](https://aws.amazon.com/blogs/media/migrating-workflows-from-amazon-elastic-transcoder-to-aws-elemental-mediaconvert/)._ **For the SAA-C03 exam:** questions may still reference Elastic Transcoder _or_ MediaConvert. Learn the **concept** — a managed, file-based video transcoding service that takes a master file and outputs multiple renditions — and treat the two names as interchangeable for that role. Don't memorize Elastic Transcoder as a "live" service.

> [!note] 📚 API Gateway is omitted for simplicity The lesson explicitly notes this is a **simplified** diagram: in reality **API Gateway** would sit between the client-side JavaScript and the Lambda functions (steps 5 and 9), providing the managed HTTP front door to Lambda. Worth knowing because API-Gateway-in-front-of-Lambda is an extremely common exam pattern.

---

## Exam Traps

- 🎯 **"Serverless" ≠ "no servers."** Servers exist; you just don't manage them. A question framing serverless as literally serverless is testing this.
- 🎯 **Third-party token → AWS = Cognito.** If a scenario logs users in via Google/Facebook and then needs AWS access, the exchange service is **Cognito** (returning **temporary** credentials). Don't pick IAM users.
- 🎯 **5,000 IAM users per account** is the trigger figure that pushes you toward federated/third-party identity for consumer-facing apps. Large/unknown user counts → _not_ IAM users.
- 🎯 **S3 static website + client-side JavaScript** is a fully serverless front end — no compute. Recognize "download HTML/JS from S3, JS runs in browser" as the front-end pattern.
- 🎯 **S3 event → Lambda** is the canonical event-driven trigger. "Object uploaded, then process it" → S3 event notification invoking Lambda.
- 🎯 **Idle serverless ≈ zero cost / consumption-based billing.** Any answer implying persistent/always-on compute cost for a serverless design is wrong.
- ⚠️ Don't select **Elastic Transcoder** as a current service in a "which service today" question — but recognize it if the exam still references it. The transcoding _role_ answer is **MediaConvert**.

---

## Quick-Reference Table

|Component (diagram)|AWS service|Serverless role|
|---|---|---|
|Static Website|**S3** (static website hosting)|Serves HTML + client-side JS front end|
|Google IDP|**Third-party IdP** (Google/Facebook/X/AD)|Authenticates the user, issues identity token|
|Cognito|**Amazon Cognito**|Exchanges third-party token for temp AWS credentials|
|Originals Bucket|**S3**|Stores uploaded master videos; emits upload event|
|Processing Lambda|**AWS Lambda** (FaaS)|Event-triggered; creates transcoding jobs|
|AWS Elemental MediaConvert _(course says Elastic Transcoder)_|**MediaConvert**|Transcodes master → multiple sizes|
|Transcode Bucket|**S3**|Stores output renditions|
|DynamoDB|**Amazon DynamoDB**|Persistent metadata store for video records|
|My Media|**AWS Lambda** (FaaS)|Client-invoked; reads DynamoDB, returns video URLs|

**Managed services doing the heavy lifting:** S3, Cognito, Lambda, MediaConvert, DynamoDB — plus an external IdP. **Self-managed compute used: none.**

---

## Open Questions (for Roger)

1. **Cognito depth** — This note treats Cognito as a single "token exchange" box. Do you want a dedicated `[[Cognito]]` sub-note distinguishing **User Pools** (identity store / sign-up-sign-in) from **Identity Pools / Federated Identities** (the token→temp-credentials exchange shown here)? The step-3 behavior in this diagram is specifically an **Identity Pool** function, which is a common exam distinction.
2. **API Gateway cross-link** — Want me to add a `[[API Gateway]]` note and wire it in, since the lesson flags it as the "real" component between client and Lambda? It pairs naturally with a future `[[Lambda]]` note.
3. **Lambda note** — There's no standalone `[[Lambda]]` note referenced yet in the vault surface I can see. Given how central FaaS is, should I stub one (execution duration billing, stateless/ephemeral environments, event sources)? I can cross-link this note's steps 5 and 9 to it.
4. **S3 event notifications** — Worth a small `[[S3 Event Notifications]]` note (or a section in an existing S3 note) covering event types and Lambda/SQS/SNS destinations, since step 5 is the trigger mechanic?
5. **MediaConvert vs. Elastic Transcoder** — Do you want me to preserve _both_ names prominently (as I've done), or would you rather I fully rewrite to MediaConvert with Elastic Transcoder only as a footnote? You've previously preferred keeping the course figure _and_ current AWS figure when both may appear on exams — I've defaulted to that here.