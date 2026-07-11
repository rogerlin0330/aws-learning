---

tags:

- aws
- saa-c03
- networking
- elastic-load-balancing
- alb
- nlb
- high-availability

---

# Application vs Network Load Balancers (ALB vs NLB)

## Overview

This lesson focuses on the two **v2 (elastic) load balancers** you must be able to choose between on the exam: the **Application Load Balancer (ALB)** and the **Network Load Balancer (NLB)**. They operate at different layers of the networking stack and are suited to _massively different_ situations, so the exam frequently tests **which one to pick** for a given scenario rather than deep configuration detail.

The mental model taught here: **memorise the short list of reasons to choose an NLB. If the scenario matches none of them, default to an ALB.**

> 🎯 **Exam trigger keywords** are surfaced inline as 🎯. Exam traps/warnings are 📛 ⚠️. Supplementary facts beyond the transcript (verified against current AWS docs) are 📚.

---

## Part 1 — Why v2 exists: consolidation of load balancers

### The Classic Load Balancer (CLB) problem

Historically, with **Classic Load Balancers** you connected instances (or an Auto Scaling group) directly to the CLB. A typical architecture:

- One domain (`catagram.io`) → one CLB → one SSL certificate → one Auto Scaling group.

The problem: **CLBs don't support SNI** (Server Name Indication), and **you can't attach multiple SSL certificates to a single CLB**. Consequently, **every unique HTTPS application needs its own CLB**.

So delivering two HTTPS apps — `catagram.io` and `dogagram.io` — requires **two separate CLBs**. Scale that to 100 legacy apps and you have 100 load balancers.

> 📛 This inability to host multiple certificates is one of the main reasons **CLBs should be avoided** for new deployments.

### The ALB solution

A single ALB can front **both** applications:

- **Listener-based rules**, each able to carry an SSL certificate, handle HTTPS for **both** domains.
- **Host-based rules** direct incoming connections to **multiple target groups**, which forward to **multiple Auto Scaling groups**.

That's a **2:1 consolidation** — half the load balancers for the same two apps. Across a large legacy estate, the savings compound dramatically. **Consolidation is one of the headline advantages of moving from v1 (CLB) to v2 (ALB).**

> 📚 The ALB's ability to host multiple certificates relies on **SNI**, which the CLB lacks. SNI is what makes the single-ALB, many-domains pattern possible.

---

## Part 2 — Application Load Balancer (ALB) deep dive

### It's a true Layer 7 load balancer

- Configured to listen on **HTTP** or **HTTPS only**.
- Understands and can **interpret** information carried in those Layer 7 application protocols.

> 📛 ⚠️ **The flip side:** an ALB understands _only_ HTTP/HTTPS. It **cannot** handle other Layer 7 protocols such as **SMTP, SSH, or custom gaming protocols**. It also **cannot** be configured to listen directly on **TCP, UDP, or TLS**.

Because it operates at Layer 7, it can inspect and make decisions based on:

- Content type
- Cookies
- Custom headers
- User location
- Application behaviour

…none of which an NLB can see.

### SSL/TLS is always terminated on the ALB

Any incoming connection — HTTP, or HTTPS (which is just HTTP transiting over SSL/TLS) — is **terminated on the ALB**. A **new, separate connection** is then made from the ALB to the backend instances.

> 📛 ⚠️ **This is a hard exam trap.** You **cannot** have an unbroken/end-to-end SSL connection from client through to the application instances when using an ALB. This matters to strict security teams and can, on its own, **exclude the ALB** from consideration.
> 
> Consequence: **every HTTPS ALB must have an SSL certificate installed on the load balancer**, because the connection terminates there.

### Performance

ALBs are **slower than NLBs** because more layers of the networking stack must be processed — more layers = more complexity = more latency.

> 🎯 If an exam question is strict about **raw performance / lowest latency**, lean toward **NLB**.

### Health checks (a Layer 7 benefit)

Because it understands Layer 7, an ALB can evaluate **application health** — it can make an actual application-layer request and confirm the app responds correctly, not merely that a network connection succeeded.

### Rules — the core of ALB flexibility

Connections arriving at a **listener** are handled according to **rules**, which are **processed in priority order**. The last rule is always the **default rule** (a catch-all); you add higher-priority rules above it.

**Rule conditions** can match on:

- Host header (which domain — Catagram vs Dogagram)
- HTTP headers
- HTTP request method
- Path pattern (e.g. `/images` vs `/api`)
- Query string
- **Source IP**

**Rule actions** can:

- **Forward** to a target group
- **Redirect** elsewhere (e.g. to another domain)
- Return a **fixed HTTP response** (a specific error or success code)
- **Authenticate** — using **OpenID Connect** or **Amazon Cognito**

**Worked scenarios from the lesson:**

1. **Simple forward** — `catagram.io`: one host-based rule, SSL cert attached, condition = _host header_, action = _forward_ to the Catagram target group.
2. **Corporate override by source IP** — Users of "Bowtie Inc." (source IP `1.3.3.7`) should get an alternative version. Add a listener rule with condition = _source IP `1.3.3.7`_, action = _forward to a separate target group / ASG_ dedicated to that client. Because the ALB is Layer 7, it can see inside the HTTP protocol and decide on anything up to Layer 7. (The client→ALB connections are terminated; a separate set of connections runs ALB→instances.)
3. **HTTP redirect** — If the same ALB also handles `dogagram.io`, add a rule matching that domain whose action is **redirect** to `catagram.io` instead of forwarding.

> 🎯 If a scenario needs routing decisions based on **path, headers, cookies, host, or HTTP-level redirects**, that's **ALB** territory.

---

## Part 3 — Network Load Balancer (NLB) deep dive

### It's a Layer 4 load balancer

- Operates at **Layer 4**, interpreting **TCP, TLS, and UDP** (and the combined `TCP_UDP` listener).
- **No visibility of HTTP/HTTPS.** It **cannot** interpret headers or cookies, and has **no concept of HTTP session stickiness** (that relies on cookies, which live at Layer 7).

### Extremely fast

NLBs can handle **millions of requests per second** and have **~25% of the latency of an ALB** (roughly a quarter), precisely because they skip the computationally heavy upper layers and deal only with Layer 4.

> 🎯 "**Millions of requests per second**", "**lowest latency**", "**highest performance**" → **NLB**.

### Ideal for non-HTTP(S) protocols

Because it's protocol-agnostic at Layer 4, the NLB is the right choice for anything that isn't web/secure-web:

- **SMTP** email
- **SSH**
- **Game servers** using non-web protocols
- **Financial applications** not using HTTP/HTTPS

> 🎯 Exam question mentions a protocol that **isn't HTTP or HTTPS**? Default to **NLB**.

### Weaker health checks

Not being Layer 7-aware, NLB health checks only verify **ICMP** and **basic TCP handshaking** — they are **not application-aware**. No detailed, application-level health checking.

### Static IP addresses

NLBs can be allocated **static IP addresses** — extremely useful for **whitelisting**.

> 🎯 If corporate clients need to **whitelist your load balancer's IPs** through their firewall → **NLB** (static IPs). Great for strict security environments.

### Unbroken end-to-end encryption (TCP pass-through)

Upper networking layers build on the layers below them. Because an NLB doesn't understand HTTP/HTTPS, you can configure a **TCP listener** that forwards TCP **straight through** to the instances. Anything layered on top of TCP (e.g. TLS/HTTPS) is therefore **not terminated** on the load balancer and **not interrupted**.

> 🎯 📛 **This is the canonical way to achieve unbroken, end-to-end encryption from client to instance: an NLB with a TCP listener.** If a question requires forwarding encrypted connections to instances _without terminating them on the load balancer_, the answer is **NLB** (not ALB).

> 📚 **Nuance not covered in the transcript:** NLB _also_ supports a **TLS listener**, which **does terminate** TLS on the NLB (and therefore needs a certificate) — useful when you want the NLB to offload TLS. The **end-to-end / unbroken** encryption behaviour above is specific to the **TCP** listener, where the NLB never decrypts. Don't conflate the two. (AWS Elastic Load Balancing docs)

### PrivateLink

NLBs are used with **AWS PrivateLink** to expose a service to other VPCs.

> 🎯 📛 Any scenario involving **PrivateLink** → **NLB**. (This is a common, high-value exam fact.)

---

## Part 4 — Decision framework (memorise this)

Remember the **reasons to choose an NLB**. If none apply, **default to ALB**.

**Choose an NLB when you need:**

1. **Unbroken / end-to-end encryption** between client and instances (TCP listener pass-through).
2. **Static IPs** for whitelisting.
3. **Absolute best performance** — millions of RPS, lowest latency.
4. **Protocols that are not HTTP or HTTPS** (SMTP, SSH, game/financial protocols, etc.).
5. **PrivateLink** — exposing a service to other VPCs.

**Otherwise → ALB**, because its Layer 7 features (rules, host/path routing, header/cookie awareness, redirects, authentication, application-aware health checks, cert consolidation) are valuable in most web scenarios.

---

## Exam Traps & Gotchas

- 📛 **ALB terminates SSL/TLS — always.** No end-to-end encryption to instances via ALB. Every HTTPS ALB needs a cert _on the ALB_.
- 📛 **NLB can't read HTTP.** No header/cookie inspection, no HTTP session stickiness, no path/host-based routing.
- 📛 **ALB listens on HTTP/HTTPS only** — never TCP/UDP/TLS directly. Non-web protocol → not an ALB.
- 📛 **End-to-end encryption pass-through = NLB + TCP listener** (not TLS listener, which terminates).
- 📛 **PrivateLink = NLB.** No other v2 LB provides this.
- 📛 **Static IP for whitelisting = NLB.** ALBs don't offer fixed IPs natively.
- 📛 **NLB health checks are shallow** (ICMP / TCP handshake only) — not application-aware.
- ⚠️ Don't be fooled by "performance-critical" wording _and_ "path-based routing" in the same question — path routing forces **ALB**; you cannot have both NLB speed _and_ Layer 7 routing on one LB.

---

## Quick-Reference: ALB vs NLB

|Attribute|Application Load Balancer (ALB)|Network Load Balancer (NLB)|
|---|---|---|
|OSI layer|**Layer 7** (application)|**Layer 4** (transport)|
|Listener protocols|**HTTP, HTTPS** only|**TCP, UDP, TCP_UDP, TLS**|
|Understands HTTP/headers/cookies|✅ Yes|❌ No|
|Host / path / header / query / source-IP routing|✅ Yes (rules)|❌ No|
|SSL/TLS behaviour|**Always terminated** on the LB (cert required)|**TCP listener = pass-through** (unbroken E2E); **TLS listener = terminates** 📚|
|End-to-end (unbroken) encryption to instances|❌ No|✅ Yes (via TCP listener)|
|Performance / latency|Higher latency|**~4× lower latency**, millions of RPS|
|Static IP addresses|❌ Not native|✅ Yes (whitelisting)|
|Health checks|**Application-aware** (L7 request)|**Basic** (ICMP / TCP handshake)|
|Redirects, fixed responses, OIDC/Cognito auth|✅ Yes|❌ No|
|PrivateLink support|❌ No|✅ **Yes**|
|Non-HTTP protocols (SMTP, SSH, game, finance)|❌ No|✅ Yes|
|Certificate consolidation (SNI, many domains)|✅ Yes|N/A (no L7)|

---

## Course vs. current AWS (flagged discrepancies)

The lesson is accurate for exam purposes, but two behaviours have changed or are worth rounding out since the course was recorded:

- 📚 ⚠️ **NLBs now support Security Groups (since 10 Aug 2023).** The older framing — that NLBs have _no_ security group and you control access at the instance level — is no longer strictly true. You can now attach SGs to an NLB to filter inbound traffic. **Important quirk:** you can only associate security groups **at NLB creation time**; if you create an NLB without SGs, you **cannot add them later** (you'd have to replace the NLB). _(AWS What's New, Aug 2023; NLB security groups docs.)_ This is unlikely to be tested against you on SAA-C03, but the "NLB has no security group" claim you may see in older material is now outdated.
    
- 📚 **There is a fourth ELB type: the Gateway Load Balancer (GWLB).** Introduced at re:Invent 2020, GWLB operates at **Layer 3**, uses the **GENEVE protocol on port 6081**, and exists to deploy/scale **third-party virtual appliances** (firewalls, IDS/IPS, deep-packet inspection) as a transparent "bump-in-the-wire." It's out of scope for this ALB-vs-NLB comparison and only lightly touched at SAA level, but it completes the ELB family: **ALB (L7) · NLB (L4) · GWLB (L3) · CLB (legacy).** _(AWS Elastic Load Balancing / GWLB docs.)_
    

---

## Open questions for Roger

- Do you want a **separate dedicated GWLB note** when the course reaches it (if it does), or is the flag above enough for now?
- Cantrill's **~25% latency** figure is his standard framing and matches AWS's older marketing; AWS doesn't publish a current hard number. I've kept it as the course framing (labelled "~4× lower / ~25%"). Flag if you'd prefer it softened to a qualitative "significantly lower latency."

---

## Source attribution

- Primary: Adrian Cantrill AWS SAA course — _"Application Load Balancing (ALB) vs Network Load Balancing (NLB)"_ (transcript).
- Verified against: AWS Elastic Load Balancing documentation (NLB security groups; listener protocols), AWS What's New (NLB security groups, Aug 2023), AWS Gateway Load Balancer documentation.