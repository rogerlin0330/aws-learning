# Wiki Log

## [2026-05-19] ingest | CloudWatch Logs

Created `wiki/services/cloudwatch.md` from `raw/aws-cert-study-notes/aws-solution-architect/cloudwatch.md`. Covers: public-endpoint regional service, log group → stream → event hierarchy, ingestion methods (native integrations, Unified CloudWatch Agent, SDK), metric filters and alarms. Copied architecture diagram to `wiki/assets/`. Added CloudWatch exam tips section to `certifications/aws-saa.md`. Updated index.

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

## [2026-05-13] ingest | Route 53 health checks

Ingested `raw/aws-cert-study-notes/aws-solution-architect/route53_health_check.md` and copied 2 images (R53_health_check_infographic, R53-health-checkers) into `wiki/assets/`. Added a new "Health Checks" section to `services/route53.md` covering: independence from DNS records, global checker fleet + firewall-allowance gotcha, 18% quorum rule, probe types (TCP / HTTP / HTTPS + string match) with timing, health check categories (Endpoint / CloudWatch Alarm / Calculated), console options, SNS integration, and architectural role alongside ELB / Auto Scaling health checks. Updated `certifications/aws-saa.md` with a Route 53 Health Checks subsection (string-matching = most accurate, 18% rule, health-checker IP ranges must be allowed, lives outside hosted zones). Updated `wiki/index.md` Route 53 description.

## [2026-05-15] update | Embed IAM-policy ∩ SCP Venn diagram

User added `iam-policy-scp-replationship.png` (filename typo preserved) to the raw SCP note. Copied to `wiki/assets/` and embedded in the "Cardinal Rule" subsection of `services/organizations.md` immediately after the intersection-rule paragraph, with descriptive alt text and a two-bullet recap of the two failure modes (identity-allow-SCP-block vs SCP-allow-no-identity-grant).

## [2026-05-15] update | Tighten cost-optimization.md scope

Removed off-topic governance phrasing from `patterns/cost-optimization.md`: condensed the Scope note and rewrote the pooling section so it stays on the cost-pooling mechanic instead of contrasting with SCPs/governance use cases (which belong on `services/organizations.md`). Renamed the pooling H2 to drop redundant "AWS Organizations" wording and link directly to the consolidated-billing section instead.

## [2026-05-15] ingest | Service Control Policies

Ingested `raw/aws-cert-study-notes/aws-solution-architect/organization_scp.md` and copied `organization-scp-infographic.png` to `wiki/assets/`. Replaced the brief SCP section in `services/organizations.md` with a comprehensive expansion covering: the cardinal "restrict-not-grant" rule and intersection model (SCP allows ∩ IAM identity policy allows), full attachment hierarchy with downward inheritance (root / OU / account), the management-account exemption and its practical implication (don't run workloads in the management account), how SCPs indirectly bound the root user (otherwise unrestrictable by IAM policies), deny-list (default `FullAWSAccess`) vs allow-list models with overhead trade-offs, SCP policy evaluation order, and typical real-world uses. Updated `concepts/identity-and-access.md` with a new "SCPs as a Permission Boundary" subsection. Expanded `certifications/aws-saa.md` with a dedicated SCP subsection (intersection rule, deny-list vs allow-list, root-user-via-SCP nuance, management-account trap).

## [2026-05-14] ingest | AWS Organizations + Cost Optimization

Ingested `raw/aws-cert-study-notes/aws-solution-architect/organization.md` and copied `organization-infographic.png` to `wiki/assets/`. Created new `services/organizations.md` (management vs member accounts, OUs, consolidated billing, role-switching workflow, SCPs incl. management-account exemption gotcha). Created new `patterns/cost-optimization.md` (Compute SP vs EC2 Instance SP vs Standard/Convertible RI vs Spot; the SP-doesn't-cover-RDS/ElastiCache/Redshift/OpenSearch gotcha; cross-account benefit pooling via Organizations; decision tree). Added matching AWS Organizations and Cost Optimization sections to `certifications/aws-saa.md` with purchase-plan comparison table and pattern-match exam tips. Updated `index.md` (added Organizations service entry, populated previously-empty Patterns section), `overview.md` (new Multi-Account Architecture section, expanded service/concept lists), and `services/iam.md` (cross-account-access section now links to organizations.md).

## [2026-05-14] ingest | IAM Groups

Ingested `raw/aws-cert-study-notes/aws-solution-architect/iam_groups.md` and copied `iam-groups-infographic.png` to `wiki/assets/`. Added a new **IAM Groups** section to `services/iam.md` between IAM Users and IAM Roles, covering: groups are containers (not identities, no credentials, no login), effective-permissions union with user policies, limits table (5,000 users/account, 10 groups/user, no nesting, 300 groups default, no built-in "all users" group), and the resource-policy reference restriction (groups cannot be named — only users/roles). Added Gotcha and Exam Tip callouts. Refined `concepts/identity-and-access.md` group row to state it is not an identity. Added an IAM Groups bullet to `certifications/aws-saa.md`.

## [2026-05-14] ingest | IAM Roles

Ingested `raw/aws-cert-study-notes/aws-solution-architect/iam_roles.md` and copied 2 images (iam-roles-infographic, iam-roles-infographic-cont) into `wiki/assets/`. Added a comprehensive **IAM Roles** section to `services/iam.md` covering: roles vs users (5,000-user limit), trust policy vs permissions policy, the STS `AssumeRole` flow (credential triplet, expiration, real-time permission enforcement), and the five canonical use cases — AWS service roles (Lambda execution role), break-glass / emergency access, SAML federation with corporate IdP, web identity federation for mobile apps, and cross-account access incl. AWS Organizations. Updated `concepts/identity-and-access.md` with new sections on Principals & Role Assumption and Identity Federation (SAML / Web Identity / Cross-account); refined credential-types table to highlight the session token. Added an "IAM Roles" subsection to `certifications/aws-saa.md` with a use-case lookup table and pattern-match exam tips. Updated `index.md` IAM description.

## [2026-05-16] ingest | S3 resource policies and ACL + resource-vs-identity policies concept

Ingested `raw/aws-cert-study-notes/aws-solution-architect/s3_resource_policies_and_acl.md` and copied `s3-resource-policies-and-acl-infographic.png` to `wiki/assets/`. Created two new pages:

- `concepts/resource-vs-identity-policies.md` — generalised the perspective split (identity = "what can this identity do?", resource = "who can access this resource?"), the `Principal` element as the syntactic marker, and the combination rules. Corrected a source inaccuracy: same-account access is a **union** (Allow in either policy is sufficient, subject to Deny), not an intersection — the intersection model is cross-account-only. Documented anonymous-only-via-resource-policy, IAM groups can't be principals, and trust-policy-is-a-resource-policy.
- `services/s3.md` — S3 service page focused on access control: private-by-default baseline, bucket-policy anatomy + ARN bucket-vs-objects distinction, advanced `Condition` patterns (NotIpAddress, Null/aws:MultiFactorAuthAge), ACLs (legacy, 5-permission table, new buckets default to ACLs-disabled / Object Ownership = Bucket owner enforced), Block Public Access as an override with the four switches, and a decision table.

Cross-references: `concepts/identity-and-access.md` Policy Evaluation section now warns about same-account union vs cross-account intersection; `services/iam.md` Identity Policies section now opens by contrasting against resource policies and links to the new concept page and S3. Updated `certifications/aws-saa.md` with new "Resource Policies vs Identity Policies" and "S3 Security" sections (pattern-match tips, condition table, Block Public Access exam traps, "enabling ACLs is the wrong answer" warning). Updated `index.md` (added S3 + the new concept) and `overview.md` (services list, resilience tier table now links to S3, topic-to-cover-next updated).

## [2026-05-01] ingest | raw/random-thoughts/2026-05-01.md

Processed three thoughts. Corrections made: (1) NACL and SG both operate at L3/4, not L3 and L7 respectively — ports are a L4 (TCP/UDP) concept, not L7; L7 is AWS WAF. (2) Route tables are L3 only. (3) SG mental model corrected from "server/instance" heuristic to "attaches to ENIs" — Lambda in VPC has SG, Lambda without VPC does not; Fargate has SG. Updated vpc.md, networking.md, and aws-saa.md.

## [2026-05-17] update | s3.md — clarify Block Public Access semantics

Added a warning callout in the Block Public Access section emphasizing that disabling BPA is not equivalent to making a bucket public. The two switches are independent: a bucket stays private by default, and turning BPA off merely removes the failsafe — public access still requires a bucket policy (or ACL) granting `Principal: "*"`. Conversely, leaving BPA on while attaching a public policy yields 403. Both a grant and the absence of the override must line up.

## [2026-05-17] update | Document IAM policy JSON case-sensitivity rules

Added "Policy JSON — Case Sensitivity" subsection to `services/iam.md` covering case-sensitive elements (top-level keys, Effect values, Principal sub-keys, condition operators), case-insensitive elements (action names, condition-key names), and value rules (StringEquals vs StringEqualsIgnoreCase, ARNs, tags). Added an exam tip. Added a cross-referencing summary on `concepts/resource-vs-identity-policies.md` pointing to the canonical IAM section. Updated index.
