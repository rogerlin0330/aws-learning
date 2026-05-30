# EC2 Network Interfaces, Instance IPs & DNS

> A guided reading of the **EC2 Network & DNS Architecture** diagram. Each section below maps to a part of the picture, so you can look at the diagram and know exactly what every box, arrow, and label is telling you.

![[ec2-network-interface-and-dns-architecture.png]]

---

## How to read the diagram

The diagram has two halves:

- **Left half** — the _physical-ish layout_: an Availability Zone containing an instance and its network interfaces.
- **Right half** — a _list of attributes_, with a big red arrow showing that these attributes belong to the **Primary ENI** (not the instance directly).

Read it left-to-right: start with the AZ boundary, find the instance and its interfaces, then follow the red arrow to see what's actually attached to an interface.

---

## Left half — instance, interfaces, and the AZ boundary

### The dotted box labelled `AZA`

The outer dotted boundary is **Availability Zone A**. Everything inside it lives in **one AZ** — this is the key constraint the diagram is showing you. An instance and _all_ of its network interfaces must sit in the same AZ. Interfaces can be in different _subnets_, but never different AZs.

### The chip icon in the middle = the EC2 instance

The orange chip is the **instance itself**. Notice it doesn't connect directly to the IPs and DNS names on the right — it connects to the **interfaces** (the dotted lines), and the interfaces hold the addressing. This is the single most important idea in the diagram.

### `Primary ENI` (top interface icon)

Every instance **always** has exactly one **primary elastic network interface (ENI)**. The big red arrow comes off _this_ interface — meaning the whole attribute list on the right describes the **primary ENI**.

> When the console shows IPs, DNS names, or security groups "on the instance," you're usually looking at the **primary ENI**. You can almost think of the primary ENI _as_ the instance — but technically they're separate, and the config lives on the interface.

### `Secondary ENI's` (bottom interface icon)

An instance can **optionally** have one or more **secondary ENIs** (how many depends on the instance type). The dotted line shows they also attach to the same instance, inside the same AZ. Secondary ENIs can sit in **different subnets** from the primary.

The note **"As Above ^^"** at the bottom of the attribute list means: a secondary ENI has the **same set of attributes** as the primary. The one crucial difference (not drawn) is that a **secondary ENI can be detached and moved to another instance**.

---

## Right half — the attribute list (what's on an ENI)

Follow the red arrow. Each line is something attached to the interface:

### `Mac Address - 00:0d:83:b1:c0:8e`

The interface's **hardware (MAC) address**, visible inside the OS. Because it looks static, it's often used for **software licensing** — a detail that pairs with the movable-secondary-ENI trick below.

### `Primary IPv4 Private IP → 10.16.0.10 → ip-10-16-0-10.ec2.internal`

This row is an **example demonstrating the private IP → private DNS relationship** — the values are illustrative, not fixed. Read it as a **chain**: the interface has a **primary private IPv4**, which maps to a matching **internal (private) DNS name**. Here `10.16.0.10` maps to `ip-10-16-0-10.ec2.internal`.

- The private IP is **static** — it stays the same for the life of the instance.
- The DNS name is **derived from** the private IP: `ip` + the private IP with hyphens instead of dots + `.ec2.internal`. (So a different private IP produces a different name in the same format.)
- This name is **only resolvable inside the VPC** and always points at the private IP. Use it for **internal communication only**.
- Choosing a VPC/subnet for the instance really means choosing it for **this primary interface**.

### `0 or more secondary IPs`

The interface can also hold **zero or more secondary private IPv4 addresses**.

### `0 or 1 Public IPv4 Address → 3.89.7.136 → ec2-3-89-7-136.compute-1.amazonaws.com`

Same idea — an **example demonstrating the public IP → public DNS relationship**, with illustrative values. An optional **public IPv4** maps to a matching **public DNS name**. Here `3.89.7.136` maps to `ec2-3-89-7-136.compute-1.amazonaws.com` (`ec2` + the public IP with hyphens + a domain like `compute-1.amazonaws.com`).

- An interface has **zero or one** public IPv4.
- This public IP is **dynamic**: it's deallocated on **stop**, and a **new** one is assigned on **start**. A **reboot does not change it**; a forced host migration does.
- The OS **never sees** this public IP — it's applied by **NAT at the Internet Gateway**. Inside the OS you always configure the **private** IPv4.

#### The `VPC = Private IP … Elsewhere = Public IP` annotation

This callout (pointing at the public DNS name) explains **split-horizon DNS** — the single most exam-relevant idea on the diagram:

- **Inside the VPC**, the public DNS name resolves to the **primary private IPv4**.
- **Elsewhere (outside the VPC)**, the same name resolves to the **public IPv4**.

So one DNS name works everywhere, and instance-to-instance traffic inside the VPC stays private (never hairpins out to the Internet Gateway and back). This matters later for hybrid networking and VPC peering.

### `1 elastic IP per private IPv4 address`

The interface can have **one Elastic IP per private IPv4** it holds. An Elastic IP is a public IPv4 **allocated to your AWS account** and associated with a private IP on an interface.

#### The `Removes the Public IPv4 / Replaces with the Elastic IP` annotation

This callout explains what happens when you attach an Elastic IP to the primary interface:

- The instance's existing **dynamic public IPv4 is removed**, and the **Elastic IP becomes** the new public IPv4.
- If you later **remove** the Elastic IP, the instance gets a **brand-new (different) dynamic public IPv4** — the original is gone for good.

### `0 or more IPv6 addresses`

The interface can hold **zero or more IPv6 addresses**. There's no public/private split for IPv6 — **all IPv6 addresses are publicly routable**, and unlike IPv4, the OS _does_ see them.

### `Security Groups`

Security groups attach to the **interface**, so a security group affects **every IP on that interface**. If you need different IPs governed by different security groups, you must split them across **multiple interfaces**.

### `Source/Destination Check`

A per-interface flag. When **enabled**, traffic is **discarded** unless one of the interface's IPs is the **source** or **destination**. It must be **disabled** for an instance to act as a **NAT instance**.

---

## Putting the diagram together (one-line summary)

> An **instance** (chip) lives in **one AZ**; it owns a **primary ENI** plus optional **secondary ENIs**; and each ENI — _not the instance_ — carries the MAC, the private/public IPv4s and their DNS names, optional Elastic and IPv6 addresses, security groups, and the source/dest check.

---

## Exam power-ups (the diagram's hidden gotchas)

- **Attributes are on the ENI, not the instance.** Security groups especially — need different SGs per IP? Use multiple ENIs.
- **Public DNS resolves split-horizon:** private IP inside the VPC, public IP outside. Keeps internal traffic internal; key for hybrid networking and VPC peering.
- **Normal public IPv4 is dynamic:** changes on stop/start and host migration, **not** on reboot. Use an **Elastic IP** for a fixed address.
- **Attach then detach an Elastic IP → original public IP is lost** (you get a different new one).
- **The OS never sees the public IPv4** (NAT via Internet Gateway) — always configure the private IPv4 in the OS. IPv6 is the exception (all public, OS-visible).
- **Source/Destination check must be disabled** for NAT instances.
- **Secondary ENIs are movable** between instances — useful for MAC-based licensing and multi-homed designs.