# Implementing DNSSEC with Route 53

> **Prerequisite:** This builds on DNS and DNSSEC fundamentals (DNS resolution flow, the chain of trust, KSK/ZSK roles, DNSKEY/RRSIG/DS records). If those aren't solid, review them first.

---

## Reading the Diagram

![[R53-implementing-dnssec.png]]

The diagram overlays the **DNSSEC signing/validation process** on top of the normal Route 53 resolution architecture. Read it in three zones:

**Left zone — normal resolution path (green background).** This is standard Route 53 working as you already know it. Bottom-right, a user on a laptop queries a record (`animals4life.org`). The dashed red arrows trace the resolution walk: laptop → **SomeISP DNSSEC Resolver** → **Root Servers** → **.ORG TLD Servers** → the four Route 53 name servers hosting the zone (the column of shield-marked servers). The `.ORG NS Records` label marks the delegation from the TLD to those Route 53 name servers.

**Center zone — what gets created when signing is enabled (green key panels).** The top panel shows the **KSK/ZSK DNSKEY** record (both public keys published together) and below it the **RRSIG DNSKEY** (the signature over that DNSKEY record). To the right, **ZSK PRIVATE** is annotated _"ZSK Creation and Rotation handled by R53 Internally"_ — KMS is not involved in the ZSK. The pink **"Add DS Record"** arrow runs from the zone up to the **.ORG TLD Servers**, and **"Provide KSK to parent"** marks the chain-of-trust step.

**Bottom + right zones — the AWS services involved.**

- **KMS box (bottom center, red border):** an **Asymmetric KMS key pair** — **Public KEY (KSK)** and **Private KEY (KSK)** — that must live in **US-EAST-1**. These back the key signing key.
- **CloudWatch box (bottom right):** _"Create Alarms for `DNSSECInternalFailure` and `DNSSECKeySigningKeysNeedingAction`."_
- **VPC box (top right, blue):** EC2 instances querying via the **VPC +2 Address (R53 Resolver)**, with the note that **DNSSEC Validation can be enabled for VPCs** — invalid results on DNSSEC-enabled zones won't be returned.

The thick purple arrows connect the resolver/VPC validation path; the pink arrows are the signing/chain-of-trust setup actions.

---

## The Starting Point: Normal Route 53 Resolution

Before DNSSEC, resolution of `animals4life.org` works the way it always has:

1. Laptop → DNS resolver
2. Resolver → **Root servers** (asking who handles `.org`)
3. Resolver → **.org TLD name servers** (asking who handles `animals4life.org`)
4. Resolver → the **four Route 53 name servers** hosting the zone

Inside an AWS VPC, instances reach the same zone via the **Route 53 Resolver at the VPC +2 address**.

DNSSEC layers signing and a chain of trust on top of this — it does **not** replace the resolution path.

---

## Enabling DNSSEC Signing — Step by Step

Enabling DNSSEC on a hosted zone is done via the **Route 53 console or CLI**. The process unfolds as follows.

### 1. KMS creates an asymmetric key pair (the KSK source)

When signing is initiated, the process **starts with KMS**. An **asymmetric key pair** (public + private) is created. This can be done as a separate step or as part of enabling signing.

> 🎯 **Conceptually** you can think of these as the **Key Signing Keys (KSKs)**. Precisely, though, the KSK is _created from_ these KMS keys — the KMS keys aren't literally the KSK itself. This nuance isn't exam-critical, but it explains why the diagram labels them "Public/Private KEY (KSK)" while also showing a separate DNSKEY entry.

> ⚠️ **The KMS keys must be in `us-east-1` (N. Virginia).** This is a hard requirement and a common exam point. Keep it in mind.

### 2. Route 53 creates the Zone Signing Keys (ZSKs) — internally

Route 53 creates the **ZSKs internally**. Both **creation and management/rotation of the ZSKs are handled entirely within Route 53**.

> 🎯 **KMS is _not_ involved in the ZSK.** KMS backs only the KSK side. This split (KSK ↔ KMS, ZSK ↔ R53 internal) is a frequent exam distinction.

### 3. Route 53 publishes the public keys in a DNSKEY record

Route 53 adds the **public parts of both the KSK and ZSK** into a **DNSKEY record** in the hosted zone. This tells DNSSEC resolvers which public keys to use to verify signatures on other records in the zone.

> 📚 In `dig` output the public keys appear as flags: **257 = KSK** public part, **256 = ZSK** public part.

### 4. The private KSK signs the DNSKEY record → RRSIG DNSKEY

The **private key signing key** signs the DNSKEY records, producing the **RRSIG DNSKEY** record. This signature lets any DNSSEC resolver confirm the DNSKEY records are **valid and unchanged**.

At this point, **in-zone signing is configured** — that's "step one" complete.

### 5. Establish the chain of trust with the parent (DS record)

The parent zone (here, the **.org TLD**) must add a **DS record (Delegated Signer)** — a **hash of the public KSK** for this zone. How you make this happen depends on where the domain is registered:

|Registration|How the DS record gets added|
|---|---|
|**Registered via Route 53**|Use the **Registered Domains** area of the console (or equivalent CLI). Route 53 liaises with the appropriate TLD and adds the DS record for you.|
|**Domain registered elsewhere** (R53 only hosts the zone)|You must perform this step **manually** at your registrar.|

Once the DS record is in place, the **.org TLD trusts this zone** via the hash of its public KSK, completing the chain of trust. Records in the zone are then signed using the KSK or ZSK as appropriate.

### 6. Configure CloudWatch alarms

As part of enabling DNSSEC, create alarms for:

- **`DNSSECInternalFailure`**
- **`DNSSECKeySigningKeysNeedingAction`**

> ⚠️ Both indicate a DNSSEC problem with the zone that needs **urgent** attention — e.g., an issue with the KSK itself or a problem interacting with KMS.

### 7. (Optional) Enable DNSSEC validation for VPCs

For DNSSEC-enabled zones, if a record **fails validation** (mismatched signature or otherwise untrusted), the result **won't be returned**.

> 🎯 This **only affects DNSSEC-enabled zones**. Non-DNSSEC zones are unaffected and will always return results.

---

## Walkthrough: What `dig` Shows at Each Stage

The lesson demonstrates the process live. The illustrative values below (domain, IP, flags) show the _pattern_ — they aren't values to memorize.

**Before enabling DNSSEC** — query for DNSKEY records:

```
dig animalsforlife.org dnskey +dnssec
```

Returns **no DNSSEC results** — expected, since the zone isn't signed.

**After enabling signing** (allow for caching/propagation — re-run after a few minutes): The same query now returns:

- A **DNSKEY 256** record → public **ZSK**
- A **DNSKEY 257** record → public **KSK**
- The corresponding **RRSIG DNSKEY** → signature over those keys using the private KSK

> ⚠️ Empty results immediately after a change are usually **DNS caching**, not a failure. Wait and re-query.

**Verifying the DS record reached the parent** — first list the .org authoritative name servers, then query one directly for the DS record:

```
dig org ns +short
dig animalsforlife.org DS @<one-of-the-.org-nameservers>
```

Once propagated, the **answer section shows the DS record** — a hash of the zone's public KSK, now living in the .org zone. (Propagation can take **minutes to a few hours**, depending on the TLD and Route 53's relationship with it.)

**Confirming records in the zone are signed** — create a test A record (e.g. `test.animalsforlife.org → 1.1.1.1`), then:

```
dig test.animalsforlife.org A            # normal result, just the A record
dig test.animalsforlife.org A +dnssec    # A record PLUS an RRSIG
```

With `+dnssec`, you also get an **RRSIG** — a signature of the A record made with the **private ZSK**, verifiable using the public ZSK from the DNSKEY record. That completes an **end-to-end chain of trust** from the DNS root down to the individual resource record.

---

## Production Caution: Order and Timing Matter

> ⚠️ On a **production zone**, the **order of these steps and TTL-based waiting periods are critical**. You must account for the TTL values in your zone before enabling signing and before each subsequent step, or you risk breaking resolution. A fresh, empty zone (as in the demo) doesn't require this care, but real zones do.

> 📚 AWS publishes time-critical prerequisites for enabling DNSSEC signing — review them before doing this on a live domain.

---

## Console Notes (from the demo)

- The KMS UI may show **"CMK"** or **"KMS key"** depending on UI version — AWS renamed CMKs to KMS keys; treat them as the same thing. You're creating an **asymmetric KMS key**.
- When adding the DS record at the registrar, match the **signing algorithm** Route 53 reports (the demo used **ECDSAP256SHA256**) and paste in the **public KSK**.
- After creating a signed record, the **hosted zone UI looks unchanged** — the signing is visible only via `dig +dnssec`.

---

## 🎯 Exam Quick Reference

|Item|Key fact|
|---|---|
|**Where to enable**|Route 53 console or CLI, per hosted zone|
|**KSK backing**|Asymmetric **KMS** key pair|
|**KMS key region**|**Must be `us-east-1`**|
|**ZSK creation/rotation**|Handled **internally by Route 53** — **KMS not involved**|
|**DNSKEY record**|Holds **public** KSK (257) + ZSK (256)|
|**RRSIG DNSKEY**|DNSKEY record signed by the **private KSK**|
|**Record RRSIGs**|Zone records signed by the **private ZSK**|
|**Chain of trust**|Parent adds a **DS record** = hash of the zone's **public KSK**|
|**DS via R53-registered domain**|Done from **Registered Domains** console; R53 liaises with TLD|
|**DS via external registrar**|Must be added **manually**|
|**CloudWatch alarms**|`DNSSECInternalFailure`, `DNSSECKeySigningKeysNeedingAction`|
|**VPC validation**|Optional; **failing records not returned** for DNSSEC zones only|
|**Production gotcha**|Respect **TTLs / step ordering**; propagation can take **minutes–hours**|

---

> 📝 **Editorial note (not from transcript):** the comparison tables, the consolidated "Exam Quick Reference," and the 🎯/⚠️/📚 callout framing are my additions for study/recall. The `dig` command forms are reconstructed from the narrated demo to illustrate the workflow — verify exact flags/output against a live run. Keep or strip these as you prefer.