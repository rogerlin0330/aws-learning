---
title: Identity and Access Concepts
type: concept
certs: [aws-saa, aws-aip]
updated: 2026-05-01
---

# Identity and Access Concepts

Cross-cutting identity, authentication, and authorization concepts in AWS. The primary service implementing these is [IAM](../services/iam.md).

## Authentication vs Authorization

- **Authentication** — verifying who you are (IAM acts as the IDP)
- **Authorization** — determining what you can do (IAM policies define permissions)

Note: IAM authorization is distinct from [network-level access control](../concepts/networking.md) (Security Groups, NACLs). A request must pass *both* — IAM says "you're allowed to call this API"; the network layer says "your packets can reach the resource."

## Identity Types

| Type | Long-term? | Use case |
| --- | --- | --- |
| IAM User | Yes | Humans, apps, service accounts needing persistent access |
| IAM Role | No (temporary STS tokens) | [EC2](../services/ec2.md) instances, Lambda, cross-account access |
| IAM Group | N/A | Collection of users — attach policies to the group |

Prefer **IAM Roles** over IAM Users wherever possible — roles use short-lived credentials and leave no long-term secret to leak.

## Policy Evaluation Order

All applicable policies are evaluated together. Resolution:

1. **Explicit Deny** wins always
2. **Explicit Allow** grants access
3. **Implicit Deny** — default if nothing allows

This applies across user policies, group policies, resource-based policies, and SCPs (Service Control Policies in AWS Organizations). See [IAM](../services/iam.md#policy-evaluation--priority) for the full breakdown with examples.

> [!tip] Exam Tip
> The presence of an explicit Deny anywhere in the policy evaluation chain blocks the request — even if another policy has an explicit Allow for the same action.

## IAM vs Network Security

[IAM](../services/iam.md) and [VPC](../services/vpc.md) security controls protect different things and must both be satisfied:

| Layer | Mechanism | Controls |
| --- | --- | --- |
| Identity | IAM policy | Which API actions are allowed |
| Network | [Security Group](../services/vpc.md#security-groups-sg) / [NACL](../services/vpc.md#network-acl-nacl) | Which network traffic can reach the resource |

A request can be IAM-allowed but network-blocked, or network-reachable but IAM-denied. Both must pass.

## Credential Types

| Credential | Used for |
| --- | --- |
| Username + password | AWS Console |
| Access Key ID + Secret Access Key | CLI / API |
| Session token (STS) | Temporary role credentials |

## References

- [IAM](../services/iam.md)
- [VPC — Security Groups & NACLs](../services/vpc.md)
- [Networking Concepts](networking.md)
