# EC2 Instance Roles & Instance Profiles

## Overview

IAM **roles** are the best-practice way to grant AWS services permissions to act on other AWS services on your behalf. When a service assumes a role, it gains the permissions that role's policy grants.

An **EC2 instance role** is a role that an EC2 instance can assume. Anything running inside that instance inherits the permissions the role grants — without needing long-term credentials stored on the instance.

> 🎯 **Exam trigger:** "Application on EC2 needs to access S3 / DynamoDB / etc." → **EC2 instance role**, _not_ access keys.

---

## Architecture

!![[wiki/assets/ec2-instance-roles.png]]

### Reading the diagram

**Top-left (the IAM Role):** An IAM role configured so that the **EC2 service** is allowed to assume it. A **permissions policy** is attached to the role (the diagram shows one granting `S3:*`, used illustratively — any permissions can be attached).

**The dotted box (`InstanceProfile`):** A wrapper that surrounds the IAM role. This is the piece that allows the role's permissions to be delivered _inside_ the instance.

**Bottom-left → middle (Profile attached to instance):** The instance profile — not the role directly — is what gets attached to the EC2 instance.

**Middle → right (credentials → resource):** Temporary credentials are delivered into the instance via **instance metadata**, and an application uses them to reach a resource such as the S3 bucket on the right. The "Automatically renewed" key icon shows that these credentials are continuously refreshed.

---

## The Instance Profile (key concept)

The instance profile is an intermediate piece of architecture — a **wrapper around an IAM role**. It exists because the role itself can't be attached to an instance directly; the profile is the delivery mechanism.

| **Console (UI)**         | Creating an instance role auto-creates an instance profile with the **same name**. You _think_ you're attaching the role to the instance, but you're attaching the **profile**. |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **CLI / CloudFormation** | You must create the **role and the instance profile separately**, and can even give them **different names**.                                                                   |

> ⚠️ **Trap:** It is always the **instance profile** that is attached to an EC2 instance — never the role directly. The same-name auto-creation in the console hides this, which is exactly why the exam tests it.

### CLI example (the three separate actions)

Because the CLI doesn't auto-create the profile for you, the role → profile → instance chain is three explicit steps. _(Names below are illustrative.)_

```bash
# 1. Create the IAM role (with a trust policy allowing the EC2 service to assume it),
#    then attach a permissions policy to it.
aws iam create-role \
  --role-name S3Access \
  --assume-role-policy-document file://ec2-trust-policy.json

aws iam attach-role-policy \
  --role-name S3Access \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# 2. Create the instance profile (a separate action — name can differ from the role).
aws iam create-instance-profile \
  --instance-profile-name Webserver

# 3. Add the role into the instance profile (the "wrapper" link).
aws iam add-role-to-instance-profile \
  --role-name S3Access \
  --instance-profile-name Webserver
```

The trust policy file (`ec2-trust-policy.json`) is what makes this an _EC2_ instance role — it allows `ec2.amazonaws.com` to assume the role:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "ec2.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

Once the profile exists, you attach it to an instance — using the **profile name**, not the role name:

```bash
# At launch:
aws ec2 run-instances ... --iam-instance-profile Name=Webserver

# Or to an already-running/stopped instance:
aws ec2 associate-iam-instance-profile \
  --instance-id i-1234567890abcdef0 \
  --iam-instance-profile Name=Webserver
```

> 🎯 **Exam takeaway:** Console = one step (profile auto-created, same name, hidden from you). CLI/API/CloudFormation = role and profile are **separate objects** you wire together yourself, and they can have different names.

---

## Credential Delivery via Instance Metadata

When the role is assumed, **temporary security credentials** are generated (they expire, and grant permissions per the role's permissions policy). Inside an EC2 instance, these are delivered through **instance metadata**.

**Metadata path to the credentials:**

```
iam/security-credentials/<role-name>
```

Accessing this path returns the temporary credentials. _(Path shown as the structural pattern, not a value to memorize.)_

### Always-valid credentials

- EC2 and the **Secure Token Service (STS)** liaise to renew credentials **before they expire**.
- As long as the role stays attached and the app keeps **checking the metadata**, it will never hit expired credentials.
- Applications should be careful about **caching** — re-check the metadata periodically / before expiry rather than holding a stale copy.

---

## Best Practice: Roles over Long-Term Credentials

- **Always use roles** where possible — this is heavily stressed for the exam.
- Roles are preferable to storing **long-term credentials (access keys)** inside an EC2 instance.
- Never store long-term credentials anywhere not securely stored (e.g., a local machine or baked into an instance).
- **AWS tooling does this automatically:** the CLI and other AWS tools running inside an instance will automatically use the instance role's credentials, as long as the profile is attached — no manual key configuration needed.

### Why the CLI "just works" — credential precedence

The reason the CLI picks up instance role credentials automatically is its **credential provider chain**: the CLI checks a fixed list of credential sources in priority order and uses the first one it finds. **EC2 instance profile credentials are the _last_ source in this chain** — the fallback when nothing more explicit is configured.

Highest → lowest priority (simplified for the exam):

| Priority | Source                                                       |
| -------- | ------------------------------------------------------------ |
| 1        | Command-line options (e.g. `--profile`, `--region`)          |
| 2        | Environment variables (`AWS_ACCESS_KEY_ID`, etc.)            |
| 3–4      | Assume-role / assume-role-with-web-identity config           |
| 5        | IAM Identity Center (SSO)                                    |
| 6        | Credentials file (`~/.aws/credentials`)                      |
| 7        | Custom external credential process                           |
| 8        | Config file (`~/.aws/config`)                                |
| 9        | **ECS** container credentials                                |
| 10       | **EC2 instance profile credentials** (via instance metadata) |

> 🎯 **Exam relevance:** Because instance profile credentials sit at the **bottom** of the chain, an instance with a role attached needs **no credentials configured at all** — the CLI falls through to the metadata service and finds the role's temporary credentials. This is precisely why roles are the best practice: zero stored secrets.

> ⚠️ **Trap / gotcha:** Anything **higher** in the chain overrides the instance role. If someone has set environment variables or dropped an `~/.aws/credentials` file (e.g. baked-in access keys) on the instance, the CLI uses _those_ instead of the role — silently. Symptom on the exam: "EC2 instance has a role attached but the app/CLI is using the wrong (or expired) permissions" → check for higher-precedence credentials shadowing the role.

> 🎯 **Exam trigger:** "access keys stored on an instance" / "hardcoded credentials" → flag as **bad practice**; the correct answer is an **instance role**.

---

## Quick-Reference Recap

| Concept                   | Key point                                                                                                                                             |
| ------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| **EC2 instance role**     | A role the EC2 service (and thus an instance) can assume; grants permissions to anything running in the instance.                                     |
| **Instance profile**      | Wrapper around the role; the thing actually **attached** to the instance.                                                                             |
| **Console behavior**      | Profile auto-created with same name as the role.                                                                                                      |
| **CLI / CloudFormation**  | Role and profile must be created **separately**.                                                                                                      |
| **Credential delivery**   | Via **instance metadata** at `iam/security-credentials/<role-name>`.                                                                                  |
| **Renewal**               | EC2 + STS auto-renew before expiry → credentials always valid while attached.                                                                         |
| **App responsibility**    | Don't over-cache; re-check metadata before expiry.                                                                                                    |
| **Best practice**         | Roles, **never** long-term access keys on an instance.                                                                                                |
| **AWS CLI/tools**         | Use instance role credentials automatically when a profile is attached.                                                                               |
| **Credential precedence** | Instance profile credentials are **last** in the CLI provider chain — used only if nothing higher (CLI flags, env vars, `~/.aws/credentials`) is set. |

**Mnemonic** _(editorial addition, not from source):_ **R**ole holds permissions, **P**rofile gets attached, **M**etadata delivers — **RPM** keeps the engine running.

---

_Source: Adrian Cantrill — AWS Solutions Architect Associate course (learn.cantrill.io). The `S3:*` permission, credential path, and role names are illustrative patterns, not values to memorize._