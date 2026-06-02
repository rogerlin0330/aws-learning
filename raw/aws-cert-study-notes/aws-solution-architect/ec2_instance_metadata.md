# EC2 Instance Metadata

## What it is

The **Instance Metadata Service (IMDS)** is a service EC2 provides _to_ its instances. It exposes data _about_ the instance — and the environment it runs in — that anything running inside the instance can query, but which the OS often can't see on its own.

> **Analogy:** The instance OS is a guest in a hotel room. It knows what's inside the room (its private IP, its IPv6), but not the building's street address (its public IPv4). The metadata service is the front desk it can phone to ask, "What address am I at?"

---

## The access endpoint (memorize this)

Metadata is reached from _inside_ the instance at a fixed link-local IP — the same on every instance:

```
http://169.254.169.254/latest/meta-data/
```

|Part|Value|
|---|---|
|IP address|`169.254.169.254`|
|Path|`/latest/meta-data/`|
|Then append|the attribute you want (e.g. `public-ipv4`)|

> 🧠 **Exam tip:** Exam questions can ask you to pick the _exact_ URL. Memorize both halves. Cantrill's mnemonic: **"169.254 repeated"** → `169.254.169.254`, and you always want **"the latest meta-data."**

---

## What metadata can tell you

Information is grouped into **categories** — hostname, events, security groups, networking, and more. The most commonly queried category is **networking**.

|Use case|Why it matters|
|---|---|
|**Networking info**|The OS can read its private IPv4 and its IPv6, but **never** its public IPv4. Metadata is how an app on the instance discovers its public address.|
|**IAM role credentials**|When an instance assumes an IAM role, the **temporary credentials** are delivered via metadata. Apps read them from here to access AWS resources.|
|**Temporary SSH keys**|EC2 Instance Connect works by injecting a short-lived SSH key behind the scenes — delivered through the metadata service.|
|**User data**|Bootstrap scripts that run automatic configuration at launch are exposed through metadata.|

---

## Public vs. private addressing (a core mental model)

This is the concept the demo is really teaching:

|Address type|Configured inside the OS?|How to see it|
|---|---|---|
|**Private IPv4**|✅ Yes|Visible in the OS (e.g. via interface config) and the console|
|**IPv6**|✅ Yes — always public, no public/private distinction|Visible in the OS and the console's Networking tab|
|**Public IPv4**|❌ **Never**|Console UI, or **metadata service** from inside the instance|

> ⚠️ **Critical fact:** At **no point** in an instance's lifecycle is the public IPv4 address configured within the operating system. The **Internet Gateway** performs the translation between the private (internal) address and the public address. The only IPv4 the instance itself "has" is its private IPv4.

---

## Security — the gotcha that shows up repeatedly

> ⚠️ **Exam favorite:** The metadata service has **no authentication** and is **not encrypted**.

- Anyone who reaches a shell on the instance can, by default, read the metadata.
- You _can_ restrict it with **local (per-instance) firewall rules** that block the `169.254.169.254` address — but that's per-instance admin overhead.
- **Treat metadata as something that can and does get exposed.** Don't rely on its obscurity to protect secrets.

---

## Querying it in practice (concepts)

Two common approaches from inside the instance:

1. **`curl`** against the metadata URL plus an attribute — e.g. requesting `public-ipv4` returns the public address the OS otherwise can't see; `public-hostname` returns the public DNS name.
2. **The `ec2-metadata` tool** — a downloadable helper (made executable with `chmod`) that wraps the same service with simple flags, making interactive queries easier. `--help` lists everything it can fetch (AMI ID, availability zone, security groups, etc.).

Both hit the _same_ underlying service; the tool is just a friendlier front end.

---

## One-line recall

> Metadata lives at **169.254.169.254 → /latest/meta-data/**, it's **unauthenticated and unencrypted**, and it's the only way an instance can discover its own **public IPv4** — because that address lives on the **Internet Gateway**, never in the OS.