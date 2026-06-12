# Systems Manager Parameter Store

## Overview

The **Systems Manager Parameter Store** is an AWS service that makes it easy to store various bits of system configuration — strings, documents, and secrets — in a resilient, secure, and scalable way.

It directly solves a problem flagged earlier in the course: passing secrets into an EC2 instance via **user data** is bad practice, because anyone with access to the instance can read all of that data. Parameter Store is the better pattern for getting configuration and secrets to instances, applications, and functions.

A parameter consists of a **parameter name** and a **parameter value**, where the value holds the actual configuration data.

---

## What You Can Store

Parameter Store supports three parameter **types**:

|Type|Description|
|---|---|
|**String**|A single plain-text value|
|**StringList**|A comma-separated list of values|
|**SecureString**|An encrypted value, integrated with KMS|

Typical use cases include license codes, database connection strings (host names, ports), DB users, full application configs, and passwords.

> 🎯 **Trigger keyword:** If an exam question asks where to store _configuration and secrets_ for EC2/Lambda/applications in a way that integrates natively with AWS services → **Parameter Store**.

---

## Key Capabilities

- **Hierarchical storage** — parameters can be organized into a tree-like path structure (see below).
- **Versioning** — Parameter Store keeps different versions of a parameter, conceptually similar to S3 object versioning.
- **Plaintext or ciphertext** — plaintext suits non-sensitive values (e.g., a DB connection string or DB user); ciphertext integrates with **KMS** to encrypt sensitive values such as passwords.
- **Public parameters** — publicly available parameters created and maintained by AWS.
- **Change events** — changes to parameters can spawn events that initiate processes in _other_ AWS services.

> ⚠️ **Trap:** Encrypting a parameter (SecureString) means a caller needs **both** the Parameter Store/SSM permissions **and** KMS permissions on the relevant key. This extra permission layer is the security benefit — and a common gotcha.

### Public parameters (AWS-created)

A frequently used example: when CloudFormation creates EC2 instances, you don't have to hard-code a specific AMI ID. Instead you reference a **public parameter** published by AWS that resolves to the latest AMI ID for a given operating system in a given region.

> _Illustrative pattern, not a value to memorize:_ the latest-AMI public parameter follows a path-style name maintained by AWS per OS and region.

---

## Hierarchical Structure

Parameters can be simple or organized into a tree, which makes both naming and permissions flexible.

```
/wordpress/dbuser
/wordpress/dbpassword
/my-cat-app/...
/dev/...
```

- A simple parameter might just be `mydbpassword` (e.g., an encrypted DB password).
- Under a branch like `/wordpress/`, you might have `dbuser` and `dbpassword`.
- Items can be retrieved **individually** by full name (e.g., `/wordpress/dbpassword`) **or** as a group by requesting the whole branch (e.g., pull the entire `/wordpress` tree).
- Branches can be organized by **application** (e.g., `/my-cat-app/`) or by **functional/organizational division** (e.g., a `/dev/` branch for the dev team's secrets).

> 🎯 Permissions can be applied to an **individual parameter** _or_ to a **whole tree** — this granularity is a key exam point and a reason to design a sensible hierarchy.

### Retrieving parameters from the CLI

These commands map the retrieve-individually vs. retrieve-by-branch concepts onto the actual API operations (e.g., from CloudShell):

```bash
# Get a single parameter by its full name
aws ssm get-parameters --names /rate-my-lizard/dbstring
aws ssm get-parameters --names /my-dog-app/dbstring
aws ssm get-parameters --names /my-cat-app/dbstring

# Get every parameter under a branch of the tree at once
aws ssm get-parameters-by-path --path /my-cat-app/

# Same, but decrypt any SecureString values in the result
aws ssm get-parameters-by-path --path /my-cat-app/ --with-decryption
```

- `get-parameters --names` → fetch one (or several explicitly named) parameters by **full name**.
- `get-parameters-by-path --path` → fetch a **whole branch** of the hierarchy in one call — the practical payoff of designing a sensible tree.
- `--with-decryption` → required to return the **plaintext** of `SecureString` values (this is the call that triggers the KMS `Decrypt` operation, so the caller also needs `kms:Decrypt`).

> _The parameter names above are illustrative paths, not values to memorize._

---

## Architecture

Parameter Store is a **public service**, so anything using it must either be an AWS service or have access to the AWS public-space endpoints.

Consumers include **applications**, **EC2 instances** (and software running on them), and **Lambda functions** — all can request parameters from the store.

Because it's an AWS service, Parameter Store is **tightly integrated with IAM**:

- Every access is **authenticated and authorized**.
- Credentials can be **long-term** (access keys) or passed in via an **IAM role**.
- If a parameter is encrypted, **KMS** is involved and the appropriate permissions on the **CMK (KMS key)** are also required.

### How to interact with it

- **Native integrations** — many AWS services integrate with Parameter Store directly (e.g., CloudFormation).
- **CLI tooling** — the AWS CLI on an EC2 instance can retrieve parameters.

### Regional service

Systems Manager — and Parameter Store within it — is a **regional service**. Parameters live in the Region where they're created, and the published limits (10,000 standard / 100,000 advanced) are **per account, per Region**.

- **Resilient within a Region:** Parameter Store is hosted across **multiple Availability Zones** in a single Region, so it stays available if an AZ fails.
- **Isolated across Regions:** there's **no built-in cross-Region replication**. The same parameter name can exist in different Regions but only once within a given Region. To make a parameter available in multiple Regions, you create it in each Region separately (e.g., one `put-parameter` call per Region) or build your own replication.

> 🎯 Mental model for the exam: **highly available within one Region, isolated across Regions.** Don't assume a parameter created in one Region is automatically reachable from another.

📚 **Reference:** [Parameter Store — multi-AZ, single-Region hosting (AWS re:Post)](https://repost.aws/questions/QUU5d0gTC2SDyafw86IT0iPA/is-it-possible-to-create-a-parameter-store-parameter-in-mutiple-region)

---

## SecureString Encryption with KMS

> The following section adds detail from the official AWS documentation beyond the course transcript. It's deeper than the associate exam strictly requires, but the **standard vs. advanced** distinction and the **envelope-encryption** concept are worth understanding.

Parameter Store itself performs **no cryptographic operations**. It relies on **AWS KMS** to encrypt and decrypt `SecureString` values. You can use the **AWS managed key** (alias `aws/ssm`) that Systems Manager creates automatically, or specify your own **customer managed key**.

> ⚠️ Parameter Store supports **only symmetric KMS keys**. Asymmetric KMS keys cannot be used to encrypt parameters.

There are **two tiers** of SecureString parameter, and they encrypt differently:

### Standard SecureString — direct encryption

A standard parameter (max **4 KB**) is encrypted **directly** under the KMS key you specify. Parameter Store calls the KMS `Encrypt` operation with the plaintext value and the chosen key; KMS returns the ciphertext, which is stored alongside the parameter name. On retrieval with `WithDecryption`, Parameter Store calls KMS `Decrypt` to return the plaintext.

> **How to read the diagram below (standard):** Start at **Parameter Store** on the left (the parameter name + plaintext value). The flow moves left-to-right: the **Parameter value** is sent through **Encrypt** to **KMS**, which uses the **KMS key** directly. The bottom row shows the return path: KMS hands back the **encrypted parameter value**, which Parameter Store stores against the parameter name.

![[ssm-parameter-store-standard-parameter-encryption.png]]

### Advanced SecureString — envelope encryption

An advanced parameter (max **8 KB**) uses **envelope encryption** via the **AWS Encryption SDK**. Each value is encrypted under a **unique data key**, and that data key is itself encrypted under the KMS key. The result — the **encrypted message** — bundles the encrypted parameter value together with the encrypted data key, and the whole message is stored as the parameter value.

> **How to read the diagram below (advanced):** Read left-to-right across three regions — **Parameter Store** (left), the **Encryption SDK** (middle), and **KMS** (right). Top row: the plaintext **Parameter value** flows into the **Encryption SDK**, which asks KMS to **Generate Data Key**. KMS (using the **KMS key**) returns _two_ copies of the data key — an **encrypted data key** and a **plaintext data key** (shown bottom-right). The SDK uses the plaintext data key to encrypt the value, then packages the **Encrypted Message** (encrypted value + encrypted data key), which Parameter Store stores against the parameter name (bottom-left).

![[ssm-parameter-store-advanced-parameter-encryption.png]]

> ⚠️ You can **upgrade** a standard parameter to advanced, but you **cannot downgrade** an advanced parameter back to standard (doing so would truncate 8 KB → 4 KB and drop any attached policies). To stop paying for an advanced parameter, delete it and recreate it as standard.

### IAM/KMS permissions for SecureString

|Action|Required KMS permission|
|---|---|
|Encrypt a **standard** SecureString|`kms:Encrypt`|
|Encrypt an **advanced** SecureString|`kms:GenerateDataKey`|
|Decrypt **either** tier|`kms:Decrypt`|

> Note: you cannot set access-control policies on the default `aws/ssm` managed key — fine-grained encrypt/decrypt control requires a **customer managed key**.

### Encryption context (good-to-know)

KMS operations use an **encryption context** of `PARAMETER_ARN` → the ARN of the parameter being encrypted. It's cryptographically bound to the data and appears in plaintext in CloudTrail logs, and can be used in IAM policy `Condition` blocks to scope decrypt permission to a specific parameter.

_Illustrative pattern (not to memorize):_ `"PARAMETER_ARN":"arn:aws:ssm:region-id:account-id:parameter/parameter-name"`

📚 **Reference:** [AWS KMS encryption for Parameter Store SecureString parameters](https://docs.aws.amazon.com/systems-manager/latest/userguide/secure-string-parameter-kms-encryption.html)

---

## Standard vs. Advanced Tier — Pricing & Limits

Tiers are configured **per parameter** (standard is the default), and they affect cost, scale, and capabilities.

||**Standard**|**Advanced**|
|---|---|---|
|Parameters per account/Region|10,000|100,000|
|Max parameter value size|4 KB|8 KB|
|Parameter policies|No|Yes|
|Encryption method|Direct KMS encrypt|Envelope encryption (AWS Encryption SDK)|
|**Cost**|**No additional charge**|**Charges apply**|

**Pricing detail** (verify current numbers on the AWS pricing page before relying on them):

- **Standard tier:** the first 10,000 parameters are at **no additional charge**, and standard-throughput API interactions for standard parameters are free.
- **Advanced tier:** charged at roughly **$0.05 per advanced parameter per month**, plus a per-API-interaction charge for advanced parameters (and for all parameters if higher throughput is enabled). Advanced parameters always incur a charge.

> ⚠️ **Trap:** Standard parameters are _free_, but you're capped at 10,000 per Region and 4 KB per value. Reach for **advanced** only when you need the higher limits, 8 KB values, or **parameter policies** — and remember it's billable and **one-way** (no downgrade).

📚 **Reference:** [Managing parameter tiers](https://docs.aws.amazon.com/systems-manager/latest/userguide/parameter-store-advanced-parameters.html) · [Systems Manager Pricing](https://aws.amazon.com/systems-manager/pricing/)

---

## 🎯 Exam Quick-Reference Recap

- **Purpose:** centrally store strings, documents, and secrets; the proper alternative to baking secrets into EC2 user data.
- **Types:** String, StringList, SecureString (SecureString = KMS-encrypted).
- **Public service** → consumers need AWS public-endpoint access; integrates natively with services like CloudFormation, and is reachable via the CLI on EC2.
- **Regional** → parameters and the 10K/100K limits are per account, per Region; multi-AZ within a Region, but **no cross-Region replication**.
- **IAM-integrated** → authenticated & authorized access via access keys or IAM roles; encrypted parameters additionally require **KMS permissions on the CMK**.
- **Hierarchies** → tree-structured names; retrieve individually or by branch; set permissions per-parameter or per-tree.
- **Versioning** → built in, S3-object-versioning-style.
- **Public parameters** → AWS-published (classic example: latest AMI ID for an OS/region, used by CloudFormation).
- **Change events** → parameter changes can trigger actions in other AWS services.
- **SecureString tiers** → standard (4 KB, direct KMS encrypt, free) vs. advanced (8 KB, envelope encryption + parameter policies, billable, no downgrade).
- **Permissions cheat:** standard encrypt = `kms:Encrypt`; advanced encrypt = `kms:GenerateDataKey`; decrypt (both) = `kms:Decrypt`.

---

### Editorial notes

- The **standard vs. advanced SecureString**, **envelope-encryption**, **encryption-context**, and **permission-matrix** material is sourced from AWS documentation and goes slightly beyond what the transcript covers; included for completeness and flagged here for your review.
- IP/ARN/region values throughout are framed as **illustrative patterns**, not facts to memorize.
- Pricing figures change over time — the per-parameter/per-API numbers are drawn from current third-party summaries and AWS docs; **confirm against the official pricing page** before quoting.