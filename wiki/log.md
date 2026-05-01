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

## [2026-05-01] ingest | raw/random-thoughts/2026-05-01.md

Processed three thoughts. Corrections made: (1) NACL and SG both operate at L3/4, not L3 and L7 respectively — ports are a L4 (TCP/UDP) concept, not L7; L7 is AWS WAF. (2) Route tables are L3 only. (3) SG mental model corrected from "server/instance" heuristic to "attaches to ENIs" — Lambda in VPC has SG, Lambda without VPC does not; Fargate has SG. Updated vpc.md, networking.md, and aws-saa.md.
