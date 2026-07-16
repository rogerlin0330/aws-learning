---

tags:

- aws
- saa-c03
- elb
- alb
- nlb
- ssl-tls
- stickiness

---

# ELB SSL Offload & Session Stickiness

## Overview

Two ELB features the exam tests at the **architecture level**, not the implementation level. You need to know the trade-offs, not the click-path.

1. **SSL/TLS connection modes** — three ways an ELB can handle encrypted connections: **bridging**, **pass-through**, **offload**.
2. **Session stickiness** — binding a user's session to one backend instance via a cookie.

The recurring theme: _who holds the certificate_, _who does the crypto work_, and _what the load balancer can see_.

---

## Diagram Walkthrough — SSL Connection Modes

![[elb-ssl-connection-modes.png]]

Three horizontal rows, each one architecture. Read left → right: **client (laptop) → load balancer (purple ELB/NLB icon) → EC2 instances (orange icons)**. Line colour encodes encryption state.

**Row 1 — Bridging (top).** The laptop connects to an **ELB** over two red/orange lines (encrypted client connections). Callout under the client: the listener is configured for **HTTPS**, the connection is **terminated on the ELB**, and the ELB **needs a certificate** for the domain name. On the right of the ELB, the lines to the instances are _also_ red/orange — meaning **re-encrypted**. Right-hand callout: the ELB initiates a **new SSL connection** to the backends, so instances **need SSL certificates** and the **compute** for cryptographic operations. Encrypted on both sides, but broken in the middle.

**Row 2 — Pass-through (middle).** A single unbroken red line runs from laptop → **NLB** → instance. It never changes colour, because it is never decrypted — one continuous tunnel. Callout: the listener is configured for **TCP**, **no encryption or decryption happens on the NLB**, the connection is passed to the backend instance. Right callout: each instance needs the **SSL cert installed**, and there is **no certificate exposure to AWS** — all self-managed.

**Row 3 — Offload (bottom).** Red/orange lines from laptop → **ELB** (encrypted), but the lines from ELB → instances are **green** — the colour change _is_ the point. Callout under the client: listener is **HTTPS**, connections are **terminated**, and backend connections **use HTTP**. Right callout: ELB-to-instance connections use **HTTP** — **no certificate** and **no cryptographic requirements** on the instances. The large pink **HTTP** arrow in the middle-right points at the green lines, labelling them as plaintext.

**Colour key:** red/orange = encrypted, green = plaintext HTTP.

---

## The Three Connection Modes

### 1. Bridging (ALB default)

- Client → ELB: **HTTPS listener**. Connection is **terminated** (decrypted) on the ELB.
- ELB → instances: **new, separate HTTPS connections**.
- **Certificate needed on the ELB _and_ on every instance.**

**Why it's the ALB default:** terminating the connection strips the SSL wrapper off the HTTP inside, so the ALB can **see and act on the HTTP** — the entire basis of L7 routing (host/path rules, headers). An ALB cannot route on content it cannot read.

|Pros|Cons|
|---|---|
|ELB sees plaintext HTTP → L7 routing decisions|Certificate stored on the ELB → **AWS has some level of access** to it|
||Certs must be distributed/maintained on every instance (admin overhead)|
||Every instance performs crypto → CPU overhead, significant at high volume|

### 2. Pass-through (NLB)

- **NLB** with a **TCP listener**. The NLB **never decrypts anything**.
- One encrypted tunnel: **client → backend instance**, end to end.
- Certificate on the **instances only** — never on the load balancer, **never exposed to AWS**.
- The NLB balances on what it _can_ see: **source/destination IP and port**.

|Pros|Cons|
|---|---|
|**No certificate exposure to AWS** — fully self-managed|**No HTTP-based load balancing** — the L7 data is never decrypted|
|Can pair with **CloudHSM** for stronger key control|Instances still need certs **and** the compute for crypto|

📚 The security-framework angle is the whole reason this mode exists on the exam: pass-through is the only one of the three where AWS never touches your private key.

### 3. SSL Offload

- Client → ELB: **HTTPS listener**, terminated on the ELB. **Certificate required on the ELB.**
- ELB → instances: **plain HTTP**. Never re-encrypted.

|Pros|Cons|
|---|---|
|**No certificates on instances**|Data crosses the **AWS network in plaintext**|
|**No crypto compute on instances** → lower per-instance overhead, potentially **smaller instance types**|Cert still stored on the ELB (AWS access)|
|Encrypted across the public internet, where it matters most||

---

## Quick-Reference — Connection Modes

||**Bridging**|**Pass-through**|**Offload**|
|---|---|---|---|
|Typical LB|**ALB (default)**|**NLB**|ALB (or NLB w/ TLS listener)|
|Listener protocol|HTTPS|**TCP**|HTTPS|
|Terminated on LB?|✅ Yes|❌ No|✅ Yes|
|Cert on LB?|✅|❌|✅|
|Cert on instances?|✅|✅|❌|
|Crypto on instances?|✅|✅|❌|
|LB can see HTTP?|✅|❌|✅|
|Cert exposed to AWS?|✅|❌|✅|
|Encrypted LB → instance?|✅|✅ (never decrypted)|❌ **plaintext**|

---

## Diagram Walkthrough — Connection Stickiness

![[alb-connection-stickiness.png]]

**Centre:** the purple **ELB** icon, with three orange **EC2-1 / EC2-2 / EC2-3** instances stacked on the right.

**Top user (male icon):** three **red** lines fan down into the ELB, and red lines fan out from the ELB to **all three** instances — this is the **no-stickiness** case: his connections are spread across the fleet.

**Bottom-left user (female icon):** three **green** lines run into the ELB, and a single **green** line runs out to **EC2-2 only** — this is **stickiness**: every session from this user lands on the same instance.

**Top-left callout:** stickiness **generates a cookie** which locks the device to a **single backend instance** for a **duration**.

**Left, large text:** **1s to 7 days** — the valid duration range.

**Bottom-left:** the **Cookie Monster** icon — Cantrill's visual mnemonic that stickiness = cookies.

**Bottom callout:** with no stickiness, connections are **distributed across all in-service backend instances**. Unless the application handles user state, this could cause **user logoffs and shopping-cart losses**.

---

## Session Stickiness Mechanics

**No stickiness (default):** every request is routed independently across all **in-service** (health-checked) targets — a fair, even spread. Fine _only_ if the app is **stateless**.

**The problem it solves:** if session state lives on the instance itself, every hop to a new instance looks like a fresh start — re-login, empty cart.

**With stickiness enabled:**

1. First request → LB picks a target and returns a cookie named **`AWSALB`**.
2. Subsequent requests carry the cookie → LB routes to **the same instance** every time.
3. Enabled at the **target group** level (not the listener, not the LB).
4. Duration is configurable: **1 second to 7 days**.

**Stickiness breaks when one of two things happens:**

- **Instance failure** — the target fails health checks, and the user is moved to a different instance.
- **Cookie expiry** — the cookie lapses; the process restarts with a new cookie and a new (possibly different) instance.

**The downside:** ⚠️ **uneven backend load.** A single heavy user is pinned to one server forever. Stickiness undermines the fair balancing you deployed a load balancer to get.

**The architect's answer:** design **stateless instances** and externalise session state — **DynamoDB** (Cantrill's canonical example), or ElastiCache/Redis. Then no cookies, no pinning, fully fair balancing, and instances become disposable (which is what Auto Scaling needs anyway).

---

## Course vs. Current AWS

📚 The lesson's framing is correct but simplified in three places. The course version is what older exam questions reflect; the current reality is worth knowing.

**1. NLB can terminate TLS — pass-through is a choice, not a limitation.** The lesson presents NLB as the pass-through option. NLB has supported **TLS termination since January 2019**, integrating with ACM and IAM for centralized certificate deployment, with optional re-encryption to targets and predefined security policies ([AWS What's New](https://aws.amazon.com/about-aws/whats-new/2019/01/network-load-balancer-now-supports-tls-termination/)). So:

- **NLB + TCP listener** = pass-through (what the lesson describes).
- **NLB + TLS listener** = offload/bridging at L4. ⚠️ In exam terms: **the listener protocol determines the mode**, not the load balancer type. TCP listener → pass-through. HTTPS/TLS listener → terminated.

**2. Two stickiness types, not one.** The lesson covers only the LB-generated cookie. ALB supports **both duration-based and application-based cookies**, and you can mix duration-based, application-based, and no stickiness across different target groups ([AWS docs](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/sticky-sessions.html)).

- **Duration-based** → `AWSALB` cookie, LB-generated, 1s–7d. This is the lesson's version.
- **Application-based** → added **February 2021**; ALB uses a **custom cookie set by your target**, with custom cookie names and criteria at the target group level, allowing stickiness to be maintained **across chained load balancers** ([AWS What's New](https://aws.amazon.com/about-aws/whats-new/2021/02/application-load-balancer-supports-application-cookie-stickiness/)). The client-visible cookie is named **`AWSALBAPP-`** plus a fragment number.

**3. The "1s–7d" number is the _stickiness duration_, not the cookie's expiry.** ⚠️ Subtle but worth knowing: the AWSALB **cookie itself has a non-configurable 7-day expiry**; your configured duration controls how long the LB keeps routing to the same target ([AWS docs](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/sticky-sessions.html)). The exam almost certainly wants "1 second to 7 days" — answer that. Just don't be surprised by a 7-day `Expires` header in the real world.

📚 Other current-reality footnotes worth a glance:

- **Enabling stickiness fails if cross-zone load balancing is disabled.**
- **Application-based stickiness does not work with weighted target groups.**
- **WebSocket connections are inherently sticky.**
- On target failure, the LB moves the session to a new healthy target and **keeps it there even if the original recovers** — the session is now stuck to the _new_ target.
- **CLB** uses a different cookie: **`AWSELB`** ([AWS docs](https://docs.aws.amazon.com/elasticloadbalancing/latest/classic/elb-sticky-sessions.html)).

---

## Exam Traps

⚠️ **Bridging requires certs in _two_ places.** ELB _and_ every instance. Easy to forget the second half.

⚠️ **"AWS never sees the certificate" → pass-through, always.** This is the only mode where the private key stays entirely yours. If the question adds **CloudHSM**, it's doubly pass-through.

⚠️ **Offload's cost is plaintext on the AWS network.** If the question mentions compliance requiring encryption in transit _end-to-end_ or _at all times_, offload is wrong.

⚠️ **Stickiness is a target-group setting.** Not a listener setting, not a load-balancer setting.

⚠️ **"Reduce instance CPU / use smaller instances" + HTTPS → offload.** Removing crypto from the backend is offload's headline benefit.

⚠️ **A question describing lost carts / repeated logins has two possible answers.** Stickiness is the _quick fix_; externalising state to DynamoDB is the _well-architected_ answer. Read for which one the question is asking — "best practice" / "scalable" / "even load distribution" wording points to stateless.

### 🎯 Trigger Keywords

|Trigger phrase|Answer|
|---|---|
|"certificate must not be exposed to AWS" / "regulatory control of private keys" / "CloudHSM"|**Pass-through** (NLB, TCP listener)|
|"route based on URL path / host header / HTTP headers" + HTTPS|**Bridging** (ALB, HTTPS listener)|
|"reduce cryptographic overhead on instances" / "smaller instance types" / "no certs on backends"|**Offload**|
|"default mode of the application load balancer"|**Bridging**|
|"end-to-end encryption"|**Pass-through** (or bridging — read whether the LB needs to see HTTP)|
|"listener configured for TCP"|**Pass-through** — no decryption|
|"listener configured for HTTPS"|Terminated on the LB → bridging _or_ offload|
|"users being logged out" / "shopping cart emptied" / "session lost"|**Stickiness** (quick fix) or **externalise state** (best practice)|
|"AWSALB cookie"|ALB **duration-based** stickiness|
|"AWSELB cookie"|**CLB** stickiness|
|"AWSALBAPP-" cookie|ALB **application-based** stickiness|
|"uneven load across backend instances"|Stickiness is the **cause**|
|"stateless" / "session state in DynamoDB"|No stickiness needed — fair balancing|
|"1 second to 7 days"|Stickiness duration range|

---

## Open Questions

- **Roger** — do you want a cross-link stub to a future **ACM** note? The cert-storage discussion here (bridging/offload need a cert on the ELB) leans directly into ACM, and Cantrill covers it later.
- Same question for **CloudHSM** — mentioned in passing here ("I'll talk about later in the course") and it's the strongest pass-through trigger.
- Worth a back-link from your existing **ALB vs. NLB** note? The "NLB can't do L7 routing" point is stated in both places and they should agree.

---

## Source

Adrian Cantrill, _AWS Certified Solutions Architect – Associate (SAA-C03)_, learn.cantrill.io — lesson: **SSL Offload & Session Stickiness**. Diagrams: Cantrill course slides.