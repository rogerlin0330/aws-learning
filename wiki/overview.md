---
title: AWS Learning — Overview
type: overview
tags: [aws, architecture, iam, vpc, organizations, cost, s3]
updated: 2026-05-21
---

# AWS Learning Overview

A synthesis of the AWS knowledge base, tracking the big picture across services, concepts, and patterns.

## Goals

- Understand how AWS services work — deeply, not just exam-surface level
- Build mental models for designing resilient, secure, cost-effective architectures
- Prepare for AWS Solutions Architect Associate (SAA-C03) and AI Practitioner (AIF-C01)

## Core Mental Model

AWS infrastructure is built in layers:

```text
Account
└── Region
    └── VPC (isolated network)
        ├── Subnet (AZ-scoped)
        │   ├── EC2 Instance
        │   │   └── ENI (network card) ← Security Group attached here
        │   └── NAT Gateway (if public subnet)
        └── Internet Gateway (regional, attaches to VPC)
```

[IAM](services/iam.md) is a **global** layer sitting above all of this — it controls who can do what to which resources, independent of region or network topology. Network security ([Security Groups](services/vpc.md#security-groups-sg), [NACLs](services/vpc.md#network-acl-nacl)) controls traffic flows *within* this infrastructure. The two layers are complementary.

## Key Cross-Cutting Themes

### Security is Defense in Depth

AWS enforces security at multiple independent layers — an attacker must defeat all of them:

- **[IAM](services/iam.md) policies** — API-level: what actions are allowed on which resources (global)
- **[Security Groups](services/vpc.md#security-groups-sg)** — L3/4 stateful firewall on each ENI (instance level)
- **[NACLs](services/vpc.md#network-acl-nacl)** — L3/4 stateless firewall at the subnet boundary
- **WAF** — L7 HTTP content inspection (future page)

See [Networking Concepts](concepts/networking.md) for OSI layer breakdown and [Identity and Access Concepts](concepts/identity-and-access.md) for how IAM evaluation works.

### Stateful vs Stateless is Everywhere

This distinction recurs across AWS — understanding it once unlocks many services:

- [NACLs](services/vpc.md#network-acl-nacl) are **stateless** → must explicitly allow return traffic (ephemeral ports)
- [Security Groups](services/vpc.md#security-groups-sg) are **stateful** → return traffic automatic
- [NAT GW](services/vpc.md#nat--nat-gateway) is **stateful** → tracks connections to map return traffic

### Resilience Tiers

Every AWS service operates at a specific resilience scope. Knowing this is critical for HA design:

| Tier | Examples |
| --- | --- |
| Global | [IAM](services/iam.md), Route 53 |
| Regional | [VPC](services/vpc.md), IGW, [S3](services/s3.md) |
| AZ | Subnet, NAT GW, [EC2](services/ec2.md) instance |

### The ENI is the Networking Atom

The [ENI (Elastic Network Interface)](concepts/networking.md) is the fundamental networking unit in AWS. Security Groups attach to ENIs, not to instances or subnets. Any resource that has an ENI in your VPC can have a Security Group — including [EC2](services/ec2.md), RDS, ALB, Fargate, and Lambda-in-VPC.

## Pages in This Wiki

### Services

- [IAM](services/iam.md) — users, groups, roles (STS / federation), policies, ARNs
- [VPC](services/vpc.md) — subnets, routing, IGW, NACL, SG, NAT GW, PrivateLink
- [EC2](services/ec2.md) — instances, AMIs, ENIs (expanding)
- [S3](services/s3.md) — buckets, bucket policies, ACLs (legacy), Block Public Access, versioning, SSE/CSE encryption, multipart upload + Transfer Acceleration, storage classes + lifecycle, replication (CRR/SRR)
- [Route 53](services/route53.md) — hosted zones, routing policies, health checks
- [Organizations](services/organizations.md) — multi-account management, consolidated billing, SCPs
- [CloudWatch](services/cloudwatch.md) — monitoring & observability; log groups, metric filters, alarms
- [KMS](services/kms.md) — key management; envelope encryption, key policies + IAM + grants

### Concepts

- [Networking](concepts/networking.md) — OSI layers, firewall layers, routing, internet connectivity patterns
- [Identity and Access](concepts/identity-and-access.md) — auth vs authz, identity types, role assumption, federation
- [Resource Policies vs Identity Policies](concepts/resource-vs-identity-policies.md) — the two policy families; cross-account & anonymous access
- [DNS](concepts/dns.md) — resolution, record types, TTL
- [TLS/HTTPS](concepts/tls-https.md) — certificate trust, TLS handshake, ACM

### Patterns

- [Cost Optimization](patterns/cost-optimization.md) — Savings Plans, RIs, Spot, decision tree, pooling via Organizations

### Certifications

- [AWS Solutions Architect Associate](certifications/aws-saa.md) — exam tips, comparison tables, common traps

## Multi-Account Architecture

Most real-world AWS deployments use **many accounts** — a per-team / per-environment / per-business-unit split provides hard blast-radius isolation that VPCs alone cannot. [AWS Organizations](services/organizations.md) is the glue: one bill, OU-based grouping, [SCP](services/organizations.md#service-control-policies-scps) guardrails, and a single login point that uses [cross-account IAM roles](services/iam.md#cross-account-access) to move between accounts. The cost-side benefit is that [Savings Plans and Reserved Instances pool org-wide](patterns/cost-optimization.md#pooling-across-accounts--aws-organizations) — so multi-account doesn't multiply purchase commitments.

## Topics To Cover Next

- EC2: instance types, AMIs, EBS volumes
- ELB / Auto Scaling: ALB vs NLB, target groups, scaling policies
- AWS AI services (for AI Practitioner cert)
- Architecture patterns: HA, multi-AZ, disaster recovery tiers
