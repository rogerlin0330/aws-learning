---
title: AWS Learning — Overview
type: overview
tags: [aws, architecture, iam, vpc]
updated: 2026-05-01
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
| Regional | [VPC](services/vpc.md), IGW, S3 |
| AZ | Subnet, NAT GW, [EC2](services/ec2.md) instance |

### The ENI is the Networking Atom

The [ENI (Elastic Network Interface)](concepts/networking.md) is the fundamental networking unit in AWS. Security Groups attach to ENIs, not to instances or subnets. Any resource that has an ENI in your VPC can have a Security Group — including [EC2](services/ec2.md), RDS, ALB, Fargate, and Lambda-in-VPC.

## Pages in This Wiki

### Services

- [IAM](services/iam.md) — policies, users, roles, ARNs, managed vs inline
- [VPC](services/vpc.md) — subnets, routing, IGW, NACL, SG, NAT GW, PrivateLink
- [EC2](services/ec2.md) — instances, AMIs, ENIs (expanding)

### Concepts

- [Networking](concepts/networking.md) — OSI layers, firewall layers, routing, internet connectivity patterns
- [Identity and Access](concepts/identity-and-access.md) — auth vs authz, identity types, policy evaluation

### Certifications

- [AWS Solutions Architect Associate](certifications/aws-saa.md) — exam tips, comparison tables, common traps

## Topics To Cover Next

- EC2: instance types, purchasing models, AMIs, EBS volumes
- S3: storage classes, lifecycle policies, bucket policies vs ACLs
- Route 53: routing policies (simple, weighted, latency, failover, geolocation)
- ELB / Auto Scaling: ALB vs NLB, target groups, scaling policies
- AWS AI services (for AI Practitioner cert)
- Architecture patterns: HA, multi-AZ, disaster recovery tiers
