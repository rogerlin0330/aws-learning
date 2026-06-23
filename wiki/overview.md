---
title: AWS Learning — Overview
type: overview
tags: [aws, architecture, iam, vpc, organizations, control-tower, cost, s3, ec2, ecs, eks, kubernetes, containers, cloudwatch, cloudtrail, cloudformation, bootstrapping, dns, dnssec]
updated: 2026-06-23
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

Every AWS service operates at a specific resilience scope. Knowing this is critical for HA design; [Resource Scope](concepts/resource-scope.md) is the detailed reference.

| Tier | Examples |
| --- | --- |
| Global | [IAM](services/iam.md), Route 53 |
| Regional | [VPC](services/vpc.md), IGW, [S3](services/s3.md) |
| AZ | Subnet, NAT GW, [EC2](services/ec2.md) instance (and its host, ENI, EBS volume, instance store) |

### The ENI is the Networking Atom

The [ENI (Elastic Network Interface)](concepts/networking.md) is the fundamental networking unit in AWS. Security Groups attach to ENIs, not to instances or subnets. Any resource that has an ENI in your VPC can have a Security Group — including [EC2](services/ec2.md), RDS, ALB, Fargate, and Lambda-in-VPC.

### Compute Packaging Changes the Management Line

[EC2](services/ec2.md) gives you the full OS and runtime surface. [Containers](concepts/containers.md) package the application and dependencies into portable images that run as isolated processes on a host OS. Two managed orchestrators run those containers: [ECS](services/ecs.md) (AWS-proprietary) and [EKS](services/eks.md) (AWS-managed [Kubernetes](concepts/kubernetes.md), the cloud-agnostic open standard). Both run tasks/pods on EC2 when you want host control and pricing flexibility, or on Fargate when you want AWS to own the host layer — so launch type is orthogonal to the orchestrator choice. Both orchestrators pull their images from a registry — on AWS that is [ECR](services/ecr.md), the managed container image registry that stores, scans, and distributes images for ECS, EKS, and Fargate. Managed compute such as Fargate or Lambda shifts more infrastructure responsibility away from you, but the exam still tests the basic boundary: VM means full guest OS; container means shared host kernel plus packaged user-space dependencies.

## Pages in This Wiki

### Services

- [IAM](services/iam.md) — users, groups, roles (STS / federation), policies, ARNs
- [VPC](services/vpc.md) — subnets, routing, IGW, NACL, SG, NAT GW, PrivateLink
- [EC2](services/ec2.md) — IaaS foundations, hosts vs instances (tenancy + AZ-resilience), instance lifecycle & state-based costs (incl. stop/start vs restart), AMIs, bootstrapping (user data), SSH/RDP access, when-to-use-EC2 decision framing
- [CloudFormation](services/cloudformation.md) — infrastructure as code (templates → stacks); cfn-init desired-state bootstrapping, configSets, cfn-hup re-runs, CreationPolicy + cfn-signal completion handshake
- [ECS](services/ecs.md) — managed container orchestration (AWS-proprietary); task definitions, services, task roles, EC2 launch type vs Fargate
- [EKS](services/eks.md) — AWS-managed Kubernetes; control plane + etcd (multi-AZ), node options (EC2 / managed node groups / Fargate), two-VPC architecture, ECS-vs-EKS choice
- [ECR](services/ecr.md) — managed container image registry; registry/repository/image/tag hierarchy, public vs private permission model, IAM + repository policies, Basic vs Enhanced (Inspector) scanning, cross-region + cross-account replication
- [S3](services/s3.md) — buckets, bucket policies, ACLs (legacy), Block Public Access, pre-signed URLs, versioning, SSE/CSE encryption, multipart upload + Transfer Acceleration, storage classes + lifecycle, replication (CRR/SRR), S3 Select / Glacier Select, event notifications + EventBridge, access logging
- [Systems Manager (SSM)](services/ssm.md) — ops-management suite; **Parameter Store** covered in depth — config/secrets store, the proper alternative to EC2 user-data secrets; String/StringList/SecureString, hierarchical names, regional + no cross-Region replication, standard vs advanced (KMS direct-encrypt vs envelope) tiers, and Parameter Store vs Secrets Manager
- [Route 53](services/route53.md) — hosted zones, routing policies, health checks
- [Organizations](services/organizations.md) — multi-account management, consolidated billing, SCPs
- [Control Tower](services/control-tower.md) — "Organizations with superpowers"; orchestrates a well-architected landing zone (foundational OUs + Audit/Log Archive accounts), guardrails (preventative SCP / detective Config), and Account Factory self-service provisioning
- [CloudWatch](services/cloudwatch.md) — monitoring & observability; log groups, metric filters, alarms, and the **CloudWatch Agent** for in-guest EC2 metrics + OS/application logs
- [CloudTrail](services/cloudtrail.md) — audit & governance; API/account-activity events, Event History vs trails, management/data/Insight events, global service events, organizational trail, log file integrity
- [KMS](services/kms.md) — key management; envelope encryption, key policies + IAM + grants

### Concepts

- [Networking](concepts/networking.md) — OSI layers, firewall layers, routing, internet connectivity patterns
- [Identity and Access](concepts/identity-and-access.md) — auth vs authz, identity types, role assumption, federation
- [Resource Policies vs Identity Policies](concepts/resource-vs-identity-policies.md) — the two policy families; cross-account & anonymous access
- [Resource Scope](concepts/resource-scope.md) — global / regional / AZ / edge tiers; blast radius and movement cost
- [DNS](concepts/dns.md) — resolution, record types, TTL
- [DNSSEC](concepts/dnssec.md) — securing DNS; origin auth + integrity, chain of trust root→record (RRSIG/DNSKEY/ZSK/KSK/DS), trust anchor + root signing ceremony, Route 53 signing + validation
- [TLS/HTTPS](concepts/tls-https.md) — certificate trust, TLS handshake, ACM
- [Containers](concepts/containers.md) — container vs VM mental model, Docker image layers, runtime R/W layer, registries, ECS as orchestration
- [Kubernetes](concepts/kubernetes.md) — cloud-agnostic orchestration; control plane vs nodes, pods/services/ingress, ephemeral vs Persistent Volume storage, mapped to AWS via EKS

### Patterns

- [Cost Optimization](patterns/cost-optimization.md) — Savings Plans, RIs, Spot, decision tree, pooling via Organizations
- [Scaling — Horizontal vs Vertical](patterns/scaling.md) — bigger box vs more boxes; the session problem; stateless design; how horizontal scaling combines with ELB + Auto Scaling for HA
- [Boot Time to Service Time](patterns/boot-time-optimization.md) — AMI baking vs bootstrapping vs combined; minimizing post-launch time while keeping flexibility

### Certifications

- [AWS Solutions Architect Associate](certifications/aws-saa.md) — exam tips, comparison tables, common traps

## Multi-Account Architecture

Most real-world AWS deployments use **many accounts** — a per-team / per-environment / per-business-unit split provides hard blast-radius isolation that VPCs alone cannot. [AWS Organizations](services/organizations.md) is the glue: one bill, OU-based grouping, [SCP](services/organizations.md#service-control-policies-scps) guardrails, and a single login point that uses [cross-account IAM roles](services/iam.md#cross-account-access) to move between accounts. The cost-side benefit is that [Savings Plans and Reserved Instances pool org-wide](patterns/cost-optimization.md#pooling-across-accounts--aws-organizations) — so multi-account doesn't multiply purchase commitments.

[AWS Control Tower](services/control-tower.md) sits one layer above Organizations: instead of assembling the governance yourself, it **orchestrates** Organizations, IAM Identity Center, CloudFormation, and Config into a well-architected **landing zone** — default OUs, isolated **Log Archive** and **Audit** foundational accounts, SSO, **guardrails** (preventative SCP vs detective Config rules), and an **Account Factory** for self-service, baselined account provisioning.

## Topics To Cover Next

- EC2: placement groups (user-data & bootstrapping now covered on the [EC2 page](services/ec2.md#bootstrapping); IMDS / IMDSv2 on the [IMDS section](services/ec2.md#instance-metadata-service-imds))
- CloudFormation: the full surface beyond cfn-init — parameters, mappings, outputs, intrinsic functions, change sets, nested stacks / StackSets, drift detection (the [CloudFormation page](services/cloudformation.md) currently covers only the bootstrapping features)
- ELB / Auto Scaling: ALB vs NLB, target groups, scaling policies — builds directly on the [Scaling pattern](patterns/scaling.md)
- Container services: ELB integration patterns and App Mesh / service discovery — builds on [ECS](services/ecs.md), [EKS](services/eks.md), [ECR](services/ecr.md), and the [Containers](concepts/containers.md) / [Kubernetes](concepts/kubernetes.md) concept pages
- AWS AI services (for AI Practitioner cert)
- Architecture patterns: HA, multi-AZ, disaster recovery tiers
