---
tags:
  - aws
  - saa
  - elb
  - load-balancing
  - networking course: Adrian Cantrill AWS SAA
---

# Elastic Load Balancer Architecture

## Overview

An **Elastic Load Balancer (ELB)** accepts connections from clients and distributes them across registered backend compute. Its core value is **abstraction**: clients are decoupled from the physical infrastructure, so backend capacity can scale up or down, and instances can fail and be replaced, all without the client being aware.

The ELB family has three types split across two versions:

|Version|Type|Layer / Protocols|Notes|
|---|---|---|---|
|v1|**Classic Load Balancer (CLB)**|HTTP, HTTPS + lower-level; _not a true L7 device_|Introduced 2009. Avoid — migrate off. One SSL cert per CLB. More expensive.|
|v2|**Application Load Balancer (ALB)**|True Layer 7: HTTP, HTTPS, WebSocket|Preferred for HTTP/S workloads.|
|v2|**Network Load Balancer (NLB)**|Layer 4: TCP, TLS, UDP|For non-HTTP/S: email, SSH, gaming/custom protocols.|

🎯 **Exam framing:** "ELB" = the whole family. v2 load balancers are faster and support **target groups + rules** (one LB for many things / route by customer). There is **no scenario** where you'd pick a v1 CLB for a new deployment — the real exam skill is **picking ALB vs NLB** for a given protocol.

---

## Diagram Walkthrough — ELB Architecture (Part 1)

![[elb-architecture-part1.png]]

> [!note] Reading the diagram The **green and blue rectangles are subnets**, arranged as two rows: **AZ A** (top) and **AZ B** (bottom). Green = public (web tier) subnets; blue = private (app/internal) subnets.

**Left context (Bob → the LB):** The user connects to the ELB's DNS name, which resolves to the **LB nodes** (small tree icons) sitting inside subnets. Two callouts anchor the whole picture:

- _Configured to run in 2+ AZs. 1+ nodes are placed into a subnet in each AZ and scale with load._
- _Each ELB is configured with an (A) record DNS name. This resolves to the ELB nodes._

**Center (the nodes and their addressing):** The large central node icon marks the two addressing choices:

- 🎯 **Internet-facing** → nodes get **public _and_ private** IPs.
- 🎯 **Internal** → nodes get **private IPs only**. This is the _only_ architectural difference between the two schemes — everything else (nodes, features) is identical.

**Main objects (node → target):** A node in the green (public) subnet talks to a **Public EC2 (Web)** instance; the orange listener/target flow shows nodes communicating with targets. Two teaching callouts:

- _Nodes are configured with **listeners** which accept traffic on a **port & protocol** and communicate with **targets** on a port and protocol._
- ⚠️ _**Internet-facing LB nodes can access public **and** private EC2 instances**_ — targets do **not** need public IPs.

**Right side (internal tier):** A second (internal) LB with private-only nodes sits between tiers — _internal load balancers can be used to allow scaling between application tiers_, reaching the **Private EC2 (APP)** instance.

**Footer sizing callout:** _8+ free IPs per subnet and a **/27 or larger** subnet to allow for scale._

---

## Core Mechanics

### It's nodes, not a box

What you provision as a "single load balancer object" is actually **multiple LB nodes** living in the subnets you selected. When you create the LB you pick **one subnet in two or more AZs** — the product deploys **one or more nodes per chosen subnet**.

- A single **A record** is created for the LB. It resolves to **all** active nodes.
- Incoming requests are **spread equally across nodes**.
- Nodes are **highly available** (a failed node is replaced) and **auto-scale** within their subnet as load rises.

### Internet-facing vs internal

🎯 This choice controls **IP addressing of the nodes only** — nothing else:

|Scheme|Node IPs|Use|
|---|---|---|
|**Internet-facing**|Public **+** private|Reachable from the internet|
|**Internal**|Private only|Separate app tiers / private routing|

⚠️ **Exam trap:** An **internet-facing LB can forward to _both_ public and private EC2 instances.** Backend instances do **not** need public addressing. The only requirement is that **LB nodes can reach the targets**. This matters for "how many subnets/tiers are required" questions — an internet-facing LB does **not** force public instances.

### Listeners

Client-side behavior is set by the **listener configuration** — the protocols/ports the LB accepts on its front side. (Dedicated lesson later; introduced here only.)

### IP addressing / subnet sizing

- LB nodes need **8+ free IPs per subnet** to function and scale.
- Strictly, a **/28** (16 total − 5 AWS reserved = 11 free) technically satisfies "8 free."
- AWS **recommends /27 or larger** so the LB has room to scale alongside backend instances.

⚠️ **Exam gotcha on subnet size:** Both `/28` and `/27` can be "correct" depending on the options shown:

- If `/27` is an option → pick **/27** (the AWS-recommended value; remember this as the default).
- If you only see `/28` and **not** `/27` → **/28** is probably the intended answer.

📚 _AWS docs confirm:_ "verify that each Availability Zone subnet for your load balancer has a CIDR block with at least a /27 bitmask … and at least eight free IP addresses per subnet." (For internet-facing NLB the 8-free-IP rule is also stated explicitly.) Source: [ALB docs](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/application-load-balancers.html), [ELB scaling strategies blog](https://aws.amazon.com/blogs/networking-and-content-delivery/scaling-strategies-for-elastic-load-balancing/).

---

## Diagram Walkthrough — Multi-Tier Abstraction (Part 2)

![[elb-architecture-part2.png]]

**Context bar:** Three tiers laid across two AZs — **WEB** (green), **APP** (blue), **DB** (blue) — with **WEB ASG** and **APP ASG** dotted boxes spanning both AZs.

**Reading order (left → right):**

1. **Bob → internet-facing LB** (public/web subnet) → a **web instance** in the WEB ASG.
2. Web instance → **internal LB** → an **app instance** in the APP ASG (note the arrow crossing AZs — the app instance it lands on is in AZ B).
3. App instance → **DB tier** (Aurora) for persistent storage.

**Footer teaching point:** _LBs allow each tier to scale independently._ Without LBs every tier is tightly coupled — Bob would talk to a _specific_ web instance, which talks to a _specific_ app instance, and any failure/scaling disrupts the chain. Inserting LBs between tiers **loosens coupling**: e.g. the APP tier can grow from 2→4 instances and the WEB tier keeps using it with **no reconfiguration**, because it only ever talks to the internal LB's DNS name.

---

## Cross-Zone Load Balancing

### The problem (historic default — disabled)

![[elb-with-cross-zone-load-balancing-disabled.png]]

Originally each LB node could **only** distribute to instances **in its own AZ**. With **4 instances in AZ A** and **1 in AZ B**:

- DNS spreads load evenly across **nodes**: each node gets **50%** (100% ÷ 2 nodes).
- Node A splits its 50% across **4** local instances → **12.5% each**.
- Node B sends its 50% to the **single** local instance → **50%**.

➡️ Result: badly uneven — one instance carries 50% while others carry 12.5% each.

### The fix — Cross-Zone Load Balancing (enabled)

![[elb-with-cross-zone-load-balancing-enabled.png]]

Cross-zone LB lets **every node distribute across _all_ registered instances in _all_ AZs**. Node A can send to the AZ B instance and vice versa → even distribution across all 5 instances.

🎯 **Exam triggers & defaults:**

- **ALB:** cross-zone is **enabled by default and cannot be turned off** at the LB level. 📚 _(AWS docs: "With Application Load Balancers, cross-zone load balancing is on by default and cannot be changed at the load balancer level.")_
- Classic/others historically **disabled** by default.
- ⚠️ Typical exam scenario: **"uneven load distribution"** → the answer is to **enable cross-zone load balancing**.

> [!note] Node distribution vs instance distribution DNS **always** splits traffic evenly across **nodes** (100% ÷ number of nodes). Cross-zone only changes what each node does **after** it receives its share — whether it can reach instances in other AZs.

---

## Verified Against AWS Docs — Subnet Selection Rules

You asked me to confirm the subnet-provisioning claims (last screenshot). I checked the current AWS documentation — **all points are accurate**, with two refinements worth noting for exam precision:

|Claim (from screenshot)|Verdict|AWS docs detail|
|---|---|---|
|You must choose subnets when provisioning an ELB|✅ Correct|Subnet selection defines which AZs the LB uses and where nodes deploy.|
|Selecting a subnet enables/determines the AZ|✅ Correct|"To enable a zone, specify a subnet in the zone. ELB creates a load balancer node in each zone that you specify."|
|**One subnet per AZ** for a single LB|✅ Correct|"You can add at most one subnet per Availability Zone."|
|Internet-facing → must use **public** subnets (route to IGW) so nodes get public IPs|✅ Correct|Internet-facing requires public subnets; nodes then receive public IPv4 addresses.|
|Backend **targets can stay in private subnets**|✅ Correct|Confirms the exam trap above — targets need not be public.|
|**ALB:** ≥ 2 subnets in 2 different AZs|✅ Correct|"You must select at least two Availability Zone subnets… each subnet must be from a different Availability Zone."|
|**NLB:** can deploy in a **single subnet/AZ**; multi-AZ is best practice; can assign a **static Elastic IP** per subnet/node|✅ Correct|NLB enables per-AZ by specifying one subnet; internet-facing NLB supports specifying an **EIP** (or AWS assigns one) for static IPs.|

⚠️ **Refinement 1 — "NLBs allow one AZ":** True at the API level (you enable AZs one subnet at a time), but the **console create flow and AWS best practice still push ≥ 2 AZs**. For the SAA exam, default to **"2+ AZs"** unless a question specifically probes the single-AZ NLB capability.

⚠️ **Refinement 2 — static IPs are NLB-specific:** Only **NLB** offers static IPs (Elastic IP per AZ for internet-facing; assignable private IP per subnet for internal). **ALB/CLB do not** — you use their DNS name. Don't generalize the screenshot's Elastic-IP point to ALB.

📚 _Sources:_ [ALB — Application Load Balancers](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/application-load-balancers.html) · [NLB — Update Availability Zones](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/availability-zones.html) · [CLB — Configure subnets](https://docs.aws.amazon.com/elasticloadbalancing/latest/classic/elb-manage-subnets.html)

---

## Quick-Reference

|Topic|Key fact|
|---|---|
|Family|CLB (v1), ALB + NLB (v2). Prefer v2 always.|
|ALB|L7 · HTTP/HTTPS/WebSocket · cross-zone **on** & unchangeable|
|NLB|L4 · TCP/TLS/UDP · static IPs (EIP) · can technically be 1 AZ|
|CLB|L1 legacy · 1 SSL cert per LB · avoid|
|Real object|Multiple **nodes** in each chosen subnet, auto-scaling|
|DNS|Single **A record** → resolves to **all** nodes|
|Node distribution|Traffic split evenly across nodes = 100% ÷ #nodes|
|Internet-facing|Nodes get **public + private** IPs; can reach **public _and_ private** targets|
|Internal|Nodes get **private** IPs only; used to separate tiers|
|Listener|Front-side config: protocol + port the LB accepts on|
|Subnet count|**One subnet per AZ**, **2+ AZs** (ALB requires it)|
|Subnet size|**8+ free IPs**; recommend **/27+**; /28 is the strict minimum|
|Cross-zone|Fixes uneven load; distributes across **all** instances in **all** AZs|

---

## Open Questions / Review Points for Roger

1. **NLB single-AZ:** I framed this as "technically allowed, but default to 2+ AZs for the exam." Confirm you want that emphasis rather than a flat "2+ AZs."
2. **Gateway Load Balancer (GWLB):** The transcript predates/omits GWLB (the _fourth_ ELB type, for L3 traffic inspection appliances). It's occasionally SAA-adjacent. Want me to add a one-line 📚 note, or keep the note scoped to the three types the course covers?
3. **Elastic IP / static-IP callout:** I flagged that static IPs are NLB-only (screenshot could be read as generalizing). Keep the ⚠️ refinement or trim it?

---

_Source: Adrian Cantrill AWS SAA — "Evolution of the ELB", "ELB Architecture Part 1 & 2". Diagrams © learn.cantrill.io. AWS doc cross-references verified against current documentation._