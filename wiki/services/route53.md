---
title: Route 53
type: service
tags: [dns, route53, networking]
certs: [aws-saa]
updated: 2026-05-07
---

# Route 53

Route 53 is AWS's **authoritative DNS service** and domain registrar. It is **globally resilient** — name servers are distributed across multiple regions and share the same dataset, so the service remains available even during regional failures. See [DNS Concepts](../concepts/dns.md) for the underlying DNS fundamentals.

Route 53 integrates tightly with [TLS/HTTPS](../concepts/tls-https.md) (ACM certificate validation uses DNS TXT records) and with [VPC](vpc.md) (private hosted zones and the built-in Resolver).

## Hosted Zones

A **hosted zone** is a DNS database for a specific domain — the container for all its DNS records.

### Public Hosted Zone

Accessible from the public internet and from any VPC via the Route 53 Resolver.

![Route 53 public hosted zone architecture showing 4 name servers, VPC resolver, and internet access](../assets/R53_public_hosted_zone_infographic.png)

- Route 53 allocates **4 public name servers** per hosted zone
- To make Route 53 authoritative for a domain, update the **NS records at the registrar** (or TLD) to point to these 4 servers
- Works with domains registered externally (GoDaddy, Hover, etc.) — update NS at the registrar, Route 53 handles the rest
- VPC resources resolve public zone records via the **VPC +2 resolver** (see Route 53 Resolver below)

**Cost:** monthly per hosted zone + small per-query fee; Alias records pointing to AWS resources are **free**

![DNS resolution walk for a public hosted zone: client → recursive resolver → root → TLD → Route 53 name servers](../assets/R53_public_hosted_zones.png)

### Private Hosted Zone

Accessible **only from explicitly associated VPCs** — never from the public internet.

![Route 53 private hosted zone architecture showing VPC association and restricted DNS resolution](../assets/R53_private_hosted_zone_infographic.png)

- Must be associated with one or more VPCs (same or different AWS accounts)
- **Cross-account association**: supported, but **CLI/API only** — not available in the AWS Console
- Requires the VPC to have `enableDnsSupport` = `true` (see [VPC](vpc.md#dns-in-vpc))

> [!tip] Exam Tip
> Private hosted zones are the right answer for **internal service discovery** — microservices resolving each other by name within a VPC without exposing anything publicly.

### Split-View (Split-Horizon) DNS

You can create both a **public** and a **private** hosted zone for the **same domain name**:

- Queries from inside an associated VPC → private zone records (internal content)
- Queries from the public internet → public zone records (public content only)

> [!tip] Exam Tip
> If a question describes needing **different DNS answers for the same domain depending on whether the query comes from inside or outside a VPC**, the answer is Split-View DNS.

## Alias Records vs CNAME

The CNAME restriction at the zone apex (naked domain) is a DNS standard limitation. Route 53 solves it with its proprietary **Alias record**.

![CNAME vs Alias record comparison showing apex domain support and AWS resource targets](../assets/R53_CNAME_vs_Alias_infographic.png)

| | CNAME | Alias |
| --- | --- | --- |
| Works at zone apex? | No | Yes |
| Standard DNS? | Yes | No — Route 53 proprietary |
| Points to | Any domain name | AWS resource DNS name |
| Query charges | Normal TTL-based billing | **Free** for AWS resource targets |
| TTL | Set on the record | Managed by Route 53 |

**Alias type must match the target resource's record type:**

| Target resource | Alias type |
| --- | --- |
| ELB | A Alias |
| CloudFront distribution | A Alias |
| S3 static website endpoint | A Alias |
| API Gateway | A Alias |
| Global Accelerator | A Alias |
| Elastic Beanstalk | A Alias |
| Another Route 53 record | Match the target record type |

> [!warning] Gotcha
> Alias type and target record type must match — e.g. ELBs return A records, so create an **A Alias**, not a CNAME Alias.

> [!tip] Exam Tip
> **Always prefer Alias over CNAME** for AWS resource targets: works at apex, free, integrates with Route 53 health checks.

## Routing Policies

Route 53 can make routing decisions based on health, geography, and load — going well beyond basic DNS.

| Policy | Behaviour |
| --- | --- |
| Simple | Single record; no health checks |
| Weighted | Split traffic by percentage (A/B testing, gradual migration) |
| Latency | Route to the AWS region with lowest latency for the client |
| Failover | Active/passive; switch to secondary when primary health check fails |
| Geolocation | Route based on client's country/continent |
| Geoproximity | Route based on geographic distance; adjustable bias |
| Multi-value | Return multiple IPs; basic load distribution (not a true load balancer) |

> [!tip] Exam Tip
> **Failover** requires a Route 53 **health check** on the primary record. Health checks can monitor an endpoint (HTTP/HTTPS/TCP), a CloudWatch alarm, or other health checks (calculated health check).

## Route 53 Resolver

Every VPC has a built-in DNS resolver reachable at **VPC base IP + 2** (e.g. `10.0.0.2` for a `10.0.0.0/16` VPC). It handles:

- Local VPC hostnames (`ec2-….compute.amazonaws.com`)
- Records in associated private hosted zones
- Recursive public DNS lookups (root → TLD → authoritative)

For hybrid environments, **Resolver Endpoints** (inbound/outbound) allow on-premises networks to resolve Route 53 private zones, and VPCs to resolve on-premises DNS.

## References

- [DNS Concepts](../concepts/dns.md)
- [VPC](vpc.md)
- [TLS/HTTPS](../concepts/tls-https.md)
- Source: `raw/aws-cert-study-notes/aws-solution-architect/route53_public_hosted_zone.md`
- Source: `raw/aws-cert-study-notes/aws-solution-architect/route53_private_hosted_zone.md`
- Source: `raw/aws-cert-study-notes/aws-solution-architect/route53_alias_vs_cname.md`
