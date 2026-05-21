# AWS S3 Object Encryption — Study Notes (CSE & SSE)

> Source: Adrian Cantrill — S3 Object Encryption (CSE/SSE)

---

## 1. Foundational Concepts

### Buckets aren't encrypted — objects are

- Encryption is defined **at the object level**, not the bucket level.
- "Bucket default encryption" is a separate feature that just sets the default behavior for new objects.
- Different objects in the same bucket can use **different** encryption settings.

### Encryption at rest vs encryption in transit

- **In transit** — HTTPS tunnel between client and S3 endpoint. This happens by default and is *not* what this lesson is about.
- **At rest** — how the object is stored on disk. This is the focus.
- If someone physically stole the disks, encryption at rest is what stops them from reading your data without the key.

---

## 2. Client-Side Encryption (CSE) vs Server-Side Encryption (SSE)

![[S3-SSE-vs-CSE.png]]

| Dimension                  | Client-Side Encryption (CSE)    | Server-Side Encryption (SSE)    |
| -------------------------- | ------------------------------- | ------------------------------- |
| Who encrypts?              | You (before upload)             | S3 (after receipt)              |
| What S3 sees in the tunnel | Ciphertext only                 | Plaintext (inside HTTPS tunnel) |
| Who manages keys?          | You                             | Varies by SSE type              |
| CPU cost of encryption     | On your client                  | On AWS                          |
| Trust model                | Zero trust in AWS for plaintext | Some trust required             |

Key insight: **both** still use HTTPS in transit. The difference is whether the bytes inside that HTTPS tunnel are already ciphertext (CSE) or still plaintext (SSE).

> ⚠️ **CSE is NOT the same as SSE-C.** SSE-C is still server-side encryption — S3 does the crypto. CSE means S3 never touches plaintext.
> 📌 **SSE is now mandatory.** AWS no longer allows storing unencrypted objects in S3. You can't opt out — you only choose *which* SSE method.

---

## 3. The Three SSE Methods

There are three SSE flavors, differing in **(a) who manages the keys** and **(b) who performs the encryption/decryption**.

| Method      | Full Name                                                          |
| ----------- | ------------------------------------------------------------------ |
| **SSE-C**   | Server-Side Encryption with **C**ustomer-provided keys             |
| **SSE-S3**  | Server-Side Encryption with **S3**-managed keys (default, AES-256) |
| **SSE-KMS** | Server-Side Encryption with **KMS** keys                           |

Two core components in every SSE flow:

1. **Encryption/decryption process** — plaintext + key + algorithm → ciphertext (symmetric: same key both ways).
2. **Key generation & management** — who creates, stores, rotates, and authorizes use of the keys.

---

## 4. SSE-C — Customer-Provided Keys

![[S3-SSE-C.png]]

**You manage keys. S3 manages encryption.**

### Upload flow (PUT)

1. You send the **plaintext object + your encryption key** to S3 over HTTPS.
2. S3 encrypts the object using your key.
3. S3 stores the ciphertext object **and a one-way hash of your key** alongside it.
4. S3 **destroys the key** — it's not stored.

### Download flow (GET)

1. You provide S3 with the **same key** used to encrypt.
2. S3 compares the hash to verify the key is correct.
3. If valid, S3 decrypts, returns plaintext over HTTPS, and discards the key again.

### Why the one-way hash matters
- The hash cannot be reversed to reconstruct the key.
- It can only **verify** whether a supplied key matches the original. This protects you against losing the key without S3 retaining a usable copy.

### When to use SSE-C
- You **must** own and manage your keys (regulatory/compliance reasons).
- You're okay offloading the CPU-heavy crypto work to S3.
- You trust S3 to discard keys after use (backed by AWS's audited processes).

---

## 5. SSE-S3 — S3-Managed Keys (Default, AES-256)

![[S3-SSE-S3.png]]

**S3 manages everything: keys and encryption.**

### How it works (envelope encryption)

1. You upload plaintext to S3.
2. S3 generates a **unique per-object key** (the "object key" / DEK).
3. S3 encrypts the object with that per-object key.
4. S3 uses an internal, service-managed **S3 master key** to encrypt the per-object key.
5. The plaintext per-object key is discarded.
6. Stored on disk: **ciphertext object + ciphertext (encrypted) per-object key**.

### Properties
- Algorithm: **AES-256** → exam tip: *"AES-256" → think SSE-S3*.
- The S3 master key is **invisible, non-configurable, and rotated by S3** — you have zero control.
- Lowest admin overhead. Good default for most workloads.

### Three limitations of SSE-S3
1. ❌ **No key control** — you can't pick, view, or manage the key.
2. ❌ **No rotation control** — rotation is on S3's schedule, not yours.
3. ❌ **No role separation** — anyone with S3 full-admin permissions can read object data. There's no way to separate "manages the bucket" from "reads the data."

---

## 6. SSE-KMS — KMS-Managed Keys

![[S3-SSE-KMS.png]]

**S3 + KMS jointly manage keys. S3 handles encryption.**

### How it works

1. You create a **customer-managed KMS key** (or use the AWS-managed default).
2. On PUT, S3 calls KMS and asks it to generate a **Data Encryption Key (DEK)** from the KMS key.
3. KMS returns **two versions of the same DEK**:
   - Plaintext DEK
   - Ciphertext DEK (encrypted by the KMS key)
4. S3 uses the plaintext DEK to encrypt the object.
5. S3 immediately discards the plaintext DEK.
6. Stored on disk: **ciphertext object + ciphertext DEK**.

### Why the DEK pattern?

- KMS keys can only directly encrypt payloads **≤ 4 KB**.
- The DEK has no such limit, so it does the actual object encryption.
- KMS only **generates** DEKs — it never stores them.

### Key benefits of SSE-KMS

- ✅ **Key rotation control** — you decide the rotation policy on the KMS key.
- ✅ **Configurable key policy** — fine-grained IAM/key-policy permissions.
- ✅ **Auditing via CloudTrail** — every KMS key usage is logged.
- ✅ **Role separation** — the killer feature (see below).

### Role separation (the "Phil" example)

- Phil is an **S3 full administrator** — full control over the bucket and its objects.
- Phil has **zero permissions on the KMS key**.
- Result: Phil can manage the bucket and objects (lifecycle, policies, etc.) but **cannot read object contents**, because he can't decrypt the DEK without the KMS key.
- This is impossible to achieve with SSE-S3 — and essential in regulated industries (finance, medical) where infrastructure admins must not see customer data.

---

## 7. Summary Table

![[S3-SSE-summary.png]]

| Method                           | Key Management | Encryption Processing | Notable Extras                                            |
| -------------------------------- | -------------- | --------------------- | --------------------------------------------------------- |
| **Client-Side Encryption (CSE)** | YOU            | YOU                   | S3 never sees plaintext                                   |
| **SSE-C**                        | YOU            | S3                    | You provide key per request; S3 stores key hash only      |
| **SSE-S3**                       | S3             | S3                    | AES-256; no key control; no role separation               |
| **SSE-KMS**                      | S3 & KMS       | S3                    | Key rotation control + role separation + CloudTrail audit |

---

## 8. Decision Guide — When to Use Which

- **CSE** — When you cannot trust AWS with plaintext, ever. You also accept the CPU cost and full key-management burden. Often used via the AWS Encryption SDK in the application.
- **SSE-C** — When *you* must own the keys but want AWS to handle the heavy crypto. You're trusting AWS to discard keys post-use.
- **SSE-S3** — Sensible default for most workloads. Don't use if you need key rotation control, key access control, or role separation.
- **SSE-KMS** — When you need **role separation**, key rotation control, key-level audit trails, or fine-grained key permissions. Standard choice for regulated workloads.

---

## 9. Exam-Day Cheat Sheet

- "Buckets are encrypted" → **wrong**. Objects are encrypted; bucket default encryption is just a default setting.
- "AES-256" → **SSE-S3**.
- "Role separation" / "S3 admin must not read data" → **SSE-KMS**.
- "Key rotation control" / "audit key usage in CloudTrail" → **SSE-KMS**.
- "Customer provides the key on every request, S3 stores a hash" → **SSE-C**.
- "AWS never sees plaintext" → **CSE** (client-side, not really an S3 feature).
- SSE is **mandatory** on S3 — you cannot store unencrypted objects anymore.
- CSE and SSE can be **combined** — nothing stops you from encrypting client-side AND letting S3 also apply SSE.
- KMS keys themselves can only encrypt ≤ 4 KB → that's why DEKs exist for the actual object encryption.
