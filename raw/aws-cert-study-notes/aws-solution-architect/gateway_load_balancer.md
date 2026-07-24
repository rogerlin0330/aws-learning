---

tags:

- aws
- saa
- networking
- load-balancing
- security

---
---

tags:

- aws
- saa
- networking
- load-balancing
- security

---

# Gateway Load Balancer

## Overview

A **Gateway Load Balancer (GWLB)** is a product AWS built to let you run and scale **third-party network security appliances** — firewalls, IDS/IPS, deep packet inspection, data analysis tools — in a way that is transparent, horizontally scalable, and decoupled from the application it protects.

The problem it solves is architectural, not functional. You can already put a security appliance inline with your traffic by running it on an EC2 instance. What you _can't_ easily do is make that arrangement elastic. Once your application scales in and out, you need a matching, independently scaling fleet of appliances, and hand-wiring that fleet into the traffic path is complex and failure-prone. The course frames this precisely: the naive design is **tightly coupled** — application and security instances are tied together, and the failure of one impacts the other. GWLB breaks that coupling.

Two components make it work:

| Component                                  | Lives in                                                | Role                                                                                                  |
| ------------------------------------------ | ------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| **Gateway Load Balancer Endpoint (GWLBE)** | The VPC whose traffic you want inspected (consumer VPC) | A VPC endpoint that **can be used as a route table target**, so it can be injected into traffic flows |
| **Gateway Load Balancer (GWLB)**           | The VPC hosting the appliances (provider/security VPC)  | Load balances packets across backend appliance instances, encapsulating with GENEVE                   |

<cite index="19-1">GWLB operates at layer 3 of the OSI model, listens for all IP packets across all ports, forwards traffic to the target group specified in the listener rule, maintains flow stickiness to a specific target appliance using 5-tuple (default), 3-tuple, or 2-tuple, and exchanges application traffic with its registered appliance instances using the GENEVE protocol on port 6081.</cite>

⚠️ **Course-vs-reality — layer classification.** The transcript describes GWLB as "layer three and four devices, similar in many ways to a network load balancer." Current AWS ELB documentation states GWLB <cite index="19-1">operates at the third layer of the OSI model, the network layer</cite>. However, AWS Prescriptive Guidance takes the course's position: <cite index="27-1">Gateway Load Balancers work at layer 3 and layer 4 of the OSI model — at layer 3 transparently routing the packet from source to third-party appliances before sending it to the destination symmetrically, and at layer 4 providing highly-available and scalable load balancing to the endpoints plus health checks.</cite> Both framings appear in AWS's own material. **For exam purposes, "Layer 3" is the safer single answer** (it's what the primary ELB service doc says), but understand the L4 claim refers to the load-balancing/health-check plane, not packet routing. Sources: [ELB — What is a Gateway Load Balancer?](https://docs.aws.amazon.com/elasticloadbalancing/latest/gateway/introduction.html), [AWS Prescriptive Guidance — Inline traffic inspection](https://docs.aws.amazon.com/prescriptive-guidance/latest/inline-traffic-inspection-third-party-appliances/welcome.html)

⚠️ **Course-vs-reality — flow stickiness tuple options.** The transcript says only that "the gateway load balancer manages flow stickiness, so one flow of data will always use one appliance." Current AWS docs specify this is <cite index="19-1">configurable: 5-tuple (default), 3-tuple, or 2-tuple</cite>. The 2-tuple option is a later addition not present when the course was recorded. Source: [ELB — Introduction](https://docs.aws.amazon.com/elasticloadbalancing/latest/gateway/introduction.html)

---

## Diagram Walkthrough

![[gwlb-architecture.png]]

**How to read this diagram.** The layout is not a left-to-right flow — it's two VPC containers side by side with a numbered packet trace (1→16) weaving between them. Read it in three passes.

**Context bar (top).** Outside both VPCs sits the client laptop and the internet gateway, joined by arrows `1` (inbound, pink) and `16` (outbound, blue). **Pink arrows are the inbound/request direction; blue arrows are the return/response direction** — this colour coding holds throughout the diagram and is the fastest way to orient yourself in the middle of the trace. To the right of the IGW is the **ingress (gateway) route table** — three rows, with `10.16.0.0/16 → LOCAL`, `10.16.32.0/20 → GWLBE1`, and `10.16.96.0/20 → GWLBE2`. The callout beside it states the rule this table implements: traffic destined for the ALB subnets is directed to the Gateway Endpoint **in the same AZ**. This table is attached to the IGW, not to a subnet — that is what makes it a _gateway_ route table.

**Main objects (middle band).** Two VPC boxes:

- **catagramVPC** (left, larger) contains two AZ columns. Each column is a vertical stack of three subnets, colour-coded: **green = public, blue = private**. Top to bottom in each column: the GWLBE subnet, the ALB subnet, the application instance subnet. The left column is AZ-A (GWLBE1 `10.16.48.0/20`, ALB `10.16.32.0/20`, app `10.16.80.0/20`); the right column is AZ-B (GWLBE2 `10.16.112.0/20`, ALB `10.16.96.0/20`, app `10.16.144.0/20`). Note the diagram draws the ALB icon in _both_ columns — that's one ALB spanning two AZs, not two load balancers.
- **securityVPC** (right) contains the GWLB icon at top and, below it, a pair of subnets holding the appliance instances. The **pink/salmon triangle** fanning down from the GWLB to the appliances represents the GENEVE tunnels — the callout beside it reads "Packets are encapsulated through to the appliances," and "Original SRC and DST are maintained."

Three route tables float in the middle of the catagramVPC box, each a two-row table. Match each to its subnet by position, not by proximity alone:

- Far left (`10.16.0.0/16 → LOCAL`, `0.0.0.0 → IGW`) and the one to its right in the second AZ column: **the GWLBE subnet route tables** — default route points at the **IGW**.
- The lower table (`10.16.0.0/16 → LOCAL`, `0.0.0.0 → GWLBE1` / `GWLBE2`): **the ALB subnet route tables** — default route points at the **GWLB endpoint in the same AZ**. The callout on the far left spells this out: any outbound return traffic from the ALB is sent via default route to the Gateway LB endpoint.

### The numbered trace: steps 1–16

The whole diagram is one request/response pair for a single client. Steps **1–8 are the inbound request** (pink arrows); steps **9–16 are the response** (blue arrows). Everything happens in the AZ-B column — the AZ-A column (GWLBE1, `10.16.32.0/20`) is drawn to show the architecture is multi-AZ, but this particular flow never touches it.

Before starting, fix three addresses in mind: the **client's public IP** (source), the **ALB's public IP** (original destination), and the **ALB's private IP** in `10.16.96.0/20` (the destination after the IGW's NAT).

#### Inbound: request (pink)

**`1` — Client → Internet Gateway.** The client browses `catagram.io`. DNS resolves to the ALB's public IP, so the packet leaves the laptop with SRC = client public IP, DST = ALB public IP, and arrives at the IGW. Nothing AWS-specific has happened yet.

**`2` — Internet Gateway → GWLBE2.** Two things happen at the IGW, **in this order**:

- First, the IGW performs its normal destination NAT: DST is rewritten from the ALB's public IP to the ALB's **private** IP inside `10.16.96.0/20`.
- Then the **ingress (gateway) route table** — the three-row table drawn to the right of the IGW — is consulted against that now-private destination. `10.16.96.0/20 → GWLBE2` is a more specific match than `10.16.0.0/16 → LOCAL`, so it wins.

🎯 The NAT precedes the route lookup. This is why the ingress table lists private subnet CIDRs rather than public IPs, and it's the single most commonly misread step in this diagram. The packet is now heading to the GWLB endpoint in the **same AZ** as the ALB subnet it was destined for — that's what the callout above the table means.

**`3` — GWLBE2 → GWLB.** The endpoint hands the packet across the VPC boundary into securityVPC. Note what does _not_ happen here: **no route table is consulted**. This hop is PrivateLink — a bound association between the endpoint and the GWLB's endpoint service. The packet still carries its original SRC (client public IP) and DST (ALB private IP), which is why the callout on the right reads "Original SRC and DST are maintained."

**`4` — GWLB → appliance (encapsulated).** The GWLB selects an appliance from its target group using flow stickiness (5-tuple by default), wraps the original packet in a **GENEVE header (UDP 6081)**, and sends it down the tunnel. This is the pink side of the salmon triangle in the diagram. Encapsulation is what allows a packet addressed `client-public-IP → 10.16.96.x` to traverse securityVPC, whose CIDR may be entirely different or even overlapping.

**`5` — Appliance → GWLB (inspected, re-encapsulated).** The appliance decapsulates, inspects the _original_ packet — firewall rules, IDS signatures, whatever the vendor does — and decides to allow or drop. Assuming allow, it re-wraps the packet in GENEVE and returns it. The GWLB strips the outer header.

⚠️ Steps `4` and `5` are where a drop would occur. The diagram only depicts the allow path; a blocked packet simply never produces a step `6`.

**`6` — GWLB → GWLBE2.** The packet goes back to the endpoint it came from. Critically, the GWLB knows _which_ endpoint that is from **GENEVE header metadata**, not from a routing decision. There is no lookup here that could send it somewhere wrong.

**`7` — GWLBE2 → ALB.** Now the packet is back in catagramVPC, decapsulated, and exposed to normal VPC routing for the first time since step `2`. It consults the **GWLBE subnet's own route table** (`10.16.0.0/16 → LOCAL`, `0.0.0.0/0 → IGW`). DST is `10.16.96.x`, inside the VPC CIDR, so the **LOCAL** route matches and the packet moves to the ALB subnet.

🎯 This is the loop-prevention step. That table contains **no route pointing at GWLBE2** — the route that sent traffic _to_ the endpoint lives in the IGW's ingress table, a different table entirely. The packet cannot be re-captured.

**`8` — ALB → application instance.** Standard ALB behaviour: it terminates the connection, picks a target from its target group, and forwards to the app instance in `10.16.144.0/20`. Everything from here down is ordinary application architecture — GWLB is finished with the inbound leg.

#### Outbound: response (blue)

**`9` — Application instance → ALB.** The app generates its response and sends it back to the ALB, which is the only thing it ever talked to. The instance has no idea inspection occurred.

**`10` — ALB → GWLBE2.** The ALB's subnet route table (`10.16.0.0/16 → LOCAL`, `0.0.0.0/0 → GWLBE2`) is consulted. The response is headed for the client's **public** IP, which is outside the VPC CIDR, so the **default route** matches and sends it to the GWLB endpoint. This is the callout on the far left of the diagram: outbound return traffic from the ALB goes via default route to the Gateway LB endpoint.

⚠️ Note the asymmetry in _how_ traffic gets steered even though the _path_ is symmetric: inbound was captured by a **specific** route in the IGW's ingress table; outbound is captured by the **default** route in the ALB subnet's table.

**`11` — GWLBE2 → GWLB.** PrivateLink again, no route lookup. Same as step `3`, opposite direction.

**`12` — GWLB → appliance (encapsulated).** The GWLB matches this packet to the existing flow and sends it to **the same appliance** that handled steps `4`/`5`. This is flow stickiness doing its job.

🎯 This is the reason stickiness exists. A stateful firewall that saw only the response half of a TCP conversation would drop it as an unsolicited packet. Both directions must land on the same appliance.

**`13` — Appliance → GWLB.** Outbound inspection — data exfiltration checks, DLP, C2 beaconing detection. The transcript's phrase for this is filtering traffic on the way back out, for example detecting and preventing information leakage. Re-encapsulated and returned; GWLB strips the header.

**`14` — GWLB → GWLBE2.** Return to the originating endpoint via GENEVE metadata, as in step `6`.

**`15` — GWLBE2 → Internet Gateway.** Decapsulated, the packet consults the **GWLBE subnet route table** — the same table used at step `7`, but this time producing a different result. DST is the client's public IP, outside the VPC CIDR, so `0.0.0.0/0 → IGW` matches. This is the "GWLBE subnet uses IGW as default outbound target" callout.

📚 Steps `7` and `15` are the same table consulted twice with opposite outcomes, and that's worth sitting with. The table doesn't know or care whether a packet has been inspected — it just does longest-prefix matching. Inbound-after-inspection matches LOCAL; outbound-after-inspection matches the default route. **Nothing in this design tracks inspection state; correctness comes purely from the fact that the endpoint is never a target in its own table.**

**`16` — Internet Gateway → client.** The IGW performs source NAT in reverse: SRC is rewritten from the ALB's private IP back to its public IP, and the response reaches the client. The client has no idea any of steps `2` through `15` happened.

#### What the trace teaches

🎯 **Symmetry is the headline.** Steps `1`–`8` and `9`–`16` are mirror images, and the same appliance handles both — that's what makes stateful inspection possible.

📚 **Four VPC boundary crossings** for one request/response pair (steps `3`, `6`, `11`, `14`). This is not inefficiency: each crossing is a PrivateLink hop over the AWS backbone, never the public internet.

📚 **Only three routing decisions in the whole trace** — steps `2`, `7`/`15`, and `10`. Every other hop is either PrivateLink, GENEVE metadata, or ordinary ALB target selection. If you can name which table is consulted at each of those three points, you understand GWLB routing.

---

## Mechanics

### GENEVE encapsulation and why it's necessary

The whole architecture rests on one requirement: **the appliance must see the packet exactly as it was sent**. Source IP, destination IP, source port, destination port, payload — all unaltered. Otherwise the appliance can't make meaningful security decisions.

But the packet has to travel from the consumer VPC into the security VPC, which may use a different or even overlapping CIDR range. Routing an unmodified packet with a foreign IP into that VPC would be nonsense.

GENEVE solves this by **encapsulation**: the original packet is wrapped intact inside a new outer packet addressed within the security VPC. <cite index="4-1">GWLB encapsulates the original packets using GENEVE, and the GENEVE header uses a Type-Length-Value (TLV) format to store information, using Option Class 0x0108; appliances must decapsulate the TLV pairs to process the original packets.</cite> The appliance strips the outer wrapper, inspects the real packet, then re-wraps it and sends it back.

Key operational facts:

- Protocol **GENEVE**, port **6081**, and <cite index="3-1">security groups for appliance instances must allow UDP traffic on port 6081</cite>
- <cite index="4-1">Target group protocol must be GENEVE on port 6081</cite>; target type is `instance` or `ip`
- <cite index="2-1">A maximum transmission unit (MTU) size of 8500 bytes is supported</cite>

### The GWLB endpoint: what makes it special

📚 The transcript's phrasing — "like a normal VPC interface endpoint with one major difference" — is worth unpacking, because the difference is the entire reason GWLB works.

A standard interface endpoint is a _destination_: you resolve a DNS name to it and send traffic there deliberately. A GWLB endpoint is a **next hop**: <cite index="19-1">traffic to and from a Gateway Load Balancer endpoint is configured using route tables</cite>, and it can be named as the target of a route. That means traffic gets steered through it **without the sender knowing or cooperating** — which is exactly what "transparent" inspection requires.

A structural constraint follows directly: <cite index="19-1">you must create the Gateway Load Balancer endpoint and the application servers in different subnets, which enables you to configure the Gateway Load Balancer endpoint as the next hop in the route table for the application subnet</cite>. 🎯 If they shared a subnet they'd share a route table, and the route pointing at the endpoint would apply to the endpoint's own traffic.

### Ingress routing (gateway route tables)

The inbound leg depends on a feature separate from GWLB itself: **VPC ingress routing**, which lets you associate a route table with the **internet gateway** rather than with a subnet. This is the table drawn top-right in the diagram. It answers the question "as traffic arrives at this VPC from outside, where should it go first?" — and the answer is: to the GWLBE, before it reaches anything else.

Order of operations at the IGW on the inbound leg:

1. IGW performs its normal public→private destination NAT (public IP of the ALB → private IP in `10.16.96.0/20`)
2. _Then_ the ingress route table is consulted against that now-private destination
3. Most specific match (`10.16.96.0/20 → GWLBE2`) wins over `10.16.0.0/16 → LOCAL`

🎯 The NAT happens **before** the ingress route lookup. This is why the ingress route table contains private subnet CIDRs rather than public IPs.

---

## Why GWLB Is Needed — and How It Differs from WAF

📚 _This section is beyond the transcript and is offered for your review — flag if you'd rather it live in a separate note._

### The short version

They inspect different things at different layers, and they are not substitutes.

|                          | **AWS WAF**                                        | **Gateway Load Balancer**                                           |
| ------------------------ | -------------------------------------------------- | ------------------------------------------------------------------- |
| **What it is**           | A managed AWS service (a rule engine)              | A traffic-steering mechanism for _your_ appliances                  |
| **OSI layer**            | Layer 7 — HTTP/HTTPS                               | Layer 3 (packet level)                                              |
| **What it inspects**     | Request URI, headers, body, query strings, cookies | Every IP packet, all ports, all protocols                           |
| **What it understands**  | Web requests                                       | Nothing — it delegates understanding to the appliance               |
| **Who writes the logic** | You (rules) or AWS/vendors (managed rule groups)   | The third-party vendor (Palo Alto, Fortinet, Check Point, etc.)     |
| **Where it attaches**    | CloudFront, ALB, API Gateway, AppSync, Cognito     | A route table, as a next hop                                        |
| **Protocol coverage**    | HTTP/HTTPS only                                    | Any IP traffic — SSH, RDP, SMTP, database protocols, custom TCP/UDP |
| **Traffic direction**    | Inbound requests to the protected resource         | Both directions, inbound and outbound                               |

<cite index="23-1">Unlike traditional network firewalls that operate at the network layer, WAF operates at the application layer (Layer 7), providing granular control over HTTP and HTTPS traffic, using customizable rules that inspect IP addresses, HTTP headers, HTTP body, URI strings, SQL injection patterns, and cross-site scripting.</cite> Source: [AWS WAF FAQs](https://aws.amazon.com/waf/faqs/)

### Scenario: a regional bank moving to AWS

Make it concrete. A bank runs an online banking portal plus a batch settlement system that exchanges files with a clearing house over SFTP.

**Where WAF is the right answer.** The customer-facing portal receives a login POST containing `' OR '1'='1` in the username field. WAF sees the HTTP request, matches the SQL injection managed rule group, and blocks it before the request reaches the ALB targets. GWLB could never do this cleanly — at layer 3 that request is just a few TCP segments carrying opaque bytes, and if the connection is TLS-terminated at the ALB, the packets crossing the wire are encrypted anyway.

**Where WAF cannot help.** Three requirements the bank has that WAF structurally cannot meet:

1. **The SFTP flow to the clearing house isn't HTTP.** WAF has nothing to attach to and no concept of the protocol. The bank still needs those flows inspected for exfiltration.
2. **Compliance mandates a specific vendor.** The bank's regulator-approved control set is written against Palo Alto VM-Series with a particular threat-intel subscription; the audit evidence, rule syntax, and the security team's runbooks all assume it. "We used AWS WAF instead" is not an acceptable answer to the auditor. The transcript makes this exact point: you might have a formal requirement to use those products, or need a specific feature which only one specific vendor can deliver.
3. **Egress inspection.** The bank needs to detect a compromised application instance beaconing out to a command-and-control server. That's outbound traffic on an arbitrary port — WAF isn't in that path at all.

**So the bank runs both.** WAF on the ALB for web-layer attacks; GWLB with Palo Alto appliances for full packet inspection of everything, in both directions. 🎯 Exam framing: **WAF and GWLB are complementary layers of defence in depth, not alternatives.**

### Why not just put the appliance on an EC2 instance inline?

You can, and the transcript's opening covers exactly why it degrades. The failure modes:

- **Scaling mismatch.** Ten app instances become a hundred; you now need proportional appliance capacity, wired in manually.
- **Tight coupling.** Appliance dies → the application behind it is unreachable. The blast radius of a security-tier failure is the application tier.
- **Route table churn.** Without an endpoint that can be a next hop, steering traffic means pointing routes at specific ENIs — which means editing route tables every time the fleet changes.
- **Multi-tenancy is impractical.** Sharing one appliance fleet across many application VPCs requires a stable, VPC-crossing insertion point. That's precisely what GWLBE + PrivateLink provides.

GWLB replaces per-instance wiring with a stable endpoint that never changes, while the fleet behind it scales freely in an Auto Scaling group. Compare to [[Auto Scaling Groups]] and [[Network Load Balancer]] — GWLB's target group model is shared with the rest of the ELB family.

---

## How GWLB Avoids Routing Loops

📚 _This section is beyond the transcript, addressing your specific question about why inspected traffic isn't re-captured and sent back to the GWLBE._

This is the sharpest question you can ask about GWLB, and the answer has **two independent mechanisms** doing the work.

### Mechanism 1 — The GWLB↔GWLBE path is not routed traffic

The critical insight: **the hop from the GWLB endpoint to the GWLB and back is not a VPC route table decision at all.**

<cite index="19-1">Gateway Load Balancers use Gateway Load Balancer endpoints to securely exchange traffic across VPC boundaries. A Gateway Load Balancer endpoint is a VPC endpoint that provides private connectivity between virtual appliances in the service provider VPC and application servers in the service consumer VPC.</cite> This is **PrivateLink**, and it's a bound, bidirectional association — not a routing lookup.

So when the packet comes back from the appliance:

<cite index="13-1">The Gateway Load Balancer removes the GENEVE header and forwards traffic to the appropriate Gateway Load Balancer endpoint — the Gateway Load Balancer uses metadata in the GENEVE headers to find the matching Gateway Load Balancer endpoint the traffic was sourced from.</cite> Source: [AWS Networking Blog — Scaling network traffic inspection using AWS Gateway Load Balancer](https://aws.amazon.com/blogs/networking-and-content-delivery/scaling-network-traffic-inspection-using-aws-gateway-load-balancer/)

The return path is determined by **GENEVE metadata**, not by a route table. The GWLB knows which endpoint the packet came from because that identity was carried in the encapsulation header the whole time. There is no lookup that could send it somewhere wrong.

### Mechanism 2 — The endpoint consults a _different_ route table on the way out

Once the packet is back at the GWLBE, decapsulated and re-exposed to normal VPC routing, it uses the route table **of the GWLBE's own subnet** — and that table has no route pointing at the GWLBE.

Look at the diagram again. The GWLBE subnet's route table is `10.16.0.0/16 → LOCAL`, `0.0.0.0/0 → IGW`. The route that sent traffic _to_ the endpoint lives in a **completely different table** — the IGW's ingress table (inbound) or the ALB subnet's table (outbound). 🎯 **The endpoint is never a target in its own subnet's route table.**

<cite index="13-1">Once at the Gateway Load Balancer endpoint, traffic destined to the Internet uses the route table associated with the Gateway Load Balancer endpoint to egress out the Internet Gateway.</cite>

Trace it with the diagram's numbering:

| Step | Packet at                | Table consulted                   | Result                              |
| ---- | ------------------------ | --------------------------------- | ----------------------------------- |
| 2    | IGW, inbound             | **Ingress (gateway) route table** | `10.16.96.0/20 → GWLBE2`            |
| 3    | GWLBE2                   | _none — PrivateLink_              | → GWLB                              |
| 6    | GWLB, post-inspection    | _none — GENEVE metadata_          | → GWLBE2                            |
| 7    | GWLBE2, now decapsulated | **GWLBE subnet route table**      | `10.16.0.0/16 → LOCAL` → ALB subnet |
| 10   | ALB subnet, outbound     | **ALB subnet route table**        | `0.0.0.0/0 → GWLBE2`                |
| 11   | GWLBE2                   | _none — PrivateLink_              | → GWLB                              |
| 14   | GWLB, post-inspection    | _none — GENEVE metadata_          | → GWLBE2                            |
| 15   | GWLBE2, decapsulated     | **GWLBE subnet route table**      | `0.0.0.0/0 → IGW`                   |

At step 7, the destination is the ALB's private IP — inside the VPC CIDR — so the **LOCAL** route matches and the packet moves to the ALB. At step 15 the destination is the client's public IP, so the **default route to the IGW** matches. In neither case is the GWLBE itself a candidate target, because it doesn't appear in that table.

📚 This is why the subnet-separation requirement exists. If the GWLBE and the ALB shared a subnet, they'd share a route table containing `0.0.0.0/0 → GWLBE`. Post-inspection outbound traffic sitting at the endpoint would consult that table, match the default route, and be sent back to the endpoint — an infinite loop. **The requirement is the loop prevention.**

### Mechanism 3 (bonus) — Flow stickiness and symmetry

Not loop prevention, but it's the other half of "why doesn't this break." Because <cite index="19-1">GWLB maintains flow stickiness to a specific target appliance using 5-tuple (default), 3-tuple, or 2-tuple</cite>, both directions of a conversation land on the same appliance. <cite index="27-1">Because firewalls are stateful appliances, the flow from source to destination and the traffic's return flow must remain on the same firewall appliance.</cite>

🎯 If a stateful firewall saw only the response half of a TCP conversation, it would drop it as an unsolicited packet. Flow stickiness is what makes stateful inspection viable behind a load balancer.

---

## Exam Traps

> [!warning] Trap: GWLB vs NLB Both are low-layer and both use target groups. The distinguishing signal is **third-party appliance** or **transparent inspection**. NLB terminates flows and load balances _to_ targets; GWLB passes packets _through_ targets and gets them back.

> [!warning] Trap: "Which endpoint type can be a route table target?" Gateway endpoints (S3/DynamoDB) are route table targets too, but they use a **prefix list** destination and are not a next hop for arbitrary traffic. The **GWLB endpoint** is the one that can be a next hop for any destination. Interface endpoints cannot be route table targets at all. See [[VPC Endpoints]].

> [!warning] Trap: Where does the GWLB live? The GWLB and its appliances live in the **security/provider VPC**. Only the **endpoint** lives in the consumer VPC. A question describing "load balancer deployed in the application VPC" alongside GWLB is likely wrong.

> [!warning] Trap: GENEVE port **6081/UDP**. If a question shows a security group on the appliances that omits UDP 6081, that's the fault.

> [!warning] Trap: Source IP preservation GWLB preserves original source and destination IP. But <cite index="12-1">Network Load Balancer client IP preservation is not supported when traffic is routed through a Gateway Load Balancer endpoint, even if the target is in the same VPC as the Network Load Balancer</cite>. Source: [PrivateLink — GWLB endpoints](https://docs.aws.amazon.com/vpc/latest/privatelink/gateway-load-balancer-endpoints.html)

> [!warning] Trap: IPv6 with egress-only IGW <cite index="12-1">If you use a Gateway Load Balancer with an egress-only internet gateway, the IPv6 traffic is dropped — use an internet gateway and inbound firewall rules instead.</cite>

> [!warning] Trap: Same-AZ endpoint selection Route to the GWLBE **in the same AZ** as the traffic. Cross-AZ hops cost money and add latency, and the diagram's ingress route table exists specifically to enforce this.

> [!warning] Trap: WAF substitution A question describing SQL injection or XSS protection wants **WAF**, not GWLB. A question describing a third-party firewall vendor, IDS/IPS, non-HTTP protocols, or egress inspection wants **GWLB**.

---

## Quick-Reference Table

| Attribute                 | Value                                                                                                 |
| ------------------------- | ----------------------------------------------------------------------------------------------------- |
| **OSI layer**             | 3 (ELB docs); L3+L4 per Prescriptive Guidance — see ⚠️ above                                           |
| **Protocol to targets**   | GENEVE, UDP **6081**                                                                                  |
| **MTU**                   | 8500 bytes                                                                                            |
| **Flow stickiness**       | 5-tuple (default), 3-tuple, or 2-tuple                                                                |
| **Target types**          | `instance` or `ip` (private IPs)                                                                      |
| **Target group protocol** | GENEVE / port 6081                                                                                    |
| **Listener**              | Single default action, forwards to one target group                                                   |
| **Endpoint type**         | Gateway Load Balancer Endpoint (PrivateLink), usable as route table next hop                          |
| **Subnet constraint**     | GWLBE and application servers must be in **different subnets**                                        |
| **Packet alteration**     | None — original SRC/DST IP and ports preserved                                                        |
| **Access logs**           | Not produced by GWLB (transparent, doesn't terminate flows); enable on appliances / use VPC Flow Logs |
| **Primary use case**      | Scaling third-party inline security appliances                                                        |
| **Typical vendors**       | Palo Alto, Fortinet, Check Point, Cisco, Trend Micro                                                  |

---

## Open Questions

Roger — a few things to decide on:

1. **Ingress routing as a sub-note?** VPC ingress routing (gateway route tables) is doing a lot of work in this architecture and is its own exam-relevant concept — it also applies to Transit Gateway inspection patterns that have nothing to do with GWLB. Worth splitting into `[[VPC Ingress Routing]]`?
    
2. **The WAF comparison section** — I wrote it as a full section with the bank scenario since you asked for it explicitly, but it's arguably a `[[AWS WAF]]` note concern. Happy to trim this to a short pointer and move the depth there if you'd prefer the GWLB note stay tighter.
    
3. **Cross-links to create.** I've referenced `[[Auto Scaling Groups]]`, `[[Network Load Balancer]]`, `[[VPC Endpoints]]`, and `[[AWS WAF]]`. Do `[[VPC Endpoints]]` and `[[AWS WAF]]` exist in the vault yet, or should those be stubs?
    
4. **The L3-vs-L3/L4 discrepancy** — I've flagged it and recommended "Layer 3" as the exam answer, but I haven't seen an actual SAA-C03 question on it. If Cantrill's practice questions treat it as L3/L4, let me know and I'll adjust the recommendation.
    
5. **Transit Gateway inspection pattern** — the centralised-inspection architecture where GWLB sits behind a TGW rather than being reached directly from an IGW is a common real-world (and exam) variant that the transcript doesn't cover. Add it here, or defer to a `[[Transit Gateway]]` note?
    
6. **The "Mechanism 3" heading in the loop-prevention section** is a slight misnomer since flow stickiness isn't loop prevention — I kept it there because it's the natural place a reader would ask about it, but say the word and I'll break it into its own subsection under Mechanics.