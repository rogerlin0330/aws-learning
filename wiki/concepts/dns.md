---
title: DNS — Domain Name System
type: concept
tags: [dns, networking, split-horizon-dns, split-view-dns, ttl, cname, alias]
updated: 2026-05-29
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
  │      └─ returns TLD nameserver for ".com", ".org", etc.
  │
  ├─ 4. TLD Server  ← e.g. Verisign for .com
  │      └─ returns the authoritative nameserver for the domain
  │
  └─ 5. Authoritative DNS Server (Nameserver)
         └─ returns the actual DNS record (A, CNAME, MX, …)
```

The recursive resolver handles steps 3–5 on behalf of the client, caches the result, and returns the answer. The client only ever talks to the recursive resolver.

## Caching and TTL

Every DNS record includes a **TTL (Time To Live)** — a value in seconds that tells caches how long to hold the record before re-querying.

| Cache layer | Who controls TTL |
| --- | --- |
| Browser | TTL from the record (capped by browser) |
| OS resolver | TTL from the record |
| Recursive resolver (ISP) | TTL from the record |
| Authoritative DNS | You set it on the record |

Lowering TTL before a migration gives you faster propagation (stale records expire sooner). Raising it for stable records reduces resolver load.

> [!warning] Gotcha
> TTL propagation is **not instantaneous**. If you lower TTL from 86400 s (1 day) to 60 s, you must wait the full 24 hours for all resolvers to re-query before the short TTL takes effect.

## DNS Record Types

| Record | Purpose | Example |
| --- | --- | --- |
| A | Domain → IPv4 | `example.com → 93.184.216.34` |
| AAAA | Domain → IPv6 | `example.com → 2606:2800:…` |
| CNAME | Alias → another domain name (subdomains only) | `www → example.com` |
| MX | Mail server for domain | `example.com → mail.example.com` |
| NS | Nameservers for a zone | points to authoritative servers |
| TXT | Free-text (SPF, DKIM, domain verification) | `"v=spf1 include:…"` |
| SOA | Start of Authority — zone metadata | one per zone, auto-created |
| PTR | Reverse DNS (IP → domain) | used for email reputation |

> [!warning] Gotcha
> A **CNAME cannot be used at the zone apex** (`example.com` itself, also called the naked domain) — DNS standards forbid it. Only subdomains (e.g. `www.example.com`) can have CNAMEs. [Route 53](../services/route53.md) solves this with its proprietary **Alias record**.

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

> [!tip] Exam Tip
> When a question mentions *"connect via the public DNS name"* alongside *"VPC peering"*, *"hybrid network"*, or *"keep traffic off the internet gateway"*, the answer leans on split-horizon DNS — internal callers automatically resolve to the private address.

## Authoritative DNS and Redundancy

The **authoritative DNS server** (nameserver) is the source of truth for a domain's records. All updates happen here. Domains typically have multiple nameservers so that resolution continues if one is unreachable.

## References

- [Route 53](../services/route53.md) — AWS authoritative DNS service
- [Networking Concepts](networking.md)
- [TLS/HTTPS](tls-https.md)
- Source: [How Does the DNS Process Work?](https://www.linkedin.com/pulse/how-does-dns-process-work-mohamed-ayman-elshazly-/)
