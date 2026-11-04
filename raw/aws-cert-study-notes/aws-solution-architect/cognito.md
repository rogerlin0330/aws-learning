---
title: Amazon Cognito - User and Identity Pools
course: Cantrill SAA-C03
section: Associate Shared
tags: [aws, saa-c03, cognito, identity, federation, jwt, iam, security]
---

# Amazon Cognito — User Pools & Identity Pools

## Overview

Cognito provides **authentication** (verify credentials / log in), **authorization** (manage access to services) and **user management** (a serverless user database) for web and mobile apps. It has two components with confusingly similar names that do **completely different jobs**:

| Component | One-line job | Output |
|---|---|---|
| **User Pool** | Sign-up / sign-in + user directory | **JWT** (user pool token) |
| **Identity Pool** | Swap an identity (or no identity) for AWS access | **Temporary AWS credentials** (via an IAM role) |

> [!important] The single biggest thing to remember
> A user pool **JWT is not AWS credentials**. It works with your own apps and a few services (e.g. API Gateway), but **most AWS services require AWS credentials** — and those come from an **identity pool**.

---

## User Pools

**Purpose:** a database of users (which can include external identities) — they sign in, they get a JWT. That's it.

Features (as presented in the course):
- Sign-up and sign-in, with a built-in customizable web UI
- MFA, phone and email verification
- Compromised-credential checks and account-takeover protection
- Customized workflows and user migration via **Lambda triggers**
- **Social sign-in** (Facebook, Google, Amazon, Apple) and enterprise sign-in via **SAML** (and OIDC)

📚 **Who holds the user data — native vs. federated users** (one pool can contain both):

| | Native (local) user | Federated user (e.g. "Sign in with Google" via the pool) |
|---|---|---|
| Who verifies the password | Cognito user pool | External IdP (Cognito never sees it) |
| Record in your user pool | ✅ Full profile + credentials | ✅ Profile only — generated username (e.g. `Google_…`) + mapped attributes, refreshed from the IdP on sign-in by default |
| Source of truth for authentication | Cognito | External IdP |
| Token the app receives | User pool JWT | User pool JWT (same) |

- The directory lives in **your** user pool (your account, your Region): AWS runs the service, you own and manage the data.

> [!warning] ⚠️ Course vs. reality — feature plans (Nov 2024)
> User pools now have three feature plans: **Lite**, **Essentials** (default for new pools), and **Plus**. Compromised-credential detection and adaptive authentication (the "account takeover protection" in the lecture) were renamed from *advanced security features* to **threat protection** and are **Plus-only**. The hosted UI was succeeded by **managed login** (Essentials/Plus); the classic hosted UI remains in all tiers. — [AWS: User pool feature plans](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-sign-in-feature-plans.html), [Cognito doc history](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-document-history.html)

### Diagram walkthrough — User Pools

![[cognito-user-pools.png]]

Read top → bottom; the three red arrows are the three things a JWT is used for.

1. **Sign-in (top row):** Inside the dotted box, the Cognito user pool fronts both its own users and external IdPs (Facebook, Google, Amazon, Apple, Twitter, OpenID). Whichever is used, the app receives **one** thing back: a **JWT**. The external identity is now represented as a *user pool user*.
2. **Self-managed resources (middle):** The app presents the JWT as **proof of authentication** to servers/databases *you* run. Your code validates the token.
3. **API Gateway → Lambda (bottom):** API Gateway can validate user pool tokens itself (the slide notes this used to require a Lambda custom authorizer). Once authorized, the request proceeds to the backend (e.g. Lambda).
   - 📚 Current mechanism: REST APIs use a **`COGNITO_USER_POOLS` authorizer**; HTTP APIs use a **JWT authorizer** pointed at the user pool issuer. Lambda authorizers still exist for custom logic — "legacy" means "the older way to do Cognito," not deprecated. — [AWS: x-amazon-apigateway-authorizer](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-swagger-extensions-authorizer.html)
   - 📚 **ALB** can also authenticate users against a user pool (`authenticate-cognito` listener rule action) — a common exam alternative to API Gateway for web apps behind a load balancer.

> [!warning] ⚠️ Twitter on the user pool slide
> User pools natively support **Facebook, Google, Login with Amazon, and Sign in with Apple** as social IdPs, plus generic **SAML 2.0 and OIDC**. Twitter/X is **not** a built-in user pool social provider (it would need a generic OIDC integration). Twitter *is* a built-in **identity pool** provider. — [AWS API: ProviderDescription](https://docs.aws.amazon.com/cognito-user-identity-pools/latest/APIReference/API_ProviderDescription.html)

What's missing from the slide on purpose: **no arrow to S3/DynamoDB/SQS** — a JWT alone cannot reach them.

---

## Identity Pools

**Purpose:** exchange an identity for **temporary AWS credentials** by having Cognito **assume an IAM role** on the identity's behalf.

Two identity types:
- **Unauthenticated (guest)** → e.g. a mobile game writing high scores to a DynamoDB leaderboard without requiring sign-up.
- **Authenticated** → a trusted IdP vouches for the user: Google, Facebook, Amazon, Apple, Twitter, **SAML 2.0** (corporate), OIDC, **and Cognito user pools** (to an identity pool, a user pool is just another IdP).

Mechanics:
- Configure **at least two IAM roles**: one **authenticated**, one **unauthenticated**.
- The app's permissions = the assumed role's permissions.
- Credentials are temporary; the app **renews via Cognito** on expiry. **No long-term credentials stored in code.**
- 📚 Identity pools also support **developer-authenticated identities** (your own backend auth) and **role mapping** (rules or token claims pick among multiple roles, not just auth/unauth). — [AWS CDK identity pool docs](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_cognito_identitypool-readme.html)

### Diagram walkthrough — Identity Pools (direct external IdP)

![[cognito-identity-pools.png]]

Follow the **red** path (identity) first, then the **orange** path (AWS credentials).

1. **Authenticate with the third-party IdP directly** (top): the user is sent to e.g. Google's sign-in page. The app never sees the user's Google password.
2. **IdP token returned:** the app receives a **Google token** (or Facebook/Amazon/… token). This token proves login but **cannot access AWS**.
3. **Token → identity pool** (red arrow into the orange box): the identity pool must be **configured for each IdP** it will accept.
4. **Cognito assumes a role:** for an authenticated identity, the **Authenticated Role**; for guests, the **Unauthenticated Role**. STS issues temporary credentials.
5. **Credentials returned** (orange arrow up): the app now holds **AWS temporary credentials**.
6. **Access AWS** (orange arrow right): the app calls **SQS, DynamoDB, S3, …** directly, limited by the role's policy.

**Pain point exposed by this design:** the app and identity pool must handle **one token type per IdP** — 5 IdPs = 5 configurations.

---

## Combined Architecture — User Pool + Identity Pool

### Diagram walkthrough

![[cognito-user-and-identity-pools.png]]

This is the two previous diagrams stacked: the **top half is the User Pools diagram**, the **bottom half is the Identity Pools diagram** — joined by the **user pool token**.

1. **Sign in via the user pool** (top, dotted box labeled *Cognito User Pool with social sign-in*): internal user or social login — either way the result is **one JWT**.
2. **Standardize on a single token** (centre caption): the user pool abstracts away all the IdP-specific tokens. One user directory, one set of profiles.
3. **JWT → identity pool:** the identity pool is configured with **only one IdP — the user pool**.
4. **Role assumed → temporary credentials → AWS services**, exactly as before.

**Benefit:** same end result as direct federation, with **less admin overhead** (one IdP configuration instead of many, one token type in the app).

---

## Key Terms

- **JWT (JSON Web Token):** the user pool token; proof of sign-in.
- **Web identity federation:** swapping any external IdP token for temporary AWS credentials. Expect this term on the exam.
- **Federation (identity pools):** the token-for-credentials exchange in general.
- **Scale:** identity pools support near-unlimited users vs. the **5,000 IAM users per account** quota (still current, not adjustable) — the reason IAM users are wrong for web-scale apps. — [AWS: IAM endpoints and quotas](https://docs.aws.amazon.com/general/latest/gr/iam-service.html)

---

## 🎯 Exam Traps

| Trigger in the question | Answer |
|---|---|
| "Mobile/web app users need to read/write **S3 / DynamoDB** directly" | **Identity pool** (temporary credentials via IAM role) |
| "Add sign-up/sign-in, MFA, user directory" | **User pool** |
| "Guest access without sign-up" | Identity pool **unauthenticated role** |
| "Secure an API Gateway API for logged-in users" | User pool + **Cognito authorizer** (REST) / JWT authorizer (HTTP API) |
| "Support many social IdPs with minimal admin" | User pool (social sign-in) → identity pool with the user pool as its single IdP |
| "Millions of users / web scale" — option suggests IAM users | ✗ IAM users (5,000 limit); use Cognito |
| "Don't embed AWS keys in the mobile app" | Identity pool temporary credentials |
| Option: "Use the user pool token to call S3" | ✗ JWT ≠ AWS credentials |
| Option: "Add users to an identity pool and use a `COGNITO_USER_POOLS` authorizer" | ✗ Identity pools don't hold users; the authorizer validates **user pool** tokens |

---

## Quick Reference

| | User Pool | Identity Pool |
|---|---|---|
| Question it answers | *Who are you?* | *What AWS access do you get?* |
| Stores users | ✅ Yes (directory + profiles) | ❌ No (maps identities to roles) |
| Inputs | Username/password, social, SAML, OIDC | IdP tokens (incl. user pool JWT), or none (guest) |
| Output | JWT | Temporary AWS credentials |
| Uses IAM roles | ❌ | ✅ Authenticated / unauthenticated |
| Accesses S3/DDB/SQS directly | ❌ | ✅ |
| Works with API Gateway | ✅ Native authorizer | ✅ via IAM (SigV4) auth |
| Guest access | ❌ | ✅ |

---

## Open Questions (for Roger)

1. 📚 The ALB `authenticate-cognito` and developer-authenticated-identities bullets go beyond the transcript — keep, trim, or move to a separate note?
2. Should this link to an existing `[[IAM]]`/`[[STS]]` or `[[API Gateway]]` note? I didn't assume your vault's note names.
3. The upcoming advanced demo (identity pools) — want it appended here or kept as its own note?
