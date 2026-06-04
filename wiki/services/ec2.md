---
title: EC2 — Elastic Compute Cloud
type: service
tags: [ec2, compute, iaas, ami, ami-baking, ami-permissions, ami-marketplace, ami-lifecycle, block-device-mapping, ssh, rdp, instance-lifecycle, shared-responsibility, ebs, key-pair, host, tenancy, dedicated-host, dedicated-instance, instance-store, az-resilience, restart-vs-stop-start, instance-types, general-purpose, compute-optimized, memory-optimized, accelerated-computing, storage-optimized, graviton, burstable, gpu, fpga, eni, elastic-network-interface, primary-eni, secondary-eni, mac-address, private-dns, public-dns, split-horizon-dns, elastic-ip, eip, ipv6, source-destination-check, security-group, imds, instance-metadata, imdsv2, link-local, user-data, ssrf, status-checks, system-status-check, instance-status-check, auto-recovery, cloudwatch-alarm, sns]
certs: [aws-saa]
updated: 2026-06-03
---

# EC2 — Elastic Compute Cloud

EC2 is AWS's default and most essential compute service. It provides virtual machines — **instances** — running on AWS-managed physical servers called **hosts**, and is the starting point for almost any compute requirement in AWS. Every EC2 instance runs inside a [VPC](vpc.md) subnet and is associated with at least one [Elastic Network Interface (ENI)](#network-interfaces-enis) — the per-interface attachment that holds the instance's IPs, DNS names, MAC address, and Security Groups.

This page covers the foundations — what an instance is, the host vs. instance distinction and its AZ implications, how it's billed, the lifecycle state machine, AMIs, when to choose EC2, and how you connect to running instances. Deeper topics (instance types, EBS volumes, Auto Scaling, ELB) will follow as separate sections or pages.

## EC2 at a Glance

![EC2 overview infographic showing the four pillars: core IaaS concepts, instance lifecycle and cost states, AMIs and networking, availability and access](../assets/ec2-basics-infographic.png)

Four pillars structure everything that follows: **core concepts (IaaS)**, **instance lifecycle & costs**, **AMIs & networking**, and **availability & access**.

## Foundations

EC2 is **Infrastructure as a Service (IaaS)** — AWS gives you a virtualized OS in a virtualized network; you bring everything from the OS up.

- **Unit of consumption:** the **instance** — an operating system configured with allocated CPU, memory, storage, and networking
- **Reach for EC2 when** you need an OS, a runtime environment, database dependencies, applications, or application interfaces you want to manage yourself
- **Private by default** — every instance launches into one specific [VPC](vpc.md) subnet in the private AWS zone. Public reachability is opt-in and explicit (see [How EC2 Fits Into the AWS Architecture](#how-ec2-fits-into-the-aws-architecture) below). The **default VPC** wires this up automatically; **custom VPCs** require manual configuration ([IGW](vpc.md#internet-gateway-igw), public subnet, route, public IP / EIP)
- **AZ resilient, not AZ fault-tolerant** — because a subnet lives in a single [Availability Zone](../concepts/networking.md), and an instance lives in a subnet, **if the AZ fails, the instance fails**. HA requires running across multiple AZs (a topic for the patterns layer, not a property of a single instance)
- **Customizable at launch** — CPU, memory, optional GPUs, advanced storage and networking, special-purpose processors. Some properties can be changed after launch (e.g. instance type via stop → modify → start); others are fixed at launch (e.g. AZ, AMI lineage)

> [!tip] Mental model
> Resilience tier in AWS is always determined by the **smallest scope you depend on**. An EC2 instance depends on a subnet, which depends on an AZ — so the instance inherits AZ scope. To climb the resilience ladder (regional), you don't make one instance more resilient; you run a **fleet across multiple AZs** and put a load balancer in front. See [Overview — Resilience Tiers](../overview.md#resilience-tiers).

### Shared responsibility (IaaS line)

EC2 sits on the **IaaS** side of the shared responsibility model. AWS is responsible for the virtualization layer and below; you are responsible for the OS and above.

| You manage | AWS manages |
| --- | --- |
| Operating system (patches, hardening) | Hypervisor / virtualization |
| Runtime, applications, dependencies | Physical hardware |
| Data on the volumes | Physical networking |
| OS-level configuration (users, services, firewall inside the guest) | Storage infrastructure (EBS service, durability) |
| In-guest security | Facilities (data centres, power, cooling) |

> [!warning] Gotcha
> "AWS patches my EC2 instance" is **false**. AWS patches the hypervisor; the guest OS, security updates, and middleware are entirely on you. Managed compute (Lambda, Fargate) is what shifts that line upward — EC2 does not.

## How EC2 Fits Into the AWS Architecture

- **Networking:** instances live in [VPC](vpc.md) subnets; traffic is controlled by [Security Groups](vpc.md#security-groups-sg) (attached to the [ENI](#network-interfaces-enis)) and [NACLs](vpc.md#network-acl-nacl) (attached to the subnet)
- **Identity:** instances access other AWS services via [IAM roles](iam.md#iam-roles) — never embed access keys directly in an instance. Temporary role credentials are delivered to the instance through the [Instance Metadata Service (IMDS)](#instance-metadata-service-imds)
- **Internet access:** public instances route through an [IGW](vpc.md#internet-gateway-igw); private instances route through a [NAT Gateway](vpc.md#nat--nat-gateway)
- **Storage:** the boot volume is typically [EBS](#storage-options) (a separate AWS service exposing network-attached block volumes); some instance types also offer **instance store** volumes physically on the host

## Hosts vs Instances

EC2 separates the **physical layer** (hosts) from the **virtual layer** (instances). The distinction is the conceptual core of the service — easy to gloss over and frequently tested.

| Aspect | **EC2 Host** | **EC2 Instance** |
| --- | --- | --- |
| What it is | Physical server hardware owned and maintained by AWS | Virtual machine running on a host |
| Manages | AWS (hardware, firmware, hypervisor) | Customer (OS and above — see [Shared responsibility](#shared-responsibility-iaas-line)) |
| Tenancy options | **Shared** (default — multiple AWS customers' instances coexist, hypervisor-isolated) **or Dedicated** (entire host reserved to one account) | Always single-tenant *inside* the OS; runs on a shared or dedicated host |
| Billing model | Billed per host (full hardware) on a **Dedicated Host**; transparent under Shared tenancy | Per-instance, based on runtime + allocated resources — see [Billing Model](#billing-model) |
| Local resources | Provides CPU, memory, **instance store** (host-local NVMe/SSD), storage networking, data networking | Consumes a slice of the host's resources |
| Lifecycle | Long-lived hardware; AWS retires, repairs, or takes down for maintenance | Customer creates, stops, starts, terminates |
| Location | One specific [Availability Zone](../concepts/networking.md) | Same AZ as its host — cannot natively cross AZs |
| Movement | Doesn't move | Stays on its host through restarts; **moves to a new host on stop-then-start, or if the host fails / is retired** — always within the same AZ |
| Co-tenants | Typically many instances from many customers (Shared), or many of your own instances (Dedicated) | Multiple instances coexist; same instance family/generation usually co-locates on hardware built for that profile |

> [!tip] Mental model
> A **host** is an apartment building AWS owns; an **instance** is a tenant's apartment unit. **Restart** = the tenant sleeping and waking up in the same apartment. **Stop/Start** = moving out and being re-housed — usually in a different unit, always in the same neighbourhood (AZ).

### Tenancy: Shared vs Dedicated

| Mode | Hardware sharing | Billing | Typical use case |
| --- | --- | --- | --- |
| **Shared** (default) | Many customers' instances coexist on the same host, isolated by the hypervisor | Per-instance (see [Billing Model](#billing-model)) | Almost everything |
| **Dedicated Instance** | Hardware not shared with other customers, but you cannot address or target the host itself | Per-instance, with a small dedicated-tenancy premium | Compliance scenarios that forbid hardware co-tenancy but don't require host targeting |
| **Dedicated Host** | One account allocates the entire host; you can see and target it | **Per host**, regardless of how many instances run on it | Hardware-bound BYOL licensing (per-socket / per-core Windows, Oracle, SQL Server), strict isolation, host-affinity workloads |

The full purchase-option treatment — including how the three tenancy modes interact with On-Demand Instances, Spot Instances, Reserved Instances, On-Demand Capacity Reservations, and Savings Plans — lives on [EC2 Purchase Options](../patterns/ec2-purchase-options.md).

> [!warning] Gotcha — "Dedicated" is two things
> AWS uses *Dedicated* in two adjacent ways: **Dedicated Instance** (no hardware co-tenancy, host opaque) vs **Dedicated Host** (you allocate and target the host, pay per host). They look similar in question stems — the Dedicated Host adds **per-socket licensing**, **host affinity**, and **per-host billing**.

### AZ Resilience — everything is AZ-bound

Almost every component an instance depends on is locked to a single AZ:

- **Host** lives in one AZ
- **Subnet** belongs to one AZ ([VPC](vpc.md) primitive)
- The instance's primary [ENI](#network-interfaces-enis) is provisioned in that subnet
- **Additional ENIs** can attach in *different subnets*, but **only if those subnets are in the same AZ**
- **[EBS](#storage-options) volumes** are AZ-scoped — they can only attach to instances in the **same AZ**
- **Instance store** is host-local, so by definition AZ-scoped

**Consequence:** if an AZ fails, **assume all instances, ENIs, and EBS volumes in that AZ are impacted**. You cannot attach an EBS volume or ENI from one AZ to an instance in another AZ to mitigate.

The only path to relocate a workload across AZs is to **snapshot the EBS volume(s) or capture an [AMI](#ami--amazon-machine-image), then launch a new instance in the target AZ** — see [Moving an instance across AZs](#moving-an-instance-across-azs) below.

> [!tip] Why this is a strength, not a limitation
> Because the blast radius of a failure is always one AZ, you can design AZ-fault-tolerance predictably: deploy across multiple AZs, put a load balancer in front, fail over deterministically. **Predictable isolation = controllable design.** See [Overview — Resilience Tiers](../overview.md#resilience-tiers).

## Network Interfaces (ENIs)

Every EC2 instance reaches the network through one or more **Elastic Network Interfaces (ENIs)**. The ENI — *not the instance* — is what holds the IP addresses, DNS names, MAC address, [Security Groups](vpc.md#security-groups-sg), and the source/destination check flag. When the console shows IPs and SGs "on the instance," they're really attached to its **primary ENI**. See [Networking Concepts — ENI](../concepts/networking.md#elastic-network-interfaces-enis) for the cross-service framing.

![EC2 network architecture diagram: AZ-A boundary containing an instance (chip icon) connected to a Primary ENI and a Secondary ENI; a red arrow leads from the Primary ENI to a vertical attribute list — MAC address 00:0d:83:b1:c0:8e, Primary IPv4 Private IP 10.16.0.10 mapped to private DNS ip-10-16-0-10.ec2.internal, 0 or more secondary IPs, 0 or 1 Public IPv4 Address 3.89.7.136 mapped to public DNS ec2-3-89-7-136.compute-1.amazonaws.com with the split-horizon callout "VPC = Private IP / Elsewhere = Public IP", 1 Elastic IP per private IPv4 with "Removes the Public IPv4 / Replaces with the Elastic IP" callout, 0 or more IPv6 addresses, Security Groups, Source/Destination Check; the Secondary ENI is labelled "As Above" meaning it carries the same attribute set](../assets/ec2-network-interface-and-dns-architecture.png)

### Primary vs Secondary ENIs

| Aspect | **Primary ENI** | **Secondary ENI** |
| --- | --- | --- |
| Count per instance | **Exactly 1** — always | **0 or more** (max depends on instance type) |
| Created with the instance? | Yes — at launch | At launch *or* attached later |
| Detach from the instance? | **No** — bound for life | **Yes** — can be detached and re-attached to another instance |
| Subnet | The instance's launch subnet | **Any subnet in the same AZ** (can differ from primary) |
| Attribute set | MAC + IPv4 primary + secondaries + optional public IPv4/DNS + EIPs + IPv6 + SGs + source/dest check | Same set as primary |

**Key consequence: ENIs are AZ-scoped.** An instance and all of its ENIs must live in the **same Availability Zone**, even though they can sit in different subnets. There is no cross-AZ ENI. See [AZ Resilience — everything is AZ-bound](#az-resilience--everything-is-az-bound).

> [!tip] Mental model
> The ENI is the "network card"; the instance is the chassis. Moving an ENI between instances is like unplugging a NIC and plugging it into a different box — its MAC, private/public IPs, EIP, DNS names, and SGs all travel with it. The *primary* NIC is soldered to the board; secondary NICs are removable.

### What Lives on an ENI

Every attribute below is per-ENI. **Per-instance** is a console abstraction — the underlying data sits on an ENI.

#### MAC address

A fixed hardware-style address visible inside the guest OS. Because it doesn't change for the life of the ENI, **MAC-locked software licensing** is a recognised use case for ENIs — bind the licence to a secondary ENI's MAC, then re-attach the ENI to a replacement instance to preserve the licence.

#### Primary private IPv4 + internal DNS

Every ENI has exactly one **primary private IPv4** — static for the life of the ENI — plus a matching **internal DNS name** that AWS derives mechanically from that IP:

```text
private IPv4   10.16.0.10
                       ↓
internal DNS    ip-10-16-0-10.ec2.internal
```

- Format: `ip-` + the private IPv4 with dots → hyphens + `.ec2.internal`
- **Resolvable only inside the VPC** (via the [Route 53 Resolver](vpc.md#dns-in-vpc) at VPC base IP +2)
- Always resolves to the private IPv4 — use it for internal service-to-service traffic
- Choosing a VPC and subnet for the instance is really choosing them for the **primary ENI**

#### 0 or more secondary private IPv4s

Any ENI can carry additional secondary private IPv4 addresses. Useful for hosting multiple TLS sites with separate IPs on one instance, or for IP-affinity failover patterns where a secondary IP "floats" between instances.

#### 0 or 1 public IPv4 + public DNS

An ENI can also have a single **auto-assigned public IPv4** (only if the subnet auto-assigns or the launch explicitly opts in). It comes with its own mechanically-derived public DNS name:

```text
public IPv4    3.89.7.136
                       ↓
public DNS      ec2-3-89-7-136.compute-1.amazonaws.com
```

- Format: `ec2-` + the public IPv4 with dots → hyphens + `.compute-1.amazonaws.com`
- **The OS never sees the public IPv4** — it's applied by 1:1 NAT at the [IGW](vpc.md#internet-gateway-igw). Inside the guest you only configure the **private IPv4**. The only way for code running on the instance to discover its own public IPv4 is to query the [Instance Metadata Service (IMDS)](#instance-metadata-service-imds)
- **Dynamic**: deallocated on **Stop**, a **new one** assigned on **Start**. A **reboot preserves it** (the instance never leaves its host — see [Stop/Start vs Restart](#stopstart-vs-restart--these-are-not-the-same)). A forced host migration replaces it
- The trap behind *"my public IP changed after I rebooted"* — the action was almost always a **Stop/Start in disguise**

#### Split-horizon DNS — the public DNS name resolves differently inside vs outside the VPC

The public DNS name resolves **differently depending on where the query originates**:

| Resolver location | The public DNS name resolves to |
| --- | --- |
| Inside the VPC | The instance's **primary private IPv4** |
| Outside the VPC (general internet) | The instance's **public IPv4** |

A single hostname works everywhere, but **instance-to-instance traffic inside the VPC stays private** — it never hairpins out to the IGW and back. This matters for [VPC peering](vpc.md), hybrid networking, and cost (private cross-AZ traffic vs internet egress). See [DNS — Split-Horizon DNS](../concepts/dns.md#split-horizon-dns) for the generic concept.

> [!tip] Exam Tip
> When a scenario combines *"connect via the public DNS name"* with *"VPC peering"* or *"hybrid network"*, split-horizon DNS is the mechanism that keeps traffic on the AWS backbone — the name resolves to the **private IP** for callers inside the VPC.

#### Elastic IP (EIP) — 1 per private IPv4

An ENI can have up to **one Elastic IP per private IPv4** it holds. An EIP is a **static public IPv4** allocated to your AWS account and explicitly associated with a private IPv4 on an ENI.

Attaching an EIP to an ENI has two side effects:

1. The ENI's current auto-assigned dynamic public IPv4 is **removed**
2. The EIP becomes the new public IPv4 (it does not stack — it replaces)

If you later **disassociate** the EIP from the ENI:

- The instance receives a **brand-new dynamic public IPv4** — *not* the original one. The original is gone for good
- The EIP itself stays allocated to your account (and bills an idle hourly fee — see [EC2 state-based costs](#what-changes-when-the-state-changes)) until you explicitly **release** it

> [!warning] Gotcha — disassociating an EIP doesn't restore the original public IP
> Attaching and then detaching an EIP is a **one-way swap**: the ENI's original dynamic public IPv4 is permanently lost, and on detach you get a *different* dynamic public IPv4. If you need a stable address, keep the EIP attached — don't detach it expecting the old IP back.

#### 0 or more IPv6 addresses

IPv6 on ENIs behaves differently from IPv4 in two ways that surface on the exam:

- **No public/private split** — all IPv6 addresses in AWS are publicly routable
- **The OS does see IPv6** directly — addresses are configured inside the guest, unlike public IPv4

For outbound-only IPv6 from a private subnet, use an [Egress-Only IGW](../concepts/networking.md#internet-connectivity-patterns); the [NAT Gateway](vpc.md#nat--nat-gateway) does not handle IPv6.

#### Security Groups

[Security Groups](vpc.md#security-groups-sg) attach to the **ENI**, not the instance. One SG (or up to 5) governs **every IP on that ENI**. If you need different SGs for different IPs on the same instance, the only way is to split those IPs across **multiple ENIs**.

#### Source/Destination Check

A per-ENI boolean flag. When **enabled (the default)**, AWS discards any packet whose source *and* destination IPs are neither owned by the ENI — the ENI cannot forward traffic on behalf of others. You **must disable** this flag for an instance to act as a [NAT instance](vpc.md#nat-gw-vs-nat-instance), a software router, or any in-line gateway.

### The Movable-Secondary-ENI Pattern

Because a **secondary ENI can be detached and re-attached to another instance**, and because its MAC and IPs travel with it, you can:

- **Run MAC-locked software licenses on a movable ENI** — preserve the license by re-attaching the ENI to a replacement instance instead of issuing a new license
- **Build IP-affinity failover** — float a "service IP" (private IPv4 + EIP) between a primary and a standby instance by re-attaching the ENI, without DNS propagation delays
- **Multi-homed instances** — give one instance two ENIs in different subnets for tier-bridging (e.g. one ENI in a management subnet, another in a workload subnet)

This is the practical answer to "how do I keep the same IP / license / SG profile when the instance dies?" — *put it on a secondary ENI, not the primary*.

### ENI Attributes — Quick Reference

| Attribute | Per-ENI count | Notes |
| --- | --- | --- |
| MAC address | 1 | Static for life of the ENI; visible inside the OS |
| Primary private IPv4 | 1 | Static for life of the ENI; derives the internal DNS name |
| Secondary private IPv4 | 0+ | Add/remove post-launch |
| Public IPv4 (auto-assigned) | 0 or 1 | Dynamic; changes on Stop/Start; **not visible inside the OS** |
| Public DNS name | 0 or 1 | Mechanically derived; **split-horizon** (private inside VPC, public outside) |
| Elastic IP | 0+ (≤ 1 per private IPv4) | Static; allocated to account; replaces the auto public IPv4 on attach |
| IPv6 | 0+ | All publicly routable; **OS-visible** |
| Security Groups | 1–5 | One SG set per ENI; applies to every IP on that ENI |
| Source/Destination Check | 1 (boolean) | Default **on**; must be **off** for NAT instances |

## Billing Model

EC2 uses **on-demand billing** — you pay only for what you consume — with three independently-billed components:

1. **Compute charge** — CPU + memory, while the instance is **running**
2. **Storage charge** — disks attached to the instance (notably [EBS](#storage-options)), **regardless of whether the instance is running or stopped**
3. **Extras** — commercial software licensing (Windows, RHEL, etc.) bundled into the rate when applicable

Billing granularity is **per second** or **per hour**, depending on the software running on the instance (Linux/most Windows AMIs bill per second with a 60-second minimum; some commercial licenses bill per hour).

For multi-year predictable workloads, EC2 is the canonical target of the [Cost Optimization](../patterns/cost-optimization.md) commitment ladder — Compute Savings Plans, EC2 Instance Savings Plans, Reserved Instances, and Spot Instances. See [EC2 Purchase Options](../patterns/ec2-purchase-options.md) for the full set (On-Demand Instances, Spot Instances, Standard & Scheduled Reserved Instances, On-Demand Capacity Reservations, Savings Plans, Dedicated Instances, Dedicated Hosts) and the discount/capacity decision matrix.

### Storage options

For the general storage vocabulary (direct-attached vs network-attached, ephemeral vs persistent, block vs file vs object, IOPS / throughput / block size), see [Storage Fundamentals](../concepts/storage.md).

- **Instance store (local host storage)** — ephemeral block volumes physically on the EC2 host. Fast, free with the instance, **lost when the instance moves to a new host** (stop/start, resize, host failure). A reboot preserves data. Available only on certain instance types. See [Instance Store](#instance-store) below for full details
- **EBS (Elastic Block Store)** — network-attached block storage, a separate AWS service. Volumes persist independently of the instance and survive stop/start. Charged by provisioned size (and IOPS/throughput on some types), not by usage. EBS is the source of the "stopped instance still costs money" trap below. See [EBS](../services/ebs.md) for volume type details (GP2 vs GP3, Provisioned IOPS, HDD types)

## Instance Lifecycle and State-Based Costs

Three states matter most: **Running**, **Stopped**, and **Terminated**.

```text
   ┌──────────┐    Stop      ┌──────────┐   Terminate    ┌────────────┐
   │ RUNNING  │ ────────────►│ STOPPED  │ ──────────────►│ TERMINATED │
   │          │◄──────────── │          │                │ (one-way!) │
   └──────────┘    Start     └──────────┘                └────────────┘
```

Intermediate states exist (`pending`, `stopping`, `shutting-down`) but rarely appear on the exam. **Stop ↔ Start** is bidirectional; **Terminate** is a one-way arrow.

### What changes when the state changes

| Component | Running | Stopped | Terminated |
| --- | :-: | :-: | :-: |
| CPU | 💰 Charged | ✅ Free | ✅ Free |
| Memory | 💰 Charged | ✅ Free | ✅ Free |
| Networking | 💰 Charged | ✅ Free | ✅ Free |
| [EBS](#storage-options) storage | 💰 Charged | 💰 **Still charged** | ✅ Free (volumes deleted) |
| Public IPv4 (auto-assigned) | Assigned | Released | Released |
| Instance store data | Present | **Lost** | Lost |
| Elastic IP (if attached) | Free while attached to running instance | **Small hourly charge** while detached/idle | Charge continues until released |

> [!warning] Gotcha — "Stopped is not free"
> A **stopped** instance still incurs **EBS storage charges**. The disks remain allocated and billable; the only way to eliminate all costs from an instance is to **terminate** it (which deletes its EBS volumes by default — see termination protection / `DeleteOnTermination` for nuance). "We stopped it overnight to save money" only saves the compute portion; the storage line item keeps ticking.

> [!warning] Gotcha — "Terminate is permanent"
> Termination is **non-reversible**. Root EBS volumes are deleted by default; instance-store data is lost; the instance ID is gone. There is no undelete. If the data matters, snapshot the EBS volume(s) **before** terminating, or detach them first and re-attach to a new instance.

### Stop/Start vs Restart — these are not the same

Both preserve the instance ID, the EBS root-volume contents, the private IPv4, and the security configuration. The difference is **what happens to the underlying host** — which is the source of several exam traps.

| Operation | Host | Instance store | Auto-assigned public IPv4 | EBS bill |
| --- | --- | --- | --- | --- |
| **Restart (reboot)** | **Same host** — quick OS reboot, no host swap | Preserved | Preserved (same IP) | No change |
| **Stop → Start** | **May land on a new host** in the same AZ | **Lost** (host changed) | **Released; new IP on start** | Same |

> [!warning] Gotcha — "I rebooted and lost my instance-store data"
> A **restart** preserves instance-store contents because the instance never leaves its host. A **stop/start** does *not*, because the instance can land on a new host — the old host's local disks belong to whoever lands there next. If a scenario describes losing scratch data after a "reboot", check whether the action was really a **stop/start** in disguise.

> [!warning] Gotcha — "Why did my public IP change?"
> The auto-assigned public IPv4 is released on **Stop** and a new one is assigned on **Start**. A plain **Restart** keeps the same public IP. If a stable public address is required across stop/start cycles, attach an **Elastic IP** (and remember it bills hourly while not attached to a running instance — see the cost table above).

> [!tip] Auto recovery is the third sibling
> [Auto recovery](#ec2-auto-recovery) can also land the instance on a new host in the same AZ, but it **preserves all IP addressing** (auto public IPv4 included) where Stop/Start does not. See [Auto Recovery vs Stop/Start vs Restart](#auto-recovery-vs-stopstart-vs-restart) for the side-by-side.

### Moving an instance across AZs

Instances **cannot natively cross AZ boundaries** — they're tied to their host, which is tied to its AZ. To relocate:

1. Stop the instance, or take a live [EBS](#storage-options) snapshot
2. Capture an [AMI](#ami--amazon-machine-image) from the instance, or snapshot the EBS volume(s) directly
3. Launch a new instance in the target AZ from the AMI / from a new volume created from the snapshot in the target AZ
4. Update any DNS, load balancer, or [Security Group](vpc.md#security-groups-sg) references to point at the new instance

This is the only path. For multi-AZ high availability the architectural answer is not "make one instance cross AZs"; it is "**run a fleet across multiple AZs and load-balance**" — see [Overview — Resilience Tiers](../overview.md#resilience-tiers).

## Status Checks and Auto Recovery

EC2 continuously monitors every instance with **two independent status checks**. Each watches a different layer of the stack, so the check that fails tells you where the problem lives — the EC2 service and host underneath, or the operating system inside the instance. A healthy, fully booted instance shows **2/2 checks passed**; anything else (after boot completes) is a problem to resolve. During boot you may briefly see `initializing` or `1/2 passed`.

### The Two Status Checks

| Check | Also shown as | What it watches | Typical causes of failure |
| --- | --- | --- | --- |
| **System status** | System Reliability Check | The EC2 **service / host** underneath the instance | Loss of system power, lost network connectivity, hardware or software fault on the EC2 host |
| **Instance status** | Instance Reachability Check | The **instance itself** (OS layer) | Corrupt file system, misconfigured networking inside the OS, kernel issues preventing proper startup |

> [!tip] Mental model
> System check = *"Is the building's power and plumbing working?"* Instance check = *"Is the problem inside my own apartment?"* The check that fails identifies whose problem it is — AWS's (system) or yours (instance).

> [!warning] Gotcha — the static public IPv4 trap
> A classic instance-check failure is statically setting a **public IPv4 address on the OS's internal interface**. This never works — the public IP is mapped externally by 1:1 NAT at the [IGW](vpc.md#internet-gateway-igw) and is never configured inside the guest OS. [IMDS](#instance-metadata-service-imds) is the only path for the OS to even *see* its public IPv4.

### Responding to a Failed Check

You can respond manually or set up an automated response.

**Manual responses:** stop and start the instance, restart (reboot), or terminate and re-launch.

**Automated response — the Status Check Alarm.** Create a [CloudWatch](cloudwatch.md) alarm via *Actions → Create Status Check Alarm* in the EC2 console. The alarm fires when **either** status check fails for **at least one consecutive 5-minute period**, publishes a notification to an **SNS** topic, and optionally takes one EC2 action:

| Action | What it does | When it's useful |
| --- | --- | --- |
| **Reboot** | OS-level reboot — same host | Quick fix for transient instance-layer issues |
| **Stop** | Stops the instance | Lets you preserve a "frozen" state for diagnostics — see [Stop/Start vs Restart](#stopstart-vs-restart--these-are-not-the-same) for what changes |
| **Terminate** | Destroys the instance | Pairs with an HA setup — an [Auto Scaling group](../patterns/scaling.md) re-provisions a replacement. On a lone instance, the instance is simply gone — nothing replaces it |
| **Recover** | Triggers **auto recovery** (next section) | Recover from an isolated host/instance failure with the configuration intact |

### EC2 Auto Recovery

Auto recovery is the "headline" automated response. It attempts to bring a failed instance back to life by taking a graduated set of actions — anywhere from a **simple restart** to **migrating the instance to a brand-new EC2 host in the same AZ**.

**What's preserved:** the instance comes up with **exactly the same configuration** — including all **IP addressing** (primary private IPv4, the auto-assigned public IPv4 if it had one, and any [Elastic IP](#elastic-ip-eip--1-per-private-ipv4)). If the software inside the instance is set to auto-start, recovery can run **fully unattended** — hence the name. From the caller's point of view, the instance simply heals.

**Purpose:** lightweight automation to handle a narrow set of **isolated host or instance failures** and "avoid waking up a sysadmin". It is **not** designed for AZ-wide or region-wide outages, and it is **not a substitute for a multi-AZ HA fleet**.

#### Requirements and Limits

| Constraint | Detail |
| --- | --- |
| **Same AZ only** | EC2 is AZ-bound — recovery moves the instance within the **same Availability Zone**. It does **NOT** protect against a full AZ failure, only isolated host or instance failures |
| **Spare capacity needed** | Relies on AWS having **spare EC2 host capacity** in the AZ. In a major multi-AZ regional event, recovery can fail if no spare exists |
| **Supported instance types only** | Older notes list **A1, C4, C5, M4, M5, R3, R4, R5** as representative. AWS has since broadened the supported set to most current-generation types and enables recovery automatically on many of them — verify against current AWS docs for the specific instance type |
| **EBS only** | The instance must use **EBS volumes exclusively**. **Instance store** is physically attached to the host — migrating to a new host cannot carry it along, so auto recovery is incompatible with instance-store volumes |

> [!tip] Why EBS-only makes sense
> Instance store lives on the host's local disks. Migrating to a new host is the whole point of auto recovery — but local storage cannot follow. Only network-attached EBS volumes can re-attach to the recovered instance on the new host. The restriction falls straight out of the [Stop/Start vs Restart](#stopstart-vs-restart--these-are-not-the-same) mechanics: any host change wipes [instance store](#instance-store).

#### Simplified Automatic Recovery (modern default)

Modern EC2 also includes **Simplified Automatic Recovery** — for most current-generation instance types, AWS automatically recovers a failed instance **without** requiring you to create a CloudWatch status check alarm. The mechanism and limits (same AZ, EBS-only, spare-capacity dependency) are the same; the difference is that you opt out, not in. The legacy "create a status check alarm with the Recover action" path is still supported and is what older study material describes.

#### Auto Recovery vs Stop/Start vs Restart

All three can result in a host migration except restart. The difference is what each preserves on migration:

| Operation | Host | Instance store data | Auto-assigned public IPv4 | Elastic IP |
| --- | --- | --- | --- | --- |
| **Restart (reboot)** | **Same host** — never migrates | Preserved | Preserved | Preserved |
| **Stop → Start** | May land on a **new host** in the same AZ | **Lost** | **Released; new IP on start** | Preserved (if attached) |
| **Auto recovery** | May land on a **new host** in the same AZ | **Lost** (EBS-only requirement) | **Preserved** | **Preserved** |

The IP preservation is what makes auto recovery distinct from a stop/start — and is why it can plausibly advertise as a transparent self-heal.

> [!warning] Gotcha — auto recovery is not HA
> Auto recovery moves the instance **within one AZ**. If the AZ itself fails (power outage, network partition, regional event), there is no host to recover onto. Multi-AZ high availability remains a fleet-and-load-balancer concern — see [Scaling — Horizontal vs Vertical](../patterns/scaling.md) and [Overview — Resilience Tiers](../overview.md#resilience-tiers). Use auto recovery as the *lightweight* automation tier, not the resilience strategy.

### Status checks and auto recovery — quick recall

> Two checks (**system** = host/service, **instance** = OS). Healthy = **2/2**. The status-check alarm fires when either check fails for **≥ 1 consecutive 5-minute period**, notifies via **SNS**, and can optionally **Reboot / Stop / Terminate / Recover**. **Recover** = auto recovery = a same-AZ heal that **preserves all IP addressing** but requires **EBS-only**, a **supported instance type**, and **spare host capacity**.

## AMI — Amazon Machine Image

An **AMI** is the template used to launch an EC2 instance. Mental models: a server image (on-prem virtualization), or a USB installer for an operating system. Every EC2 instance you launch is created from an AMI — even picking "Amazon Linux 2" in the console is just picking an Amazon-provided AMI under the hood.

The interesting case is creating **your own** AMI from an already-configured instance — capture-then-replay as a reusable launch template.

### Where AMIs come from

| Source | Example | Notes |
| --- | --- | --- |
| **AWS-provided** | Amazon Linux 2, Windows Server | Official AWS base images |
| **Community-provided** | Red Hat, CentOS, Ubuntu | Vendor-published OS distributions |
| **Marketplace** | Pre-packaged commercial software | Billed at the **normal instance hourly rate plus a per-hour software-license charge** bundled into the AMI |

### Core Properties

**Regional.** An AMI exists in **one region** and can only be used in that region. The *same* OS (e.g. Amazon Linux 2) has a **different AMI ID in every region** — `us-east-1`, `us-west-1`, Sydney, and so on. IDs use the format `ami-<random alphanumeric>`. **To use an AMI in another region you must copy it** (see [Key Gotchas](#key-gotchas) below).

**Permissions.** An AMI controls who can use it. There are three permission models:

| Permission | Who can use it |
| --- | --- |
| **Private** (default) | Only the AMI's owning AWS account — also called **Owner-only** |
| **Public** | Anyone — e.g. AWS-provided Amazon Linux / Ubuntu / Windows base images |
| **Explicit** | Specific named AWS accounts the owner has granted launch permission to (e.g. other accounts in your org, or partner accounts) |

### The AMI Lifecycle

Think of the lifecycle as **four phases** — and note the two separate Launch phases are intentional.

![AMI lifecycle diagram with four labelled bands: 1. Launch shows an AMI tile, an arrow into an Instance icon, and the resulting Boot volume /dev/xvda and Data volume /dev/xvdf attached to that instance; 2. Configure shows the same instance with its two volumes plus a cloud-shaped "Customizations" badge applied on top; 3. Create Image shows the customized instance + its volumes feeding into a new AMI tile, with two snapshot icons branching off labelled Block Device Mapping; 4. Launch repeats step 1's layout — the new AMI is used to stamp out a fresh Instance with the same two volumes /dev/xvda and /dev/xvdf, demonstrating that the AMI is both produced from an instance and used to produce more instances](../assets/ec2-ami-lifecycle.png)

1. **Launch** — Use an AMI to launch an EC2 instance. Most people only ever experience this phase, and that's fine — custom AMIs are one of several provisioning patterns, not a requirement.
2. **Configure** — Take the launched instance (the instance plus its attached [EBS](ebs.md) volumes) and customize it: install applications, tune the OS, attach additional volumes, install and configure an application suite.
3. **Create Image** — Capture that customized configuration as a new AMI (see [What an AMI Actually Contains](#what-an-ami-actually-contains) below).
4. **Launch (again)** — Use the new AMI to stamp out brand-new instances with **the same data and the same volume configuration** as the original.

The flow most people know — using an AMI to launch an instance — can be run in **reverse** (capturing an existing instance as an AMI) to turn an existing configuration into a reusable launch template.

### What an AMI Actually Contains

An AMI is best understood as a **logical container** — it does **not itself hold any real volume data**. It holds:

- An **AMI ID**
- **Permissions** (Private / Public / Explicit — see above)
- A **block device mapping** that references EBS snapshots

When you create an AMI from an instance, the key step is that **an EBS [snapshot](ebs.md#snapshots--crossing-the-az-and-region-boundaries) is taken of each attached EBS volume first** — and the AMI's block device mapping points at those snapshots. (Recall that EBS snapshots are incremental, but the *first* snapshot is a full copy of the volume's used data.)

#### Block Device Mapping

A small table inside the AMI. For each snapshot, it records:

- the **snapshot ID**, and
- the **device ID** the original volume had on the source instance.

A two-volume instance produces a mapping with two entries:

| Snapshot | Original device ID | Role |
| --- | --- | --- |
| snapshot of boot volume | `/dev/xvda` | boot |
| snapshot of data volume | `/dev/xvdf` | data |

When the AMI is later used to launch a new instance, the referenced **snapshots are used to create new EBS volumes in the target Availability Zone**, and those volumes are attached using the **same device IDs** recorded in the mapping. This is what reproduces the original volume layout on the new instance.

#### Regional Storage — AZ Reach Within the Region

The AMI object itself is stored in the region. The underlying snapshots live on [S3](s3.md), which is already regional and replicated across 3+ AZs. Because of this, **an AMI created from an instance in one AZ can deploy new instances back into that same AZ or into any other AZ in the region**. This is the standard path for [moving an instance across AZs](#moving-an-instance-across-azs) — and the foundation for multi-AZ HA fleets built from a single golden image.

### AMI Baking

**AMI baking** is the practice of taking an instance, installing all software and applying all configuration, then "baking" that finished state into an AMI. Example: instead of manually installing/configuring WordPress on every instance, do it once, bake an AMI, then launch tens or hundreds of pre-configured instances from it. Hardening / golden-image pipelines typically automate this with **EC2 Image Builder** or **Packer**.

### Key Gotchas

**AMIs cannot be edited.** There is no "edit AMI" operation. To change an existing AMI's configuration you must (1) launch an instance from it, (2) make your changes, then (3) create a **brand-new** AMI. The old AMI stays untouched.

**AMIs can be copied between regions.** Since AMIs are regional, copying is how you get the same image into another region (`aws ec2 copy-image`). Copying duplicates the underlying snapshots into the destination region. Copied AMIs retain the **default Private permissions** unless you change them — sharing does not travel with the copy.

**Encrypted AMIs** require [KMS](kms.md) grants for services like Auto Scaling to launch from them. The encrypted snapshot's DEK is wrapped by a KMS key in the AMI's region; copying across regions re-wraps under a destination-region KMS key. See [EBS Encryption](ebs.md#ebs-encryption).

### Billing

You are billed for an AMI based on the **storage capacity used by the EBS snapshots it references** — *not* for the AMI container itself.

Because [snapshots only store the *used* data](ebs.md#snapshots--crossing-the-az-and-region-boundaries) (not the full allocated volume size), the cost can be much smaller than the source volume's allocation suggests. A 40 GB volume with 10 GB written becomes a snapshot billed for ~10 GB. **Marketplace AMIs** add the bundled software license to the running instance's hourly rate — the AMI's snapshot storage is billed the same way as a custom AMI.

## Connecting to Instances

Two protocols cover ~all cases on the exam:

| OS | Protocol | Port | Authentication |
| --- | --- | :-: | --- |
| **Linux** | SSH | **22** | SSH key pair — private key authenticates you directly |
| **Windows** | RDP | **3389** | SSH key pair → decrypt local Administrator password → RDP login with that password |

### SSH key pair mechanics

When you create an EC2 key pair, AWS generates two halves:

- **Private key** — *you* download this **once and only once**, at the moment the key pair is created. Keep it safe; AWS does not retain a copy. **Losing the private key means losing access to instances that depend on it** (you can recover by detaching the root volume and attaching it to a different instance, but it's painful)
- **Public key** — AWS retains this and places it on any instance launched with that key pair selected (in `~/.ssh/authorized_keys` for Linux)

### How authentication actually works

- **Linux:** SSH performs a public-key challenge against the public key already on the instance. Your local SSH client signs the challenge with the private key; if it verifies against the public key, you're in. **No password is involved.**
- **Windows:** AWS encrypts the local Administrator account password with the public key at first boot. You use the private key to **decrypt** that password (in the console: *Get Windows Password*), then connect over RDP using `Administrator` + the decrypted password. The SSH key never touches the RDP session itself.

> [!tip] Exam Tip
> "I can't SSH to my instance" troubleshooting checklist: (1) **Security Group** allows inbound TCP 22 from your source IP, (2) the **subnet's route table** has a route to an [IGW](vpc.md#internet-gateway-igw) and the instance has a public IPv4, (3) the [NACL](vpc.md#network-acl-nacl) allows TCP 22 inbound **and** the ephemeral return range outbound (NACLs are stateless), (4) the right **key pair** is selected and the OS user is correct (`ec2-user` on Amazon Linux, `ubuntu` on Ubuntu, `admin`/`ec2-user` varies by distro), (5) the OS-level firewall inside the instance isn't blocking 22.

> [!warning] Gotcha
> The **private key is downloadable once**. Re-launching the wizard for the same key pair does **not** let you re-download it. If you lose it, create a new key pair; you cannot recover the original.

## Instance Metadata Service (IMDS)

The **Instance Metadata Service** is a service EC2 provides *to* its instances — a small HTTP endpoint that any process inside the instance can query to learn about the instance itself and the environment it runs in. Some of that information **only IMDS can supply** — most notably the [auto-assigned public IPv4](#0-or-1-public-ipv4--public-dns), which is never configured inside the OS.

> [!tip] Mental model
> The OS is a guest in a hotel room. It knows what's inside the room (its private IPv4, its IPv6) but not the building's street address (its public IPv4). The metadata service is the front desk it can phone to ask, *"what address am I at?"*

### The Endpoint

IMDS is reached at a fixed **link-local IPv4** — the same address on every EC2 instance in the world. Link-local means it is reachable only from inside the instance itself; the request never leaves the instance.

```text
http://169.254.169.254/latest/meta-data/<attribute>
```

| Part | Value |
| --- | --- |
| IP address | `169.254.169.254` — link-local, identical on every instance |
| Base path | `/latest/meta-data/` |
| Append | The attribute name (e.g. `public-ipv4`, `iam/security-credentials/<role-name>`) |

> [!tip] Exam Tip
> The exam can ask you to pick the *exact* URL. Memorize both halves: **"169.254 repeated"** for the IP, and always **"the latest meta-data"** for the path. The literal endpoint shows up in MCQ stems.

### What IMDS Exposes

Information is grouped into categories — networking, IAM, hostname, security groups, AMI ID, AZ, events, user data, and more. The exam-relevant uses:

| Use case | What IMDS provides | Why it matters |
| --- | --- | --- |
| **Public IPv4 discovery** | `public-ipv4`, `public-hostname` | The OS never sees the public IPv4 (1:1 NAT at the [IGW](vpc.md#internet-gateway-igw)). IMDS is the only path for code inside the instance to learn its own public address |
| **IAM role credentials** | `iam/security-credentials/<role-name>` returns Access Key ID + Secret Access Key + **session token** + expiry, refreshed automatically by AWS | When the instance has an [IAM role](iam.md#iam-roles) attached, every AWS SDK on the instance reads its credentials from here. This is why you must **never** bake access keys into AMIs or user data |
| **User data** | `/latest/user-data/` (separate path from `/meta-data/`) | Bootstrap scripts and cloud-init payloads passed at launch are exposed here for the instance to read and execute |
| **EC2 Instance Connect SSH keys** | `meta-data/public-keys/` | Instance Connect injects a short-lived SSH public key into IMDS; the SSH daemon picks it up via a helper script and accepts the matching one-time login |
| **Self-identifying attributes** | `instance-id`, `instance-type`, `ami-id`, `placement/availability-zone`, `security-groups`, `mac`, `local-ipv4`, `hostname`, etc. | Configuration management, telemetry tagging, AZ-aware routing inside the application |

### Querying in Practice

Two common ways from inside the instance:

1. **`curl`** against the metadata URL plus an attribute. Example: `curl http://169.254.169.254/latest/meta-data/public-ipv4` returns the public address the OS otherwise can't see.
2. **The `ec2-metadata` helper** — a small shell script bundled with Amazon Linux (`/usr/bin/ec2-metadata`) that wraps the same service with friendlier flags. `ec2-metadata --help` lists every attribute it can fetch.

Both call the same HTTP endpoint. SDKs (boto3, AWS SDK for Java, etc.) call it transparently when they need role credentials — you do not write that code yourself.

### IMDSv1 vs IMDSv2 — the security upgrade

IMDS has two versions. The exam mostly asks about the difference and why v2 exists.

| | **IMDSv1** | **IMDSv2** |
| --- | --- | --- |
| Authentication | None | **Session token** required on every request |
| How to use | Single `GET` to the metadata URL | (1) `PUT` to `/latest/api/token` (with a `X-aws-ec2-metadata-token-ttl-seconds` header) to obtain a token, (2) include that token in the `X-aws-ec2-metadata-token` header on every subsequent `GET` |
| Mitigates SSRF? | No | **Yes** — a server-side request forgery vulnerability that tricks an app into making a `GET` cannot obtain a token (browsers and most HTTP libraries won't issue the required `PUT`) |
| AWS guidance | Legacy, still supported for compatibility | **Recommended default for all new instances** |

A per-instance setting (`HttpTokens`: `optional` allows both; `required` forces v2-only) controls which versions are accepted. Setting `HttpTokens=required` is the modern hardening baseline and is what AWS Foundational Security Best Practices, CIS, and most exam scenarios expect.

> [!warning] Gotcha — IMDS is unauthenticated by default (IMDSv1)
> IMDSv1 has **no authentication** and **no encryption**. Anyone with a shell on the instance — or any web app vulnerable to SSRF — can read everything IMDS exposes, including the **IAM role's temporary credentials**. The 2019 Capital One breach is the textbook example: a misconfigured WAF allowed SSRF, the attacker pulled the EC2 role credentials from IMDSv1, and used them to read 100M+ S3 objects. The fix is **enforce IMDSv2** (`HttpTokens=required`) and treat the role's permissions like any other production secret — least privilege.

> [!tip] Exam Tip
> Pattern-match: *"how does code on an EC2 instance discover its own public IPv4?"* → **IMDS** (`http://169.254.169.254/latest/meta-data/public-ipv4`). *"how does an app on EC2 get AWS credentials without storing keys?"* → **IAM role + IMDS** (the SDK reads `iam/security-credentials/<role>` automatically). *"mitigate SSRF against the metadata service"* → **enforce IMDSv2** (`HttpTokens=required`). *"restrict access to metadata at the host level"* → block `169.254.169.254` with a local OS firewall rule (per-instance, high-overhead — IMDSv2 is the real answer).

### Quick recall

> Metadata lives at **`169.254.169.254/latest/meta-data/`**, IMDSv1 is **unauthenticated and unencrypted** (use **IMDSv2** with session tokens), and it is the only way an instance can discover its own **public IPv4** — because that address lives on the [IGW](vpc.md#internet-gateway-igw), never in the OS.

## When to Use EC2

EC2 is the **default compute service** in AWS. Reach for it first; deviate only when a specific requirement clearly favours a managed alternative.

| Pattern | Why EC2 fits |
| --- | --- |
| **Traditional OS + application stack** | You need a specific OS, vendor-supported runtime, or per-host licensing — Lambda / Fargate can't host these |
| **Long-running compute (24/7/365)** | EC2 has no runtime cap, unlike Lambda's 15-minute limit |
| **Server-style applications** waiting for inbound connections inside an OS | Web servers, databases, application servers, queue workers |
| **Burst or steady-state CPU/memory workloads** | Burstable (`t` family), compute-optimised (`c`), memory-optimised (`r` / `x`), and many specialised instance types cover both shapes |
| **Monolithic application stacks** (app + DB + middleware on one host) | A single OS hosts everything; no need to decompose first |
| **Workload migrations from on-prem VMs** | Lift-and-shift maps naturally — same OS, same boot artefacts (via [AMI](#ami--amazon-machine-image)) |
| **Disaster-recovery environments** for traditional server-based systems | Cold/warm DR instances + EBS snapshots + cross-region AMI copy give a familiar restoration shape |

Move *away from* EC2 when a niche requirement clearly favours another service:

- **Event-driven, short-lived, sporadic** → Lambda
- **Containerised microservices without OS management** → ECS / EKS (with Fargate for the no-host case)
- **Fully managed runtime / PaaS** → Elastic Beanstalk, App Runner
- **Specialised workload primitives** (queues, streams, data warehouses) → use the managed primitive, not a VM hosting it

> [!tip] Exam framing
> When a question lists a generic "VM / OS / runtime / per-socket license / long-running server" requirement, the answer is almost always **EC2**. The question shifts to a managed compute service only when it explicitly names an event-driven, container, or no-OS-management constraint. Justify deviations; don't volunteer them.

## Instance Types

Every EC2 instance has an **instance type** — a code like `R5dn.8xlarge` — that determines its vCPU count, memory, local storage presence, network bandwidth, and any specialised hardware (GPUs, FPGAs). Picking the right type is the difference between a well-tuned workload and a performance problem.

### The Five Categories

| Category | Family examples | Best for |
| --- | --- | --- |
| **General Purpose** | T3, T3a, M5, M5a, M5n, A1, M6g | Default starting point — even CPU-to-memory ratio |
| **Compute Optimized** | C5, C5n | More CPU than memory per dollar — media encoding, HPC, scientific modelling, ML training, gaming servers |
| **Memory Optimized** | R5, R5a, X1, X1e, High Memory (u-Xtb1), z1d | Large memory per dollar — in-memory databases, caches, real-time analytics |
| **Accelerated Computing** | P3, G4, F1, Inf1 | GPUs (ML training/inference, graphics) or FPGAs (genomics, financial analysis) |
| **Storage Optimized** | I3/I3en, D2, H1 | Fast local storage — NoSQL, data warehousing, Elasticsearch, Hadoop |

### Decoding an Instance Type Name

Instance type names follow a strict pattern. The **full string** is the instance type — use it in full when communicating with operations teams.

![Anatomy of an EC2 instance type name: family letter, generation number, optional capability letters, and size](../assets/ec2-instance-type-code-explanation.png)

**Example: `R5dn.8xlarge`**

| Component | Example | Meaning |
| --- | --- | --- |
| **Family** | `R` | Letter identifying the family — `R` = RAM / memory |
| **Generation** | `5` | Higher = newer. Pick the newest generation available |
| **Additional capabilities** | `dn` | Optional letters between generation and size: `a` = AMD CPUs, `d` = NVMe local storage, `n` = network optimised, `e` = extra capacity (RAM/storage), `g` = Graviton (ARM) |
| **Size** | `8xlarge` | Scales CPU and memory: nano → micro → small → medium → large → xlarge → 2xlarge → 4xlarge → 8xlarge → … |
| **Full type string** | `R5dn.8xlarge` | The precise, unambiguous identifier |

### Family Reference

![Common EC2 instance types organized by category — T, M, C, R, I, D, G, P families with use cases](../assets/ec2-typical-instance-types.png)

### Letter Mnemonics

| Letter | Mnemonic |
| --- | --- |
| **T** | **T**urbo / burstable — cheap for mostly-idle with occasional peaks |
| **M** | "**M**ain" — general purpose steady state |
| **C** | **C**ompute |
| **R** | **R**AM / memory |
| **I** | **I**/O (local NVMe SSD) |
| **D** | **D**ense storage (high-throughput HDD) |
| **G** | **G**PU |
| **P** | **P**arallel processing (larger GPU configurations) |

### Instance Type Selection Guide

Start in **General Purpose (T3/M5)**. Only move out when there is a clear workload bottleneck:

- Bottleneck is **CPU** → Compute Optimized (C5, C5n)
- Bottleneck is **memory / large in-memory dataset** → Memory Optimized (R5, X1, High Memory, z1d)
- Workload needs **GPUs, FPGAs, or ML accelerators** → Accelerated Computing (P3, G4, F1, Inf1)
- Workload needs **fast local storage or very high local IOPS** → Storage Optimized (I3/I3en, D2, H1)
- Workload is **mostly idle with occasional bursts** → T3 / T3a (burstable) — cheaper per hour, limited by CPU credit balance
- Workload is **sustained steady-state** → M5 / M5a / M5n (non-burstable)
- Workload runs **ARM-compatible code** and you want cost efficiency → A1 or M6g (Graviton)
- Workload is **EBS-heavy** → verify the instance has sufficient storage network bandwidth — the **instance**, not just the EBS volume, can be the limiting factor. See [EBS and storage options](#storage-options)

> [!tip] Exam Tip
> SAA instance-type questions typically describe a performance symptom: high CPU, high memory, I/O saturation, or GPU requirement. The core skill is **category selection**: C = more CPU per dollar, R = more RAM per dollar, I/D = more local storage throughput. Specific family generation numbers matter less than getting the letter right.

> [!warning] Gotcha — burstable T3 under sustained load
> **T3** instances earn CPU credits at idle and spend them during bursts. Under sustained high CPU load, credits are exhausted and the instance throttles to its baseline rate (often well below peak). The exam trap: *"T3 performance degraded over time under consistent load"* → CPU credit exhaustion. Move to **M5** for sustained CPU workloads.

> [!warning] Gotcha — prefer newer generations
> Within a family, higher generation number = better price-to-performance. Reasons to stay on an older generation: it is not yet available in your region, or internal testing/approval processes still require it. These are the only legitimate exceptions.

## Instance Store

Instance store provides **block storage physically attached to the EC2 host** — functionally similar to EBS but local rather than network-based. It is the highest-performing storage option in AWS, and the most constrained.

For the decision framework (when to choose instance store vs EBS), see [EBS vs Instance Store](../services/ebs.md#ebs-vs-instance-store--decision-framework).

### The Three Essentials

#### 1. Ephemeral — Not Persistent

Instance store is temporary. Data is **lost** whenever the instance moves to a new host:

- **Stop/Start** — forces a host migration (the instance may land on a different host in the same AZ)
- **Resize / instance type change** — forces a host migration
- **Host failure or maintenance** — the instance relocates to a new host
- **Volume hardware failure** — the physical disk itself fails

When an instance migrates, it receives **new, blank** ephemeral volumes — old data is gone.

> [!warning] Gotcha — Reboot is not a host move
> A **restart (reboot)** keeps the instance on the same host — instance store data **survives**. Only events that *move the instance to a new host* wipe the data. A common exam trap describes losing scratch data "after a reboot" — check whether the action was actually a **stop/start** in disguise.

#### 2. Local to the Host (vs EBS = Network-Based)

- Instance store volumes are **physically connected to one EC2 host** — no network hop
- Only instances on that host can access them
- Because there is no network latency, they deliver **the highest storage performance in AWS**

#### 3. Attach at Launch Time Only

- Instance store volumes **cannot be added after an instance launches**. Unlike EBS, you cannot attach more later
- You may choose not to use the allocated volumes, but you cannot gain more post-launch

> [!tip] Exam Tip
> Any question about attaching instance store volumes *after* launch — the answer is always **no, launch time only**.

### Architecture Details

- Each EC2 host has a set of instance store volumes backed by **physical devices on that host**
- Volumes are named `ephemeral0`, `ephemeral1`, `ephemeral2`, …
- Multiple instances on one host divide the host's volumes among themselves — e.g. Instance A uses `ephemeral0`; Instance B uses `ephemeral1` and `ephemeral2`
- When an instance migrates to a new host, it still gets an `ephemeral0`, but it is a **new, blank** physical volume on the new host

### Cost

- Instance store is **included in the instance price** — you pay for it whether you use it or not
- Different instance types come with different numbers and sizes of instance store volumes
- Since you are already paying for it, there is **no advantage to leaving allocated volumes unused**

### Performance

The whole point of instance store is accepting tight restrictions in exchange for the highest data performance in AWS.

| Instance Family | Media | Peak Throughput | Notes |
| --- | --- | --- | --- |
| **D3** (storage-optimized) | HDD | ~4.6 GB/s | High value for large sequential workloads |
| **i3** (storage-optimized) | NVMe SSD | up to ~16 GB/s | Far exceeds even the highest-performance EBS |

- Certain i3 instances reach **~2 million read IOPS** and **~1.6 million write IOPS** — compare to EBS's maximum of 256,000 IOPS per volume or ~260,000 IOPS per instance via RAID 0
- Volume size and count **vary by instance type and size** — larger sizes generally include more and larger volumes
- **Some instance types have no instance store** — verify the instance type's specification before assuming

### Instance Store Key Takeaways

- **Local to the host** → any host migration destroys the data
- **Launch time only** → cannot add instance store volumes after launch
- **Reboot = data survives; Stop/Start = data may be lost** — stop/start can land on a new host
- **Highest performance in AWS** — the trade-off for accepting all restrictions
- **Included in instance price** → no benefit to leaving allocated volumes unused
- **Never use for persistent or irreplaceable data**

## Scaling EC2 — Vertical vs Horizontal

EC2 capacity grows in two fundamentally different ways. The choice cascades into every other architectural decision (load balancing, session storage, availability tier, cost shape):

- **Vertical scaling** — resize one instance to a bigger type. Simple, works for monoliths, no application changes, but requires a **stop / modify / start** sequence (disruption) and is capped at the largest available instance size.
- **Horizontal scaling** — add more instances behind a load balancer. No upper limit, no disruption to scale, but requires **stateless application design** with sessions stored off-host ([ElastiCache](../patterns/scaling.md#the-critical-consideration--sessions), DynamoDB, RDS).

The instance-level concerns are EC2-specific (resize is a stop/start, see [Stop/Start vs Restart](#stopstart-vs-restart--these-are-not-the-same)); the design pattern itself is broader. See [Scaling — Horizontal vs Vertical](../patterns/scaling.md) for the full treatment, including the session problem, sticky-session gotcha, and how horizontal scaling combines with ELB and Auto Scaling Groups to deliver HA.

## Key Takeaways

1. **EC2 = IaaS** — you manage OS and above; AWS manages everything below
2. **EC2 is the default compute service** in AWS — start here unless a managed option (Lambda, Fargate, Beanstalk) fits better
3. **Host = physical, AZ-bound; instance = virtual, AZ-bound through its host.** Hosts, subnets, ENIs, and EBS are all AZ-scoped — no cross-AZ attachment
4. **Tenancy: Shared (default) vs Dedicated Host (per-host billing, host targeting, BYOL licensing) vs Dedicated Instance (no co-tenancy, host opaque)**
5. **Private by default** — public access is explicit (public subnet + IGW + public IP + SG rule)
6. **AZ resilient, not fault-tolerant** — an instance dies with its AZ; HA needs multiple AZs and a load balancer; cross-AZ moves require snapshot/AMI
7. **Three key states:** Running, Stopped, Terminated. **Stop/Start ≠ Restart** — Stop/Start can move the instance to a new host (losing instance-store data and auto-public-IP); Restart cannot
8. **Stopped ≠ free** — you still pay for attached [EBS](#storage-options) storage; only **Terminated** ends all charges
9. **Terminated is permanent** — root volumes deleted by default, instance-store data lost, no undelete
10. **AMI = a logical container, not a data carrier** — it holds an AMI ID, permissions (Private / Public / Explicit), and a block device mapping pointing at EBS snapshots; the real volume data lives in the snapshots. Sources: AWS / Community / Marketplace (Marketplace adds a per-hour license charge on top of the instance rate). **Region-specific** — copy across regions. **Cannot be edited** — relaunch, reconfigure, recreate. Stored in the region but deployable to **any AZ within it**. Billed on the **snapshot used data**, not the AMI itself or the volume's allocated size. **AMI baking** = configure once, capture, stamp out many
11. **Linux: SSH on 22**; **Windows: RDP on 3389** (private key decrypts the admin password, then RDP)
12. **Private key downloads once** — lose it and you've lost the easy path back in
13. **Instance types:** five categories (General Purpose / Compute / Memory / Accelerated / Storage). Start in General Purpose; deviate when a bottleneck is clear. **T3 is burstable** (credit-based); **M5 is steady-state**. Full type string = family + generation + optional capability letters + size
14. **Instance store** — physically local to the host; highest performance in AWS; launch-time only; **lost on any host move** (stop/start, resize, host failure); reboot preserves data; included in instance price. See [Instance Store](#instance-store)
15. **ENI is the unit of network identity, not the instance** — IPs, DNS names, MAC, EIPs, IPv6, [Security Groups](vpc.md#security-groups-sg), and source/destination check **all live on the ENI**. Every instance has exactly 1 primary ENI (cannot detach) + 0+ secondary ENIs (detachable, movable between instances — the MAC-licensing / IP-failover pattern). All ENIs on an instance must be in the **same AZ**, but can sit in different subnets. The auto-assigned public IPv4 is **not visible inside the OS** (NAT at the [IGW](vpc.md#internet-gateway-igw)); IPv6 is. **Split-horizon DNS**: the public DNS name resolves to the **private IP inside the VPC**, the public IP **outside**. See [Network Interfaces (ENIs)](#network-interfaces-enis)
16. **Instance Metadata Service (IMDS)** — fixed link-local endpoint **`http://169.254.169.254/latest/meta-data/`** that every instance can query about itself. Exposes networking (including the **public IPv4 the OS cannot see**), **IAM role credentials** (how SDKs get keys without baking them in), user data, and EC2 Instance Connect SSH keys. **IMDSv1 is unauthenticated and unencrypted** — vulnerable to SSRF exfiltration of role credentials (Capital One 2019); enforce **IMDSv2** (`HttpTokens=required`) which adds a session-token handshake. See [Instance Metadata Service (IMDS)](#instance-metadata-service-imds)
17. **Status checks and auto recovery** — every instance has **two independent status checks**: **System** (the host/service underneath) and **Instance** (the OS layer); healthy = **2/2**. A **CloudWatch Status Check Alarm** can fire when either check fails for **≥ 1 consecutive 5-minute period**, notify via **SNS**, and optionally **Reboot / Stop / Terminate / Recover**. **Auto recovery** = same-AZ heal that **preserves all IP addressing** (private + auto public IPv4 + EIP); requires **EBS-only**, a **supported instance type**, and **spare host capacity**. Same-AZ only — **not a substitute for multi-AZ HA**. Modern current-generation types get **Simplified Automatic Recovery** by default with no alarm needed. See [Status Checks and Auto Recovery](#status-checks-and-auto-recovery)

## Quick Reference

| Concept | Detail |
| --- | --- |
| Service category | IaaS — Infrastructure as a Service |
| Physical layer | **Host** — AWS-managed physical server, lives in one AZ |
| Virtual layer | **Instance** — VM on a host, AZ-bound through that host |
| Tenancy modes | Shared (default) / Dedicated Instance / Dedicated Host |
| Resilience tier | AZ-specific (single-AZ failure = instance failure) |
| AZ-scoped components | Host, subnet, ENI, EBS volume, instance store |
| Cross-AZ movement | Not native — requires snapshot or [AMI](#ami--amazon-machine-image) launch in target AZ |
| Default network posture | Private (in a [VPC](vpc.md) subnet) |
| Billing granularity | Per second (most) or per hour (some commercial licenses) |
| Free state | **Terminated only** (volumes deleted, instance gone) |
| Hidden cost trap | **Stopped** instances still pay for [EBS](#storage-options) |
| Restart vs Stop/Start | Restart = same host (IP + instance store preserved); Stop/Start = may land on a new host (auto public IP + instance store **lost**) |
| Instance store | Ephemeral; local to host; launch-time only; lost on any host move; **highest performance in AWS** — see [Instance Store](#instance-store) |
| ENI | Per-instance: 1 primary (non-detachable) + 0+ secondary (detachable, movable); holds IPs / DNS / MAC / EIPs / IPv6 / SGs / src-dst check; all ENIs share AZ; can span subnets — see [Network Interfaces (ENIs)](#network-interfaces-enis) |
| Public DNS resolution | **Split-horizon** — inside VPC → private IPv4; outside VPC → public IPv4 |
| Public IPv4 visibility in OS | **Not visible** (NAT at IGW); IPv6 **is** visible |
| Elastic IP attach/detach | Attach **replaces** the dynamic public IP; detach gives a **new, different** dynamic IP — original is lost |
| Source/destination check | Default on; **must be off** for NAT instances / software routers |
| Template object | AMI (Amazon Machine Image) — region-specific; deployable to any AZ in the region |
| AMI permission modes | **Private** (default, owner-only) / **Public** / **Explicit** (named accounts) |
| AMI sources | AWS-provided / Community / Marketplace (Marketplace = instance rate + per-hour license) |
| AMI contents | Logical container: **AMI ID + permissions + block device mapping** pointing at EBS snapshots (no volume data in the AMI itself) |
| AMI editing | **Cannot be edited** — to change one, (1) launch from it, (2) modify the instance, (3) create a new AMI |
| AMI billing | **EBS snapshot used data**, not the AMI container, not the volume's allocated size |
| AMI baking | Configure an instance, install everything, capture as AMI, launch many identical instances |
| Linux access | SSH on port 22 + private key |
| Windows access | RDP on port 3389 + private key (used to decrypt admin password) |
| Key-pair download policy | Private key downloadable **once** only |
| Instance Metadata Service endpoint | **`http://169.254.169.254/latest/meta-data/`** — link-local, same IP on every instance |
| IMDS use cases | Public IPv4 discovery, IAM role credentials, user data, EC2 Instance Connect SSH keys, self-identifying attributes (AMI ID, AZ, instance type, security groups) |
| IMDSv1 vs IMDSv2 | v1 = unauthenticated `GET` (SSRF-exploitable); v2 = `PUT`-token handshake + `X-aws-ec2-metadata-token` header on every `GET`. Enforce v2 via `HttpTokens=required` |
| Status checks | **System** (host/service) + **Instance** (OS). Healthy = **2/2 checks passed**. Anything else after boot = problem |
| Status Check Alarm trigger | Either check fails for **≥ 1 consecutive 5-minute period** |
| Status Check Alarm destinations | **SNS** topic (notification) + optional EC2 action: **Reboot / Stop / Terminate / Recover** |
| Auto recovery | Same-AZ heal that **preserves all IP addressing** (private + auto public IPv4 + EIP); requires EBS-only, supported instance type, spare capacity |
| Auto recovery limits | **Same AZ only** (not cross-AZ — not HA), **EBS-only** (no instance store), supported instance types only, spare host capacity required |
| Simplified Automatic Recovery | Default on most current-generation instance types — same mechanism, no CloudWatch alarm required |

## References

- [VPC — networking, SG, NACL, IGW, NAT GW](vpc.md)
- [IAM — roles for EC2](iam.md)
- [Networking Concepts — ENI, OSI layers, ephemeral ports](../concepts/networking.md)
- [Identity & Access — auth vs authz, credential types](../concepts/identity-and-access.md)
- [Cost Optimization — Savings Plans, Reserved Instances, Spot Instances](../patterns/cost-optimization.md)
- [EC2 Purchase Options — On-Demand Instances, Spot Instances, Reserved Instances, On-Demand Capacity Reservations, Savings Plans, Dedicated Hosts, Dedicated Instances](../patterns/ec2-purchase-options.md)
- [Scaling — Horizontal vs Vertical](../patterns/scaling.md)
- [AWS SAA Exam Notes — EC2](../certifications/aws-saa.md#ec2)
- Source: `raw/aws-cert-study-notes/aws-solution-architect/ec2_101.md`
- Source: `raw/aws-cert-study-notes/aws-solution-architect/ec2_architecture_and_resilience.md`
- Source: `raw/aws-cert-study-notes/aws-solution-architect/ec2_instance_type.md`
- Source: `raw/aws-cert-study-notes/aws-solution-architect/ec2_instance_store_volumes.md`
- Source: `raw/aws-cert-study-notes/aws-solution-architect/ec2_instance_store_vs_ebs.md`
- Source: `raw/aws-cert-study-notes/aws-solution-architect/ec2_eni_and_dns.md`
- Source: `raw/aws-cert-study-notes/aws-solution-architect/ec2_ami.md`
- Source: `raw/aws-cert-study-notes/aws-solution-architect/ec2_horizontal_and_vertical_scaling.md`
- Source: `raw/aws-cert-study-notes/aws-solution-architect/ec2_instance_metadata.md`
- Source: `raw/aws-cert-study-notes/aws-solution-architect/ec2_instance_status_checks_and_auto_recovery.md`
- [EC2 Instance Types — official AWS docs](https://aws.amazon.com/ec2/instance-types/)
- [ec2instances.info (Vantage)](https://instances.vantage.sh/) — sortable reference with pricing
