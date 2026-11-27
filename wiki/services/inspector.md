---
title: Amazon Inspector
type: service
tags: [inspector, vulnerability-management, cve, ec2, ecr, lambda, ssm, ebs-snapshots, network-reachability, cis-benchmarks, findings, eventbridge, security-hub, organizations, delegated-administrator]
certs: [aws-saa]
updated: 2026-11-23
---

# Amazon Inspector

Amazon Inspector is a **vulnerability management** service. It finds your workloads, scans them, and keeps scanning them for two kinds of problem:

1. **Software vulnerabilities.** Installed packages and code with known security flaws, identified by their **CVE** (Common Vulnerabilities and Exposures) IDs.
2. **Unintended network exposure.** EC2 instances whose ports can be reached from outside the VPC when they probably should not be.

AWS says Inspector "discovers and scans Amazon EC2 instances, container images in Amazon ECR, and Lambda functions" ([AWS: What is Amazon Inspector?](https://docs.aws.amazon.com/inspector/latest/user/what-is-inspector.html)). Those three workload types are the ones the lecture covers. Newer, optional scan types also cover machine images and source code repositories (see [Scan Types](#scan-types)).

Like [GuardDuty](guardduty.md), [Macie](macie.md), and [AWS Config](config.md), Inspector is a **detective** service. It reports problems as **findings** and does not patch or reconfigure anything. To act on a finding, you send it to [EventBridge](eventbridge.md) and let another service respond, or you fix the problem yourself.

> [!note] Inspector Classic is retired
> The lecture contrasts the current Inspector with "the previous classic Inspector", which needed an agent, assessment templates, and scheduled assessment runs. Amazon Inspector Classic reached end of support on **May 20, 2026** ([AWS: services in full shutdown](https://docs.aws.amazon.com/general/latest/gr/full_shutdown_services.html)). Everything on this page describes the current service, whose API is `inspector2`.

## How Inspector Works

There is no schedule to set up for vulnerability scanning. Once you activate a scan type, Inspector finds the eligible resources and scans them **continuously**. It rescans a resource when something changes that could affect its risk ([AWS: scanning EC2 instances](https://docs.aws.amazon.com/inspector/latest/user/scanning-ec2.html), [AWS: scanning ECR images](https://docs.aws.amazon.com/inspector/latest/user/scanning-ecr.html)):

- A new instance is launched, a new image is pushed, or a new Lambda function is deployed.
- New software is installed or a patch is applied.
- A new CVE is published that affects software Inspector already knows about.

The last trigger is the important one. You do not need to change anything for a resource to become vulnerable. A package that was safe yesterday can have a CVE published today, and Inspector reports it without a new scan request.

Each finding gets an **Inspector score** and a severity, so you can fix the worst problems first. When you fix the problem, Inspector detects the fix on a later scan and **closes the finding automatically** ([AWS: understanding findings](https://docs.aws.amazon.com/inspector/latest/user/findings-understanding.html)).

You turn scan types on or off **per account and per Region**. AWS recommends repeating activation in each Region where you want coverage ([AWS: activating a scan type](https://docs.aws.amazon.com/inspector/latest/user/activate-scans.html)).

## Scan Types

The scan types you activate decide what Inspector checks ([AWS: scan types](https://docs.aws.amazon.com/inspector/latest/user/scanning-resources.html)):

| Scan type                     | What it checks                                                                                     | Finding types                                 | In the lecture? |
| ----------------------------- | -------------------------------------------------------------------------------------------------- | --------------------------------------------- | --------------- |
| **EC2 scanning**              | Operating system and application packages on instances, plus open network paths to them            | Package vulnerability, network reachability   | Yes             |
| **ECR scanning**              | OS and programming language packages in container images stored in [ECR](ecr.md)                   | Package vulnerability                         | Yes             |
| **Lambda standard scanning**  | Dependencies in [Lambda](lambda.md) functions and layers                                           | Package vulnerability                         | Yes             |
| **Lambda code scanning**      | Your own function code, for problems such as injection flaws, data leaks, and weak cryptography    | Code vulnerability                            | No              |
| **Machine image scanning**    | Amazon Machine Images (AMIs), optional                                                             | Package vulnerability                         | No              |
| **Code Security**             | Source code, third-party dependencies, and infrastructure as code in GitHub or GitLab repositories | Code vulnerability (not sent to Security Hub) | No              |
| **CIS scans** (EC2, targeted) | EC2 operating system settings against CIS Benchmarks                                               | Pass, fail, or skipped checks; not findings   | Yes             |

Lambda code scanning needs Lambda standard scanning turned on first ([AWS: Lambda scanning](https://docs.aws.amazon.com/inspector/latest/user/scanning-lambda.html)). Code Security was added in June 2025 and is activated separately ([AWS: Code Security](https://docs.aws.amazon.com/inspector/latest/user/code-security-assessments.html), [AWS: document history](https://docs.aws.amazon.com/inspector/latest/user/doc-history.html)). The lecture mentions code vulnerability findings without saying where they come from: they come from Lambda code scanning and Code Security.

## EC2 Scanning

EC2 scanning does two separate jobs: it looks for vulnerable software on each instance, and it checks which network paths can reach the instance.

### Agent-based and agentless scanning

Inspector has two ways to see the software on an EC2 instance ([AWS: scanning EC2 instances](https://docs.aws.amazon.com/inspector/latest/user/scanning-ec2.html)):

| Method          | How Inspector sees the software                                                                                                      | Requirements                                                                                                                                        | How often                                                                     |
| --------------- | ------------------------------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------- |
| **Agent-based** | The SSM Agent and an Inspector component collect a software inventory from inside the instance                                       | The instance must be a managed instance in [Systems Manager](ssm.md): SSM Agent running and an instance profile with `AmazonSSMManagedInstanceCore` | Continuous: on launch, on software changes (Linux and macOS), and on new CVEs |
| **Agentless**   | Inspector takes [EBS snapshots](ebs.md#snapshots--crossing-the-az-and-region-boundaries) of the instance's volumes and analyzes them | EBS-backed instance not managed by SSM; supported OS; ext3, ext4, or xfs file systems; fewer than 8 volumes totaling 1,200 GB or less               | Every 24 hours                                                                |

For agentless scans, Inspector creates the snapshots in your account, tags them with the key `InspectorScan`, and deletes them after the scan. Nothing is installed on the instance.

**Scan modes.** An account uses one of two EC2 scan modes:

- **Hybrid** (the default when you first activate Inspector): Inspector uses the agent-based method for instances that SSM manages, and the agentless method for eligible EBS-backed instances that SSM does not manage ([AWS: scan types](https://docs.aws.amazon.com/inspector/latest/user/scanning-resources.html)).
- **Agent-based**: Inspector scans only SSM-managed instances. Instances without the agent are not scanned.

There is no agentless-only mode. Standalone accounts and the delegated administrator can change the mode; an administrator's choice applies to all member accounts.

> [!warning] Gotcha: hybrid does not run both methods on one instance
> The lecture says hybrid mode uses both methods "together". It does, but across your fleet, not on each instance. Each instance gets one method: agent-based if SSM manages it, agentless if it does not and it meets the agentless requirements. To get continuous, change-driven scanning, make instances SSM-managed; otherwise they are scanned only once a day by snapshot.

**What the agent finds.** On Linux, agent-based scanning also does **deep inspection**: it finds programming language packages, such as Python or Java libraries, in default paths such as `/usr/lib` and `/usr/local/lib`, plus up to five custom paths. Deep inspection is on automatically for accounts that activated Inspector after April 17, 2023 ([AWS: deep inspection](https://docs.aws.amazon.com/inspector/latest/user/deep-inspection.html)). The component that collects inventory is delivered through SSM. Since May 2026 that can be the **Amazon Inspector VM Scanner** instead of the older Inspector SSM plugin, and since July 2026 AWS recommends the VM Scanner ("Enhanced EC2 Scanning") ([AWS: document history](https://docs.aws.amazon.com/inspector/latest/user/doc-history.html)).

**Excluding instances.** Tag an instance with the key `InspectorEc2Exclusion` and Inspector creates no findings for it and does not charge for it ([AWS: scanning EC2 instances](https://docs.aws.amazon.com/inspector/latest/user/scanning-ec2.html)).

### Network reachability

Network reachability scanning checks whether TCP and UDP ports on your instances can be reached from the **edges of the VPC** ([AWS: finding types](https://docs.aws.amazon.com/inspector/latest/user/findings-types.html)):

- The internet, through an [internet gateway](vpc.md#internet-gateway-igw)
- Another VPC, through a [VPC peering connection](vpc-peering.md)
- An on-premises network, through a [VPN connection](site-to-site-vpn.md) on a virtual private gateway

Key facts:

- **EC2 only.** Inspector generates network reachability findings only for EC2 instances.
- **No agent.** Inspector does not look inside the instance. It analyzes the network configuration around it, so no agent or snapshot is involved.
- **Every 12 hours.** The check runs automatically every 12 hours once Inspector is enabled. It is part of EC2 scanning, not a separate scan type.
- **End to end.** Inspector follows the whole path, not just one firewall. It takes into account EC2 instances, Application Load Balancers, Direct Connect, Elastic Load Balancers, network interfaces, internet gateways, network ACLs, route tables, security groups, subnets, VPCs, virtual private gateways, VPC endpoints, VPC peering connections, and VPN connections.

Because it reasons about configuration rather than sending test traffic, the analysis is like a static check of your [networking](../concepts/networking.md) setup: "is there a route, and do the [security groups](vpc.md#security-groups-sg) and [NACLs](vpc.md#network-acl-nacl) allow it?"

## ECR and Lambda Scanning

**ECR.** Activating Inspector's ECR scanning switches the account's private registry from ECR **basic scanning** to **enhanced scanning** ([AWS: activating a scan type](https://docs.aws.amazon.com/inspector/latest/user/activate-scans.html)). Enhanced scanning covers both OS packages and programming language packages. A repository can use ([AWS: scanning ECR images](https://docs.aws.amazon.com/inspector/latest/user/scanning-ecr.html)):

- **Continuous scanning**: scan on push, then rescan when a relevant CVE is published.
- **Scan on push**: scan only when the image is pushed.

Continuous rescanning is not forever. Inspector keeps monitoring an image while it was pushed or last used within the **re-scan duration**, which you can set from 3 to 180 days. For accounts activated on or after October 15, 2026, the default is 7 days based on when the image was last in use; older accounts have older defaults ([AWS: ECR re-scan duration](https://docs.aws.amazon.com/inspector/latest/user/scanning_resources_configure_duration_setting_ecr.html)). Inspector can also show which running ECS tasks and EKS pods use a vulnerable image. See [ECR image scanning](ecr.md#image-scanning) for how this compares with basic scanning.

**Lambda.** Standard scanning checks the dependencies in functions and layers. When you first activate it, Inspector scans functions invoked or updated in the last 90 days, and then rescans on new deployments, code or dependency updates, and new CVEs. Code scanning adds checks on your own code, such as injection flaws, data leaks, missing encryption, and weak cryptography. Functions encrypted with a customer managed KMS key are not supported ([AWS: Lambda scanning](https://docs.aws.amazon.com/inspector/latest/user/scanning-lambda.html)).

## Findings

### Finding types

Inspector produces three finding types ([AWS: finding types](https://docs.aws.amazon.com/inspector/latest/user/findings-types.html)):

| Finding type              | What it means                                                                         | Comes from                           |
| ------------------------- | ------------------------------------------------------------------------------------- | ------------------------------------ |
| **Package vulnerability** | An installed package has a known CVE. The finding lists the CVE ID                    | EC2, ECR, Lambda standard, AMI scans |
| **Code vulnerability**    | Lines of your own code could be exploited, for example by injection                   | Lambda code scanning, Code Security  |
| **Network reachability**  | A TCP or UDP port on an EC2 instance can be reached from a VPC edge, and by what path | EC2 scanning                         |

A **CVE** is an entry in a public catalog of known security vulnerabilities, each with its own ID such as `CVE-2024-3094`. Package vulnerability findings are a match between the packages Inspector found and that catalog. On Windows, a finding can be grouped by Microsoft KB update instead of by individual CVE.

A **network reachability** finding points at the configuration that is too open, such as a security group rule, a NACL, or an internet gateway route, so you can restrict access. Its severity depends on the port and path: for example, SSH (22) open to the internet is Medium, Telnet (23) or FTP (21) open to the internet is High, and HTTPS is Low ([AWS: severity levels](https://docs.aws.amazon.com/inspector/latest/user/findings-understanding-severity.html)).

> [!warning] Gotcha: Inspector identifies; it does not fix
> Inspector does not install patches, change security groups, or rebuild images. You fix the problem, or build automation that does, for example an EventBridge rule that starts an SSM Automation runbook or a Lambda function.

### Inspector score and severity

For package vulnerabilities, Inspector starts from the **CVSS** base score (v3.1 by default) and adjusts it for your environment, "such as network reachability data and exploitability data", to produce the **Inspector score** ([AWS: Inspector score](https://docs.aws.amazon.com/inspector/latest/user/findings-understanding-score.html)). The same CVE can therefore rank higher on an instance reachable from the internet than on an isolated one. Severity levels are Informational, Low, Medium, High, and Critical, plus **Untriaged** when the vendor has not scored a CVE yet ([AWS: severity levels](https://docs.aws.amazon.com/inspector/latest/user/findings-understanding-severity.html)).

### Status and retention

A finding is **Active** until you fix it, **Suppressed** if a suppression rule hides it, and **Closed** once Inspector detects the fix. If the problem comes back within seven days, Inspector reopens the finding ([AWS: understanding findings](https://docs.aws.amazon.com/inspector/latest/user/findings-understanding.html)).

Closed findings are deleted after a short time. The findings page says 3 days when the resource was deleted or is no longer eligible, and 30 days for other closures. The document history has a February 12, 2024 entry that changed closed-finding retention from 30 days to 7 days, so the two pages disagree (checked 2026-11-23). Either way, export findings to keep a long-term record.

**Suppression rules** hide findings that you accept, such as a CVE in a package you do not use. AWS says they "only hide findings from view. You cannot create a suppression rule that closes or remediates findings" ([AWS: suppression rules](https://docs.aws.amazon.com/inspector/latest/user/findings-managing-supression-rules.html)). A suppressed finding is kept, and Inspector still closes it when the problem is fixed.

## CIS Benchmark Scans

The **Center for Internet Security (CIS)** publishes Benchmarks: industry consensus checklists of secure configuration settings for operating systems. Each Benchmark has a **Level 1** profile (essential settings for any system) and a **Level 2** profile (stricter settings for high-security environments) ([AWS: CIS scans](https://docs.aws.amazon.com/inspector/latest/user/scanning-cis.html)).

Inspector can check EC2 instance operating systems against these Benchmarks. Unlike vulnerability scanning, a CIS scan is **targeted** and you choose when it runs ([AWS: creating a CIS scan configuration](https://docs.aws.amazon.com/inspector/latest/user/scanning-cis-create-cis-scan-configuration.html)):

- **When:** a one-time scan, or recurring daily, weekly, or monthly at a time you choose.
- **Which instances:** chosen by **instance tags**, and for a delegated administrator, by account.
- **Requirements:** EC2 scanning must be activated, the instance must be **SSM-managed** with the Inspector SSM plugin, and the instance profile needs `AmazonSSMManagedInstanceCore` and `AmazonInspector2ManagedCisPolicy`. Instances in private subnets need VPC endpoints for Inspector, S3, and SSM. AWS says CIS scans will stay on the SSM plugin and will not move to the VM Scanner.

> [!warning] Gotcha: CIS results are not findings
> The lecture's Findings slide lists CIS scans with the finding types. CIS scans produce their own **scan results**: for each instance, each Benchmark check **passed, failed, or was skipped**. You view them under CIS scans in the console or with `GetCisScanReport`, and can download them as PDF or CSV ([AWS: viewing CIS scan results](https://docs.aws.amazon.com/inspector/latest/user/scanning-cis-view-cis-scan-configuration.html)). They do not appear in the findings list.

CIS scans check **configuration hardening**, such as password policy or disabled services. Package vulnerability scanning checks for **known flaws in installed software**. An instance can pass one and fail the other.

## Publishing Findings: EventBridge and Security Hub CSPM

**EventBridge.** Inspector publishes findings to [EventBridge](eventbridge.md) automatically, with the event source `aws.inspector2`. It also publishes events for scan coverage changes and when the initial scan of a resource completes ([AWS: EventBridge integration](https://docs.aws.amazon.com/inspector/latest/user/eventbridge-integration.html)). A rule can then:

- Notify a team through [SNS](sns.md), for example for Critical findings.
- Start a [Lambda](lambda.md) function or an SSM Automation runbook that patches the instance, or tightens a security group after a network reachability finding.

This is how "notify or auto-remediate" in the lecture works. Inspector raises the finding, and everything after that is something you configure. It is the same pattern as [GuardDuty](guardduty.md#architecture) and [Macie](macie.md#publishing-findings-eventbridge-and-security-hub-cspm).

**Security Hub CSPM.** The lecture says Security Hub; Inspector's documentation now calls it **AWS Security Hub CSPM**. When you enable Security Hub CSPM, the integration turns on automatically and Inspector sends all its findings there in the AWS Security Finding Format (ASFF). Findings from **Code Security** are the exception: they are not sent ([AWS: Security Hub CSPM integration](https://docs.aws.amazon.com/inspector/latest/user/securityhub-integration.html)).

## Multi-Account Management

With [AWS Organizations](organizations.md), the management account designates one **delegated administrator** for Inspector. That account can activate or deactivate scan types for every member account, see all their findings, and set the EC2 scan mode for the organization. It can manage up to **10,000** member accounts ([AWS: managing multiple accounts](https://docs.aws.amazon.com/inspector/latest/user/managing-multiple-accounts.html)). Organization policies can also turn on scan types automatically in new accounts.

The delegated administrator is **Regional**: you designate it in every Region where you use Inspector, and it must be the same account in each one.

## Pricing

Inspector charges per resource scanned. The prices below are from the worked examples on the pricing page for US East (N. Virginia), checked on 2026-11-23; the page did not show a full rate table ([AWS: Inspector pricing](https://aws.amazon.com/inspector/pricing/)):

| What is scanned                | Example price                                       |
| ------------------------------ | --------------------------------------------------- |
| EC2 instance, agent-based      | $1.2528 per instance per month                      |
| EC2 instance, agentless        | $1.75 per instance per month                        |
| ECR container image            | $0.09 for the initial scan, then $0.01 per rescan   |
| Lambda function, standard      | $0.30 per function per month                        |
| Lambda function, code scanning | $0.60 per function per month, on top of standard    |
| CIS scan                       | $0.03 per instance per assessment                   |
| Code Security                  | $0.15 per scan type (SAST, SCA, IaC) per repository |

- New accounts get a **15-day free trial** covering EC2, ECR, Lambda, AMI, and code repository scanning. CIS assessments and on-demand AMI scans are not included.
- Excluded instances (tagged `InspectorEc2Exclusion`) are not charged.

## Inspector and Related Security Services

| Service                   | Job                                                                                     |
| ------------------------- | --------------------------------------------------------------------------------------- |
| **Inspector**             | Find software vulnerabilities and network exposure in EC2, container images, and Lambda |
| [GuardDuty](guardduty.md) | Detect threats and malicious activity happening now, from logs and runtime activity     |
| [Macie](macie.md)         | Find sensitive data in S3 objects and report bucket settings that reduce security       |
| [AWS Config](config.md)   | Record resource configuration and check it against rules                                |
| [SSM](ssm.md)             | Manage instances; Patch Manager installs the patches that Inspector findings call for   |
| AWS Security Hub CSPM     | Collect findings from Inspector, GuardDuty, Macie, Config, and others in one place      |

Inspector and GuardDuty are easy to confuse. Inspector asks "**what could be exploited?**": a vulnerable package or an open port, whether or not anyone is attacking. GuardDuty asks "**is something bad happening?**": suspicious activity in logs, such as an instance talking to a known malicious IP address. A typical setup uses both.

## Exam Recap

| Concept              | Key fact                                                                                              |
| -------------------- | ----------------------------------------------------------------------------------------------------- |
| What it is           | Continuous vulnerability management for EC2 (and its OS), ECR container images, and Lambda functions  |
| Looks for            | Software vulnerabilities (CVEs) and unintended network exposure                                       |
| Scheduling           | None for vulnerability scans; automatic rescans on new software, patches, and new CVEs                |
| EC2 methods          | Agent-based (SSM Agent) or agentless (EBS snapshots, every 24 hours); hybrid is the default           |
| Network reachability | EC2 only, no agent, every 12 hours; open TCP or UDP paths from IGW, VPC peering, or VPN               |
| Finding types        | Package vulnerability (CVE IDs), code vulnerability, network reachability                             |
| Prioritization       | Inspector score: CVSS adjusted for your environment                                                   |
| Fixes problems?      | **No.** Identifies only; findings close automatically once fixed                                      |
| Response             | Findings to EventBridge automatically, then SNS, Lambda, or SSM Automation; also to Security Hub CSPM |
| CIS Benchmarks       | Targeted EC2 OS configuration scans, one-time or scheduled; pass or fail results, not findings        |
| Multi-account        | Delegated administrator through Organizations (10,000 members); Regional                              |

> [!tip] Exam Tip
>
> | Requirement | Answer |
> | --- | --- |
> | Continuously scan EC2 instances, ECR images, or Lambda functions for CVEs | Amazon Inspector |
> | Find EC2 instances with ports reachable from the internet, without an agent | Inspector network reachability |
> | Scan EC2 for vulnerabilities without installing an agent | Inspector agentless scanning (EBS snapshots) |
> | Check EC2 operating systems against CIS Benchmarks on a schedule | Inspector CIS scans |
> | Scan container images for OS and language package vulnerabilities | ECR enhanced scanning, which is Inspector |
> | Automatically notify or remediate on a new Critical vulnerability | Inspector finding to EventBridge, then SNS or Lambda |
> | Detect an instance communicating with a known malicious IP address | GuardDuty, not Inspector |
> | Find PII in S3 | Macie, not Inspector |

## References

- Adrian Cantrill, AWS Certified Solutions Architect – Associate (SAA-C03): "Amazon Inspector"
- [AWS: What is Amazon Inspector?](https://docs.aws.amazon.com/inspector/latest/user/what-is-inspector.html)
- [AWS: scan types](https://docs.aws.amazon.com/inspector/latest/user/scanning-resources.html)
- [AWS: activating a scan type](https://docs.aws.amazon.com/inspector/latest/user/activate-scans.html)
- [AWS: scanning EC2 instances](https://docs.aws.amazon.com/inspector/latest/user/scanning-ec2.html)
- [AWS: deep inspection for Linux instances](https://docs.aws.amazon.com/inspector/latest/user/deep-inspection.html)
- [AWS: scanning ECR container images](https://docs.aws.amazon.com/inspector/latest/user/scanning-ecr.html)
- [AWS: ECR re-scan duration](https://docs.aws.amazon.com/inspector/latest/user/scanning_resources_configure_duration_setting_ecr.html)
- [AWS: scanning Lambda functions](https://docs.aws.amazon.com/inspector/latest/user/scanning-lambda.html)
- [AWS: Code Security](https://docs.aws.amazon.com/inspector/latest/user/code-security-assessments.html)
- [AWS: finding types](https://docs.aws.amazon.com/inspector/latest/user/findings-types.html)
- [AWS: understanding findings](https://docs.aws.amazon.com/inspector/latest/user/findings-understanding.html)
- [AWS: Inspector score](https://docs.aws.amazon.com/inspector/latest/user/findings-understanding-score.html)
- [AWS: severity levels](https://docs.aws.amazon.com/inspector/latest/user/findings-understanding-severity.html)
- [AWS: suppression rules](https://docs.aws.amazon.com/inspector/latest/user/findings-managing-supression-rules.html)
- [AWS: CIS scans](https://docs.aws.amazon.com/inspector/latest/user/scanning-cis.html)
- [AWS: creating a CIS scan configuration](https://docs.aws.amazon.com/inspector/latest/user/scanning-cis-create-cis-scan-configuration.html)
- [AWS: viewing CIS scan results](https://docs.aws.amazon.com/inspector/latest/user/scanning-cis-view-cis-scan-configuration.html)
- [AWS: EventBridge integration](https://docs.aws.amazon.com/inspector/latest/user/eventbridge-integration.html)
- [AWS: Security Hub CSPM integration](https://docs.aws.amazon.com/inspector/latest/user/securityhub-integration.html)
- [AWS: managing multiple accounts](https://docs.aws.amazon.com/inspector/latest/user/managing-multiple-accounts.html)
- [AWS: document history](https://docs.aws.amazon.com/inspector/latest/user/doc-history.html)
- [AWS: services in full shutdown (Inspector Classic)](https://docs.aws.amazon.com/general/latest/gr/full_shutdown_services.html)
- [AWS: Inspector pricing](https://aws.amazon.com/inspector/pricing/)
