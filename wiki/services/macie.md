---
title: Amazon Macie
type: service
tags: [macie, data-security, data-privacy, sensitive-data, pii, phi, s3, data-identifiers, regex, findings, eventbridge, lambda, security-hub, organizations, delegated-administrator, kms]
certs: [aws-saa]
updated: 2026-11-23
---

# Amazon Macie

Amazon Macie is a **data security and data privacy** service for [S3](s3.md). It answers two questions about your buckets:

1. **Where is my sensitive data?** Macie looks inside objects for personally identifiable information (PII), personal health information (PHI), financial data such as credit card numbers, and credentials such as AWS secret access keys and private keys.
2. **Are the buckets holding it protected?** Macie keeps an inventory of your buckets and reports when a bucket's settings change in a way that makes it less secure, for example when it becomes public.

AWS describes Macie as a service that "discovers sensitive data by using machine learning and pattern matching, provides visibility into data security risks, and enables automated protection against those risks" ([AWS: What is Macie?](https://docs.aws.amazon.com/macie/latest/user/what-is-macie.html)).

The lecture's reason for the service is that S3 can hold huge amounts of data and can be made public, so a misconfigured bucket can leak data. To control the security of your data, you first need to know where it is and what it contains. Macie gives you that inventory.

Like [GuardDuty](guardduty.md) and [AWS Config](config.md), Macie is a **detective** service. It reports problems as **findings** and does not change buckets or objects itself. To act on a finding, you send it to [EventBridge](eventbridge.md) and let another service respond.

> [!note] Macie only analyzes S3
> Macie works on S3 **general purpose buckets**. It does not analyze databases, EBS volumes, or other data stores. AWS says Macie "is optimized for Amazon S3" and suggests moving data from other services into S3 if you want Macie to analyze it ([AWS: supported storage classes and formats](https://docs.aws.amazon.com/macie/latest/user/discovery-supported-storage.html)).

## Architecture

![Amazon Macie architecture. Two S3 buckets on the left feed Amazon Macie, labeled detect and classify. Macie runs a discovery job, which is triggered by a discovery schedule and uses managed data identifiers from AWS and custom data identifiers. The job produces findings, and each finding is also sent as a finding event to EventBridge, which delivers it to a Lambda function](../assets/macie-architecture.png)

The diagram reads left to right:

1. **S3 buckets (left).** These are the buckets you want Macie to analyze. They can be in one account or in several accounts that Macie manages centrally.
2. **Amazon Macie, "detect and classify".** Macie reads objects from the buckets and classifies the data it finds.
3. **Discovery job (center).** You create a job and choose which buckets it analyzes. The **discovery schedule** above it controls when and how often the job runs: once, or on a recurring schedule.
4. **Data identifiers (bottom).** The job checks object contents against **managed data identifiers** (built and maintained by AWS; the icons show a passport, a credit card, a hospital bed, and a key) and **custom data identifiers** (rules you define). See [Data Identifiers](#data-identifiers).
5. **Findings (right).** When the job finds sensitive data, Macie creates a finding. You can review findings in the console.
6. **EventBridge, then Lambda (top right).** Macie also publishes every finding to EventBridge as an event, without any setup. An EventBridge rule can then send it to a [Lambda](lambda.md) function that fixes the problem automatically, or to another target such as [SNS](sns.md).

> [!warning] Gotcha: the diagram shows only one way Macie finds things
> The lecture diagram shows all findings coming from a discovery job. That is true for **sensitive data findings** from jobs, but it leaves out two things:
>
> - **Policy findings** do not come from a job. Macie produces them from its continuous monitoring of bucket settings (see [Findings](#findings)).
> - **Automated sensitive data discovery**, added in November 2022, finds sensitive data without a job. It samples objects across your buckets continuously (see [Automated discovery and discovery jobs](#automated-discovery-and-discovery-jobs)).

## What Macie Does

Macie has two kinds of work. They are independent of each other:

| Work                            | What Macie looks at                                                                                     | Output                  |
| ------------------------------- | ------------------------------------------------------------------------------------------------------- | ----------------------- |
| Bucket inventory and monitoring | Bucket **settings**: Block Public Access, bucket policy, ACLs, default encryption, replication, sharing | Policy findings         |
| Sensitive data discovery        | Object **contents**, checked against data identifiers                                                   | Sensitive data findings |

As soon as you enable Macie, it builds an inventory of your S3 general purpose buckets and starts monitoring them for security and access control ([AWS: What is Macie?](https://docs.aws.amazon.com/macie/latest/user/what-is-macie.html)). Sensitive data discovery runs either as automated discovery or as jobs that you create.

### Automated discovery and discovery jobs

| Option                                 | How it works                                                                                                                                                                    | Best for                                                                    |
| -------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------- |
| **Automated sensitive data discovery** | Macie "continually evaluates your S3 bucket inventory" and uses sampling to pick representative objects to analyze. It covers all buckets by default, and you can exclude some. | A broad, low-cost picture of where sensitive data is across the estate      |
| **Sensitive data discovery job**       | You choose the buckets, the objects, the identifiers, and the schedule. The job analyzes all the data in each object it selects.                                                | Deep, targeted analysis of specific buckets, such as for a compliance audit |

Sources: [AWS: automated sensitive data discovery](https://docs.aws.amazon.com/macie/latest/user/discovery-asdd.html), [AWS: discovery jobs](https://docs.aws.amazon.com/macie/latest/user/discovery-jobs.html).

**Automated discovery.** Automated discovery reached general availability on November 28, 2022 ([AWS: automated discovery default settings](https://docs.aws.amazon.com/macie/latest/user/discovery-asdd-settings-defaults.html)). By default it uses a recommended set of managed data identifiers and no custom identifiers or allow lists; you can change that. It gives each bucket a **sensitivity score** from -1 to 100. A score of 50 means the bucket has not been analyzed yet, 51 to 99 means sensitive, 1 to 49 means not sensitive, -1 means a classification error, and 100 is set only when you mark a bucket sensitive yourself ([AWS: sensitivity scoring](https://docs.aws.amazon.com/macie/latest/user/discovery-scoring-s3.html)). The Getting Started guide says Macie may begin automated discovery when you enable it, "depending on your account settings", and you can turn it on or off in the console or with `UpdateAutomatedDiscoveryConfiguration` ([AWS: getting started](https://docs.aws.amazon.com/macie/latest/user/getting-started.html), [AWS: enabling automated discovery](https://docs.aws.amazon.com/macie/latest/user/discovery-asdd-account-enable.html)).

**Discovery jobs.** A job runs once, for on-demand analysis, or on a daily, weekly, or monthly schedule ([AWS: discovery jobs](https://docs.aws.amazon.com/macie/latest/user/discovery-jobs.html), [AWS: job scope](https://docs.aws.amazon.com/macie/latest/user/discovery-jobs-scope.html)). Its scope has two parts:

- **Buckets.** Either a fixed list of up to 1,000 buckets, or **bucket criteria** such as account ID, bucket name, tags, or whether the bucket is public. A recurring job re-evaluates the criteria on every run, so new matching buckets are included automatically.
- **Objects.** Optional filters on file name extension, last modified date, prefix, size, and tags, plus a **sampling depth**: the percentage of eligible objects Macie picks at random.

On its first run, a recurring job can analyze all existing objects or only objects created or changed after the job was created. Later runs analyze only objects created or changed since the previous run ([AWS: job scope](https://docs.aws.amazon.com/macie/latest/user/discovery-jobs-scope.html)).

> [!warning] Gotcha: you cannot edit a job
> To change a job's settings, AWS says to cancel it, copy it, adjust the copy, and save the copy as a new job ([AWS: managing jobs](https://docs.aws.amazon.com/macie/latest/user/discovery-jobs-manage.html)). Plan a job's scope and identifiers before you create it.

## Data Identifiers

A data identifier is a rule that Macie checks object contents against. The lecture compares them to rules that your objects are assessed against. There are two kinds, and you can add allow lists to suppress known false positives.

### Managed data identifiers

Managed data identifiers are built in and maintained by AWS. They use "machine learning and pattern matching" to detect a "large and growing" list of sensitive data types across many countries and regions ([AWS: managed data identifiers](https://docs.aws.amazon.com/macie/latest/user/managed-data-identifiers.html)). AWS groups them into categories:

- **Credentials:** AWS secret access keys, and private keys such as SSH, PGP, and PKCS keys
- **Financial information:** credit card numbers and expiry dates, bank account numbers
- **Personal information:** PHI, such as health insurance and medical ID numbers, and PII, such as names, addresses, birth dates, driver's license numbers, and passport numbers

For some data types, a managed identifier reports a match only when a **keyword** is near the data. For example, a 16-digit number counts as a credit card number only near a word such as "credit card". In unstructured text, such as PDF, Word, or plain text files, the keyword typically has to be within 30 characters. In CSV, Excel, JSON, Parquet, and other structured files, the keyword has to be in the same value or in the column or field name ([AWS: keyword requirements](https://docs.aws.amazon.com/macie/latest/user/managed-data-identifiers-keywords.html)).

### Custom data identifiers

Custom data identifiers are rules that you create for data specific to your organization, such as employee IDs or customer account numbers. For common data types such as credit card numbers, use the managed identifiers instead.

A custom data identifier has one required part and three optional refiners ([AWS: custom data identifier options](https://docs.aws.amazon.com/macie/latest/user/cdis-options.html)):

| Part                       | What it does                                                      | Limits                                                         |
| -------------------------- | ----------------------------------------------------------------- | -------------------------------------------------------------- |
| **Regex** (required)       | The pattern to match in the data                                  | Up to 512 characters; a subset of PCRE                         |
| **Keywords**               | Words that must appear near a regex match for it to count         | Up to 50 keywords, 3 to 90 characters each, not case sensitive |
| **Maximum match distance** | How close, in characters, a keyword must be to the regex match    | 1 to 300 characters; default 50                                |
| **Ignore words**           | If a regex match contains an ignore word, Macie ignores the match | Up to 10 words, 4 to 90 characters each, case sensitive        |

The lecture's example regex is `[A-Z]-\d{8}`: one uppercase letter, a hyphen, then eight digits, such as `E-12345678`. On its own, that pattern could match other codes in your data, such as part numbers. Adding keywords such as `employee` and `staff id` with a short maximum match distance makes Macie report a match only when one of those words is nearby.

Macie's regex support is limited. It does not support backreferences, capturing groups, lookahead or lookbehind, or flags such as `/i` ([AWS: custom data identifier options](https://docs.aws.amazon.com/macie/latest/user/cdis-options.html)). You can also set **severity** by number of matches. By default every finding is Medium; you can instead set, for example, Low for 1 or more matches, Medium for 50 or more, and High for 100 or more. If an object has fewer matches than the lowest threshold, Macie creates no finding.

> [!warning] Gotcha: custom data identifiers cannot be changed
> AWS says "you can't change a custom data identifier after you create it" ([AWS: custom data identifier options](https://docs.aws.amazon.com/macie/latest/user/cdis-options.html)). Test the regex and refiners against sample text in the console before you save it.

### Allow lists

An **allow list** tells Macie which text to ignore even if it matches a managed or custom identifier, such as a public company phone number or sample data. A list can be **predefined text**, which Macie ignores when it matches an entry exactly, or a **regular expression**, which Macie ignores when the text matches the whole pattern ([AWS: allow lists](https://docs.aws.amazon.com/macie/latest/user/allow-lists.html)). Allow lists are not in the lecture.

## What Macie Can Read

Macie skips objects it cannot read. It still reports their metadata, but it does not analyze their contents.

**Encryption** ([AWS: supported encryption types](https://docs.aws.amazon.com/macie/latest/user/discovery-supported-encryption-types.html)):

| Object encryption                               | Can Macie analyze the contents?                                                                                                                          |
| ----------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| SSE-S3                                          | Yes                                                                                                                                                      |
| SSE-KMS or DSSE-KMS with an AWS managed key     | Yes                                                                                                                                                      |
| SSE-KMS or DSSE-KMS with a customer managed key | Only if the [key policy](kms.md#key-policy-the-resource-policy) allows the Macie service-linked role `AWSServiceRoleForAmazonMacie` to use `kms:Decrypt` |
| SSE-C                                           | No, metadata only                                                                                                                                        |
| Client-side encryption                          | No, metadata only                                                                                                                                        |

For a customer managed key in another account, the key owner must also let the bucket owner create a grant for Macie. See [S3 encryption](s3.md#object-encryption--sse-and-cse) for how these methods differ.

> [!warning] Gotcha: customer managed KMS keys block analysis by default
> If objects are encrypted with a customer managed KMS key and the key policy does not allow Macie's service-linked role to decrypt, Macie cannot read the objects and reports no sensitive data in them. Fix the key policy, not the bucket policy.

**Storage classes and formats** ([AWS: supported storage classes and formats](https://docs.aws.amazon.com/macie/latest/user/discovery-supported-storage.html)):

- Macie analyzes objects in S3 Standard, Standard-IA, One Zone-IA, Intelligent-Tiering, Glacier Instant Retrieval, and Reduced Redundancy. It does not analyze Glacier Flexible Retrieval or Glacier Deep Archive objects, S3 Express One Zone, or directory buckets.
- Supported formats include text formats (CSV, JSON, JSON Lines, TSV, XML, HTML, YAML, plain text), Avro, Parquet, PDF, Word, Excel, email (EML), and GZIP, TAR, and ZIP archives. Images, audio, and video are not analyzed.

## Findings

Macie produces two types of finding ([AWS: finding types](https://docs.aws.amazon.com/macie/latest/user/findings-types.html)).

### Policy findings

A **policy finding** means a bucket's settings changed in a way that reduces the security of the bucket or its objects. Macie reports only changes made **after** Macie was enabled. If the same change happens again, Macie updates the existing finding instead of creating a new one.

| Finding type                                  | Raised when                                                                                             |
| --------------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| `Policy:IAMUser/S3BlockPublicAccessDisabled`  | All the bucket-level [Block Public Access](s3.md#block-public-access) settings were turned off          |
| `Policy:IAMUser/S3BucketEncryptionDisabled`   | The bucket's default encryption was reset to the S3 default, SSE-S3                                     |
| `Policy:IAMUser/S3BucketPublic`               | A bucket policy or ACL now allows anonymous access, or access by all authenticated AWS identities       |
| `Policy:IAMUser/S3BucketReplicatedExternally` | Replication was set up to a bucket in an AWS account outside your Macie organization                    |
| `Policy:IAMUser/S3BucketSharedExternally`     | A bucket policy or ACL now shares the bucket with an AWS account outside your Macie organization        |
| `Policy:IAMUser/S3BucketSharedWithCloudFront` | The bucket policy now allows a [CloudFront OAI or OAC](cloudfront.md#protect-the-s3-origin-oai-and-oac) |

The lecture covers the first four. `S3BucketEncryptionDisabled` does not mean the bucket stopped encrypting. Since January 5, 2023, S3 encrypts every new object with at least SSE-S3, and you can no longer turn default encryption off. The finding means the bucket moved from a stronger default, such as SSE-KMS, back to SSE-S3. Findings of this type generated before that date meant default encryption had been disabled ([AWS: finding types](https://docs.aws.amazon.com/macie/latest/user/findings-types.html)).

### Sensitive data findings

A **sensitive data finding** means Macie found sensitive data in an S3 object, during automated discovery or a job. Every one is a new finding.

| Finding type                              | Raised when the object contains                                                       |
| ----------------------------------------- | ------------------------------------------------------------------------------------- |
| `SensitiveData:S3Object/Credentials`      | Credentials, such as AWS secret access keys or private keys                           |
| `SensitiveData:S3Object/CustomIdentifier` | Text that matches one or more of your custom data identifiers                         |
| `SensitiveData:S3Object/Financial`        | Financial information, such as credit card or bank account numbers                    |
| `SensitiveData:S3Object/Personal`         | PII, such as passport or driver's license numbers, or PHI, such as medical ID numbers |
| `SensitiveData:S3Object/Multiple`         | More than one of the categories above                                                 |

The lecture's example of `Multiple`: an object with 50 credit card numbers is `Financial`, but an object with credit card numbers and a passport number is `Multiple`.

### Retention and discovery results

Macie keeps policy and sensitive data findings for **90 days** ([AWS: finding types](https://docs.aws.amazon.com/macie/latest/user/findings-types.html)). To keep a record longer, send findings to EventBridge or Security Hub CSPM and store them there or downstream.

Separately, Macie writes a **sensitive data discovery result** for every object it analyzes or tries to analyze, including objects where it found nothing or could not read the object. These results are an audit record of what was checked. They do not contain the sensitive data itself. Macie keeps them for only 90 days and you cannot view them in the console. To keep them, configure a **discovery results repository**: an S3 bucket plus a customer managed symmetric [KMS](kms.md) key that Macie uses to encrypt the results ([AWS: storing discovery results](https://docs.aws.amazon.com/macie/latest/user/discovery-results-repository-s3.html)).

## Publishing Findings: EventBridge and Security Hub CSPM

**EventBridge.** Macie publishes both policy and sensitive data findings to EventBridge automatically. There is nothing to enable. Events have the source `aws.macie` ([AWS: EventBridge integration](https://docs.aws.amazon.com/macie/latest/user/findings-monitor-events-eventbridge.html)). Timing ([AWS: publication frequency](https://docs.aws.amazon.com/macie/latest/user/findings-publish-frequency.html)):

- A **new** finding of either type is published as soon as Macie finishes processing it.
- **Updates** to an existing policy finding are published on a recurring cycle: every 15 minutes by default, or every hour or every 6 hours (`FIFTEEN_MINUTES`, `ONE_HOUR`, `SIX_HOURS` in the [API](https://docs.aws.amazon.com/macie/latest/APIReference/macie.html)).
- Findings archived by a **suppression rule** are not published.

From EventBridge, a rule can invoke Lambda to fix the problem, for example turning Block Public Access back on, or notify a team through SNS. This is the same pattern as [GuardDuty's findings](guardduty.md#architecture).

**Security Hub CSPM.** The lecture calls this Security Hub; Macie's documentation now calls it **AWS Security Hub CSPM**. When both Macie and Security Hub CSPM are enabled, the integration turns on automatically. By default Macie publishes only **policy findings** there; publishing **sensitive data findings** is a separate setting you turn on. Findings that existed before you turned on a setting are not sent ([AWS: Security Hub CSPM integration](https://docs.aws.amazon.com/macie/latest/user/securityhub-integration.html)).

> [!warning] Gotcha: Security Hub does not get sensitive data findings by default
> The lecture says publishing to Security Hub is a setting you choose per account. More precisely, policy findings flow automatically once both services are enabled, while sensitive data findings need the "Publish sensitive data findings to Security Hub CSPM" setting. Both settings, like the publication frequency, are configured per Region.

## Multi-Account Management

A single account can use Macie on its own. With several accounts, one account becomes the Macie **administrator** and the others become **members**. The administrator can see members' bucket inventories and findings, run discovery jobs on members' buckets, and manage settings for them. There are two ways to set this up ([AWS: managing multiple accounts](https://docs.aws.amazon.com/macie/latest/user/macie-accounts.html), [AWS: quotas](https://docs.aws.amazon.com/macie/latest/user/macie-quotas.html)):

| Method                                                                             | How members join                                                                                                 | Maximum members |
| ---------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- | --------------- |
| [AWS Organizations](organizations.md) delegated administrator (recommended by AWS) | The management account designates a delegated Macie administrator, which can add any account in the organization | 10,000          |
| Invitation                                                                         | The administrator sends an invitation, and the other account accepts it                                          | 1,000           |

With Organizations, the administrator sees every account in the organization, including accounts that have not enabled Macie yet. With invitations, it sees only the accounts it has added.

Macie is a **Regional** service. Findings, publication settings, and the discovery results repository are all configured per Region, and each quota applies per account per Region ([AWS: quotas](https://docs.aws.amazon.com/macie/latest/user/macie-quotas.html)). To cover buckets in several Regions, enable Macie in each one.

## Pricing

Macie charges for three things. The examples below are US East (N. Virginia) prices from the pricing page, checked on 2026-11-23 ([AWS: Macie pricing](https://aws.amazon.com/macie/pricing/)):

| Dimension                                         | Example price                                                      |
| ------------------------------------------------- | ------------------------------------------------------------------ |
| S3 buckets evaluated for inventory and monitoring | $0.10 per bucket per month                                         |
| S3 objects monitored for automated discovery      | $0.01 per 100,000 objects per month                                |
| Data inspected (automated discovery and jobs)     | $1 per GB; the first 1 GB per month of automated discovery is free |

- A **30-day free trial** applies when you first enable Macie. It covers bucket inventory and monitoring and automated discovery, up to 150 GB inspected per account. It does **not** cover discovery jobs.
- By default, discovery jobs can analyze up to 5 TB per account per month. You can raise this quota ([AWS: quotas](https://docs.aws.amazon.com/macie/latest/user/macie-quotas.html)).

Automated discovery samples objects, so it costs far less than a job that reads every object in large buckets. Use jobs where you need full coverage of specific buckets.

## Macie and Related Security Services

| Service                          | Job                                                                                      |
| -------------------------------- | ---------------------------------------------------------------------------------------- |
| **Macie**                        | Find sensitive data in S3 objects and report bucket settings that reduce security        |
| [GuardDuty](guardduty.md)        | Detect threats and malicious activity, including suspicious S3 access with S3 Protection |
| [AWS Config](config.md)          | Record resource configuration and check it against rules                                 |
| [CloudTrail](cloudtrail.md)      | Record API calls: who did what, when                                                     |
| [Amazon Inspector](inspector.md) | Scan EC2, container images, and Lambda for software vulnerabilities                      |
| AWS Security Hub CSPM            | Collect findings from Macie, GuardDuty, Inspector, Config, and others in one place       |

Macie and GuardDuty are easy to confuse because both can report on S3. Macie looks at **what data is stored** and how the bucket is configured. GuardDuty looks at **who is accessing it** and whether that activity looks malicious.

## Exam Recap

| Concept                 | Key fact                                                                                                    |
| ----------------------- | ----------------------------------------------------------------------------------------------------------- |
| What it is              | Data security and privacy service: find PII, PHI, financial data, and credentials in **S3** only            |
| How it detects          | Managed data identifiers (ML and patterns, maintained by AWS) and custom data identifiers (regex you write) |
| Custom identifier parts | Regex, plus optional keywords, maximum match distance, and ignore words; cannot be edited after creation    |
| Discovery               | Automated discovery (continuous sampling) or discovery jobs (one-time, daily, weekly, monthly)              |
| Finding types           | Policy findings (bucket settings) and sensitive data findings (object contents)                             |
| Prevents leaks?         | **No.** Detective only                                                                                      |
| Response                | Findings to EventBridge automatically, then Lambda to remediate or SNS to notify                            |
| Security Hub CSPM       | Policy findings by default; sensitive data findings optional                                                |
| Multi-account           | Delegated administrator through Organizations (10,000 members) or invitations (1,000)                       |
| Scope                   | Regional; findings kept 90 days                                                                             |

> [!tip] Exam Tip
>
> | Requirement | Answer |
> | --- | --- |
> | Discover and classify PII, PHI, or financial data stored in S3 | Macie |
> | Detect a company-specific data pattern, such as employee IDs, in S3 | Macie custom data identifier (regex, optionally with keywords) |
> | Automatically remediate when sensitive data or a public bucket is found | Macie finding to EventBridge, then a rule to Lambda |
> | Alert when a bucket becomes public or is shared with another account | Macie policy findings (or AWS Config rules) |
> | Find sensitive data across every account in an organization | Macie delegated administrator through AWS Organizations |
> | Detect malicious access to S3 or unusual API calls | GuardDuty, not Macie |
> | Find sensitive data in an RDS database or on EBS volumes | Not Macie directly; Macie analyzes S3 only |

## References

- Adrian Cantrill, AWS Certified Solutions Architect – Associate (SAA-C03): "Amazon Macie"
- [AWS: What is Amazon Macie?](https://docs.aws.amazon.com/macie/latest/user/what-is-macie.html)
- [AWS: getting started with Macie](https://docs.aws.amazon.com/macie/latest/user/getting-started.html)
- [AWS: automated sensitive data discovery](https://docs.aws.amazon.com/macie/latest/user/discovery-asdd.html)
- [AWS: automated discovery default settings](https://docs.aws.amazon.com/macie/latest/user/discovery-asdd-settings-defaults.html)
- [AWS: enabling automated discovery](https://docs.aws.amazon.com/macie/latest/user/discovery-asdd-account-enable.html)
- [AWS: sensitivity scoring for S3 buckets](https://docs.aws.amazon.com/macie/latest/user/discovery-scoring-s3.html)
- [AWS: sensitive data discovery jobs](https://docs.aws.amazon.com/macie/latest/user/discovery-jobs.html)
- [AWS: job scope options](https://docs.aws.amazon.com/macie/latest/user/discovery-jobs-scope.html)
- [AWS: managing jobs](https://docs.aws.amazon.com/macie/latest/user/discovery-jobs-manage.html)
- [AWS: managed data identifiers](https://docs.aws.amazon.com/macie/latest/user/managed-data-identifiers.html)
- [AWS: keyword requirements for managed data identifiers](https://docs.aws.amazon.com/macie/latest/user/managed-data-identifiers-keywords.html)
- [AWS: custom data identifiers](https://docs.aws.amazon.com/macie/latest/user/custom-data-identifiers.html)
- [AWS: custom data identifier options](https://docs.aws.amazon.com/macie/latest/user/cdis-options.html)
- [AWS: allow lists](https://docs.aws.amazon.com/macie/latest/user/allow-lists.html)
- [AWS: supported encryption types](https://docs.aws.amazon.com/macie/latest/user/discovery-supported-encryption-types.html)
- [AWS: supported storage classes and formats](https://docs.aws.amazon.com/macie/latest/user/discovery-supported-storage.html)
- [AWS: finding types](https://docs.aws.amazon.com/macie/latest/user/findings-types.html)
- [AWS: storing sensitive data discovery results](https://docs.aws.amazon.com/macie/latest/user/discovery-results-repository-s3.html)
- [AWS: EventBridge integration](https://docs.aws.amazon.com/macie/latest/user/findings-monitor-events-eventbridge.html)
- [AWS: publication frequency for findings](https://docs.aws.amazon.com/macie/latest/user/findings-publish-frequency.html)
- [AWS: Macie API reference, Macie session (`FindingPublishingFrequency`)](https://docs.aws.amazon.com/macie/latest/APIReference/macie.html)
- [AWS: Security Hub CSPM integration](https://docs.aws.amazon.com/macie/latest/user/securityhub-integration.html)
- [AWS: managing multiple Macie accounts](https://docs.aws.amazon.com/macie/latest/user/macie-accounts.html)
- [AWS: Macie quotas](https://docs.aws.amazon.com/macie/latest/user/macie-quotas.html)
- [AWS: Macie pricing](https://aws.amazon.com/macie/pricing/)
