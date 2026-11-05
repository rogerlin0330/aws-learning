---
title: API Gateway
type: service
tags: [api-gateway, serverless, application-integration, rest, http, websocket, caching, authentication, migration]
certs: [aws-saa]
updated: 2026-11-04
---

# API Gateway

## Overview

API Gateway is a managed service that lets you **create, publish, and manage APIs**. An API (application programming interface) is how applications talk to each other — the Netflix app on your TV uses an API to talk to Netflix's backend.

Architecturally, API Gateway is an **entry point** that sits *between* the clients consuming an API and the **integrations** — the backend services that actually provide the API's functionality. Everything else on this page is a consequence of that position in the request path.

Key characteristics:

- **Highly available and scalable** — delivered as a managed service, so you manage neither.
- Handles **authorization** (who can access your APIs) and **throttling** (how often individuals can call them).
- Performs **caching**, reducing how often backend integrations are called.
- Supports **CORS**, controlling the security of cross-domain calls within browsers.
- Supports **transformations**, reshaping requests and responses between client and integration.
- Supports the **OpenAPI spec**, so API definition files can be imported and exported easily.
- **Integrates directly with AWS services** — you can write to DynamoDB, start a [Step Functions](step-functions.md) execution, or publish to an [SNS](sns.md) topic *without any backing compute*.
- Is a **public service** — it can front services running in AWS **or** on-premises.
- Provides **HTTP**, **REST**, or **WebSocket** APIs.

It is an excellent **migration tool**: it gives clients a consistent front end while the backend behind it moves from on-premises into AWS, or is re-architected from a monolith into [serverless](../patterns/serverless-architecture.md) components. In the [global and regional architecture model](../patterns/global-regional-architecture.md) it is the **regional entry point** for API-shaped workloads, the same role an [Application Load Balancer](elb.md) plays for ordinary HTTP applications, and it is the standard front door in front of [Lambda](lambda.md).

## Architecture

![API Gateway overview diagram: on the left three clients — mobile applications, another API, and a laptop running a JavaScript web app downloaded from an S3 static hosting bucket — all send arrows into the API Endpoint DNS name of a large red-bordered API Gateway box in the centre; the box is split into a green upper band labelled 1 Request reading AUTHORIZE, VALIDATE, TRANSFORM with a green arrow pointing right, and an orange lower band labelled 3 Response reading TRANSFORM, PREPARE, RETURN with an orange arrow pointing left; on the right a dashed box labelled 2 Integrations lists DDB, SNS, Step Functions, HTTP Endpoints, and Lambda; below the gateway sit an API Cache box joined by a double-headed arrow and a CloudWatch icon receiving a one-way arrow, with captions saying the cache reduces backend calls and improves client performance, CloudWatch stores full stage request and response logs plus metrics for client and integration sides, and that API Gateway is an intermediary between clients and integrations](../assets/api-gateway-overview.png)

Read it left to right: **clients**, **API Gateway**, **integrations**, with the cache and [CloudWatch](cloudwatch.md) hanging underneath.

**Clients (left).** Three example consumers — mobile applications, other APIs, and web applications served from [S3 static hosting](s3.md). They all reach the API through the **API endpoint DNS name**.

**API Gateway (centre).** The box is split into two bands showing the request and response halves of an interaction.

**Integrations (right).** The backends that provide the functionality: DynamoDB, [SNS](sns.md), [Step Functions](step-functions.md), HTTP endpoints (in AWS or on-premises), and [Lambda](lambda.md).

**Footer services.** The **API cache** reduces the number of calls made to backend integrations and improves client performance. **CloudWatch** stores full stage request and response **logs**, plus **metrics** for both the client side and the integration side.

### The three interaction phases

Most API Gateway interactions have three phases:

1. **Request** — the client makes a request that flows through API Gateway to the integration. API Gateway **authorizes**, **validates**, and **transforms** it into a form the integration can handle.
2. **Integration** — the backend service does the work.
3. **Response** — API Gateway takes the integration's output, **transforms** and **prepares** it, and **returns** it to the client.

This is the whole mental model: API Gateway is an intermediary, and each of the features below attaches to one of these three phases.

## REST API object relationships

A REST API organizes **Resources** (paths such as `/orders`) and their **Methods** (such as `GET` and `POST`). A **Stage** is a named reference to a deployment snapshot of that API; it selects which deployed configuration clients reach. A resource path is only one part of the URL: the default invoke URL is `https://{apiId}.execute-api.{region}.amazonaws.com/{stage}/{resourcePath}`. See [stages and deployments](#stages-and-deployments) and [AWS method request setup](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-method-settings-method-request.html).

Both diagrams begin with the general `METHOD https://{api_id}.execute-api.us-east-1.amazonaws.com/{stage}/{resource}` request format, a concrete example, and the component mapping. Both diagrams use the same color scheme: **blue** for API configuration and routing, **yellow** for authorization and usage controls, **green** for backend integration, **red** for rejected requests, and **orange** for the client. Labels and connector shapes carry the meaning independently of color.

![English UML configuration diagram in Excalidraw hand-drawn style: API composes Resources and Stages; Resource composes Methods; Method owns client-facing Method Request settings and a separate Integration; Stages reference Deployments; Usage Plans associate stages and API keys; Method Request references a reusable Authorizer. All links describe configuration, not request execution](../assets/api-gateway-request-model.png)

[Editable UML source](../assets/api-gateway-request-model.excalidraw). This diagram shows **configuration relationships only**: composition for ownership, shared aggregation for usage-plan selection of existing stages, and associations/dependencies for references and use. In particular, **Method → Integration is an ownership relationship, not a request path that bypasses Method Request checks**. Method Request is the client-facing configuration aspect of Method; Integration contains its backend-facing configuration. Runtime execution is illustrated separately below. Response configuration and canary deployments are omitted for clarity.

### Reading the URL: API versus stage versus resource

With the default REST API endpoint, the **API corresponds to the hostname level**. The general request format is:

```text
METHOD https://{api_id}.execute-api.us-east-1.amazonaws.com/{stage}/{resource}
```

`METHOD` is the HTTP verb, written before the URL rather than as part of it. `{resource}` can include multiple path segments. For example:

```text
GET https://abc123.execute-api.us-east-1.amazonaws.com/prod/orders
```

| URL/request component | API Gateway concept |
| --- | --- |
| `abc123.execute-api.us-east-1.amazonaws.com` | Default API hostname; `abc123` identifies the API |
| `/prod` | Stage selecting a deployment of that API |
| `/orders` | Resource path within that deployed API |
| `GET` | Method on the resource; carried in the HTTP request, not in the URL |

This makes **API → hostname, Stage → deployment path prefix, Resource → resource path, Method → HTTP verb** a useful reading rule for the default URL. The API is still the configuration container, not a DNS record or an HTTP route alone.

**Custom domains loosen this one-API/one-hostname relationship.** An API mapping associates a custom domain and optional path prefix with an **API + stage**. For example, `api.example.com/shop` can map to API A's `prod` stage, while `api.example.com/billing` maps to API B's `prod` stage. A request to `/shop/orders` then reaches API A's `/orders` resource after the mapping prefix is removed; the stage name need not appear in the external URL. A mapping prefix is therefore not necessarily a Resource path. See [AWS custom-domain API mappings](https://docs.aws.amazon.com/apigateway/latest/developerguide/rest-api-mappings.html).

### Configuration relationships

| Concept | Responsibility | Relationship |
| --- | --- | --- |
| API | Defines the overall API | Contains resources, stages, deployments, and API-scoped authorizer definitions |
| Resource | Identifies a path, for example `/orders` | Belongs to one API; can have child resources and multiple methods |
| Method | Defines a client-facing operation, for example `GET /orders` | Belongs to a resource; has request configuration and an integration when configured for invocation |
| Method Request | Defines authorization, API key requirement, request parameters, body models, and optional validation | A configuration aspect of the Method, **not a separate AWS API resource** |
| Authorizer | Validates identity and determines API access | A method can reference a Lambda or Cognito authorizer; multiple methods can reuse it. `NONE` and `AWS_IAM` do not reference an Authorizer object |
| Integration Request | Defines the backend target, integration type, and request construction/mapping | Represented by the Method's Integration object |
| Integration Method | Chooses the HTTP verb used to call the backend | The Integration object's `httpMethod` property, independent of the client's Method |
| Stage | Provides a named deployment entry point and stage-specific settings | References a deployment in the same API; several stages can reference the same deployment |
| Usage Plan | Defines per-API-key throttling and optional periodic quotas | Associates selected API stages, potentially across multiple APIs, and API keys; may define method-specific throttles |

### Runtime request flow

For example, a client sends **`GET /prod/orders`**. Stage `prod` selects a deployment containing resource `/orders` and method `GET`. The method's request configuration specifies the applicable authorization and validation requirements. If API keys are required, a key associated with a usage plan covering this API stage enables per-client metering and throttling. **The request must pass the applicable checks before the integration constructs and sends the backend request.** A [Lambda](lambda.md) integration uses **`POST`** to invoke the Lambda service, even though the client's method is `GET`. Lambda proxy integration constructs the event automatically and preserves the client's method in `event.httpMethod`; custom integration supports explicit mappings. See [AWS Method reference](https://docs.aws.amazon.com/apigateway/latest/api/API_Method.html) and [AWS Integration reference](https://docs.aws.amazon.com/apigateway/latest/api/API_Integration.html).

![English request-flow diagram in Excalidraw hand-drawn style: client GET request resolves API, Stage, Resource, and Method; configured Method Request checks and admission controls must accept the request before Integration Request constructs a POST invocation to the backend Lambda; failed checks reject the request without invoking the integration Lambda](../assets/api-gateway-request-flow.png)

[Editable request-flow source](../assets/api-gateway-request-flow.excalidraw). This is an **uncached Lambda-backed example**, not the UML configuration graph. Authorization, usage controls, and validation are grouped without prescribing their internal evaluation order. Usage-plan throttling is an admission control associated with API keys and API stages, not a separately invoked backend. Rejected requests do not invoke the integration Lambda; a custom authorizer Lambda may already have run as part of authorization. Responses and cache hits are omitted.

> [!warning] Gotcha — separate permission and throttling boundaries
> A Lambda authorizer's policy allows or denies **`execute-api:Invoke`** on API methods; it does not give the caller direct **`lambda:InvokeFunction`** permission. API Gateway needs separately configured permission to invoke the backend. A [Cognito](cognito.md) user pool authorizer validates tokens natively without an additional authorizer Lambda. API keys identify usage consumers and are not a substitute for authentication. A key can be linked to multiple plans overall, but only one plan for a given API stage. Usage-plan throttles and quotas are best-effort targets; account and stage/method limits also apply. See [AWS authorizer output](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-lambda-authorizer-output.html), [Lambda invocation permissions](https://docs.aws.amazon.com/lambda/latest/dg/services-apigateway.html#services-apigateway-permissions), and [AWS usage plans](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-api-usage-plans.html).

## Authentication

API Gateway supports a range of authentication methods, including **fully open access** with no authentication at all. Three paths are worth knowing:

1. **[Cognito user pools](cognito.md)** — the client authenticates with Cognito and receives a **Cognito token**, then passes that token with each request. Because of API Gateway's tight integration with Cognito, it can **natively validate** the token with no code of yours involved.
2. **Lambda authorizer** (formerly "custom authorization") — used when API Gateway cannot natively validate the credential. A [Lambda function](lambda.md) inspects the credential and returns a verdict.
3. **[IAM](iam.md)** — authenticate with **SigV4-signed requests**, including a session token when using temporary credentials; the secret access key itself is not transmitted.

For **REST APIs**, the native Cognito authorizer type is `COGNITO_USER_POOLS`; methods can use ID tokens for identity checks or access tokens with OAuth scopes. For **HTTP APIs**, configure a JWT authorizer with the user pool issuer and audience, and require scopes when appropriate. An identity pool is unnecessary for these token-authorizer paths; it is useful when the client instead needs temporary credentials for IAM/SigV4 authentication. See [Cognito architecture choices](cognito.md#choosing-an-architecture), [AWS REST authorizers](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-integrate-with-cognito.html), and [AWS HTTP JWT authorizers](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-jwt-authorizer.html).

> [!note] Depth beyond the exam
> Cantrill notes that IAM authentication "is beyond what's required for the exam." Know that it exists and uses SigV4-signed request headers, but focus on **Cognito user pools** and **Lambda authorizers**.

![API Gateway authentication diagram showing two overlaid auth paths: on the left, a pink dotted arrow from the client laptop at top centre to a Cognito icon labelled authenticate with Cognito and receive token, a solid pink arrow down from the client to the API Gateway icon labelled pass token with request, and a pink double-headed arrow between API Gateway and Cognito labelled verify token validity; on the right, orange arrows show the client calling API Gateway with a bearer token (ID), API Gateway calling a Lambda Authorizer on the far right (labelled Lambda authorizer called), the authorizer making up-and-down calls to an ID-badge icon labelled call to ID Provider or compute based check of ID, and the authorizer returning an IAM Policy and principal identifier to API Gateway; a green arrow points down from API Gateway to a separate Lambda function, captioned handles request via lambda integration or returns 403 ACCESS_DENIED to client](../assets/api-gateway-authentication.png)

The **Cognito path** is the left half: authenticate with Cognito, receive a token, pass the token with each request, and API Gateway verifies it natively.

The **Lambda authorizer path** is the right half, and it is the flow the diagram spends most of its ink on:

1. The client calls API Gateway with a **bearer token** asserting some identity.
2. API Gateway does not know how to validate that token natively, so it **calls the Lambda authorizer**.
3. The authorizer does the work: it **calls an external identity provider**, or performs a **compute-based check of the identity** against something like a local user store.
4. The authorizer returns an **IAM policy and a principal identifier** to API Gateway.
5. API Gateway **evaluates the policy** and either **handles the request via the Lambda integration**, or returns **`403 ACCESS_DENIED`** to the client.

> [!warning] Gotcha — two different Lambda functions
> The **Lambda authorizer** (validates identity, returns a policy) is a completely separate function from the **integration Lambda** (does the actual application work). Separate functions, separate jobs. The diagram draws them at opposite ends for exactly this reason.

## Endpoint types

Three endpoint types can be configured for an API:

| Endpoint type | Behavior | When to use |
| --- | --- | --- |
| **Edge-optimized** | Incoming requests are routed to the **nearest CloudFront point of presence** | Geographically distributed clients |
| **Regional** | A regional endpoint with **no CloudFront network** in front, so lower overhead | Clients in the **same region** |
| **Private** | Only accessible **inside a VPC through an interface endpoint** | Completely **private** APIs |

The private endpoint type is the same [interface VPC endpoint / PrivateLink](vpc.md) mechanism used by other private-service access patterns. Access to a private API is additionally governed by an [API Gateway resource policy](../concepts/resource-vs-identity-policies.md) — [a private REST API requires one to grant access](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-private-apis.html).

## Stages and deployments

When you deploy an API configuration in API Gateway, you deploy it **to a stage** (for example `prod` and `dev`). **Most things in API Gateway are configured per stage.**

- Each stage has its **own unique endpoint URL** and its **own settings**.
- Each stage is deployed onto **individually** — version 1 of the config in `prod` while version 2 sits in `dev`.
- Deployments on a stage can be **rolled back**, which gives good isolation between testing and production.

> [!note] Stage names are arbitrary labels, not enforced environments
> `prod` and `dev` are convention. The name is just a string that becomes part of the invoke URL path (`.../{stage}/...`), so you could equally name stages by version, team, tenant, or feature. Nothing about the name "prod" makes API Gateway treat it as production — behavior comes from the deployment attached to it and from the stage's own settings and variables. Source: [AWS — Setting up a stage](https://docs.aws.amazon.com/apigateway/latest/developerguide/set-up-stages.html).

**Deployments versus stages** are two distinct concepts that the "V1/V2" labels on the diagram below blur together:

- A **deployment** is an **immutable snapshot** of your API's resources, methods, integrations, and settings, each with a **unique deployment ID**. Changes you make do not take effect until you create a *new* deployment.
- A **stage** is a **named pointer** to one deployment.

> [!warning] Gotcha — "V1/V2" is a backend concept, not an API Gateway one
> API Gateway itself has no numbered "versions"; it has deployments (unique IDs) and stages (names). The versioning in the diagram lives in the **backend** — typically [Lambda function versions and aliases](lambda.md#versions--aliases). The diagram collapses the API Gateway layer and the backend layer into "V1 with LambdaV1, V2 with LambdaV2" for legibility. The backend need not be Lambda either; it could be an HTTP endpoint or another AWS service, so "V1/V2" generically means old backend behavior versus new.

### Canary deployments

Stages can be **enabled for canary deployments**. Once enabled:

- Any **new deployment to that stage** lands on the **canary sub-part** of the stage, **not the base stage**.
- The **traffic split** between the base stage and the canary is a **user-configurable percentage**.
- The canary can eventually be **promoted** to become the new base stage, and the cycle repeats.
- If the canary is unhealthy (bugs, worse performance), you can **remove it and revert** to the base stage.

**Worked example.** Version 2 is tested by the dev team, then canary is enabled on `prod`. Version 2 is deployed to `prod`, so it lands in the **canary** rather than the base. You adjust the traffic split between base and canary until you are satisfied, then **promote** the canary to become the full base stage, and repeat next time.

![API Gateway stages diagram: two stages of the same API stacked vertically, api.catagram.io/prod at the top receiving a solid blue arrow from a Customers laptop icon and api.catagram.io/dev at the bottom receiving a solid pink arrow from a Developers icon; a green arrow at top left carries the caption that APIs are deployed to stages and each stage has one deployment; on the right, a V1 gateway icon feeds LambdaV1 and sends a solid blue arrow into the prod stage, while a V2 gateway icon feeds LambdaV2 and sends a dotted pink arrow into the prod stage and a solid pink arrow into the dev stage; captions state that stages enabled for canary deployments receive deployments to the canary not the stage, that a configurable percentage of traffic is sent to the canary and can be adjusted over time, and that the canary can be promoted to become the new base stage](../assets/api-gateway-stages.png)

The solid **blue** arrow into `prod` is the base deployment carrying the bulk of production traffic; the dotted **pink** arrow into `prod` is the canary taking a configurable percentage. Note the distinct `/prod` and `/dev` path suffixes: each stage really is a separate endpoint URL.

> [!note] Canary mechanics beyond the lecture
>
> - **Canary releases are supported only by API Gateway REST APIs** — not by API Gateway HTTP APIs or WebSocket APIs. REST APIs and HTTP APIs are both RESTful API products; HTTP APIs offer a smaller feature set at a lower price. Sources: [AWS — Canary release deployments](https://docs.aws.amazon.com/apigateway/latest/developerguide/canary-release.html), [REST API versus HTTP API](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-vs-rest.html).
> - **You cannot create a canary on a stage that does not exist.** A normal deploy *creates* the stage if the name is new; a canary deploy **errors** instead. Source: [AWS — Create a canary release deployment](https://docs.aws.amazon.com/apigateway/latest/developerguide/create-canary-deployment.html).
> - Canary state is **sticky**: once enabled, the **base freezes** at whatever deployment it had, and **every subsequent deploy to that stage lands on the canary** until you promote it or remove the canary settings.
> - The canary holds **one deployment slot**, overwritten (not stacked) by each new deploy.
> - **Promoting** makes the canary's deployment the new base and disables the canary. **Removing the settings** rolls back to base and discards the canary.
> - The traffic split is **random per request**, and the percentage is adjustable at any time.
> - **Canary stage variables** can override the base stage's — the common way to point the canary at LambdaV2 while base stays on LambdaV1.
> - The canary gets its **own CloudWatch metrics and log group** (the base log group `API-Gateway-Execution-Logs/{rest-api-id}/{stage-name}` plus a `.../Canary`-suffixed one), which is how you validate before promoting.

> [!tip] Exam Tip — the canary redeploy trap
> With canary enabled, a redeploy updates the **canary, not the live base version**. Candidates often assume a redeploy immediately updates production. It does not, while a canary is active.

## Caching

Caching is **configured per stage**, which matters both for the exam and in production.

**Without a cache**, every request hits the backend integrations. **With a cache**, backends are called **only on a cache miss**, which reduces load, reduces cost, and improves performance through lower latency.

| Property | Value |
| --- | --- |
| Cache size | **500 MB (0.5 GB) to 237 GB** |
| Default TTL | **300 seconds** |
| TTL range | **0 (disabled) to 3,600 seconds** |
| Encryption | **Can be encrypted** |
| Scope | **Per stage** |

![API Gateway caching diagram: a client laptop on the left joined to a central API Gateway STAGE icon by a solid pink double-headed arrow, with an orange dashed double-headed arrow running right from the stage to a dashed box of integrations listing DDB, SNS, Step Functions, HTTP Endpoints, and Lambda; three boxes labelled PROD, STAGE, and DEV sit under that arrow beside the caption that cache is defined per stage within API Gateway; a green double-headed arrow runs down from the stage to an API Cache box; a bullseye and sad-face icon on the left carries the caption that calls are only made to backend integrations if the request is a cache miss; top-left facts state cache TTL default 300 seconds, configurable min 0 and max 3600 seconds, can be encrypted, cache size 500MB to 237GB; top-right states reduced load and cost, improved performance](../assets/api-gateway-caching.png)

The three boxes labelled PROD, STAGE, and DEV in the middle of the diagram exist purely to make the per-stage point: the cache belongs to a stage, not to the API.

> [!note] Additional caching facts, verified against current AWS docs
>
> - **Caching is a REST API feature only**, not available on HTTP or WebSocket APIs — the documentation page is literally titled "Cache settings for REST APIs". Source: [AWS — Cache settings for REST APIs](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-caching.html).
> - By default only **`GET` methods** are cached; other methods can be enabled through **method-level** settings, which **override** stage settings.
> - The maximum cacheable **response size is 1,048,576 bytes (1 MB)**.
> - A client can request a cache bypass with the header **`Cache-Control: max-age=0`**. You can require `execute-api:InvalidateCache` authorization for this, otherwise **any client can invalidate your cache**.
> - Enabling, resizing, or deleting a cache provisions or replaces a dedicated cache instance and takes **about 4 minutes**; resizing **deletes all existing cached data**.

## Error codes

Error codes fall into two categories: **4xx** are client errors (something wrong on the client side, or in how it is making the request) and **5xx** are server errors (a valid request, but a backend problem).

| Code | Meaning | Notes |
| --- | --- | --- |
| **400** | Generic **client-side** error | Hard to diagnose — many possible root causes |
| **403** | **Access denied** | The authorizer denied the request, **or** something like **WAF** filtered it |
| **429** | **Throttling** | A configured throttle amount was exceeded |
| **502** | **Bad Gateway** | Backend returned **invalid output**, for example a Lambda returning something malformed |
| **503** | **Service unavailable** | Backing endpoint offline or having major service issues |
| **504** | **Integration failure or timeout** | Backend did not respond within the integration timeout |

> [!tip] Exam Tip — memorize the mapping
> **429** is throttling, **502** is bad backend output, **503** is backend offline, **504** is timeout, **403** is access denied.

### The 29-second integration timeout

The lecture states a **29-second limit** for any request through API Gateway: even though [Lambda can run for up to 15 minutes](lambda.md#timeout), if a Lambda backs an API and the request takes longer than 29 seconds, the client gets a **504**. So back your APIs with functions that respond well inside 29 seconds.

> [!warning] Course-vs-reality — 29 seconds is now a soft default
> Since **June 2024**, API Gateway allows the integration timeout to be raised **beyond 29 seconds** for **Regional REST APIs and private REST APIs**, via a Service Quotas increase on "Maximum integration timeout in milliseconds" — AWS's wording is that ["you can raise the integration timeout to greater than 29 seconds for Regional REST APIs and private REST APIs, but this might require a reduction in your account-level throttle quota limit"](https://aws.amazon.com/about-aws/whats-new/2024/06/amazon-api-gateway-integration-timeout-limit-29-seconds/). The change was driven partly by generative-AI workloads that need longer than 30 seconds. Edge-optimized REST APIs, HTTP APIs, and WebSocket APIs are not covered. The floor is still **50 ms**. For the exam, the 29-second default producing a **504** is still the expected answer.

## Throttling

Throttling is what produces a **429**. API Gateway applies limits at several levels, and the account-level default is the one worth remembering: **10,000 requests per second** per region across all APIs in the account, with a **burst of 5,000 concurrent requests**. Source: [AWS — API Gateway quotas](https://docs.aws.amazon.com/apigateway/latest/developerguide/limits.html). Below that you can set per-stage and per-method throttles, and **usage plans** tied to **API keys** throttle individual consumers — the standard way to give different customers different rate limits on the same API.

## Course-vs-current-AWS discrepancies

| Topic | Course framing | Current AWS reality |
| --- | --- | --- |
| **Integration timeout** | 29-second **hard limit**, producing a 504 | Since **June 2024** it can be raised beyond 29 seconds for Regional and private **REST** APIs via Service Quotas, at the cost of a possible throttle-quota reduction. Floor still 50 ms. It is a soft default now |
| **"V1/V2" versioning** | Drawn as an API Gateway property | API Gateway has **deployments (IDs)** and **stages (names)**; versioning is a **backend** concept, for example a Lambda alias |

## Exam Traps

> [!tip] API Gateway exam triggers
>
> | If you see… | Think… |
> | --- | --- |
> | "cache per…" | **Per stage** (default TTL 300 s, max 3,600 s, 0.5 GB to 237 GB, encryptable) |
> | "validate a Cognito token" | **Cognito user pools** — native validation |
> | "custom / bearer token / external identity provider" | **Lambda authorizer** (returns an IAM policy plus a principal) |
> | "credentials in headers" | **IAM** authentication |
> | "nearest CloudFront point of presence" | **Edge-optimized** endpoint |
> | "only inside a VPC / interface endpoint" | **Private** endpoint |
> | "same region, low overhead" | **Regional** endpoint |
> | "gradually shift a percentage of traffic to a new version" | **Canary** deployment on a stage |
> | "redeploy while canary is enabled" | Lands on the **canary, not the base** |
> | **429** | **Throttling** |
> | **502** | Backend returned **bad or invalid output** |
> | **503** | Backend **offline** / service unavailable |
> | **504** | **Integration timeout** (29-second default) |
> | **403** | **Access denied** (authorizer or WAF) |
> | "write to DynamoDB / start a Step Functions execution / publish to SNS with no compute" | **Direct AWS service integration** |
> | "consistent front end during a migration" | API Gateway as a **migration façade** |

## Quick Reference

| Dimension | Values |
| --- | --- |
| API types | HTTP, REST, WebSocket |
| Endpoint types | Edge-optimized, Regional, Private |
| Auth methods | Open, Cognito user pools, Lambda authorizer, IAM |
| Integrations | Lambda, HTTP endpoints (AWS or on-premises), direct AWS services (DynamoDB, SNS, Step Functions, and others) |
| Phases | Request (authorize, validate, transform), Integration, Response (transform, prepare, return) |
| Cache | Per stage · TTL 300 s default (0 to 3,600) · 0.5 to 237 GB · encryptable · REST only · `GET` by default |
| Integration timeout | 29 s default producing a 504 (raisable beyond 29 s on Regional and private REST APIs since 2024) |
| Account throttle default | 10,000 RPS per region, 5,000 burst |
| Monitoring | CloudWatch logs and metrics, on both the client and integration sides |

## References

- [Lambda](lambda.md) — the most common integration, and the home of both authorizer and backend functions
- [Step Functions](step-functions.md), [SNS](sns.md) — direct service integrations reachable with no backing compute
- [Serverless Architecture](../patterns/serverless-architecture.md) — where API Gateway sits as the client-to-Lambda front door
- [Global & Regional AWS Architecture](../patterns/global-regional-architecture.md) — API Gateway as the regional entry point / web tier
- [TLS & HTTPS](../concepts/tls-https.md) — API Gateway terminates TLS; custom domain names need an ACM certificate
- [AWS — Setting up a stage](https://docs.aws.amazon.com/apigateway/latest/developerguide/set-up-stages.html)
- [AWS — Method API reference](https://docs.aws.amazon.com/apigateway/latest/api/API_Method.html)
- [AWS — Integration API reference](https://docs.aws.amazon.com/apigateway/latest/api/API_Integration.html)
- [AWS — Usage plans and API keys](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-api-usage-plans.html)
- [AWS — Custom-domain API mappings](https://docs.aws.amazon.com/apigateway/latest/developerguide/rest-api-mappings.html)
- [AWS — Canary release deployments](https://docs.aws.amazon.com/apigateway/latest/developerguide/canary-release.html)
- [AWS — Cache settings for REST APIs](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-caching.html)
- [AWS — API Gateway quotas](https://docs.aws.amazon.com/apigateway/latest/developerguide/limits.html)
- [AWS What's New — Integration timeout increase (June 2024)](https://aws.amazon.com/about-aws/whats-new/2024/06/amazon-api-gateway-integration-timeout-limit-29-seconds/)
- Adrian Cantrill, AWS SAA course — API Gateway 101
- Source: `raw/aws-cert-study-notes/aws-solution-architect/api_gateway_101.md`
