---
title: Resource Policies vs Identity Policies
type: concept
tags: [iam, s3, security, policies]
certs: [aws-saa]
updated: 2026-05-17
---

# Resource Policies vs Identity Policies

Two complementary ways to grant AWS permissions. Both are JSON policies and both go through the same [policy evaluation rules](../services/iam.md#policy-evaluation--priority), but they answer different questions and attach to different things.

| | Identity policy | Resource policy |
| --- | --- | --- |
| Attached to | An IAM **identity** (user, group, role) | An AWS **resource** (S3 bucket, KMS key, SNS topic, …) |
| Question it answers | *What can this identity do?* | *Who can access this resource?* |
| Has a `Principal` field? | **No** — the identity is implicit | **Yes** — must explicitly name the principal(s) |
| Can grant **cross-account** access on its own? | No — the principal must also be allowed by the resource's policy | Yes — names external account IDs / ARNs / IAM users / roles |
| Can grant **anonymous / public** access? | No — there is no authenticated identity to attach a policy to | Yes — `"Principal": "*"` |
| Typical AWS services | IAM (works for every service) | S3, KMS, SNS, SQS, Lambda, ECR, Secrets Manager, API Gateway, … |

## Why two mechanisms exist

The choice of where to attach a permission reflects **whose problem the permission solves**:

- An admin who manages **many resources for known identities** in one account uses **identity policies**. One IAM policy attached to a role can grant access to dozens of buckets, queues, and keys.
- An admin who owns **a single resource** and wants to control who outside their account (or outside AWS entirely) can touch it uses a **resource policy**. The control lives with the resource, not with the consumer.

This split is also the only way AWS can express two patterns that have no "identity" attached to them:

- **Anonymous access** — a website's public S3 bucket. No IAM user calls `GetObject`; the request is unauthenticated. There is no identity to attach a policy to, so the grant must live on the resource.
- **Cross-account access** — an IAM user in account A cannot have permissions to call APIs in account B unless account B explicitly says so. Account A cannot just attach a policy to its own user that grants access in account B; account B's resource must opt in.

## How the two combine

Effective permissions depend on whether the request crosses an account boundary.

### Same-account access

When the identity and the resource are in the **same AWS account**, AWS evaluates **all applicable policies together** under the [standard rules](../services/iam.md#policy-evaluation--priority) (Explicit Deny > Explicit Allow > Implicit Deny). An **Allow in either** the identity policy or the resource policy is sufficient:

- Identity policy allows, resource policy is silent → **allowed**
- Identity policy is silent, resource policy allows → **allowed**
- Either one denies → **denied**

> [!warning] Gotcha
> A common misconception is that same-account access requires *both* an identity-policy Allow **and** a resource-policy Allow. That's the cross-account model, not the same-account model. Within one account, a bucket policy alone can grant a same-account IAM user access without touching the user's identity policy at all.

### Cross-account access

When the principal and the resource live in **different accounts**, AWS evaluates each side separately and the request is allowed only if **both sides allow it**:

1. The principal's home account must allow the API call via an **identity policy** (so account A's admin authorizes their user/role to attempt the action).
2. The resource's account must allow that principal via the **resource policy** (so account B's admin authorizes this external principal).

This is a genuine **intersection**: a bucket policy that says "allow account A" does nothing unless account A's admin also grants their user an identity policy for `s3:GetObject` on that bucket. Both sides have a veto.

### Anonymous access

There is no identity policy, so **only the resource policy matters**. `"Principal": "*"` with no condition is the canonical pattern for a public-read bucket — read [Block Public Access](../services/s3.md#block-public-access) before deploying one.

## The `Principal` element

The presence of a `Principal` element is the syntactic marker of a resource policy. It names who the statement applies to:

| `Principal` value                                 | Meaning                                                                                   |
| ------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| `"*"`                                             | Anyone — including anonymous requesters and identities in other AWS accounts              |
| `{"AWS": "arn:aws:iam::111122223333:user/alice"}` | A specific IAM user in account `111122223333`                                             |
| `{"AWS": "arn:aws:iam::111122223333:root"}`       | The account `111122223333` — *any* identity in that account that its own admin authorizes |
| `{"Service": "lambda.amazonaws.com"}`             | An AWS service (used in trust policies and some resource policies)                        |
| `{"Federated": "cognito-identity.amazonaws.com"}` | A federated identity provider                                                             |

IAM **groups** cannot be named as principals — only users and roles. See [IAM Groups](../services/iam.md#iam-groups).

## Shared JSON grammar — case sensitivity

Identity and resource policies are **the same JSON dialect**, so the case-sensitivity rules are identical. In short:

- Top-level keys (`Version`, `Statement`, `Effect`, `Principal`, `Action`, `Resource`, `Condition`), `Effect` values (`Allow`/`Deny`), `Principal` sub-keys (`AWS`, `Service`, `Federated`), and condition operators (`StringEquals`, `IpAddress`, …) are **case-sensitive** — must be the documented PascalCase, or IAM rejects the policy.
- Action names (`s3:GetObject`) and condition-key names (`aws:SourceIp`) **match case-insensitively** at evaluation time, but follow the documented casing for readability.
- Values may or may not be case-sensitive depending on the operator (`StringEquals` vs `StringEqualsIgnoreCase`) and the resource (ARNs, tag values are generally case-sensitive).

Full table and exam tip in [IAM — Policy JSON Case Sensitivity](../services/iam.md#policy-json--case-sensitivity).

## When to use which

| Situation | Use |
| --- | --- |
| Granting one identity access to many resources across the account | **Identity policy** |
| Centralised, IAM-native management; identities are all internal | **Identity policy** |
| Granting many identities the same access to **one** resource | **Resource policy** (one policy on the resource beats N policies on N identities) |
| Granting access to an **external AWS account** | **Resource policy** (plus an identity policy on the other side) |
| Granting **anonymous / public** access | **Resource policy** — identity policies cannot express this |
| Granting access to an **AWS service** acting on your behalf | Both — a [role's trust policy](../services/iam.md#trust-policy-vs-permissions-policy) (resource-policy-style) plus its permissions policy (identity-policy-style) |
| Per-bucket conditions like "require MFA" or "deny non-corporate IPs" | **Resource policy** — see [S3 bucket policy examples](../services/s3.md#advanced-conditions) |

## Concrete examples

- **[S3 bucket policies](../services/s3.md#bucket-policies)** are the canonical resource policy: every example above maps directly to S3.
- **[IAM role trust policies](../services/iam.md#trust-policy-vs-permissions-policy)** are a resource policy in disguise — they live on the role (the resource) and name who is allowed to assume it (the principal).
- **[KMS key policies](../services/kms.md#key-policies)**, **SNS / SQS access policies**, **Lambda function policies** (for invoke permissions), and **ECR repository policies** all follow the same pattern. KMS adds a twist: the key policy is **mandatory** and must explicitly trust the account before IAM policies in that account can grant any access — there is no "IAM-only" path to a KMS key.

> [!tip] Exam Tip
> Spot the question type by what's being controlled. *"What can this user do?"* → identity policy. *"Who can access this bucket / key / queue?"* → resource policy. *"How do we grant anonymous read access?"* → must be a resource policy. *"How do we let another AWS account in?"* → resource policy on this side + identity policy on their side.

## References

- [IAM — Policy Evaluation & Priority](../services/iam.md#policy-evaluation--priority)
- [IAM Roles — Trust policy vs permissions policy](../services/iam.md#trust-policy-vs-permissions-policy)
- [S3 — Bucket Policies](../services/s3.md#bucket-policies)
- [Identity and Access Concepts](identity-and-access.md)
- Source: `raw/aws-cert-study-notes/aws-solution-architect/s3_resource_policies_and_acl.md`
