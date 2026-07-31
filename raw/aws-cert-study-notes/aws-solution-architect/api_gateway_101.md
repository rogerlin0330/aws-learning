---

tags:

- aws
- saa
- api-gateway
- serverless
- networking

---

# API Gateway 101

## Overview

**API Gateway** is a managed service that lets you **create, publish, and manage APIs**. An API (application programming interface) is how applications talk to each other — e.g. the Netflix app on your TV uses an API to talk to Netflix's backend.

Architecturally, API Gateway is an **entry point / endpoint** that sits _between_ the clients consuming an API and the **integrations** — the backend services that actually provide the API's functionality.

Key characteristics:

- **Highly available and scalable** — delivered as a managed service, so you don't manage either.
- Handles **authorization** (who can access your APIs).
- Handles **throttling** (how often individuals can call APIs).
- Performs **caching** (reduces how often backend integrations are called).
- Supports **CORS** (controls the security of cross-domain calls within browsers).
- Supports **transformations** (reshapes requests/responses between client and integration).
- Supports the **OpenAPI spec**, so API definition files can be imported/exported easily.
- **Directly integrates with AWS services** — you can write to DynamoDB, start a Step Function, or send to an SNS topic _without any backing compute_.
- Is a **public service** — it can front services running in AWS **or** on-premises.
- Effective **migration tool**: provides a consistent front end while backend services are moved on-prem → AWS, or re-architected monolith → serverless.
- Can provide **HTTP**, **REST**, or **WebSocket** APIs.

---

## Diagram walkthrough — Overview

![[api-gateway-overview.png]]

**How to read it:** clients on the left, API Gateway (the large red-bordered box) in the middle, integrations on the right, supporting services (cache + CloudWatch) below.

**Clients (left).** Three example consumers — mobile applications, other APIs, and web applications loaded from **S3 static hosting** (note the JS app served from an S3 bucket, whose arrow points _up_ into the laptop client). They all connect to the API using the **API Endpoint DNS** name.

**API Gateway (center).** The box is split into two horizontal bands showing the three interaction phases:

- **① Request (top, green band):** reading left-to-right — **AUTHORIZE → VALIDATE → TRANSFORM**. Incoming client requests are authorized, validated, and transformed into a form the integration can handle. The green arrow points _right_ toward the integrations.
- **② Integrations (right, dashed box):** the backends providing functionality — DynamoDB (DDB), SNS, Step Functions, HTTP Endpoints (AWS or on-prem), and Lambda.
- **③ Response (bottom, orange band):** reading right-to-left — **TRANSFORM → PREPARE → RETURN**. The integration's output is transformed, prepared, and returned to the client. The orange arrow points _left_ back toward the clients.

**Footer services.**

- **API Cache** (blue-bordered box, double-headed green arrow): reduces the number of calls made to backend integrations and improves client performance.
- **CloudWatch** (pink icon, one-way arrow in): stores **logs** (full stage request/response logs) and **metrics** for both the client side and the integration side.

📚 The centre caption sums the whole thing up: _"API Gateway is an intermediary between clients and integrations."_

---

## The three interaction phases

Most API Gateway interactions have **three phases**:

1. **Request phase** — the client makes a request; it flows through API Gateway to the integration. At a high level this does three things: **authorize, validate, and transform** the incoming request into a form the integration can handle.
2. **Integration** — the backend service does the work.
3. **Response phase** — takes the integration's output, **transforms / prepares** it, and **returns** it to the client.

API Gateway integrates with **CloudWatch** to store logging and metric data for both request-side and response-side operations, and provides a **cache** that improves client performance while reducing backend calls.

---

## Authentication

API Gateway supports a range of authentication methods, including **fully open access** (no auth). The three supported auth paths worth knowing:

### 1. Cognito user pools

The client **authenticates with Cognito** and receives a **Cognito token**. It passes that token in with each request to API Gateway. Because of API Gateway's tight integration with Cognito, it can **natively validate** the token.

### 2. Lambda-based authorization (formerly "custom authorization")

Used when API Gateway can't natively validate the credential.

### 3. IAM

Authenticate/authorize by passing **credentials in the headers**.

> 🎯 The transcript notes IAM auth "is beyond what's required for the exam" — know that it exists (credentials in headers) but focus on **Cognito** and **Lambda authorizers**.

### Diagram walkthrough — Authentication (Lambda authorizer flow)

![[api-gateway-authentication.png]]

**How to read it:** client top-center, API Gateway center, Cognito on the left, Lambda Authorizer on the right, the request-handling Lambda at the bottom. Two auth paths are overlaid on one picture.

**Cognito path (left, pink dotted + double arrow).**

- Dotted arrow, client → Cognito: the client **authenticates with Cognito and receives a token**.
- Solid arrow down, client → API Gateway: **pass token with request**.
- Double-headed arrow, API Gateway ↔ Cognito: API Gateway **verifies token validity** natively.

**Lambda authorizer path (right, orange arrows).** This is the flow the diagram spends most ink on:

1. Client calls API Gateway with a **bearer token (ID)** — some token asserting an identity.
2. API Gateway, not knowing how to validate it natively, **calls the Lambda Authorizer** ("Lambda authorizer called" — orange arrow API GW → Lambda).
3. The authorizer does its job: **calls an external ID Provider, or does compute-based checking of the ID** (e.g. against a local user store) — the up/down arrows to the ID-badge icon top-right.
4. The authorizer returns an **IAM Policy and a principal identifier** back to API Gateway (orange arrow Lambda → API GW).
5. API Gateway **evaluates the policy** and either:
    - **handles the request via the Lambda integration** (green arrow down to the request-handling Lambda), **or**
    - returns a **`403 ACCESS_DENIED`** to the client.

📚 Note the **two different Lambdas** in this diagram: the **Lambda Authorizer** (validates identity, returns a policy) is distinct from the **integration Lambda** (does the actual application work). They're separate functions with separate jobs.

---

## Endpoint types

Three endpoint types can be configured for an API:

| Endpoint type      | Behavior                                                                       | When to use                              |
| ------------------ | ------------------------------------------------------------------------------ | ---------------------------------------- |
| **Edge-optimized** | Incoming requests routed to the **nearest CloudFront PoP** (point of presence) | Geographically distributed clients       |
| **Regional**       | Regional endpoint, **no CloudFront network**, low overhead                     | Clients/consumers in the **same region** |
| **Private**        | Only accessible **within a VPC via an interface endpoint**                     | Completely **private** APIs              |

---

## Stages

When you deploy an API configuration in API Gateway, you deploy it **to a stage** (e.g. `prod` and `dev` for the Catagram app). **Most things in API Gateway are defined per stage.**

- Each stage has its **own unique endpoint URL** and its **own settings**.
- Each stage is deployed onto **individually** — e.g. v1 of the config (backed by v1 of a Lambda) in `prod`, v2 (backed by a separate Lambda) in `dev`.
- Deployments on a stage can be **rolled back** → good isolation/testing.

📚 **Stage names are arbitrary labels, not enforced environments.** `prod`/`dev` are convention. The name is just a string that becomes part of the invoke URL path (`.../{stage}/...`). You could equally name stages by version, team, tenant, or feature. Nothing about the name "prod" makes API Gateway treat it as production — behavior comes from the deployment attached and the stage's settings/variables. _(Source: [AWS — Setting up a stage](https://docs.aws.amazon.com/apigateway/latest/developerguide/set-up-stages.html))_

📚 **Deployments vs. stages (precise model).** API Gateway has two distinct concepts the diagram's "V1/V2" labels blur together:

- **Deployment** — an **immutable snapshot** of your API's resources, methods, integrations, and settings, each with a **unique deployment ID**. Changes you make don't take effect until you create a _new_ deployment.
- **Stage** — a **named pointer** to one deployment.

⚠️ **"V1/V2" is a _backend_ concept, not an API Gateway one.** API Gateway itself has no numbered "versions"; it has deployments (unique IDs) and stages (names). The versioning in Cantrill's Stages diagram lives in the **backend** — e.g. **Lambda function versions/aliases**. The diagram collapses the API Gateway layer and the backend layer into "V1/V2 → LambdaV1/LambdaV2" for legibility. Also, the backend needn't be Lambda — it could be an HTTP endpoint or another AWS service, so "V1/V2" generically means "old backend behavior vs. new."

### Canary deployments

Stages can be **enabled for canary deployments**. Once enabled:

- Any **new deployment to that stage** is deployed onto the **canary sub-part** of the stage, **not the base stage** itself.
- **Traffic distribution** between the base stage and the canary is set by a **user-configurable percentage**.
- The canary can eventually be **promoted** to become the new base stage, and the cycle repeats.
- If the canary is unhealthy (bugs, worse performance), you can **remove it and revert** to the base stage.

**Worked example (from the lecture):** v2 is tested by the dev team, then canary is enabled on `prod`. v2 is deployed to `prod` → it lands in the **canary** (because canary is enabled). Adjust traffic split between base `prod` and its canary until happy → **promote** the canary to full base stage → repeat.

📚 **Canary workflow — extra mechanics beyond the lecture:**

- **Canary is a REST API feature only** — not available on HTTP APIs or WebSocket APIs. _(Source: [AWS — Canary release deployments](https://docs.aws.amazon.com/apigateway/latest/developerguide/canary-release.html))_
- **Console flow:** deploy the API to a stage first, then on that stage's **Canary** tab, create the canary and set the traffic percentage. It's configured on an _existing_ stage after deployment, not a checkbox during the first deploy.
- **You can't create a canary on a non-existent stage.** A normal deploy will _create_ a stage if the name doesn't exist; a canary deploy will **error** instead. _(Source: [AWS — Create a canary release deployment](https://docs.aws.amazon.com/apigateway/latest/developerguide/create-canary-deployment.html))_
- **"Sticky" canary state:** once canary is enabled, the **base freezes** at whatever deployment it had, and **every subsequent deploy to that stage lands on the canary** — this continues until you **promote** or **disable/remove** the canary settings.
- The canary holds **one deployment slot**, overwritten (not stacked) by each new deploy. The base stays put.
- **Promote** → canary's deployment becomes the new base, canary disabled. **Remove settings** → rolls back to base, canary discarded.
- Traffic split is **random per request** and the **percentage is adjustable** at any time.
- **Canary stage variables** can _override_ the base stage's — a common way to point the canary at LambdaV2 while base stays on LambdaV1.
- The canary gets its **own CloudWatch metrics and log group** (base log group `API-Gateway-Execution-Logs/{rest-api-id}/{stage-name}` plus a `.../Canary`-suffixed one) — this is how you validate before promoting. _(Source: [AWS — Canary release deployments](https://docs.aws.amazon.com/apigateway/latest/developerguide/canary-release.html))_

> 🎯 **Exam trap:** with canary enabled, a redeploy updates the **canary, not the live/base version**. Candidates often assume a redeploy immediately updates production — it doesn't while a canary is active.

### Diagram walkthrough — Stages

![[api-gateway-stages.png]]

**How to read it:** two stages of the same API stacked vertically — `prod` (top, customers) and `dev` (bottom, developers) — with backend Lambdas on the right.

- **Green arrow (top-left):** "APIs are deployed to stages, each stage has one deployment" → points at the `prod` stage.
- **`api.catagram.io/prod`** — used by **Customers** (blue solid arrow in). Note the distinct `/prod` path suffix.
- **`api.catagram.io/dev`** — used by **Developers** (pink solid arrow in). Distinct `/dev` suffix, its own endpoint.
- **V1 → LambdaV1** (top right): the base deployment. Solid **blue** arrow V1 → `prod` = the bulk of production traffic hits the base.
- **V2 → LambdaV2** (bottom right): the canary. Dotted **pink** arrow V2 → `prod` = a configurable _percentage_ of production traffic split off to the canary.
- Right-side captions restate the canary rules: deployments go **to the canary not the stage**; the traffic percentage is configurable and **adjustable over time**; the canary can be **promoted** to become the new base 'stage'.

📚 In practice the V1→V2 backend switch is usually handled by a **Lambda alias with weighted routing** rather than two entirely separate functions — API Gateway keeps pointing at one alias while the alias shifts weight from Version 1 to Version 2. The diagram draws two Lambda boxes to make the concept legible; the exam-relevant idea is the **stage-level traffic split**, not the exact backend wiring.

---

## Error codes (exam figures)

Error codes fall into two categories:

- **4xx — client errors:** something wrong on the client side (or how it's making the request).
- **5xx — server errors:** a valid request, but a backend issue.

| Code    | Meaning                           | Notes                                                                                    |
| ------- | --------------------------------- | ---------------------------------------------------------------------------------------- |
| **400** | Generic **client-side** error     | Hard to diagnose — many possible root causes                                             |
| **403** | **Access denied**                 | Authorizer denied the request, **or** filtered by something like **WAF**                 |
| **429** | **Throttling**                    | Exceeded a configured throttle amount                                                    |
| **502** | **Bad Gateway**                   | Backend returned **invalid output** (e.g. a Lambda returning something malformed)        |
| **503** | **Service unavailable**           | Backing endpoint offline / major service issues — _"I've seen that come up in the exam"_ |
| **504** | **Integration failure / timeout** | Backend didn't respond within the integration timeout (see below)                        |

> 🎯 Memorize the mapping: **429 = throttling**, **502 = bad backend output**, **503 = backend offline**, **504 = timeout**, **403 = access denied**.

### The 29-second integration timeout

The lecture states a **29-second limit** for any request to API Gateway: even though Lambda can run up to 15 minutes, if a Lambda backs an API and the request takes longer than **29 seconds**, you get a **504**. So back your APIs with functions that respond within 29 seconds.

⚠️ **Course-vs-reality: 29 s is now a soft default, not a hard cap.** As of **June 2024**, API Gateway allows raising the integration timeout **beyond 29 seconds** for **Regional REST APIs and private REST APIs**, via a Service Quotas increase on _"Maximum integration timeout in milliseconds."_ This was driven partly by GenAI/LLM workloads that need longer than 30 s. The trade-off: increasing it **may require a reduction in your account-level throttle quota**. The floor is still **50 ms** minimum. For the exam, the **29-second default → 504** relationship is still the expected answer, but know it's no longer an absolute hard limit. _(Source: [AWS What's New — Integration timeout increase](https://aws.amazon.com/about-aws/whats-new/2024/06/amazon-api-gateway-integration-timeout-limit-29-seconds/); [AWS — REST API quotas](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-execution-service-limits-table.html))_

---

## Caching

Caching is **configured per stage** (matters both for the exam and for production).

**Without a cache:** every request hits the backend integrations. **With a cache:** backends are called **only on a cache miss** → reduced load, reduced cost, improved performance (lower latency).

| Property        | Value                               |
| --------------- | ----------------------------------- |
| **Cache size**  | **500 MB (0.5 GB) → 237 GB**        |
| **Default TTL** | **300 seconds**                     |
| **TTL range**   | **0 (disabled) → 3,600 seconds**    |
| **Encryption**  | **Can be encrypted** ✅ (exam point) |
| **Scope**       | **Per stage**                       |

📚 Additional exam-relevant facts (fact-checked):

- **Caching is available for REST APIs only** (not HTTP/WebSocket APIs). _(Source: [AWS — API Gateway caching](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-caching.html))_
- By default, only **GET methods** are cached; other methods can be cached via **method-level** cache settings, which can **override** stage settings.
- Max cacheable **response size is 1,048,576 bytes (1 MB)**.
- Clients can request a cache bypass with the header **`Cache-Control: max-age=0`** (per-key invalidation can require authorization).
- Enabling/resizing a cache provisions a dedicated instance and takes **~4–5 minutes**.

### Diagram walkthrough — Caching

![[api-gateway-caching.png]]

**How to read it:** client (left) ↔ API Gateway **STAGE** (center) ↔ integrations (right, dashed box); the **cache** hangs below the stage.

- **Top-left facts block:** TTL default **300 s**, configurable **min 0 / max 3600 s**, **can be encrypted**, cache size **500 MB → 237 GB**.
- **Client ↔ Stage** (solid pink double arrow): normal request/response traffic.
- **Stage ↔ CACHE** (green double arrow, "API Cache"): responses served from cache on hits.
- **Bullseye + sad-face icon:** the **cache miss** — _"calls are only made to backend integrations if request is a cache miss."_
- **Stage ↔ integrations** (orange dashed arrow) → the **PROD / STAGE / DEV** boxes (illustrating that cache is set **per stage**) → the integration list: DDB, SNS, Step Functions, HTTP Endpoints, Lambda.
- **Caption:** _"Cache is defined per stage within API Gateway."_
- **Top-right:** the payoff — **reduced load & cost, improved performance.**

---

## Course-vs-current-AWS discrepancies

| Topic                   | Course framing                   | Current AWS reality                                                                                                                                                              |
| ----------------------- | -------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Integration timeout** | 29-second **hard limit** → 504   | Since **June 2024**, raisable **beyond 29 s** for Regional & private **REST** APIs via Service Quotas (trade-off: lower throttle quota). Floor still 50 ms. _(soft default now)_ |
| **"V1/V2" versioning**  | Shown as an API Gateway property | API Gateway has **deployments (IDs)** + **stages (names)**; versioning is a **backend** (e.g. Lambda alias) concept                                                              |

---

## Exam Traps (🎯 trigger keywords)

| If you see…                                                     | Think…                                                                  |
| --------------------------------------------------------------- | ----------------------------------------------------------------------- |
| "cache per…"                                                    | **Per stage** (default TTL 300 s, max 3600, 0.5 GB–237 GB, encryptable) |
| "validate a Cognito token"                                      | **Cognito user pools** — native validation                              |
| "custom / bearer token / external ID provider"                  | **Lambda authorizer** (returns IAM policy + principal)                  |
| "credentials in headers"                                        | **IAM** auth                                                            |
| "nearest CloudFront PoP"                                        | **Edge-optimized** endpoint                                             |
| "only inside a VPC / interface endpoint"                        | **Private** endpoint                                                    |
| "same region, low overhead"                                     | **Regional** endpoint                                                   |
| "gradually shift % of traffic to new version"                   | **Canary** deployment on a stage                                        |
| "redeploy while canary enabled"                                 | Lands on **canary, not base**                                           |
| **429**                                                         | **Throttling**                                                          |
| **502**                                                         | Backend returned **bad/invalid output**                                 |
| **503**                                                         | Backend **offline** / service unavailable                               |
| **504**                                                         | **Integration timeout** (29-s default)                                  |
| **403**                                                         | **Access denied** (authorizer or WAF)                                   |
| "write to DynamoDB / start Step Function / SNS with no compute" | **Direct AWS service integration**                                      |
| "consistent front end during migration"                         | API Gateway as a **migration façade**                                   |

---

## Quick reference

| Dimension               | Values                                                                                         |
| ----------------------- | ---------------------------------------------------------------------------------------------- |
| **API types**           | HTTP, REST, WebSocket                                                                          |
| **Endpoint types**      | Edge-optimized, Regional, Private                                                              |
| **Auth methods**        | Open, Cognito user pools, Lambda authorizer, IAM                                               |
| **Integrations**        | Lambda, HTTP endpoints (AWS/on-prem), direct AWS services (DDB, SNS, Step Functions, …)        |
| **Phases**              | Request (authorize/validate/transform) → Integration → Response (transform/prepare/return)     |
| **Cache**               | Per stage · TTL 300 s default (0–3600) · 0.5–237 GB · encryptable · REST only · GET by default |
| **Integration timeout** | 29 s default → 504 (raisable beyond 29 s on Regional/private REST since 2024)                  |
| **Monitoring**          | CloudWatch logs + metrics (client & integration sides)                                         |

---

## Open questions for Roger

1. **Cross-linking** — which of these already exist as vault notes to wikilink, and which to create as stubs? Candidates: [[Cognito]] (User Pools vs Identity Pools), [[Lambda]], [[Step Functions]], [[SNS]], [[DynamoDB]], [[CloudFront]], [[WAF]], [[CloudWatch]], [[VPC Interface Endpoints]], [[S3 Static Website Hosting]].
2. **Sub-note candidates flagged across sessions** that this lesson touches — do you want standalone notes for any? **API Gateway (standalone/deep)**, **Cognito User Pools vs Identity Pools**, **Lambda authorizers**, **S3 Event Notifications** (unrelated but on the list).
3. **Depth on canary** — I folded in the full workflow we discussed (sticky state, promotion, stage variables, separate logs). Want that kept here, or split into its own **API Gateway — Canary Deployments** sub-note with just a pointer from here?
4. **Throttling** — the lesson mentions it only in passing (→ 429). Want a short throttling section added (account-level 10,000 RPS steady-state / 5,000 concurrent defaults), or is that developer-stream depth you'd rather skip for SAA?

---

_Source: Adrian Cantrill SAA-C03 — "API Gateway 101" (SHAREDALL). Diagrams © learn.cantrill.io. Fact-checks against current AWS documentation as of 2026-07 cited inline._