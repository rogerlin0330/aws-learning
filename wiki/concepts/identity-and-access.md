---
title: Identity and Access Concepts
type: concept
certs: [aws-saa, aws-aip]
updated: 2026-05-14
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
| IAM Role | No (temporary STS tokens) | [EC2](../services/ec2.md) instances, Lambda, cross-account access, federation |
| IAM Group | N/A — not an identity | Container of users for policy attachment; cannot log in, cannot be named in resource policies |

Prefer **IAM Roles** over IAM Users wherever possible — roles use short-lived credentials and leave no long-term secret to leak. The **5,000 IAM user per account** hard limit also forces roles-plus-federation for any large user base.

## Principals and Role Assumption

A **principal** is any entity that can be authenticated — a person, application, device, or AWS service. Principals act on AWS in one of two ways:

- **As an IAM user** — long-term credentials, the user *is* the identity.
- **By assuming a role** — the principal calls `sts:AssumeRole` (or a variant); if the role's **trust policy** allows them, [STS](../services/iam.md#how-assumerole-works-sts) returns temporary credentials and the principal then *becomes* the role for the credential's lifetime.

A role has **two policies** that answer different questions:

| Policy | Question |
| --- | --- |
| Trust policy | *Who* can assume this role? |
| Permissions policy | *What* can the role do once assumed? |

External identities (corporate AD, Google/Facebook, users in other AWS accounts) **cannot call AWS APIs directly** — they must assume a role first. This is how AWS handles authentication for any principal that isn't already an IAM user. See [IAM Roles](../services/iam.md#iam-roles) for the full role model.

## Identity Federation

Federation lets external identity providers serve as the source of truth for who someone is, while AWS handles authorization via roles:

| Federation type | Source identity | STS API |
| --- | --- | --- |
| **SAML / Corporate IdP** | Active Directory, Okta, etc. | `sts:AssumeRoleWithSAML` |
| **Web Identity** | Google / Facebook / Apple / Cognito | `sts:AssumeRoleWithWebIdentity` |
| **Cross-account** | IAM user/role in another AWS account | `sts:AssumeRole` |

In every case the user authenticates *somewhere else*, then exchanges that proof for temporary AWS credentials tied to a role. This pattern scales to millions of users and bypasses the 5,000 IAM user limit. See [IAM Roles — When to Use Roles](../services/iam.md#when-to-use-roles) for the canonical scenarios.

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

| Credential | Used for | Source |
| --- | --- | --- |
| Username + password | AWS Console | IAM user |
| Access Key ID + Secret Access Key | CLI / API (long-term) | IAM user |
| Access Key ID + Secret Access Key + **Session Token** | CLI / API (short-term) | [STS — role assumption](../services/iam.md#how-assumerole-works-sts) |

Temporary STS credentials are always a **triplet** including a session token — that token is what identifies the credentials as belonging to an assumed role rather than to an IAM user. Sessions typically last 15 min – 12 h; permission changes to the role take effect immediately on in-flight sessions.

## References

- [IAM](../services/iam.md)
- [IAM Roles](../services/iam.md#iam-roles)
- [VPC — Security Groups & NACLs](../services/vpc.md)
- [Networking Concepts](networking.md)
