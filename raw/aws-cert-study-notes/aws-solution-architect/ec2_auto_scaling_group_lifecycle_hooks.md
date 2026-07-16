---

tags:

- aws
- saa-c03
- ec2
- auto-scaling
- asg
- lifecycle-hooks
- event-driven aliases:
- Auto Scaling Group Lifecycle Hooks
- ASG Hooks

---

# ASG Lifecycle Hooks

## Overview

**Lifecycle hooks let you configure custom actions that run during Auto Scaling group instance launch or terminate transitions.**

Normally, when an ASG scales out or scales in, the launch/terminate process is entirely under the ASG's control. The moment it decides to provision or terminate an instance, that process runs to completion with **no ability for you to influence the outcome**.

A lifecycle hook changes that. It inserts a **pause** into the flow — the instance enters a _wait_ state and stays there until one of two things happens:

1. A **configurable timeout expires** (default **3600 seconds**), at which point the ASG action either **continues** or is **abandoned**, or
2. You **explicitly resume** the process by calling **`CompleteLifecycleAction`** once your custom activity is done.

There are exactly two lifecycle transitions you can hook:

|Transition|API value|Fires on|
|---|---|---|
|Instance **launch**|`autoscaling:EC2_INSTANCE_LAUNCHING`|Scale **out**|
|Instance **terminate**|`autoscaling:EC2_INSTANCE_TERMINATING`|Scale **in**|

Additionally, lifecycle hooks integrate with **EventBridge** or **SNS notifications**, letting your systems perform **event-driven processing** based on the launch or termination of EC2 instances within an ASG.

---

## Diagram Walkthrough

![[asg-lifecycle-hooks.png]]

### Context — the outer frame

The whole diagram sits inside an **orange dotted boundary** labelled **Auto Scaling Group** (orange ASG icon, top-left). Everything inside the dotted line is ASG-managed state. The two services below the dotted line (SNS, EventBridge) are _outside_ the ASG — they're integration targets, which is why the arrows to them cross the boundary.

The diagram is split into two mirrored halves:

- **Left half** — the **Scale Out** / launch flow (labelled with a **green** "Scale Out" and a green up-arrow)
- **Right half** — the **Scale In** / terminate flow (labelled with a **red** "Scale In" and a red down-arrow)

The color coding is meaningful: **green = healthy/growing**, **red = shrinking/terminating**.

### Left side — the launch flow

Read the left half **bottom-to-top, then left-to-right**:

1. **Green up-arrow → `PENDING` (yellow)** — a scale-out event fires and an instance is launched. It starts in `Pending`.
2. **Grey dotted line from `PENDING` → `InService` (green)** — this is the **normal path with no hook**. The dotted style signals "this is what _would_ happen"; it's the shortcut you're replacing. Note the state box for `InService` is **green** — the only green box, because it's the only "instance is doing its job" state.
3. **Grey solid arrow up: `PENDING` → `PENDING: Wait` (yellow)** — with a launch hook defined, the instance diverts here instead and **waits**.
4. **Database icon + "e.g. Load or Index data"** — the callout sits _between_ the wait and proceed states, indicating this is the window where your custom action runs. The **plus badge** on the database icon signals data being _added_ (loaded/indexed).
5. **`PENDING: Wait` → `PENDING: Proceed`** — the grey arrow across the top. This is the transition triggered by timeout expiry or `CompleteLifecycleAction`.
6. **`PENDING: Proceed` → `InService`** — grey arrow down. Instance enters service.

All launch-flow boxes are **yellow** (transitional) except the terminal `InService` in **green**.

### Right side — the terminate flow

Read the right half **top-to-bottom, then left-to-right**, mirroring the left:

1. **Red down-arrow → `Terminating` (orange)** — a scale-in event fires.
2. **Grey dotted line `Terminating` → `Terminated` (red)** — again the **no-hook normal path**, dotted for the same reason.
3. **`Terminating` → `Terminating: Wait` (orange)** — with a terminate hook, the instance diverts down and waits.
4. **Database icon + "e.g. Backup data or logs"** — same visual grammar as the left side, but the badge on the database icon is a **minus**, signalling data being _removed/extracted_ before the instance dies.
5. **`Terminating: Wait` → `Terminating: Proceed`** — grey arrow right.
6. **`Terminating: Proceed` → `Terminated` (red)** — grey arrow up.

Terminate-flow boxes are **orange** (transitional) with the terminal `Terminated` in **red**.

### The centre callout — the hinge of the whole diagram

Between the two halves, with a **stopwatch icon**:

> **State changes when timeout (`3600s` default) expires or `CompleteLifecycleAction`**

Note the grey arrow from this callout points at **`Terminating: Wait`** — but the rule applies **equally to both `Pending: Wait` and `Terminating: Wait`**. The arrow is positioned for layout convenience, not to scope the rule to termination only. `3600s` and `CompleteLifecycleAction` are both rendered in **pink**, Cantrill's convention for "exam-relevant value/API name".

### Footer — the integrations

Two service icons sit below the ASG dotted boundary, with **red arrows** running from the wait states down and outward:

- **Left: SNS icon** — "Notifications for lifecycle hooks can be sent to an SNS topic". Arrow points **from** the ASG **to** SNS (push notification out).
- **Right: EventBridge icon** — "EventBridge can be used to initiate other processes based on Hooks". Arrow points **from** the ASG **to** EventBridge (event fires, downstream automation runs).

Both arrows originate from a shared trunk rooted in the wait states — the visual point being that **entering a wait state is the event** that triggers the notification.

---

## Mechanics

### The full state sequences

**Launch (scale out):**

```
Pending → Pending:Wait → Pending:Proceed → InService
```

**Terminate (scale in):**

```
Terminating → Terminating:Wait → Terminating:Proceed → Terminated
```

Without a hook, the intermediate `:Wait` and `:Proceed` states **do not occur** — `Pending → InService` and `Terminating → Terminated` directly.

> 📚 It's the **`:Wait` and `:Proceed` pair** that creates the opportunity to run custom actions. Remembering that both hook states exist (not just `Wait`) is a common gap.

### Resolving the wait

|Mechanism|What happens|
|---|---|
|**`CompleteLifecycleAction`**|You explicitly signal done → instance moves to `:Proceed`. Pass `CONTINUE` or `ABANDON` as the result.|
|**Timeout expires**|ASG performs the action set in **`DefaultResult`**.|
|**`RecordLifecycleActionHeartbeat`** 📚|Restarts the timeout clock — buys more time without completing the action.|

### 📚 `DefaultResult` — CONTINUE vs ABANDON

Not covered in the lesson, but exam-relevant. <cite index="6-1">The result of the lifecycle hook can be either abandon or continue. If an instance is launching, continue indicates that your actions were successful, and that Amazon EC2 Auto Scaling can put the instance into service. Otherwise, abandon indicates that your custom actions were unsuccessful, and that we can terminate and replace the instance. If an instance is terminating, both abandon and continue allow the instance to terminate. However, abandon stops any remaining actions, such as other lifecycle hooks, and continue allows any other lifecycle hooks to complete.</cite>

|Result|On **launch**|On **terminate**|
|---|---|---|
|`CONTINUE`|Instance → `InService`|Instance terminates; **other hooks still run**|
|`ABANDON`|Instance **terminated and replaced**|Instance terminates; **remaining hooks skipped**|

⚠️ The asymmetry is the trap: on **launch**, `ABANDON` kills the instance. On **terminate**, `ABANDON` still terminates — it just stops other hooks from running. **`ABANDON` never rescues an instance.**

> ⚠️ **The `DefaultResult` default value differs by tooling** — this catches people out:
> 
> - **CloudFormation / CLI / SDK**: <cite index="5-1">The action the Auto Scaling group takes when the lifecycle hook timeout elapses or if an unexpected failure occurs. The default value is ABANDON.</cite>
> - **AWS CDK** (`BasicLifecycleHookProps`): defaults to **`CONTINUE`**.
> 
> For the exam, treat **`ABANDON` as the API-level default**. 📚 The practical implication: if you define a launch hook and forget to call `CompleteLifecycleAction`, after an hour your instances get **destroyed and replaced** in a loop — not left running.

### 📚 Failing hooks are rate-limited

<cite index="6-1">Amazon EC2 Auto Scaling limits the rate at which it allows instances to launch if the lifecycle hooks are failing consistently, so make sure to test and fix any permanent errors in your lifecycle actions.</cite> A broken hook doesn't just slow scaling — it throttles it.

### Integration targets

|Target|Use|
|---|---|
|**SNS topic**|Transition notifications — message carries instance info **and a token** used to control the lifecycle action 📚|
|**EventBridge**|Event-driven initiation of other processes (e.g. invoke Lambda)|
|**SQS queue** 📚|Also a valid `NotificationTargetARN` — per the API reference, the target can be <cite index="4-1">an Amazon SNS topic or an Amazon SQS queue</cite>. The lesson only names SNS.|
|**Script on the instance** 📚|The instance can call `CompleteLifecycleAction` for itself using its own instance ID.|

---

## ⚠️ Course vs. Current AWS

### 1. Timeout maximum — the transcript is incomplete, and the numbers are easy to conflate

The lesson states the default timeout is **3600s** and stops there. That default is confirmed — but there are **two distinct ceilings** current docs describe, and they are frequently mixed up:

|Concept|Value|Source|
|---|---|---|
|**Default** heartbeat timeout|**3600s** (1 hour)|<cite index="5-1">The range is from 30 to 7200 seconds. The default value is 3600 seconds (1 hour).</cite>|
|**`HeartbeatTimeout` parameter range**|**30 – 7200s** (max 2 hours)|Same|
|**Global timeout** (total wait, incl. heartbeats)|<cite index="4-1">The maximum is 172800 seconds (48 hours) or 100 times HeartbeatTimeout, whichever is smaller.</cite>|EC2 Auto Scaling API Reference|

⚠️ **These are not the same number.** `HeartbeatTimeout` caps a _single_ timeout window at 7200s. The **global timeout** caps the _total_ time an instance can sit in a wait state across repeated `RecordLifecycleActionHeartbeat` calls — <cite index="6-1">The global timeout is 48 hours or 100 times the heartbeat timeout, whichever is smaller.</cite>

> 📚 Worked example: with the default `HeartbeatTimeout` of 3600s, 100 × 3600s = 360,000s ≈ 100 hours. That's larger than 48 hours, so **48 hours wins** — the global cap. But with a `HeartbeatTimeout` of 60s, 100 × 60s = 6000s ≈ 1.67 hours, which is _smaller_ than 48 hours, so **6000s wins**. The "whichever is smaller" clause is doing real work at low heartbeat values.

### 2. CloudWatch Events → EventBridge

The lesson says EventBridge (correct, current). Older AWS docs and some third-party material still say **CloudWatch Events** — same service, renamed. Exam questions may use either name. 📚

### 3. Warm pools 📚

Not mentioned in the lesson. AWS later added **warm pools**, which introduce additional lifecycle states (`Warmed:Pending`, `Warmed:Running`, `Warmed:Stopped`, etc.) and can also have hooks attached. Out of scope for most SAA-C03 questions, but if a question mentions pre-initialised instances that are stopped and ready to go, that's warm pools — not lifecycle hooks alone.

---

## ⚠️ Exam Traps

|Trap|Reality|
|---|---|
|"Hooks let you cancel a scale-out"|✅ Sort of — `ABANDON` on a launch hook **terminates and replaces** the instance. It doesn't cancel the scaling decision.|
|"`ABANDON` on terminate saves the instance"|❌ **No.** Both `CONTINUE` and `ABANDON` still terminate it. `ABANDON` just skips remaining hooks.|
|"Default timeout is 300s / 1 minute"|❌ **3600s** (1 hour).|
|"Max wait is 7200s"|⚠️ That's the max **`HeartbeatTimeout`** for a single window. **Global** max is **48h or 100× heartbeat, whichever is smaller**.|
|"You must wait for the full timeout"|❌ Call **`CompleteLifecycleAction`** to proceed immediately.|
|"Hook fires on `InService` → healthy"|❌ Only **two** transitions: `EC2_INSTANCE_LAUNCHING` and `EC2_INSTANCE_TERMINATING`.|
|"SNS is the only notification target"|❌ **SNS or SQS** via `NotificationTargetARN`; EventBridge separately.|
|"Timeout expiry always lets the instance proceed"|❌ Depends on **`DefaultResult`** — API default is **`ABANDON`**.|
|Confusing with **ASG health checks**|Health checks decide _whether to replace_ an unhealthy instance. Hooks pause a _transition already in progress_.|
|Confusing with **user data / cfn-init**|User data runs at boot with no ASG coordination. A hook **holds the ASG state machine** until you signal — that's the difference.|

### 🎯 Trigger Keywords

|Keyword in question|Points to|
|---|---|
|"run a script **before** the instance receives traffic"|**Launch hook** (`Pending:Wait`)|
|"**pre-load** / warm cache / index data before serving"|**Launch hook**|
|"**copy logs off** / drain connections / back up before termination"|**Terminate hook** (`Terminating:Wait`)|
|"**graceful shutdown**" / "flush data before scale-in"|**Terminate hook**|
|"**delay** registration behind the load balancer"|**Launch hook**|
|"**troubleshoot** an instance before it's terminated"|**Terminate hook** (hold in `Terminating:Wait`)|
|"**pause** the instance in the lifecycle"|Lifecycle hooks (generic)|
|"**event-driven** response to scaling"|Hooks + **EventBridge**|
|"notify a **topic** when scaling occurs"|Hooks + **SNS**|
|"3600 / 1 hour / default timeout"|Lifecycle hook heartbeat timeout|
|"resume / signal completion early"|**`CompleteLifecycleAction`**|
|"need **more time**, extend the wait"|**`RecordLifecycleActionHeartbeat`**|

---

## Quick Reference

|Property|Value|
|---|---|
|**Transitions**|`autoscaling:EC2_INSTANCE_LAUNCHING`, `autoscaling:EC2_INSTANCE_TERMINATING`|
|**Launch states**|`Pending` → `Pending:Wait` → `Pending:Proceed` → `InService`|
|**Terminate states**|`Terminating` → `Terminating:Wait` → `Terminating:Proceed` → `Terminated`|
|**Default heartbeat timeout**|**3600s** (1 hour)|
|**Heartbeat timeout range**|30 – 7200s|
|**Global timeout**|48h **or** 100 × heartbeat, whichever is **smaller**|
|**`DefaultResult` (API default)**|**`ABANDON`**|
|**Complete early**|`CompleteLifecycleAction`|
|**Extend the clock**|`RecordLifecycleActionHeartbeat`|
|**Notification targets**|SNS topic, SQS queue|
|**Event-driven integration**|EventBridge|
|**CFN resources**|`AWS::AutoScaling::LifecycleHook`, or `LifecycleHookSpecificationList` inline on the ASG|

---

## Open Questions

- **Roger** — do you want a separate note on **warm pools**? They extend this state machine considerably and have their own hook transitions (`autoscaling:EC2_INSTANCE_LAUNCHING` behaves differently in a warm pool). Cantrill may cover it later; flagging in case it doesn't come up.
- Worth cross-linking to `[[Auto Scaling Groups]]` and `[[EC2 Launch Templates]]` once those notes are settled — the hook is defined _on the ASG_, not on the launch template, which is itself a mildly exam-worthy distinction.
- Does the course cover `RecordLifecycleActionHeartbeat` anywhere else? It's absent from this transcript but it's the only way the "100 × heartbeat" global timeout rule makes sense.

---

## Sources

- Lesson transcript: _ASG Lifecycle Hooks_, Adrian Cantrill AWS SAA course — [learn.cantrill.io](https://learn.cantrill.io/)
- Diagram: Adrian Cantrill (`asg-lifecycle-hooks.png`)
- [Amazon EC2 Auto Scaling lifecycle hooks — AWS Docs](https://docs.aws.amazon.com/autoscaling/ec2/userguide/lifecycle-hooks.html)
- [LifecycleHook — EC2 Auto Scaling API Reference](https://docs.aws.amazon.com/autoscaling/ec2/APIReference/API_LifecycleHook.html)
- [AWS::AutoScaling::LifecycleHook — CloudFormation](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-resource-autoscaling-lifecyclehook.html)
- [Delay the termination of unhealthy EC2 instances — AWS re:Post](https://repost.aws/knowledge-center/auto-scaling-delay-termination)