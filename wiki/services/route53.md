---
title: Route 53
type: service
tags: [dns, route53, networking, health-checks, routing-policy, weighted-routing, latency-routing, geolocation-routing, geoproximity-routing, multi-value-routing, failover-routing, registrar, dns-hosting, interoperability, dnssec, dnssec-signing, dnssec-validation]
certs: [aws-saa]
updated: 2026-06-23
---

# Route 53

Route 53 is AWS's **authoritative DNS service** and domain registrar. It is **globally resilient** — name servers are distributed across multiple regions and share the same dataset, so the service remains available even during regional failures. See [DNS Concepts](../concepts/dns.md) for the underlying DNS fundamentals.

Route 53 integrates tightly with [TLS/HTTPS](../concepts/tls-https.md) (ACM certificate validation uses DNS TXT records) and with [VPC](vpc.md) (private hosted zones and the built-in Resolver).

## Two Services & Global Resilience

At its core Route 53 is AWS's **managed DNS** product, bringing the abstract [DNS](../concepts/dns.md) building blocks (registries, zones, name servers, delegation) into an AWS-native service. It does **two conceptually separate jobs**:

1. **Domain registration** — Route 53 acts as a **registrar** with relationships to all the major TLD **registries**, registering domains on your behalf. Covered in depth under [Registrar vs DNS Hosting](#registrar-vs-dns-hosting-interoperability).
2. **Hosted zones** — Route 53 hosts your zone files on the **AWS-managed name servers** it provides. This is DNS-as-a-service. Covered under [Hosted Zones](#hosted-zones).

These two jobs are related but independent, and either can be handed to a third party — that split is the whole point of the [interoperability](#registrar-vs-dns-hosting-interoperability) section below.

### Global service characteristics

Route 53 is one of very few AWS services that runs as a **single global service backed by a single database**, rather than as independent per-region deployments:

- **No region to pick** — the console has no region selector for Route 53.
- Its data is **distributed globally as a single set** and **replicated between regions**, making it **globally resilient**.
- It can **tolerate the failure of one or more regions** and keep resolving.

> [!tip] Exam Tip
> "Globally resilient," "single global service," and "no region to pick" are Route 53 trigger phrases. Contrast with most AWS services, which are **region-scoped** — see [Resource Scope](../concepts/resource-scope.md).

### Domain registration mechanics

When you register a domain through Route 53, four things happen behind the scenes:

1. **Availability check** — Route 53 checks with the **registry for that TLD** (for example **PIR** for `.org`) whether the name is free.
2. **Create the zone file** — Route 53 creates a **zone file** (a database of all the domain's DNS records). In Route 53 terminology this becomes a **hosted zone**.
3. **Allocate name servers** — Route 53 creates and manages name servers distributed globally — **generally four per zone** — and places the zone file onto them.
4. **Liaise with the TLD registry** — Route 53 gets **NS (name server) delegation records** added into the **TLD's zone file**, pointing back at those four name servers. This delegation is what makes your name servers **authoritative** for the domain.

![Route 53 acting as registrar: the purple Route 53 icon top-left has relationships with the .com/.io/.net registries (red arrow to the Registries box) and contacts the .org registry (PIR) to register animals4life; Route 53 creates the animals4life.org zone file and four managed name servers (dotted arrows), places the zone file onto the four servers (red arrow), and PIR adds NS delegation records into the .org TLD zone pointing back at those four name servers](../assets/R53-function-register-domains.png)

> [!note] DNS is a system of delegation
> The end state is always the same: the **TLD registry holds NS records pointing at the name servers that host your zone file**. Resolution then flows down that delegation chain ([DNS Resolution Chain](../concepts/dns.md#resolution-chain)). Registering a domain is just arranging that delegation.

## Hosted Zones

A **hosted zone** is a DNS database for a specific domain — the container for all its DNS records. Route 53 provides both the **zone** and the **hosting**: when a hosted zone is created, a set of **managed name servers is allocated and linked to it**, so a hosted zone and its name servers are effectively one and the same. As above, you generally get **four** managed name servers per zone.

![Route 53 hosted zones summary: a hosted zone is a zone file hosted in AWS on four managed name servers (shown stacked on the right inside a RECORDS box); it can be public (reachable on the internet) or private (linked to one or more VPCs), and it stores DNS records, also called record sets](../assets/R53-function-hosted-zones.png)

> [!note] Records vs record sets
> Inside the Route 53 console, DNS records are labelled **record sets**. There is a tiny technical distinction (a record set groups records of the same name and type), but for study purposes treat **records ≈ record sets** — they refer to the same thing.

### Zone vs hosting — two separable things

A working DNS zone needs **two distinct ingredients**, and a hosted zone bundles both:

- **The zone (the data)** — the **zone file** itself: the database holding all your records (A, CNAME, MX, NS, SOA, …). This is the *content*.
- **The hosting (the servers)** — the actual **name server machines**, running DNS software and reachable on the internet, that *serve* that zone file in response to queries. This is the *infrastructure*.

Historically these were two separate jobs: you authored a zone file, then found name servers to put it on. A Route 53 hosted zone gives you both at once, and you **cannot have one without the other** — the four name servers are allocated and loaded with the zone file automatically, and they cannot be detached. That is what "Route 53 provides both the zone and the hosting" means.

### Three things called "NS"

The word "name server" / "NS" is overloaded, which is the usual source of confusion. Keep **three** meanings apart:

| "NS" meaning | What it is | Where it lives | Route 53's role |
| --- | --- | --- | --- |
| Name server **machines** | The 4 servers that actually serve the zone | AWS global infrastructure | Provides them — this *is* "the hosting" |
| NS **record set** | A record inside the zone that lists those 4 server hostnames | Inside **your** hosted zone | Auto-created with the zone |
| NS **delegation record** | The pointer that lets resolvers *find* your zone | In the **parent TLD** zone (e.g. `.org`) | Added during registration, via the registrar |

The first two travel with the hosted zone itself. The **third lives up in the TLD zone** and is the [delegation handshake](#registrar-vs-dns-hosting-interoperability) that connects a *registered domain* to *this hosted zone* — the same point made in [Domain registration mechanics](#domain-registration-mechanics).

> [!warning] Gotcha
> "Route 53 provides the zone and the hosting" refers only to the first two rows — the zone file and the four name servers. It does **not** include the delegation NS record in the TLD; that is a separate step that the **registrar** arranges. A hosted zone with no matching TLD delegation is perfectly valid but unreachable from the public internet — nothing points resolvers at its four name servers. See [interoperability](#registrar-vs-dns-hosting-interoperability).

### Public Hosted Zone

Accessible from the public internet and from any VPC via the Route 53 Resolver.

![Route 53 public hosted zone architecture showing 4 name servers, VPC resolver, and internet access](../assets/R53_public_hosted_zone_infographic.png)

- Route 53 allocates **4 public name servers** per hosted zone
- To make Route 53 authoritative for a domain, update the **NS records at the registrar** (or TLD) to point to these 4 servers
- Works with domains registered externally (GoDaddy, Hover, etc.) — update NS at the registrar, Route 53 handles the rest
- VPC resources resolve public zone records via the **VPC +2 resolver** (see Route 53 Resolver below)

**Cost:** monthly per hosted zone + small per-query fee; Alias records pointing to AWS resources are **free**

![DNS resolution walk for a public hosted zone: client → recursive resolver → root → TLD → Route 53 name servers](../assets/R53_public_hosted_zones.png)

### Private Hosted Zone

Accessible **only from explicitly associated VPCs** — never from the public internet.

![Route 53 private hosted zone architecture showing VPC association and restricted DNS resolution](../assets/R53_private_hosted_zone_infographic.png)

- Must be associated with one or more VPCs (same or different AWS accounts)
- **Cross-account association**: supported, but **CLI/API only** — not available in the AWS Console
- Requires the VPC to have `enableDnsSupport` = `true` (see [VPC](vpc.md#dns-in-vpc))

> [!tip] Exam Tip
> Private hosted zones are the right answer for **internal service discovery** — microservices resolving each other by name within a VPC without exposing anything publicly.

### Split-View (Split-Horizon) DNS

You can create both a **public** and a **private** hosted zone for the **same domain name**:

- Queries from inside an associated VPC → private zone records (internal content)
- Queries from the public internet → public zone records (public content only)

> [!tip] Exam Tip
> If a question describes needing **different DNS answers for the same domain depending on whether the query comes from inside or outside a VPC**, the answer is Split-View DNS.

> [!warning] Gotcha: a private zone *replaces* the namespace inside the VPC — it does not overlay it
> The most common split-view mistake is assuming the private zone only needs the records you want to *override*, and that everything else "falls through" to the public zone. It does not. Once a private hosted zone matches the query name, the Resolver answers **only** from that private zone; a missing record returns **NXDOMAIN**, never the public answer (see [How association and namespace resolution work](#how-association-and-namespace-resolution-work)).
>
> **Worked example.** You run a private and a public hosted zone, both named `example.com`, and associate the private zone with your VPC. You want split-horizon for `www` but no split for `customer`:
>
> | Query from inside the VPC | In private zone? | Result |
> | --- | --- | --- |
> | `www.example.com` | yes (employee portal) | employee portal — the split you wanted |
> | `customer.example.com` | **no** (only in public zone) | **NXDOMAIN** — *not* the public record |
>
> The private `example.com` zone is a **parent match** for `customer.example.com`, so it claims the name even though no record exists for it. Per the AWS docs: *"If there's a matching private hosted zone but there's no record that matches the domain name and type in the request, VPC Resolver doesn't forward the request to a public DNS resolver. Instead, it returns NXDOMAIN."*
>
> **Fix:** duplicate `customer.example.com` into the private zone pointing at the **same value as the public record**. It is not a split — both zones return the same answer — but the name will not resolve inside the VPC unless the private zone carries it. An apex-level private zone must list **every** `example.com` name you want reachable from the VPC, not just the ones you intend to override.

## Registrar vs DNS Hosting (Interoperability)

Registering a domain *looks* like one action, but Route 53 is really doing **two conceptually separate jobs** — and either one can be handed to a third party. Keeping them apart is the whole topic.

| Role | What it does | R53 console home | Fee |
| --- | --- | --- | --- |
| **Registrar** | Liaises with the TLD **registry**; registers the domain on your behalf; gets the **NS delegation records** added to the TLD zone | **Registered Domains** | One-off per-year (or per-3-year) registration fee |
| **DNS Hosting** | Allocates the **4 name servers**, creates and hosts the **zone file**, lets you manage records — this is exactly a [public hosted zone](#public-hosted-zone) | **Public Hosted Zone** | Recurring **monthly** hosting fee |

Three more terms keep the roles straight:

- **Registry** — the entity that owns a specific TLD's zone (for `.org` it is **PIR**, Public Interest Registry). It is *not* the registrar.
- **Registrar** — a company authorised to register domains *with* a registry on customers' behalf.
- **NS delegation records** — the entry the registry adds to the TLD zone, pointing the domain at its 4 name servers. This is the **handshake** that links registrar to host, and it is present in **every** configuration below.

> [!warning] Gotcha
> The registration fee (per-year / per-3-year, paid to the **registrar**) and the hosting fee (**monthly**, paid per **hosted zone** to the **host**) are *different fees*. Don't conflate them. Also don't confuse the **registry** (owns the TLD) with the **registrar** (registers on your behalf).

**Interoperability** is the term for splitting these two roles across different providers — Route 53 doing one while a third party (e.g. Hover) does the other. There are three arrangements.

### Both roles via Route 53 (the default)

The traditional setup: you pay R53 to register, and R53 also hosts the zone. The R53 registrar liaises with the R53 hosting side to create a public hosted zone with 4 name servers, then hands the **registration fee + domain name + 4 NS** to the registry (PIR for `.org`), which adds the NS delegation records to the TLD. You pay **once** to register plus a **monthly** hosting fee.

![Route 53 acting as both registrar and DNS host: a left purple "Registered Domains" panel where a person registers and pays the per-year fee, a right purple "Public Hosted Zone" panel holding the 4 name servers and zone file for animals4life.org with a monthly fee, internal orange arrows creating the hosted zone and returning the 4 NS to the registrar, and red arrows where the PIR registry adds NS delegation records into the TLD — both panels the same colour to signal both roles belong to R53](../assets/R53-interoperability-both.png)

The visual "tell" across all three diagrams: **panel colour = ownership**. Purple = Route 53; green/teal (with Hover branding) = third party. Whichever side is green is the part R53 is *not* doing.

### Route 53 as registrar only

R53 registers the domain, but a **third party hosts** the zone. You create the hosted zone *at the third party* (monthly fee), retrieve **its** name server details, and pass those external NS to R53 the registrar, which sets the TLD's NS records to point at the third-party name servers.

![Route 53 as registrar only: left purple "Registered Domains" panel unchanged, right panel now green/teal "3rd Party Domain Hosting" with a Hover-branded zone file and the 4 external name servers, green arrows pulling the external NS details back to the admin and into R53, and red arrows where the registry adds NS delegation records pointing at the external Hover name servers](../assets/R53-interoperability-registrar-only.png)

> [!warning] Gotcha
> This is **rare and discouraged** — arguably the worst way to manage domains. R53's registrar function is unremarkable; the real value of R53 is its **DNS hosting**, which this arrangement throws away. If a question frames registrar-only as "getting R53's main benefit," that's the trap.

### Route 53 as hosting only

The popular split: a **third party registers** the domain, but **R53 hosts** the zone. You'd do this when the business must register elsewhere (an existing deal or discount), or when a domain was **historically registered** elsewhere and you now want R53's DNS hosting. You create a **public hosted zone in R53** (4 NS, monthly fee), then pass those R53 NS details to the external registrar, who relays them to the registry to set the TLD's NS delegation records.

![Route 53 as hosting only: left panel now green/teal "3rd Party Domain Registrar" shown as a Hover logo box where a person registers and pays the per-year fee, right panel back to purple "Public Hosted Zone" holding R53's 4 name servers and zone file, purple arrows creating the R53 hosted zone and passing its NS to Hover, and red arrows where the registry adds NS delegation records pointing at the R53 name servers](../assets/R53-interoperability-hosting-only.png)

> [!tip] Exam Tip
> Hosting-only can be set up **at registration time or after the fact**. The classic scenario is **migrating an old, externally-registered domain to R53 DNS**: create a public hosted zone in R53 and update the **NS records at the existing registrar** to the 4 R53 name servers. No re-registration needed.

**The universal pattern:** regardless of who plays which role, the end state is identical — the **TLD registry holds NS delegation records pointing at 4 name servers that host the zone file**, and resolution works through that delegation (the same chain described in [DNS Concepts](../concepts/dns.md#resolution-chain)). All that changes between configurations is **who registers** and **who hosts**.

## Alias Records vs CNAME

The CNAME restriction at the zone apex (naked domain) is a DNS standard limitation. Route 53 solves it with its proprietary **Alias record**.

![CNAME vs Alias record comparison showing apex domain support and AWS resource targets](../assets/R53_CNAME_vs_Alias_infographic.png)

| | CNAME | Alias |
| --- | --- | --- |
| Works at zone apex? | No | Yes |
| Standard DNS? | Yes | No — Route 53 proprietary |
| Points to | Any domain name | AWS resource DNS name |
| Query charges | Normal TTL-based billing | **Free** for AWS resource targets |
| TTL | Set on the record | Managed by Route 53 |

**Alias type must match the target resource's record type:**

| Target resource | Alias type |
| --- | --- |
| ELB | A Alias |
| CloudFront distribution | A Alias |
| S3 static website endpoint | A Alias |
| API Gateway | A Alias |
| Global Accelerator | A Alias |
| Elastic Beanstalk | A Alias |
| Another Route 53 record | Match the target record type |

> [!warning] Gotcha
> Alias type and target record type must match — e.g. ELBs return A records, so create an **A Alias**, not a CNAME Alias.

> [!tip] Exam Tip
> **Always prefer Alias over CNAME** for AWS resource targets: works at apex, free, integrates with Route 53 health checks.

## Routing Policies

Route 53 can make routing decisions based on health, geography, and load — going well beyond basic DNS.

| Policy | Behaviour | Health checks |
| --- | --- | --- |
| Simple | Single record | **Not supported** — a simple record can't be associated with a health check |
| Weighted | Split traffic by percentage (A/B testing, gradual migration) | **Optional** — an unhealthy record is skipped and the draw repeats (the total weight is unchanged) |
| Latency | Route to the AWS region with lowest latency for the client | **Optional** — if the lowest-latency record is unhealthy, the next-lowest healthy one is returned |
| Failover | Active/passive; switch to secondary when primary health check fails | **Required on the primary** — the health check *is* the switching mechanism |
| Geolocation | Route based on client's state/country/continent | **Optional** — an unhealthy record falls through as if it didn't exist, continuing the specificity search |
| Geoproximity | Route based on geographic distance; adjustable bias | **Optional** — an unhealthy record is excluded and traffic shifts to the next-closest |
| Multi-value | Return multiple IPs; basic load distribution (not a true load balancer) | **Optional, per record** — each record is independently checked when a check is attached; a record with no check is always treated as healthy. Per-record checking is the whole point of multi-value, so it's almost always used |

Every policy starts the same way: a hosted zone holding **multiple records of the same name**. What differs is how Route 53 chooses which value(s) to return — by weight, by latency, by health, or by location. The three location-aware policies are easy to confuse, so keep them straight: **geolocation** matches on tagged location (relevance, never distance), **latency** picks the lowest estimated latency, and **geoproximity** picks the lowest physical distance with an adjustable bias.

Health checks are **optional** on every policy except **Failover** (and unsupported on **Simple**). Where optional, a record with **no health check attached is always treated as healthy** — so don't mix checked and unchecked records in one same-name group, or the unchecked ones can never drop out. See [Health Checks](#health-checks) for the full mechanics.

## Weighted Routing

Weighted routing controls **how often each of several same-name records is returned**. That makes it the policy for **simple load balancing** across resources, and for **controlled rollout / testing of a new software version** — send a small slice of traffic (say 5 %) to the record running the new build.

You assign each same-name record a **weight**. Route 53 sums all the weights for that name into a **total weight**, then returns each record with probability `record weight ÷ total weight`. Weights do **not** have to add up to 100: three `www` records weighted 40, 40, and 20 total 100 and so happen to read directly as 40 %, 40 %, and 20 %, but the same formula applies for any values.

![Route 53 weighted routing: three same-name www A records weighted 40/40/20 summing to a total weight of 100, returned 40%/40%/20% of the time, with callouts for zero-weight behaviour and health-check interaction](../assets/R53-weighted-routing.png)

### Zero weight and health checks

- A weight of **0** means the record is **never returned** — useful for temporarily pulling a record out of rotation. The one exception: if **every** record for the name is 0, Route 53 considers them all, so resolution can never break.
- Weighted records can carry **health checks**, and the interaction is a classic trap. A record is first selected by the weight calculation; if the selected record is **unhealthy** it is **skipped** and the draw **repeats** until a healthy record is chosen. Health checks do **not** remove a record from the calculation, so the **total weight (the denominator) never changes** — an unhealthy record is skipped over, not subtracted.

> [!warning] Gotcha
> Health checks don't re-balance weights. People expect an unhealthy record's share to be redistributed proportionally, but Route 53 just re-draws until it lands on a healthy record — the denominator stays the same.

## Latency-Based Routing

Latency-based routing returns the record expected to give the user the **lowest latency**, so it's the policy for **performance / user-experience optimization** of a global application.

Each same-name record is tagged with the **AWS region** where its infrastructure lives — you get **one record per name per region**. AWS maintains a **latency database** of the latency between user locations and AWS regions. At resolution time Route 53:

1. Determines the user's approximate location with an **IP-geolocation lookup** (see note below).
2. Looks up the modelled latency from that location to each tagged region.
3. Returns the record for the region with the **lowest estimated latency that is also healthy**. If the lowest-latency record is unhealthy, the **next-lowest** healthy record is returned instead.

![Route 53 latency-based routing: three same-name records tagged us-east-1/us-west-1/ap-southeast-2, a user in Australia matched against AWS's latency database where ap-southeast-2 (42 ms) beats us-west-1 (280) and us-east-1 (400), returning the ap-southeast-2 record](../assets/R53-latency-based-routing.png)

> [!warning] What "IP lookup" means — and whose IP
> The IP lookup is **not a separate service**. It's an internal **geo-IP** lookup: Route 53 maps the **source IP of the incoming DNS query** to an approximate location using a built-in IP-to-location database. The same mechanism drives [Geolocation Routing](#geolocation-routing) below.
> Crucially, the IP Route 53 sees is usually the user's **DNS resolver** (their ISP's resolver, `8.8.8.8`, etc.), **not the end-user device** — DNS is recursive, so the resolver is what queries Route 53. Route 53 geolocates the resolver on the assumption it sits near the user. If the resolver supports **EDNS Client Subnet (ECS)** — an EDNS0 extension that forwards a truncated slice of the client's IP — Route 53 uses that client subnet for a far more accurate location. Public resolvers (Google, OpenDNS) support ECS; many ISP resolvers don't. This is why the result is always *approximate*.

> [!warning] Gotcha
> The latency database is an **estimate** — updated in the background, **not real-time**, and blind to local network conditions. It's a strong performance aid, not a live measurement. Don't confuse it with **geoproximity**, which routes by physical distance plus an adjustable bias (a different mechanism even though both feel location-aware).

> [!tip] Exam Tip
> **Failover** requires a Route 53 **health check** on the primary record. Health checks can monitor an endpoint (HTTP/HTTPS/TCP), a CloudWatch alarm, or other health checks (calculated health check). See [Health Checks](#health-checks) below.

## Failover Routing

Failover routing implements **active/passive failover**: one DNS name normally resolves to a primary resource, and automatically falls back to a backup resource when the primary becomes unavailable.

You create **two records of the same name** in the hosted zone, each tagged with a failover role:

- A **Primary** record pointing at the main resource (e.g. an EC2 instance).
- A **Secondary** record of the same name pointing at a different resource (e.g. an S3 bucket).

A [health check](#health-checks) — generally attached to the **primary** record — decides which value is returned:

- Primary health check **healthy**: the name resolves to the **primary** record's value.
- Primary health check **unhealthy**: the name resolves to the **secondary** record's value.

![Route 53 failover routing in the animals4life.org hosted zone: a www query resolves through the primary record to an EC2 instance while its health check is healthy, and to the secondary record's S3 bucket when the health check is unhealthy](../assets/R53-failover-routing-architecture.png)

In the diagram, two `www` records share a name inside the hosted zone. A health check probes the EC2 instance and is wired to the primary record. While it reports **healthy**, the client's `www` query follows the orange path to the **EC2 instance** running the app. When it reports **unhealthy**, the same query follows the pink path to the **S3 bucket** instead.

### Out-of-band failure / maintenance pattern

The canonical use is an **"out-of-band" failure or maintenance architecture**: the primary and secondary deliberately point at *different, independent services* (EC2 primary, S3 static-site secondary). Because the backup runs on a separate platform, it can still serve a maintenance page or static fallback even when the primary's underlying platform is the thing that has failed — the failure path does not depend on the same infrastructure that broke.

> [!warning] Gotcha
> Don't confuse failover with **weighted** or **multi-value** routing. Failover is strictly *active/passive* — one resource serves at a time, chosen by health. Weighted and multi-value **distribute** traffic across multiple healthy resources at once; they are not a primary/backup mechanism.

## Geolocation Routing

Geolocation routing makes decisions from the **location of the user** and the **location tagged on each record**. Use it to **restrict content** to a market, serve **language-specific** content, or **load balance across regional endpoints** by customer location. It resembles latency routing but keys off location rather than latency.

> [!warning] Gotcha
> The single most important point (and a repeated exam trap): geolocation returns **relevant (location-matching) records only — never the closest record as a fallback**. A user in the UK will *not* receive a Canadian record just because no nearer record exists. The outcome is always a matching record, the default record, or "no answer" — never a proximity-chosen "nearest".

You tag each record with one of four location types, most specific to least:

1. **Subdivision** — a US state (the most specific)
2. **Country** — ISO country code
3. **Continent** — ISO continent code (e.g. `SA` for South America)
4. **Default** — an optional catch-all

At resolution, the same **geo-IP lookup** described under [Latency-Based Routing](#latency-based-routing) determines the location — geolocating the source IP of the DNS query, which is normally the user's resolver unless EDNS Client Subnet is in play. Route 53 then checks for matching records in specificity order, **stopping at the first level that produces a match**: state, then country, then continent, then default. It returns the **most specific** relevant record. If nothing matches and there is **no default record**, Route 53 returns **"no answer"**.

![Route 53 geolocation routing: records tagged by US state / country / continent / default, with the matching order State then Country then Continent then Default, returning the most specific relevant record or NO ANSWER when nothing matches](../assets/R53-geolocation-routing.png)

## Geoproximity Routing

Geoproximity routing answers with the resource that is **physically closest** to the user by **distance**, and adds an adjustable **bias** so you can deliberately shift that boundary to send more or less traffic to a resource.

You define **rules**, each specifying where a resource lives:

- For an **AWS resource**: its **AWS region** (e.g. `us-east-1`, `ap-southeast-2`).
- For an **external (non-AWS) resource**: **latitude/longitude coordinates**.

Each rule also takes a **bias** that changes how the distance calculation is performed rather than using raw distance:

- A **positive (`+`) bias grows** a resource's effective service area, pulling in more surrounding traffic — and correspondingly **shrinks neighbouring** regions.
- A **negative (`-`) bias shrinks** it.

![Route 53 geoproximity routing: three resources in America, the UK, and Australia each with a tinted service area, where a large plus bias on the Australian resource expands its area across the Middle East so Saudi Arabian users route to Australia instead of the geographically closer UK](../assets/R53-geoproximity-routing.png)

**Worked example from the diagram:** a user in Saudi Arabia has no local resource and sits roughly 6,700 km from a UK resource and 10,000 km from an Australian one, so by raw distance the **UK** would win. Applying a large **`+` bias to the Australian resource** expands its service area across the Middle East, routing the Saudi user to **Australia instead of the UK** despite the greater distance.

> [!tip] Exam Tip
> Keep the location-aware trio distinct: **geolocation** = relevance by tagged location (no concept of distance or bias); **latency** = lowest *estimated latency* (no bias); **geoproximity** = lowest *physical distance* plus an adjustable bias. Only geoproximity lets you tune how much traffic a resource attracts.

## Multi-Value Routing

Multi-value routing behaves like a blend of **simple** and **failover** routing: it returns **multiple same-name records at once**, each **independently health-checkable**, and lets the client pick one — an active/active approach to DNS.

- Route 53 returns **up to 8 healthy records** per query; if more than 8 healthy records exist, **8 are chosen at random**.
- Each record can have its **own health check** (optional, but the reason to use multi-value). A record failing its check is **excluded** from responses; a record with **no** check attached is always treated as healthy.
- The **client selects one value** from the set it receives and connects to that resource directly.

![Route 53 multi-value routing: three same-name www records each with an independent health check, the two healthy records returned to the client while the failing one is struck through, and the client choosing one value to connect to](../assets/R53-multi-value-routing.png)

This positions it between the simpler policies — it adds per-record health checking that **simple** routing lacks, while returning many records at once rather than the strict primary/backup of **failover**:

| Policy | Health checks? | Typical use |
| --- | --- | --- |
| Simple | No | A single resource (e.g. one web server) |
| Failover | Yes | Active/backup architectures (e.g. EC2 primary, S3 backup) |
| Multi-value | Optional, per record | Many resources that can all serve requests; health-checked and returned at random |

> [!warning] Gotcha
> Multi-value routing is **not a load balancer**. It improves **availability** by returning several health-checked addresses over DNS, but it does **not** manage connections at the network level — the client does the choosing and connecting. When a question needs true load balancing, the answer is an **ELB**, not multi-value routing.

## Health Checks

Route 53 health checks monitor the availability of endpoints and feed the result back into DNS routing decisions. They are the engine behind **Failover** routing and a useful filter for **Multi-value** and **Weighted** policies, where unhealthy records are excluded from query responses.

![Route 53 health checks infographic: core principles, types (endpoint/CloudWatch/calculated), verification methods (TCP/HTTP/string), and operational control (intervals/notifications/failover)](../assets/R53_health_check_infographic.png)

### Key properties

- **Independent objects**: health checks are created in the Route 53 console under **Health Checks**, *not* inside a hosted zone. A single health check can be referenced by many DNS records.
- **Not limited to AWS**: any publicly reachable IP or DNS name can be monitored — useful for hybrid or multi-cloud setups.
- **Globally distributed checkers**: AWS runs a fleet of health checkers across many regions. Each checker probes the target independently and reports back.
- **Default interval 30 s; fast mode 10 s** (extra cost). **Failure threshold** = number of consecutive failed probes before status flips.

- **No check means "healthy"**: a record with **no health check associated** is always treated as **healthy** and stays eligible to be returned. This is why health checks are *optional* on every policy except **Failover** (where the primary's check is the switching mechanism). For [Alias records](#alias-records-vs-cname) the equivalent switch is **Evaluate Target Health** — left off, the alias is always healthy; turned on, the alias inherits the health of its target (e.g. the ELB's own target health).

> [!warning] Gotcha
> Your firewall ([Security Group](vpc.md#security-groups-sg)/[NACL](vpc.md#network-acl-nacl)) must **allow inbound traffic from Route 53 health checker IP ranges**. Blocking them by accident causes false alarms — endpoints look unhealthy even though real traffic is fine. AWS publishes the IP ranges and lets you scope checks to specific checker regions.

> [!warning] Gotcha
> **Don't mix checked and unchecked records in the same same-name group.** Because a record with no health check is permanently "healthy", a record you forgot to attach a check to can **never drop out** of responses — it keeps receiving traffic even when its endpoint is down. Attach checks to **all** records in a group, or none.

### Quorum: the 18 % rule

![Route 53 distributed health checkers — if 18%+ report healthy, the check is healthy; unhealthy records are normally removed from query responses](../assets/R53-health-checkers.png)

Each checker probes the endpoint independently. Route 53 aggregates the results:

- If **more than 18 %** of checkers report **healthy** → overall status is **healthy**
- Otherwise → **unhealthy**, and the record is normally excluded from DNS responses

> [!tip] Exam Tip
> The 18 % aggregation makes Route 53 health checks resilient to localised network failures — a checker in one region failing to reach your endpoint will not, on its own, mark you down.

### Endpoint check types

A single endpoint health check uses one of three probe types:

| Probe | What passes | Timing | Notes |
| --- | --- | --- | --- |
| **TCP** | TCP handshake completes | within 10 s | Basic L4 reachability — does not validate application |
| **HTTP / HTTPS** | TCP within 4 s, then HTTP status `200–399` within 2 s | 4 s + 2 s | Validates that the web server actually responds |
| **HTTP/HTTPS + string match** | Above, plus a specified string appears in the **first 5,120 bytes** of the response body, matched within 2 s | 4 s + 2 s + 2 s | Most accurate — confirms the app is returning real content |

> [!tip] Exam Tip
> **String matching** is the most accurate signal of application health: it catches the case where the web server returns 200 OK but the app behind it is broken (e.g. database down → error page rendered with HTTP 200). Use string matching whenever the question describes verifying that the application is *actually working*, not just reachable.

### Health check categories (by input source)

Endpoint checks are only one of three types — the others compose existing signals:

| Category | Input | Use case |
| --- | --- | --- |
| **Endpoint** | IP / domain probed directly with TCP / HTTP / HTTPS | Most common — monitor a web endpoint |
| **CloudWatch Alarm** | An existing CloudWatch alarm state | Use any metric or in-app signal the CloudWatch agent can collect (CPU, queue depth, custom metric) |
| **Calculated** | Boolean of multiple other health checks | Aggregate component checks into an overall application health status |

A **calculated** health check is the building block for monitoring multi-component apps — e.g. "healthy only if both `web` and `db` checks pass."

### Console options worth knowing

- **Host header** — needed when one IP serves multiple virtual hosts
- **Request path** — point HTTP checks at `/health` rather than `/` to avoid scraping homepages
- **Invert status** — flips healthy/unhealthy (rarely useful; mostly for inverse failover patterns)
- **Disable** — pause a check during planned maintenance instead of deleting it
- **Checker regions** — restrict to a subset of regions (cost / compliance / locality)
- **Latency graph** — Route 53 records and graphs probe latency from each checker region

### Integration with SNS

Health checks can publish state changes to an **SNS topic** — fan them out to email, SMS, Lambda, or downstream incident systems for alerting. This is independent of how DNS routing uses the same check.

### Architectural role

Health checks are the link between *monitoring* and *traffic steering*:

- **Failover routing** uses them to flip from primary to secondary
- **Multi-value / Weighted** policies use them to drop unhealthy records from responses
- **CloudWatch + calculated** checks let DNS respond to *application-level* signals, not just network reachability

Combined with **ELB target group health checks** (load-balancer-level — exclude bad targets) and **Auto Scaling health checks** (instance replacement), Route 53 health checks form the **outermost layer** of AWS availability monitoring — the one that decides what IP a client even gets in its DNS answer.

## Route 53 Resolver

Every VPC has a built-in DNS resolver reachable at **VPC base IP + 2** (e.g. `10.0.0.2` for a `10.0.0.0/16` VPC). It handles:

- Local VPC hostnames (`ec2-….compute.amazonaws.com`)
- Records in associated private hosted zones
- Recursive public DNS lookups (root → TLD → authoritative)

For hybrid environments, **Resolver Endpoints** (inbound/outbound) allow on-premises networks to resolve Route 53 private zones, and VPCs to resolve on-premises DNS.

The Resolver can also be configured for **[DNSSEC](../concepts/dnssec.md) validation** per-VPC, so it rejects answers whose signatures fail to verify (the `Bogus` outcome). This is the resolver-side complement to Route 53's signing of public hosted zones — see [DNSSEC](#dnssec).

### How association and namespace resolution work

Associating a VPC with a [private hosted zone](#private-hosted-zone) does **not** copy the zone's records into the VPC. The records never leave Route 53. The `.2` resolver address is not a server living in your subnet — it is a per-VPC doorway into the regional Route 53 Resolver service, and because the doorway is per-VPC, **every query arrives stamped with its origin VPC ID**. Association is just a control-plane authorization entry — a mapping that says *"this VPC may resolve against this private zone."* Resolution is then a live lookup at query time, not a preload.

For each query, the Resolver decides the **zone first, record second**:

1. **Zone selection — does the query name fall under any associated private zone?** A private zone "matches" if its name is **identical to** or a **parent of** the query name. For `seattle.accounting.example.com`, both `accounting.example.com` and `example.com` match (the closest parent counts). If **two** private zones match, the Resolver picks the **most specific** one (`accounting.example.com` beats `example.com`).
2. **If no private zone matches** the query name, the Resolver does ordinary recursive/public resolution. *This is the only path that reaches the public zone.*
3. **If a private zone matches**, the Resolver searches **only that zone** for a record of the requested name and type. A hit is returned; a miss returns **NXDOMAIN** — it does **not** fall back to public DNS.

> [!warning] Public fall-through is decided at the **zone** level, never per record
> The trap is reading step 2 as "if the record isn't found, try public." It is not. Public fall-through happens **only when no associated private zone owns the namespace at all**. Once a private zone claims the name, a missing record is a hard NXDOMAIN. See the [split-view gotcha](#split-view-split-horizon-dns) for the canonical `www` + `customer` failure this causes.

Two escape hatches override this default behavior:

- A **Resolver rule** for the same domain **takes precedence over** a private hosted zone — matching queries are forwarded to the target (e.g. on-premises) instead of being answered from the zone.
- **NS records inside a private zone** can **delegate a subdomain** to another resolver, carving a hole in the otherwise all-or-nothing namespace.

> [!tip] Exam Tip
> "I created a private zone for my apex domain and now public names under it (like `customer.example.com`) stopped resolving from the VPC" is a classic symptom. The cause is namespace replacement; the fix is to recreate the still-needed public records inside the private zone.

## DNSSEC

[DNSSEC](../concepts/dnssec.md) is the secure *add-on* to DNS that adds origin authentication and integrity via a cryptographically verifiable chain of trust. Route 53 implements it on two sides:

- **DNSSEC signing** (authoritative) — Route 53 can sign a **public hosted zone**. You provide an asymmetric **[KMS](kms.md) customer managed key** (ECDSA P-256, in `us-east-1`) that Route 53 uses as the **KSK**; Route 53 generates and manages the **ZSK** internally and produces the RRSIGs. To complete the chain of trust you must add the matching **DS record to the parent zone** — at your registrar, or in the parent hosted zone if it is also in Route 53. Until that DS exists, the zone is signed but not part of any validated chain.
- **DNSSEC validation** (resolver) — the [Route 53 Resolver](#route-53-resolver) can validate signatures for queries from a VPC, rejecting `Bogus` answers.

> [!warning] Enable/disable ordering
> Turning DNSSEC on or off follows a strict order so validating resolvers never hit a broken (`Bogus`) zone: when **enabling**, sign the zone *before* adding the parent DS record; when **disabling**, remove the parent DS record and wait out its TTL *before* unsigning. The wrong order makes validating resolvers return **SERVFAIL** for the entire zone.

See the [DNSSEC concept page](../concepts/dnssec.md) for the full mechanics (RRSET / RRSIG / DNSKEY / ZSK / KSK / DS), the trust anchor and root signing ceremony, a worked `www.icann.org` chain, and `dig` troubleshooting.

## References

- [DNSSEC](../concepts/dnssec.md) — securing DNS; Route 53 signing + Resolver validation

- [DNS Concepts](../concepts/dns.md)
- [VPC](vpc.md)
- [TLS/HTTPS](../concepts/tls-https.md)
- [AWS Docs — Considerations when working with a private hosted zone](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/hosted-zone-private-considerations.html) (NXDOMAIN-vs-fall-through and most-specific-match rules)
- [AWS Docs — Working with private hosted zones](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/hosted-zones-private.html)
- Source: `raw/aws-cert-study-notes/aws-solution-architect/route53_public_hosted_zone.md`
- Source: `raw/aws-cert-study-notes/aws-solution-architect/route53_private_hosted_zone.md`
- Source: `raw/aws-cert-study-notes/aws-solution-architect/route53_alias_vs_cname.md`
- Source: `raw/aws-cert-study-notes/aws-solution-architect/route53_health_check.md`
- Source: `raw/aws-cert-study-notes/aws-solution-architect/route53_failover_routing.md`
- Source: `raw/aws-cert-study-notes/aws-solution-architect/route53_weighted_routing.md`
- Source: `raw/aws-cert-study-notes/aws-solution-architect/route53_latency_based_routing.md`
- Source: `raw/aws-cert-study-notes/aws-solution-architect/route53_geolocation_routing.md`
- Source: `raw/aws-cert-study-notes/aws-solution-architect/route53_geoproximity_routing.md`
- Source: `raw/aws-cert-study-notes/aws-solution-architect/route53_multi_value_routing.md`
- Source: `raw/aws-cert-study-notes/aws-solution-architect/route53_interoperability.md`
- Source: `raw/aws-cert-study-notes/aws-solution-architect/route53_fundamentals.md`
