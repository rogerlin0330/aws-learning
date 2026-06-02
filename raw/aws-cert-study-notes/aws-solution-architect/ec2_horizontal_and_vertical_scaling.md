# Horizontal vs. Vertical Scaling

## Core Concept

**Scaling** is how a system grows or shrinks in response to changes in load placed on it by customers. Technically, it means **adding or removing resources** from a system.

A "system" can be:

- A single compute device (e.g., one EC2 instance), or
- Hundreds, thousands, or even hundreds of thousands of devices working together.

There are two fundamentally different ways to scale: **vertical** (bigger box) and **horizontal** (more boxes).

---

## Vertical Scaling — "Bigger Server"

You increase capacity by **resizing the individual instance** to a larger size.

**Example:** An app on a `T3.large` (2 vCPU / 8 GiB) starts to struggle under load → resize to `T3.xlarge` (4 vCPU / 16 GiB) → if needed, `T3.2xlarge` (8 vCPU / 32 GiB). Each step roughly doubles the resources.

### Pros

|Benefit|Why it matters|
|---|---|
|**Simple**|No special architecture required.|
|**No application modification**|If an app runs on an instance, it runs on a _bigger_ instance.|
|**Works for any app — including monoliths**|The entire codebase runs on one instance, so a single larger instance scales it cleanly.|

### Cons

|Limitation|Why it matters|
|---|---|
|**Downtime during resize**|Resizing an instance usually requires a **restart**, causing customer disruption.|
|**Outage windows only**|Because of disruption, you can typically only scale during **pre-agreed maintenance windows** — slow to react to rapidly changing load.|
|**Non-linear cost**|Larger instances carry a **price premium**; cost increases faster than capacity at the top end.|
|**Hard upper cap**|There is always a **maximum instance size**. Vertical scaling can never exceed the largest available instance.|

> 🎯 **Exam tip:** Vertical scaling = resize the instance = **restart/downtime** = scale only in outage windows. The keyword association to remember is _single instance has a size cap_.

---

## Horizontal Scaling — "More Servers"

Instead of growing one instance, you **add more instances** (1 → 2 → 8 → …), each typically a smaller, commodity-sized instance. This is sometimes called _scaling out_ (and _scaling in_ when removing).

Because you now have **many running copies** of the application, they must share the load — which requires a **load balancer**.

### The Load Balancer

A load balancer is an appliance that sits **between customers and your instances**. It distributes incoming requests fairly across all available instances. For a single customer, consecutive interactions (each click) may land on the **same instance or a different one** at random.

### ⚠️ The Critical Consideration: Sessions

> **"With horizontal scaling, sessions are everything."**

A **session** is the state of a customer's interaction with an app (login status, video position, shopping cart contents, etc.).

- On a **single server**, sessions live on that server — fine.
- With **horizontal scaling**, a customer constantly shifts between instances. If sessions are stored _on the instance_, then moving from Instance 1 to Instance 10 means a **different session or no session** → the user gets logged out → app is **unusable**.

**The fix — off-host sessions:** Store session data **externally** (e.g., in a separate database). This makes the instances **stateless** ("dumb" copies of the app). It no longer matters which instance a customer hits — their state is always retrievable externally.

> This requires **deliberate design / application support**. It is the price of admission for horizontal scaling — but once met, you unlock all its benefits.

### Pros (once sessions are off-host)

|Benefit|Why it matters|
|---|---|
|**No disruption while scaling**|Adding instances doesn't touch existing ones — current customer connections are unaffected.|
|**No disruption while scaling _in_**|Because sessions are external, connections can be moved off a removed instance without logging anyone out.|
|**No real upper limit**|You use lots of small commodity instances and just keep adding — no single-instance size cap.|
|**Often cheaper**|Smaller commodity instances avoid the premium charged for large instances.|
|**Granular scaling**|Adding 1 instance to 5 = ~20% increase. (Compare to vertical, where large → xlarge ≈ doubling.) The smaller the instances, the finer the granularity.|

---

## Side-by-Side Summary

|Dimension|Vertical Scaling|Horizontal Scaling|
|---|---|---|
|**Method**|Resize one instance (bigger)|Add more instances|
|**Disruption**|Restart / downtime|None (add or remove freely)|
|**When you can scale**|Outage windows only|Anytime|
|**Upper limit**|Capped at max instance size|Effectively unlimited|
|**Cost profile**|Premium at large sizes (non-linear)|Cheaper commodity instances|
|**Granularity**|Coarse (steps ≈ doubling)|Fine (one small instance at a time)|
|**App requirements**|None — works for monoliths|Needs load balancer + off-host/stateless sessions|
|**Complexity**|Simple|Requires design effort|

---

## 🧠 Exam Power-Up: The "Bob" Mnemonic

When you blank on which is which in the exam, picture the course's guest star, Bob:

- **Horizontal scaling = add/remove Bobs.** One Bob → two Bob → four Bob → near-**infinite** Bob. (Adds _more things_, no real limit.)
- **Vertical scaling = one bigger Bob.** Small Bob → medium Bob → large Bob. (One thing _grows_, until it can't.)

**Many Bobs = horizontal. One giant Bob = vertical.**

---

## Looking Ahead

Horizontal scaling is later combined with **elasticity** in the High Availability & Scaling section to build **highly available and fault-tolerant** architectures.