---
title: Amazon GuardDuty
type: service
tags: [guardduty, threat-detection, security, findings, threat-intelligence, machine-learning, vpc-flow-logs, cloudtrail, dns-logs, eventbridge, lambda, sns, remediation, organizations, delegated-administrator, protection-plans]
certs: [aws-saa]
updated: 2026-11-13
---

# Amazon GuardDuty

Amazon GuardDuty is a **continuous security monitoring** service. Once you enable it, it keeps analyzing activity in your account, looking for anything **unexpected or unauthorized**: a compromised EC2 instance talking to a known command-and-control server, credentials used from an unusual location, an instance mining cryptocurrency, or a burst of S3 deletions.

GuardDuty combines three things to decide what is suspicious:

1. **Supported data sources**, such as [CloudTrail](cloudtrail.md) events, [VPC Flow Logs](vpc-flow-logs.md), and DNS query logs.
2. **Machine learning and anomaly detection.** GuardDuty learns what normal activity looks like in your account and uses it as a baseline. You do not have to tell it which operations you normally perform.
3. **Threat intelligence feeds**, which are lists of known malicious IP addresses and domains from AWS and third parties.

Anything that needs your attention is a **finding**. GuardDuty only reports findings. It does not block traffic or change resources by itself. It is a **detective** service, like [AWS Config](config.md): to act on a finding, you send it to [EventBridge](eventbridge.md) and let other services notify someone or fix the problem.

GuardDuty fits next to the other monitoring services. [CloudTrail](cloudtrail.md#where-cloudtrail-fits) records who made which API call, Config records what a resource looks like and whether it is compliant, and GuardDuty looks across that activity for signs of an attack. In the defense-in-depth picture on the [overview](../overview.md#security-is-defense-in-depth), [WAF](waf.md), [Shield](shield.md), security groups, and NACLs try to stop an attack, and GuardDuty detects the attacks and compromises that get past them.

## Architecture

![Amazon GuardDuty architecture. On the left, four data sources feed the GuardDuty service: DNS logs, VPC Flow Logs, CloudTrail management events, and CloudTrail S3 data events. Threat intelligence feeds also feed GuardDuty from above. GuardDuty sends findings to EventBridge, which sends notifications to SNS and invokes Lambda functions](../assets/guardduty-architecture.png)

The diagram reads left to right.

**Data sources (left).** The lecture lists four inputs:

- **DNS logs**: the DNS requests made through the Route 53 Resolver in your VPCs.
- **VPC Flow Logs**: metadata about traffic flowing through a VPC, such as source and destination IP addresses and ports.
- **CloudTrail management events**: control plane API calls, such as `RunInstances` or `CreateUser`.
- **CloudTrail S3 data events**: object-level calls, such as `GetObject` and `DeleteObject`. GuardDuty analyzes these only if you enable its **S3 Protection** plan.

**Threat intelligence feeds (top).** GuardDuty compares the activity in those sources against known malicious IP addresses and domains.

**Findings to EventBridge (right).** GuardDuty sends each finding to EventBridge, and an EventBridge rule routes it:

- to **[SNS](sns.md)**, to notify the security team or an external security system
- to **[Lambda](lambda.md)**, to call AWS APIs and remediate automatically

The lecture's remediation example is a Lambda function that adds an explicit **deny** rule for an attacker's IP address to a [network ACL](vpc.md#network-acl-nacl). A NACL is the right place for this because [security groups](vpc.md#security-groups-sg) can only allow traffic. They have no deny rules.

## Data Sources

### Foundational data sources

When you enable GuardDuty, it starts analyzing three **foundational data sources** straight away: CloudTrail management events, VPC Flow Logs for EC2 instances, and Route 53 Resolver DNS query logs ([AWS: foundational data sources](https://docs.aws.amazon.com/guardduty/latest/ug/guardduty_data-sources.html)).

The lecture's diagram makes it look as if you deliver your logs to GuardDuty. That is not how it works. GuardDuty pulls its **own independent copy** of each stream directly from the AWS service. AWS describes the flow log feed as "an independent and duplicative stream of flow logs" ([AWS: foundational data sources](https://docs.aws.amazon.com/guardduty/latest/ug/guardduty_data-sources.html)).

> [!warning] Gotcha: you do not enable Flow Logs or a trail for GuardDuty
> GuardDuty does not need a CloudTrail trail, a VPC flow log, or Resolver query logging. Enabling GuardDuty is enough. It also does not store these logs for you. If you need the logs for your own investigation or compliance, configure [CloudTrail trails](cloudtrail.md#trails--the-unit-of-configuration) and [VPC Flow Logs](vpc-flow-logs.md) separately. The foundational sources cannot be turned off without disabling GuardDuty ([AWS: getting started](https://docs.aws.amazon.com/guardduty/latest/ug/guardduty_settingup.html)).

> [!warning] Gotcha: DNS findings need the VPC resolver
> GuardDuty sees DNS queries only when instances use the AWS-provided resolver, which is the default ([Route 53 Resolver](route53.md#route-53-resolver) at VPC base + 2). If instances use your own DNS servers or a public resolver such as Google DNS or OpenDNS, GuardDuty cannot see those queries and produces no DNS-based findings ([AWS: foundational data sources](https://docs.aws.amazon.com/guardduty/latest/ug/guardduty_data-sources.html)).

### Protection plans

The lecture notes that its list of sources is from the time it was recorded. GuardDuty has since added more optional **protection plans**, each covering a specific type of resource ([AWS: What is GuardDuty?](https://docs.aws.amazon.com/guardduty/latest/ug/what-is-guardduty.html)):

| Protection plan        | What it analyzes                                                                                                                                                                                                                |
| ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **S3 Protection**      | CloudTrail S3 data events, to detect data exfiltration or destruction in [S3](s3.md) buckets. You do not need to enable S3 data events in a trail                                                                               |
| **EKS Protection**     | [EKS](eks.md) audit logs                                                                                                                                                                                                        |
| **Runtime Monitoring** | Operating system events (processes, files, network connections) on [EKS](eks.md), [ECS](ecs.md) (including Fargate), and [EC2](ec2.md), through a GuardDuty security agent                                                      |
| **Malware Protection** | For EC2: scans the [EBS](ebs.md) volumes attached to an instance when GuardDuty sees suspicious behavior. For S3: scans newly uploaded objects. For [AWS Backup](aws-backup.md): scans EBS snapshots, AMIs, and recovery points |
| **RDS Protection**     | Login activity, and data activity such as exfiltration, for [Aurora](aurora.md) and supported [RDS](rds.md) engines                                                                                                             |
| **Lambda Protection**  | Network activity from [Lambda](lambda.md) function executions                                                                                                                                                                   |
| **AI Protection**      | Threats to AI workloads that use Amazon Bedrock and Amazon SageMaker AI                                                                                                                                                         |

When you first enable GuardDuty, most protection plans are turned on and included in the free trial. Runtime Monitoring and Malware Protection for AWS Backup are not enabled by default ([AWS: protection plans and Extended Threat Detection](https://aws.amazon.com/blogs/security/navigating-amazon-guardduty-protection-plans-and-extended-threat-detection/)).

## Findings

A **finding** is GuardDuty's report of one potential security issue. Each finding has a type, the affected resource, the details of the activity, and a severity.

### Finding types

Finding type names follow the pattern `ThreatPurpose:ResourceType/ThreatName`. Some examples ([AWS: finding types](https://docs.aws.amazon.com/guardduty/latest/ug/guardduty_finding-types-active.html)):

| Finding type                           | Meaning                                                        |
| -------------------------------------- | -------------------------------------------------------------- |
| `UnauthorizedAccess:EC2/SSHBruteForce` | An EC2 instance is part of an SSH brute-force attack           |
| `Recon:EC2/PortProbeUnprotectedPort`   | A known malicious host is probing an open port on EC2          |
| `CryptoCurrency:EC2/BitcoinTool.B!DNS` | An EC2 instance is querying a domain used for crypto mining    |
| `Backdoor:EC2/C&CActivity.B!DNS`       | An EC2 instance is querying a known command-and-control domain |

### Extended Threat Detection

A single finding shows one suspicious event. An attacker usually takes several steps: steal credentials, look around, then take data. **Extended Threat Detection** correlates related findings and events over time into one **attack sequence** finding, with a timeline and a mapping to MITRE ATT&CK tactics. It is enabled automatically at no extra cost, and since December 2025 it also covers groups of EC2 instances and ECS clusters ([AWS: Extended Threat Detection for EC2 and ECS](https://aws.amazon.com/about-aws/whats-new/2025/12/guardduty-extended-threat-detection-ec2-ecs/)). Attack sequence findings have **Critical** severity ([AWS: attack sequence finding types](https://docs.aws.amazon.com/guardduty/latest/ug/guardduty-attack-sequence-finding-types.html)).

### Delivery and retention

- **EventBridge.** GuardDuty sends **new** findings to EventBridge in near real time. Later occurrences of an **existing** finding are sent every **6 hours** by default, and you can change this to 1 hour or 15 minutes ([AWS: findings with EventBridge](https://docs.aws.amazon.com/guardduty/latest/ug/guardduty_findings_eventbridge.html)).
- **Retention.** GuardDuty keeps findings for **90 days**. To keep them longer, export them to an S3 bucket, encrypted with a KMS key ([AWS: exporting findings](https://docs.aws.amazon.com/guardduty/latest/ug/guardduty_exportfindings.html)).

## Tuning GuardDuty

The lecture says you can influence what GuardDuty treats as normal, for example by adding IP addresses to allow lists. There are two kinds of custom lists ([AWS: entity lists and IP address lists](https://docs.aws.amazon.com/guardduty/latest/ug/guardduty_upload-lists.html)):

- **Trusted lists**: GuardDuty does not generate findings for activity involving these entries, such as your own office or partner IP addresses. You can have one trusted list per account per Region.
- **Threat lists**: GuardDuty generates findings for activity involving these entries, such as IP addresses from your own threat intelligence. You can have up to six threat lists per account per Region.

The original lists accepted only IP addresses. Since September 2025, **entity lists** can contain IP addresses, domain names, or both ([AWS: custom entity lists](https://aws.amazon.com/about-aws/whats-new/2025/09/threat-detection-amazon-guardduty-custom-entity-lists/)). If the same entry is in both a trusted list and a threat list, the trusted list wins.

**Suppression rules** are the other tuning tool. A suppression rule automatically archives findings that match a filter, for example findings from a vulnerability scanner you run on purpose. The findings are still generated, but they do not reach EventBridge or appear as active findings.

## Multi-Account and Multi-Region

GuardDuty supports multiple accounts with an **administrator** account and **member** accounts. The administrator account can view and manage findings for all its members, which gives you one place to manage threat detection.

There are two ways to set it up ([AWS: GuardDuty with Organizations](https://docs.aws.amazon.com/guardduty/latest/ug/guardduty_organizations.html)):

1. **[AWS Organizations](organizations.md) (recommended).** From the management account, choose a **delegated GuardDuty administrator** account. That account can enable and manage GuardDuty for every account in the organization, and can **auto-enable** GuardDuty for new accounts. Up to **50,000** member accounts.
2. **Invitation.** The administrator invites accounts one at a time, and each account accepts. Up to **5,000** member accounts ([AWS: GuardDuty quotas](https://docs.aws.amazon.com/guardduty/latest/ug/guardduty_limits.html)).

> [!warning] Gotcha: GuardDuty is regional
> GuardDuty, its findings, and the delegated administrator setting are all **per Region**. Set up GuardDuty and its multi-account configuration in every Region you want to monitor. AWS recommends enabling GuardDuty in **all supported Regions**, including ones you do not use, so it can detect activity there, such as an attacker launching instances in an unused Region ([AWS: getting started](https://docs.aws.amazon.com/guardduty/latest/ug/guardduty_settingup.html)).

To stop a member account from turning GuardDuty off, use an [SCP](organizations.md#typical-uses) that denies the GuardDuty disable actions.

## Suspend or Disable

| Action      | What happens                                                                                                   |
| ----------- | -------------------------------------------------------------------------------------------------------------- |
| **Suspend** | GuardDuty stops analyzing and stops billing. Existing findings and settings are kept, and you can resume later |
| **Disable** | Existing findings and settings are **deleted** and cannot be recovered. Export findings to S3 first            |

Both actions apply to one Region at a time ([AWS: suspending or disabling GuardDuty](https://docs.aws.amazon.com/guardduty/latest/ug/guardduty_suspend-disable.html)).

## Pricing

Each account gets a **30-day free trial** in each Region the first time it enables GuardDuty there. After that, you pay for the volume of data analyzed: the number of CloudTrail events and the volume of flow log and DNS log data, plus usage of each protection plan you enable. The console shows estimated costs during the trial ([AWS: GuardDuty pricing](https://docs.aws.amazon.com/guardduty/latest/ug/guardduty-pricing.html)).

## GuardDuty and Related Security Services

The exam often lists several security services as answer options. Each does a different job:

| Service                          | Job                                                                                      |
| -------------------------------- | ---------------------------------------------------------------------------------------- |
| **GuardDuty**                    | Detect threats and malicious activity from logs and runtime activity                     |
| [AWS Config](config.md)          | Record resource configuration and check it against rules                                 |
| [CloudTrail](cloudtrail.md)      | Record API calls: who did what, when                                                     |
| [Amazon Inspector](inspector.md) | Scan EC2, container images, and Lambda for software vulnerabilities and network exposure |
| [Amazon Macie](macie.md)         | Find sensitive data, such as personal data, in S3                                        |
| Amazon Detective                 | Investigate the root cause of a finding using graphs of related activity                 |
| AWS Security Hub                 | Collect findings from GuardDuty, Inspector, Macie, Config, and others in one place       |
| [AWS WAF](waf.md)                | Block malicious HTTP requests                                                            |
| [AWS Shield](shield.md)          | Protect against DDoS attacks                                                             |

GuardDuty findings can also feed [OpenSearch](opensearch.md#use-cases) for SIEM-style analysis.

## Exam Recap

| Concept              | Key fact                                                                                            |
| -------------------- | --------------------------------------------------------------------------------------------------- |
| What it is           | Continuous threat detection: ML and anomaly detection plus threat intelligence feeds                |
| Foundational sources | CloudTrail management events, VPC Flow Logs, Route 53 Resolver DNS logs                             |
| Setup of sources     | None. GuardDuty reads its own independent streams; you do not enable Flow Logs or a trail           |
| Optional sources     | Protection plans: S3 (CloudTrail S3 data events), EKS, Runtime Monitoring, Malware, RDS, Lambda, AI |
| Output               | Findings, kept 90 days                                                                              |
| Prevents attacks?    | **No.** Detective only                                                                              |
| Response             | Findings to EventBridge, then SNS to notify or Lambda to remediate                                  |
| Multi-account        | Administrator and member accounts; delegated administrator through Organizations (recommended)      |
| Scope                | Regional; enable in all Regions                                                                     |
| Tuning               | Trusted and threat lists (IP addresses and domains), suppression rules                              |

> [!tip] Exam Tip
>
> | Requirement | Answer |
> | --- | --- |
> | Detect compromised instances, unusual API calls, or communication with known malicious IPs, with minimal setup | GuardDuty |
> | Automatically respond to a GuardDuty finding | EventBridge rule to Lambda (for example, add a NACL deny rule) |
> | Notify the security team about GuardDuty findings | EventBridge rule to SNS |
> | Manage GuardDuty across every account in an organization | Delegated administrator through AWS Organizations, with auto-enable |
> | Detect threats from S3 data access, such as mass deletions | GuardDuty S3 Protection |
> | Scan EC2 volumes for malware without installing an agent | GuardDuty Malware Protection for EC2 |
> | Find software vulnerabilities on EC2 | [Amazon Inspector](inspector.md), not GuardDuty |
> | Find personal data stored in S3 | [Amazon Macie](macie.md), not GuardDuty |

## References

- Adrian Cantrill, AWS Certified Solutions Architect – Associate (SAA-C03): "Amazon GuardDuty"
- [AWS: What is GuardDuty?](https://docs.aws.amazon.com/guardduty/latest/ug/what-is-guardduty.html)
- [AWS: foundational data sources](https://docs.aws.amazon.com/guardduty/latest/ug/guardduty_data-sources.html)
- [AWS: getting started with GuardDuty](https://docs.aws.amazon.com/guardduty/latest/ug/guardduty_settingup.html)
- [AWS: protection plans and Extended Threat Detection](https://aws.amazon.com/blogs/security/navigating-amazon-guardduty-protection-plans-and-extended-threat-detection/)
- [AWS: finding types](https://docs.aws.amazon.com/guardduty/latest/ug/guardduty_finding-types-active.html)
- [AWS: attack sequence finding types](https://docs.aws.amazon.com/guardduty/latest/ug/guardduty-attack-sequence-finding-types.html)
- [AWS: Extended Threat Detection for EC2 and ECS](https://aws.amazon.com/about-aws/whats-new/2025/12/guardduty-extended-threat-detection-ec2-ecs/)
- [AWS: processing findings with EventBridge](https://docs.aws.amazon.com/guardduty/latest/ug/guardduty_findings_eventbridge.html)
- [AWS: exporting findings to S3](https://docs.aws.amazon.com/guardduty/latest/ug/guardduty_exportfindings.html)
- [AWS: entity lists and IP address lists](https://docs.aws.amazon.com/guardduty/latest/ug/guardduty_upload-lists.html)
- [AWS: custom entity lists launch](https://aws.amazon.com/about-aws/whats-new/2025/09/threat-detection-amazon-guardduty-custom-entity-lists/)
- [AWS: managing GuardDuty accounts with Organizations](https://docs.aws.amazon.com/guardduty/latest/ug/guardduty_organizations.html)
- [AWS: GuardDuty quotas](https://docs.aws.amazon.com/guardduty/latest/ug/guardduty_limits.html)
- [AWS: suspending or disabling GuardDuty](https://docs.aws.amazon.com/guardduty/latest/ug/guardduty_suspend-disable.html)
- [AWS: GuardDuty pricing](https://docs.aws.amazon.com/guardduty/latest/ug/guardduty-pricing.html)
