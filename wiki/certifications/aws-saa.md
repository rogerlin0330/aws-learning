---
title: AWS Solutions Architect Associate — Exam Notes
type: certification
tags: [iam, vpc, networking, ec2, route53]
certs: [aws-saa]
updated: 2026-05-14
---

# AWS Solutions Architect Associate (SAA-C03)

A running collection of exam tips, gotchas, and comparison tables. Deep-dive pages: [IAM](../services/iam.md) · [VPC](../services/vpc.md) · [EC2](../services/ec2.md) · [Route 53](../services/route53.md) · [Networking](../concepts/networking.md) · [Identity & Access](../concepts/identity-and-access.md) · [DNS](../concepts/dns.md) · [TLS/HTTPS](../concepts/tls-https.md).

## IAM

See [IAM](../services/iam.md) and [Identity and Access Concepts](../concepts/identity-and-access.md) for full details.

- **Deny overrides Allow**: explicit Deny in any policy wins over any Allow — across all policy sources
- Managed policies > inline policies: prefer managed for reusability and lower overhead
- IAM Users = long-term credentials; [IAM Roles](../services/iam.md#iam-roles) = temporary credentials — always use roles for [EC2](../services/ec2.md), Lambda, cross-account access
- IAM is **global** — not region-specific
- Hard limit: **5,000 IAM users per account** — for larger user bases use roles + federation
- **IAM Groups** are containers, **not identities**: no credentials, no nesting, cannot be referenced by [resource policies](../services/iam.md#policy-evaluation--priority) (only users/roles can). Limits: **10 groups per user**, default **300 groups per account**, no "all users" built-in group. Effective permissions = user policies ∪ all group policies (same Deny-wins evaluation)

### IAM Roles

See [IAM Roles](../services/iam.md#iam-roles) for full details.

- A role has **two policies**: **trust policy** (who can assume it) and **permissions policy** (what it can do)
- A principal denied by the trust policy never gets credentials — `sts:AssumeRole` fails before permissions are checked
- Assuming a role returns a **triplet**: Access Key ID + Secret Access Key + **Session Token** — the session token is the marker of temporary credentials
- Permissions changes apply to **in-flight sessions immediately** — STS does not re-issue credentials, AWS re-checks the permissions policy on every API call
- Roles can be referenced in **resource policies** (e.g. S3 bucket policy granting a role ARN) — both the role policy and the resource policy are evaluated together
- **External identities cannot directly access AWS** — Active Directory users, Google/Facebook users, and users in other AWS accounts must all assume a role first

#### When the answer is "use a role"

| Scenario | Mechanism |
| --- | --- |
| AWS service acting on your behalf (Lambda, EC2 → S3) | **Service role** — trust policy trusts the service principal (e.g. `lambda.amazonaws.com`) |
| Emergency / rare elevated access | **Break-glass role** — assume only on demand, CloudTrail logged |
| Existing corporate IdP (Active Directory, Okta) | **SAML federation** — `sts:AssumeRoleWithSAML` |
| Mobile/web app with millions of users (Google/Facebook login) | **Web identity federation** — `sts:AssumeRoleWithWebIdentity` |
| Access to another AWS account / AWS Organizations | **Cross-account role** — `sts:AssumeRole` |

> [!tip] Exam Tip
> Pattern-match: "company has existing AD" or "millions of mobile users" → IAM **role + federation**, not IAM users. "Lambda needs to write to S3" → **execution role**, never embedded access keys. "Two AWS accounts" → cross-account **role assumption**, not a duplicate IAM user.

> [!warning] Gotcha
> Never bake long-term IAM user access keys into Lambda code, EC2 user-data, or container images. Use a role — exam questions describing "access keys in code" are usually pointing at this anti-pattern.

## VPC & Networking

See [VPC](../services/vpc.md) and [Networking Concepts](../concepts/networking.md) for full details.

### Firewall Rules

- Both [NACL](../services/vpc.md#network-acl-nacl) and [Security Group](../services/vpc.md#security-groups-sg) operate at **Layer 3/4** — not Layer 7; they filter by IP, protocol, and port only
- ICMP rules = Layer 3 only (no ports); TCP/UDP rules = Layer 4
- Layer 7 (HTTP content, headers, URLs) is handled by **WAF**, not NACL or SG
- **NACL** = stateless, attaches to **subnets** (L3 constructs), supports explicit Deny → use to block bad IPs
- **Security Group** = stateful, attaches to **ENIs** (L3/4 constructs), no explicit Deny → use for fine-grained per-resource control
- Because NACLs are stateless: always allow **ephemeral ports (1024–65535)** for return traffic
- The real distinction between NACL and SG is **stateful vs stateless** + **attachment point**, not OSI layer

### IGW vs NAT GW

| | IGW | NAT GW |
| --- | --- | --- |
| Direction | Bidirectional (public subnet) | Outbound only (private subnet) |
| IP mapping | 1:1 (private ↔ Elastic IP) | Many-to-one |
| Resilience | Regional | Zonal or Regional (see below) |
| IPv6 support | Yes | No |

### Zonal vs Regional NAT GW (availability mode — as of Nov 2025)

| | Zonal NAT GW | Regional NAT GW (RNAT) |
| --- | --- | --- |
| Scope | Single AZ | VPC-level |
| Public subnet required? | Yes | No |
| High availability | Manual — one per AZ, per-AZ routes | Automatic — single ID, auto-expands |
| Private NAT support | Yes | No |
| Use when | Private NAT; strict per-AZ IP control | Most public egress use cases |

> [!tip] Exam Tip
> Pre-Nov 2025 courses only cover **zonal** NAT GW. For region resilience with zonal: deploy one per AZ, configure each AZ's route table to its local NAT GW. With **regional** NAT GW, a single ID covers all AZs automatically — no public subnet needed.

### NAT GW vs NAT Instance

- **NAT GW**: AWS-managed, no SG support, charged by duration + data
- **NAT Instance**: self-managed, supports SG, cheaper at high data volumes — must disable **source/destination checks**

### IPv4 vs IPv6

- Public IPv4 is **not configured in the OS** — only visible at the IGW level
- IPv6 is configured directly in the OS (all IPv6 is publicly routable in AWS)
- IPv6 outbound-only: use **Egress-Only IGW**, not NAT GW (NAT GW doesn't support IPv6)

### VPC Design

- Default VPC CIDR: `172.31.0.0/16` — avoid in production
- 5 reserved IPs per subnet: network, router (+1), DNS (+2), reserved (+3), broadcast (last)
- `enableDnsHostnames` = false by default on custom VPCs; set to true to get public DNS names

## DNS and Route 53

See [Route 53](../services/route53.md) and [DNS Concepts](../concepts/dns.md) for full details.

- Route 53 is an **authoritative DNS** service — **globally resilient**, name servers span multiple regions
- Each hosted zone gets **4 public name servers**; NS records at the registrar must point to them
- **Alias record**: Route 53-specific; works at the zone apex (`example.com`); CNAME cannot; **no query charge** to AWS resources
- CNAME only works for **subdomains** — never for the root domain
- Alias type must **match the target resource's record type** (e.g. ELB returns A → create A Alias)
- **Private Hosted Zone**: resolves only within a VPC; requires `enableDnsSupport` and `enableDnsHostnames` on the VPC
- **Cross-account private hosted zone association**: CLI/API only — not available in console
- **Split-View DNS**: same domain name, public + private zone → different answers based on query origin
- TTL = how long resolvers cache a record; **lower TTL before migrations**, raise after

### Route 53 Routing Policies

| Policy | When to use |
| --- | --- |
| Simple | Single endpoint, no health checks |
| Weighted | A/B testing, gradual traffic shifts |
| Latency | Route to lowest-latency AWS region |
| Failover | Active/passive HA; requires health check on primary |
| Geolocation | Route by user's country/continent |
| Multi-value | Return multiple IPs; basic load distribution |

> [!tip] Exam Tip
> **Failover routing** requires a Route 53 **health check** on the primary record. If the primary fails the health check, Route 53 automatically returns the secondary record.

### Route 53 Health Checks

See [Route 53 Health Checks](../services/route53.md#health-checks) for full details.

- Health checks are **separate objects** from DNS records — created under Route 53 → Health Checks, then attached to records
- Can monitor **any public endpoint** by IP or domain — not just AWS resources
- Three categories: **Endpoint** (direct probe), **CloudWatch Alarm** (any metric), **Calculated** (boolean of other checks)
- Endpoint probe types: **TCP** (handshake in 10 s), **HTTP/HTTPS** (status 200–399), **HTTP/HTTPS + string match** (matches first 5,120 bytes of body)
- **String matching is the most accurate** application health signal — catches "200 OK but app is broken" cases
- Default interval **30 s**; fast mode **10 s** at extra cost
- **18 % rule**: if more than 18 % of distributed checkers report healthy → overall healthy; otherwise unhealthy
- Firewalls ([SG](../services/vpc.md#security-groups-sg)/[NACL](../services/vpc.md#network-acl-nacl)) must **allow Route 53 health checker IP ranges** or checks will false-fail
- Unhealthy records are normally **excluded from DNS responses**
- **SNS integration** for alerting on state change — independent of routing behaviour

> [!tip] Exam Tip
> When a question describes verifying that an app is **actually working** (not just reachable), the answer is **HTTP/HTTPS with string matching**. TCP only proves the port is open; plain HTTP only proves the web server responds; string matching proves the response body is correct.

> [!warning] Gotcha
> Health checks live **outside hosted zones**. Look for them under Route 53 → Health Checks in the console, not inside the record set.

## TLS, HTTPS, and ACM

See [TLS/HTTPS Concepts](../concepts/tls-https.md) for full details.

- **ACM** (AWS Certificate Manager): free, auto-renewed certs for ALB, CloudFront, API GW
- ACM certificates **cannot be exported** — private key never leaves AWS
- **CloudFront + ACM**: the certificate **must be in `us-east-1`** (N. Virginia), regardless of distribution origin regions
- **ALB + ACM**: certificate must be in the **same region as the ALB**
- ACM domain validation via **DNS** (add TXT record to Route 53) is fully automated if hosted zone is in the same AWS account
- **ACM Private CA**: issues private (non-publicly-trusted) certs for internal services

> [!warning] Gotcha
> **TLS termination at ALB** means ALB → EC2 traffic is plaintext by default. For end-to-end encryption, configure HTTPS on the target group and install a cert (can be self-signed) on the instances.

> [!tip] Exam Tip
> The TLS handshake uses **asymmetric encryption (RSA/ECDH) only for key exchange** — the actual data is encrypted with symmetric AES, which is far faster. The server's public key (from the cert) is used to securely establish a shared secret; after that, RSA is done.

## Key Service Properties to Memorize

| Service | Resilience level |
| --- | --- |
| [IAM](../services/iam.md) | Global |
| [VPC](../services/vpc.md) | Regional |
| IGW | Regional |
| Subnet | AZ-specific |
| NAT GW | Zonal (need one per AZ for HA) |
| [EC2](../services/ec2.md) instance | AZ-specific |

## Common Exam Traps

- [NACLs](../services/vpc.md#network-acl-nacl) apply to **subnet boundary** traffic — intra-subnet traffic is not filtered
- [Security Groups](../services/vpc.md#security-groups-sg) attach to **ENIs**, not instances or subnets
- NAT GW needs an **Elastic IP** (assigned automatically on creation)
- 1 VPC → at most 1 IGW; 1 IGW → at most 1 VPC
- DHCP Options Sets cannot be edited — create a new one and associate it
- [IAM](../services/iam.md) and network security ([SG](../services/vpc.md#security-groups-sg)/[NACL](../services/vpc.md#network-acl-nacl)) are independent layers — both must pass

## References

- [IAM](../services/iam.md)
- [VPC](../services/vpc.md)
- [EC2](../services/ec2.md)
- [Networking Concepts](../concepts/networking.md)
- [Identity and Access Concepts](../concepts/identity-and-access.md)
- [Route 53](../services/route53.md)
- [DNS Concepts](../concepts/dns.md)
- [TLS/HTTPS Concepts](../concepts/tls-https.md)
