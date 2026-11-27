---
title: Border Gateway Protocol (BGP)
type: concept
tags: [bgp, routing, autonomous-system, asn, as-path, as-path-prepending, local-preference, med, ebgp, ibgp, dynamic-routing, hybrid-networking, networking, bgp-authentication, tcp-md5, tcp-ao]
certs: [aws-saa]
updated: 2026-11-26
---

# Border Gateway Protocol (BGP)

**BGP** is a routing protocol. It decides how data gets from network A, through networks B and C, to network D. It is how the internet's core networks share routes with each other. In AWS, it is how routes are exchanged over [Site-to-Site VPN](../services/site-to-site-vpn.md) with dynamic routing and over AWS Direct Connect.

BGP is a large topic. For AWS you need the model on this page: who talks to whom, what they exchange, how a path is chosen, and how to influence that choice. For where BGP fits beside IPsec, see [Hybrid Connectivity](networking.md#hybrid-connectivity-vpn-and-bgp).

## Autonomous Systems and ASNs

BGP sees the world as a set of **autonomous systems (AS)**. An AS is a network, or a group of routers, controlled by one organization. BGP treats each AS as a **black box**: it does not care about the routers and routing inside it, only about which networks can be reached through it.

Each AS is identified by an **autonomous system number (ASN)**. Public ASNs are allocated through [IANA (Internet Assigned Numbers Authority)](https://www.iana.org/about) and the regional internet registries. Some ranges are **private**, and you can use them in your own peering without registering them:

| ASN format      | Full range      | Private range               |
| --------------- | --------------- | --------------------------- |
| 16-bit (2-byte) | 0–65,535        | 64,512–65,534               |
| 32-bit (4-byte) | 0–4,294,967,295 | 4,200,000,000–4,294,967,294 |

> [!note] ASNs are no longer only 16-bit
> The lecture describes ASNs as 16-bit, 0 to 65,535. That was the original format. 32-bit ASNs have been standard for years because the 16-bit pool ran low ([RFC 6793](https://www.rfc-editor.org/rfc/rfc6793)), and a 32-bit private range was added alongside the 16-bit one ([RFC 6996](https://www.rfc-editor.org/rfc/rfc6996)). AWS accepts both private ranges for the Amazon side of a VPN ([AWS: customer gateway options](https://docs.aws.amazon.com/vpn/latest/s2svpn/cgw-options.html)).

## How BGP Peers Work

- **TCP port 179.** BGP runs over TCP, so the session gets TCP's reliable, ordered delivery.
- **Peering is manual.** Two routers do not find each other. You configure a **peering** (a BGP session) between them on both sides, with each side's ASN. Matching configuration is not authentication; see [Securing BGP Sessions](#securing-bgp-sessions).
- **Peers share what they know.** Once peered, an AS advertises the networks it can reach. Anything an AS learns from one peer, it passes on to its other peers. Through this chain, every network learns about networks it is not directly connected to.
- **eBGP vs iBGP.** **eBGP** runs between different autonomous systems. **iBGP** runs between routers inside the same AS. Cloud hybrid connections, such as Site-to-Site VPN and Direct Connect, use eBGP.

BGP is a **path-vector** protocol. It tells its peers the **best path** it has to each destination, not every path it knows. That path is written as the **AS_PATH**: the list of ASNs the traffic would pass through.

By default, BGP pays no attention to link speed, latency, or load. It compares paths, not the quality of the links under them.

## Walking Through the Example

![Map of Australia with three BGP autonomous systems: Brisbane (ASN 200, 10.16.0.0/16), Adelaide (ASN 201, 10.17.0.0/16), and Alice Springs (ASN 202, 10.18.0.0/16). Brisbane to Adelaide and Adelaide to Alice Springs are 1 Gbps fibre links; Brisbane to Alice Springs is a 5 Mbps satellite link. Each site's route table lists destination, next hop, and AS path, with direct routes in pink and longer indirect routes in blue. A red entry in Brisbane's table shows the satellite route to 10.18.0.0/16 with AS path 202,202,202,i after AS path prepending, which makes the fibre path through Adelaide (201,202,i) shorter. Notes say BGP exchanges the shortest AS path, so Brisbane would use the satellite link by default even though the longer fibre would perform better, and that AS path prepending can make the satellite path look longer](../assets/bgp-as-path-architecture.png)

Three sites form a ring. Brisbane to Adelaide and Adelaide to Alice Springs are 1 Gbps fibre. Brisbane to Alice Springs is a 5 Mbps satellite link.

| Site          | ASN | Network      | Router    |
| ------------- | --- | ------------ | --------- |
| Brisbane      | 200 | 10.16.0.0/16 | 10.16.0.1 |
| Adelaide      | 201 | 10.17.0.0/16 | 10.17.0.1 |
| Alice Springs | 202 | 10.18.0.0/16 | 10.18.0.1 |

Each route table entry has three parts: the **destination**, the **next hop** (the router to send the traffic to), and the **AS_PATH**.

1. **Before peering**, each site knows only its own network. Brisbane's table has `10.16.0.0/16`, next hop `0.0.0.0` (locally connected), AS_PATH `i`. The `i` marks the **origin**: this network was learned from a locally connected network.
2. **Peerings are configured** between all three pairs of sites.
3. **Direct routes arrive.** When an AS advertises a route, it **prepends its own ASN** to the AS_PATH. Adelaide advertises `10.17.0.0/16` to Brisbane as `201, i` with next hop `10.17.0.1`. Alice Springs advertises `10.18.0.0/16` to Brisbane over the satellite link as `202, i`. The other sites learn the same way, and now every site can reach the other two.
4. **Indirect routes arrive.** Because it is a ring, there is a second way to each network. Adelaide passes its route to Alice Springs on to Brisbane as `201, 202, i`, with next hop `10.17.0.1`. This route goes through two autonomous systems.
5. **The shortest AS_PATH wins.** Brisbane now has two routes to Alice Springs: `202, i` over the satellite and `201, 202, i` over the fibre. It prefers the shorter one, the satellite. The longer route stays as a backup. If a site or link fails, BGP withdraws the routes through it and traffic moves to the remaining path.

BGP also prevents loops: a router rejects a route whose AS_PATH already contains its own ASN, so routes do not circle the ring forever.

### AS path prepending

The satellite is slower and has much higher latency, but BGP chooses it because the path is shorter. To make it a backup only, the Alice Springs administrator can use **AS path prepending**: add its own ASN extra times to the routes it advertises over the satellite link.

Alice Springs advertises `10.18.0.0/16` to Brisbane as `202, 202, 202, i`. Brisbane now compares a path of three ASNs over the satellite with `201, 202, i` over the fibre. The fibre path is shorter, so Brisbane sends traffic for Alice Springs through Adelaide. The satellite route is still there if the fibre fails.

> [!warning] Gotcha: prepending steers traffic coming in, not traffic going out
> Prepending changes how **other** networks reach you, because it changes the routes you advertise. In the example, Brisbane-to-Alice Springs traffic moves to the fibre. Traffic from Alice Springs to Brisbane still follows Alice Springs' own best route, which is still the direct satellite path (`200, i`). The two directions now take different paths. This is called **asymmetric routing**. To move both directions, Alice Springs must also prefer the fibre for its outbound traffic, usually with **local preference**, or Brisbane must prepend too.

### AS_PATH length is not the first tie-breaker

The lecture says BGP "decides everything based on path length". That is the default outcome when nothing else is set, but BGP's best-path selection has several steps. Two matter for AWS:

- **Local preference** is set inside your AS and is evaluated **before** AS_PATH length. It is the usual way to choose your outbound path ([AWS: Direct Connect routing policies and BGP communities](https://docs.aws.amazon.com/directconnect/latest/UserGuide/routing-and-bgp.html)).
- **MED (multi-exit discriminator)** is compared **after** AS_PATH. It is a hint from a neighbor about which of its entry points to use. A lower MED is preferred.

## Securing BGP Sessions

BGP has no strong authentication of its own, so any protection comes from the TCP session underneath or from the network the session runs over. How that works depends on where the BGP session runs.

### What BGP checks by itself

A session comes up only if both sides are configured for each other. Each router expects a specific neighbor IP address and ASN, and rejects a BGP OPEN message that does not match. This is a **configuration match, not authentication**. A device that can send packets from the expected IP address and claim the expected ASN can still attempt the session.

### Authentication at the TCP layer

The real authentication options sign the TCP segments that carry BGP:

| Option                                                                                     | How it works                                                                                                                                                      | Status                                                                                                                                 |
| ------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| **TCP MD5 signature** ([RFC 2385](https://www.rfc-editor.org/rfc/rfc2385))                 | Both peers configure the same password. Every TCP segment carries an MD5 digest of the segment and the password, and segments without a valid digest are dropped. | The most widely deployed option, but formally **obsolete**. The RFC itself notes that MD5 is "vulnerable to collision search attacks". |
| **TCP Authentication Option, TCP-AO** ([RFC 5925](https://www.rfc-editor.org/rfc/rfc5925)) | The replacement for MD5. It supports stronger algorithms and changing keys without resetting the session.                                                         | Current standard. A session uses either MD5 or TCP-AO, never both.                                                                     |

Both prove that a segment came from someone holding the key and was not altered. Neither one **encrypts** the routes: anyone who can see the traffic can still read them.

### BGP authentication in AWS

| Where the BGP session runs                                                   | What protects it                                                                                                                                                                                                                                                                                                                                                                                                                            |
| ---------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **AWS Direct Connect virtual interface**                                     | **MD5 is required.** AWS says "AWS enables MD5 by default. You cannot modify this option." You can provide your own MD5 key, or let AWS generate one ([AWS: Direct Connect virtual interfaces](https://docs.aws.amazon.com/directconnect/latest/UserGuide/WorkingWithVirtualInterfaces.html)).                                                                                                                                              |
| **[Site-to-Site VPN](../services/site-to-site-vpn.md) with dynamic routing** | **IPsec.** Each tunnel contains "an IKE security association, an IPsec security association, and a BGP peering" ([AWS: customer gateway device requirements](https://docs.aws.amazon.com/vpn/latest/s2svpn/CGRequirements.html)). The BGP session runs inside a tunnel whose peers already [authenticated](ipsec-vpn.md#ike-phase-1-authenticate-and-agree-on-a-key) with a pre-shared key or certificate, and the tunnel also encrypts it. |
| **Transit Gateway Connect** (GRE tunnel plus BGP)                            | AWS's documentation describes its BGP timers and addressing but says nothing about BGP authentication ([AWS: Transit Gateway Connect](https://docs.aws.amazon.com/vpc/latest/tgw/tgw-connect.html)). GRE is not encrypted, so the protection is the private network the tunnel runs over, usually Direct Connect.                                                                                                                           |

The pattern follows from the transport. Direct Connect is a private physical link but is not encrypted, so the BGP session gets its own MD5 key. A VPN is already authenticated and encrypted by IPsec, so the BGP session inside it relies on the tunnel.

> [!note] Site-to-Site VPN and BGP MD5
> AWS's device requirements, customer gateway, and example configuration pages do not mention a BGP MD5 key for Site-to-Site VPN (checked 2026-11-26). That means the documentation does not describe one, not that AWS explicitly forbids it.

## BGP in AWS

- **Site-to-Site VPN with dynamic routing** runs a BGP session over each IPsec tunnel between the AWS side and your customer gateway device ([Site-to-Site VPN: static vs dynamic routing](../services/site-to-site-vpn.md#static-vs-dynamic-routing)). Your device needs an ASN for this. A virtual private gateway uses ASN 64512 unless you choose a private ASN when you create it, and you cannot change it afterward ([AWS: how Site-to-Site VPN works](https://docs.aws.amazon.com/vpn/latest/s2svpn/how_it_works.html)).
- **AWS route choice.** When a virtual private gateway has the same prefix from several BGP-based VPN connections, it prefers the **shortest AS_PATH**, then the **lowest MED** ([AWS: route priority](https://docs.aws.amazon.com/vpn/latest/s2svpn/vpn-route-priority.html)).
- **AWS advice on prepending.** If your device supports asymmetric routing, AWS does **not** recommend AS_PATH prepending across the two tunnels. Equal AS_PATHs let AWS use the MED it sets during tunnel maintenance to pick the tunnel. Prepending and local preference are for devices that cannot handle asymmetric routing, and AWS warns that a change in the egress path "may cause traffic to drop" ([AWS: route priority](https://docs.aws.amazon.com/vpn/latest/s2svpn/vpn-route-priority.html)).
- **Direct Connect** uses BGP on its virtual interfaces ([AWS: Direct Connect routing policies](https://docs.aws.amazon.com/directconnect/latest/UserGuide/routing-and-bgp.html)). When a VPC route table has the same prefix from Direct Connect and from a VPN, it prefers the Direct Connect route ([AWS: route priority](https://docs.aws.amazon.com/vpn/latest/s2svpn/vpn-route-priority.html)). This is why a Site-to-Site VPN works as a backup for Direct Connect.

> [!tip] Exam Tip
>
> - **BGP = dynamic routing between autonomous systems** over **TCP 179**. Peering is configured manually on both sides.
> - **Private ASNs:** 64512–65534 (16-bit) and 4200000000–4294967294 (32-bit). A VGW's default ASN is **64512**.
> - **Shortest AS_PATH wins** by default; BGP ignores link speed. **AS path prepending** makes a path look longer so it becomes the backup.
> - "Routes learned automatically, failover between links, Direct Connect" points to **BGP** (dynamic routing). Static VPN routing has no BGP.
> - **BGP authentication:** Direct Connect BGP sessions always use an **MD5 key**. Site-to-Site VPN BGP sessions are protected by the **IPsec tunnel** they run inside.

## References

- Adrian Cantrill, AWS Certified Solutions Architect – Associate (SAA-C03): "Border Gateway Protocol (BGP) 101"
- [AWS: how Site-to-Site VPN works](https://docs.aws.amazon.com/vpn/latest/s2svpn/how_it_works.html)
- [AWS: customer gateway options](https://docs.aws.amazon.com/vpn/latest/s2svpn/cgw-options.html) (checked 2026-11-26)
- [AWS: route tables and Site-to-Site VPN route priority](https://docs.aws.amazon.com/vpn/latest/s2svpn/vpn-route-priority.html) (checked 2026-11-26)
- [AWS: Direct Connect routing policies and BGP communities](https://docs.aws.amazon.com/directconnect/latest/UserGuide/routing-and-bgp.html)
- [RFC 4271: A Border Gateway Protocol 4 (BGP-4)](https://www.rfc-editor.org/rfc/rfc4271)
- [RFC 6793: BGP Support for Four-Octet AS Number Space](https://www.rfc-editor.org/rfc/rfc6793)
- [RFC 6996: Autonomous System Reservation for Private Use](https://www.rfc-editor.org/rfc/rfc6996)
- [RFC 2385: Protection of BGP Sessions via the TCP MD5 Signature Option](https://www.rfc-editor.org/rfc/rfc2385) (obsoleted by RFC 5925)
- [RFC 5925: The TCP Authentication Option](https://www.rfc-editor.org/rfc/rfc5925)
- [AWS: Direct Connect virtual interfaces](https://docs.aws.amazon.com/directconnect/latest/UserGuide/WorkingWithVirtualInterfaces.html) (checked 2026-11-26)
- [AWS: customer gateway device requirements](https://docs.aws.amazon.com/vpn/latest/s2svpn/CGRequirements.html) (checked 2026-11-26)
- [AWS: Transit Gateway Connect](https://docs.aws.amazon.com/vpc/latest/tgw/tgw-connect.html) (checked 2026-11-26)
