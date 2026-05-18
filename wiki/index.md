# Wiki Index

## Overview

- [Overview](overview.md) — Big-picture synthesis; mental models, resilience tiers, topics covered

## Services

- [IAM](services/iam.md) — Identity & access management; users, roles (trust + permissions policies, STS), policies (JSON syntax & case sensitivity), ARNs, federation
- [VPC](services/vpc.md) — Virtual network; subnets, routing, IGW, NACL, SG, NAT GW, PrivateLink
- [EC2](services/ec2.md) — Virtual compute; AMIs (stub, pending expansion)
- [S3](services/s3.md) — Object storage; bucket policies, ACLs (legacy), Block Public Access, cross-account & anonymous access
- [Route 53](services/route53.md) — Authoritative DNS; hosted zones, routing policies, Alias records, health checks, Resolver
- [Organizations](services/organizations.md) — Multi-account management; OUs, consolidated billing, SCPs, cross-account role-switching

## Concepts

- [Networking](concepts/networking.md) — Firewall layers, routing, internet connectivity patterns, ephemeral ports
- [DNS](concepts/dns.md) — Resolution chain, record types, TTL, caching (generic concepts)
- [TLS/HTTPS](concepts/tls-https.md) — Certificate chain of trust, CA issuance, TLS handshake, ACM integration
- [Identity and Access](concepts/identity-and-access.md) — Auth vs authz, identity types, policy evaluation, credential types
- [Resource Policies vs Identity Policies](concepts/resource-vs-identity-policies.md) — Two policy families; cross-account & anonymous access; same-account union vs cross-account intersection

## Patterns

- [Cost Optimization](patterns/cost-optimization.md) — Savings Plans (Compute / EC2 Instance), Reserved Instances, Spot; decision tree; pooling via Organizations

## Certifications

- [AWS Solutions Architect Associate](certifications/aws-saa.md) — Exam tips, comparison tables, common traps for SAA-C03
