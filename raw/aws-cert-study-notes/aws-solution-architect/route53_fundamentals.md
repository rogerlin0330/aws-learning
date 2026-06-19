# Route 53 (R53) Fundamentals

## Overview

**Route 53** is AWS's **managed DNS product**. It brings the abstract DNS concepts (registries, zones, name servers, delegation) into an AWS-native service.

Route 53 provides **two main services**:

1. **Domain registration** — register domains (R53 acts as a registrar with relationships to the major registries).
2. **Hosted zones** — host zone files for you on **AWS-managed name servers** it provides. Essentially **DNS-as-a-service**.

### Global service characteristics

- Route 53 is a **global service with a single database** — one of very few AWS services that operates as a single global service.
- **No region selection** needed in the console UI.
- Its data is **distributed globally as a single set** and **replicated between regions** → **globally resilient**.
- Can **tolerate the failure of one or more regions** and keep operating.

> 🎯 **Exam angle:** "globally resilient" / "single global service" / "no region to pick" are Route 53 trigger phrases. Contrast with most AWS services, which are region-scoped.

---

## Service 1 — Register Domains

Route 53 has **relationships with all of the major domain registries** — the companies that manage the top-level domains (TLDs). These registries were **delegated their authority by IANA**, who manage the **root zone** for DNS.

Each registry manages **one specific TLD zone**:

- `.com` → one registry
- `.net` → another
- `.io` → another
- `.org` → **PIR** (Public Interest Registry) ← used in the course scenario (`animals4life.org` / `animalsforlife.org`)

_(These are examples of the registry-per-TLD pattern, not an exhaustive list.)_

### What happens when you register a domain

1. **Availability check** — Route 53 checks with the **registry for that TLD** whether the domain is available. _(Assume available for this walkthrough.)_
2. **Create the zone file** — Route 53 creates a **zone file** for the domain. A zone file is just a **database containing all the DNS information** for a particular domain (e.g. `animalsforlife.org`). In R53 terminology this becomes a **hosted zone**.
3. **Allocate name servers** — Route 53 creates and manages **name servers**, distributed globally — **generally four per zone**. It puts the zone file **onto these four managed name servers**.
4. **Liaise with the TLD registry** — Route 53 communicates with the TLD registry (**PIR** for `.org`) and gets **name server (NS) records** added into the **TLD's zone file** (the `.org` zone). These NS records **point back at our four name servers**.

> The NS records are how the registry (PIR) **delegates admin of the domain to us** — by adding them to the `.org` zone, PIR indicates that **our four name servers are authoritative** for our domain.

### Registration in one sentence

> Create a zone file → create managed name servers → put the zone file on those servers → get NS records added to the TLD zone pointing back at those servers. **DNS is just a system of delegation.**

### Diagram walkthrough — "Register Domains"

![[R53-function-register-domains.png]]

Read this diagram from the **top-left (Route 53)** outward along the arrows:

- **Top-left — Route53 (the purple "53" shield icon):** the actor performing all the work.
- **Top-right dotted box — "Registries":** holds building icons for **`.com`, `.io`, `.net`**, representing the various TLD registries. The thick **red arrow from Route53 to this box** = Route 53's relationships with the major registries (it can talk to any of them).
- **Middle-right — `.org` (building icon):** the **specific registry (PIR)** used for this scenario. The thick **red arrow from Route53 down to `.org`** = Route 53 contacting the `.org` registry to register the domain. The label **"PIR"** names the organization, and **"animals4life"** with a pink **"NS"** label sits in the lower dotted box = the `.org` zone receiving our domain's NS records.
- **Bottom-left — `animals4life.org` (document/code-page icon):** the **zone file** Route 53 created. The thin **dotted arrows from Route53** show R53 creating both the zone file and the name servers.
- **Bottom-middle — four server icons:** the **four managed name servers**. The **red arrow from the zone-file document into the servers** = Route 53 placing the zone file onto the name servers.
- **Red arrow from the `.org`/PIR box pointing left into the servers (labelled NS):** PIR's `.org` zone now carries **NS records pointing back at our four name servers** — i.e. delegation completed.

> 💡 **Editorial note (mine, for clarity):** the diagram uses both `animals4life` (in the registry box) and `animalsforlife.org` (in the transcript) — same scenario domain, just rendered two ways across Cantrill's materials.

---

## Service 2 — Hosted Zones

A **hosted zone** is Route 53's term for a **zone file hosted on AWS-managed name servers**. Route 53 provides both the **DNS zones** and the **hosting** for them.

- When a hosted zone is created, **name servers are allocated and linked** to it. The hosted zone and its name servers are **essentially one and the same** — every hosted zone has a set of allocated managed name servers.
- A hosted zone **hosts DNS records**.

### Public vs Private hosted zones

|                   | **Public hosted zone**                                                       | **Private hosted zone**                                      |
| ----------------- | ---------------------------------------------------------------------------- | ------------------------------------------------------------ |
| **Accessibility** | Accessible on the **public internet**                                        | Accessible **only from within linked VPC(s)**                |
| **Name servers**  | Live logically in the **AWS public zone**; part of the **public DNS system** | Linked to **one or more VPCs**                               |
| **Use case**      | Public-facing DNS                                                            | **Sensitive DNS records** you don't want publicly accessible |

### Records vs Record Sets

- Inside Route 53, records are referred to as **record sets**.
- There is a **tiny technical difference**, but for now you can treat **records ≈ record sets** as the same thing.

### Diagram walkthrough — "Hosted Zones"

![[R53-function-hosted-zones.png]]

The slide is a bullet summary on the **left**, with a **visual on the right**:

- **Right side — four server icons stacked vertically inside a red dotted box labelled "RECORDS":** the **four managed name servers** that host the zone, and the dotted box represents the **DNS records (record sets)** stored within the hosted zone.
- **Left bullets** restate the key facts:
    - **Zone files in AWS** — a hosted zone is a zone file hosted in AWS.
    - **Hosted on four managed name servers** — the four servers shown on the right.
    - **Can be public** — accessible on the public internet.
    - **Or private** — **linked to VPC(s)**.
    - **Stores records (recordsets)** — what lives inside the zone.

> 🎯 **Exam angle:** "four managed name servers" and "private hosted zone linked to a VPC" are common detail points. Private = VPC-scoped = good for internal/sensitive DNS.

---

## ⚠️ Traps / Gotchas

- **Don't confuse registry vs registrar vs hosted zone.** Route 53 plays **registrar** (registers domains via registries) **and** **DNS host** (hosted zones). These are the two distinct services — they're related but separate.
- **NS records live in TWO places.** The NS records added to the **TLD zone** (e.g. `.org`, by PIR) are the **delegation** records pointing at your name servers — distinct from NS records you'd manage inside your own hosted zone.
- **Global, not regional.** Don't expect to pick a region for Route 53. Treat its resilience as **global**, not multi-AZ-within-a-region.
- **Records ≈ record sets**, but the term you'll see in the R53 console is **record set**.

---

## 🎯 Exam Quick-Reference Recap

| Concept                   | Key fact / trigger keywords                                                                                     |
| ------------------------- | --------------------------------------------------------------------------------------------------------------- |
| **Route 53 = ?**          | AWS **managed DNS** product                                                                                     |
| **Two services**          | (1) **Register domains**, (2) **Hosted zones** (DNS-as-a-service)                                               |
| **Scope / resilience**    | **Global** service, **single database**, **globally resilient**, region-failure tolerant, **no region to pick** |
| **Name servers per zone** | **Generally four** managed name servers                                                                         |
| **Zone file**             | DB containing all DNS info for a domain; called a **hosted zone** in R53                                        |
| **Registry**              | Manages a **single TLD** zone (delegated by **IANA**, who runs the **root zone**)                               |
| **PIR**                   | The **`.org`** registry                                                                                         |
| **Delegation mechanism**  | **NS records** added to the **TLD zone** point at your name servers → make them **authoritative**               |
| **Public hosted zone**    | On the **public internet**, part of public DNS system                                                           |
| **Private hosted zone**   | Linked to **VPC(s)**, internal only — for **sensitive** records                                                 |
| **Records**               | Stored in a hosted zone; shown as **record sets** in R53                                                        |

### Mnemonic _(editorial addition — mine)_

> **"Route 53 does R & H: Register & Host."**
> 
> - **R**egister domains (talk to registries / IANA-delegated TLD owners)
> - **H**ost zones (public on the internet, or private in a VPC)