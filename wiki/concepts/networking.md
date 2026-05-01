---
title: Networking Concepts
type: concept
certs: [aws-saa]
updated: 2026-05-01
---

# Networking Concepts

Cross-cutting networking ideas that span multiple AWS services. The foundational service is [VPC](../services/vpc.md); [EC2](../services/ec2.md) instances (and other resources) run inside it.

## OSI Layers of AWS Network Controls

Understanding OSI layers clarifies *what each control can and cannot do*:

| AWS Construct | OSI Layer | Attaches to | Notes |
| --- | --- | --- | --- |
| Route table | L3 only | Subnet | IP destinations and next hops; no port/protocol awareness |
| NACL | L3/4 | Subnet (L3 construct) | Stateless; ICMP rules = L3, TCP/UDP rules = L4 |
| Security Group | L3/4 | ENI (L3/4 construct) | Stateful; same filter criteria as NACL |
| WAF | L7 | ALB / CloudFront / API GW | Inspects HTTP content: headers, URLs, body |

Both [NACLs](../services/vpc.md#network-acl-nacl) and [Security Groups](../services/vpc.md#security-groups-sg) operate at L3/4 — neither inspects application-layer content. The real distinction between them is **stateful vs stateless** and **where they attach** (subnet vs ENI), not OSI layer.

## Firewall Layers

AWS networking enforces security at two complementary layers. See [Identity and Access Concepts](identity-and-access.md) for the third layer (IAM) which sits above the network entirely.

| | NACL | Security Group |
| --- | --- | --- |
| OSI layer | 3/4 | 3/4 |
| Attaches to | Subnet | ENI |
| Stateful? | No | Yes |
| Explicit Deny? | Yes | No |

Best practice: use [Security Groups](../services/vpc.md#security-groups-sg) as the primary control; add [NACLs](../services/vpc.md#network-acl-nacl) only when you need explicit IP blocking.

## Routing Hierarchy

Traffic flows through these components in order:

1. **Security Group** — outbound check on the source ENI (L3/4, stateful)
2. **Subnet Route Table** — determines next hop (L3 only)
3. **VPC Router** — moves traffic between subnets or to a gateway (L3)
4. **NACL** — inbound check on the destination subnet boundary (L3/4, stateless)
5. **Security Group** — inbound check on the destination ENI (L3/4, stateful)

## Internet Connectivity Patterns

| Scenario | Solution |
| --- | --- |
| Private subnet → internet (IPv4) | NAT GW in public subnet + route `0.0.0.0/0 → NAT GW` |
| Public subnet → internet (IPv4) | IGW + route `0.0.0.0/0 → IGW` |
| IPv6 bidirectional | IGW + route `::/0 → IGW` |
| IPv6 outbound only | Egress-Only IGW + route `::/0 → EIGW` |
| Private → AWS services (no internet) | VPC Endpoint (PrivateLink or Gateway) |

See [VPC](../services/vpc.md) for the full details on each of these components.

## Ephemeral Ports

When a client connects to a server, the OS assigns a random **ephemeral (source) port** for the return traffic — typically in the range **1024–65535**.

[NACLs](../services/vpc.md#network-acl-nacl) are stateless, so **both directions must be explicitly allowed**:

- Inbound rule: allow destination port (e.g. 443)
- Outbound rule: allow ephemeral port range (1024–65535) back to the client

[Security Groups](../services/vpc.md#security-groups-sg) handle this automatically (stateful).

## IP Address Types

| Type | Scope | Configured in OS? |
| --- | --- | --- |
| Private IPv4 | Within VPC | Yes |
| Public IPv4 | Internet-routable | No — mapped at IGW |
| Elastic IP (EIP) | Static public IPv4 | No — mapped at IGW |
| IPv6 | Globally unique, public | Yes |

> [!tip] Exam Tip
> A public IPv4 address is **not visible inside the EC2 OS** — the OS only sees the private IP. The IGW performs the 1:1 NAT translation. IPv6 is different: it's configured directly in the OS.

## References

- [VPC](../services/vpc.md)
- [EC2](../services/ec2.md)
- [Identity and Access Concepts](identity-and-access.md)
