---
title: DNS — Domain Name System
type: concept
tags: [dns, networking, split-horizon-dns, split-view-dns, ttl, cname, alias, ns, mx, txt, a-record, aaaa-record, delegation, authoritative]
updated: 2026-06-19
---

# DNS — Domain Name System

DNS translates human-readable domain names (e.g. `example.com`) into IP addresses that routers can forward to. It underpins every internet request. The AWS implementation is [Route 53](../services/route53.md).

See [Networking Concepts](networking.md) for broader networking context, and [TLS/HTTPS](tls-https.md) for how DNS and TLS work together to establish a secure connection.

## Resolution Chain

When you navigate to a domain, DNS resolution follows a strict delegation chain — but caching short-circuits most of it in practice:

```text
Client
  │
  ├─ 1. Local cache (browser + OS)  ← checked first
  │
  ├─ 2. Recursive Resolver (ISP or custom, e.g. 8.8.8.8)
  │      └─ also has a local cache
  │
  ├─ 3. Root DNS Servers  ← ~13 logical roots, anycast globally
  │      └─ returns Top-Level Domain (TLD) nameserver for ".com", ".org", etc.
  │
  ├─ 4. TLD Server  ← e.g. Verisign for .com
  │      └─ returns the authoritative nameserver for the domain
  │
  └─ 5. Authoritative DNS Server (Nameserver)
         └─ returns the actual DNS record (A, CNAME, MX, …)
```

The recursive resolver handles steps 3–5 on behalf of the client, caches the result, and returns the answer. The client only ever talks to the recursive resolver.

![DNS resolution walkthrough: a user computer queries the ISP recursive resolver for www.example.com; the resolver asks the root server (which points to the .com TLD server), then the TLD server (which points to the authoritative server), then the authoritative server (which returns 217.64.213.12); the resolver finally returns that IP to the user](../assets/how-dns-works.png)

## Caching and TTL

Every DNS record includes a **TTL (Time To Live)** — a value in seconds that tells caches how long to hold the record before re-querying.

| Cache layer | Who controls TTL |
| --- | --- |
| Browser | TTL from the record (capped by browser) |
| OS resolver | TTL from the record |
| Recursive resolver (ISP) | TTL from the record |
| Authoritative DNS | You set it on the record |

Lowering TTL before a migration gives you faster propagation (stale records expire sooner). Raising it for stable records reduces resolver load.

### Authoritative vs Non-Authoritative Answers

TTL is what creates the distinction between the two kinds of DNS answer:

| Term | Source | Speed | Accuracy |
| --- | --- | --- | --- |
| **Authoritative** | The nameserver that is the source of truth for the domain | Slower — the resolver must "walk the tree" | Always current; the single source of truth, and the preferred answer |
| **Non-authoritative** | A **cached** copy held at a resolver | Faster — served immediately | Usually correct, but can be **stale** until the TTL expires |

![TTL caching: a client queries the resolver, which walks root, .com, then the amazon.com nameserver for an authoritative answer with TTL 3600; the resolver caches it, and the next client gets a fast non-authoritative answer from cache](../assets/dns-ttl.png)

The first query for a record walks the full [resolution chain](#resolution-chain) and returns an **authoritative** answer. The resolver then caches it for the record's TTL (1 hour for `TTL 3600`), so the **next** client querying the same name gets a **non-authoritative** answer served instantly from cache. TTL is the balance: a **low** TTL means more queries against your nameservers but fast propagation when you change a record; a **high** TTL means fewer queries but stale values linger after a change.

> [!warning] Gotcha
> TTL propagation is **not instantaneous**. If you lower TTL from 86400 s (1 day) to 60 s, you must wait the full 24 hours for all resolvers to re-query before the short TTL takes effect. This is why best practice is to **lower TTLs well in advance** of any migration — sometimes days ahead. The classic failure: migrating email to a new provider while the **MX TTL is high**, so resolvers keep using the old cached MX values and mail delivery breaks until the cache expires.

> [!warning] Gotcha
> A resolver *should* honour TTL but is not forced to — its operator can override the value locally. TTL/caching behaviour is a frequent cause of real-world DNS problems precisely because you don't fully control downstream caches.

## DNS Record Types

A DNS zone stores several **record types**, each with a distinct job. The summary below is the quick reference; the subsections that follow add the detail and exam traps for the types that get tested.

| Record | Purpose | Example |
| --- | --- | --- |
| A | Domain to IPv4 | `example.com → 93.184.216.34` |
| AAAA | Domain to IPv6 | `example.com → 2606:2800:…` |
| CNAME | Alias to another domain name (subdomains only) | `www → example.com` |
| MX | Mail server for domain | `example.com → mail.example.com` |
| NS | Nameservers for a zone | points to authoritative servers |
| TXT | Free-text (SPF, DKIM, domain verification) | `"v=spf1 include:…"` |
| SOA | Start of Authority — zone metadata | one per zone, auto-created |
| PTR | Reverse DNS (IP to domain) | used for email reputation |

### NS — Delegation

**NS (Nameserver) records are what make delegation possible.** They are how one zone hands off responsibility for a child zone to another set of nameservers. The NS records live *inside the parent zone* and point outward at the servers that host the child zone — the child zone then holds the actual records (`www`, `mail`, and so on).

![NS-record delegation: the .com zone holds NS records such as `amazon.com NS pdns1.ultradns.net.` that point at the nameservers hosting the amazon.com zone, which in turn holds the real www/mail records](../assets/dns-ns-records.png)

This is the same mechanism at every level of the [resolution chain](#resolution-chain): the root zone delegates `.com` by holding NS records for the `.com` nameservers (VeriSign), and the `.com` zone delegates `amazon.com` by holding NS records for Amazon's nameservers. Delegation works **both ways down the hierarchy** — root, then TLD, then domain.

### A and AAAA — Hostname to IP

Both map a **hostname to an IP address**; they differ only in IP version. `A` returns an **IPv4** address, `AAAA` returns an **IPv6** address.

![A and AAAA records inside the google.com zone: the host `www` has an A record pointing at an IPv4 address and an AAAA record pointing at an IPv6 address](../assets/dns-a-aaaa-records.png)

Admins commonly create **two records with the same name** — one A and one AAAA. The client's OS or DNS software then picks the address type it supports (AAAA when IPv6-capable, otherwise A). A useful anchor: the short name `A` maps to v4, the longer name `AAAA` maps to the longer v6 address.

### CNAME — Host-to-Host Alias

A **CNAME** maps one name to **another name** — the DNS equivalent of a shortcut or alias. In the diagram below, an `A` record `server` points at an IPv4 address, and three CNAMEs (`ftp`, `mail`, `www`) all point at `server`, so all four names resolve to the same IP.

![CNAME records: an A record `server` points at an IPv4 address; the CNAMEs ftp, mail, and www all point at `server`, so they resolve to the same address](../assets/dns-cname-records.png)

The payoff is **reduced admin overhead**: one host often runs multiple roles (FTP, mail, web). If its IP changes you update **only the A record**, and every CNAME pointing at it follows automatically.

> [!warning] Gotcha
> A **CNAME can only point at another name — never directly at an IP address.** This is a classic exam trick. Related trap: a **CNAME cannot be used at the zone apex** (`example.com` itself, also called the naked domain) — DNS standards forbid it; only subdomains (e.g. `www.example.com`) can have CNAMEs. [Route 53](../services/route53.md) solves the apex case with its proprietary **Alias record**.

### MX — Mail Servers

**MX (Mail Exchange) records let a sending mail server find the SMTP server for a domain** — they are how internet email is routed. Each record has two parts: `MX <priority> <value>`.

![MX records in the google.com zone with priority values, pointing at a `mail` host inside the zone and an external FQDN, which resolve via A records to mail-server IPs](../assets/dns-mx-records.png)

Two details carry most of the exam weight:

- **The value: host vs FQDN.** A value with **no trailing dot** (e.g. `mail`) is treated as a host *inside the same zone*, so `mail` means `mail.google.com`. A value **with a trailing dot** (e.g. `mail.other.domain.`) is a **Fully Qualified Domain Name** and can point inside *or outside* the zone — e.g. an external provider like Office 365. The host name the MX points at is arbitrary (`mail`, `apple`, `fluffy` — it does not matter to how MX works).
- **Priority: lower value wins.** `MX 10 mail` is tried before `MX 20 mail.other.domain.`; the higher number is the fallback. Equal priorities mean any may be chosen.

Delivery flow: the sending server reads the domain from the recipient address (`hi@google.com` gives `google.com`), performs an MX query through DNS, picks the record with the lowest priority number, resolves it to an IP, and connects over SMTP to deliver.

> [!tip] MX is the *receiving* side
> The MX record is published by the domain that **receives** mail — it advertises "deliver mail for me here." The description above is from the *remote sender's* lookup perspective, but the record exists for inbound delivery: no MX record, no incoming mail. There is **no DNS record for a domain's outbound/sending server** — that is client configuration. DNS's only role on the sending side is *authorization* (SPF/DKIM/DMARC in [TXT records](#txt--arbitrary-text)), not routing.

The mail pipeline is a chain of **four agents** ([RFC 5598](https://datatracker.ietf.org/doc/html/rfc5598)) — **MUA** (Mail User Agent, your client), **MSA** (Mail Submission Agent), **MTA** (Mail Transfer Agent), and **MDA** (Mail Delivery Agent) — and MX governs only the MTA-to-MTA hop. Whether you call it "three legs" or "four legs" just depends on whether you slice by *transport phase* or by *software agent*; the wire transactions are identical. The table maps the agents to the protocol, port, and DNS record each transition uses:

| Phase (agents) | What moves | Protocol | Typical port(s) | DNS record |
| --- | --- | --- | --- | --- |
| **Submission** (MUA to MSA) | client hands the message to your provider's submission server | SMTP submission | 587 (STARTTLS) / 465 (TLS) | none — client config |
| **Transfer / relay** (MTA to MTA) | sender's server relays across the internet to the recipient's server | SMTP | 25 | **MX** of recipient domain |
| **Local delivery** (MTA to MDA) | receiving server filters (spam/virus) and deposits into the mailbox store | internal / LMTP | — | none |
| **Retrieval** (mailbox to MUA) | recipient's client reads mail from its mailbox store | **IMAP** or **POP3** | IMAP 143 / 993, POP3 110 / 995 | none — client config |

There is **no MX equivalent for IMAP/POP3**: a client finds its mailbox server by manual configuration or autoconfiguration (vendor autodiscover, or optional `SRV` records), never by MX. Mental model: **SMTP + MX move mail *to* a mailbox; IMAP/POP3 read it *from* the mailbox** — the two never overlap. **IMAP** keeps mail on the server and syncs read/folder state across devices (the modern default); **POP3** historically downloads and deletes (single-device).

> [!warning] Gotcha
> The authentication records **SPF, DKIM, and DMARC** (all [TXT records](#txt--arbitrary-text)) are *verified by the **receiving** server*, not by the sender's MSA. On the way out, the sending infrastructure authenticates the user and DKIM-*signs* the message; the recipient's MTA then checks the connecting IP against SPF, validates the DKIM signature, and applies the DMARC policy.

### TXT — Arbitrary Text

**TXT records attach arbitrary text to a domain**, extending DNS with extra functionality. Two common uses:

- **Proving domain ownership** (the most common): an external system (Google Workspace, Office 365, Amazon WorkMail) asks you to add a TXT record containing a specific string. It then queries that TXT record and, if the value matches, confirms you control the domain.
- **Anti-spam (e.g. SPF):** a TXT record lists which servers are authorized to send email for the domain. Mail from unlisted servers is a strong spam signal.

![TXT-record ownership proof: an admin adds a TXT record into the animals4life.org zone; an external verifier queries the TXT record, matches the value, and returns OK](../assets/dns-txt-records.png)

## Split-Horizon DNS

**Split-horizon DNS** (also called *split-view DNS*) is the practice of returning **different answers for the same DNS name** depending on where the query originates — typically *inside* a private network vs *outside* on the public internet. One hostname works everywhere, but resolves to the appropriate address for the caller's network context.

This avoids two common problems:

- **Hairpin routing** — without split-horizon, internal callers resolve to a public IP and traffic leaves the private network only to come back in
- **Hostname duplication** — applications don't need separate "internal" and "external" hostnames; the same name works from both sides

### Where you see it in AWS

| Mechanism | Behaviour |
| --- | --- |
| **EC2 public DNS name** (e.g. `ec2-3-89-7-136.compute-1.amazonaws.com`) | Inside the VPC → resolves to the **private IPv4**; outside → resolves to the **public IPv4**. Built-in; no configuration required. See [EC2 — Network Interfaces (ENIs)](../services/ec2.md#split-horizon-dns--the-public-dns-name-resolves-differently-inside-vs-outside-the-vpc) |
| **Route 53 split-view hosted zones** | Same domain registered as both a **public hosted zone** *and* a **private hosted zone** associated with a VPC — VPC queries hit the private zone, internet queries hit the public zone. See [Route 53](../services/route53.md) |

> [!warning] Gotcha
> Route 53 split-view is **not** an overlay where the private zone overrides a few records and the rest fall through to public. Inside the VPC the private zone **replaces** the whole namespace: a name not present in the private zone returns **NXDOMAIN**, not the public answer. Any public name you still need internally must be **duplicated** into the private zone. See [Route 53 — How association and namespace resolution work](../services/route53.md#how-association-and-namespace-resolution-work).

> [!tip] Exam Tip
> When a question mentions *"connect via the public DNS name"* alongside *"VPC peering"*, *"hybrid network"*, or *"keep traffic off the internet gateway"*, the answer leans on split-horizon DNS — internal callers automatically resolve to the private address.

## Authoritative DNS and Redundancy

The **authoritative DNS server** (nameserver) is the source of truth for a domain's records. All updates happen here. Domains typically have multiple nameservers so that resolution continues if one is unreachable.

## References

- [Route 53](../services/route53.md) — AWS authoritative DNS service
- [Networking Concepts](networking.md)
- [TLS/HTTPS](tls-https.md)
- Source: [How Does the DNS Process Work?](https://www.linkedin.com/pulse/how-does-dns-process-work-mohamed-ayman-elshazly-/)
