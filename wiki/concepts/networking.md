---
title: Networking Concepts
type: concept
tags: [networking, eni, elastic-network-interface, osi, firewall, layer-7-firewall, waf, ephemeral-ports, ip-addresses, routing, hybrid-networking, vpn, ipsec, bgp]
certs: [aws-saa]
updated: 2026-11-26
---

# Networking Concepts

Cross-cutting networking ideas that span multiple AWS services. The foundational service is [VPC](../services/vpc.md); [EC2](../services/ec2.md) instances (and other resources) run inside it. For name resolution see [DNS](dns.md); for encrypted transport see [TLS/HTTPS](tls-https.md); for connecting on-premises networks see [Hybrid Connectivity](#hybrid-connectivity-vpn-and-bgp).

## Elastic Network Interfaces (ENIs)

An **Elastic Network Interface (ENI)** is the AWS-native equivalent of a virtual NIC. It is the **unit of network identity** in a VPC — IP addresses, DNS names, MAC address, [Security Groups](../services/vpc.md#security-groups-sg), and the source/destination check flag all attach to the ENI, *not* to whatever resource owns it.

- Almost every VPC-attached resource owns one or more ENIs: [EC2 instances](../services/ec2.md), RDS, ALB / NLB, ECS Fargate tasks, in-VPC Lambdas, MSK brokers
- Each ENI lives in **exactly one subnet** (and therefore one AZ); a single owner can have ENIs in different subnets *of the same AZ*
- An ENI's attributes survive being detached and re-attached to a different owner — that's how patterns like "float a service IP between instances" or "preserve a MAC-locked licence across replacement" work

For the per-attribute breakdown (primary vs secondary ENIs, MAC, private/public IPv4 + DNS, EIP, IPv6, SGs, source/dest check, split-horizon DNS), see [EC2 — Network Interfaces (ENIs)](../services/ec2.md#network-interfaces-enis) — EC2 is the canonical consumer and the diagrammed example.

## OSI Layers of AWS Network Controls

Understanding OSI layers clarifies *what each control can and cannot do*:

| AWS Construct             | OSI Layer | Attaches to                                   | Notes                                                                                                |
| ------------------------- | --------- | --------------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| Route table               | L3 only   | Subnet                                        | IP destinations and next hops; no port/protocol awareness                                            |
| NACL                      | L3/4      | Subnet (L3 construct)                         | Stateless; ICMP rules = L3, TCP/UDP rules = L4                                                       |
| Security Group            | L3/4      | ENI (L3/4 construct)                          | Stateful; same filter criteria as NACL                                                               |
| [WAF](../services/waf.md) | L7        | CloudFront / ALB / API GW / AppSync / Cognito | Inspects HTTP content: headers, URLs, body (see [Layer 7 Firewalls](#layer-7-application-firewalls)) |

Both [NACLs](../services/vpc.md#network-acl-nacl) and [Security Groups](../services/vpc.md#security-groups-sg) operate at L3/4 — neither inspects application-layer content. The real distinction between them is **stateful vs stateless** and **where they attach** (subnet vs ENI), not OSI layer.

## Firewall Layers

AWS networking enforces security at two complementary layers. See [Identity and Access Concepts](identity-and-access.md) for the third layer (IAM) which sits above the network entirely.

|                | NACL   | Security Group |
| -------------- | ------ | -------------- |
| OSI layer      | 3/4    | 3/4            |
| Attaches to    | Subnet | ENI            |
| Stateful?      | No     | Yes            |
| Explicit Deny? | Yes    | No             |

Best practice: use [Security Groups](../services/vpc.md#security-groups-sg) as the primary control; add [NACLs](../services/vpc.md#network-acl-nacl) only when you need explicit IP blocking.

## Layer 7 (Application) Firewalls

A firewall can only understand the layers it operates at and the layers below. That one fact explains what each kind of firewall can and cannot do.

### What Layer 3/4 and Session-Aware Firewalls See

![Two firewalls between a laptop and a server. Top: a Layer 3/4 firewall sees the request from laptop to server and the response from server to laptop as two separate, unrelated flows of packets and segments. Bottom: a firewall with Layer 5 session capability treats request and response as one session, which reduces admin overhead and allows contextual security. A red band across the middle says the application layer (Layer 7) is opaque to the layers below and is just data](../assets/l3-l4-l5-firewalls.png)

A **Layer 3/4 firewall** sees packets and segments: IP addresses, protocols, ports, and TCP flags. It sees a request from the client and a response from the server as **two separate, unrelated flows**, so it needs one rule for each direction. This is how a [NACL](../services/vpc.md#network-acl-nacl) behaves, and why NACLs need [ephemeral port](#ephemeral-ports) rules for return traffic.

A firewall that tracks **sessions** sees the request and its response as **one conversation**. This means one rule instead of two, and it allows contextual security: the firewall can treat a packet differently depending on whether it is a reply to a request it already allowed. A [security group](../services/vpc.md#security-groups-sg) works this way. The lecture calls this "Layer 5 (session) capability". In practice it is called a **stateful** firewall, and the state it tracks is the Layer 4 connection, not the OSI session layer.

Both kinds share the same limit: the application data on top is **opaque**. HTTP headers, URLs, and request bodies are just bytes. To these firewalls, a normal product image, an SQL injection attempt, and a malware download look the same, as long as they use an allowed IP and port.

### What a Layer 7 Firewall Adds

![A Layer 7 firewall between a laptop and a web server. The client's HTTPS connection is terminated and decrypted on the firewall, and a new encrypted connection is made from the firewall to the backend. Between the two, the firewall reads plain HTTP and can identify normal or abnormal requests and protocol-specific attacks. Data can be inspected and blocked, replaced, or tagged (adult content, spam, off-topic, malware, exploits), and specific applications such as Facebook can be identified, blocked, or adjusted. The firewall keeps all Layer 3, 4, and 5 features but can react to Layer 7 elements such as DNS names, rate, content, and headers](../assets/l7-firewall.png)

A **Layer 7 firewall** understands one or more application protocols, such as HTTP. It keeps everything the lower-layer firewalls do and adds:

- **Protocol awareness.** It knows how a valid HTTP request is built (method, host, headers, body), so it can tell a normal request from an abnormal one and stop protocol-specific attacks.
- **Visibility into encrypted traffic.** For HTTPS, the client's TLS connection ends at the firewall. The firewall reads the request as plain HTTP, then opens a **new** encrypted connection to the backend server. The client and server do not notice. See [TLS/HTTPS](tls-https.md) for how termination works.
- **Content decisions.** Data can be inspected and **blocked**, **replaced** (for example, swapping an inappropriate image for a placeholder), or **tagged**. Rules can target spam, adult content, malware, and exploits, or specific applications such as Facebook. They can also stop business data from leaving for services such as Dropbox.
- **Layer 7 criteria.** Rules can use DNS names, request rate, content, and headers.

What a Layer 7 firewall understands depends on its software. Some understand only HTTP; others understand protocols such as SMTP for email.

### Layer 7 Firewalls in AWS

| Option                                                                                                             | What it is                                                                                         | Where it inspects                                           |
| ------------------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------- | ----------------------------------------------------------- |
| [AWS WAF](../services/waf.md)                                                                                      | Managed HTTP/HTTPS firewall attached to CloudFront, ALB, API Gateway, AppSync, Cognito, and others | Inbound web requests, after the host service terminates TLS |
| [Gateway Load Balancer](../services/elb.md#gwlb-vs-waf--complementary-not-substitutes) plus third-party appliances | Routes all IP traffic through vendor firewalls that may understand many protocols                  | Inbound, outbound, and east-west traffic of any protocol    |

AWS WAF does not decrypt traffic itself. The service it is attached to, such as CloudFront or an ALB, terminates HTTPS and hands WAF the plain request. This is why WAF cannot be attached to a [Network Load Balancer](../services/elb.md) passing TLS through, or to an EC2 instance directly.

A Layer 7 firewall also defends against application-layer DDoS attacks, such as floods of expensive search requests, which look like normal HTTP to lower-layer controls. [AWS Shield](../services/shield.md) covers Layer 3 and 4 DDoS attacks automatically and uses WAF for Layer 7 protection in its Advanced tier.

## Observing Network Traffic

[VPC Flow Logs](../services/vpc-flow-logs.md) makes ENI traffic visible as delayed metadata. VPC/subnet scopes widen the set of interfaces monitored; they do not turn the logs into packet captures. [CloudWatch Logs](../services/cloudwatch.md) supports operational log analysis, while [S3](../services/s3.md) supports stored-log analysis with [Athena](../services/athena.md).

Firewall statefulness and logging direction are separate ideas: an allowed request and its reply can produce separate records even though the security group tracks one conversation. In the [ICMP example](../services/vpc-flow-logs.md#reading-request-and-response-records), an accepted request and a rejected reply expose an outbound NACL restriction. Correlate the traffic and inspect rules rather than inferring the firewall type from the number of records.

## Routing Hierarchy

Traffic flows through these components in order:

1. **Security Group** — outbound check on the source ENI (L3/4, stateful)
2. **Subnet Route Table** — determines next hop (L3 only)
3. **VPC Router** — moves traffic between subnets or to a gateway (L3)
4. **NACL** — inbound check on the destination subnet boundary (L3/4, stateless)
5. **Security Group** — inbound check on the destination ENI (L3/4, stateful)

## Direct Connectivity vs Transit Routing

[VPC peering](../services/vpc-peering.md) creates one direct relationship between two non-overlapping VPCs. Routes select the peering connection as the next hop, and both endpoints need a return path. A–B and B–C peering do not permit A → B → C: B cannot act as a transit router between its peers. Full-mesh connectivity among n VPCs requires n(n − 1)/2 peering connections, which makes topology and route management increasingly complex as the network grows.

Peering also does not share a peer's internet/NAT gateway, VPN, Direct Connect connection, or [gateway endpoint](../services/vpc-gateway-endpoints.md). For centralized transit, evaluate a dedicated routing service such as [AWS Transit Gateway](https://docs.aws.amazon.com/vpc/latest/tgw/what-is-transit-gateway.html). Separately configure [DNS](dns.md) visibility: a route to a private IP does not automatically expose the destination's private hosted zone.

## Hybrid Connectivity: VPN and BGP

Connecting an on-premises network to a VPC raises three separate questions, and each has its own page:

1. **How is the traffic protected on the way?** An [IPsec](ipsec-vpn.md) tunnel encrypts it across the public internet. IKE Phase 1 authenticates the two peers and agrees on a key; Phase 2 builds the security associations that carry the data.
2. **How does each side learn the other's networks?** Either you type the routes in (static routing), or the two sides exchange them with [BGP](bgp.md) (dynamic routing). BGP also notices when a path fails and withdraws its routes.
3. **Which AWS product terminates the connection?** [AWS Site-to-Site VPN](../services/site-to-site-vpn.md) builds two IPsec tunnels from a virtual private gateway or transit gateway to your customer gateway device. AWS Direct Connect is a private physical link instead of the internet. A Site-to-Site VPN can back it up or encrypt traffic over it.

The VPC side still follows the [routing hierarchy](#routing-hierarchy) above. Routes for the on-premises networks must reach the subnet route tables, by static entries or by route propagation, and security groups and NACLs must still allow the traffic.

## Internet Connectivity Patterns

| Scenario                                         | Solution                                                                                         |
| ------------------------------------------------ | ------------------------------------------------------------------------------------------------ |
| Private subnet → internet (IPv4)                 | NAT GW in public subnet + route `0.0.0.0/0 → NAT GW`                                             |
| Public subnet → internet (IPv4)                  | IGW + route `0.0.0.0/0 → IGW`                                                                    |
| IPv6 bidirectional                               | IGW + route `::/0 → IGW`                                                                         |
| IPv6 outbound only                               | Egress-Only IGW + route `::/0 → EIGW`                                                            |
| Private → same-Region S3/DynamoDB (no internet)  | [Gateway endpoint](../services/vpc-gateway-endpoints.md) + service-prefix-list route             |
| Private → other supported services (no internet) | [PrivateLink interface endpoint](../services/vpc-interface-endpoints.md) + endpoint ENIs and DNS |

See [VPC](../services/vpc.md) for the full details on each of these components.

## Private Service Access: Routing vs Addressing

[Gateway endpoints](../services/vpc-gateway-endpoints.md) show why a public destination address does not necessarily require public internet connectivity. The client resolves the service endpoint, and a route matching the AWS-managed service prefix list directs traffic to the gateway endpoint. A [PrivateLink interface endpoint](../services/vpc-interface-endpoints.md), by comparison, presents private [ENI](#elastic-network-interfaces-enis) addresses in chosen subnets.

For an interface endpoint, DNS supplies an endpoint ENI address and the ordinary network routes carry the packets; “uses DNS” does not mean “does not use routing.” A [private hosted zone for SNS](../services/vpc-interface-endpoints.md#sns-walkthrough-keep-the-hostname-change-the-dns-answer) can preserve the application's service hostname while choosing that private destination. See the [gateway/interface comparison](../services/vpc-interface-endpoints.md#gateway-endpoint-vs-interface-endpoint) for the distinct availability, security-group, and cost models.

A prefix list is a reusable set of CIDRs: the route destination references the list, while the route target identifies the next hop. Routes determine reachability; [IAM and resource policies](resource-vs-identity-policies.md) independently determine authorization. A gateway endpoint policy constrains requests using that path; a bucket policy can enforce the required path. Neither creating the endpoint nor restricting its policy removes other egress routes.

## Ephemeral Ports

When a client connects to a server, the OS assigns a random **ephemeral (source) port** for the return traffic — typically in the range **1024–65535**.

[NACLs](../services/vpc.md#network-acl-nacl) are stateless, so **both directions must be explicitly allowed**:

- Inbound rule: allow destination port (e.g. 443)
- Outbound rule: allow ephemeral port range (1024–65535) back to the client

[Security Groups](../services/vpc.md#security-groups-sg) handle this automatically (stateful).

## IP Address Types

| Type             | Scope                   | Configured in OS?  |
| ---------------- | ----------------------- | ------------------ |
| Private IPv4     | Within VPC              | Yes                |
| Public IPv4      | Internet-routable       | No — mapped at IGW |
| Elastic IP (EIP) | Static public IPv4      | No — mapped at IGW |
| IPv6             | Globally unique, public | Yes                |

> [!tip] Exam Tip
> A public IPv4 address is **not visible inside the EC2 OS** — the OS only sees the private IP. The IGW performs the 1:1 NAT translation. IPv6 is different: it's configured directly in the OS.

## Anycast and Global Application Entry

[Global Accelerator](../services/global-accelerator.md) advertises stable anycast addresses from AWS edges. Traffic reaches an edge and travels over the backbone to a configured regional endpoint. Those addresses belong to the accelerator entry, not an EC2 ENI. [CloudFront](../services/cloudfront.md) instead adds HTTP caching and content controls; [Edge Delivery](edge-delivery.md) explains how these services combine network paths, DNS discovery, and application security.

## References

- [VPC](../services/vpc.md)
- [EC2](../services/ec2.md)
- [Identity and Access Concepts](identity-and-access.md)
- [DNS](dns.md)
- [TLS/HTTPS](tls-https.md)
- [AWS WAF](../services/waf.md)
- [AWS Shield](../services/shield.md)
- Adrian Cantrill, AWS Certified Solutions Architect – Associate (SAA-C03): "Application Layer (L7) Firewall"
- [IPsec VPN Fundamentals](ipsec-vpn.md)
- [BGP](bgp.md)
- [AWS Site-to-Site VPN](../services/site-to-site-vpn.md)
