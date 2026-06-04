# EC2 Instance Status Checks & Auto Recovery

## Core Idea

Every EC2 instance is continuously monitored by **two independent status checks**. Together they answer a simple question: _is the platform underneath healthy, and is the instance itself healthy?_ When something breaks, AWS gives you both manual and automated ways to respond — the headline automated option being **auto recovery**.

---

## The Two Status Checks

Each check is a separate set of tests, so **which one fails tells you where the problem lives**. On launch you'll often see `initializing`, then maybe `1/2`, and a healthy instance settles into **`2/2 checks passed`**. Anything other than 2/2 (on an instance that isn't still booting) means there's a problem to resolve.

|Check|Also shown as|What it watches|Typical failure causes|
|---|---|---|---|
|**System status**|System Reliability Check|The EC2 **service / host** underneath the instance|Loss of system power, lost network connectivity, hardware or software fault on the EC2 host|
|**Instance status**|Instance Reachability Check|The **instance itself** (OS layer)|Corrupt file system, misconfigured networking inside the OS, kernel issues preventing proper startup|

**Mental model:** System check = _"Is the building's power and plumbing working?"_ Instance check = _"Is the problem inside my own apartment?"_

> ⚠️ **Gotcha — the static public IPv4 trap:** A classic instance-check failure is statically setting a **public IPv4 address on the OS's internal interface**. This never works — the public IP is mapped externally by AWS, never configured inside the guest OS.

---

## Responding to a Failed Check

### Manual options (you do it)

- Stop and start the instance
- Restart / reboot the instance
- Terminate and recreate the instance

### Automated option — the Status Check Alarm

You create a **CloudWatch alarm** (Actions → _Create Status Check Alarm_) that fires when the instance fails **either** status check.

- **Trigger condition:** any status check fails for **at least one consecutive 5-minute period**.
- **Notification:** alarm sends a message to an **SNS topic**.
- **Optional action** (in addition to the notification), choose one of:

|Action|What it does|When it's useful|
|---|---|---|
|**Reboot**|OS-level reboot|Quick fix for transient instance issues|
|**Stop**|Stops the instance|Lets you run diagnostics on a "frozen" state|
|**Terminate**|Destroys the instance|Pairs with HA setups (e.g., Auto Scaling re-provisions a replacement). On a lone instance, it's just gone — nothing replaces it|
|**Recover**|Triggers **auto recovery** (see below)|Recover from an isolated host/instance failure with config intact|

---

## EC2 Auto Recovery (the main event)

Auto recovery attempts to bring a failed instance back to life by taking a set of actions — anywhere from a **simple restart** to **migrating the instance to a brand-new EC2 host**.

**What's preserved:** the instance comes up with **exactly the same configuration** — including all **IP addressing** (private IP, public IP, Elastic IP, etc.). If software inside the instance is set to **auto-start**, the instance can recover _fully_ with no human involvement. Hence the name.

**Purpose:** lightweight automation to handle a **narrow set of error scenarios** and "avoid waking up a sysadmin." It is **not** designed to fix large-scale or complex system problems.

### Limits & Requirements (high exam value)

|Constraint|Detail|
|---|---|
|**Same AZ only**|EC2 is an **AZ-based service**. Recovery moves the instance within the **same availability zone** → it does **NOT** protect against a full AZ failure, only isolated host/instance failures|
|**Spare capacity needed**|Relies on AWS having **spare EC2 host capacity**. In a major multi-AZ regional failure, recovery may not work if there's no spare capacity|
|**Supported instance types only**|Requires modern types — e.g. **A1, C4, C5, M4, M5, R3, R4, R5** (check AWS docs for the full current list)|
|**EBS only**|Won't work if the instance uses **instance store volumes** — the instance must use **EBS volumes exclusively**|

> 💡 **Why EBS-only makes sense:** Instance store is physically attached to the host. Migrating to a new host means that local storage can't follow — so auto recovery only works when all storage (EBS) is network-attached and independent of the host.

---

## Exam Tips

- **2/2 = healthy.** Memorize which check maps to which layer: **System → host/service**, **Instance → OS inside the instance**.
- Alarm fires on **either** check failing for **one 5-minute period**.
- **Recover** preserves **all IP addressing** and stays in the **same AZ** → no protection against AZ-wide outages.
- Auto recovery needs: **supported instance type + EBS-only volumes + spare host capacity**.
- **Terminate** action only auto-replaces the instance if you've built **HA** around it (e.g., Auto Scaling); on its own it just deletes.