# DNS 101 — How DNS Actually Works (Walking the Tree)

## Problem / Context — What DNS Actually Solves

Strip away the technical detail and DNS does one thing: a person, device, or service has a **DNS name** and needs the **IP address(es)** that provide services for that name.

> Example: you have `www.netflix.com` and you need the IP address(es) you can connect to in order to reach Netflix.

Somewhere in the world there is a **DNS zone** for `netflix.com` that holds the **record** linking `www.netflix.com` to one or more IPs. The entire challenge is: **how do you find that specific zone?**

That is the whole job of DNS — **a huge global distributed database** whose function is to let you **locate the one specific zone that can give you an authoritative answer**, so you can query it and get a response.

> 🎯 **Exam framing:** Everything in the query flow below exists for one reason — _to find the correct zone_. No single nameserver holds all the answers, but every query moves you one step closer.

---

## Solution / Mechanics — The Query Flow

![[DNS-101-walking-the-tree.png]]

### How to read this diagram

- **Context bar (top):** the trust chain — Root → `.com` Registry → `netflix.com` NS. Each `TRUST` arrow is a **delegation** via **NS records** (each level hands control of the next level down).
- **Main objects (the walk):** the DNS Resolver (bottom-left) does the work _on your behalf_, querying each tier left-to-right and climbing the tree. Green = queries going out; pink = answers coming back.
- **Footer (flasks, right):** the two takeaways — _No one NS has all the answers_ and _Every query gives you the next step_.

### Step-by-step (numbered as in the diagram)

|#|Actor|Action|
|---|---|---|
|1|Client (your machine)|Checks **local cache & hosts file** first|
|2|Client → Resolver|Queries the **DNS Resolver**|
|3|Resolver|Checks its own **local cache**|
|4|Resolver → Root|Asks **Root** for `www.netflix.com`|
|5|Root → Resolver|Returns the **`.com` NS** records|
|6|Resolver → `.com` Registry|Asks **`.com` TLD NS** for `www.netflix.com`|
|7|`.com` → Resolver|Returns the **`netflix.com` NS** records|
|8|Resolver → Netflix NS|Asks **`netflix.com` NS** for `www.netflix.com`|
|9|Netflix NS → Resolver|Returns the **DNS record** (`netflix.com` ⇒ IP) — **authoritative**|
|10|Resolver|**Caches** the result|
|11|Resolver → Client|Returns the **result to the client**|

### The walk, in plain terms

1. **Hosts file / local cache first.** The hosts file is a **static name→IP mapping that overrides DNS**, so it's checked first; the OS/application may also cache.
2. **DNS Resolver takes over.** A resolver typically runs on a **home router or at your ISP** and performs the query for you. It also caches.
3. **Root zone.** Resolvers have the **root server IPs hard-coded** (the list is maintained by the OS vendor). Root can't answer for `www.netflix.com`, but it holds **NS records for `.com`** pointing at the `.com` TLD nameservers — this is how the root **delegates `.com` to VeriSign**.
4. **`.com` TLD nameservers (VeriSign).** They can't give the final answer either, but (assuming `netflix.com` is registered) they hold **NS records for `netflix.com`** — this is how VeriSign **delegates `netflix.com`** at registration time.
5. **`netflix.com` nameservers.** These are **authoritative** (they host the zone/zone file _and_ are pointed at by the `.com` TLD zone), so they return the **authoritative result**.
6. **Cache + return.** The resolver caches the answer (faster next time) and returns it to the client.

This end-to-end process is called **walking the tree**.

---

## Authoritative vs Non-Authoritative

||Authoritative|Non-Authoritative|
|---|---|---|
|Who gives it|The nameserver the **TLD zone points at** for the domain (hosts the actual zone file)|A **resolver** answering from its **cache**|
|Trust|The definitive answer|A cached copy — may be stale|
|🎯 Exam angle|Only the delegated NS is authoritative|A resolver **always** returns cached results as **non-authoritative**, even if correct — "in most cases nobody cares, but know the distinction"|

---

## The Tech — What the Tools Actually Show

![[DNS-101-walking-the-tree-the-tech.png]]

### How to read this diagram

Three stacked boxes = the three tiers (**Root Zone → `.com` Zone → `netflix.com` Zone`**). At each tier (left), the server says _"Non-authoritative... can't find it directly"_ but (right) responds with **"Authoritative answers can be found from:"** followed by the **NS records for the next tier down** — visually proving "every query gives you the next step." The final tier returns an actual record (here a **CNAME**, step 9).

- **Step 5 — Root returns `.com` TLD nameservers** (e.g. `a.gtld-servers.net` … `d.gtld-servers.net`).
- **Step 7 — `.com` returns `netflix.com` nameservers** (e.g. AWS Route 53 servers like `ns-81.awsdns-10.com`, `ns-659.awsdns-18.net`, `ns-1984.awsdns-56.co.uk`, `ns-1372.awsdns-43.org`).
- **Steps 8–9 — `netflix.com` NS returns the record**, which here is a **CNAME**: `www.netflix.com` → `www.dradis.netflix.com`.

> ⚠️ **Trap:** The authoritative answer isn't always an IP. Here it's a **CNAME** (another DNS name), so the resolver must **walk the tree again** for _that_ name. Many real queries end in another DNS name requiring further queries — a common reason apps perform poorly when many DNS calls are chained and network latency stacks up.

> 📝 _Editorial note (mine, not from the transcript): the diagram's NS server names (`*.gtld-servers.net`, `*.awsdns-*`) and the IP `205.251.192.81#53` are **illustrative examples** of the format, not values to memorize._

---

### Command Examples — `nslookup` and `dig`

> 📝 _Editorial addition (mine — to reproduce the screenshot's terminal output yourself). These are illustrative commands; exact NS names/IPs returned will differ._

The screenshot's output style (`Non-authoritative answer`, `Authoritative answers can be found from:`, `canonical name =`) is classic **`nslookup`**. Below are both tools for each tier of the walk.

**1. See the full recursive answer (what your resolver gives you)**

```bash
# nslookup — note "Non-authoritative answer" since it comes from the resolver cache
nslookup www.netflix.com

# dig — concise; +short prints just the result chain
dig www.netflix.com
dig +short www.netflix.com
```

**2. Step 4–5: Ask the ROOT zone (returns .com TLD nameservers)**

```bash
# nslookup: set query type to NS and ask a root server directly
nslookup -type=NS com. a.root-servers.net

# dig: ask a root server for the .com delegation (look in the AUTHORITY section)
dig @a.root-servers.net com. NS
```

**3. Step 6–7: Ask a .com TLD nameserver (returns netflix.com nameservers)**

```bash
# nslookup
nslookup -type=NS netflix.com. a.gtld-servers.net

# dig
dig @a.gtld-servers.net netflix.com. NS
```

**4. Step 8–9: Ask the netflix.com authoritative NS directly (authoritative answer)**

```bash
# nslookup — answer is authoritative because you're asking the delegated NS itself
nslookup www.netflix.com ns-81.awsdns-10.com

# dig — note the absence of the "Non-authoritative" tag; check the 'aa' flag
dig @ns-81.awsdns-10.com www.netflix.com
```

**5. Walk the entire tree automatically (great for seeing every delegation step)**

```bash
# dig +trace performs the full walk from root downward, printing each referral
dig +trace www.netflix.com
```

**6. Just the nameservers / just a CNAME**

```bash
nslookup -type=NS netflix.com        # list the domain's nameservers
nslookup -type=CNAME www.netflix.com # reveal the CNAME (e.g. www.dradis.netflix.com)

dig netflix.com NS +short
dig www.netflix.com CNAME +short
```

> 💡 **Reading the output:**
> 
> - **`nslookup`** prefixes resolver answers with **`Non-authoritative answer:`**; querying the delegated NS directly drops that line.
> - **`dig`** shows an **`aa`** flag in the header `flags:` line when the answer is **authoritative**; delegations appear in the **AUTHORITY** section, and `dig +trace` makes the whole walk visible.

---

## ⚠️ Traps & Gotchas

- **Hosts file overrides DNS** and is checked _before_ any DNS query — a classic "why isn't my DNS change working?" gotcha.
- **Resolver answers are always non-authoritative**, even when cached correctly.
- **Root server IPs are hard-coded** into DNS software (maintained by the OS vendor) — they are not "looked up."
- **Authoritative ≠ IP.** A **CNAME** answer means another full walk of the tree for the target name.
- **Delegation = trust.** Each tier only _points at_ the next tier's NS (via NS records); it does not hold the lower zone's data.

---

## 🎯 Exam Quick-Reference Recap

|Concept|Key fact|
|---|---|
|**Function of DNS**|Locate the **one zone** that can give an **authoritative answer**|
|**Walking the tree**|Root → TLD (`.com`) → domain (`netflix.com`) NS, each step closer|
|**Resolver**|Queries on your behalf (home router / ISP); caches; returns **non-authoritative**|
|**Hosts file**|Static name→IP map, **overrides DNS**, checked **first**|
|**Root zone**|Hard-coded IPs; holds **NS records for TLDs** (e.g. `.com` → VeriSign)|
|**TLD zone (`.com`)**|Run by VeriSign; holds **NS records for registered domains**|
|**Authoritative NS**|Hosts the zone file **and** is pointed at by the TLD zone|
|**Delegation**|Done via **NS records**; this is how **trust** flows down the tree|

### Trigger keywords (exam → answer)

- "Static mapping that overrides DNS" → **hosts file**
- "Queries on your behalf / runs on router or ISP" → **resolver**
- "Returns cached result, marked non-authoritative" → **resolver cache**
- "Hard-coded IPs maintained by OS vendor" → **root servers**
- "Points `.com` at VeriSign" / "points domain at its NS" → **NS records / delegation**
- "Hosts the zone file and is pointed at by the TLD" → **authoritative nameserver**
- "Process from root down to the domain" → **walking the tree**

### Mnemonic _(editorial addition — mine, keep or cut)_

**"H‑R‑R‑T‑D"** → **H**osts file, **R**esolver, **R**oot, **T**LD, **D**omain — the order a query is resolved.