# EC2 Architecture and Resilience — Study Note

## 1. What EC2 Is (High-Level)

- EC2 instances are **virtual machines**: an OS plus allocated vCPU, memory, optional local storage, optional network storage, and access to networking/GPU hardware.
- Instances run on **EC2 hosts**, which are physical servers managed by AWS.
- EC2 is the **default compute service** in AWS — start with it, and only move to alternatives (Lambda, ECS, etc.) if a specific requirement pushes you away.

---

## 2. Hosts vs. Instances — The Key Distinction

This is the conceptual core of the lesson. Hosts and instances are easily confused, but they operate at different layers.

|Aspect|**EC2 Host**|**EC2 Instance**|
|---|---|---|
|**What it is**|Physical server hardware managed by AWS|Virtual machine running on a host|
|**Who owns/manages**|AWS owns and maintains the hardware|Customer configures and uses the VM|
|**Layer**|Physical / hypervisor layer|Virtualization layer (guest OS)|
|**Tenancy options**|**Shared** (default — multiple AWS customers share the hardware, but are isolated from each other) **or Dedicated** (entire host reserved to one AWS account)|Always belongs to one customer; runs on either a shared or dedicated host|
|**Billing model**|On a **dedicated host**, you pay for the _host itself_ regardless of instances on it|On a **shared host**, you pay per instance based on runtime and allocated resources|
|**Local resources**|Provides CPU, memory, **instance store** (temporary local storage), storage networking, data networking|Consumes a slice of the host's resources; can use the instance store while on that host|
|**Lifecycle**|Long-lived physical hardware; can fail or be taken down by AWS for maintenance|Created, stopped, started, terminated by the customer|
|**Location**|Lives in **one specific Availability Zone**|Lives in the **same AZ as its host**; cannot natively cross AZs|
|**Movement**|Doesn't move|Stays on its host through restarts; **moves to a new host only if (a) the host fails/is taken down, or (b) the instance is stopped and then started** — and the new host is still in the same AZ|
|**Sharing**|A single host typically runs many instances from many customers (when shared)|Multiple instances can co-exist on one host; usually same type/generation, different sizes|

### Quick mental model

Think of a **host** as an apartment building AWS owns, and an **instance** as a tenant's apartment unit. A restart is the tenant going to sleep and waking up in the same apartment. A stop/start is the tenant moving out and being re-housed in a different unit — but always in the same neighborhood (AZ).

---

## 3. Availability Zone Resilience (The Most-Repeated Point)

EC2 is an **AZ-resilient service**, not a region-resilient one. Almost every component is locked to a single AZ:

- **Hosts** run inside one AZ.
- **Subnets** belong to one AZ; an instance's primary elastic network interface (ENI) is provisioned in a subnet and maps to host hardware.
- An instance can have **multiple network interfaces**, even in different subnets — but only if those subnets are in the **same AZ**.
- **EBS (Elastic Block Store)** also runs per-AZ. Volumes can only be attached to instances in the **same AZ**.
- You **cannot** attach a network interface or EBS volume from one AZ to an instance in a different AZ.

**Consequence:** if an AZ fails, assume all instances, ENIs, and EBS volumes in that AZ are impacted.

### Moving between AZs

Instances cannot natively move across AZs. The only way is to take a **snapshot / AMI** of the instance and launch a new one in another AZ (covered in a later lesson).

### Why this is a strength, not a limitation

Because the "blast radius" of a failure is always contained to one AZ, solutions architects can design highly available systems by deploying across multiple AZs and load-balancing between them. Predictable isolation = controllable design.

---

## 4. Instance Behavior on a Host

- An instance stays on its host through **restarts**.
- It moves to a new host (same AZ) when:
    1. The host fails or AWS takes it down for maintenance.
    2. The instance is **stopped and then started** (not the same as a restart).
- **Instance store** local storage is _temporary_ — if the instance moves to a different host, the contents of the instance store are lost.
- Instances of **different sizes** can share a host; generally instances of the **same type and generation** end up co-located on hosts built for that generation's CPU/memory/storage profile.

---

## 5. When to Use EC2

EC2 is the right default for:

- **Traditional OS + application** workloads needing a specific OS, runtime, or vendor-supported configuration.
- **Long-running compute** (24/7/365) — unlike many AWS compute services, EC2 has no runtime limit.
- **Server-style applications** that wait for incoming connections inside an OS.
- **Burst or steady-state workloads** — different instance types cover both patterns.
- **Monolithic application stacks** (app + database + middleware on one OS).
- **Workload migrations** from on-prem VMs.
- **Disaster recovery** environments for traditional server-based systems.

Move away from EC2 only when a niche requirement (event-driven, containerized microservices, fully managed runtime, etc.) clearly favors Lambda, ECS, or another service.

---

## 6. Exam Tips

- If a question mentions EC2, **find the AZ in the scenario** — it almost always matters.
- Remember: **host = physical, AZ-bound; instance = virtual, AZ-bound through its host**.
- **Stop/start ≠ restart.** Stop/start can move an instance to a new host; restart cannot.
- **Shared host = default**, customers isolated. **Dedicated host = you pay for the whole host**.
- **EBS and ENIs are AZ-scoped** — no cross-AZ attachment.
- Snapshots/AMIs are the mechanism for moving workloads across AZs (or regions).
- EC2 is the **default compute choice** — justify deviations.