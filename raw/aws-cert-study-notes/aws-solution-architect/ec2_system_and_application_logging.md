# System & Application Logging and Metrics on EC2 — The CloudWatch Agent

## The Core Problem

CloudWatch monitors performance and reliability metrics for EC2, but **only what is visible on the external face of an instance** (e.g. CPU utilization at the hypervisor level). CloudWatch and CloudWatch Logs **cannot natively see inside an instance** — the inside of an instance is _opaque_ to both products by default.

There are two categories of "inside the instance" visibility you often need:

- **OS-level performance metrics** — process list, per-process memory consumption, disk I/O (read/write), per-core CPU detail (I/O wait, user values), etc. These exist _inside_ the OS and aren't exposed externally.
- **System and application logs** — log files written by the OS and by applications running on the instance.

> 🎯 **Exam trigger:** "memory usage", "disk space/usage", "process-level metrics", "in-OS / guest logs", or "application/system logs from EC2" → the answer involves the **CloudWatch Agent**. Standard CloudWatch metrics do **not** include memory or disk-space-used from inside the OS.

---

## What the CloudWatch Agent Is

The **CloudWatch Agent** is a piece of software that runs **inside an EC2 instance** (on the OS). It captures OS-visible data — metrics _and_ logs — and sends it into **CloudWatch** (metrics) and **CloudWatch Logs** (log data), where it can then be viewed and visualized in the respective consoles.

For the agent to function it needs two things in addition to being installed:

1. **Configuration** — telling the agent exactly _what_ metrics and _which_ log files to capture and inject.
2. **Permissions** — credentials to interact with CloudWatch and CloudWatch Logs.

---

## Architecture

```
                 ┌─────────────────────────────────────────┐
                 │             EC2 Instance                 │
                 │                                          │
                 │   ┌──────────────────────────────────┐   │
   IAM Role  ───▶│   │     CloudWatch Agent             │   │
 (permissions)   │   │   + Agent Configuration          │   │
                 │   └──────────────────────────────────┘   │
                 │        │ captures OS metrics & logs       │
                 └────────┼─────────────────────────────────┘
                          │
            ┌─────────────┴──────────────┐
            ▼                            ▼
     ┌─────────────┐            ┌──────────────────┐
     │ CloudWatch  │            │ CloudWatch Logs  │
     │ (metrics:   │            │ (log groups +    │
     │  CWAgent NS)│            │  log streams)    │
     └─────────────┘            └──────────────────┘
```

Reading the diagram — the EC2 instance is the central object. Inside it sit the **agent** plus its **configuration**. Feeding into the instance from the side is the **IAM role** that grants permissions. Flowing out of the instance are two destinations: **CloudWatch** for metrics and **CloudWatch Logs** for log data.

### Permissions — use an IAM role, not credentials

Adding long-term credentials to an instance is bad practice and hard to manage at scale. Best practice:

- Create an **IAM role** with permissions to interact with CloudWatch / CloudWatch Logs.
- **Attach the role to the EC2 instance.** Anything running on the instance (including the agent) then inherits that access.

> ⚠️ **Trap:** Don't hard-code IAM access keys onto the instance for the agent. The exam-correct pattern is an **instance role**.

### Log groups and log streams

The agent injects logs into CloudWatch Logs using **log groups** and **log streams**:

- **One log group per log file** you want to capture.
- **One log stream per instance** that is injecting that particular log.

So for _N_ log files across _M_ instances, you get _N_ log groups, and within each group up to _M_ log streams (named after the instance ID by convention).

> 🎯 Mnemonic — **"Group by file, Stream by instance."**

### Deploying at scale

For a single instance you can do everything manually (log in, install, configure, attach role, start). At scale you automate:

- **CloudFormation** to include the agent configuration on every instance provisioned.
- Store the agent **config in Parameter Store** once, then reference it from many instances (especially with Auto Scaling groups, covered later).
- Other install methods exist too: **Systems Manager**, baking it into **AMIs**, or **bootstrapping** via user data.

---

## Demo Walkthrough — Installing & Configuring the Agent

> The demo runs on an _Animals for Life_ WordPress instance (Apache web server installed) in the **Northern Virginia (us-east-1)** region, deployed via a one-click CloudFormation template. _(Region and instance names are illustrative demo specifics, not values to memorize.)_

The agent is set up to collect **three log files**:

|Log file path|What it shows|
|---|---|
|`/var/log/secure`|Secure-login events to the instance (system log)|
|`/var/log/httpd/access_log`|Apache access log (application log)|
|`/var/log/httpd/error_log`|Apache error log — requests not successfully served, server/module errors|

### Step 1 — Install the agent

```bash
sudo dnf install amazon-cloudwatch-agent
```

This installs the agent but does **not** start it. Before starting, you need a config file and permissions.

### Step 2 — Create and attach the IAM role

Create an IAM role (Type: **EC2** / AWS service role) with two managed policies attached:

- `CloudWatchAgentServerPolicy` — lets the agent send metrics/logs.
- `AmazonSSMFullAccess` — lets the agent read/write the Parameter Store config.

Name it `CloudWatchRole`, then attach it to the instance: **EC2 → Instances → right-click instance → Security → Modify IAM Role → select `CloudWatchRole` → Update IAM role.**

> ⚠️ Reconnect to the instance after attaching the role — the previous SSH/Instance Connect tab may have timed out.

### Step 3 — Run the configuration wizard

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard
```

Accept defaults through most prompts. Notable non-default choices:

- **Default metric config:** choose **Advanced** (enter `3`), not Basic — captures additional OS-level metrics.
- **Log files to monitor:** add the three files above. For each, when asked for the **log group name**, the default is just the bare log name, but the demo enters the **full path** (e.g. `/var/log/secure`) — a cleaner naming convention for system logs.
- **Log stream name:** accept the default (the **instance ID**).
- **Retention:** accept the default for each.

> ⚠️ **Changed since the demo — agent run-as user.** The demo (an older recording) runs the agent as **root**. Current AWS wizards present **`cwagent`** as the **default** run-as user (a Linux user created by the RPM/DEB package), which the wizard writes as `"run_as_user": "cwagent"` in the agent section of the config. This matters because **the agent process can only read a log file if the run-as user has Read (`r`) permission on that file.** Several monitored files are root-restricted — notably `/var/log/secure` (typically `0600 root:root`) — so a `cwagent`-run agent would **silently fail to read them** unless you either:
> 
> - run the agent as **root** (override the wizard default), **or**
> - grant the `cwagent` user read access to those files (e.g. via group membership or a filesystem ACL).
> 
> 🎯 **Exam-relevant takeaway:** if the agent is installed and configured but **certain logs never appear** in CloudWatch Logs, suspect a **file-permission mismatch** between the log file and the agent's run-as user — not a config or IAM problem. (IAM governs whether the agent can _send_ to CloudWatch; OS file permissions govern whether it can _read_ the source log.)
> 
> _Note:_ AWS docs state the agent's documented default is `root`, but the **wizard's** presented default is `cwagent` — so the two "defaults" differ. Verify the `run_as_user` value in your generated config rather than assuming.

The wizard writes the config locally to:

```
/opt/aws/amazon-cloudwatch-agent/bin/config.json
```

…and also offers to store it in **Parameter Store**. Accept this. Default parameter name: `AmazonCloudWatch-linux`. Region auto-detected (us-east-1). Credentials are taken from the attached instance role.

> 💡 The stored Parameter Store value is a **JSON document** containing the full agent configuration. Create it once, then reuse it to configure the agent on any number of instances at scale, securely (no config baked into each instance).

### Step 4 — Bug fix (Amazon Linux)

The agent expects `CollectD` system software, which is **not installed on Amazon Linux**. Without this the agent **won't start**. Create the directory and database file it expects:

```bash
sudo mkdir -p /usr/share/collectd/
sudo touch /usr/share/collectd/types.db
```

### Step 5 — Load config from Parameter Store and start the agent

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config -m ec2 -c ssm:AmazonCloudWatch-linux -s
```

Breakdown of the command:

- `amazon-cloudwatch-agent-ctl` — the agent control utility.
- `-a fetch-config` — action: fetch the configuration.
- `-m ec2` — mode: running on EC2.
- `-c ssm:AmazonCloudWatch-linux` — config source: the **SSM (Parameter Store)** parameter named `AmazonCloudWatch-linux`.
- `-s` — start the agent.

This pulls the config from Parameter Store, configures the agent, ensures it's running, and begins injecting logs (and metrics) into CloudWatch.

---

## Verifying the Result

**Logs — CloudWatch → Log Groups.** Scroll to the bottom to find the created groups. Log groups/entries only appear once new log entries are generated and sent.

- If `access_log` isn't there yet, browse to the instance's **public IPv4 address** in a new tab to generate Apache activity → access entries get written → injected into CloudWatch Logs.
- Open a log group → the **log stream** is named after the instance ID.

> ⚠️ You may see _many_ unrelated log groups. The Animals for Life VPC template uses a Lambda function (an IPv6 workaround, covered later) that creates its own. Ignore these; look for your three.

**Metrics — CloudWatch → Metrics.** Alongside the standard AWS namespaces there is now a **`CWAgent`** namespace. Inside it (dimensioned by image ID / instance ID / instance type) are the richer OS-level metrics — e.g. **disk I/O read/write**, and **per-core CPU** detail including **I/O wait** and **user** values — none of which are available without the agent.

> 🎯 **Exam trigger:** A new **`CWAgent`** namespace in CloudWatch is the fingerprint of the CloudWatch Agent being installed.

---

## Cleanup

- **EC2 → instance → Security → Modify IAM Role →** detach (remove) the CloudWatch role.
- **IAM → Roles →** delete `CloudWatchRole`.
- **CloudWatch → Log Groups →** delete the three groups (`/var/log/secure`, `/var/log/httpd/access_log`, `/var/log/httpd/error_log`).
- **CloudFormation →** delete the `CWAgent` stack.
- **Leave the Parameter Store value in place** — it's a _standard_ parameter (no charge) and is reused in later lessons.

---

## 🎯 Exam Quick-Reference Recap

- **Standard CloudWatch = outside-the-instance metrics only.** No memory usage, no disk-space-used, no in-OS logs.
- **CloudWatch Agent** runs _inside_ the OS to capture **OS-level metrics** + **system/application logs**, injecting them into **CloudWatch** and **CloudWatch Logs**.
- Agent needs **(1) configuration** and **(2) permissions via an IAM (instance) role** — never long-term credentials.
- **Run-as user:** modern wizards default to **`cwagent`** (older demos use `root`). The agent can only read a log file if its run-as user has **Read permission** on it — root-restricted logs like `/var/log/secure` won't be captured by `cwagent` unless you run as root or grant read access. _IAM = can it send; OS file perms = can it read._
- **Log group per log file; log stream per instance** ("Group by file, Stream by instance"). Streams default to the **instance ID**.
- Agent config can be stored in **SSM Parameter Store** (as JSON) and reused **at scale** — pairs naturally with CloudFormation and Auto Scaling groups.
- Installed-agent metrics appear under the **`CWAgent`** namespace.
- Other install/deploy methods: **Systems Manager**, **AMI baking**, **user-data bootstrapping**, **CloudFormation**.

---

### Reference

- CloudWatch Agent: https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Install-CloudWatch-Agent.html
- Storing agent config in Parameter Store: https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/install-CloudWatch-Agent-on-EC2-Instance-fleet.html

_Source: Adrian Cantrill AWS SAA — "System and Application Logging on EC2" + "DEMO: Logging and Metrics with CloudWatch Agent" Parts 1 & 2. Commands from the AL2023 lesson_commands file._

> 📝 **Editorial notes:** The architecture ASCII diagram and the "Group by file, Stream by instance" mnemonic are additions for clarity (not in the source). Region (us-east-1), instance/log-group names, and the `AmazonCloudWatch-linux` parameter name are illustrative demo specifics, not memorizable facts. The `-a/-m/-c/-s` flag breakdown of the `fetch-config` command was added editorially from the command itself.