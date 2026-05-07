---
title: DNS — Domain Name System
type: concept
tags: [dns, networking]
updated: 2026-05-07
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

## Authoritative DNS and Redundancy

The **authoritative DNS server** (nameserver) is the source of truth for a domain's records. All updates happen here. Domains typically have multiple nameservers so that resolution continues if one is unreachable.

## References

- [Route 53](../services/route53.md) — AWS authoritative DNS service
- [Networking Concepts](networking.md)
- [TLS/HTTPS](tls-https.md)
- Source: [How Does the DNS Process Work?](https://www.linkedin.com/pulse/how-does-dns-process-work-mohamed-ayman-elshazly-/)
