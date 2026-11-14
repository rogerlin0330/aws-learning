---
title: AWS Shield
type: service
tags: [shield, shield-standard, shield-advanced, ddos, volumetric, syn-flood, http-flood, srt, proactive-engagement, cost-protection, protection-groups, health-based-detection, waf, firewall-manager, security, pricing]
certs: [aws-saa]
updated: 2026-11-13
---

# AWS Shield

AWS Shield protects applications running in AWS against **distributed denial of service (DDoS)** attacks. It comes in two tiers:

- **Shield Standard** is free and protects every AWS customer automatically.
- **Shield Advanced** is a paid subscription that adds protection for specific resources, Layer 7 protection through [AWS WAF](waf.md), cost protection, and access to the AWS Shield Response Team.

For the SAA exam, the usual question is which tier a scenario needs. Words like **DDoS**, **SYN flood**, and **UDP reflection** with no other requirement point to Shield Standard, which you already have. Words like **24/7 DDoS response team**, **protection against DDoS cost spikes**, or **advanced DDoS visibility** point to Shield Advanced.

## Types of DDoS Attack

The lecture groups DDoS attacks by the [OSI layer](../concepts/networking.md#osi-layers-of-aws-network-controls) they target:

| Type                   | Layer | How it works                                                                                                                      | Example                                        |
| ---------------------- | ----- | --------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------- |
| **Network volumetric** | 3     | Sends as much raw traffic as possible to saturate the target's network capacity                                                   | UDP reflection and amplification floods        |
| **Network protocol**   | 4     | Opens a huge number of connections, often from spoofed IPs, and never finishes them, so the target cannot accept real connections | TCP SYN flood                                  |
| **Application layer**  | 7     | Sends requests that are cheap to send but expensive to answer                                                                     | HTTP request floods against a slow search page |

A protocol attack does not need to use up CPU or memory. The lecture compares it to a call center where callers dial in and then stay silent: the operators are idle, but no new calls can get through. Protocol attacks can also have a volumetric part, but treat the two as different types.

An application-layer attack uses the application as intended. The lecture's example is a search URL such as `query.php?search=all_the_platypus_images_ever`. Each request costs the attacker almost nothing and can be sent thousands of times a second, while each one may take the server two or three seconds to answer. Because these requests look like normal HTTP, only a firewall that understands HTTP can filter them. See [Layer 7 Firewalls](../concepts/networking.md#layer-7-application-firewalls).

## Shield Standard

Shield Standard is **free** for all AWS customers and is **always on**. There is nothing to configure.

- **What it protects against:** the most common network-layer (L3) and transport-layer (L4) DDoS attacks, such as SYN floods and UDP reflection ([AWS: Shield Standard](https://docs.aws.amazon.com/waf/latest/developerguide/ddos-standard-summary.html)).
- **Where it works:** at the edge of the AWS network, for traffic through [CloudFront](cloudfront.md), [Route 53](route53.md), or [Global Accelerator](global-accelerator.md), and at the Regional perimeter, as traffic enters your [VPC](vpc.md).
- **Best protection:** Route 53 hosted zones, CloudFront distributions, and Global Accelerator standard accelerators get "comprehensive availability protection against all known network and transport layer attacks". This is why AWS recommends putting public applications behind these edge services.

> [!warning] Gotcha: Shield Standard does not stop HTTP floods
> Shield Standard covers Layers 3 and 4 only. A flood of valid-looking HTTP requests passes straight through it. For Layer 7 protection, add [AWS WAF](waf.md) with a rate-based rule or the Anti-DDoS managed rule group, with or without Shield Advanced.

## Shield Advanced

### Cost and Commitment

| Item         | Lecture                           | Current                                                                                                                                                                |
| ------------ | --------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Subscription | $3,000 per month per organization | $3,000 per month, billed once per **payer account** (the management account of an Organization with consolidated billing), covering the accounts and resources it owns |
| Commitment   | 1 year, so $36,000 a year         | 1 year, auto-renewing each year                                                                                                                                        |
| Usage        | A charge for data out             | **Data transfer out** fees from CloudFront, ELB, EC2, and Global Accelerator for protected resources, on top of the subscription                                       |

Source: [AWS Shield pricing](https://aws.amazon.com/shield/pricing/). In AWS's examples, the data transfer out fee is $0.025 per GB for CloudFront and $0.050 per GB for an ALB; check the pricing page for current rates.

Because the fee is per payer account rather than per account, an organization with many accounts pays it once. The lecture's advice still applies: put all accounts that need Shield Advanced in the same [Organization](organizations.md) so they share one subscription. Cancellation is only allowed in a window before the yearly renewal, and the benefits, including cost protection, depend on completing the one-year commitment.

### Protected Resources

Shield Advanced protects ([AWS: resources Shield Advanced protects](https://docs.aws.amazon.com/waf/latest/developerguide/ddos-advanced-summary-protected-resources.html)):

- CloudFront distributions
- Route 53 hosted zones
- Global Accelerator standard accelerators
- Application Load Balancers and Classic Load Balancers
- Elastic IP addresses, and through them **EC2 instances** and **Network Load Balancers**

> [!warning] Gotcha: NLBs and EC2 instances are protected through Elastic IPs
> The lecture lists NLBs alongside ALBs and CLBs. In fact, Shield Advanced protects an NLB or an EC2 instance only through an Elastic IP associated with it. Gateway Load Balancers and Global Accelerator custom routing accelerators are not supported, and protection is IPv4 only.

### Protection Is Not Automatic

Subscribing does not protect anything by itself. You must add each resource as a protected resource in Shield Advanced, or use an **AWS Firewall Manager Shield Advanced policy**, which applies protection to matching resources across the accounts of an Organization, including new ones as they are created.

> [!tip] Exam Tip
> Shield Standard is automatic. Shield Advanced protections must be **explicitly enabled** per resource, or applied at scale with Firewall Manager.

### Cost Protection

If a DDoS attack that Shield Advanced should have stopped makes your bill go up, AWS credits the extra charges. The lecture's example is an EC2 Auto Scaling group that scales out because the attack was not stopped.

Covered charges include ([AWS: requesting a credit](https://docs.aws.amazon.com/waf/latest/developerguide/ddos-request-service-credit.html)):

- Shield Advanced data transfer out
- CloudFront requests and data transfer out
- Route 53 queries
- Global Accelerator standard accelerator data transfer
- ALB load balancer capacity units
- EC2 instances launched by a scaling policy in response to the attack

Conditions:

- The resource must have been protected **before** the attack. Resources added during an attack are not eligible.
- For CloudFront and ALB, the resource needs a WAF web ACL with a rate-based rule in Block mode.
- You request the credit with a billing case in AWS Support, with "DDoS Concession" in the subject, **within 15 days** after the end of the billing month in which the attack happened. Credits are valid for 12 months.

### Shield Response Team and Proactive Engagement

The **Shield Response Team (SRT)** is a group of AWS DDoS specialists available 24/7. Access requires a **Business or Enterprise Support** plan ([AWS: SRT support](https://docs.aws.amazon.com/waf/latest/developerguide/ddos-srt-support.html)).

- **You contact the SRT** through a support case. The response time follows your support plan; the lecture mentions one hour or 15 minutes, which match the fastest response times of Business and Enterprise Support.
- **The SRT can act for you.** If you grant it an IAM role with the `AWSShieldDRTAccessPolicy` managed policy, it can analyze your WAF logs and, with your approval, change your web ACLs to block an attack. You can also let it read up to 10 S3 buckets of other logs, as long as they are not encrypted with KMS keys ([AWS: SRT access](https://docs.aws.amazon.com/waf/latest/developerguide/ddos-srt-access.html)).
- **Proactive engagement** reverses the direction: the SRT contacts **you** when the availability or performance of a protected application is affected by a possible attack. The SRT can start work before it reaches you. It requires a Business or Enterprise Support plan, at least one emergency contact (up to 10), and a **Route 53 health check** associated with each resource ([AWS: proactive engagement](https://docs.aws.amazon.com/waf/latest/developerguide/ddos-srt-proactive-engagement.html)).

### Layer 7 Protection with AWS WAF

Shield Advanced uses [AWS WAF](waf.md) to protect against Layer 7 attacks, and the subscription includes the basic WAF charges for protected resources: Web ACL, rule, and request fees for up to 1,500 WCU and up to 50 billion requests per payer account per month. Bot Control, Fraud Control, CAPTCHA, WCU above 1,500, and larger body inspection are still charged ([AWS Shield pricing](https://aws.amazon.com/shield/pricing/)).

For CloudFront and ALBs with a web ACL, Shield Advanced can respond to HTTP floods automatically:

- **Automatic application layer DDoS mitigation** learns a traffic baseline for each resource (24 hours to 30 days), then adds and manages WAF rules in a rule group in your web ACL when it detects an attack. The rule group uses 150 WCU ([AWS: automatic application layer mitigation](https://docs.aws.amazon.com/waf/latest/developerguide/ddos-automatic-app-layer-response.html)).
- Since **March 26, 2026**, the WAF **Anti-DDoS managed rule group** is the default for HTTP flood protection and replaces this older automatic mitigation for new setups. AWS says it detects and mitigates within seconds rather than minutes. Existing customers can keep the older option.

The Anti-DDoS managed rule group can also be used with WAF alone, without Shield Advanced, for its own fee (see [WAF pricing](waf.md#pricing)).

### Visibility

- **CloudWatch metrics** in the `AWS/DDoSProtection` namespace, such as `DDoSDetected` (non-zero during an event), `DDoSAttackBitsPerSecond`, `DDoSAttackPacketsPerSecond`, and `DDoSAttackRequestsPerSecond`. AWS suggests an alarm on `DDoSDetected` as a starting point ([AWS: Shield metrics](https://docs.aws.amazon.com/waf/latest/developerguide/shield-metrics.html)). See [CloudWatch](cloudwatch.md).
- **Event details** for each detected event in the Shield console and API, including attack vectors and mitigations.

The Shield console's **global threat dashboard** and account-level summary are available to Shield Standard customers too. What Advanced adds is per-resource detection, metrics, and event detail.

### Health-Based Detection

You can associate a **Route 53 health check** with a protected resource (any type except Route 53 hosted zones). Shield Advanced then uses the application's health in its detection ([AWS: health-based detection](https://docs.aws.amazon.com/waf/latest/developerguide/ddos-advanced-health-checks.html)):

- When the resource is **unhealthy**, Shield reacts to smaller traffic changes and reports events faster. For EIPs, NLBs, and Global Accelerator, it can also place mitigations sooner and at lower thresholds.
- When the resource is **healthy**, Shield needs a larger change before it alerts, which reduces false positives.

Health-based detection is **required for proactive engagement**, because the SRT uses the health check to decide whether an event is affecting your application. See [Route 53](route53.md) for how health checks work.

### Protection Groups

A **protection group** treats several protected resources as one unit for detection and mitigation ([AWS: protection groups](https://docs.aws.amazon.com/waf/latest/developerguide/ddos-protection-groups.html)). Membership can be all protected resources, all resources of a type, or a chosen list. New resources that match the criteria join automatically, which is the admin saving the lecture points out.

The group's traffic is combined in one of three ways:

| Aggregation | Uses                                 | Suits                                                                  |
| ----------- | ------------------------------------ | ---------------------------------------------------------------------- |
| **Sum**     | Total traffic across the group       | EIPs for a fleet of EC2 instances                                      |
| **Mean**    | Average traffic across the group     | Load balancers and accelerators that share traffic evenly              |
| **Max**     | Highest traffic of any single member | CloudFront distributions and their origins, which see the same traffic |

## Standard vs Advanced

|                                           | **Shield Standard**                                                            | **Shield Advanced**                                                           |
| ----------------------------------------- | ------------------------------------------------------------------------------ | ----------------------------------------------------------------------------- |
| Cost                                      | Free                                                                           | $3,000 per month per payer account, 1-year commitment, plus data transfer out |
| Enabled                                   | Automatically, for everyone                                                    | Explicitly, per resource or by Firewall Manager policy                        |
| Layers                                    | 3 and 4                                                                        | 3, 4, and 7 (Layer 7 through WAF)                                             |
| Resources                                 | All AWS customers' traffic; best with CloudFront, Route 53, Global Accelerator | CloudFront, Route 53, Global Accelerator, ALB, CLB, EIPs (EC2, NLB)           |
| WAF fees                                  | Normal WAF pricing                                                             | Basic WAF fees included for protected resources                               |
| Cost protection                           | No                                                                             | Credits for attack-driven scaling charges                                     |
| Shield Response Team                      | No                                                                             | 24/7, with Business or Enterprise Support                                     |
| Proactive engagement                      | No                                                                             | Yes, with Route 53 health checks                                              |
| Metrics and event detail                  | Global threat dashboard and account summary                                    | Per-resource CloudWatch metrics and event detail                              |
| Health-based detection, protection groups | No                                                                             | Yes                                                                           |

> [!tip] Exam Tip
>
> | Requirement | Answer |
> | --- | --- |
> | Protect against common DDoS (SYN floods, UDP reflection) at no cost | Shield Standard, already on |
> | Best DDoS resilience for a public website | CloudFront and Route 53 in front, Shield Standard automatically |
> | Stop an HTTP flood or a login flood | WAF rate-based rule or Anti-DDoS managed rule group |
> | 24/7 access to AWS DDoS experts during an attack | Shield Advanced with Business or Enterprise Support |
> | AWS should contact us when an attack affects our application | Shield Advanced proactive engagement with Route 53 health checks |
> | Refund for scaling costs caused by a DDoS attack | Shield Advanced cost protection |
> | Enable DDoS protection across all accounts in an Organization | Firewall Manager Shield Advanced policy |
> | Protect an EC2 instance or NLB with Shield Advanced | Protect its Elastic IP |

## Related Services

- [AWS WAF](waf.md) provides the Layer 7 rules that Shield Advanced relies on, and can be used on its own.
- **AWS Firewall Manager** applies Shield Advanced protection and WAF rules across the accounts of an [Organization](organizations.md).
- **CloudFront flat-rate plans** (November 2025) bundle CloudFront, WAF, and DDoS protection for one monthly price; see [WAF pricing](waf.md#pricing).
- **AWS Shield network security director** analyzed network resources and configurations against AWS best practices. Its public preview has ended, and AWS has not announced general availability ([AWS: network security director](https://docs.aws.amazon.com/waf/latest/developerguide/nsd-chapter.html)). It is not part of Shield Advanced.

## References

- Adrian Cantrill, AWS Certified Solutions Architect – Associate (SAA-C03): "AWS Shield"
- [AWS Shield pricing](https://aws.amazon.com/shield/pricing/)
- [AWS: Shield Standard](https://docs.aws.amazon.com/waf/latest/developerguide/ddos-standard-summary.html)
- [AWS: Shield Advanced](https://docs.aws.amazon.com/waf/latest/developerguide/ddos-advanced-summary.html)
- [AWS: resources Shield Advanced protects](https://docs.aws.amazon.com/waf/latest/developerguide/ddos-advanced-summary-protected-resources.html)
- [AWS: requesting a credit (cost protection)](https://docs.aws.amazon.com/waf/latest/developerguide/ddos-request-service-credit.html)
- [AWS: Shield Response Team support](https://docs.aws.amazon.com/waf/latest/developerguide/ddos-srt-support.html)
- [AWS: SRT access](https://docs.aws.amazon.com/waf/latest/developerguide/ddos-srt-access.html)
- [AWS: proactive engagement](https://docs.aws.amazon.com/waf/latest/developerguide/ddos-srt-proactive-engagement.html)
- [AWS: automatic application layer DDoS mitigation](https://docs.aws.amazon.com/waf/latest/developerguide/ddos-automatic-app-layer-response.html)
- [AWS: health-based detection](https://docs.aws.amazon.com/waf/latest/developerguide/ddos-advanced-health-checks.html)
- [AWS: protection groups](https://docs.aws.amazon.com/waf/latest/developerguide/ddos-protection-groups.html)
- [AWS: Shield metrics](https://docs.aws.amazon.com/waf/latest/developerguide/shield-metrics.html)
- [AWS: network security director (preview)](https://docs.aws.amazon.com/waf/latest/developerguide/nsd-chapter.html)
