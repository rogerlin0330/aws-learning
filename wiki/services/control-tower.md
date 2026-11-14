---
title: AWS Control Tower
type: service
tags: [control-tower, multi-account, governance, organizations, landing-zone, guardrails, account-factory]
certs: [aws-saa]
updated: 2026-11-13
---

# AWS Control Tower

AWS Control Tower's core job is the **quick and easy setup of a multi-account environment that is well-architected by default**. The single most important mental model: **Control Tower does not reinvent the wheel — it orchestrates other AWS services.** It is best thought of as **"[AWS Organizations](organizations.md) with superpowers"** — an evolution of Organizations that layers automation, sensible defaults, and ongoing governance on top.

For the SAA exam a **high-level conceptual understanding** is enough: know what each component does and which underlying service it drives. You do not need hands-on depth.

> [!tip] Exam Tip
> Trigger phrase: "quickly set up a **secure, well-architected, multi-account** environment with governance baked in" → **Control Tower**, not raw [Organizations](organizations.md). Organizations gives you the account structure and SCPs; Control Tower automates standing up the *whole* well-architected environment around them.

## What Control Tower Orchestrates

Control Tower is a thin orchestration layer over services covered elsewhere in this wiki:

| Service                                                                                                  | Role inside Control Tower                                                                                               |
| -------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| **[AWS Organizations](organizations.md)**                                                                | The multi-account structure — OUs and [Service Control Policies (SCPs)](organizations.md#service-control-policies-scps) |
| **[IAM Identity Center](../concepts/identity-and-access.md#identity-federation)** (formerly **AWS SSO**) | Single sign-on and identity federation across every account                                                             |
| **CloudFormation**                                                                                       | Implements the automation — creates the stacks that build your environment                                              |
| **[AWS Config](config.md)**                                                                              | Detective guardrails / compliance checks (Config rules)                                                                 |
| **[CloudWatch](cloudwatch.md), [CloudTrail](cloudtrail.md), SNS**                                        | Centralized monitoring, auditing, and notifications                                                                     |
| **Service Catalog**                                                                                      | Front-end for self-service account provisioning (Account Factory)                                                       |

> [!warning] Gotcha
> **IAM Identity Center is the product formerly known as AWS SSO.** The exam may use either name — treat them as the same service. Likewise, AWS has rebranded **"guardrails" as "controls"** in the console; older material (and many exam questions) still says guardrails. Same thing.

## The Four Parts You Must Distinguish

These four terms are the backbone of the topic:

1. **Landing Zone** — the multi-account environment itself. This is what most people *mean* when they say "Control Tower." Everything else surrounds it.
2. **Guardrails (Controls)** — rules that detect or mandate standards across accounts.
3. **Account Factory** — automation that creates, modifies, and deletes accounts in a standardized way.
4. **Dashboard** — single-page oversight of the entire organization.

## Reading the Architecture Diagram

![AWS Control Tower architecture: the whole canvas is the Landing Zone; the Management Account (top-left) runs Control Tower, which drives CloudFormation and Config robots; Account Factory provisions baselined accounts (top-right) behind an SCP guardrail barrier; Federated and Internal identities funnel through IAM Identity Center SSO; the Security/Foundational OU holds the Audit account (SNS + CloudWatch) and Log Archive account (Config + CloudTrail)](../assets/control-tower.png)

Read the diagram in three passes — context bar (top), main objects (center), footer accounts (bottom):

- **The whole canvas is the Landing Zone.** Top-left is the **Management Account** — the account you create Control Tower *from*, exactly like the management account in [Organizations](organizations.md#core-architecture). Top-right are the **Account Factory provisioned accounts**, the fleet the factory churns out.
- **Identities feed in from the left.** Two arrows — **Federated ID** (your existing corporate identity store) and **Internal ID** (identities living in Identity Center) — both point at the **SSO** icon. All identity types funnel through IAM Identity Center for single sign-on.
- **Center — Management Account internals.** Control Tower sits at the top and orchestrates everything, fanning out **CloudFormation** and **Config** (drawn as robots) to do the automation work. **AWS Organizations** provides the structure underneath, with arrows up to the two default OUs: **Custom OU** (default name *Sandbox*) and **Foundational OU** (default name *Security*).
- **The guardrails barrier.** The yellow-and-black hazard strip across the provisioned-accounts region **is** the guardrails. The **SCP** label next to it is a reminder that **preventative guardrails are implemented via Service Control Policies**.
- **Provisioned accounts (top-right)** inherit an **Account Baseline** and a **Network Baseline** — the cookie-cutter configs every new account gets, including non-overlapping VPC IP addressing.
- **Footer — the two foundational accounts**, both living inside the Foundational/Security OU: the **Audit Account** (SNS + CloudWatch) and the **Log Archive Account** (Config + CloudTrail).

## Landing Zone

The landing zone is the feature that lets *anyone* stand up a well-architected multi-account environment. It is **built from** Organizations, AWS Config, CloudFormation, IAM Identity Center, CloudWatch, and SNS — orchestration, not reinvention.

- **Home region:** the region you initially deploy Control Tower into. You can explicitly **allow or deny** other regions, but the **home region is always available and cannot be denied**.
- **Default OU structure:**
  - **Foundational OU** (default name *Security*) — holds the **Audit** and **Log Archive** accounts.
  - **Custom OU** (default name *Sandbox*) — testing and less-rigid workloads.
- You can add more OUs and accounts, including **nested OUs**, to model real-world structures.
- **Identity:** IAM Identity Center provides SSO and supports **federation** with existing identity stores.
- **Self-service provisioning:** end users can provision new accounts through **Service Catalog**.

### The Two Foundational Accounts

| Account         | Purpose                                                                        | Key services                                                                                            | Access model                                                         |
| --------------- | ------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------- |
| **Log Archive** | Central, isolated store of logging data for **all enrolled accounts**          | AWS Config + [CloudTrail](cloudtrail.md) logs                                                           | Secure, **read-only** archive; access must be **explicitly granted** |
| **Audit**       | Visibility into audit/governance data; home for **third-party auditing tools** | SNS (governance/security change notifications), [CloudWatch](cloudwatch.md) (landing-zone-wide metrics) | Granted to users needing audit/governance visibility                 |

> [!tip] Exam Tip
> "Isolated, **read-only** store of CloudTrail/Config logs from every account" → **Log Archive account**. "Notifications of governance/security policy changes" or "home for third-party audit tooling" → **Audit account**.

## Guardrails (Controls)

**Guardrails are rules for multi-account governance.** They are classified along **two independent axes** — don't conflate them.

### Axis 1 — Enforcement level

| Level                    | Behavior                                     |
| ------------------------ | -------------------------------------------- |
| **Mandatory**            | Always applied; you cannot turn them off     |
| **Strongly recommended** | AWS strongly advises enabling                |
| **Elective**             | Optional; for niche or specific requirements |

### Axis 2 — Functional type

| Type             | Mechanism                                                                         | What it does                                               | States                             |
| ---------------- | --------------------------------------------------------------------------------- | ---------------------------------------------------------- | ---------------------------------- |
| **Preventative** | [Service Control Policies (SCPs)](organizations.md#service-control-policies-scps) | **Stops** disallowed actions before they happen            | Enforced / Not enabled             |
| **Detective**    | [AWS Config rules](config.md#config-rules)                                        | **Identifies / flags** non-compliant config after the fact | Clear / In violation / Not enabled |

Examples — *preventative:* deny use of certain regions; disallow S3 bucket-policy changes in landing-zone accounts. *Detective:* check whether CloudTrail is enabled; check whether any EC2 instance has a public IPv4 address.

> [!warning] Gotcha
> **Preventative guardrails STOP things (SCP). Detective guardrails only IDENTIFY things after the fact (Config).** Detective is *not* blocking — it reports, it does not prevent. Mnemonic: **P**reventative = **P**olicy (SCP) and *Prohibits*; **D**etective = config **D**rift *Discovery*.

> [!tip] Exam Tip
> A modern addition you may see: **proactive controls**, implemented via **CloudFormation hooks**, block non-compliant resources *at deploy time* — before they are even created. For SAA the core distinction remains **preventative (SCP) vs detective (Config)**; treat proactive as a third, deploy-time variant if it appears.

## Account Factory

Automated account provisioning — a "team of robots" that creates, modifies, or deletes AWS accounts on demand.

- **Who can provision:** cloud administrators **or end users** with the right permissions — true self-service.
- **Guardrails auto-applied:** any defined guardrails are automatically attached to newly provisioned accounts.
- **Admin grant:** the provisioning user (or anyone you nominate) can be granted **admin rights** over the new account, within parameters you define.
- **Standardized config (cookie-cutter):** every account ships with baseline **account + network** configuration — for example, **VPC IP addressing chosen to avoid overlap**, essential at scale.
- **Lifecycle:** accounts can be long-running or short-lived, and can later be **closed or repurposed**.
- **Interfaces:** Control Tower console, **Service Catalog**, or **APIs** — so it can wire into your SDLC (spin up accounts for a build stage, client demos, or testing).

> [!tip] Exam Tip
> Trigger phrase: "self-service, standardized, automated AWS account creation with guardrails pre-applied" → **Account Factory**.

## Control Tower vs Organizations

|                  | [AWS Organizations](organizations.md)                  | AWS Control Tower                                                                                   |
| ---------------- | ------------------------------------------------------ | --------------------------------------------------------------------------------------------------- |
| **What it is**   | The account-grouping and policy *primitive*            | An *orchestration layer* that builds a governed environment on top of Organizations                 |
| **You get**      | Root, OUs, member accounts, consolidated billing, SCPs | A pre-built landing zone: default OUs, foundational accounts, SSO, baselines, guardrails, dashboard |
| **Effort**       | You assemble governance yourself                       | One-click, well-architected by default                                                              |
| **Relationship** | The foundation                                         | Uses Organizations under the covers (and turns SCPs into preventative guardrails)                   |

## Resilience and Scope

Control Tower is governed from the **management account** and operates across the regions you enable; the **home region is always available**. Like Organizations, its account structure and guardrails apply org-wide. See [Resource Scope](../concepts/resource-scope.md) for the global/regional picture.

## Exam Recap — Rapid Fire

- **Control Tower = orchestration**, not a new engine. It drives Organizations, IAM Identity Center, CloudFormation, Config, CloudWatch, CloudTrail, SNS, and Service Catalog.
- **Four parts:** Landing Zone, Guardrails, Account Factory, Dashboard.
- **Landing Zone** = the multi-account environment; has a **home region** (always available; other regions can be allowed/denied).
- **Default OUs:** Foundational/**Security** (holds Audit + Log Archive) and Custom/**Sandbox**.
- **Log Archive** = isolated, **read-only**, explicitly-granted store of Config + CloudTrail logs for all enrolled accounts. **Audit** = governance/security change notifications (SNS) + landing-zone-wide monitoring (CloudWatch) + third-party audit tooling.
- **Guardrails, two axes:** enforcement (**mandatory / strongly recommended / elective**) and type (**preventative = SCP, stops; detective = Config rules, flags only**).
- **Account Factory** = self-service automated account provisioning, guardrails auto-applied, cookie-cutter account + network baselines (non-overlapping VPC IPs), via console / Service Catalog / API.
- Identities (**Federated + Internal**) both flow through **IAM Identity Center** for SSO and federation.

## References

- [AWS Organizations](organizations.md) — the underlying account/SCP primitive
- [IAM](iam.md) and [Identity and Access](../concepts/identity-and-access.md) — IAM Identity Center, federation
- [CloudTrail](cloudtrail.md) and [CloudWatch](cloudwatch.md) — logging and monitoring in the foundational accounts
- [Resource Scope](../concepts/resource-scope.md) — global vs regional governance
- [AWS Control Tower Documentation](https://docs.aws.amazon.com/controltower/)
- Source: `raw/aws-cert-study-notes/aws-solution-architect/control_tower_101.md`
