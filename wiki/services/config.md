---
title: AWS Config
type: service
tags: [config, config-rules, configuration-item, configuration-history, compliance, auditing, remediation, ssm-automation, eventbridge, sns, aggregator, conformance-packs, governance, security, pricing]
certs: [aws-saa]
updated: 2026-11-13
---

# AWS Config

AWS Config records how the configuration of your AWS resources changes over time and checks those configurations against standards you define. It has two jobs:

1. **Record configuration changes** on resources, so you can see what a resource looked like at any point in time and audit what changed.
2. **Evaluate compliance** of those resources against **Config rules**, and report each resource as compliant or non-compliant.

AWS Config is easy to misunderstand because it sounds like a control. It is not. It is a **detective** service: it observes and reports, and it **does not prevent changes**. Blocking an action is the job of [IAM policies](iam.md) and [SCPs](organizations.md#service-control-policies-scps). This is the same split [Control Tower](control-tower.md#axis-2--functional-type) uses for its guardrails: preventative controls are SCPs, detective controls are Config rules.

Config sits alongside two other governance and observability services. [CloudTrail](cloudtrail.md#where-cloudtrail-fits) answers who made an API call, [CloudWatch](cloudwatch.md) answers whether a workload is healthy, and Config answers what a resource's configuration is and how it changed. [GuardDuty](guardduty.md) is a different kind of detective service: it looks for threats and malicious activity, not for misconfigured resources.

## Architecture

![AWS Config architecture. Standard features on the left: account resources send configs to the AWS Config recorder, which uses periodic or continuous change recording and delivers to a Config bucket; every change creates a configuration item (CI), and all CIs for a resource form its configuration history in S3. Optional features on the right: resources are evaluated against Config rules (AWS managed or custom with Lambda or Guard) and marked compliant or non-compliant; Config sends a configuration stream and compliance notifications to SNS for people and applications, compliance status changes to EventBridge for Lambda remediation, and uses SSM Automation documents for native remediation](../assets/aws-config-architecture.png)

The diagram splits Config into two halves.

**Standard features (left).** You enable Config per account and per Region. Enabling it starts the **configuration recorder**, which tracks every supported resource type you choose to record. Each change produces a **configuration item (CI)**, and the recorder delivers the data to an S3 **Config bucket**. If you stop here, you already have a full change history of your resources.

**Optional features and integration (right).**

- **Config rules** define what a resource should look like. Each evaluated resource is marked **compliant** or **non-compliant**.
- **[SNS](sns.md)** receives a **configuration stream** (every change) and **compliance notifications**, for people or applications to act on.
- **[EventBridge](eventbridge.md)** receives an event whenever a resource's **compliance status changes**. A rule can invoke [Lambda](lambda.md) to fix the resource.
- **[SSM](ssm.md) Automation documents** are Config's own **native remediation** feature, with no EventBridge or Lambda needed.

## Configuration Items and History

A **configuration item (CI)** is the configuration of one resource at one point in time. AWS describes it as containing metadata, attributes, the resource's **relationships** to other resources, its current configuration, and **related events** ([AWS: Config concepts](https://docs.aws.amazon.com/config/latest/developerguide/config-concepts.html)). All CIs for one resource make up its **configuration history**.

The lecture's example is a security group attached to an EC2 instance. If someone adds an inbound rule, Config records:

- the configuration before the change and after it
- the relationship to the attached EC2 instance
- the related CloudTrail event, so the timeline links to the API call that caused the change

Config can show this as a timeline for the resource. Config shows **what** changed and links to the CloudTrail event. CloudTrail itself is the record of **who** made the call.

You can read the data in three ways:

- in the **Config console** and through the **Config APIs**
- as files delivered to the **S3 bucket**: configuration history files and configuration snapshots
- with **advanced queries**, which run SQL `SELECT` statements over current resource configurations in one account and Region, or across an aggregator ([AWS: advanced queries](https://docs.aws.amazon.com/config/latest/developerguide/querying-AWS-resources.html))

Config keeps CIs for **7 years (2,557 days)** by default. You can set a shorter retention period, with one retention configuration per Region ([AWS: deleting Config data](https://docs.aws.amazon.com/config/latest/developerguide/delete-config-data-with-retention-period.html)).

## Recording

### What is recorded

The recorder records either all supported resource types in the Region or only the types you select. Most resources are **regional**, so each Region needs its own recorder.

> [!warning] Gotcha: global resources and duplicate CIs
> IAM users, groups, roles, and policies are global resources. If every Region's recorder includes them, each Region records the same IAM changes and you pay for the duplicate CIs. Record global resource types in **one Region** only ([AWS: recording global resources](https://docs.aws.amazon.com/config/latest/developerguide/select-resources.html)).

### Continuous or periodic

| Mode                     | What it records                                                                                         | Price per CI |
| ------------------------ | ------------------------------------------------------------------------------------------------------- | ------------ |
| **Continuous** (default) | A CI for every change, shortly after it happens                                                         | $0.003       |
| **Periodic** (daily)     | One CI per resource per 24 hours, showing the latest state, and only if it differs from the previous CI | $0.012       |

Prices are from [AWS Config pricing](https://aws.amazon.com/config/pricing/). Periodic recording was added in November 2023 ([AWS: periodic recording](https://aws.amazon.com/about-aws/whats-new/2023/11/aws-config-periodic-recording)). AWS describes a daily CI as "the most recent state of your resources over the last 24-hour period, only if it's different from the previous CI recorded" ([AWS: recording frequency](https://docs.aws.amazon.com/config/latest/developerguide/managing-recorder_console-change-recording-frequency.html)).

The lecture presents periodic recording as change-by-change detail traded for "daily detail with a lower cost". That is only partly right. A periodic CI costs **four times** as much as a continuous one, so periodic recording saves money only on resources that change **more than four times a day**. For a resource that changes once a day or less, periodic recording costs more. You can also set the frequency per resource type.

> [!warning] Gotcha: periodic recording and Firewall Manager
> AWS Firewall Manager depends on continuous recording. If you use Firewall Manager, keep the recorder on Continuous ([AWS: recording frequency](https://docs.aws.amazon.com/config/latest/developerguide/managing-recorder_console-change-recording-frequency.html)).

### Service-linked recorders

Since December 2024, an AWS service can create a **service-linked recorder** that records only the resources that service needs. It runs independently of your own recorder, and you cannot change its settings directly. The first one was for CloudWatch telemetry configurations, at no extra cost ([AWS: service-linked recorder](https://aws.amazon.com/about-aws/whats-new/2024/12/aws-config-service-linked-recorder)).

## Config Rules

A Config rule spells out what a compliant resource looks like. Config evaluates resources in the rule's scope and marks each one **compliant** or **non-compliant**.

### Rule types

| Type              | Who writes it                                                   | Example                                                                             |
| ----------------- | --------------------------------------------------------------- | ----------------------------------------------------------------------------------- |
| **AWS managed**   | AWS; you set parameters                                         | `s3-bucket-public-read-prohibited`, `encrypted-volumes`, `root-account-mfa-enabled` |
| **Custom Guard**  | You, in AWS CloudFormation **Guard**, a policy-as-code language | A rule checking that a resource has specific property values                        |
| **Custom Lambda** | You, as a Lambda function in any supported runtime              | Logic Guard cannot express, such as calling an external system                      |

The lecture's tagging example is that every resource must have `CostCenter` and `Owner` tags, and `CostCenter` must be one of the values the accounting team approved. You do not need custom code for this. The AWS managed rule **`required-tags`** checks up to six tag keys, each with an optional comma-separated list of allowed values ([AWS: required-tags](https://docs.aws.amazon.com/config/latest/developerguide/required-tags.html)). Use a custom rule when the check is more complex than that.

### When rules run

A rule's **trigger type** decides when it evaluates ([AWS: Config rule components](https://docs.aws.amazon.com/config/latest/developerguide/evaluate-config-rules.html)):

- **Configuration changes:** runs when a resource in scope changes
- **Periodic:** runs on a schedule (every 1, 3, 6, 12, or 24 hours)
- **Hybrid:** both

> [!warning] Gotcha: two different "periodic" settings
> **Periodic recording** controls how often the recorder creates CIs. A **periodic trigger** controls how often a rule evaluates. They are separate settings.

### Detective and proactive evaluation

Rules normally run in **detective** mode, against resources that already exist. Since November 2022, rules that support it can also run in **proactive** mode, which evaluates a resource configuration **before** it is deployed ([AWS: proactive compliance](https://aws.amazon.com/blogs/aws/new-aws-config-rules-now-support-proactive-compliance/)).

This is what the lecture's slide means by "does not prevent changes … but can proactively warn". Proactive evaluation only returns a result. AWS states that proactive rules "do not remediate resources that are flagged as NON_COMPLIANT or prevent them from being deployed" ([AWS: Config rule components](https://docs.aws.amazon.com/config/latest/developerguide/evaluate-config-rules.html)). The caller decides what to do with the result. For example, a deployment pipeline or a CloudFormation hook can stop a deployment when Config says the resource would be non-compliant. Config itself still blocks nothing.

### Conformance packs

A **conformance pack** is a collection of Config rules and remediation actions, defined in a YAML template and deployed as one unit to an account and Region, or across an [Organization](organizations.md). AWS provides sample packs for common frameworks ([AWS: conformance packs](https://docs.aws.amazon.com/config/latest/developerguide/conformance-packs.html)).

## Notifications and Remediation

### SNS and EventBridge

- **SNS:** the delivery channel can send a **configuration stream** and **compliance notifications** to an SNS topic.
- **EventBridge:** Config sends an event when a rule changes a resource's compliance state, in either direction (compliant to non-compliant, or non-compliant to compliant). An EventBridge rule can then call a target such as Lambda.

The lecture's example: a Config rule defines what a security group should look like. Someone adds an inbound rule allowing SSH from `0.0.0.0/0`. Config marks the security group non-compliant and sends the change to EventBridge. EventBridge invokes a Lambda function that removes the rule and returns the security group to a compliant state.

The EventBridge and Lambda pattern is common, but it is **not part of Config**. EventBridge and Lambda are separate services that react to Config's events.

### Native remediation with SSM Automation

Config has its own remediation feature. You attach a remediation action to a rule, and Config runs an **SSM Automation document** against non-compliant resources ([AWS: remediation](https://docs.aws.amazon.com/config/latest/developerguide/remediation.html)).

- Use an AWS-provided document (there are many prebuilt ones) or write your own.
- Run it **manually** from the console, or **automatically** when a resource becomes non-compliant.
- Automatic remediation retries a failed action up to a set number of attempts (default 5) within a time window (default 60 seconds). After that, Config adds a remediation exception for the resource ([AWS: RemediationConfiguration](https://docs.aws.amazon.com/config/latest/APIReference/API_RemediationConfiguration.html)).

The lecture's example is an Automation document that installs missing critical security patches on an EC2 instance.

| Approach                            | Part of Config? | Use when                                                  |
| ----------------------------------- | --------------- | --------------------------------------------------------- |
| Remediation action (SSM Automation) | Yes             | A prebuilt or custom runbook can fix the resource         |
| EventBridge rule to Lambda          | No              | You need custom logic, or to notify or call other systems |

## Multi-Account and Multi-Region

Config is a **regional** service: a recorder covers one account in one Region. To see everything in one place, create an **aggregator**. An aggregator collects configuration and compliance data from multiple accounts and Regions, or from every account in an Organization, into one aggregator account ([AWS: multi-account multi-Region aggregation](https://docs.aws.amazon.com/config/latest/developerguide/aggregate-data.html)).

An aggregator is a **read-only view**. It does not deploy rules or remediate. To deploy rules across an Organization, use organization Config rules or conformance packs. [Control Tower](control-tower.md) does this for you: it enables Config in enrolled accounts, deploys detective controls as Config rules, and stores Config data in the Log Archive account.

Other AWS services build on Config. AWS Security Hub CSPM runs many of its controls as Config rules, and since June 2026 AWS services can deploy **internal service-linked rules** whose results go to that service at no charge to you ([AWS: internal service-linked rules](https://aws.amazon.com/about-aws/whats-new/2026/06/aws-config-supports-internal-service-linked-rules)).

## Pricing

Config has no upfront fee. You pay for three things ([AWS Config pricing](https://aws.amazon.com/config/pricing/)):

| Item                        | Price                                                                             |
| --------------------------- | --------------------------------------------------------------------------------- |
| Configuration item recorded | $0.003 continuous, $0.012 periodic                                                |
| Config rule evaluation      | $0.001 for the first 100,000, $0.0008 for the next 400,000, $0.0005 after 500,000 |
| Conformance pack evaluation | Same tiers as rule evaluations                                                    |

An evaluation is one rule checking one resource. A 10-rule conformance pack checking 50 resources is 500 evaluations each time it runs. Lambda invocations for custom rules, S3 storage, and SNS messages are billed by those services.

## Exam Recap

| Concept           | Key fact                                                                          |
| ----------------- | --------------------------------------------------------------------------------- |
| Primary job       | Record resource configuration changes over time (configuration items and history) |
| Second job        | Evaluate resources against Config rules: compliant or non-compliant               |
| Prevents changes? | **No.** Detective only. To block actions use IAM or SCPs                          |
| Scope             | Regional, per account; aggregator for multi-account and multi-Region views        |
| Storage           | CIs kept 7 years by default; history files and snapshots delivered to S3          |
| Recording         | Continuous (default) or periodic (daily, only if changed)                         |
| Rule types        | AWS managed, or custom with Guard or Lambda                                       |
| Rule triggers     | Configuration change, periodic, or both                                           |
| Notifications     | SNS for configuration stream and compliance notifications                         |
| Events            | Compliance state changes go to EventBridge                                        |
| Remediation       | Native SSM Automation documents (manual or automatic), or EventBridge to Lambda   |

> [!tip] Exam Tip
>
> | Requirement | Answer |
> | --- | --- |
> | What did this resource's configuration look like last week? | Config configuration history |
> | Who changed the security group? | CloudTrail (Config links to the CloudTrail event) |
> | Detect S3 buckets that are public or EBS volumes that are not encrypted | Config managed rules |
> | Automatically fix a non-compliant resource | Config remediation with SSM Automation, or EventBridge to Lambda |
> | Alert a team when a resource becomes non-compliant | Config compliance change to EventBridge or SNS |
> | Prevent a non-compliant change from happening | IAM policy or SCP, not Config |
> | One compliance view across all accounts and Regions | Config aggregator |
> | Deploy a standard set of rules and remediations across an Organization | Conformance packs |

## References

- Adrian Cantrill, AWS Certified Solutions Architect – Associate (SAA-C03): "AWS Config"
- [AWS Config pricing](https://aws.amazon.com/config/pricing/)
- [AWS: Config concepts](https://docs.aws.amazon.com/config/latest/developerguide/config-concepts.html)
- [AWS: recording frequency](https://docs.aws.amazon.com/config/latest/developerguide/managing-recorder_console-change-recording-frequency.html)
- [AWS: periodic recording launch](https://aws.amazon.com/about-aws/whats-new/2023/11/aws-config-periodic-recording)
- [AWS: recording global resources](https://docs.aws.amazon.com/config/latest/developerguide/select-resources.html)
- [AWS: service-linked recorder](https://aws.amazon.com/about-aws/whats-new/2024/12/aws-config-service-linked-recorder)
- [AWS: Config rule components](https://docs.aws.amazon.com/config/latest/developerguide/evaluate-config-rules.html)
- [AWS: proactive compliance](https://aws.amazon.com/blogs/aws/new-aws-config-rules-now-support-proactive-compliance/)
- [AWS: required-tags](https://docs.aws.amazon.com/config/latest/developerguide/required-tags.html)
- [AWS: conformance packs](https://docs.aws.amazon.com/config/latest/developerguide/conformance-packs.html)
- [AWS: remediation](https://docs.aws.amazon.com/config/latest/developerguide/remediation.html)
- [AWS: RemediationConfiguration API](https://docs.aws.amazon.com/config/latest/APIReference/API_RemediationConfiguration.html)
- [AWS: multi-account multi-Region aggregation](https://docs.aws.amazon.com/config/latest/developerguide/aggregate-data.html)
- [AWS: advanced queries](https://docs.aws.amazon.com/config/latest/developerguide/querying-AWS-resources.html)
- [AWS: deleting Config data (retention)](https://docs.aws.amazon.com/config/latest/developerguide/delete-config-data-with-retention-period.html)
- [AWS: internal service-linked rules](https://aws.amazon.com/about-aws/whats-new/2026/06/aws-config-supports-internal-service-linked-rules)
