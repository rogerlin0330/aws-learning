---
title: AWS CloudHSM
type: service
tags: [cloudhsm, hsm, kms, custom-key-store, fips-140-3, non-fips, pkcs11, jce, cng, ksp, openssl, ssl-offload, tde, oracle, certificate-authority, encryption, key-management, security]
certs: [aws-saa]
updated: 2026-11-23
---

# AWS CloudHSM

AWS CloudHSM gives you **single-tenant hardware security modules (HSMs)** in the AWS cloud. An HSM is a dedicated, tamper-resistant device that generates and stores cryptographic keys and performs cryptographic operations with them. AWS provisions and maintains the hardware, but only users you create inside the HSM can use or manage the keys.

[KMS](kms.md) also runs on HSMs. The difference is who controls them. KMS is a shared, AWS-managed service: you call AWS APIs, [IAM](iam.md) and key policies control access, and you trust AWS's controls to keep your key material isolated. CloudHSM gives you an HSM that you control directly, through industry-standard crypto APIs rather than AWS APIs. That control comes with less AWS integration and more operational work.

For the SAA exam, CloudHSM is a less common topic. It usually appears when a question names a requirement that KMS alone cannot meet: a single-tenant HSM you control, PKCS #11 or JCE access, non-FIPS algorithms, or keys that must stay out of AWS's control.

## How CloudHSM Differs From KMS

|                         | [KMS](kms.md)                                                                                                                  | CloudHSM                                                                                                                       |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------ |
| Tenancy                 | Shared, multi-tenant service; your keys are isolated logically                                                                 | Single-tenant HSM; the hardware [may be shared, but each HSM is accessible only to you](https://aws.amazon.com/cloudhsm/faqs/) |
| Who manages keys        | AWS operates the service; you manage keys through AWS APIs                                                                     | AWS provisions the HSM; you manage users and keys inside it                                                                    |
| Access control          | [IAM](iam.md) policies, key policies, and grants                                                                               | HSM users (admins and crypto users) created inside the HSM. IAM only controls the cluster management API                       |
| Interfaces              | AWS API (`Encrypt`, `Decrypt`, `GenerateDataKey`, …)                                                                           | Industry-standard libraries: PKCS #11, JCE, Microsoft CNG/KSP, OpenSSL                                                         |
| AWS service integration | Native: S3, EBS, RDS, Secrets Manager, and many others                                                                         | None directly; reach AWS services through a KMS CloudHSM key store                                                             |
| FIPS validation         | [FIPS 140-3 Security Level 3](https://docs.aws.amazon.com/kms/latest/developerguide/overview.html) (outside the China Regions) | [hsm2m.medium: FIPS 140-3 Level 3](https://docs.aws.amazon.com/cloudhsm/latest/userguide/hsm-types.html)                       |
| Non-FIPS algorithms     | No                                                                                                                             | Yes, in a non-FIPS mode cluster                                                                                                |
| Pricing                 | Per key per month plus per request                                                                                             | Per HSM per hour                                                                                                               |

FIPS 140 (Federal Information Processing Standard Publication 140) is the US government standard for validating cryptographic modules. It defines four security levels, and Level 3 adds physical tamper resistance and identity-based authentication. The lecture links the [Wikipedia article on FIPS 140-2](https://en.wikipedia.org/wiki/FIPS_140-2) for background; FIPS 140-3 is its successor and is the version current AWS validations use.

The lecture makes the point that FIPS level alone no longer separates the two services. In the past KMS was validated at a lower level, but as of the lecture and the current KMS documentation (checked 2026-11-23), both are FIPS 140-3 Level 3. The reasons to choose CloudHSM are therefore:

1. You need a **single-tenant HSM that you control**, with AWS kept outside the key-access boundary.
2. You need **industry-standard APIs** such as PKCS #11, JCE, CNG/KSP, or OpenSSL.
3. You need **algorithms that are not FIPS-approved**, which a non-FIPS cluster can provide and KMS cannot.

> [!tip] Exam Tip
> If a question mentions PKCS #11, JCE, CNG, KSP, or the OpenSSL Dynamic Engine, the answer involves CloudHSM. If it mentions S3, EBS, or RDS encryption with standard AWS integration, the answer is KMS, or KMS with a CloudHSM key store when the question also needs your own HSMs.

## What AWS Manages and What You Manage

According to the [CloudHSM FAQ](https://aws.amazon.com/cloudhsm/faqs/), AWS provisions the HSMs, monitors their health, replaces failed HSMs, manages firmware, and takes backups. AWS "has no access to any keys or data inside your CloudHSM Cluster". The HSMs are partitioned and physically tamper-resistant, so the part of the device that holds key material is outside AWS's reach.

You manage everything inside the HSM: users, keys, and cryptographic operations. With Client SDK 5 and the CloudHSM CLI, the [HSM user types](https://docs.aws.amazon.com/cloudhsm/latest/userguide/understanding-users.html) are:

- **Admin**: manages users, for example creating and deleting users and changing passwords. The lecture and older Client SDK 3 tools call this role a crypto officer (CO).
- **Crypto user (CU)**: creates, shares, imports, and exports keys and uses them to encrypt, decrypt, and sign.
- **Appliance user (AU)**: used by AWS to clone and synchronize HSMs in the cluster. It cannot view or change your users or keys.
- **Unactivated admin**: a temporary user on the first HSM of a cluster that has not yet been activated.

The CloudHSM API, which IAM controls, can create and delete clusters, HSMs, and backups. The FAQ notes that it "cannot access the contents of the HSM or modify any users, policies, or other settings".

> [!warning] Gotcha
> AWS cannot recover your HSM user credentials. Backups can restore a lost HSM or cluster, but if you lose the credentials, nobody at AWS can give you access to the keys again. The extra security depends on you keeping your own access safe.

## Architecture

![CloudHSM architecture: on the left, a customer VPC (labelled AEM-VPC) spans AZ-A and AZ-B, each with an EC2 instance running the AWS CloudHSM Client SDK and an ENI; the SDK exposes PKCS#11, Java Cryptography Extensions (JCE), Microsoft CNG/KSP, and OpenSSL Dynamic Engine libraries, and instances can reach the ENI in their own AZ or cross AZs. On the right, an AWS CloudHSM VPC holds one HSM per AZ inside a dashed cluster boundary, with red arrows from each HSM to its ENI in the customer VPC and a vertical arrow between the two HSMs. Labels say HSMs operate in an AWS managed HSM VPC with interfaces added to the customer VPC, HSMs keep keys and policies in sync when nodes are added or removed, and AWS provisions the HSM but has no access to the secure area where key material is held.](../assets/cloudhsm-architecture.png)

The diagram shows how CloudHSM connects to your network:

1. **The HSMs are not in your VPC.** They run in a separate VPC in an AWS account that the CloudHSM service owns.
2. **Each HSM gets an elastic network interface (ENI) in your VPC.** When you create an HSM, CloudHSM [places a network interface in the subnet you specified](https://docs.aws.amazon.com/cloudhsm/latest/userguide/cluster-architecture.html), in the same Availability Zone as the HSM. Your clients reach the HSM through that ENI, either in their own AZ or across AZs. This is the same pattern the [VPC](vpc.md) page describes for other services that place interfaces into customer subnets.
3. **HSMs are grouped into a cluster.** HSMs in a cluster are [automatically synchronized and load-balanced](https://aws.amazon.com/cloudhsm/faqs/). Keys, users, and policies stay in sync as HSMs are added or removed.
4. **Clients use the CloudHSM Client SDK.** Your application loads a CloudHSM library, and the client spreads requests across the HSMs and handles HSM failures.
5. **A cluster security group controls access.** CloudHSM creates a security group named `cloudhsm-cluster-<clusterID>-sg`. It allows traffic on [TCP ports 2223–2225](https://docs.aws.amazon.com/cloudhsm/latest/userguide/configure-sg.html) within that group, so a client instance must be in the group to reach the HSMs. See [Security Groups](vpc.md#security-groups-sg).

### High availability

A single HSM is a single point of failure, and it lives in one AZ. AWS [strongly recommends at least two HSMs in two different AZs](https://aws.amazon.com/cloudhsm/faqs/) for production, and at least three HSMs across at least two AZs for mission-critical workloads. Create one subnet per AZ for the cluster.

A cluster can hold up to [28 HSMs](https://docs.aws.amazon.com/cloudhsm/latest/userguide/clusters.html), but the default quota is 6 HSMs per account per Region (checked 2026-11-23).

### Backups

AWS takes [automatic, encrypted backups](https://aws.amazon.com/cloudhsm/faqs/) of the cluster daily, and also when HSMs are added or removed. A backup contains users, keys, and policies. The HSM encrypts the data before it leaves the device, and [AWS cannot decrypt backups](https://docs.aws.amazon.com/cloudhsm/latest/userguide/backups.html). Backups are stored in a service-controlled [S3](s3.md) bucket in the same Region, and they let you restore a lost HSM or create a new cluster.

> [!warning] Gotcha
> A key created after the last backup exists only in the HSMs. If a cluster has one HSM and that HSM fails, recently created keys can be lost. The FAQ says you are responsible for key durability between backups, which is another reason to keep two HSMs in two AZs.

## HSM Types and Cluster Modes

| HSM type       | Status (checked 2026-11-23)                                                                                                                                                    | FIPS validation    | Cluster mode     |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------ | ---------------- |
| `hsm2m.medium` | Current                                                                                                                                                                        | FIPS 140-3 Level 3 | FIPS or non-FIPS |
| `hsm1.medium`  | [End of support on March 31, 2026](https://docs.aws.amazon.com/cloudhsm/latest/userguide/compliance-dep-notif.html); automatic migration to hsm2m.medium began in January 2026 | FIPS 140-2 Level 3 | FIPS only        |

The cluster mode is chosen at creation:

- **FIPS mode** allows only FIPS-approved keys and algorithms.
- **Non-FIPS mode** allows all keys and algorithms CloudHSM supports, including ones that are not FIPS-approved. KMS has no equivalent.

A [cluster's mode cannot be changed](https://docs.aws.amazon.com/cloudhsm/latest/userguide/cluster-hsm-types.html) after creation, and a backup can only be restored to a cluster in the same mode. Moving to non-FIPS mode can also affect your compliance posture if a regulation requires FIPS-validated modules.

## Client SDK and Industry-Standard APIs

CloudHSM is accessed through libraries that implement industry-standard interfaces, not through AWS APIs. [Client SDK 5](https://docs.aws.amazon.com/cloudhsm/latest/userguide/use-hsm.html) includes:

- **PKCS #11 library**: the common C interface for HSMs and smart cards.
- **JCE provider**: Java Cryptography Extension for Java applications.
- **Key Storage Provider (KSP) and CNG provider**: Microsoft's Windows cryptography interfaces.
- **OpenSSL Dynamic Engine** and the newer **OpenSSL Provider**: let OpenSSL-based software such as NGINX, Apache, and HAProxy use HSM-resident keys.

Client SDK 3 is deprecated, and hsm2m.medium HSMs need Client SDK 5.9.0 or later.

Because these are standard interfaces, software written for an on-premises HSM can often use CloudHSM with configuration changes rather than a rewrite. That is also why CloudHSM lacks native AWS integration: AWS services talk to KMS, not to PKCS #11.

## KMS With a CloudHSM Key Store

KMS can use your CloudHSM cluster as a [custom key store](https://docs.aws.amazon.com/kms/latest/developerguide/keystore-cloudhsm.html). KMS keys created in that key store have their key material generated and held in your HSMs. AWS services still talk to KMS as usual, so S3, EBS, RDS, and other services can use keys whose material is in a cluster you control.

Requirements and limits (checked 2026-11-23):

- The cluster must have **at least two active HSMs in different AZs** when you create the key store.
- You create a crypto user named **`kmsuser`** in the cluster. KMS logs in as `kmsuser` to use the keys, and you should not use that account for anything else.
- Only **symmetric AES encryption keys** are supported. Asymmetric keys, HMAC keys, imported key material, automatic key rotation, and multi-Region keys are not.

This is the route when a question needs both AWS service integration and keys held in HSMs you control.

## Use Cases

The lecture's main point is that CloudHSM suits work that is not AWS-specific, while KMS suits work that needs AWS integration. Common scenarios from the lecture and the [CloudHSM use cases](https://docs.aws.amazon.com/cloudhsm/latest/userguide/use-cases.html):

### No direct AWS integration

You cannot point [S3 server-side encryption](s3.md#object-encryption--sse-and-cse) directly at CloudHSM. Two options remain:

- **Client-side encryption**: your application encrypts the data with CloudHSM keys before uploading it.
- **Server-side encryption through KMS**: use SSE-KMS with a KMS key in a CloudHSM key store.

The same applies to EBS, RDS, and other AWS services.

### SSL/TLS offload for web servers

A web server can keep its TLS private key in CloudHSM instead of on disk. NGINX, Apache, or HAProxy uses the OpenSSL libraries, and IIS on Windows uses KSP. A web server using an HSM is still the TLS endpoint. Only the private-key operation in the [TLS handshake](../concepts/tls-https.md#3-tls-handshake) moves to the HSM.

[How it works](https://docs.aws.amazon.com/cloudhsm/latest/userguide/ssl-offload-overview.html) with RSA key exchange:

1. The client encrypts the pre-master secret with the server's public key.
2. The web server sends it to the HSM, which decrypts it with the private key and returns the result.
3. The client and server each derive the session keys, and the server encrypts the rest of the session itself with symmetric keys.

The lecture describes this as offloading CPU-heavy processing from the server. That is partly true: the HSM handles the expensive asymmetric operation, but bulk traffic encryption still runs on the web server. The bigger benefit is that the private key never leaves the HSM.

This works well with [ELB TCP pass-through](elb.md#pass-through-nlb-tcp-listener), where the load balancer never decrypts traffic, so the web servers are the TLS endpoints holding the certificates.

### Oracle TDE

Oracle Database can use CloudHSM to protect its [Transparent Data Encryption (TDE)](rds.md#2-tde--transparent-data-encryption-engine-native) master key, because Oracle supports PKCS #11 HSMs. The lecture's example is **Oracle running on EC2**, which matches current AWS guidance.

> [!warning] Gotcha — RDS for Oracle cannot use CloudHSM
> The [Amazon RDS security FAQ](https://aws.amazon.com/rds/features/security/) states that an RDS for Oracle instance cannot be integrated with AWS CloudHSM. For TDE with CloudHSM, Oracle must be installed on EC2. The 2015 RDS for Oracle integration was built for the old CloudHSM Classic service and is no longer available. For RDS, the CloudHSM FAQ points to a KMS CloudHSM key store instead. See [RDS Data Security](rds.md#data-security).

### Certificate authority private keys

An organization running its own certificate authority, for example on Windows Server, can keep the issuing CA's private key in CloudHSM. Certificate signing then happens inside the HSM, so the most sensitive key in the [certificate chain](../concepts/tls-https.md#1-certificates-and-the-chain-of-trust) is never stored on a server disk.

### Other cryptographic work

The CloudHSM user guide also lists general encryption and decryption, document signing and verification, HMAC and CMAC message authentication, and random number generation.

## Pricing

You pay an [hourly fee for each HSM](https://aws.amazon.com/cloudhsm/pricing/) from creation until you delete it, and the rate varies by Region. The [KMS pricing page](https://aws.amazon.com/kms/pricing/) gives an example of $1.60 per HSM per hour in US East (N. Virginia), checked 2026-11-23. At that rate, the recommended two-HSM production cluster costs roughly $2,300 per month before data transfer.

A cluster with no HSMs is not billed, and backup storage is free. To save money in non-production environments, you can delete all HSMs and create new ones from a backup later.

## Related Services

- [KMS](kms.md): the AWS-integrated key service; it can use CloudHSM as a custom key store.
- [Secrets Manager](secrets-manager.md): stores secrets such as passwords and API keys, encrypted with KMS. It does not hold HSM keys.
- [ELB](elb.md#ssltls-connection-modes--bridging-pass-through-offload): TCP pass-through keeps TLS certificates on instances, which can keep their private keys in CloudHSM.
- [RDS](rds.md#data-security): RDS uses KMS for at-rest encryption, and RDS for Oracle does not integrate with CloudHSM.
- [ACM](acm.md): AWS-managed certificates for AWS endpoints; ACM keys are not held in your CloudHSM cluster.

## References

- Adrian Cantrill, AWS Certified Solutions Architect – Associate (SAA-C03): "CloudHSM"
- [Wikipedia: FIPS 140-2](https://en.wikipedia.org/wiki/FIPS_140-2) — linked from the lecture description; background on the FIPS 140 security levels (FIPS 140-3 is the successor standard)
- [AWS CloudHSM FAQs](https://aws.amazon.com/cloudhsm/faqs/) — responsibilities, AWS access, HA recommendations, backups, AWS service integration, billing (checked 2026-11-23)
- [AWS CloudHSM pricing](https://aws.amazon.com/cloudhsm/pricing/)
- [AWS KMS pricing](https://aws.amazon.com/kms/pricing/) — CloudHSM key store example at $1.60 per HSM-hour in us-east-1 (checked 2026-11-23)
- [AWS: CloudHSM cluster architecture](https://docs.aws.amazon.com/cloudhsm/latest/userguide/cluster-architecture.html)
- [AWS: CloudHSM clusters](https://docs.aws.amazon.com/cloudhsm/latest/userguide/clusters.html) — 1 to 28 HSMs per cluster, default quota 6 per account per Region
- [AWS: CloudHSM backups](https://docs.aws.amazon.com/cloudhsm/latest/userguide/backups.html)
- [AWS: CloudHSM HSM types](https://docs.aws.amazon.com/cloudhsm/latest/userguide/hsm-types.html) and [cluster modes](https://docs.aws.amazon.com/cloudhsm/latest/userguide/cluster-hsm-types.html)
- [AWS: CloudHSM deprecation notifications](https://docs.aws.amazon.com/cloudhsm/latest/userguide/compliance-dep-notif.html) — hsm1.medium end of support March 31, 2026
- [AWS: HSM user types for CloudHSM CLI](https://docs.aws.amazon.com/cloudhsm/latest/userguide/understanding-users.html)
- [AWS: Client SDK 5 components](https://docs.aws.amazon.com/cloudhsm/latest/userguide/use-hsm.html)
- [AWS: cluster security group](https://docs.aws.amazon.com/cloudhsm/latest/userguide/configure-sg.html)
- [AWS: CloudHSM use cases](https://docs.aws.amazon.com/cloudhsm/latest/userguide/use-cases.html)
- [AWS: how SSL/TLS offload with CloudHSM works](https://docs.aws.amazon.com/cloudhsm/latest/userguide/ssl-offload-overview.html)
- [AWS KMS: AWS CloudHSM key stores](https://docs.aws.amazon.com/kms/latest/developerguide/keystore-cloudhsm.html)
- [AWS KMS developer guide overview](https://docs.aws.amazon.com/kms/latest/developerguide/overview.html) — KMS HSMs are FIPS 140-3 Security Level 3 validated
- [Amazon RDS security FAQ](https://aws.amazon.com/rds/features/security/) — RDS for Oracle cannot be integrated with CloudHSM (checked 2026-11-23)
