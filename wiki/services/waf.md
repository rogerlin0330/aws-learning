---
title: AWS WAF
type: service
tags: [waf, web-application-firewall, layer-7, firewall, web-acl, protection-pack, rule-groups, managed-rules, rate-based-rules, wcu, captcha, challenge, labels, bot-control, fraud-control, logging, security, pricing]
certs: [aws-saa]
updated: 2026-11-13
---

# AWS WAF

AWS WAF (Web Application Firewall) is AWS's managed **Layer 7 firewall**. It understands HTTP and HTTPS, so it can allow or block a web request based on what the request contains: its URI path, query string, headers, cookies, body, HTTP method, source IP, or country. The general idea of an application-layer firewall, and why Layer 3/4 firewalls cannot do this, is covered in [Networking: Layer 7 Firewalls](../concepts/networking.md#layer-7-application-firewalls).

WAF does not sit in your network as a box you route traffic to. It is attached to a service that already handles HTTP requests, such as [CloudFront](cloudfront.md), an [Application Load Balancer](elb.md), or [API Gateway](api-gateway.md). That service terminates HTTPS, hands each request to WAF for inspection, and forwards or rejects it based on WAF's decision ([AWS WAF: How AWS WAF works](https://docs.aws.amazon.com/waf/latest/developerguide/how-aws-waf-works.html)). This is why WAF can read the content of encrypted traffic without managing any certificates itself.

For the SAA exam, the deciding words are **SQL injection**, **cross-site scripting (XSS)**, **HTTP floods**, **rate limiting by IP**, **block by country**, and **bad bots**. Those all point to WAF.

## What WAF Can Protect

WAF protects resources that receive HTTP requests. The unit you attach is a **Web ACL**, and it is created for one of two scopes ([AWS WAF: resources you can protect](https://docs.aws.amazon.com/waf/latest/developerguide/how-aws-waf-works-resources.html)):

| Scope                   | Resources                                                                                                                                                        | Where the Web ACL is created     |
| ----------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------- |
| **Global (CloudFront)** | CloudFront distributions, and AWS Amplify apps                                                                                                                   | US East (N. Virginia), us-east-1 |
| **Regional**            | Application Load Balancer, API Gateway REST API, AppSync GraphQL API, Cognito user pool, App Runner service, Verified Access instance, Bedrock AgentCore Gateway | The same Region as the resource  |

Rules for association:

- A resource can have **one** Web ACL. One Web ACL can be associated with **many** resources of the same scope.
- A CloudFront Web ACL cannot be associated with a regional resource, and a regional Web ACL cannot be associated with CloudFront.
- ALBs on AWS Outposts cannot use WAF.

The lecture lists CloudFront, ALB, AppSync, and API Gateway. The newer recording of the same lesson adds that "many other services" support WAF, which is the list above. See [Resource Scope](../concepts/resource-scope.md) for how global and regional resources differ in general.

> [!warning] Gotcha: WAF cannot protect an NLB or an EC2 instance directly
> WAF needs a service that terminates HTTP. A [Network Load Balancer](elb.md) works at Layer 4 and passes TLS through, and an EC2 instance with a public IP has no WAF integration. To protect either with WAF, put an ALB or CloudFront in front of it.

> [!tip] Exam Tip
> For WAF on CloudFront, the Web ACL lives in **us-east-1**, the same rule as the [ACM](acm.md) viewer certificate for CloudFront. For an ALB or API Gateway, the Web ACL is created in the resource's own Region.

In the WAF console, Web ACLs are now shown as **protection packs**. This is only a console name. The API, CLI, and CloudFormation still use Web ACLs, and nothing about how they work changed ([AWS WAF: How AWS WAF works](https://docs.aws.amazon.com/waf/latest/developerguide/how-aws-waf-works.html)).

## Architecture

![AWS WAF architecture: a valid user, a bot, and an attacker send requests toward CloudFront, ALB, AppSync, and API Gateway; the WAF Web ACL in front allows the valid user and blocks the bot and attacker; the Web ACL contains rule groups (AWS managed rules, allow list, deny list, SQL injection, XSS, HTTP flood, IP reputation, bots); the Web ACL is updated manually by a person, by an EventBridge scheduled IP list parser Lambda, and by a Lambda that processes WAF logs delivered to S3 every 5 minutes, CloudWatch Logs, or Data Firehose into S3, forming a feedback loop](../assets/waf-architecture.png)

The diagram shows a fairly complex setup. Read it in three parts.

**1. Traffic.** A valid user, a bot, and an attacker all send requests to the protected services. The Web ACL decides each request. The valid user's request reaches CloudFront; the bot's and attacker's requests are blocked.

**2. The Web ACL.** The Web ACL holds the rules. Here it uses AWS managed rules, an allow list, a deny list, SQL injection and XSS rules, an HTTP flood (rate-based) rule, an IP reputation rule, and bot rules. The slide's footer sums up the structure: rules sit inside rule groups, and rule groups sit inside the Web ACL.

**3. Keeping the rules current.** The Web ACL can be updated in three ways:

1. **Manually**, by a person responding to a security event.
2. **On a schedule.** An [EventBridge](eventbridge.md) scheduled rule runs a [Lambda](lambda.md) function that downloads public lists of known-bad IP addresses and writes them into the deny list.
3. **From the logs.** WAF writes a log record for each request it inspects. A Lambda function, or a pipeline using S3 event notifications, [Athena](athena.md), and EventBridge, reads those logs, finds patterns such as one IP that keeps getting blocked, and updates the Web ACL.

The third path is the point of the diagram: WAF logs feed back into WAF rules without a human in the loop. AWS published a ready-made version of this design as [Security Automations for AWS WAF](https://aws.amazon.com/solutions/implementations/security-automations-for-aws-waf/).

### Logging Destinations

WAF can send logs to three places ([AWS WAF: logging destinations](https://docs.aws.amazon.com/waf/latest/developerguide/logging-destinations.html)):

| Destination                      | Delivery                                                                            | Naming rule                                                                                                           |
| -------------------------------- | ----------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| [S3](s3.md)                      | Files every **5 minutes**, gzip-compressed                                          | Bucket name must start with `aws-waf-logs-`                                                                           |
| [CloudWatch Logs](cloudwatch.md) | Streamed to a log group                                                             | Log group name must start with `aws-waf-logs-`                                                                        |
| [Data Firehose](kinesis.md)      | Streamed, then delivered to any Firehose destination (S3, OpenSearch, Splunk, HTTP) | Stream name must start with `aws-waf-logs-`; Direct PUT source; same Region as the Web ACL (us-east-1 for CloudFront) |

> [!tip] Exam Tip
> If the question needs to **react quickly** to WAF logs, S3 alone is too slow because files arrive every five minutes. Use CloudWatch Logs or Firehose. For searching WAF logs with dashboards, Firehose into [OpenSearch](opensearch.md) is the common answer; for cheap long-term analysis, S3 plus Athena.

## Web ACLs

The **Web ACL** (web access control list) is the main unit of configuration. It is what decides whether a request is allowed or blocked, and it is the thing you associate with resources.

**Default action.** Every Web ACL has a default action, **Allow** or **Block**, applied to any request that no rule matched. The choice reflects your approach:

| Default action | Approach                                       | Rules you write                                |
| -------------- | ---------------------------------------------- | ---------------------------------------------- |
| **Allow**      | Block known-bad traffic, allow everything else | Block rules for exploits, bad IPs, bad bots    |
| **Block**      | Allow only known-good traffic                  | Allow rules for trusted IPs, paths, or clients |

Most public websites use a default of Allow, because a default of Block requires listing every legitimate request pattern in advance.

**Rules are processed in order.** A Web ACL on its own does nothing. You add rules and rule groups, and WAF evaluates them by priority, top to bottom. You can reorder them.

**Capacity (WCU).** Each rule has a compute cost measured in **Web ACL Capacity Units (WCU)**. Simple rules, such as an IP match, cost little; regular expressions and rules that inspect many parts of a request cost more. A Web ACL includes **1,500 WCU** in its base price and can use up to a hard limit of **5,000 WCU** at extra cost ([AWS WAF: quotas](https://docs.aws.amazon.com/waf/latest/developerguide/limits.html)).

> [!warning] Gotcha: the 1,500 WCU limit is no longer a support ticket
> The original lecture says a Web ACL has a default maximum of 1,500 WCU that can be raised with a support ticket. The newer recording and current AWS docs say 1,500 WCU is included, you can go up to 5,000 WCU without a ticket, and each extra 500 WCU adds to the request price (see [Pricing](#pricing)).

> [!warning] Gotcha: WAF WCU is not DynamoDB WCU
> In WAF, WCU means Web ACL Capacity Units, a measure of rule complexity. In [DynamoDB](dynamodb.md), WCU means write capacity units, a measure of write throughput.

**Association takes time; changes are quicker.** Associating a Web ACL with a resource can take a while, especially for CloudFront, which must update the distribution and push it to every edge location. Changing a Web ACL that is already associated is faster: the newer recording's slide and AWS both say changes take from a few seconds to a few minutes to propagate ([AWS WAF: custom requests and responses](https://docs.aws.amazon.com/waf/latest/developerguide/waf-custom-request-response.html)).

## Rule Groups

A **rule group** is a reusable set of rules. It exists to make rules easier to manage and share.

- **No default action.** A rule group only contains rules. The default action belongs to the Web ACL that uses the group, and it applies to anything not matched by the group's rules or by rules added directly to the Web ACL.
- **Reusable.** One rule group can be referenced by many Web ACLs.
- **Capacity is fixed at creation.** When you create your own rule group you set its WCU capacity, and you cannot change it later. The maximum is **5,000 WCU**. The capacity tells anyone building a Web ACL how much of their budget the group will use.

A rule group is managed (by AWS or a Marketplace vendor), yours, or service-owned:

| Owner                               | Examples                                                                                                  | Cost                                                    |
| ----------------------------------- | --------------------------------------------------------------------------------------------------------- | ------------------------------------------------------- |
| **Managed** by AWS                  | Core rule set (OWASP-style), SQL database, known bad inputs, Amazon IP reputation list, anonymous IP list | Most are free beyond the normal per-rule-group charge   |
| **Managed** by a Marketplace vendor | Vendor rule sets (for example F5, Fortinet, Imperva)                                                      | A subscription set by the seller, on top of WAF charges |
| **Yours**                           | Your own allow lists, deny lists, and custom rules                                                        | Normal WAF charges                                      |
| **Service-owned**                   | Groups created by [AWS Shield Advanced](shield.md) or AWS Firewall Manager                                | Managed by that service                                 |

The AWS managed rule groups that cost extra are **Bot Control**, **Fraud Control account takeover prevention (ATP)**, **Fraud Control account creation fraud prevention (ACFP)**, and the **Anti-DDoS** rule group. The original lecture names only Bot Control and ATP; the newer recording adds ACFP and Anti-DDoS.

## Rules

Every rule has three parts:

1. **Type**: regular or rate-based. This decides how the rule works.
2. **Statement**: what to match.
3. **Action**: what WAF does when the statement matches.

### Regular and Rate-Based Rules

A **regular rule** matches when something is present. A **rate-based rule** matches when something happens too often.

Example from the newer recording: a regular rule allows requests to `/admin` only from the office IP address. A rate-based rule blocks any client that sends more than 5,000 requests to `/login` in five minutes, because that pattern suggests a password brute-force attack.

> [!warning] Gotcha: the original lecture's SSH examples are wrong
> The original recording uses "allow SSH from an IP" and "5,000 SSH connections in five minutes" as examples. WAF only inspects HTTP requests that reach a supported service, so it never sees SSH traffic. SSH is controlled with [security groups and NACLs](vpc.md#security-groups-sg), or with an appliance behind a [Gateway Load Balancer](elb.md#gwlb-vs-waf--complementary-not-substitutes). The newer recording replaced the SSH examples with `/admin` and `/login`.

How a rate-based rule counts ([AWS WAF: rate-based rule statement](https://docs.aws.amazon.com/waf/latest/developerguide/waf-rule-statement-type-rate-based.html)):

- **Aggregation key.** WAF groups requests by a key and applies the limit to each group separately. The default and most common key is the **source IP address**. Other options are an IP address in a header such as `X-Forwarded-For` (useful behind a proxy), ASN, or custom keys such as a header, cookie, query argument, label namespace, URI path, HTTP method, or JA3/JA4 TLS fingerprint, alone or combined. You can also **count all** requests that match a scope-down statement as one group.
- **Scope-down statement.** An optional filter, so only matching requests are counted. "5,000 requests to `/login`" is a rate-based rule with a scope-down statement on the URI path.
- **Window and limit.** The evaluation window is 1, 2, 5, or 10 minutes (default 5). The minimum limit is 10 requests.

### Statements

A statement says what part of the request to look at and how to compare it. Request parts that can be matched:

- Origin country (geo match) and source IP address (IP set)
- Labels added by earlier rules
- Headers, cookies, query parameters, query string, URI path, HTTP method
- The request body

Comparison options include exact match, starts with, ends with, contains, and regular expressions. There are also built-in **SQL injection** and **XSS** detection statements. A rule can have one statement or combine several with **AND**, **OR**, and **NOT**.

WAF does not inspect the whole body of large requests. The original lecture says WAF checks only the first **8,192 bytes** and calls this an exam fact. That is still true for some resources, but not all ([AWS WAF: body inspection](https://docs.aws.amazon.com/waf/latest/developerguide/web-request-body-inspection.html)):

| Resource                                                      | Body inspected                                                       |
| ------------------------------------------------------------- | -------------------------------------------------------------------- |
| ALB, AppSync                                                  | First **8 KB** (fixed)                                               |
| CloudFront, API Gateway, Cognito, App Runner, Verified Access | First **16 KB** by default; can be raised to **64 KB** at extra cost |

Headers and cookies are inspected up to the first 8 KB and the first 200 items. For content beyond the limit, you choose how WAF treats it: continue, treat as a match, or treat as no match.

> [!tip] Exam Tip
> "WAF only inspects the first 8 KB of the body" is the classic exam answer and is still correct for an ALB. If a question involves large uploads through CloudFront or API Gateway, the current limit is 16 KB by default, up to 64 KB.

### Actions

| Action        | What happens                                                        | Stops evaluation?                                                    | Allowed on rate-based rules? |
| ------------- | ------------------------------------------------------------------- | -------------------------------------------------------------------- | ---------------------------- |
| **Allow**     | Request is forwarded to the resource                                | Yes                                                                  | No                           |
| **Block**     | Request is rejected, by default with HTTP 403                       | Yes                                                                  | Yes                          |
| **Count**     | Request is counted and logged; WAF moves on to the next rule        | No                                                                   | Yes                          |
| **CAPTCHA**   | The client must solve a puzzle                                      | No if the client has a valid token; otherwise the request is blocked | Yes                          |
| **Challenge** | A silent background JavaScript check in the browser, with no puzzle | No if the client has a valid token; otherwise the request is blocked | Yes                          |

Source: [AWS WAF: rule actions](https://docs.aws.amazon.com/waf/latest/developerguide/waf-rule-action.html). The original lecture lists Allow, Block, Count, and CAPTCHA; the newer recording adds **Challenge**.

Notes on the actions:

- **Allow is not available on rate-based rules.** A rate-based rule exists to act when a client sends too much. Allowing that client makes no sense.
- **CAPTCHA and Challenge use tokens.** A client that passes receives a token, so it is not asked again on every request until the token expires.
- **Count** is how you test a new rule safely: set it to Count, watch the logs and metrics, then switch it to Block.

**Customizing what the action does:**

- **Block** can return a **custom response**: your own status code, headers, and body instead of the default 403.
- **Allow, Count, CAPTCHA, and Challenge** can add **custom request headers**. WAF prefixes each header name with `x-amzn-waf-`. The protected application can read the header and react, for example by showing a warning or logging the request as suspicious.

### Labels and Multi-Stage Rules

A rule can add a **label** to a request. A later rule in the same Web ACL can match on that label. This lets you build decisions in stages.

Labels only work if evaluation continues after the labeling rule. Allow and Block stop evaluation, so the labeling rule usually uses **Count**. CAPTCHA and Challenge also continue when the client has a valid token.

Example of a two-stage flow:

1. Rule 1, priority 0, action **Count**: if the request comes from outside your operating countries, add the label `custom:foreign`.
2. Rule 2, priority 1, rate-based with action **Block**: count only requests carrying `custom:foreign` and block any IP that sends more than 100 of them in five minutes.

Domestic users get no rate limit from this rule, while foreign traffic gets a tight one. AWS managed rule groups also add labels, so you can, for example, run Bot Control in Count mode and block only the bot categories you care about.

Labels exist only while WAF evaluates the request. They are not sent to the protected application, but they do appear in WAF logs and metrics.

## Pricing

Only the original single-video lecture covers pricing. Its figures are below, checked against the current [AWS WAF pricing page](https://aws.amazon.com/waf/pricing/). As the lecture says, prices change, and WAF prices can vary by Region.

### Base Charges

| Charge                    | Lecture figure    | Current                                                                                      |
| ------------------------- | ----------------- | -------------------------------------------------------------------------------------------- |
| Per **Web ACL**           | $5 per month      | $5 per month (prorated hourly)                                                               |
| Per **rule** in a Web ACL | $1 per month      | $1 per month per rule. Each **rule group** or **managed rule group** also costs $1 per month |
| Per **request** processed | $0.60 per million | $0.60 per million requests, per Web ACL                                                      |

How these add up:

- A Web ACL is charged once even if it is associated with several resources, which is a reason to reuse one. The request charge still grows with every resource that sends traffic through it.
- An AWS managed rule group counts as one $1 charge, no matter how many rules it contains. In AWS's worked examples, your own rule group is charged $1 for the group plus $1 for each rule inside it.

### Extra Charges on Top of the Base

These did not exist when the original lecture was recorded:

| Extra                                                                                                      | Current price                                                              |
| ---------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------- |
| Web ACL using more than 1,500 WCU                                                                          | +$0.20 per million requests for each 500 WCU above 1,500 (up to 5,000 WCU) |
| Body inspection above the default (16 KB on CloudFront, API Gateway, Cognito, App Runner, Verified Access) | +$0.30 per million requests for each extra 16 KB inspected                 |

### Intelligent Threat Mitigation

These optional features have their own fees on top of the base charges:

| Feature                                   | Lecture figure                                | Current                                                                                                                                  |
| ----------------------------------------- | --------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| **Bot Control**, Common level             | $10 per month and $1 per million requests     | $10 per month per Web ACL and $1 per million requests; first 10 million requests per month free                                          |
| **Bot Control**, Targeted level           | Not covered                                   | $10 per month per Web ACL and $10 per million requests; first 1 million requests per month free. Includes Common                         |
| **CAPTCHA**                               | $0.40 per 1,000 challenge attempts            | $0.40 per 1,000 CAPTCHA attempts. Challenge responses are also billed per response; see the pricing page                                 |
| **Fraud Control ATP** (account takeover)  | $10 per month and $1 per 1,000 login attempts | $10 per month per Web ACL; 10,000 requests free, then $1 per 1,000 for the first 2 million, with lower rates per 1,000 at higher volumes |
| **Fraud Control ACFP** (account creation) | Not covered                                   | $10 per month per Web ACL, same tiered request pricing as ATP                                                                            |
| **Anti-DDoS** managed rule group          | Not covered                                   | Its own fee (see the pricing page); requests it blocks are not charged as WAF requests                                                   |
| **Marketplace** rule groups               | Extra cost                                    | The seller's price, on top of WAF charges                                                                                                |

> [!tip] Scope down the expensive rule groups
> Bot Control and Fraud Control are charged per request they inspect. Use a scope-down statement so they only see the requests that matter, for example only `/login` for ATP or only non-static paths for Bot Control. In one of AWS's pricing examples, a Web ACL handles 20 million requests and Bot Control is scoped down to half of them. The 10 million it inspects fall inside the free allowance, so Bot Control costs only the $10 subscription.

### Worked Example

A web store uses one Web ACL on both its CloudFront distribution and its ALB. The Web ACL has 8 custom rules and 2 AWS managed rule groups (the core rule set and the SQL database group). It processes 50 million requests a month.

| Item                                               | Calculation               | Monthly cost    |
| -------------------------------------------------- | ------------------------- | --------------- |
| Web ACL                                            | 1 × $5                    | $5.00           |
| Rules and rule groups                              | (8 + 2) × $1              | $10.00          |
| Requests                                           | 50 × $0.60                | $30.00          |
| **WAF total**                                      |                           | **$45.00**      |
| Add Common Bot Control on all traffic              | $10 + (50 − 10 free) × $1 | +$50.00         |
| Add Bot Control scoped down to 20 million requests | $10 + (20 − 10 free) × $1 | +$20.00 instead |

The base WAF cost is small. The optional threat mitigation features, especially Fraud Control at high volume, are where cost grows. Use the [AWS Pricing Calculator](https://calculator.aws/) for real estimates, as the lecture recommends.

### Other Ways to Pay

- **[AWS Shield Advanced](shield.md)** subscribers get the basic WAF charges for protected resources included in the subscription: Web ACL, rule, and request fees for up to 1,500 WCU and up to 50 billion requests per payer account per month. The saving does not apply to the intelligent threat mitigation features, such as Bot Control and Fraud Control, or to WCU and body inspection above the defaults ([AWS Shield pricing](https://aws.amazon.com/shield/pricing/)).
- **CloudFront flat-rate plans** (November 2025) bundle CloudFront, WAF, DDoS protection, Route 53 DNS, and other services for one monthly price with no overage charges. The tiers are Free ($0), Pro ($15), Business ($200), and Premium ($1,000) per month ([AWS: flat-rate pricing plans](https://aws.amazon.com/about-aws/whats-new/2025/11/aws-flat-rate-pricing-plans); [CloudFront: flat-rate pricing plans](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/flat-rate-pricing-plan.html)). For a small site behind CloudFront, this can replace pay-as-you-go WAF billing.
- **Logging**: WAF includes 500 MB of CloudWatch Logs (vended logs) ingestion for every million WAF requests. Log volume above that is billed by CloudWatch.

## WAF Compared with Other Controls

| Control                                                                                    | Layer                                      | What it sees                                   | Use it for                                                |
| ------------------------------------------------------------------------------------------ | ------------------------------------------ | ---------------------------------------------- | --------------------------------------------------------- |
| [Security group](vpc.md#security-groups-sg)                                                | 3/4, stateful                              | IPs, protocols, ports                          | Allowing only the ports and sources a resource needs      |
| [NACL](vpc.md#network-acl-nacl)                                                            | 3/4, stateless                             | IPs, protocols, ports                          | Subnet-wide IP blocks                                     |
| **AWS WAF**                                                                                | 7 (HTTP)                                   | URI, headers, cookies, body, IP, country       | SQL injection, XSS, rate limiting, bots, geo blocking     |
| [Gateway Load Balancer](elb.md#gwlb-vs-waf--complementary-not-substitutes) plus appliances | 3 (all IP traffic)                         | Whatever the third-party appliance understands | Vendor firewalls, IDS/IPS, non-HTTP and egress inspection |
| [AWS Shield](shield.md)                                                                    | 3/4 (Standard) and 7 (Advanced, using WAF) | Network-level DDoS patterns                    | DDoS protection; Standard is automatic and free           |

These are layers, not alternatives. A typical public web application uses security groups on the instances, WAF on the ALB or CloudFront, and [Shield Standard](shield.md#shield-standard) automatically. AWS Firewall Manager can push the same WAF rules to every account in an [Organization](organizations.md).

> [!tip] Exam Tip
>
> | Requirement | Answer |
> | --- | --- |
> | Block SQL injection or XSS | WAF (managed rule groups or SQLi/XSS statements) |
> | Block requests from specific countries | WAF geo match (or CloudFront geo restriction for whole-distribution blocking) |
> | Limit requests per IP to stop HTTP floods or brute force | WAF rate-based rule |
> | Block a list of bad IPs at Layer 7 | WAF IP set in a deny rule |
> | Block an IP range at the subnet for all protocols | NACL |
> | Protect a service behind an NLB with WAF | Put an ALB or CloudFront in front; WAF cannot attach to an NLB |
> | Apply the same WAF rules across many accounts | AWS Firewall Manager |
> | Third-party firewall appliance or non-HTTP inspection | Gateway Load Balancer |

## References

- Adrian Cantrill, AWS Certified Solutions Architect – Associate (SAA-C03): "Web Application Firewall (WAF), WEBACLs, Rule Groups and Rules" (original single video, including pricing) and its newer two-part recording; "Application Layer (L7) Firewall"
- [AWS WAF: How AWS WAF works](https://docs.aws.amazon.com/waf/latest/developerguide/how-aws-waf-works.html)
- [AWS WAF: resources you can protect](https://docs.aws.amazon.com/waf/latest/developerguide/how-aws-waf-works-resources.html)
- [AWS WAF: quotas](https://docs.aws.amazon.com/waf/latest/developerguide/limits.html)
- [AWS WAF: body inspection](https://docs.aws.amazon.com/waf/latest/developerguide/web-request-body-inspection.html)
- [AWS WAF: rule actions](https://docs.aws.amazon.com/waf/latest/developerguide/waf-rule-action.html)
- [AWS WAF: rate-based rule statement](https://docs.aws.amazon.com/waf/latest/developerguide/waf-rule-statement-type-rate-based.html)
- [AWS WAF: logging destinations](https://docs.aws.amazon.com/waf/latest/developerguide/logging-destinations.html)
- [AWS WAF pricing](https://aws.amazon.com/waf/pricing/)
- [AWS: flat-rate pricing plans for website delivery and security](https://aws.amazon.com/about-aws/whats-new/2025/11/aws-flat-rate-pricing-plans)
- [Security Automations for AWS WAF](https://aws.amazon.com/solutions/implementations/security-automations-for-aws-waf/)
