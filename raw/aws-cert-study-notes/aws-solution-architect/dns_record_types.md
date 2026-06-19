# DNS Record Types

DNS stores different **record types**, each serving a distinct function. The exam tests recognition of each type's purpose, plus a few classic trick questions (especially around CNAME and MX). The lesson closes with **TTL**, a concept essential to avoiding real-world DNS problems.

---

## NS — Nameserver Records

**Purpose:** The record type that makes **delegation** possible in DNS. NS records are how one zone hands off responsibility for a subdomain to another set of servers.

![[dns-ns-records.png]]

**How to read the diagram:**

- **Left box (`.com zone`):** Contains the NS records. These records (e.g. `amazon.com NS pdns1.ultradns.net.`) live _inside_ the parent zone and point outward.
- **Arrows → right box:** Each NS record points at a server that hosts the child zone.
- **Right box (`amazon.com zone`):** The delegated zone, hosted on those nameservers, which holds the actual **RECORDS** (www, mail, etc.).

**Delegation chain (end-to-end):**

- The **root zone** delegates `.com` by holding NS records pointing at the `.com` nameservers (managed by VeriSign).
- The **`.com` zone** delegates `amazon.com` by holding NS records pointing at the Amazon-managed nameservers.
- Those nameservers host the `amazon.com` zone and its records.

> 🎯 **Exam angle:** NS records are how delegation works _both ways_ through the hierarchy — root → TLD → domain. If a question describes "how a parent zone points at the servers hosting a child zone," the answer is **NS records**.

---

## A and AAAA Records

**Purpose:** Map a **hostname → IP address** (Host to IP). Both do the same job; they differ only in the IP version returned.

![[dns-a-aaaa-records.png]]

|Record|Maps host to|IP version|
|---|---|---|
|**A**|IPv4 address (e.g. `172.217.25.36`)|IPv4|
|**AAAA**|IPv6 address (e.g. `2404:6800:4006:802::2004`)|IPv6|

_(The IP values above are format **examples** of each pattern, not values to memorize.)_

**How to read the diagram:** Inside the `google.com` zone, the host `www` has both an `A` record (→ IPv4) and an `AAAA` record (→ IPv6).

**Common practice:** Admins typically create **two records with the same name** — one A and one AAAA. The client OS / DNS software then picks the address type it supports (AAAA if IPv6-capable, otherwise A).

> 🎯 **Mnemonic (editorial addition — keep or cut):** **AAAA = 4× the letters = IPv_6_** isn't a perfect fit, but the easy anchor is "**A → v4, the longer AAAA → the longer v6 address**."

---

## CNAME Records

**Purpose:** Create **host-to-host** records — the DNS equivalent of a **shortcut/alias** pointing one name at another name.

![[dns-cname-records.png]]

**How to read the diagram:** An `A` record `server` points at an IPv4 address. Three CNAMEs — `ftp`, `mail`, `www` — all point at `server`, so they all resolve to the same IPv4 address.

**Why use them:** **Reduce admin overhead.** A single server often runs multiple roles (FTP, mail, web). If the server's IP changes, you update **only the A record** — every CNAME referencing it follows automatically.

> ⚠️ **Trap / classic exam trick:** A **CNAME cannot point directly at an IP address — only at another name.** Expect a question designed to catch this.

---

## MX Records

**Purpose:** Let a sending mail server **find the mail server (SMTP) for a domain.** Critical to how internet email works.

![[dns-mx-records.png]]

**Structure — two parts:** `MX <priority> <value>`

```
MX 10 mail
MX 20 mail.other.domain.
```

**The value — host vs FQDN (key distinction):**

- **No trailing dot** (e.g. `mail`) → treated as a **host inside the same zone**, so `mail` means `mail.google.com`.
- **Trailing dot** (e.g. `mail.other.domain.`) → a **Fully Qualified Domain Name (FQDN)**, which can point inside _or outside_ the zone (e.g. an external provider like Office 365).

> ⚠️ **Trap:** The A-record name the MX points to is arbitrary — it could be `mail`, `rabbits`, `apple`, or `fluffy`. The **name doesn't matter** to how MX works.

**Priority — lower value = higher priority:**

- `MX 10 mail` is used **first**; `MX 20 mail.other.domain.` is the fallback if the primary is non-functional.
- **Equal priority** → any of them may be selected.

**Email delivery flow:**

1. Sending server reads the **domain** from the "to" address (e.g. `hi@google.com` → `google.com`).
2. Performs an **MX query** via DNS (root → `.com` → `google.com`), retrieving the MX records.
3. Uses **priority** to choose a record, resolves it to an IP, and connects to the mail server via **SMTP** to deliver.

> 🎯 **Exam angle:** Trigger phrase "find the mail server for a domain" / "where to deliver email" = **MX record**. Lower priority number wins.

---

## TXT Records

**Purpose:** Add **arbitrary text** to a domain. Extends DNS with extra functionality.

![[dns-txt-records.png]]

**Common use cases:**

- **Prove domain ownership** (most common): An external system (Google Workspace, Office 365, Amazon WorkMail) tells you to add a TXT record with a specific string (e.g. `cats are the best`). The admin adds it; the external party then **queries the TXT record** and, if it matches, ownership is confirmed.
- **Anti-spam (e.g. SPF):** Specify which entities are authorized to send email on the domain's behalf. Mail from unlisted servers is a strong spam indicator.

**How to read the diagram:** An admin **adds** a TXT record (`cats are the best`) into the `animals4life.org` zone → an external verifier **queries TXT** → match returns **OK**, proving ownership.

---

## TTL — Time To Live

**Purpose:** A numeric value (in **seconds**) set on DNS records controlling **how long a record can be cached** by resolvers.

![[dns-ttl.png]]

**Authoritative vs Non-Authoritative (key concept):**

|Term|Source|Speed|Accuracy|
|---|---|---|---|
|**Authoritative**|The nameserver that is the source of truth for the domain (e.g. `amazon.com`'s nameserver)|Slower (must "walk the tree")|Always accurate — single source of truth, **preferred**|
|**Non-Authoritative**|A **cached** copy at a resolver|Faster (immediate)|Usually matches authoritative, but can be stale|

**How to read the diagram:**

1. **Client → Resolver** (hosted at the ISP). Resolver walks the tree: **Root → `.com` → `amazon.com`** nameserver, retrieving the `www` record. This is the slow, full path; the result is an **authoritative answer**.
2. The `amazon.com` admin set **TTL 3600** (1 hour) on the record. The resolver **caches** the result for 3600 seconds.
3. The **next client** querying the same record gets a **non-authoritative** answer — served instantly from the resolver's cache.

**TTL is a balance:**

- **Low TTL** → more queries against your nameservers, but **more control** (changes propagate fast).
- **High TTL** → fewer queries, but **less control** — old cached values persist after you change a record.

> ⚠️ **Real-world trap:** Migrating email to a new provider with a **high MX TTL** can **delay/break delivery**, because old MX IPs stay cached and keep getting used.

> 🎯 **Best practice:** Before any DNS change (migrations, upgrades), **lower the TTL well in advance** — sometimes days/weeks ahead — to minimize caching issues when you cut over. You can also leave TTLs permanently low.

> ⚠️ **Gotcha:** A resolver _should_ obey TTL but **may ignore it** — the resolver's admin can override this locally. DNS is a frequent cause of project failures specifically because of TTL/caching behavior.

---

## ⭐ Exam Quick-Reference Recap

|Record|Maps / Does|Trigger keywords|
|---|---|---|
|**NS**|Delegation — parent zone → child zone's nameservers|"delegation," "points at the servers hosting the zone"|
|**A**|Host → **IPv4**|"hostname to IPv4"|
|**AAAA**|Host → **IPv6**|"hostname to IPv6"|
|**CNAME**|Host → **another host (name)**|"alias," "shortcut," "reduce admin overhead" — ⚠️ **never points at an IP**|
|**MX**|Find the **mail server (SMTP)** for a domain|"mail server," "email delivery"; **lower priority # = preferred**; trailing dot = FQDN|
|**TXT**|Arbitrary text on a domain|"prove domain ownership," "verification," "SPF / anti-spam"|
|**TTL**|Seconds a record can be **cached**|"caching," "authoritative vs non-authoritative," "lower before migration"|

**Top traps to lock in:**

- ⚠️ **CNAME → name only, never an IP.**
- ⚠️ **MX: lower priority number = higher priority.** Trailing dot = FQDN (can be external).
- ⚠️ **TTL too high before a change = stale cache problems.** Lower it in advance.