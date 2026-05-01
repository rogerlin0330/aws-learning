---
title: AWS Solutions Architect Associate — Exam Notes
type: certification
tags: [iam, vpc, networking, ec2]
certs: [aws-saa]
updated: 2026-05-01
---

# AWS Solutions Architect Associate (SAA-C03)

A running collection of exam tips, gotchas, and comparison tables. Deep-dive pages: [IAM](../services/iam.md) · [VPC](../services/vpc.md) · [EC2](../services/ec2.md) · [Networking](../concepts/networking.md) · [Identity & Access](../concepts/identity-and-access.md).

## IAM

See [IAM](../services/iam.md) and [Identity and Access Concepts](../concepts/identity-and-access.md) for full details.

- **Deny overrides Allow**: explicit Deny in any policy wins over any Allow — across all policy sources
- Managed policies > inline policies: prefer managed for reusability and lower overhead
- IAM Users = long-term credentials; [IAM Roles](../services/iam.md#iam-users) = temporary credentials — always use roles for [EC2](../services/ec2.md), Lambda, cross-account access
- IAM is **global** — not region-specific

## VPC & Networking

See [VPC](../services/vpc.md) and [Networking Concepts](../concepts/networking.md) for full details.

### Firewall Rules

- Both [NACL](../services/vpc.md#network-acl-nacl) and [Security Group](../services/vpc.md#security-groups-sg) operate at **Layer 3/4** — not Layer 7; they filter by IP, protocol, and port only
- ICMP rules = Layer 3 only (no ports); TCP/UDP rules = Layer 4
- Layer 7 (HTTP content, headers, URLs) is handled by **WAF**, not NACL or SG
- **NACL** = stateless, attaches to **subnets** (L3 constructs), supports explicit Deny → use to block bad IPs
- **Security Group** = stateful, attaches to **ENIs** (L3/4 constructs), no explicit Deny → use for fine-grained per-resource control
- Because NACLs are stateless: always allow **ephemeral ports (1024–65535)** for return traffic
- The real distinction between NACL and SG is **stateful vs stateless** + **attachment point**, not OSI layer

### IGW vs NAT GW

| | IGW | NAT GW |
| --- | --- | --- |
| Direction | Bidirectional (public subnet) | Outbound only (private subnet) |
| IP mapping | 1:1 (private ↔ Elastic IP) | Many-to-one |
| Resilience | Regional | Zonal or Regional (see below) |
| IPv6 support | Yes | No |

### Zonal vs Regional NAT GW (availability mode — as of Nov 2025)

| | Zonal NAT GW | Regional NAT GW (RNAT) |
| --- | --- | --- |
| Scope | Single AZ | VPC-level |
| Public subnet required? | Yes | No |
| High availability | Manual — one per AZ, per-AZ routes | Automatic — single ID, auto-expands |
| Private NAT support | Yes | No |
| Use when | Private NAT; strict per-AZ IP control | Most public egress use cases |

> [!tip] Exam Tip
> Pre-Nov 2025 courses only cover **zonal** NAT GW. For region resilience with zonal: deploy one per AZ, configure each AZ's route table to its local NAT GW. With **regional** NAT GW, a single ID covers all AZs automatically — no public subnet needed.

### NAT GW vs NAT Instance

- **NAT GW**: AWS-managed, no SG support, charged by duration + data
- **NAT Instance**: self-managed, supports SG, cheaper at high data volumes — must disable **source/destination checks**

### IPv4 vs IPv6

- Public IPv4 is **not configured in the OS** — only visible at the IGW level
- IPv6 is configured directly in the OS (all IPv6 is publicly routable in AWS)
- IPv6 outbound-only: use **Egress-Only IGW**, not NAT GW (NAT GW doesn't support IPv6)

### VPC Design

- Default VPC CIDR: `172.31.0.0/16` — avoid in production
- 5 reserved IPs per subnet: network, router (+1), DNS (+2), reserved (+3), broadcast (last)
- `enableDnsHostnames` = false by default on custom VPCs; set to true to get public DNS names

## Key Service Properties to Memorize

| Service | Resilience level |
| --- | --- |
| [IAM](../services/iam.md) | Global |
| [VPC](../services/vpc.md) | Regional |
| IGW | Regional |
| Subnet | AZ-specific |
| NAT GW | Zonal (need one per AZ for HA) |
| [EC2](../services/ec2.md) instance | AZ-specific |

## Common Exam Traps

- [NACLs](../services/vpc.md#network-acl-nacl) apply to **subnet boundary** traffic — intra-subnet traffic is not filtered
- [Security Groups](../services/vpc.md#security-groups-sg) attach to **ENIs**, not instances or subnets
- NAT GW needs an **Elastic IP** (assigned automatically on creation)
- 1 VPC → at most 1 IGW; 1 IGW → at most 1 VPC
- DHCP Options Sets cannot be edited — create a new one and associate it
- [IAM](../services/iam.md) and network security ([SG](../services/vpc.md#security-groups-sg)/[NACL](../services/vpc.md#network-acl-nacl)) are independent layers — both must pass

## References

- [IAM](../services/iam.md)
- [VPC](../services/vpc.md)
- [EC2](../services/ec2.md)
- [Networking Concepts](../concepts/networking.md)
- [Identity and Access Concepts](../concepts/identity-and-access.md)
