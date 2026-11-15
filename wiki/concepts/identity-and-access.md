---
title: Identity and Access Concepts
type: concept
tags: [identity, access, authentication, authorization, iam, policies, roles, federation, scp, credentials]
certs: [aws-saa, aws-aip]
updated: 2026-11-13
---

# Identity and Access Concepts

Cross-cutting identity, authentication, and authorization concepts in AWS. [IAM](../services/iam.md) governs access to AWS resources; [Amazon Cognito](../services/cognito.md) provides application sign-in and brokers temporary AWS credentials for web and mobile users.

## Authentication vs Authorization

- **Authentication** — verifying who you are; the identity provider might be IAM, a Cognito user pool, or an external provider.
- **Authorization** — determining what you can do; IAM policies govern AWS API access, while applications can enforce permissions using token claims and scopes.

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

### Application tokens vs AWS credentials

A [Cognito user pool](../services/cognito.md#user-pools--application-sign-in) issues **ID and access tokens** for application identity and authorization. An [API Gateway](../services/api-gateway.md#authentication) authorizer or an application backend can validate these tokens without giving the client AWS credentials.

A [Cognito identity pool](../services/cognito.md#identity-pools--temporary-aws-credentials) bridges from a trusted provider to **temporary AWS credentials** for direct AWS calls. With both pools, the user pool ID token is the input to this exchange. With an existing external provider, a user pool is optional. Guests can receive credentials only when guest access is enabled, using the configured guest role.

The distinction is between **a token accepted by an application** and **credentials used to sign an AWS request**. Federation does not always end in STS credentials: a user pool can federate sign-in and issue application tokens only. The STS table above describes federation for AWS resource access.

## Policy Evaluation Order

All applicable policies are evaluated together. Resolution within any policy set:

1. **Explicit Deny** wins always
2. **Explicit Allow** grants access
3. **Implicit Deny** — default if nothing allows

This applies across user policies, group policies, [resource-based policies](resource-vs-identity-policies.md), and [SCPs](../services/organizations.md#service-control-policies-scps) (Service Control Policies in AWS Organizations). See [IAM](../services/iam.md#policy-evaluation--priority) for the full breakdown with examples.

> [!warning] Gotcha
> Same-account vs cross-account access combine the two policy families differently. Within one account, an Allow in **either** the identity policy or the resource policy is enough (subject to no Deny). Across accounts, **both** sides must explicitly Allow — see [Resource Policies vs Identity Policies — How the two combine](resource-vs-identity-policies.md#how-the-two-combine).

> [!tip] Exam Tip
> The presence of an explicit Deny anywhere in the policy evaluation chain blocks the request — even if another policy has an explicit Allow for the same action.

### SCPs as a Permission Boundary

[Service Control Policies](../services/organizations.md#service-control-policies-scps) sit **above** identity policies and act as a ceiling on the entire account:

- SCPs **never grant** permissions — only IAM identity policies grant.
- An action is effective only when **both** the SCP allows it **and** the identity policy allows it — i.e. the **intersection** of the two.
- SCPs **do not apply to the management account** of an AWS Organization.
- SCPs are one of the few mechanisms that can indirectly restrict the **account root user**, which is otherwise immune to IAM identity policies.

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

## Secret Retrieval vs Database Authentication

[Secrets Manager](../services/secrets-manager.md#architecture-retrieve-connect-rotate) illustrates two separate credential layers: an application uses temporary IAM-role credentials to retrieve a secret through an AWS API, then uses the returned username/password to authenticate to [RDS](../services/rds.md). Permission to retrieve the secret does not grant database privileges by itself.

With a customer-managed [KMS key](../services/kms.md), secret retrieval also needs decryption authorization, allowing secret metadata administration and plaintext access to be separated. A rotation Lambda has its own execution role and target-system permissions; these are distinct from the application's read permissions.

## References

- [IAM](../services/iam.md)
- [IAM Roles](../services/iam.md#iam-roles)
- [Resource Policies vs Identity Policies](resource-vs-identity-policies.md)
- [VPC — Security Groups & NACLs](../services/vpc.md)
- [Networking Concepts](networking.md)
