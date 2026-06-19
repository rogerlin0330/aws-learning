# Route 53 Interoperability

## Overview & Problem Context

When most people register a domain through Route 53, it looks like a single action. In reality, Route 53 performs **two conceptually distinct jobs** at the same time:

1. **Domain Registrar** — the company that registers the domain on your behalf with the relevant registry.
2. **Domain (DNS) Hosting** — allocating name servers, creating the zone file, and hosting/managing the DNS records.

**Interoperability** is the term for splitting these two jobs across different providers — using Route 53 for _one_ of these roles while a third party handles the other. The whole topic becomes easy once you firmly separate "registrar" from "hosting" in your mind.

> 🎯 **Exam angle:** The exam loves to test whether you understand that _registering_ a domain and _hosting_ a domain are independent functions. A question may describe a domain registered with one provider but managed elsewhere, and expect you to know this is valid and how the name server records connect the two.

---

## The Two Roles (Core Mental Model)

|Role|What it does|Where it lives in the R53 console|Fee type|
|---|---|---|---|
|**Registrar**|Liaises with the TLD registry; registers the domain on your behalf; gets NS delegation records added to the TLD zone|**Registered Domains**|One-off per-year (or per-3-year) registration fee|
|**DNS Hosting**|Allocates 4 name servers; creates and hosts the zone file; lets you add/manage records|**Public Hosted Zone**|Recurring **monthly** hosting fee|

> ⚠️ **Trap:** These are _different_ fees. Registration is a periodic (yearly / 3-yearly) one-off; hosting (the hosted zone) is billed **monthly**. Don't conflate them.

### Key supporting term: the Registry

The **registry** is the company/entity responsible for a specific top-level domain (TLD). For `.org`, the registry is **PIR** (Public Interest Registry). The registrar liaises with the registry to add an entry for your domain into the TLD zone, pointing at your name servers via **NS delegation records**.

---

## What Happens When You Register Normally (Both Roles via R53)

This is the default, "traditional" configuration — R53 acts as **both** registrar and host.

1. You pay R53 the **registration fee** (per-year / per-3-year).
2. The R53 **registrar** liaises with the R53 **DNS hosting** entity, which creates a **public hosted zone** and allocates **4 Route 53 name servers**.
3. Those **4 NS are returned** to the registrar.
4. The registrar passes the **registration fee, domain name, and 4 × NS** to the **`.org` (PIR) registry**.
5. The registry adds an **entry in the TLD** using **NS delegation records pointing at the 4 × NS** — the domain is now active on the public DNS system.

Billing: pay **once** (registration) to R53 the registrar, plus a **monthly** hosting fee to R53 the host.

![[R53-interoperability-both.png]]

### How to read the "Both Roles" diagram

- **Left panel (purple)** = _Route53 Registrar Role (Registered Domains)_. Bottom-left: a person **registers the domain** and pays the **p/y (per-year) fee** (pink arrow into the R53 shield).
- **Right panel (purple)** = _Route53 DNS Hosting Role (Public Hosted Zone)_. It contains the **4 Name Servers**, the **Zone File** (R53 shield) for `animals4life.org`, and the **monthly fee for hosting**.
- **Orange dotted arrow (left→right, along the bottom):** the registrar liaising with R53 hosting to **create the public hosted zone**.
- **Orange dotted arrow (right→left):** **4 × NS returned** from hosting back to the registrar.
- **Orange dotted arrow (up to the registry):** registrar provides **registration fee + domain name + 4 × NS** to the PIR registry.
- **Red dotted arrows (top, registry→name servers):** the **entry added in the TLD** — NS delegation records pointing at the 4 × NS.
- **Colour coding note:** orange/green text highlights the _registration fee_ and _NS provided_ path; red highlights the _TLD delegation_ path. The two panels being the **same colour** signals both roles belong to R53.

> 📝 _Editorial note (added for clarity — not in source):_ The arrow-colour logic (orange = R53-internal handoff + provisioning to registry; red = registry's delegation into the TLD) is the same across all three diagrams. Once you learn it here, the other two diagrams read the same way with only the panel ownership swapped.

---

## Configuration 2 — Route 53 as Registrar Only

Here R53 registers the domain, but a **third party (e.g. Hover) hosts** the zone file and name servers.

- You still pay R53 to register, and R53 still liaises with the TLD registry.
- **You** (the DNS admin) create a hosted zone **at the third party**, which charges a **monthly** fee and gives you the details of **its** name servers.
- You pass those **external NS details to R53** (the registrar).
- R53 sets the **NS records in the TLD** to point at the **third-party (Hover) name servers**.

Result: the **domain** is managed via R53, but the **zone file and records** live with the third-party host.

![[R53-interoperability-registrar-only.png]]

### How to read the "Registrar Only" diagram

- **Left panel (purple)** = _Route53 Registrar Role (Registered Domains)_ — same registrar setup as before (person registers, pays p/y fee into the R53 shield).
- **Right panel (green/teal)** = _3rd Party Domain Hosting_ — note the **colour change**: hosting is now external. It holds the **Name Servers**, the **Zone File** (now a **Hover** shield), `animals4life.org`, and the **monthly hosting fee**.
- **Green solid arrow (right→left, into the central person):** the admin **retrieves the external NS details** from Hover...
- **...green solid arrow continuing into the registrar / orange dotted arrow up:** ...and passes those **4 × external NS** to R53, which provides **registration fee + domain name + 4 × external NS** to the registry.
- **Red dotted arrows (top, registry→Hover name servers):** TLD entry with NS delegation records pointing at the **4 × external (Hover) NS**.
- **Caption metaphor:** "the domain is registered via R53, but the NS records are set to externally hosted Name Servers."

> ⚠️ **Trap / real-world note:** The instructor says this is rarely seen in the wild — arguably the **worst** way to manage domains. R53's registrar function is "nothing special"; the real value of R53 is its **DNS hosting**, which this architecture _doesn't_ use.

---

## Configuration 3 — Route 53 as Hosting Only

The more popular split: a **third party (e.g. Hover) registers** the domain, but **R53 hosts** the zone.

Why you'd do this:

- The business must register via a third party (existing deal / discount), **or**
- Domains were **historically registered** elsewhere and you now want R53's DNS hosting benefits.

Flow:

1. Domain is registered via the **third-party registrar (Hover)**, who liaises with the TLD registry.
2. You **create a public hosted zone in R53** (either during registration or later) — this creates the zone + 4 name servers, for a **monthly fee**.
3. You **pass the R53 NS details to the registrar (Hover)**.
4. The registrar passes them to the registry; the **NS records are added to the TLD** → the R53 hosted zone is now live.

> 🎯 **Exam angle:** This can be done **at registration time** _or_ **after the fact** — e.g. an old domain registered years ago that you migrate to R53 hosting by creating a hosted zone and updating the NS records at the existing registrar.

![[R53-interoperability-hosting-only.png]]

### How to read the "Hosting Only" diagram

- **Left panel (green/teal)** = _3rd Party Domain Registrar_ — note the **colour change on the left**: the registrar is now external, shown as a **Hover** logo box (the R53 shield is gone). Person registers + pays p/y fee to Hover.
- **Right panel (purple)** = _Route53 DNS Hosting Role (Public Hosted Zone)_ — back to R53 ownership: 4 Name Servers, Zone File (R53 shield), `animals4life.org`, monthly hosting fee.
- **Purple dotted arrows (bottom, right→centre→left):** **create a R53 public hosted zone**, then **pass the allocated NS details to the registrar (Hover)**.
- **Green/purple dotted arrow up to registry:** Hover provides **registration fee + domain name + 4 × NS** to the registry.
- **Red dotted arrows (top, registry→R53 name servers):** TLD entry with NS delegation records pointing at the **4 × (R53) NS**.
- **Caption:** "Using the Registrar, NS records for domain can also be updated AFTER registration if you decide to host records via R53."

> 📝 _Editorial note (added — not in source):_ The visual "tell" across all three diagrams is **panel colour = ownership**. Purple = Route 53; green/teal (+ Hover branding) = third party. Whichever side is green is the part R53 is _not_ doing.

---

## The Universal Pattern (All Three Architectures)

Regardless of who plays which role, the end state is always the same:

> The **TLD registry** holds an entry for the domain containing **NS delegation records** pointing at **4 name servers** that host the **zone file** — and DNS resolution works through that delegation.

What changes between configurations is only **who** registers (provides those NS to the registry) and **who** hosts (operates the name servers / zone file).

---

## Exam Quick-Reference Recap

**Trigger keywords → answer:**

- _"Register a domain but use the DNS hosting elsewhere / existing registrar"_ → **R53 Hosting Only** (create public hosted zone in R53, give NS to the external registrar). ✅ Common, valid.
- _"Register with R53 but host the zone/records on a third party"_ → **R53 Registrar Only**. ⚠️ Valid but rare/discouraged.
- _"Migrate an old externally-registered domain to R53 DNS"_ → create R53 **public hosted zone**, update **NS records** at the existing registrar. (Hosting Only, done _after_ registration.)
- _Registration fee_ = **per-year / per-3-year, one-off**, paid to the **registrar**.
- _Hosting fee_ = **monthly**, paid to the **hosting** provider (per hosted zone).
- _"4 name servers" + "NS delegation records" + "TLD entry"_ = the link between registrar and host; always present.
- _Registry for `.org`_ = **PIR**. The **registry** owns the TLD zone; the **registrar** liaises with it; the **host** runs the name servers.

**One-line mnemonic (editorial addition):**

> 📝 **"Registrar Registers, Host Hosts — delegation (NS) is the handshake between them."**

---

## ⚠️ Common Traps Summary

- ❌ Treating "register a domain" as a single indivisible action → it's **two** roles.
- ❌ Confusing the **registry** (owns the TLD) with the **registrar** (registers on your behalf). Different entities.
- ❌ Mixing up the **one-off (yearly) registration fee** with the **monthly hosting fee**.
- ❌ Assuming R53 must do both — it can do **either one** independently, with a third party doing the other.
- ❌ Thinking Registrar-Only gives you R53's main value — it does **not**; the DNS **hosting** is where R53's value is.