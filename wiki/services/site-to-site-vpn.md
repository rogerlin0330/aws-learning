---
title: AWS Site-to-Site VPN
type: service
tags: [site-to-site-vpn, pfsense, demo, vpn, ipsec, bgp, virtual-private-gateway, vgw, customer-gateway, cgw, transit-gateway, hybrid-networking, high-availability, route-propagation, ecmp, direct-connect, accelerated-vpn, vpn-concentrator]
certs: [aws-saa]
updated: 2026-11-26
---

# AWS Site-to-Site VPN

AWS Site-to-Site VPN (S2S VPN) is a **logical connection between a [VPC](vpc.md) and a network outside AWS**, such as an office, a data center, or another cloud. The traffic is encrypted with [IPsec](../concepts/ipsec-vpn.md) because it normally travels over the **public internet**. Resources on both sides then talk to each other using **private IP addresses**.

It is the quickest way to connect an outside network to AWS. Everything is software configuration, so you can have a connection working in **under an hour**, compared with the weeks or months a physical AWS Direct Connect link can take.

Three things decide whether it works well:

1. **High availability.** AWS gives you two tunnels in different Availability Zones, but your own router can still be a single point of failure.
2. **Routing.** Routes are either typed in (static) or exchanged with [BGP](../concepts/bgp.md) (dynamic).
3. **Limits of the internet.** Bandwidth is capped per tunnel, and latency varies.

For how these pieces fit with the rest of AWS networking, see [Hybrid Connectivity](../concepts/networking.md#hybrid-connectivity-vpn-and-bgp).

> [!note] VPN over Direct Connect
> "Over the public internet" is the default. You can also run Site-to-Site VPN over a Direct Connect connection, to encrypt traffic on that private link. See [VPN and Direct Connect](#vpn-and-direct-connect).

## Components

| Component                           | What it is                                                                                                                                                   |
| ----------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **VPC**                             | The AWS network you want private connectivity to                                                                                                             |
| **Virtual private gateway (VGW)**   | The VPN endpoint on the AWS side. You create it and attach it to **one** VPC. It is a gateway object that route tables can target, like an internet gateway. |
| **Transit gateway / Cloud WAN**     | Alternative AWS-side endpoints. A transit gateway is a regional hub that many VPCs and VPNs attach to.                                                       |
| **Customer gateway device**         | The router or firewall on your side, physical or virtual                                                                                                     |
| **Customer gateway (CGW) resource** | The AWS object that describes that device: its public IP address, and its ASN if you use BGP                                                                 |
| **VPN connection**                  | The configuration that links **one** AWS-side gateway to **one** customer gateway. Creating it creates **two VPN tunnels**.                                  |
| **VPN tunnel**                      | An encrypted link from one AWS endpoint to your device. Data flows both ways.                                                                                |

The term **customer gateway** can mean either the AWS object or the real device. The lecture uses it for both. AWS's documentation keeps them apart as "customer gateway" and "customer gateway device" ([AWS: how Site-to-Site VPN works](https://docs.aws.amazon.com/vpn/latest/s2svpn/how_it_works.html)).

The lecture uses a virtual private gateway throughout. That is still the simplest way to connect one VPC. A transit gateway is the usual choice when many VPCs share the connection, and some newer features only work there (see [VGW vs Transit Gateway](#virtual-private-gateway-vs-transit-gateway)).

### Customer gateway requirements

- **A static public IP address**, IPv4 or IPv6. IPv6 outer addresses work only with a transit gateway or Cloud WAN ([AWS: customer gateway options](https://docs.aws.amazon.com/vpn/latest/s2svpn/cgw-options.html)).
- **Behind NAT:** use the public IP of the NAT device and allow **UDP 500 and UDP 4500** so NAT traversal can work. If you authenticate with a private certificate from AWS Private CA instead of a pre-shared key, AWS lets you leave the IP address out ([AWS: customer gateway options](https://docs.aws.amazon.com/vpn/latest/s2svpn/cgw-options.html)).
- **An ASN** only if you use dynamic routing. A static-routing customer gateway does not need one. AWS reserves some ASNs, such as 7224 in every Region ([AWS: customer gateway options](https://docs.aws.amazon.com/vpn/latest/s2svpn/cgw-options.html)).
- **A route-based IPsec setup**, or a policy-based one limited to a single security association pair per tunnel ([IPsec: what this means on AWS](../concepts/ipsec-vpn.md#what-this-means-on-aws)).

## Architecture

![Single customer gateway design. A VPC (10.16.0.0/16) with private subnets in AZ-A, AZ-B, and AZ-C connects through the VPC router to a virtual private gateway. The VGW's VPN connection has endpoint 1 and endpoint 2 in the AWS public zone, each with a tunnel across the public internet to one customer gateway with external IP 1.3.3.7 in the corporate office (192.168.10.0/24). Static config 192.168.10.0/24 is set on the AWS side and 10.16.0.0/16 on the customer side. The VGW is labeled highly available and the customer gateway is labeled a single point of failure](../assets/site-to-site-vpn-architecture.png)

The lecture's example connects the Aussie Mart VPC (`10.16.0.0/16`, private subnets in three AZs) to its corporate office (`192.168.10.0/24`).

1. **Gather the information.** You need the VPC's CIDR, the on-premises network's CIDR, and the public IP of the on-premises router.
2. **Create a virtual private gateway** and attach it to the VPC.
3. **Create a customer gateway** with the router's public IP, so AWS knows where to build the tunnels.
4. **Create a VPN connection** that links the VGW and the customer gateway, and choose static or dynamic routing. AWS creates **two tunnels**, one from each of two AWS endpoints. Each endpoint has its own public IP and sits in a **different Availability Zone** ([AWS: tunnel options](https://docs.aws.amazon.com/vpn/latest/s2svpn/VPNTunnels.html)).
5. **Configure your device** with both tunnels. AWS provides a downloadable configuration file for common devices.
6. **Add routes.** In this static example, the AWS side is told about `192.168.10.0/24` and your router is told about `10.16.0.0/16`. The VPC route tables need a route to `192.168.10.0/24` with the VGW as the target, added by hand or by [route propagation](#route-propagation).
7. **Allow the traffic.** [Security groups and NACLs](vpc.md#security-groups-sg) must still permit it. The VPN only provides the path.

Traffic then flows from an instance, through the VPC router, to the VGW, over a tunnel, to the office, and back again. As long as **one** tunnel is up, the two networks are connected.

## High Availability

The AWS side of this design is already highly available. The VGW is a highly available gateway object, and its two endpoints are in different AZs, so losing one AZ leaves one tunnel working.

The design as a whole is **not** highly available. Both tunnels end at **one customer router** on **one internet connection**. If that router or line fails, the whole connection is down.

![Highly available design. The same VPC and VGW now have two VPN connections. The first connection's two endpoints have tunnels to customer gateway 1 (1.3.3.7). A second VPN connection adds two more endpoints with tunnels to customer gateway 2 (1.3.3.8). Both the AWS side and the customer side are labeled highly available](../assets/site-to-site-vpn-ha-architecture.png)

To make it fully highly available:

1. Add a **second customer router**, with a **second internet connection**, ideally in a **second building**.
2. Create a second customer gateway for it.
3. Create a **second VPN connection** from the same VGW to the new customer gateway. AWS creates two more endpoints, each with a public IP, and two more tunnels.

You now have **four tunnels**: two from separate AZs to each customer router. Either customer router can fail, and either AZ can fail, and the networks stay connected.

> [!warning] Gotcha: configure both tunnels on every device
> AWS sometimes takes one tunnel endpoint out of service for patches, hardware retirement, or replacement. It does this **one tunnel at a time**, and it also replaces endpoints when you change the connection. If your device has only one tunnel configured, routine AWS maintenance takes your connection down ([AWS: tunnel endpoint replacements](https://docs.aws.amazon.com/vpn/latest/s2svpn/endpoint-replacements.html); [AWS: your customer gateway device](https://docs.aws.amazon.com/vpn/latest/s2svpn/your-cgw.html)).

### Which tunnel carries traffic

Two tunnels do not mean double the bandwidth. Traffic **from AWS to your network prefers one tunnel**. On a virtual private gateway, AWS picks **one tunnel across all the VPN connections on that gateway** as the egress path. With BGP, you can tell which one from the MED value AWS sets on it. With static routing, you can tell from the traffic statistics ([AWS: route priority](https://docs.aws.amazon.com/vpn/latest/s2svpn/vpn-route-priority.html)).

Traffic from your side can use both tunnels, and the primary tunnel can change. AWS therefore "strongly" recommends devices that support **asymmetric routing**: traffic leaving on one tunnel and returning on the other ([AWS: route priority](https://docs.aws.amazon.com/vpn/latest/s2svpn/vpn-route-priority.html)).

To use several tunnels at once (active-active), you need a transit gateway with ECMP. See [Bandwidth and Performance](#bandwidth-and-performance).

## Static vs Dynamic Routing

![Static vs dynamic routing for the path VPC subnet, VPC router, VGW, CGW (192.168.10.0/24). Static: routes for the remote side are added to route tables as static routes, the remote networks are statically configured on the VPN connection, and there is no load balancing or multi-connection failover. Dynamic: BGP is configured on both the customer and AWS side using ASNs, and networks are exchanged via BGP](../assets/site-to-site-vpn-static-vs-dynamic.png)

The architecture is the same for both. The difference is **how each side learns the other side's networks**.

|                              | Static routing                                                                                        | Dynamic routing (BGP)                                                       |
| ---------------------------- | ----------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------- |
| How networks are learned     | You list the on-premises prefixes on the VPN connection and configure the AWS prefixes on your router | A BGP session over each tunnel exchanges prefixes automatically             |
| Device requirement           | IPsec only, so it works with almost any router                                                        | The device must support BGP, and needs an ASN                               |
| Failover between tunnels     | Based on tunnel health. Dead peer detection takes a tunnel's routes away when it fails.               | BGP also withdraws routes when a session drops                              |
| Several connections or links | Limited. No ECMP on a transit gateway.                                                                | Supports ECMP, multiple connections, and Direct Connect backup              |
| Route limits on a VGW        | 100 static routes from the customer gateway                                                           | 100 dynamic routes from the customer gateway; 1,000 routes advertised to it |

If your router does not support BGP, you must use static routing. If you need advanced high availability, several connections, or a VPN working alongside Direct Connect, use dynamic routing. On a VGW you cannot mix them: with BGP, you cannot also add static routes to the connection ([AWS: routing options](https://docs.aws.amazon.com/vpn/latest/s2svpn/VPNRoutingTypes.html)).

The BGP session needs no password of its own. It runs inside the IPsec tunnel, whose peers already authenticated with a pre-shared key or certificate, and the tunnel encrypts it ([AWS: customer gateway device requirements](https://docs.aws.amazon.com/vpn/latest/s2svpn/CGRequirements.html)). Direct Connect is different: its BGP sessions always use an MD5 key. See [Securing BGP Sessions](../concepts/bgp.md#securing-bgp-sessions).

> [!note] Refining "no multi-connection failover"
> The slide says static VPNs have "no load balancing and multi connection failover". Static routing still fails over between the **two tunnels of one connection** based on tunnel health. What it lacks is a way to exchange routes and path information, so it cannot use ECMP on a transit gateway, and it is not the right choice when traffic must shift between several connections or between a VPN and Direct Connect ([AWS: quotas, ECMP](https://docs.aws.amazon.com/vpn/latest/s2svpn/vpn-limits.html)).

Transit gateway limits are higher: 1,000 dynamic routes from the customer gateway and 5,000 routes advertised to it ([AWS: quotas](https://docs.aws.amazon.com/vpn/latest/s2svpn/vpn-limits.html), checked 2026-11-26).

### Route propagation

Without route propagation, you add the on-premises routes to each VPC route table by hand. With **route propagation** turned on for a route table, AWS adds the VPN's routes for you, as **propagated routes**, while the VPN is up. This works for both types: the static prefixes of a static VPN, and the BGP-learned routes of a dynamic VPN ([AWS: routing options](https://docs.aws.amazon.com/vpn/latest/s2svpn/VPNRoutingTypes.html)).

### Route priority

When several routes match, the VPC route table picks one in this order ([AWS: route priority](https://docs.aws.amazon.com/vpn/latest/s2svpn/vpn-route-priority.html)):

1. **Longest prefix match**, among routes with healthy endpoints.
2. The **local VPC route** beats any overlapping propagated route, even a more specific one.
3. For an identical prefix: **Direct Connect BGP routes**, then **static VPN routes**, then **BGP VPN routes**.
4. Between BGP VPN routes for the same prefix: the **shortest AS_PATH**, then the **lowest MED**.

## Tunnel Initiation and Dead Peer Detection

By default, **your device starts** the IKE negotiation that brings a tunnel up. A tunnel with no traffic can go down. To keep it up, send regular traffic, or set the tunnel's startup action so that AWS starts the negotiation. AWS-side initiation works with **IKEv2 only** and needs your device's public IP ([AWS: tunnel initiation options](https://docs.aws.amazon.com/vpn/latest/s2svpn/initiate-vpn-tunnels.html)).

**Dead peer detection (DPD)** notices when the other end stops responding. By default, AWS then ends the IKE session, takes the tunnel down, and removes its routes. You can instead have AWS restart IKE or do nothing. DPD is required for dynamically routed connections ([AWS: tunnel initiation options](https://docs.aws.amazon.com/vpn/latest/s2svpn/initiate-vpn-tunnels.html); [AWS: customer gateway device requirements](https://docs.aws.amazon.com/vpn/latest/s2svpn/CGRequirements.html)).

## Bandwidth and Performance

> [!note] The 1.25 Gbps limit is per tunnel
> The lecture says "a single VPN connection with two tunnels has a maximum throughput of 1.25 gigabits per second". AWS states the limit **per tunnel**: up to 1.25 Gbps and up to 140,000 packets per second ([AWS: quotas](https://docs.aws.amazon.com/vpn/latest/s2svpn/vpn-limits.html), checked 2026-11-26). On a virtual private gateway the practical result is close to the lecture's figure, because AWS sends its traffic over one tunnel and a VGW does not support ECMP.

| Need                                       | Option                                                                                                                                                                                                                            |
| ------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Up to 1.25 Gbps                            | A standard tunnel, on a VGW or transit gateway                                                                                                                                                                                    |
| Up to 5 Gbps per tunnel                    | **Large Bandwidth Tunnels**, transit gateway or Cloud WAN only, both tunnels set the same, up to 400,000 packets per second ([AWS: tunnel options](https://docs.aws.amazon.com/vpn/latest/s2svpn/VPNTunnels.html); launched 2025) |
| More than one tunnel's worth               | **ECMP** on a transit gateway: traffic is spread across tunnels and connections. Requires **dynamic routing**. A single flow still uses one tunnel ([AWS: quotas](https://docs.aws.amazon.com/vpn/latest/s2svpn/vpn-limits.html)) |
| Consistent high throughput and low latency | AWS Direct Connect                                                                                                                                                                                                                |

Your **customer gateway device** may be the real limit. Encryption and decryption take significant processing, and many routers cannot reach these speeds.

**Packet size:** the maximum MTU through a tunnel is 1,446 bytes (maximum MSS 1,406 bytes). Jumbo frames are not supported ([AWS: quotas](https://docs.aws.amazon.com/vpn/latest/s2svpn/vpn-limits.html)). AWS's device guide recommends a 1,399-byte MTU on the tunnel interface ([AWS: customer gateway device requirements](https://docs.aws.amazon.com/vpn/latest/s2svpn/CGRequirements.html)).

### Latency and Accelerated Site-to-Site VPN

The tunnels cross the public internet, so there may be many hops between you and AWS. Each hop adds latency and variation. For a latency-sensitive application, consider Direct Connect.

**Accelerated Site-to-Site VPN** reduces the internet portion of the path. AWS creates two [Global Accelerator](global-accelerator.md) accelerators, one per tunnel, so your traffic enters the AWS network at the nearest edge location. It works only on a **transit gateway**, needs NAT traversal, and must be chosen when you create the connection. You cannot turn it on or off later ([AWS: accelerated VPN](https://docs.aws.amazon.com/vpn/latest/s2svpn/accelerated-vpn.html)).

## Virtual Private Gateway vs Transit Gateway

|                                       | Virtual private gateway        | Transit gateway                                     |
| ------------------------------------- | ------------------------------ | --------------------------------------------------- |
| Connects                              | One VPC                        | Many VPCs, VPNs, and Direct Connect through one hub |
| Use of several tunnels at once (ECMP) | No                             | Yes, with BGP                                       |
| Large Bandwidth Tunnels (5 Gbps)      | No                             | Yes (also Cloud WAN)                                |
| Accelerated VPN                       | No                             | Yes                                                 |
| IPv6 tunnels                          | No                             | Yes                                                 |
| VPN Concentrator                      | No                             | Yes                                                 |
| Extra cost                            | None beyond the VPN connection | A per-hour attachment fee and per-GB processing     |

Sources: [AWS: quotas](https://docs.aws.amazon.com/vpn/latest/s2svpn/vpn-limits.html), [AWS: accelerated VPN](https://docs.aws.amazon.com/vpn/latest/s2svpn/accelerated-vpn.html), [AWS: customer gateway options](https://docs.aws.amazon.com/vpn/latest/s2svpn/cgw-options.html), [AWS: VPN Concentrators](https://docs.aws.amazon.com/vpn/latest/s2svpn/vpn-concentrator.html).

**VPN Concentrator** (November 2025) is for many small sites. A single transit gateway attachment carries up to 100 low-bandwidth sites (up to 100 Mbps each, 5 Gbps in total). Each site still gets its own VPN connection with two tunnels. It supports BGP only, with no static routing and no ECMP ([AWS: VPN Concentrators](https://docs.aws.amazon.com/vpn/latest/s2svpn/vpn-concentrator.html)).

## VPN and Direct Connect

AWS Direct Connect is a dedicated physical link to AWS. It is faster and more consistent than a VPN, but it can take weeks or months to provision. The two are often used together:

- **VPN first, Direct Connect later.** Set up a VPN to get connected in hours, order Direct Connect, then replace the VPN or keep both.
- **VPN as the backup.** Instead of a second physical link, use a VPN as the secondary path. With BGP on both, a VPC route table prefers the Direct Connect route for the same prefix and falls back to the VPN when Direct Connect withdraws it ([AWS: route priority](https://docs.aws.amazon.com/vpn/latest/s2svpn/vpn-route-priority.html)).
- **VPN over Direct Connect.** Direct Connect is private but not encrypted by itself. Running IPsec over it adds encryption. With **Private IP VPN**, the tunnels use private IP addresses over a Direct Connect transit virtual interface to a transit gateway, with no public IPs ([AWS: Private IP VPN](https://docs.aws.amazon.com/vpn/latest/s2svpn/private-ip-dx.html)).

## Monitoring

Site-to-Site VPN publishes CloudWatch metrics per tunnel and per connection ([AWS: CloudWatch metrics](https://docs.aws.amazon.com/vpn/latest/s2svpn/monitoring-cloudwatch-vpn.html)):

- **`TunnelState`:** 1 when a tunnel is up (or its BGP session established), 0 when it is not. For a connection, a value between 0 and 1 means at least one tunnel is down. An alarm on this catches a lost redundant tunnel before the second one fails.
- **`TunnelDataIn` and `TunnelDataOut`:** bytes received and sent on the AWS side.

## Pricing

You pay for each **VPN connection-hour** while the connection exists, whether or not traffic flows, plus normal **data transfer out** charges. AWS's US East (Ohio) examples, checked 2026-11-26 ([AWS: VPN pricing](https://aws.amazon.com/vpn/pricing/)):

| Item                              | Example rate                                                                |
| --------------------------------- | --------------------------------------------------------------------------- |
| Standard VPN connection           | $0.05 per connection-hour                                                   |
| Large Bandwidth Tunnel connection | $0.60 per connection-hour                                                   |
| Transit gateway VPN attachment    | $0.05 per attachment-hour, on top of the connection                         |
| VPN Concentrator                  | $1.95 per hour, plus $0.01 per remote site per hour                         |
| Accelerated VPN                   | Two Global Accelerators at $0.025 per hour each, plus Data Transfer Premium |
| Data transfer out                 | Standard EC2 data transfer rates                                            |

Also count your **own internet costs**. If your line has a data cap, heavy VPN traffic counts against it.

## Quotas

Default quotas per Region, checked 2026-11-26; all can be raised ([AWS: quotas](https://docs.aws.amazon.com/vpn/latest/s2svpn/vpn-limits.html)):

- 50 customer gateways and 50 VPN connections
- 5 virtual private gateways
- 10 VPN connections per virtual private gateway

## Exam Recap

> [!tip] Exam Tip
>
> - **"Quick", "encrypted", "over the internet", "hours not weeks"** points to Site-to-Site VPN. **"Consistent latency", "high bandwidth", "doesn't use the internet"** points to Direct Connect.
> - **Components:** VPC, **VGW** (attached to one VPC), **CGW** (your router's public IP, plus an ASN for BGP), and a **VPN connection** that creates **two tunnels** to endpoints in **different AZs**.
> - **HA:** the AWS side is already HA. The single point of failure is **your router**. Fix it with a second customer gateway, second internet line, and second VPN connection, giving four tunnels.
> - **Static** = routes typed in, works with any IPsec device. **Dynamic** = **BGP**, needed for ECMP, several connections, and Direct Connect backup. **Route propagation** adds VPN routes to VPC route tables automatically for both.
> - **1.25 Gbps per tunnel.** For more: transit gateway with **ECMP** (BGP required), **Large Bandwidth Tunnels** (5 Gbps, transit gateway or Cloud WAN), or Direct Connect.
> - **Lower latency across the internet:** use **Accelerated Site-to-Site VPN** (Global Accelerator, transit gateway only).
> - **Encrypt Direct Connect traffic:** run a VPN over Direct Connect (**Private IP VPN** with a transit gateway).
> - **Same prefix from Direct Connect and a VPN:** the Direct Connect route wins, and the VPN is the backup.

## Demo: Simple Site-to-Site VPN with pfSense

This section follows the course's "Simple Site2Site VPN" mini project. It builds a **static-routing** VPN from a virtual private gateway to a **pfSense** router in a simulated on-premises network, then proves it works by browsing from an on-premises server to a server in the VPC. It was recorded in 2022 in US East (N. Virginia), so console screens may look different now. This page describes the lesson; the demo was not re-run for the wiki.

### Demo architecture

![Simple Site-to-Site VPN demo. Left: the AWS VPC (10.16.0.0/16) with awsServerA in AZ A and a subnet in AZ B whose route table gets routes through route propagation, both behind the VPC router and a virtual private gateway. Middle: VPN endpoint 1 and VPN endpoint 2 in the AWS network, with tunnel 1 and tunnel 2 across the public internet. Right: the on-premises network (192.168.8.0/21) with a public subnet (192.168.12.0/24) holding onpremRouter, a pfSense router, and a private subnet (192.168.10.0/24) holding onpremServer and a user](../assets/site-to-site-vpn-demo-architecture.png)

| Side                       | Network           | What runs there                                                                                                         |
| -------------------------- | ----------------- | ----------------------------------------------------------------------------------------------------------------------- |
| AWS VPC                    | `10.16.0.0/16`    | `awsServerA`, a web server with a private IP only                                                                       |
| On-premises public subnet  | `192.168.12.0/24` | `onpremRouter`: pfSense, with a **WAN** interface facing the internet and a **LAN** interface facing the private subnet |
| On-premises private subnet | `192.168.10.0/24` | `onpremServer`: a Windows machine, standing in for a user's desk                                                        |
| Whole on-premises range    | `192.168.8.0/21`  | The prefix AWS is told about                                                                                            |

The on-premises side is **simulated inside AWS**: a second VPC with a pfSense instance from AWS Marketplace. So where a real office would use its own firewall rules and LAN routing, the demo uses security groups and a VPC route table. One CloudFormation stack builds both networks and all three instances. The only manual setup is subscribing to pfSense in Marketplace and creating an EC2 key pair, which the demo later uses to decrypt the Windows password.

The work follows the layers on this page in order: first the AWS side, then the customer gateway device, then routing and security, then a test.

### Stage 1: The AWS side

All in the VPC console:

1. **Create a customer gateway.** Enter the pfSense router's **public IP** (from the stack outputs). Leave the BGP ASN and certificate fields alone: this is static routing with pre-shared keys.
2. **Create a virtual private gateway** with the default Amazon ASN (unused with static routing), then **attach it to the AWS VPC**.
3. **Create the VPN connection:**
   - Target gateway type: **virtual private gateway**, and select the new VGW.
   - Customer gateway: **existing**, and select the one from step 1.
   - Routing: **static**, with static IP prefix **`192.168.8.0/21`**. With BGP, the two sides would exchange this automatically.
   - Everything else, including the tunnel options, left at the defaults.

   The connection starts as **pending** and AWS builds the two endpoints in different AZs. Wait until it is **available** before configuring the other side.
4. **Download the configuration.** Choose vendor and platform **pfSense** and IKE version **IKEv1**. AWS offers files for many vendors and a generic one. The file has everything the device needs for **each** tunnel, including the AWS endpoint's outside IP address and the **pre-shared key**, which differs between the two tunnels.

### Stage 2: The customer gateway device (pfSense)

First, pfSense's second network interface is assigned as the **LAN** interface and enabled. It takes its address by DHCP, which is fine for a demo; a production router would use a static address.

Then, **for each of the two tunnels**, one Phase 1 entry and one Phase 2 entry are created: four pieces of configuration in total. These are the two [IKE phases](../concepts/ipsec-vpn.md#ike-phase-1-authenticate-and-agree-on-a-key). The values come from the downloaded file:

| Phase   | Setting                    | Value used                                   | Meaning                                                |
| ------- | -------------------------- | -------------------------------------------- | ------------------------------------------------------ |
| Phase 1 | Key exchange               | **IKEv1**, main mode                         | AWS supports main mode only with IKEv1                 |
| Phase 1 | Remote gateway             | The AWS endpoint's outside IP                | **Different** for tunnel 1 and tunnel 2                |
| Phase 1 | Authentication             | Mutual PSK, with the tunnel's pre-shared key | Proves each side's identity. **Different** per tunnel. |
| Phase 1 | Encryption, hash, DH group | AES-128, SHA-1, DH group 2 (1024-bit)        | Protects the IKE SA                                    |
| Phase 1 | Lifetime                   | 28,800 seconds (8 hours)                     | When the IKE SA is rekeyed                             |
| Phase 1 | Dead peer detection        | On; delay 10 seconds, max failures 3         | Detects a dead peer, as on the AWS side                |
| Phase 1 | NAT traversal              | Auto                                         | Wraps ESP in UDP 4500 when NAT is in the path          |
| Phase 2 | Mode                       | Tunnel IPv4                                  | IPsec tunnel mode                                      |
| Phase 2 | Local / remote network     | `192.168.10.0/24` / `10.16.0.0/16`           | Which traffic enters the tunnel                        |
| Phase 2 | Protocol, encryption, hash | ESP, AES-128, SHA-1                          | Protects the data                                      |
| Phase 2 | PFS key group              | 2 (1024-bit)                                 | A fresh DH exchange for the IPsec keys                 |
| Phase 2 | Lifetime                   | 3,600 seconds (1 hour)                       | When the IPsec SAs are rekeyed                         |
| Phase 2 | Keep alive                 | Ping `awsServerA`'s private IP               | Keeps traffic flowing so the tunnel stays up           |

After applying the changes, **Status, IPsec** shows both tunnels. Selecting "Connect P1 and P2s" on each starts the negotiation, and both should show **Established**. If one does not, a value is wrong, most often the remote gateway IP or a pre-shared key copied from the other tunnel.

![pfSense Status, IPsec, Overview page with two connections, AWS-Tunnel-AZ1 and AWS-Tunnel-AZ2. Each has local ID 192.168.12.239 on port 4500 with NAT-T, a different AWS remote host on port 4500, role IKEv1 initiator, algorithms AES_CBC (128), HMAC_SHA1_96, PRF_HMAC_SHA1, and MODP_1024, status Established, and one connected child SA entry](../assets/site-to-site-vpn-demo-pfsense-ipsec-status.png)

The status page shows several ideas from earlier on this page and from the [IPsec page](../concepts/ipsec-vpn.md):

- **Two tunnels, two AWS endpoints.** Each connection has a different remote IP, one per AZ.
- **NAT traversal is in use.** pfSense's own address is the private `192.168.12.239`. Its public IP belongs to the instance and is translated by the VPC's internet gateway, so IKE moves to UDP 4500 and ESP is carried inside UDP. An office router behind a NAT device behaves the same way. This is why the customer gateway is created with the **public** IP.
- **pfSense is the initiator.** By default the customer side starts IKE ([tunnel initiation](#tunnel-initiation-and-dead-peer-detection)). AWS-side initiation needs IKEv2, so with IKEv1 the demo relies on the keep-alive ping to stop an idle tunnel from going down.
- **One child SA per tunnel.** pfSense's Phase 2 lists local and remote networks, which looks policy-based. Each tunnel has exactly one Phase 2 entry, so each tunnel builds one IPsec SA pair, which is what AWS requires ([IPsec: what this means on AWS](../concepts/ipsec-vpn.md#what-this-means-on-aws)).

> [!note] The demo uses AWS's minimum cryptography
> AES-128, SHA-1, and DH group 2 are what AWS's sample configuration files use; AWS describes them as "the minimum requirements" ([AWS: example configuration files](https://docs.aws.amazon.com/vpn/latest/s2svpn/example-configuration-files.html)). They work, but are not the best choice for production. AWS also supports IKEv2, AES-256, SHA-2 (256), and DH groups 14 to 24 ([AWS: customer gateway device requirements](https://docs.aws.amazon.com/vpn/latest/s2svpn/CGRequirements.html), checked 2026-11-26). You can choose these on your device and restrict the AWS tunnel options to match.

At this point the tunnels are up, but **no traffic can flow yet**.

### Stage 3: Routing and security

Tunnels that show "Established" only mean the IPsec layer works. Each side still needs a **route** to the other network and a **firewall rule** that allows it, in **both directions**.

**Routing:**

| Route table                            | Route                                           | How                                                                                                                                                         |
| -------------------------------------- | ----------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| AWS VPC route table (both subnets)     | `192.168.8.0/21` to the virtual private gateway | **Route propagation** enabled for the VGW. The route appears as "propagated", copied from the VPN connection's static prefix. Adding it by hand also works. |
| On-premises private subnet route table | `10.16.0.0/16` to the pfSense **private ENI**   | Added by hand. The target must be the **network interface**, not the instance, because pfSense has more than one interface.                                 |

**Security groups**, standing in for on-premises firewall rules:

| Security group                             | Inbound rule added                          | Why                                                                                                                                                                                                             |
| ------------------------------------------ | ------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| AWS default SG (on `awsServerA`)           | All traffic from `192.168.8.0/21`           | Lets the on-premises network reach the AWS server                                                                                                                                                               |
| On-premises default SG (on `onpremServer`) | All traffic from `10.16.0.0/16`             | Lets the AWS side reach on-premises machines                                                                                                                                                                    |
| pfSense router SG                          | All traffic from the on-premises default SG | Traffic from `onpremServer` to AWS enters pfSense's LAN interface on its way into the tunnel. The group only allowed HTTPS for the admin page. A production setup would allow the on-premises IP range instead. |

In a real deployment, the on-premises rules would live on the on-premises firewall, and NACLs on the AWS side may also need rules.

> [!note] Inference: source/destination check
> An EC2 instance that forwards traffic for other machines, like this pfSense router, normally needs **source/destination check** turned off on its interfaces ([EC2 network interfaces](ec2.md#network-interfaces-enis)). The lesson never shows this, so the CloudFormation stack presumably handles it.

### Stage 4: Testing

The test plays a user sitting at the on-premises Windows machine:

1. Connect to `onpremServer`, which has no public IP, with **Fleet Manager Remote Desktop**, part of [SSM](ssm.md). The Administrator password is decrypted with the key pair created at the start.
2. From a command prompt, **ping** `awsServerA`'s **private** IP. A reply means packets cross the VPN in both directions.
3. In a browser, open `http://<awsServerA private IP>`. The web page loads and shows the instance ID, which matches `awsServerA` in the EC2 console.

Both tests use **private** IP addresses from one network to the other. That is the point of a Site-to-Site VPN.

### Clean-up

Delete the VPN connection first, then detach and delete the virtual private gateway, then delete the customer gateway, and finally delete the CloudFormation stack. The VPN connection is billed per hour until it is deleted ([Pricing](#pricing)).

### How the demo compares with a production design

| Demo choice                           | Production consideration                                                                                                                                  |
| ------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| One pfSense router                    | The AWS side is highly available, but this router is a **single point of failure**. See [High Availability](#high-availability).                          |
| Static routing                        | Fine for one simple network. BGP adds automatic route exchange and failover across connections ([Static vs Dynamic Routing](#static-vs-dynamic-routing)). |
| IKEv1, AES-128, SHA-1, DH group 2     | Choose stronger options that AWS supports, such as IKEv2, AES-256, SHA-2, and DH group 14 or higher                                                       |
| Security groups for on-premises rules | Firewall rules on the real on-premises firewall                                                                                                           |
| Keep-alive ping                       | Use IKEv2 and let AWS start the negotiation, or keep regular traffic flowing                                                                              |

The steps stay the same with other vendors' devices: create the AWS side, configure two tunnels on the device from the downloaded file, then add routes and firewall rules on both sides.

## References

- Adrian Cantrill, AWS Certified Solutions Architect – Associate (SAA-C03): "AWS Site-to-Site VPN" (Part 1 and Part 2)
- Adrian Cantrill, AWS Certified Solutions Architect – Associate (SAA-C03): "[Advanced Demo] Simple Site2Site VPN" (Stages 0–5)
- [AWS: What is Site-to-Site VPN?](https://docs.aws.amazon.com/vpn/latest/s2svpn/VPC_VPN.html)
- [AWS: how Site-to-Site VPN works](https://docs.aws.amazon.com/vpn/latest/s2svpn/how_it_works.html)
- [AWS: tunnel options](https://docs.aws.amazon.com/vpn/latest/s2svpn/VPNTunnels.html)
- [AWS: tunnel initiation options](https://docs.aws.amazon.com/vpn/latest/s2svpn/initiate-vpn-tunnels.html)
- [AWS: tunnel endpoint replacements](https://docs.aws.amazon.com/vpn/latest/s2svpn/endpoint-replacements.html)
- [AWS: routing options](https://docs.aws.amazon.com/vpn/latest/s2svpn/VPNRoutingTypes.html)
- [AWS: route tables and route priority](https://docs.aws.amazon.com/vpn/latest/s2svpn/vpn-route-priority.html)
- [AWS: your customer gateway device](https://docs.aws.amazon.com/vpn/latest/s2svpn/your-cgw.html)
- [AWS: customer gateway options](https://docs.aws.amazon.com/vpn/latest/s2svpn/cgw-options.html)
- [AWS: customer gateway device requirements](https://docs.aws.amazon.com/vpn/latest/s2svpn/CGRequirements.html)
- [AWS: example configuration files](https://docs.aws.amazon.com/vpn/latest/s2svpn/example-configuration-files.html)
- [AWS: accelerated Site-to-Site VPN](https://docs.aws.amazon.com/vpn/latest/s2svpn/accelerated-vpn.html)
- [AWS: Private IP VPN with Direct Connect](https://docs.aws.amazon.com/vpn/latest/s2svpn/private-ip-dx.html)
- [AWS: VPN Concentrators](https://docs.aws.amazon.com/vpn/latest/s2svpn/vpn-concentrator.html)
- [AWS: CloudWatch metrics for Site-to-Site VPN](https://docs.aws.amazon.com/vpn/latest/s2svpn/monitoring-cloudwatch-vpn.html)
- [AWS: Site-to-Site VPN quotas](https://docs.aws.amazon.com/vpn/latest/s2svpn/vpn-limits.html)
- [AWS: VPN pricing](https://aws.amazon.com/vpn/pricing/)
- [AWS: Large Bandwidth Tunnels announcement](https://aws.amazon.com/about-aws/whats-new/2025/11/aws-site-to-site-vpn-5-gbps-bandwidth-tunnels/)
- [AWS: VPN Concentrator announcement](https://aws.amazon.com/about-aws/whats-new/2025/11/aws-site-to-site-vpn-concentrator/)
