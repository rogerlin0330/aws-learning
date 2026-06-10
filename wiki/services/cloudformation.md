---
title: CloudFormation
type: service
tags: [cloudformation, iac, infrastructure-as-code, template, stack, cfn-init, configsets, cfn-signal, cfn-hup, creation-policy, desired-state, bootstrapping, ec2, user-data]
certs: [aws-saa]
updated: 2026-06-10
---

# CloudFormation

**CloudFormation (CFN)** is AWS's native **infrastructure-as-code** service: you describe the resources you want in a **template** (YAML or JSON), hand it to CloudFormation, and it creates, updates, and deletes those resources for you as a single managed unit called a **stack**.

> [!note] Scope of this page
> This page currently focuses on **`cfn-init` and the EC2 bootstrapping features** of CloudFormation, because that is where the wiki's source material sits. `cfn-init`, creation policies, and signals are **slightly beyond the SAA exam's technical depth** — you will not be asked to write the YAML, but the *architecture* matters for automation-flavoured questions. Understand the concepts; don't memorize the template. The full CloudFormation surface (parameters, mappings, outputs, intrinsic functions, change sets, nested/stack sets, drift detection) will be filled in as more sources are ingested.

## Core Vocabulary

| Term | What it is |
| --- | --- |
| **Template** | A YAML/JSON document declaring the resources you want (the desired infrastructure). Immutable artifact you author and version. |
| **Stack** | A living instance of a template — the actual set of resources CloudFormation created from it. Updating the template and re-deploying updates the stack. |
| **Logical resource** | A named entry in the template (e.g. `EC2Instance`). |
| **Physical resource** | The real AWS resource CloudFormation provisions for that logical resource (e.g. `i-0abc123…`). |

The basic flow: a **template** is used to create a **stack**, and the stack creates the real resources (for our purposes, an **[EC2](ec2.md) instance**). CloudFormation also **substitutes** template placeholders such as `${AWS::StackId}`, `${AWS::StackName}`, and `${AWS::Region}` with real values before anything reaches the instance — this is how in-instance helper scripts know which stack and region to talk back to.

## cfn-init — Desired-State Bootstrapping

`cfn-init` is a **helper script** pre-installed on AWS-provided EC2 OSes (Amazon Linux and others). In practice it behaves like a **lightweight configuration-management system** — considerably more powerful than raw [user data](ec2.md#bootstrapping). It is the CloudFormation-native way to pass **complex bootstrapping instructions** into an instance, with the configuration defined inside the template itself.

### The central distinction — procedural vs. desired state

| Aspect | User Data | cfn-init |
| --- | --- | --- |
| **Model** | **Procedural** only — a script the OS runs line by line | **Procedural** *or* **desired state** |
| **Meaning** | Does exactly what each line says, in order | You declare *what you want to be true*; cfn-init does whatever is needed to get there |

**Desired-state example.** You tell cfn-init you want Apache version X:

- already installed at version X, then nothing happens;
- not installed, then it installs it;
- an older version is present, then it updates to X.

> [!tip] One-line mental model
> **User data does; cfn-init ensures a state.** User data is "run this script once and hope it works." cfn-init is "here is the state I want — achieve it, re-apply it on change, and (with a creation policy) report back whether it worked."

### What cfn-init can manage

- **Packages** — install/ensure a version (version-aware, unlike a hand-written user-data script).
- **Groups** and **users** — OS-level identities.
- **Sources** — download and extract archives locally (can authenticate).
- **Files** — create with specified content, permissions, and ownership.
- **Commands** — run arbitrary commands and **test** that a condition holds afterward.
- **Services** — start them and enable them on boot.

### How it works

The directives live in a special metadata block on a logical resource: **`AWS::CloudFormation::Init`** (under the resource's `Metadata`). The metadata defines one or more **configSets**, each an ordered list of **config keys** that run in sequence. `cfn-init` itself is **invoked from the instance's [user data](ec2.md#bootstrapping)** — so the "user data runs once at first launch" rule still governs the single line that *kicks off* cfn-init.

The runtime sequence:

1. The template creates a **stack**, which creates the **EC2 instance**.
2. The instance's **user data** runs `cfn-init` on first launch.
3. `cfn-init` calls the **CloudFormation service** (using the substituted stack ID and region) and retrieves the desired-state config from the resource's `Metadata`.
4. `cfn-init` implements that desired state on the instance.

In the WordPress demo template, the configSet `wordpress_install` runs five config keys in order: `install_cfn`, `software_install`, `configure_instance`, `install_wordpress`, `configure_wordpress`.

![CloudFormation cfn-init architecture. Top row shows the high-level flow Template, Stack, Instance connected by solid pink arrows — the template creates the stack, which creates the EC2 instance. On the left, the template's contents: the EC2Instance logical resource carries a CreationPolicy plus Metadata containing AWS::CloudFormation::Init with the configSets and five config keys (install_cfn, software_install, configure_instance, install_wordpress, configure_wordpress) — the desired-state definition. At the bottom, the UserData block wraps a short script in Fn::Base64 and !Sub that runs cfn-init (highlighted) then cfn-signal — the only procedural part, kicking everything off at launch. Numbered dotted arrows trace the runtime: arrow 1 from the launched instance running its cfn-init line, arrow 2 from cfn-init consulting the stack using the substituted StackId and Region to pull the desired-state config, arrow 3 returning the config to be applied. Footer reads "WORKS WITH STACK UPDATES"](../assets/cfn-init.png)

Reading the diagram: the top row is the familiar template-to-stack-to-instance flow. On the left, the template holds the `AWS::CloudFormation::Init` metadata (the desired state) plus a `CreationPolicy`. At the bottom, the `UserData` block is the *only* procedural part — a short `Fn::Base64`-wrapped script that runs `cfn-init` and then `cfn-signal`. The numbered arrows trace the loop: the instance runs its `cfn-init` line (1), `cfn-init` consults the stack to pull the config (2), and the config is applied back to the instance (3). The footer — **"works with stack updates"** — points at the feature plain user data lacks.

### Re-running on updates — cfn-hup

[User data](ec2.md#bootstrapping) runs **only once**, at first launch. cfn-init can do more: a daemon called **cfn-hup** can be configured to **watch the resource's metadata** and **re-run `cfn-init`** whenever that metadata changes. So when you update the stack with new `AWS::CloudFormation::Init` directives, cfn-hup notices and re-applies the new desired state on the already-running instance — no replacement, no manual intervention. (In the demo, the `install_cfn` config key is what writes `cfn-hup.conf` and the auto-reloader hook, and starts the `cfn-hup` service.)

> [!warning] Comparison trap
> "Needs to **re-apply** config on a **stack update**, not just at first launch" points to **cfn-init + cfn-hup** — **not** user data. User data is strictly one-shot.

## Creation Policies & cfn-signal

This solves a real gap: by default, **CloudFormation has no idea whether in-instance bootstrapping actually succeeded.**

### The problem

CloudFormation creates the instance and passes in the user data, but it does **not understand** that user data. If the bootstrap fails, the instance is **still marked `CREATE_COMPLETE`** — because, as far as EC2 is concerned, the instance launched fine. That is acceptable for a blank resource (if EC2 says it provisioned, it did), but **not** when there is meaningful in-instance configuration that could silently fail. This mirrors the same "[failed bootstrap still passes status checks](ec2.md#bootstrapping)" trap that affects raw user data.

### The solution

A **CreationPolicy** attached to a logical resource creates a **hold point**. With it, CloudFormation does **not** move the resource to `CREATE_COMPLETE` when EC2 reports the launch succeeded; instead it **waits for a signal from the resource itself**, up to a configured **timeout** (the demo uses `PT15M`, 15 minutes).

The instance sends that signal by running **`cfn-signal`**:

```bash
/opt/aws/bin/cfn-signal -e $? --stack ${AWS::StackId} --resource EC2Instance --region ${AWS::Region}
```

- **`-e $?`** captures the **exit code of the previous command** — the `cfn-init` run. If `cfn-init` succeeded, `$?` is `0` and a success signal is sent; if it errored, the error code is sent.
- The stack ID, resource name, and region are substituted by CloudFormation before injection.

What CloudFormation does with the signal:

| Signal received | Result |
| --- | --- |
| Success code | Resource moves to `CREATE_COMPLETE` |
| Error code | Resource fails |
| **No signal within the timeout** | CloudFormation **assumes failure** — the resource errors and the stack does not complete |

![Creation policy and signals architecture, contrasting two rows. Top row, without a creation policy: Template, Stack, Instance — the instance launches and is marked complete with a green tick, but a yellow question-mark box on the far right shows the problem: CloudFormation has no idea whether the in-instance bootstrapping actually worked, it trusts EC2's "provisioned" report and moves on. Bottom row, with a creation policy: the EC2Instance resource now carries CreationPolicy, ResourceSignal, Timeout PT15M shown in a left code block. The Template-Stack-Instance flow is the same, but the stack now waits at a hold point instead of auto-completing. An orange dotted arrow traces the signal from the instance's UserData back to the stack — the cfn-signal -e $? line (highlighted at the bottom) sends the exit code of the preceding cfn-init command. Only on a success signal does the resource reach CREATE_COMPLETE, shown as a green-tick instance at the bottom right](../assets/cfn-init-CreationPolicy-and-signals.png)

Reading the diagram: the **top row (no creation policy)** ends in a yellow "?" — CloudFormation marks the instance complete on EC2's word alone, blind to whether the bootstrap worked. The **bottom row (with creation policy)** adds `CreationPolicy → ResourceSignal → Timeout: PT15M`, turning that blind completion into a **hold point**. The orange dotted arrow is the `cfn-signal -e $?` line reporting the result of `cfn-init` back to the stack; only a success signal advances the resource to `CREATE_COMPLETE`. The creation policy converts the top row's blind "?" into the bottom row's instance-confirmed completion.

> [!tip] Why it matters
> A creation policy is how you **explicitly know** the in-instance configuration finished — you are no longer trusting "EC2 says it's running 2/2 [status checks](ec2.md#status-checks-and-auto-recovery)." Creation policies show up on **EC2 instances** and **Auto Scaling Groups** in real templates.

## Diagnosing on the Instance

When you read the user data on a cfn-init instance (via [IMDS](ec2.md#instance-metadata-service-imds)), it is **short** — only the lines that invoke `cfn-init` and `cfn-signal` — and the `${...}` placeholders are **already replaced** with the real stack ID and region (proof that CloudFormation substituted them before injection).

```bash
# IMDSv2: token first, then read user data
TOKEN=`curl -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600"`
curl -H "X-aws-ec2-metadata-token: $TOKEN" -v \
  http://169.254.169.254/latest/user-data/
```

Relevant logs in `/var/log/`:

| Log file | Source |
| --- | --- |
| `cloud-init.log` / `cloud-init-output.log` | User-data / cloud-init processing (the line that launches cfn-init) |
| `cfn-init-cmd.log` | Output of the **cfn-init** process — shows each config key executing |

```bash
sudo cat /var/log/cfn-init-cmd.log    # watch each config key run
```

> [!warning] Gotcha
> The exact `cfn-init*` log filenames can vary slightly by helper-script version — confirm the name on the instance if it comes up.

## User Data vs cfn-init — Quick Comparison

| Feature | **User Data** | **cfn-init** |
| --- | --- | --- |
| Configuration model | Procedural (script, line by line) | Procedural **and** desired-state |
| Execution frequency | **Once**, at first launch only | Can **re-run** on stack updates (via cfn-hup) |
| Version-aware package install | No (you script it manually) | Yes (declare the desired version) |
| Where config is stored | The user-data block itself | `AWS::CloudFormation::Init` metadata in the template |
| How it's invoked | OS runs it directly at launch | Invoked **from** user data (the `cfn-init` command) |
| Knows if config succeeded? | No (resource completes regardless) | Yes, when paired with **CreationPolicy + cfn-signal** |
| Complexity | Simple | More complex, more powerful |
| Exam relevance | **Core** | Architecture awareness only |

## Exam Recap

- **cfn-init** is desired-state config management — **version-aware** and **re-runnable on updates** (via **cfn-hup**); [user data](ec2.md#bootstrapping) is one-shot procedural.
- Config lives in **`AWS::CloudFormation::Init`** metadata; `cfn-init` is **invoked from the user data**.
- CloudFormation **substitutes** `${AWS::StackId}` / `${AWS::Region}` before injecting, so the helper scripts can reach the right stack.
- **CreationPolicy + cfn-signal** is the only way CloudFormation **knows** in-instance bootstrapping succeeded — it **waits for a signal** with a **timeout** and assumes failure if none arrives.
- **`-e $?`** in `cfn-signal` passes the exit code of the preceding `cfn-init` run as the success/failure signal.
- Creation policies appear on **EC2 instances** and **Auto Scaling Groups**.
- Diagnose with `/var/log/cfn-init-cmd.log`.

## References

- [EC2 — Bootstrapping (user data)](ec2.md#bootstrapping)
- [Boot Time to Service Time — AMI Baking vs Bootstrapping](../patterns/boot-time-optimization.md)
- [EC2 — Instance Metadata Service (IMDS)](ec2.md#instance-metadata-service-imds)
- Source: Adrian Cantrill SAA course — Advanced EC2 (Enhanced Bootstrapping with CFN-INIT; CFN Creation Policies). Demo artifact: `cfn-init-template.yaml`
