---
title: IPsec VPN Fundamentals
type: concept
tags: [ipsec, vpn, ike, ikev1, ikev2, diffie-hellman, security-association, policy-based-vpn, route-based-vpn, encryption, networking, hybrid-networking]
certs: [aws-saa]
updated: 2026-11-26
---

# IPsec VPN Fundamentals

**IPsec** is a group of protocols that work together to build **secure tunnels across an insecure network**, usually the public internet. The tunnel runs between two **peers**: one router or firewall at each end, called the local and remote peer. A business might use IPsec to join its offices, or to join an on-premises network to a private network in AWS. That second case is [AWS Site-to-Site VPN](../services/site-to-site-vpn.md).

IPsec gives you two things:

1. **Authentication.** Only peers that are known to each other can connect. They prove their identity with a **pre-shared key** (a shared password) or a **certificate**.
2. **Protected data.** Traffic inside the tunnel is encrypted, so anyone watching the internet sees only ciphertext. It is also integrity-protected, so it cannot be changed in transit without the change being detected.

This page explains how the two peers get to the point of sharing a secret key without ever sending it, and the two ways a VPN decides which traffic to send through the tunnel. For where these tunnels fit in AWS networking, see [Networking Concepts](networking.md#hybrid-connectivity-vpn-and-bgp).

## Tunnels and Interesting Traffic

![Business site 1 connected to business sites 2 and 3 through IPsec tunnels across the insecure public internet. A label says tunnels are created as required for interesting traffic, and data inside the tunnels is encrypted, giving a secure connection over an insecure network](../assets/ipsec-tunnels-overview.png)

**Interesting traffic** is traffic that matches the VPN's rules. The rules might match network prefixes or more specific traffic types. When a peer sees interesting traffic, it builds a tunnel to carry it. When there is no interesting traffic for a while, the tunnel can be torn down, and it is built again the next time interesting traffic appears.

The tunnel uses the public internet for transport, but the data inside it is encrypted the whole way across.

> [!note] Idle tunnels on AWS
> On AWS, a tunnel with no traffic can go down. By default, your customer gateway device starts the IKE negotiation, so AWS recommends sending regular traffic or having AWS start the negotiation instead (IKEv2 only). See [Site-to-Site VPN tunnel initiation](../services/site-to-site-vpn.md#tunnel-initiation-and-dead-peer-detection).

## Why IPsec Uses Two Kinds of Cryptography

IPsec uses the same idea as [TLS](tls-https.md#3-tls-handshake): slow public-key mathematics to agree on a key once, then fast symmetric encryption for all the data.

|                | Symmetric encryption                              | Public-key (asymmetric) cryptography                                           |
| -------------- | ------------------------------------------------- | ------------------------------------------------------------------------------ |
| Keys           | One key both encrypts and decrypts                | A public key that can be shared, and a private key that never leaves its owner |
| Speed          | Fast, low CPU overhead                            | Slow                                                                           |
| Problem it has | How do you get the same key to both sides safely? | Too slow for bulk data                                                         |
| Role in IPsec  | Encrypts the negotiation, then the data           | Lets the two sides agree on symmetric keys over the internet                   |

> [!note] Key agreement, not key encryption
> The lecture summarizes Phase 1 as "asymmetric encryption is used to agree on, create and share symmetric keys". The method it then shows, **Diffie-Hellman (DH)**, is more precise: nothing is encrypted and sent. Each side combines its own private value with the other side's public value, and both arrive at the same secret. The secret itself never crosses the network. This is different from TLS 1.2's RSA key exchange, where the client encrypts a secret with the server's public key and sends it.

## IKE Phase 1: Authenticate and Agree on a Key

**IKE (Internet Key Exchange)** is the protocol IPsec peers use to authenticate each other and agree on keys. There are two versions. **IKEv1** is older. **IKEv2** is newer, uses fewer messages, and has more features. AWS Site-to-Site VPN supports both ([AWS: customer gateway device requirements](https://docs.aws.amazon.com/vpn/latest/s2svpn/CGRequirements.html)).

Phase 1 is the **slow, heavy** part. Its result is the **IKE security association (IKE SA)**, also called the Phase 1 tunnel.

![IKE Phase 1 between Bob's router at business site 1 and Julie's router at business site 2, in seven steps: 1, certificate or pre-shared key authentication; 2, each side creates a Diffie-Hellman private key; 3, each side derives a public key; 4, public keys are exchanged over the public internet; 5, each side combines its own private key with the remote peer's public key and independently generates the same Diffie-Hellman key; 6, using the DH key, the peers exchange key material and agreements; 7, each side generates a symmetric key from the DH key and the exchanged material. The result is the IKE security association, the Phase 1 tunnel](../assets/ipsec-ike-phase-1.png)

1. **Authentication.** Each peer must prove it should be part of this VPN, using a certificate or a pre-shared key. This is about **identity**, not about encrypting your data. The slide numbers it first because it is the foundation. The lecture notes that the actual proof happens inside the protected exchange that Phase 1 builds.
2. **Each side creates a DH private key.** It never leaves that side.
3. **Each side derives a DH public key** from its private key.
4. **The public keys are exchanged** over the public internet. They are not sensitive. An attacker who captures them cannot work back to either private key or to the shared secret.
5. **Each side computes the same DH key.** It combines its own private key with the other side's public key. Both sides get the same value independently, and it was never transmitted.
6. **The DH key protects an exchange of key material and agreements.** You can think of this as a negotiation.
7. **Each side generates the same Phase 1 symmetric key** from the DH key and the exchanged material. This key encrypts everything that passes through the IKE SA.

At the end of Phase 1, both peers share a symmetric key that was never sent between them.

## IKE Phase 2: Build the IPsec Security Associations

Phase 2 is **fast and light**, because Phase 1 did the hard work. It is protected by the IKE SA and is about how the data will be encrypted: which methods, which keys, and for which traffic.

![IKE Phase 2 on top of the IKE security association. Step 1, the Phase 1 symmetric key encrypts agreements and more key material; step 2, the best shared encryption and integrity methods are communicated and agreed; step 3, the DH key and exchanged key material create a symmetric IPsec key on each side; step 4, the IPsec key is used for bulk encryption and decryption of interesting traffic. At the bottom, two IPsec SAs, one in each direction, form the Phase 2 tunnel](../assets/ipsec-ike-phase-2.png)

1. **The Phase 1 key encrypts the negotiation.** The peers exchange agreements and more key material through the IKE SA.
2. **The peers agree on a cipher suite.** One peer lists the encryption and integrity methods it supports. The other picks the best option they share and replies.
3. **Each side creates the IPsec key.** The keying material from Phase 1 and the new exchange produce a symmetric **IPsec key**, designed for efficient bulk encryption with the agreed method.
4. **The IPsec key encrypts and decrypts interesting traffic.**

The result is a pair of **IPsec SAs**, one for each direction. An SA is one-way, so two-way traffic always needs a pair. This pair is what the lecture calls the Phase 2 tunnel.

> [!note] Two refinements to the lecture's picture
>
> - **Where the Phase 2 key comes from.** In IKE, Phase 2 keys are derived from the IKE SA's keying material plus fresh random values. A **new** Diffie-Hellman exchange in Phase 2 is optional; it is called **perfect forward secrecy (PFS)**. With PFS, breaking one IPsec key does not expose keys made later. AWS lists the DH groups it accepts for each phase separately ([AWS: customer gateway device requirements](https://docs.aws.amazon.com/vpn/latest/s2svpn/CGRequirements.html); [RFC 7296](https://www.rfc-editor.org/rfc/rfc7296)).
> - **"Tunnel inside a tunnel" is a picture, not the packet path.** The IKE SA protects the **negotiation** only. Your data does not travel inside it. Data packets are sent as **ESP** (Encapsulating Security Payload, IP protocol 50) using the IPsec SAs, or inside UDP port 4500 when a NAT device sits in the path (**NAT traversal**). IKE itself uses UDP 500 ([RFC 4301](https://www.rfc-editor.org/rfc/rfc4301); [RFC 3948](https://www.rfc-editor.org/rfc/rfc3948)). This is why a firewall in front of a VPN device must allow UDP 500, UDP 4500, and often ESP.

### Why the two phases are separate

The split lets the expensive part last. The IPsec SAs can be torn down when interesting traffic stops, while the IKE SA stays up. When traffic starts again, the peers only need a new Phase 2, which is quick.

Both kinds of SA also have **lifetimes**. Before one expires, the peers **rekey**: they negotiate a replacement, so long-lived tunnels keep changing keys without dropping traffic.

## Policy-Based vs Route-Based VPNs

The two VPN types differ in **how they match interesting traffic**, and therefore in how many IPsec SA pairs they build.

![One Phase 1 tunnel between business sites 1 and 2 carrying Phase 2 tunnels. A route-based VPN has a single SA pair with a single IPsec key. A policy-based VPN has one SA pair per policy, each with a unique IPsec key: one for infrastructure servers, one for CCTV cameras, and one for financial systems](../assets/ipsec-policy-vs-route-vpn.png)

|                        | Policy-based VPN                                                                                           | Route-based VPN                                                     |
| ---------------------- | ---------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------- |
| How traffic is matched | Rule sets (policies) such as source, destination, and traffic type                                         | Routes: traffic routed into the tunnel interface is sent through it |
| SA pairs               | One SA pair per matching policy, each with its own IPsec key                                               | One SA pair for the tunnel, one IPsec key                           |
| Flexibility            | Different security settings for different traffic, for example infrastructure, CCTV, and financial systems | The same settings for all traffic in the tunnel                     |
| Setup                  | Harder to configure                                                                                        | Simpler                                                             |

> [!note] One SA pair per tunnel, not per prefix
> The lecture describes a route-based VPN as having "a single pair of SAs for each network prefix". More precisely, a route-based VPN has one SA pair for the **tunnel**, and the routing table decides which prefixes go into it. Adding another network does not add another SA pair; it adds a route.

### What this means on AWS

AWS Site-to-Site VPN is **route-based**. AWS says you are "limited to one unique security association (SA) pair per tunnel (one inbound and one outbound)". If your device is policy-based, you must limit it to a single SA. Some policy-based devices create one SA for every access-list entry, and those extra SAs cause problems ([AWS: customer gateway device requirements](https://docs.aws.amazon.com/vpn/latest/s2svpn/CGRequirements.html)).

> [!warning] Gotcha
> A policy-based firewall with several networks on each side can create several SA pairs and break or flap the AWS tunnel. Use a route-based (virtual tunnel interface) configuration, or one policy that covers all the networks, for example any-to-any.

## How AWS Uses IPsec

- **[AWS Site-to-Site VPN](../services/site-to-site-vpn.md)** builds two IPsec tunnels between a [VPC](../services/vpc.md) (through a virtual private gateway) or a transit gateway and your customer gateway device. AWS supports IKEv1 (main mode only) and IKEv2, AES-128 or AES-256, SHA-1 or SHA-2 (256), and DH groups 2 and 14 to 24 for Phase 1, plus group 5 for Phase 2 ([AWS: customer gateway device requirements](https://docs.aws.amazon.com/vpn/latest/s2svpn/CGRequirements.html), checked 2026-11-26).
- Routes across the tunnels can be static or learned with [BGP](bgp.md).
- **Private IP VPN** runs IPsec over AWS Direct Connect, so traffic on a private link is also encrypted ([AWS: Private IP VPN](https://docs.aws.amazon.com/vpn/latest/s2svpn/private-ip-dx.html)).

> [!tip] Exam Tip
>
> - **IPsec = encrypted tunnel over the public internet between two peers.** Authentication is by pre-shared key or certificate.
> - **IKE Phase 1** is slow and heavy: authentication plus Diffie-Hellman, producing the IKE SA. **IKE Phase 2** is fast: it agrees on ciphers and keys, producing a pair of IPsec SAs that carry the data.
> - **Policy-based** = one SA pair per rule, different settings per traffic type. **Route-based** = one SA pair, traffic chosen by routes. AWS Site-to-Site VPN is route-based.

## References

- Adrian Cantrill, AWS Certified Solutions Architect – Associate (SAA-C03): "IPsec VPN Fundamentals"
- [AWS: customer gateway device requirements](https://docs.aws.amazon.com/vpn/latest/s2svpn/CGRequirements.html) (checked 2026-11-26)
- [AWS: Site-to-Site VPN tunnel initiation options](https://docs.aws.amazon.com/vpn/latest/s2svpn/initiate-vpn-tunnels.html)
- [AWS: Private IP VPN with Direct Connect](https://docs.aws.amazon.com/vpn/latest/s2svpn/private-ip-dx.html)
- [RFC 4301: Security Architecture for the Internet Protocol](https://www.rfc-editor.org/rfc/rfc4301)
- [RFC 7296: Internet Key Exchange Protocol Version 2 (IKEv2)](https://www.rfc-editor.org/rfc/rfc7296)
- [RFC 3948: UDP Encapsulation of IPsec ESP Packets](https://www.rfc-editor.org/rfc/rfc3948)
