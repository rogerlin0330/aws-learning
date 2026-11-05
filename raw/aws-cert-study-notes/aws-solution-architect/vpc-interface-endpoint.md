---
tags: [aws, saa-c03, networking, vpc, vpc-endpoints, privatelink, dns]
course: Cantrill SAA-C03
lesson: VPC Endpoints – Interface
related: ["[[VPC Gateway Endpoints]]", "[[Route 53]]", "[[Security Groups]]"]
---

# VPC Interface Endpoints

## Overview

Interface endpoints give **private resources private access to AWS public-zone services** (e.g. SNS, SQS) — same goal as gateway endpoints, completely different mechanism.

| | Gateway endpoint | Interface endpoint |
|---|---|---|
| What it is | Logical VPC gateway object | **ENI** with a private IP in a specific subnet |
| How traffic reaches it | **Routing** — prefix list added to route tables | **DNS** — a name resolves to the ENI's private IP |
| Under the hood | — | **AWS PrivateLink** |

Key consequences of "it's just an ENI in a subnet":

- **One subnet = one AZ → one endpoint ENI = one AZ.** Not HA by default; deploy one per AZ you use (2 AZs → 2 ENIs, 3 AZs → 3).
- **Security groups apply** (impossible with gateway endpoints).
- **Endpoint policies** also supported, same as gateway endpoints — restrict *what* can be accessed through the endpoint.

### PrivateLink

The technology that **injects a service's network interfaces into your VPC subnets** — either an AWS service (that's what an interface endpoint is) or a **third-party/partner service**. Useful in regulated environments: consume third-party services from fully private VPCs with no IGW, NAT, or peering.

---

## Diagram Walkthrough

### 1. Without interface endpoints — the problem

![[interface-endpoints-without.png]]

Read top-to-bottom, left-to-right: a VPC across **AZ-A / AZ-B**, each with a private (blue) and public (green) subnet; VPC router in the centre; IGW on the VPC edge; SNS outside the VPC (public AWS zone).

1. **Name resolution (pink/red labels at top)** — *every* instance, public or private, resolves the **default service name** `sns.<region>.amazonaws.com` to a **public** IP. DNS itself succeeds for the private instance too.
2. **Public instances (red arrows)** → VPC router → IGW → SNS. Works, because they have public IPv4 addresses the IGW can translate.
3. **Private instance (green dashed arrow)** → VPC router → reaches the IGW but **stops there**: no public IP, so the IGW has nothing to translate. Hence "No Access via IGW".

> The failure is **connectivity, not DNS** — this framing matters for diagram 2, because the fix *is* DNS.

### 2. With an interface endpoint & Private DNS — the fix

![[interface-endpoints-with-private-dns.png]]

Same VPC, but an **interface endpoint ENI** now sits in the AZ-B public subnet, and the IGW is **off the traffic path** entirely.

1. **Header — "w/ Private DNS: `sns.<region>.amazonaws.com`"**: with Private DNS on, the *default* service name now resolves to the **endpoint's private IP**.
2. **Footer — "Endpoint-specific DNS: `vpce-…​.sns.<region>.vpce.amazonaws.com`"**: the alternative without Private DNS — apps must be reconfigured to use this name.
3. **Pink path (private instance, AZ-A)** → VPC router → endpoint ENI in AZ-B → SNS. Works with only private addressing.
4. **Blue path (public instance)** → also goes via the endpoint, *not* the IGW — Private DNS changes resolution for **everything in the VPC**, not just private instances.
5. **Dashed black link (endpoint ↔ SNS)** — the PrivateLink connection; traffic never touches the public internet path.
6. **Router's role** — it's only the normal VPC **local route** delivering packets to a private IP in the VPC; no special route entry exists (contrast with the gateway endpoint's prefix list).
7. **Blue-subnet callout — "Not Highly Available by default"**: the diagram deliberately shows **one** ENI in AZ-B. If AZ-B fails, AZ-A instances lose SNS access → add an ENI in AZ-A too.

> Example names above (SNS, `vpce-123-xyz`) illustrate the *shape* of each DNS name — not values to memorize.

---

## Mechanics: DNS Names

Creating an interface endpoint produces several DNS names:

| Name | Resolves to | Use |
|---|---|---|
| **Regional** (endpoint-specific) | ENI IPs in *any* AZ | Simplicity + HA; one name everywhere |
| **Zonal** (endpoint-specific, one per AZ) | The ENI in **that one AZ** | Pin traffic to a specific AZ |
| **Private DNS** (default service name, e.g. `sns.<region>.amazonaws.com`) | ENI private IPs | **Unmodified apps/SDKs** use the endpoint automatically |

**How Private DNS works:** AWS associates a (hidden, AWS-managed) **Route 53 private hosted zone** with the VPC that overrides the default service name to point at the endpoint ENI IPs.

### Traffic outcomes (exam logic)

| Configuration | App uses default name | App uses endpoint-specific name |
|---|---|---|
| No interface endpoint | Public IP → IGW (private instances **fail**) | n/a |
| Endpoint, **Private DNS off** | Still public IP → IGW (unchanged) | Endpoint private IP ✅ |
| Endpoint, **Private DNS on** | Endpoint private IP ✅ | Endpoint private IP ✅ |

- 📚 Private DNS requires the VPC attributes **DNS hostnames** and **DNS resolution** to both be enabled. ([AWS docs](https://docs.aws.amazon.com/vpc/latest/privatelink/vpce-interface.html))
- 📚 By default the endpoint gets the VPC's **default security group**; the SG must allow **inbound HTTPS (443)** from clients for AWS service APIs. ([AWS docs](https://docs.aws.amazon.com/vpc/latest/privatelink/vpce-interface.html))
- 📚 **Cost:** interface endpoints are billed **per hour per AZ + per GB processed**; gateway endpoints are free. ([AWS docs](https://docs.aws.amazon.com/vpc/latest/privatelink/privatelink-access-aws-services.html))

---

## ⚠️ Course vs Reality

| Course claim | Current reality (verified Nov 2026) | Exam note |
|---|---|---|
| DynamoDB is **gateway-only** | DynamoDB supports interface endpoints (PrivateLink) since **Mar 2024** ([What's New](https://aws.amazon.com/about-aws/whats-new/2024/03/amazon-dynamodb-aws-privatelink/)) | Older questions may assume gateway-only. Gateway remains the free/default answer for in-VPC access; interface fits on-prem/cross-region access |
| S3 "recently" supports interface endpoints | Confirmed — both types available for S3 | See S3 Private DNS caveat below |
| Interface endpoints are **IPv4 only** | **IPv6 and dual-stack** supported since **May 2022** ([What's New](https://aws.amazon.com/about-aws/whats-new/2022/05/aws-privatelink-ipv6)) | Course flagged this as a future exam item — it's now outdated; don't pick "IPv4-only" as a limitation |
| Interface endpoints are **TCP only** | PrivateLink supports **UDP** since **Oct 2024** (endpoint services behind NLB) ([What's New](https://aws.amazon.com/about-aws/whats-new/2024/10/aws-udp-privatelink-dual-stack-network-load-balancers)); AWS docs now describe interface endpoints as carrying "TCP or UDP" ([docs](https://docs.aws.amazon.com/vpc/latest/privatelink/concepts.html)) | AWS service APIs are HTTPS anyway, so this rarely matters for AWS-service endpoints |
| Private DNS is **enabled by default** | True for most services — **except S3**, where you must enable it yourself ([AWS docs](https://docs.aws.amazon.com/vpc/latest/privatelink/vpce-interface.html)) | — |

### 📚 S3: gateway + interface coexistence

S3 Private DNS (since 2023) adds an **"Enable private DNS only for inbound endpoint"** option, selected by default when you enable DNS names for S3. It **requires an existing S3 gateway endpoint**: in-VPC clients keep resolving to public S3 IPs → free gateway endpoint; on-prem clients (via Route 53 Resolver **inbound endpoints**) → interface endpoint. ([AWS Storage Blog](https://aws.amazon.com/blogs/storage/introducing-private-dns-support-for-amazon-s3-with-aws-privatelink/))

---

## 🎯 Exam Traps

| Trigger in question | Answer / trap |
|---|---|
| "Highly available" + interface endpoint | Need **one endpoint ENI per AZ** used; a single endpoint is a single-AZ dependency |
| "Control access to the endpoint with security groups" | **Interface** endpoint only — gateway endpoints can't have SGs |
| "Without modifying the application" | Gateway endpoint (routing) **or** interface endpoint **with Private DNS** |
| Interface endpoint exists, private DNS off, app unchanged, still fails / still uses IGW | App is still resolving the **default public name** → enable Private DNS or switch to endpoint-specific DNS |
| "Route table entry for the endpoint" | Gateway endpoints only (prefix list). Interface endpoints **don't use routing** — they use DNS |
| "Expose a third-party / SaaS / partner service privately into a VPC" | **PrivateLink** |
| "Access S3/DynamoDB from on-premises over DX/VPN privately" | **Interface** endpoint (gateway endpoints aren't reachable from on-prem) 📚 |
| "Cheapest private S3/DynamoDB access from inside the VPC" | **Gateway** endpoint (free) 📚 |

---

## Quick Reference: Gateway vs Interface

| Attribute | Gateway endpoint | Interface endpoint |
|---|---|---|
| Services | S3, DynamoDB | Most AWS services incl. S3 and (now) DynamoDB, plus PrivateLink partner/custom services |
| Mechanism | Prefix list in **route tables** | **DNS** → ENI private IP |
| Placement | VPC-level, associated with route tables | ENI in a subnet (**one per AZ**) |
| High availability | **HA by design** | **Not HA by default** — deploy in multiple AZs |
| Security groups | ❌ | ✅ |
| Endpoint policies | ✅ | ✅ (if service supports them) |
| App changes needed | Never | None with Private DNS; otherwise use endpoint-specific names |
| Reachable from on-prem / peered VPCs | ❌ 📚 | ✅ 📚 |
| Cost | Free 📚 | Hourly per AZ + per GB 📚 |

---

## Open Questions (for Roger)

1. Should this link to an existing **[[VPC Gateway Endpoints]]** note under a different title in your vault? I've assumed that name.
2. The 📚 on-prem/peered-VPC and cost rows go beyond the transcript — keep them in the comparison table, or move them to a separate "beyond the course" callout?
3. Want a stub **[[AWS PrivateLink]]** note now, or wait for the dedicated PrivateLink lesson later in the course?
