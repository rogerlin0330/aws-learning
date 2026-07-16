---
title: Elastic Load Balancer (ELB)
type: service
tags: [elb, alb, nlb, clb, gwlb, load-balancing, networking, ha, scaling, sni, listener-rules, end-to-end-encryption, privatelink]
certs: [aws-saa]
updated: 2026-07-15
---

# Elastic Load Balancer (ELB)

An **Elastic Load Balancer (ELB)** accepts connections from clients and distributes them across registered backend compute. Its core value is **abstraction**: clients are decoupled from the physical infrastructure, so backend capacity can scale up or down, and instances can fail and be replaced, all without the client being aware. This abstraction is exactly what makes [horizontal scaling](../patterns/scaling.md) possible, and it is the standard entry point of the web tier in the [regional architecture model](../patterns/global-regional-architecture.md).

## The ELB Family

"ELB" refers to the whole family, which has three course-covered types split across two versions:

| Version | Type | Layer / Protocols | Notes |
| --- | --- | --- | --- |
| v1 | **Classic Load Balancer (CLB)** | HTTP, HTTPS plus lower-level; *not a true L7 device* | Introduced 2009. Legacy — avoid for new deployments; migrate off. One SSL certificate per CLB. More expensive. |
| v2 | **Application Load Balancer (ALB)** | True Layer 7: HTTP, HTTPS, WebSocket | Preferred for HTTP/S workloads. |
| v2 | **Network Load Balancer (NLB)** | Layer 4: TCP, TLS, UDP | For non-HTTP/S workloads: email, SSH, gaming and other custom protocols. |

v2 load balancers are faster and support **target groups and rules**, so one load balancer can serve many purposes (for example, routing by customer).

> [!tip] Exam Tip
> There is **no scenario** where a v1 CLB is the right answer for a new deployment. The real exam skill is **picking ALB vs NLB** for a given protocol: HTTP/HTTPS/WebSocket means ALB; TCP/TLS/UDP (email, SSH, gaming, anything non-HTTP) means NLB.

There is also a fourth member the course does not cover: the **Gateway Load Balancer (GWLB)**, introduced at re:Invent 2020, which operates at Layer 3 using the **GENEVE protocol on port 6081** and is used to insert fleets of third-party security/inspection appliances (firewalls, IDS/IPS, deep-packet inspection) into the traffic path as a transparent "bump-in-the-wire" ([GWLB docs](https://docs.aws.amazon.com/elasticloadbalancing/latest/gateway/introduction.html)). It appears occasionally in SAA-adjacent questions but is out of scope for this page. The complete family: **ALB (L7), NLB (L4), GWLB (L3), CLB (legacy)**.

## Architecture — It's Nodes, Not a Box

![ELB architecture across two AZs: load balancer nodes in public subnets accept connections from a user via the ELB DNS name, forward to public web EC2 instances, and an internal load balancer with private-only nodes connects the web tier to private app instances](../assets/elb-architecture-part1.png)

What you provision as a "single load balancer object" is actually **multiple load balancer nodes** living in the subnets you selected. When you create a load balancer, you select **one subnet per enabled AZ**. ALB requires at least two AZs; NLB can technically run in one AZ, but is normally deployed across 2+ AZs for HA. In each enabled AZ, ELB deploys **one or more nodes** in the selected subnet.

- A single **A record** is created for the load balancer. It resolves to **all** active nodes (see [DNS](../concepts/dns.md) for record mechanics).
- Across many clients, DNS resolution tends to spread requests across nodes; the effect is statistical rather than per-request.
- Nodes are **highly available** (a failed node is replaced) and **auto-scale** within their subnet as load rises.

> [!note] The node spread is AWS-managed, not user-configurable — and statistical, not guaranteed
> The node spread is delivered at the **DNS layer**: the A record returns the node IPs with a **60-second TTL**, so clients re-resolve as AWS adds, removes, or replaces nodes ([How Elastic Load Balancing works](https://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/how-elastic-load-balancing-works.html)). There is **no setting** for per-node weighting; your only levers are indirect: **which subnets/AZs you enable** (determining where nodes exist at all) and **cross-zone load balancing** (which changes only what a node does with traffic *after* receiving it — node-to-target, never client-to-node). The evenness is **statistical**: it relies on clients resolving fresh and picking among the returned IPs. A client that caches a resolved IP (the JVM's default indefinite DNS cache is the classic case) pins itself to one node for its lifetime — the same connection-level DNS behavior as the [Aurora reader endpoint](aurora.md#how-the-reader-endpoint-balances-load). Across many independent clients it evens out; a few heavy, long-lived clients can skew it.

### Internet-facing vs internal

This choice controls **the IP addressing of the nodes only** — nothing else about the nodes or features differs:

| Scheme | Node IPs | Use |
| --- | --- | --- |
| **Internet-facing** | Public **and** private | Reachable from the internet |
| **Internal** | Private only | Separating application tiers / private routing |

> [!warning] Gotcha — internet-facing does not force public instances
> An **internet-facing load balancer can forward to both public and private [EC2](../services/ec2.md) instances.** Backend targets do **not** need public addressing — the only requirement is that the **load balancer nodes can reach the targets** (via their private IPs). This matters for "how many subnets/tiers are required" questions: an internet-facing load balancer does not force the instances behind it to be public.

### Listeners

Client-side behavior is set by the **listener configuration**: the protocol and port the load balancer accepts on its front side. Nodes then communicate with **targets** on a (possibly different) port and protocol. Listeners get a dedicated lesson later; this page only introduces them.

### Subnet selection rules

The following provisioning rules are verified against current AWS documentation ([ALB docs](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/application-load-balancers.html), [NLB availability zones](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/availability-zones.html), [CLB subnets](https://docs.aws.amazon.com/elasticloadbalancing/latest/classic/elb-manage-subnets.html)):

- You choose **subnets** when provisioning; selecting a subnet is what enables that AZ — "to enable a zone, specify a subnet in the zone," and ELB creates a node in each zone you specify.
- **At most one subnet per AZ** for a single load balancer.
- **Internet-facing** load balancers must use **public subnets** (route to an [IGW](../services/vpc.md)) so nodes can get public IPs; backend **targets can stay in private subnets**.
- **ALB requires at least two subnets in two different AZs.**
- **NLB** can technically be deployed to a **single subnet/AZ** (you enable AZs one subnet at a time at the API level), but the console create flow and AWS best practice still push **two or more AZs** — default to "2+ AZs" on the exam unless a question specifically probes the single-AZ NLB capability.
- **Static IPs are NLB-specific**: an internet-facing NLB supports one **Elastic IP** per subnet/node (or AWS assigns one), and an internal NLB can take an assignable private IP per subnet. **ALB and CLB have no static IPs** — you always use their DNS name.

### IP addressing and subnet sizing

- Load balancer nodes need **8+ free IPs per subnet** to function and scale.
- Strictly, a **/28** ($2^{(32 - 28)}$, i.e., 16 total addresses minus [5 AWS-reserved](../services/vpc.md) leaves 11 free) technically satisfies "8 free."
- AWS **recommends /27 or larger** so the load balancer has room to scale alongside backend instances — the docs say to "verify that each Availability Zone subnet for your load balancer has a CIDR block with at least a /27 bitmask and at least eight free IP addresses per subnet" ([ALB docs](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/application-load-balancers.html), [ELB scaling strategies blog](https://aws.amazon.com/blogs/networking-and-content-delivery/scaling-strategies-for-elastic-load-balancing/)).

> [!warning] Gotcha — /27 vs /28 on the exam
> Both can be "correct" depending on the options shown. If `/27` is an option, pick **/27** (the AWS-recommended value; remember it as the default). If you only see `/28` and not `/27`, then **/28** is probably the intended answer (it does satisfy the 8-free-IP rule).

## Multi-Tier Abstraction

![Three-tier architecture across two AZs: a user reaches an internet-facing load balancer in front of a web-tier Auto Scaling group, web instances reach an internal load balancer in front of an app-tier Auto Scaling group, and app instances reach an Aurora database tier](../assets/elb-architecture-part2.png)

Internal load balancers are what let **each tier scale independently**. In the three-tier picture (WEB, APP, DB across two AZs):

1. The user connects to the **internet-facing load balancer**, which forwards to a web instance in the WEB [Auto Scaling group](ec2-auto-scaling.md).
2. The web instance talks to an **internal load balancer**, which forwards to an app instance in the APP Auto Scaling group — possibly in a different AZ.
3. The app instance talks to the DB tier ([Aurora](aurora.md)) for persistent storage.

Without load balancers, every tier is tightly coupled: the user would talk to a *specific* web instance, which talks to a *specific* app instance, and any failure or scaling event disrupts the chain. Inserting load balancers between tiers **loosens the coupling**: the APP tier can grow from 2 to 4 instances and the WEB tier keeps using it with **no reconfiguration**, because it only ever talks to the internal load balancer's DNS name. This is the same session-independence story as [stateless design in horizontal scaling](../patterns/scaling.md#the-critical-consideration--sessions) — the load balancer abstracts *which* instance serves a request.

## Cross-Zone Load Balancing

### The problem (historic default — disabled)

![ELB without cross-zone load balancing: each node only distributes to instances in its own AZ, so with 4 instances in AZ A and 1 in AZ B, the lone AZ B instance receives 50% of all traffic while each AZ A instance receives 12.5%](../assets/elb-with-cross-zone-load-balancing-disabled.png)

Originally each load balancer node could **only** distribute to instances **in its own AZ**. With 4 instances in AZ A and 1 in AZ B:

- DNS distribution tends to split load across **nodes**: in the idealized two-node example, each node gets 50% (100% divided by 2 nodes).
- Node A splits its 50% across 4 local instances, so each gets 12.5%.
- Node B sends its entire 50% to the single local instance.

The result is badly uneven: one instance carries 50% of all traffic while the others carry 12.5% each.

### The fix — cross-zone load balancing (enabled)

![ELB with cross-zone load balancing enabled: every node can distribute to all registered instances in all AZs, so all 5 instances receive an even 20% share of traffic](../assets/elb-with-cross-zone-load-balancing-enabled.png)

Cross-zone load balancing lets **every node distribute across all registered instances in all AZs**. Node A can send to the AZ B instance and vice versa, producing an even distribution across all 5 instances.

> [!tip] Exam Tip — defaults and triggers
>
> - **ALB:** cross-zone is **enabled by default and cannot be changed at the load balancer level**; target groups can override that inherited setting ([ALB target group attributes](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/edit-target-group-attributes.html#modify-target-group-attributes)).
> - **Classic and others** were historically **disabled** by default.
> - The typical exam scenario is "**uneven load distribution**" across AZs — the answer is to **enable cross-zone load balancing**.

> [!warning] Gotcha — node distribution vs instance distribution
> DNS responses tend to distribute clients across **nodes**; over many clients, that approximates an even split. Cross-zone only changes what each node does **after** it receives its share — whether it can reach instances in other AZs.

## ALB vs NLB — Why v2 Exists: Consolidation

The original CLB pain point that motivated v2: **CLBs do not support SNI** (Server Name Indication, see [TLS/HTTPS](../concepts/tls-https.md)), and **a single CLB cannot hold multiple SSL certificates**. Every unique HTTPS application therefore needs its own CLB — two HTTPS apps (`catagram.io` and `dogagram.io`) means two CLBs; 100 legacy apps means 100 load balancers.

A single ALB can front both applications instead:

- **Listener rules**, each able to carry its own SSL certificate (via SNI), handle HTTPS for both domains.
- **Host-header rules** direct connections to **multiple target groups**, each forwarding to its own Auto Scaling group.

That is a 2:1 consolidation for two apps, and the savings compound dramatically across a large legacy estate. **Consolidation is one of the headline advantages of moving from v1 (CLB) to v2 (ALB).**

## ALB Deep Dive — a True Layer 7 Load Balancer

An ALB listens on **HTTP or HTTPS only**, and because it understands those protocols it can inspect and make decisions on **content type, cookies, custom headers, user location, and application behaviour** — none of which an NLB can see.

> [!warning] Gotcha — HTTP/HTTPS only
> The flip side of being a true L7 device: an ALB understands *only* HTTP/HTTPS. It **cannot** handle other Layer 7 protocols (SMTP, SSH, custom gaming protocols), and it **cannot** listen directly on TCP, UDP, or TLS. A non-web protocol in the scenario rules the ALB out.

### SSL/TLS is always terminated on the ALB

Any incoming HTTPS connection is **terminated on the ALB**, and a **new, separate connection** is made from the ALB to the backend targets (which can itself be HTTPS, but it is still a different connection — hop-by-hop, never unbroken).

> [!warning] Gotcha — no unbroken end-to-end encryption through an ALB
> You **cannot** have an unbroken end-to-end SSL/TLS connection from client through to the application instances with an ALB. This matters to strict security teams and can, on its own, exclude the ALB from consideration. Consequence: **every HTTPS ALB must have an SSL certificate installed on the load balancer**, because the connection terminates there. The canonical unbroken-encryption answer is an [NLB with a TCP listener](#unbroken-end-to-end-encryption-tcp-pass-through).

### Performance and health checks

ALBs are **slower than NLBs** — more layers of the networking stack must be processed, and more layers means more latency. In exchange, ALB health checks are **application-aware**: the ALB can make an actual application-layer request and confirm the app responds correctly, not merely that a network connection succeeds.

### Rules — the core of ALB flexibility

Connections arriving at a listener are handled according to **rules**, processed in **priority order**; the last rule is always the **default rule** (a catch-all), and you add higher-priority rules above it.

Rule **conditions** can match on:

- Host header (which domain)
- HTTP headers
- HTTP request method
- Path pattern (for example `/images` vs `/api`)
- Query string
- **Source IP**

Rule **actions** can:

- **Forward** to a target group
- **Redirect** elsewhere (for example to another domain)
- Return a **fixed HTTP response** (a specific status code)
- **Authenticate** — using **OpenID Connect** or **Amazon Cognito**

Worked examples: a host-header rule forwards `catagram.io` traffic to the Catagram target group; a source-IP rule sends one corporate client (say `1.3.3.7`) to a dedicated target group running an alternative version; a host-header rule for `dogagram.io` **redirects** to `catagram.io` instead of forwarding.

> [!tip] Exam Tip
> Routing decisions based on **path, headers, cookies, host, HTTP redirects, or built-in authentication** are **ALB** territory — an NLB can do none of these.

## NLB Deep Dive — Layer 4

An NLB operates at **Layer 4**, interpreting **TCP, TLS, and UDP** (plus the combined `TCP_UDP` listener). It has **no visibility into HTTP/HTTPS**: it cannot interpret headers or cookies, and has no concept of HTTP session stickiness (which relies on cookies at Layer 7).

- **Extremely fast** — NLBs can handle **millions of requests per second** with roughly **25% of the latency of an ALB** (the course's "about 4 times lower latency" framing; AWS no longer publishes a hard number), because they skip the computationally heavy upper layers.
- **Ideal for non-HTTP(S) protocols** — SMTP email, SSH, game servers on custom protocols, financial applications not using HTTP/HTTPS. Protocol-agnostic at Layer 4.
- **Shallower health checks** — the course frames NLB health checks as ICMP and basic TCP handshaking only, not application-aware. (Current AWS docs actually allow **HTTP/HTTPS health check protocols on NLB target groups** ([NLB health checks](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/target-group-health-checks.html)), but the exam-level contrast stands: the ALB is the application-aware option.)
- **Static IP addresses** — one Elastic IP per subnet/AZ, extremely useful for **whitelisting** through corporate firewalls (see [subnet selection rules](#subnet-selection-rules) above; ALB and CLB have no static IPs).

### Unbroken end-to-end encryption (TCP pass-through)

Because an NLB does not understand HTTP/HTTPS, a **TCP listener** forwards TCP **straight through** to the instances. Anything layered on top of TCP — TLS, and therefore HTTPS — is **never terminated** on the load balancer.

> [!tip] Exam Tip — the canonical end-to-end encryption answer
> **Unbroken, end-to-end encryption from client to instance means an NLB with a TCP listener** — never an ALB. If a question requires forwarding encrypted connections to instances *without terminating them on the load balancer*, the answer is NLB.

> [!warning] Gotcha — TCP listener vs TLS listener
> An NLB *also* supports a **TLS listener**, which **does terminate** TLS on the NLB (and therefore needs a certificate) — useful for TLS offload. The unbroken end-to-end behaviour is specific to the **TCP listener**, where the NLB never decrypts. Don't conflate the two ([NLB listeners](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/load-balancer-listeners.html)).

### PrivateLink

NLBs are the load balancer used with **[AWS PrivateLink](vpc.md)** to expose a service to other VPCs.

> [!tip] Exam Tip
> Any scenario involving **PrivateLink** means **NLB**. This is a common, high-value exam fact.

## The Decision Framework (Memorise This)

Memorise the short list of **reasons to choose an NLB**. If the scenario matches none of them, **default to ALB**.

Choose an **NLB** when you need:

1. **Unbroken / end-to-end encryption** between client and instances (TCP listener pass-through).
2. **Static IPs** for whitelisting.
3. **Absolute best performance** — millions of requests per second, lowest latency.
4. **Protocols that are not HTTP or HTTPS** (SMTP, SSH, game/financial protocols).
5. **PrivateLink** — exposing a service to other VPCs.

Otherwise choose an **ALB**: its Layer 7 features (rules, host/path routing, header/cookie awareness, redirects, authentication, application-aware health checks, certificate consolidation) win in most web scenarios.

> [!warning] Gotcha — you can't have both
> Don't be fooled by "performance-critical" wording *and* "path-based routing" in the same question — path routing forces **ALB**; you cannot have NLB speed and Layer 7 routing on one load balancer.

### ALB vs NLB quick comparison

| Attribute | ALB | NLB |
| --- | --- | --- |
| OSI layer | **Layer 7** (application) | **Layer 4** (transport) |
| Listener protocols | **HTTP, HTTPS** only | **TCP, UDP, TCP_UDP, TLS** |
| Understands HTTP / headers / cookies | Yes | No |
| Host / path / header / query / source-IP routing | Yes (rules) | No |
| SSL/TLS behaviour | **Always terminated** on the LB (cert required) | **TCP listener = pass-through** (unbroken); **TLS listener = terminates** |
| Unbroken end-to-end encryption to instances | No | Yes (TCP listener) |
| Performance / latency | Higher latency | Much lower latency; millions of requests/sec |
| Static IP addresses | No (DNS name only) | Yes (Elastic IP; whitelisting) |
| Health checks | **Application-aware** (L7 request) | Basic (course framing; HTTP/HTTPS checks exist today) |
| Redirects, fixed responses, OIDC/Cognito auth | Yes | No |
| PrivateLink support | No | **Yes** |
| Non-HTTP protocols (SMTP, SSH, game, finance) | No | Yes |
| Certificate consolidation (SNI, many domains) | Yes | N/A (no L7) |

### Course vs current AWS

- **NLBs support security groups since 10 Aug 2023** ([announcement](https://aws.amazon.com/about-aws/whats-new/2023/08/network-load-balancer-supports-security-groups/)). Older material claiming "NLBs have no security group — control access at the instance level" is outdated. Quirk: security groups can only be associated **at NLB creation time**; an NLB created without them cannot have them added later (you would have to replace the NLB).
- NLB target-group **health checks support HTTP/HTTPS protocols today**, not just TCP (noted above).

## Quick Reference

| Topic | Key fact |
| --- | --- |
| Family | CLB (v1), ALB + NLB (v2). Prefer v2 always. |
| ALB | L7; HTTP/HTTPS/WebSocket; cross-zone **on** by default at the LB level; target groups can override |
| NLB | L4; TCP/TLS/UDP; static IPs (Elastic IP); can technically run in 1 AZ |
| CLB | v1 legacy; 1 SSL certificate per LB; avoid |
| Real object | Multiple **nodes** in each chosen subnet, auto-scaling |
| DNS | Single **A record** resolving to **all** nodes |
| Node distribution | Traffic split evenly across nodes: 100% divided by the number of nodes |
| Internet-facing | Nodes get **public + private** IPs; can reach **public and private** targets |
| Internal | Nodes get **private** IPs only; used to separate tiers |
| Listener | Front-side configuration: protocol + port the LB accepts on |
| Subnet count | **One subnet per AZ**, **2+ AZs** (ALB requires it) |
| Subnet size | **8+ free IPs**; recommend **/27 or larger**; /28 is the strict minimum |
| Cross-zone | Fixes uneven load; distributes across **all** instances in **all** AZs |
| ALB SSL/TLS | **Always terminated** on the LB; cert required on the LB; never unbroken end-to-end |
| NLB TCP listener | **Pass-through** — the unbroken end-to-end encryption answer |
| PrivateLink | **NLB only** |
| Decision rule | Memorise the 5 NLB reasons (E2E encryption, static IPs, performance, non-HTTP, PrivateLink); otherwise **ALB** |

## References

- [Scaling — Horizontal vs Vertical](../patterns/scaling.md) — why a load balancer is the price of admission for horizontal scaling
- [Global & Regional AWS Architecture](../patterns/global-regional-architecture.md) — the load balancer as the web-tier regional entry point
- [VPC](vpc.md) — subnets, reserved IPs, public vs private subnet mechanics, PrivateLink
- [Networking](../concepts/networking.md) — ENIs, firewall layers
- [TLS/HTTPS](../concepts/tls-https.md) — certificates, SNI, TLS termination
- AWS docs: [Application Load Balancers](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/application-load-balancers.html) · [NLB availability zones](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/availability-zones.html) · [NLB listeners](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/load-balancer-listeners.html) · [NLB health checks](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/target-group-health-checks.html) · [CLB subnets](https://docs.aws.amazon.com/elasticloadbalancing/latest/classic/elb-manage-subnets.html) · [ALB target group attributes](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/edit-target-group-attributes.html#modify-target-group-attributes) · [ELB scaling strategies blog](https://aws.amazon.com/blogs/networking-and-content-delivery/scaling-strategies-for-elastic-load-balancing/)
- Source: `raw/aws-cert-study-notes/aws-solution-architect/elb_architecture.md` (Adrian Cantrill AWS SAA — "Evolution of the ELB", "ELB Architecture Part 1 & 2"; diagrams © learn.cantrill.io)
- Source: `raw/aws-cert-study-notes/aws-solution-architect/alb_vs_nlb.md` (Adrian Cantrill AWS SAA — "Application Load Balancing (ALB) vs Network Load Balancing (NLB)")
