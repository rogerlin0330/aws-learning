---
title: EC2 — Elastic Compute Cloud
type: service
certs: [aws-saa]
updated: 2026-05-01
---

# EC2 — Elastic Compute Cloud

EC2 provides resizable virtual compute instances in the cloud. Every EC2 instance runs inside a [VPC](vpc.md) subnet and is associated with at least one [ENI](../concepts/networking.md) (Elastic Network Interface).

## How EC2 Fits Into the AWS Architecture

- **Networking**: instances live in [VPC](vpc.md) subnets; traffic is controlled by [Security Groups](vpc.md#security-groups-sg) (attached to the ENI) and [NACLs](vpc.md#network-acl-nacl) (attached to the subnet)
- **Identity**: instances access other AWS services via [IAM roles](iam.md) — never embed access keys directly in an instance
- **Internet access**: public instances route through an [IGW](vpc.md#internet-gateway-igw); private instances route through a [NAT Gateway](vpc.md#nat--nat-gateway)

## AMI (Amazon Machine Image)

> Stub — content to be added.

An AMI is a template containing the OS, application server, and application configuration used to launch an instance. AMIs are region-specific.

## References

- [VPC — networking, SG, NACL, IGW, NAT GW](vpc.md)
- [IAM — roles for EC2](iam.md)
- [Networking Concepts — ENI, OSI layers](../concepts/networking.md)
- Source: `raw/aws-cert-study-notes/aws-solution-architect/ec2.md`
