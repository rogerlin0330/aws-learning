# Wiki Log

## [2026-05-01] seed | Initial wiki setup

Seeded wiki from existing study notes in `raw/aws-cert-study-notes/aws-solution-architect/`. Created pages: IAM, VPC, EC2 (stub), Networking concepts, Identity and Access concepts, AWS SAA cert notes, Overview. Wrote CLAUDE.md schema.

## [2026-05-01] update | Restructure wiki for broader AWS learning

Renamed `exams/` to `certifications/`; added `patterns/` to structure. Reframed wiki as AWS learning broadly (how things work, architecture patterns, hands-on) with certifications as a subset. Added `.markdownlint.json` config. Fixed various lint warnings.

## [2026-05-01] update | Move aws-certificates/ into raw/

Moved `aws-certificates/aws-solution-architect/` and `aws-certificates/aws-ai-practitioner/` into `raw/` using `git mv`, aligning with the LLM Wiki pattern (raw = immutable source inputs, wiki = synthesized output). Updated CLAUDE.md and wiki references accordingly.

## [2026-05-01] update | Reorganize raw/ — group cert notes under aws-cert-study-notes/

Moved `raw/aws-solution-architect/` and `raw/aws-ai-practitioner/` under `raw/aws-cert-study-notes/`. Updated all references in CLAUDE.md and wiki pages.

## [2026-05-01] ingest | Regional NAT Gateway (Nov 2025)

Ingested two sources: AWS blog post and official docs on Regional NAT GW. Restructured vpc.md NAT section to distinguish two independent dimensions (connectivity type: public/private; availability mode: zonal/regional). Kept all zonal knowledge; added RNAT coverage: no public subnet required, auto-expands across AZs, single ID, automatic/manual config modes, port exhaustion protection, limitations (no private NAT), conversion steps. Updated aws-saa.md exam notes and vpc.md References.

## [2026-05-01] update | Add images to wiki pages

Copied 13 images from `raw/aws-cert-study-notes/aws-solution-architect/assets/` to `wiki/assets/`. Embedded in iam.md (policy priority, multiple policies) and vpc.md (subnet tiers, IGW inbound/outbound, stateless/stateful firewalls, NACL, NACL complex, SG, SG logical ref, SG self-ref, NAT GW vs NAT instance). Updated CLAUDE.md with image convention.

## [2026-05-01] update | Wire cross-references across all wiki pages

Added inline cross-references throughout all pages to build a connected knowledge graph. iam.md ↔ identity-and-access.md ↔ vpc.md ↔ networking.md ↔ ec2.md ↔ overview.md ↔ aws-saa.md. Updated CLAUDE.md with cross-referencing rules. ec2.md expanded with architectural context.

## [2026-05-07] update | Embed R53 images in route53.md; update CLAUDE.md image guidance

Copied 4 images (R53_public_hosted_zone_infographic, R53_public_hosted_zones, R53_private_hosted_zone_infographic, R53_CNAME_vs_Alias_infographic) from raw/assets/ to wiki/assets/. Embedded each in the relevant section of route53.md. Updated CLAUDE.md to instruct that Obsidian-style image references in raw notes should be treated as important and embedded in wiki pages.

## [2026-05-07] update | Restructure DNS content — extract Route 53 into services/

Separated Route 53-specific content out of `concepts/dns.md` into a new `services/route53.md`. `dns.md` now covers only generic DNS concepts (resolution chain, TTL, record types, CNAME apex limitation). `route53.md` covers hosted zones (public/private), Split-View DNS, Alias vs CNAME, routing policies, and Route 53 Resolver. Updated index.md, aws-saa.md, and cross-references.

## [2026-05-07] ingest | Route 53 hosted zones and Alias vs CNAME notes

Ingested three study notes from `raw/aws-cert-study-notes/aws-solution-architect/`: updated public hosted zone, new private hosted zone, new Alias vs CNAME. Expanded `concepts/dns.md` with: Alias vs CNAME comparison table (including A-alias vs CNAME-alias type matching), Route 53 Hosted Zones section (public: 4 NS per zone, VPC +2 resolver, cost; private: VPC association, cross-account CLI/API only), Split-View DNS pattern, Route 53 Resolver details. Updated `certifications/aws-saa.md` with new exam points (globally resilient, Alias type matching, cross-account association, Split-View DNS).

## [2026-05-07] ingest | DNS and TLS/HTTPS web clippings

Ingested two web clippings from `raw/web-clippings/`. Created `concepts/dns.md` (resolution chain, TTL, record types, Route 53 routing policies) and `concepts/tls-https.md` (CA issuance flows, certificate chain of trust, TLS 1.2/1.3 handshake, ACM). Corrected a source inaccuracy: the SSL article labeled its handshake diagram "TLS 1.3" but depicted the TLS 1.2 ClientKeyExchange flow; wiki documents both accurately. Added Route 53 and ACM exam tips to aws-saa.md. Updated networking.md cross-references and index.md.

## [2026-05-01] ingest | raw/random-thoughts/2026-05-01.md

Processed three thoughts. Corrections made: (1) NACL and SG both operate at L3/4, not L3 and L7 respectively — ports are a L4 (TCP/UDP) concept, not L7; L7 is AWS WAF. (2) Route tables are L3 only. (3) SG mental model corrected from "server/instance" heuristic to "attaches to ENIs" — Lambda in VPC has SG, Lambda without VPC does not; Fargate has SG. Updated vpc.md, networking.md, and aws-saa.md.
