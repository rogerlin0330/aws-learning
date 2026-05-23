# Wiki Index

## Overview

- [Overview](overview.md) — Big-picture synthesis; mental models, resilience tiers, topics covered

## Services

- [IAM](services/iam.md) — Identity & access management; users, roles (trust + permissions policies, STS), policies (JSON syntax & case sensitivity), ARNs, federation
- [VPC](services/vpc.md) — Virtual network; subnets, routing, IGW, NACL, SG, NAT GW, PrivateLink
- [EC2](services/ec2.md) — Virtual compute; AMIs (stub, pending expansion)
- [S3](services/s3.md) — Object storage; bucket policies, ACLs (legacy), Block Public Access, cross-account & anonymous access, pre-signed URLs (time-limited single-object access; IAM user vs role; permissions evaluated at use time), versioning + MFA Delete, Object Lock (WORM immutability for object versions; Compliance vs Governance retention; Legal Hold; bucket-level permanence — auto-enables versioning and cannot be disabled), object encryption (SSE-S3 / SSE-KMS / SSE-C / CSE) + S3 Bucket Keys, multipart upload + Transfer Acceleration, storage classes (Standard / IA / One Zone-IA / Glacier tiers / Intelligent-Tiering), lifecycle configuration (transitions + expiration; waterfall rules; 30-day Standard minimum; single-rule IA-to-Glacier gap), replication (CRR / SRR; same-account vs cross-account; RTC 15-min SLA; what's not replicated), server-side filtering (S3 Select / Glacier Select; SQL-like row/column filter pushed into the service; per-object only — Athena for multi-object), event notifications (Lambda / SQS / SNS destinations; Create/Delete/Restore/Replicate event types; destination resource policy must trust `s3.amazonaws.com`; EventBridge as the broader alternative), access logging (async best-effort request-level audit log; source → target bucket; Apache-style records; ACL grant to the S3 Log Delivery Group; CloudTrail data events as the real-time / structured alternative)
- [Route 53](services/route53.md) — Authoritative DNS; hosted zones, routing policies, Alias records, health checks, Resolver
- [Organizations](services/organizations.md) — Multi-account management; OUs, consolidated billing, SCPs, cross-account role-switching
- [CloudWatch](services/cloudwatch.md) — Monitoring & observability; CloudWatch Logs hierarchy (log group → stream → event), metric filters, alarms, agent
- [KMS](services/kms.md) — Key Management Service; symmetric/asymmetric keys, 4 KB direct limit, envelope encryption with DEKs, key ownership tiers (AWS Owned / AWS Managed / Customer Managed), key policies + IAM + grants

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
