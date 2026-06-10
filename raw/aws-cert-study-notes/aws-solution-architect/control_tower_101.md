# AWS Control Tower 101

> **Exam relevance:** Control Tower is becoming required real-world knowledge and is appearing more frequently across all AWS certification paths. For the SAA, a **high-level conceptual understanding** is sufficient — you don't need hands-on depth, but you must know what each component does and which underlying services it orchestrates.

---

## 1. What Control Tower Is

Control Tower's core job is the **quick and easy setup of multi-account environments** that are well-architected by default.

The key mental model: **Control Tower does not reinvent the wheel — it orchestrates other AWS services.** It is best thought of as **"AWS Organizations with superpowers"** — an evolution of Organizations adding more features, intelligence, and automation.

Services it orchestrates under the covers:

|Service|Role in Control Tower|
|---|---|
|**AWS Organizations**|Multi-account structure — OUs and Service Control Policies (SCPs)|
|**IAM Identity Center** (formerly **AWS SSO**)|Single sign-on and ID federation across all accounts|
|**CloudFormation**|Implements the automation (creates stacks in your environment)|
|**AWS Config**|Detective guardrails / compliance checks (config rules)|
|**CloudWatch, CloudTrail, SNS**|Centralized logging, monitoring, auditing, notifications|

> 🎯 **Trigger keyword:** "quickly set up a secure, well-architected **multi-account** environment with governance baked in" → **Control Tower**, not raw Organizations.

> ⚠️ **Trap:** IAM Identity Center is the **product formerly known as AWS SSO**. The exam may use either name — treat them as the same thing.

---

## 2. The Four Parts You Must Distinguish

These four terms are the backbone of the topic. Get the distinction clear now.

1. **Landing Zone** — the multi-account environment itself (what most people _mean_ when they say "Control Tower").
2. **Guardrails** — rules that detect or mandate standards across accounts.
3. **Account Factory** — automation that creates/modifies/deletes accounts in a standardized way.
4. **Dashboard** — single-page oversight of the entire organization.

Everything else **surrounds the landing zone**.

---

## 3. Reading the Architecture Diagram

![[control-tower.png]]

Work through this diagram in three passes: **context bar (top) → main objects (center) → footer accounts (bottom).**

### Context bar (top)

The whole canvas is the **landing zone**. Top-left is the **Management Account** — the account you create Control Tower _from_, exactly like the management account in AWS Organizations. The top-right stacked boxes are the **Account Factory Provisioned Accounts** — the fleet of accounts the factory churns out.

### Left edge — identities feeding in

Two orange arrows enter from the left: **Federated ID** (your existing corporate identity store) and **Internal ID** (identities living in Identity Center). Both arrows point at the **SSO** icon, illustrating that _all_ identity types funnel through IAM Identity Center for single sign-on.

### Center — the Management Account internals

- **Control Tower** (pink diamond icon) sits at the top and orchestrates everything. Two pink arrows fan out from it labeled **CloudFormation** and **Config** — these are the robots (literally drawn as robot icons) it drives to do the automation work.
- **AWS Organizations** (bottom pink icon) provides the structure. Pink arrows run from it up to the two OUs:
    - **Custom OU** (default name **Sandbox**)
    - **Foundational OU** (default name **Security**)
- The thick **orange arrows** show automation/enforcement flowing _outward_ from the Management Account into the provisioned accounts and across the guardrails.

### The Guardrails strip

The yellow-and-black hazard barrier running horizontally across the bottom of the provisioned-accounts region **is** the guardrails. The label **SCP** sits next to the arrow hitting it — a visual reminder that **preventative guardrails are implemented via Service Control Policies**.

### Provisioned accounts region (top right)

Inside the stacked boxes: **Account Baseline (template)** and **Network Baseline (template)** — the "cookie-cutter" configs every new account inherits (note the cookie-monster joke: _"Did someone say cookie cutter?"_). These ensure consistent account + networking setup, including non-overlapping VPC IP addressing.

### Footer — the two foundational accounts

Both live inside the **Foundational/Security OU** (dotted lines connect them back to it):

- **Audit Account** — contains **SNS** + **CloudWatch** icons (notifications + landing-zone-wide monitoring).
- **Log Archive Account** — contains **AWS Config** + **CloudTrail** icons (the isolated, read-only log store).

---

## 4. Landing Zone — Detail

The landing zone is the feature that lets _anyone_ implement a well-architected multi-account environment.

- **Home region:** the region you initially deploy Control Tower into (e.g. `us-east-1` — _illustrative example_, not a value to memorize). You can explicitly **allow or deny** the use of other regions, but the **home region is always available** and cannot be denied.
- **Built from:** AWS Organizations + AWS Config + CloudFormation + more (orchestration, not reinvention).
- **Default OU structure:**
    - **Foundational OU** (default: **Security**) → holds the **Audit** and **Log Archive** accounts.
    - **Custom OU** (default: **Sandbox**) → testing / less rigid security.
- You can create additional OUs and accounts, including **nested OUs**, for real-world structures.
- **Identity:** IAM Identity Center provides SSO across accounts and supports **ID federation** (use existing identity stores).
- **Monitoring/notifications:** CloudWatch + SNS.
- **Self-service provisioning:** end users can provision new accounts via **Service Catalog**.

### The two foundational accounts

|Account|Purpose|Key services|Access model|
|---|---|---|---|
|**Log Archive**|Central store of logging info for **all enrolled accounts**; isolated for integrity|AWS Config + CloudTrail logs|Secure, **read-only** archive; access must be **explicitly granted**|
|**Audit**|Access to audit info; landing point for **third-party auditing tools**|SNS (governance/security change notifications), CloudWatch (landing-zone-wide metrics)|Granted to users needing audit/governance visibility|

> 🎯 **Trigger keyword:** _"isolated, read-only store of CloudTrail/Config logs from every account"_ → **Log Archive account**. _"notifications of governance/security policy changes" or "home for third-party audit tooling"_ → **Audit account**.

---

## 5. Guardrails — Detail

**Guardrails = rules for multi-account governance.** They are classified along **two independent axes**: _enforcement level_ and _functional type_. Don't conflate the two.

### Axis 1 — Enforcement level (how strongly AWS pushes it)

|Level|Behavior|
|---|---|
|**Mandatory**|Always applied — you can't turn them off|
|**Strongly recommended**|AWS strongly advises enabling|
|**Elective**|Optional; used for niche / specific requirements|

### Axis 2 — Functional type (how it works)

|Type|Mechanism|What it does|Possible states|
|---|---|---|---|
|**Preventative**|**Service Control Policies (SCPs)**|**Stops** disallowed actions before they happen|**Enforced** or **Not enabled**|
|**Detective**|**AWS Config rules**|**Identifies / flags** non-compliant config (compliance check)|**Clear**, **In violation**, or **Not enabled**|

**Examples**

- _Preventative:_ deny use of certain AWS regions; disallow S3 bucket policy changes inside landing-zone accounts.
- _Detective:_ check whether CloudTrail is enabled in an account; check whether any EC2 instances have public IPv4 addresses.

> ⚠️ **Critical exam distinction:** **Preventative guardrails STOP things from occurring (SCP). Detective guardrails only IDENTIFY things after the fact (Config).** Detective ≠ blocking — it reports, it does not prevent.

> 🧠 **Mnemonic:** **P**reventative = **P**olicy (SCP) and _Prohibits_. **D**etective = config **D**rift _Discovery_.

---

## 6. Account Factory — Detail

Automated account provisioning — a "team of robots" that creates, modifies, or deletes AWS accounts on demand.

- **Who can provision:** cloud administrators **or end users** with appropriate permissions (true self-service).
- **Guardrails auto-applied:** any defined guardrails are automatically attached to newly provisioned accounts.
- **Admin grant:** the provisioning user (or anyone you nominate) can be given **admin rights** over the new account — within tightly controlled parameters you define.
- **Standardized config (cookie-cutter):** every account ships with baseline **account + network** configuration, e.g. **VPC IP addressing chosen to avoid overlap** — essential at scale.
- **Lifecycle:** accounts can be long-running or short-term, and can later be **closed or repurposed**.
- **Interfaces:** Control Tower console, **Service Catalog**, or **APIs**.
- **SDLC integration:** can be wired into your software development lifecycle — e.g. spin up accounts for a build stage, client demos, or software testing.

> 🎯 **Trigger keyword:** _"self-service, standardized, automated AWS account creation with guardrails pre-applied"_ → **Account Factory**.

---

## 7. Source / Terminology Note

- The transcript and infographic use **"SSO"** throughout; the current AWS name is **IAM Identity Center**. They are the same service — expect either label on the exam.
- "Foundational OU" and "Security OU" refer to the same OU (Security is its default name); likewise "Custom OU" ↔ "Sandbox". The notes preserve both so a renamed OU in a question doesn't trip you up.

---

## 8. Exam Recap — Rapid Fire

- **Control Tower = orchestration**, not a new engine. It drives **Organizations, IAM Identity Center, CloudFormation, Config, CloudWatch, CloudTrail, SNS**.
- **Four parts:** Landing Zone, Guardrails, Account Factory, Dashboard.
- **Landing Zone** = the multi-account environment; has a **home region** (always available; other regions can be allowed/denied).
- **Default OUs:** Foundational/**Security** (holds Audit + Log Archive accounts) and Custom/**Sandbox**.
- **Log Archive** = isolated, **read-only**, explicitly-granted store of Config + CloudTrail logs for all enrolled accounts.
- **Audit** = governance/security change notifications (SNS) + landing-zone-wide monitoring (CloudWatch) + third-party audit tooling.
- **Guardrails — two axes:** enforcement level (**mandatory / strongly recommended / elective**) and functional type (**preventative = SCP, stops; detective = Config rules, flags only**).
- **Preventative states:** enforced / not enabled. **Detective states:** clear / in violation / not enabled.
- **Account Factory** = self-service automated account provisioning, guardrails auto-applied, cookie-cutter account + network baselines (non-overlapping VPC IPs), admin rights grantable, integrates with SDLC via console / Service Catalog / API.
- Identities (**Federated + Internal**) both flow through **IAM Identity Center** for SSO + ID federation.