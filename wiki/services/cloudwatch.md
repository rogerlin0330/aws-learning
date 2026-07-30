---
title: Amazon CloudWatch
type: service
tags: [cloudwatch, monitoring, logging, observability, ec2, lambda, cloudtrail, vpc, cloudwatch-agent, cwagent, metrics, parameter-store, instance-role, log-group, log-stream]
certs: [aws-saa]
updated: 2026-06-12
---

# Amazon CloudWatch

CloudWatch is AWS's native **monitoring and observability** platform. It covers two related but distinct areas: **Metrics** (numeric telemetry over time) and **Logs** (text-based event data). This page focuses on CloudWatch Logs; CloudWatch Metrics are used throughout AWS and are referenced on individual service pages.

## CloudWatch Logs

CloudWatch Logs is a **public AWS service** — it is hosted in the AWS public zone and is accessible via public endpoints from:

- Within [VPCs](vpc.md) (with appropriate routing),
- On-premises environments (with network connectivity and [IAM](iam.md) permissions),
- Other cloud platforms.

The service's primary function is to **store, monitor, and access logging data**.

### Log Data Model

Every log entry is a **log event**, which consists of:

- A **timestamp** (year, month, day, hour, minute, second, timezone)
- A **message block** — raw, unstructured data; CloudWatch Logs does not interpret the content

Log events are organized in a hierarchy:

![CloudWatch log group → log stream → log event hierarchy](../assets/cloudwatch-log-group-stream-event-relationship.png)

**Log Streams** — an ordered sequence of log events from a single source (e.g. one EC2 instance writing `/var/log/messages`). Multiple instances each get their own stream.

**Log Groups** — containers for multiple log streams of the same type. All configuration lives at the group level and applies to every stream within it:

- **Retention policies** — how long log events are kept (default: never expire)
- **Permissions** — [IAM](iam.md) resource policies controlling who can read/write the group
- **Metric filters** — pattern-matching rules that generate CloudWatch Metrics from log data

### Ingestion Methods

| Method | When to use |
| --- | --- |
| Native AWS service integration | [EC2](ec2.md), [VPC](vpc.md) Flow Logs, Lambda, [CloudTrail](cloudtrail.md), Route 53 — zero config, logs flow automatically via service/IAM roles |
| **[Unified CloudWatch Agent](#cloudwatch-agent)** | Custom OS logs (e.g. `/var/log/messages` on EC2), on-premises servers, non-AWS compute |
| AWS SDK | Application-level logging from any language — call `PutLogEvents` directly |

Security for native integrations is managed via **IAM service roles** attached to the originating service (e.g. a Lambda execution role that includes `logs:CreateLogGroup`, `logs:PutLogEvents`).

### Metric Filters and Alarms

Metric filters let you **generate numeric metrics from log content**:

1. Define a filter pattern on a log group (e.g. match the string `"Failed password"` in sshd logs)
2. Each match increments a custom CloudWatch Metric
3. Set an **alarm** on that metric to notify ([SNS](sns.md)) or trigger automated responses (Lambda, Auto Scaling, EC2 Actions)

This pipeline is how AWS converts unstructured logs into actionable telemetry without exporting data to a separate tool.

> [!warning] Gotcha — Alarms vs EventBridge
> A CloudWatch **Alarm** reacts to a **metric crossing a threshold**. [Amazon EventBridge](eventbridge.md) (formerly CloudWatch Events, which split out of CloudWatch) reacts instead to **state-change events** and **schedules**. Same family, different triggers — don't blur them on the exam.

The canonical example of the alarm + **EC2 action** pipeline is the [EC2 Status Check Alarm](ec2.md#status-checks-and-auto-recovery) — a built-in alarm on either of EC2's two instance status checks that can notify via SNS and optionally **Reboot / Stop / Terminate / Recover** the instance (the **Recover** action triggers [EC2 auto recovery](ec2.md#ec2-auto-recovery)).

## CloudWatch Logs as the Logging Backbone

CloudWatch Logs is the default log destination for most AWS services. Key integrations:

| Service | What it logs |
| --- | --- |
| [CloudTrail](cloudtrail.md) | API call history — who called what, when, from where |
| Lambda | Function execution output (stdout/stderr) |
| [VPC](vpc.md) Flow Logs | Network traffic flow records (accepted/rejected) |
| [EC2](ec2.md) | OS and application logs (via CloudWatch Agent) |
| Route 53 | DNS query logs |

## CloudWatch Agent

Standard CloudWatch only sees an instance from the **outside** — host-level metrics measured at the hypervisor, such as CPU utilization. Everything **inside** the guest OS is opaque to CloudWatch by default: memory usage, disk-space-used, per-process and per-core detail, and any system or application **log files**. The **CloudWatch Agent** (the *unified* agent, which replaced the older standalone Logs agent) closes that gap. It runs inside the instance, captures OS-visible metrics and logs, and injects them into CloudWatch (metrics) and [CloudWatch Logs](#cloudwatch-logs).

### Why CPU is visible but memory is not

The dividing line is **what the hypervisor measures to do its job** versus **what is bookkeeping internal to the guest OS**:

- **CPU is time-shared.** The hypervisor schedules your vCPUs onto physical cores and meters that dispatch directly — when the guest is idle it halts the vCPU and the hypervisor sees it yield. "What fraction of allotted CPU time did the instance actually use?" is a number the hypervisor already tracks, so `CPUUtilization` needs nothing inside the guest. (This is also why it can differ slightly from `top` — it is measured externally.)
- **Memory is space-allocated.** The instance gets a fixed RAM quota up front; after that the hypervisor's view is static ("these pages are assigned to this guest"). Used vs. free vs. cache vs. available is an abstraction the **guest kernel** maintains in its own structures (`/proc/meminfo`). The hypervisor does not parse those, and "pages ever touched" is useless because Linux fills nearly all free RAM with page cache. Only a process inside the guest can report the real figure.

Disk splits across both rules, which confirms the pattern:

| Metric | Visible without the agent? | Why |
| --- | --- | --- |
| Disk **I/O** (read/write ops, bytes, queue length) | Yes — EBS metrics | EBS is a network service AWS meters; it is *activity / throughput*, like CPU time |
| Disk **space used** (filesystem % full) | No — needs the agent | Filesystem fullness is guest bookkeeping; to AWS the volume is just a fixed-size blob |

So the real rule is not "CPU yes, memory no." It is: **activity/throughput of a resource AWS meters** (CPU time, network bytes, EBS IOPS) is externally observable, while **capacity-used accounting owned by the guest OS** (memory used, disk-space used) is not — and reading the latter requires a process inside the guest, which is exactly what the agent provides.

> [!tip] Exam Tip
> Any question mentioning **memory usage**, **disk space used**, **per-process metrics**, or **OS / application logs from EC2** points to the **CloudWatch Agent**. Standard CloudWatch metrics do **not** include in-guest memory or disk-used.

### What the agent needs

Installing the agent is not enough; it also requires two things:

1. **Configuration** — a JSON document telling it *which* metrics and *which* log files to collect.
2. **Permissions** — credentials to write to CloudWatch and CloudWatch Logs. The correct pattern is an **[EC2 instance role](ec2.md#instance-roles--instance-profiles)** (the managed `CloudWatchAgentServerPolicy`), never long-term access keys baked onto the box. Anything running on the instance — the agent included — inherits the role.

### Log groups and log streams

When it ships logs, the agent follows a simple mapping into the [CloudWatch Logs hierarchy](#log-data-model):

- **One log group per log file** being captured (for example `/var/log/secure`, `/var/log/httpd/access_log`).
- **One log stream per instance** sending that file, named after the instance ID by default.

So *N* log files across *M* instances produce *N* log groups, each holding up to *M* streams. The mnemonic: **group by file, stream by instance**.

### Storing config at scale — Parameter Store

The agent's JSON config can be stored once in [SSM Parameter Store](ssm.md#parameter-store) and pulled by any number of instances, which is what makes the agent practical across a fleet (especially with Auto Scaling groups). The control utility fetches the config and starts the agent in one command:

```bash
amazon-cloudwatch-agent-ctl -a fetch-config -m ec2 -c ssm:AmazonCloudWatch-linux -s
```

Here `-a fetch-config` fetches the config, `-m ec2` sets EC2 mode, `-c ssm:<name>` names the Parameter Store parameter, and `-s` starts the agent. Other ways to deliver the agent and its config: [CloudFormation](cloudformation.md), Systems Manager, baking it into an [AMI](ec2.md#ami-baking), or [user-data bootstrapping](ec2.md#bootstrapping).

### The CWAgent namespace

The richer OS-level metrics — disk I/O read/write, per-core CPU including I/O wait and user time — appear under a new **`CWAgent`** namespace in CloudWatch Metrics, dimensioned by image ID, instance ID, and instance type. A `CWAgent` namespace appearing is the fingerprint that the agent is installed and running.

> [!warning] Gotcha — run-as user vs file permissions
> The agent runs as an OS user (modern config wizards default to **`cwagent`**; older setups and AWS's documented default run as **`root`**). The agent can only capture a log file if **its run-as user has read permission on that file** — a question entirely separate from IAM. Root-restricted logs such as `/var/log/secure` (mode `0600 root:root`) are **silently skipped** by a `cwagent`-run agent. So if the agent is installed and configured but **certain logs never appear**, suspect a **file-permission mismatch**, not an IAM or config error: *IAM governs whether the agent can **send**; OS file permissions govern whether it can **read** the source log.*

## Resilience and Scope

CloudWatch is a **regional service** — log groups and their data are region-scoped. Cross-region log aggregation requires explicit forwarding (e.g. Kinesis Data Firehose or Lambda subscription filters to a central region).

> [!tip] Exam Tip
> **Log Group = unit of configuration.** Retention, permissions, and metric filters all attach to the log group and inherit down to every stream it contains. Stream-level configuration does not exist.

> [!warning] Gotcha
> The Unified CloudWatch Agent is required for **any log source that isn't a native AWS service** — including OS-level logs from EC2 instances. EC2 does not automatically send logs to CloudWatch; only metrics like CPU are available without the agent.

## References

- [IAM](iam.md) — roles and permissions for log access
- [VPC](vpc.md) — VPC Flow Logs sent to CloudWatch Logs
- [EC2](ec2.md) — OS and app logs via the [CloudWatch Agent](#cloudwatch-agent); [instance roles](ec2.md#instance-roles--instance-profiles) supply the agent's permissions
- [Systems Manager (SSM)](ssm.md#parameter-store) — Parameter Store holds the agent config for fleet-wide reuse
- [CloudTrail](cloudtrail.md) — API/account-activity audit; delivers events to CloudWatch Logs for search + alarms
- [EventBridge](eventbridge.md) — CloudWatch Events' successor; event bus reacting to state changes and schedules (vs Alarms' metric thresholds)
- [Organizations](organizations.md) — CloudTrail integration context
- Source: `raw/aws-cert-study-notes/aws-solution-architect/ec2_system_and_application_logging.md`
