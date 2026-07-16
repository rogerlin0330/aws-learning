---
title: TLS and HTTPS
type: concept
tags: [tls, https, security, certificates, acm, networking]
certs: [aws-saa]
updated: 2026-06-04
---

# TLS and HTTPS

HTTPS = HTTP over TLS. TLS (Transport Layer Security) provides three guarantees: **confidentiality** (encrypted data), **integrity** (tamper detection), and **authentication** (server identity verified). It operates at Layer 6/7 and sits above the [Networking](networking.md) controls (SG, NACL) that operate at L3/4.

AWS services that terminate TLS: ALB, CloudFront, API Gateway. Certificate provisioning is managed by **ACM (AWS Certificate Manager)** — see the AWS connection points section below.

## 1. Certificates and the Chain of Trust

A TLS certificate binds a public key to a domain name, signed by a **Certificate Authority (CA)**. The signature is what lets clients trust a certificate they've never seen before.

### Certificate Chain

```text
Root CA Certificate          ← pre-installed in OS/browser trust store
    │
    └─ signs →
Intermediate CA Certificate  ← sent by server during handshake
    │
    └─ signs →
Server Certificate           ← sent by server; contains its public key
```

The **Root CA cert is never sent over the wire** — clients already have it locally. The server sends only its own cert plus any intermediates needed to build the chain up to a trusted root.

### CA Certificate Issuance

**Flow A — Owner-managed (self-service):**

1. Owner generates an RSA key pair locally
2. Creates a **CSR (Certificate Signing Request)** — contains the domain name, public key, and organization info
3. Submits CSR to a CA (e.g. Let's Encrypt, DigiCert)
4. CA validates domain ownership (DNS TXT record, email, or HTTP challenge)
5. CA signs the certificate: hash the TBS (To-Be-Signed) content → encrypt hash with CA's private key → attach as signature
6. Owner receives `certificate.pem`; keeps `private.pem` secret

**Flow B — Provider-managed (e.g. AWS ACM):**

1. Request a certificate via the provider's dashboard (e.g. ACM console)
2. Provider generates the key pair and CSR on your behalf
3. Provider submits CSR to the CA
4. Domain validation challenge sent back (Route 53 DNS TXT record is the easiest option with ACM)
5. CA issues certificate; provider stores it and deploys it automatically to ALB/CloudFront/etc.

> [!warning] Gotcha
> In Flow B the **provider holds the private key**. With ACM specifically, the private key never leaves AWS — ACM manages it and deploys it to integrated services; you can't download it.

> [!tip] Exam Tip
> ACM certificates are **free** and **auto-renewed** when used with integrated AWS services (ALB, CloudFront, API Gateway). You cannot export the private key from ACM-issued certificates.

## 2. Certificate Verification

When a client receives a certificate, it verifies the **chain of trust** from the server cert up to a trusted root:

```text
For each cert in the chain (server → intermediate → root):
  1. Re-compute: Hash = SHA256(TBS portion of the cert)
  2. Decrypt the cert's signature using the issuer's public key
  3. If Hash == decrypted signature → cert is authentic and untampered
  4. Additional checks:
     - Is the cert within its validity period?
     - Does the domain (CN / SAN) match the URL?
     - Is the cert revoked? (CRL or OCSP check)
```

The root CA's public key comes from the OS/browser trust store — not from the server.

## 3. TLS Handshake

The handshake establishes a shared **symmetric session key** before any application data is exchanged. Asymmetric crypto (slow) is used only for the key exchange; symmetric crypto (fast) handles all subsequent data.

### TLS 1.2 Handshake (RSA key exchange)

```text
Client                          Server
  │── ClientHello ─────────────►│  (TLS version, ciphers, Client_Random)
  │◄─ ServerHello ──────────────│  (selected cipher, Server_Random)
  │◄─ Certificate ──────────────│  (cert chain with Server_PublicKey)
  │◄─ ServerHelloDone ──────────│
  │
  │   [Client verifies cert — see §2]
  │   [Client generates 48-byte Pre-Master Secret]
  │
  │── ClientKeyExchange ────────►│  Pre-Master Secret encrypted with Server_PublicKey
  │── ChangeCipherSpec ─────────►│
  │── Finished (AES encrypted) ─►│
  │◄─ ChangeCipherSpec ──────────│
  │◄─ Finished (AES encrypted) ──│
  │
  │═══════ Application Data (AES) ══════════════│
```

Both sides independently derive the same symmetric session keys from:

```text
MasterSecret = PRF(PreMasterSecret, "master secret", Client_Random + Server_Random)

Session keys derived from MasterSecret:
  - Client Write Key (AES key for client→server data)
  - Server Write Key (AES key for server→client data)
  - Client Write IV
  - Server Write IV
```

### TLS 1.3 Improvements

TLS 1.3 (current standard) removed RSA key encipherment. Key differences:

| | TLS 1.2 | TLS 1.3 |
| --- | --- | --- |
| Key exchange | RSA or ephemeral DH | Ephemeral ECDH only |
| Round trips | 2-RTT | 1-RTT (0-RTT for resumption) |
| Forward secrecy | Optional | Mandatory |
| Cipher suites | Many (some weak) | 5 strong suites only |

With TLS 1.3, the server sends its `key_share` in ServerHello, and the client responds with its `key_share` — no pre-master secret is sent over the wire at all; instead both sides compute a shared secret from ephemeral DH.

> [!tip] Exam Tip
> **Forward secrecy** means a compromise of the server's long-term private key does **not** decrypt past sessions — because ephemeral keys were used for each session and then discarded.

## 4. AWS Connection Points

| AWS Service | Role |
| --- | --- |
| **ACM** (AWS Certificate Manager) | Issues and manages TLS certificates; integrates with ALB, CloudFront, API GW |
| **ACM Private CA** | Issues private certificates for internal services (not publicly trusted) |
| **ALB** | Terminates TLS; forwards plaintext to backends (or re-encrypts) |
| **CloudFront** | Terminates TLS at edge; can use custom certificates from ACM or IAM |
| **API Gateway** | Terminates TLS; custom domain names require ACM cert |
| **Route 53** | DNS TXT record for ACM domain validation — see [DNS](dns.md) |

> [!tip] Exam Tip
> ACM certificates used with **CloudFront must be in `us-east-1`** (N. Virginia), regardless of where your CloudFront distribution serves traffic. ALB certificates must be in the **same region as the ALB**.

> [!warning] Gotcha
> **TLS termination at ALB** means traffic between ALB and EC2 targets is plaintext by default. To encrypt the backend leg, configure HTTPS on the target group and install a cert on the EC2 instance (can be self-signed for this leg). These arrangements have names — bridging (re-encrypt), offload (plain HTTP), pass-through (never decrypted, NLB TCP listener only) — covered in [ELB SSL/TLS connection modes](../services/elb.md#ssltls-connection-modes--bridging-pass-through-offload).

## References

- [Networking Concepts](networking.md)
- [DNS](dns.md)
- Source: [How SSL and HTTPS Works](../../raw/web-clippings/How%20SSL%20and%20HTTPS%20Works.md)
