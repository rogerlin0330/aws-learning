---

tags:

- aws
- saa-c03
- rds
- rds-proxy
- databases
- lambda
- high-availability course: Adrian Cantrill AWS SAA-C03 lesson: "Relational Database Service (RDS) - RDS Proxy" level: associate-shared

---

# RDS Proxy

## Overview

RDS Proxy is a **fully managed database proxy** that sits between your clients and your RDS/Aurora database instances. Rather than each client opening its own connection to the database, clients connect to the proxy, and the proxy maintains a **long-lived pool of connections** to the database that it reuses and multiplexes across many client connections.

It matters for the exam both in its own right _and_ because it underpins many other RDS architectures — especially serverless ones.

### Why it exists — the three problems

**1. Connection establishment is expensive.** Opening and closing a database connection takes time and consumes database resources (memory + CPU). For small operations — reading or writing a tiny amount of data — the connection overhead can dominate the actual work. This is worst with **serverless**: many Lambda invocations each opening and closing their own connection, while you're billed for that connection setup time as compute.

**2. Handling database failure in application code is hard.** How long do you wait? What does the app do while waiting? When do you declare failure? How do you react to a Multi-AZ failover to the standby? Building all of this into your application adds significant overhead and risk.

**3. Running your own proxy is hard.** A database proxy solves problems 1 and 2 — but you may have no proxy experience, and even if you do, managing proxies _at scale_ is its own burden. This is where the "fully managed" part of RDS Proxy adds the value.

### The architectural change

```
BEFORE:  Client ──(new connection every time)──> Database
AFTER:   Client ──(fast, cheap connection)──> RDS Proxy ══(long-term pool, multiplexed)══> Database
```

Two things follow from this:

- **Connection pooling** — the proxy→DB connections are created once and maintained long-term. They are _not_ created and torn down per application request or per Lambda invocation.
- **Multiplexing** — the proxy can service a **larger** number of client→proxy connections using a **smaller** number of proxy→DB connections. This is especially valuable on small DB instances where resources are at a premium.

---

## Diagram Walkthrough

![[rds-proxy-architecture.png]]

**Context (outer frame).** A dashed blue box labelled with a flag icon marks the **us-east-1 region**. Inside it, a purple-outlined box is **VPC1**. The entire RDS Proxy architecture lives inside the VPC — nothing here is public.

**The AZ bands (right side).** Three horizontally banded rows run across the VPC, labelled on the far right: **AZA** (orange band), **AZB** (purple band), **AZC** (pink band). Each band contains subnets rendered as blue-grey squares with a small blue padlock icon in the corner — the padlock denotes a **private subnet**.

**The database tier (rightmost column).**

- In **AZB** sits the **Primary** RDS instance.
- In **AZC** sits the **Standby**.
- A **green arrow** runs from Primary down to Standby: this is the **Multi-AZ synchronous replication** relationship.

**The proxy tier (middle column).** An **RDS Proxy icon appears in AZA, AZB, and AZC**, and a **dotted vertical box** encircles all three. That dotted box is the visual shorthand for _"this is one logical proxy, spanning all three AZs"_ — the proxy is a managed, highly available, auto-scaling service, not three separate things you manage.

**The application tier (left column inside VPC).**

- **AZA** and **AZC** each contain an **EC2 instance** (the Catagram application servers), each with a **double-headed orange arrow** to its local RDS Proxy — client↔proxy connections.
- **AZB** contains a purple circular **ENI icon** — this is the **Lambda ENI**, since these Lambda functions are configured for VPC networking and run from the AZB subnet.

**The connection lines — read these carefully, this is the whole point of the diagram.**

- **Left of the proxy (client→proxy):** a _thick bundle_ of orange lines runs from the five Lambda function icons, through the Lambda ENI, into the RDS Proxy in AZB. **Many** connections.
- **Right of the proxy (proxy→DB):** another bundle of orange lines runs from the proxy toward the Primary — but they **pass through a small pink ellipse** that visually pinches them together. A **pink arrow labelled "Long Term Connection Pool"** points at that ellipse; a **blue arrow labelled "Multiplexing"** points into that same region. The pinch is the diagram _showing_ you the multiplexing: a large number of client connections funnelling into a smaller, persistent pool of database connections.

**The Lambda functions (far left).** Five orange Lambda (λ) icons sit outside the VPC box, with orange arrows into the VPC — representing invocations. They're drawn outside because Lambda is a regional service; the ENI is how they reach into the VPC.

**The four flask callouts (🧪 orange beaker icons).** These are Cantrill's "key point" markers:

1. _"RDS Proxy => DB Instance connections can be reused... avoiding the lag of establishment, usage & termination for each invocation"_ — the connection-reuse benefit.
2. _"Abstracts client away from DB failure or failover events"_ — the resilience benefit.
3. _"Client => Proxy connection is established and WAITS even if the Target DB is unresponsive"_ — this is _how_ the abstraction works. The client isn't errored out; it's held.
4. _"Each Lambda function connects to the Proxy. This is MUCH quicker to establish vs direct to database"_ — the serverless benefit.

**Reading the failover story off the diagram.** Clients hold connections to the proxy endpoint. If the Primary in AZB fails, the proxy establishes new pool connections to the promoted Standby in AZC **in the background**. The client-side lines never break — the clients stay connected to the same endpoint and simply wait. The application never knows a failover happened.

---

## Mechanics

### Connection flow

1. **Client → Proxy.** Fast to establish. Places **no load on the database server** — the connection terminates at the proxy.
2. **Proxy → DB (the pool).** Long-lived, created and maintained independently of client activity. Reused across clients and across Lambda invocations.
3. **Multiplexing.** N client connections → M database connections, where M < N. Reduces DB-side connection count, memory, and CPU.

### Failover behaviour

The critical mental model: **the client's endpoint never changes.**

Without the proxy, a failover means waiting for the **database CNAME to update** from primary to standby — and your application (or its DNS cache) has to notice, re-resolve, and reconnect.

With the proxy:

- Clients stay connected to the single **proxy endpoint**.
- The proxy detects the failure and reconnects its pool to the new primary in the background.
- Client connections **wait** rather than error.
- The failover becomes **transparent to the application** — it doesn't even know it's talking to a proxy; it thinks it's talking to a database.

📚 **Why the improvement is so large:** the proxy **bypasses DNS caching** entirely — that's the mechanism behind the "up to 66%" figure. <cite index="14-1">RDS Proxy bypasses DNS caches to reduce failover times by up to 66% for RDS and Aurora Multi-AZ databases</cite>. AWS also notes that <cite index="14-1">during failovers the application may experience increased latency and in-flight transactions may need to be retried</cite> — the failover is transparent, but not _free_.

---

## When to use RDS Proxy (exam scenarios)

This is the section that maps most directly to exam questions. Cantrill frames these as the scenario types to recognise:

|Scenario|Why RDS Proxy|
|---|---|
|**"Too many connections" errors**|Proxy reduces the connection count hitting the DB. Especially relevant on **small/burst instances (t2, t3)** where connection limits are low.|
|**AWS Lambda**|Eliminates per-invocation connection setup/usage/termination. Reuses the long-running pool. Can also leverage **existing IAM authentication** available to the function via its **execution role**.|
|**Long-running applications (e.g. SaaS) where low latency matters**|No connection establishment per user interaction — reuse the existing pool.|
|**Resilience to database failure is a priority**|Failover time reduced and made transparent to the application.|

---

## Key Facts (exam-critical)

- **Fully managed** database proxy — usable with **RDS and Aurora**.
- **Auto-scaling and highly available by default.** You don't manage it. Much lower admin overhead vs. self-managed proxy.
- **Connection pooling** significantly reduces database load, for two reasons:
    1. No constant open/close cycling stressing the DB.
    2. **Multiplexing** — fewer proxy→DB connections than client→proxy connections.
- **VPC-only.** ⚠️ **Not accessible from the public internet.** Access must come from within a VPC or from private VPC-connected networks (VPN/Direct Connect).
- **Accessed via a proxy endpoint** — behaves like a normal database endpoint, completely transparent to the application.
- **Can enforce SSL/TLS** connections.
- **Reduces failover time by up to ~66%** (Cantrill: "over 60%... in the region of 66% to 67%") vs. connecting to Aurora directly.
- **Abstracts database failure away from the application** — clients wait at the proxy rather than erroring.

---

## Course vs. Current AWS Reality

The lesson is deliberately high-level and holds up well. A few points to layer on:

### Supported engines — broader than "MySQL/PostgreSQL"

The course says "usable with RDS and Aurora" without enumerating engines. Current reality: <cite index="4-1">RDS Proxy is available for Aurora MySQL, Aurora PostgreSQL, RDS for MariaDB, RDS for MySQL, RDS for PostgreSQL, and RDS for SQL Server</cite>. Notably **RDS for Oracle is not supported**.

📚 At the API/CLI level this is expressed as an **engine family**: <cite index="11-1">MYSQL covers Aurora MySQL, RDS for MariaDB and RDS for MySQL; POSTGRESQL covers Aurora PostgreSQL and RDS for PostgreSQL; SQLSERVER covers RDS for Microsoft SQL Server</cite>.

⚠️ Older exam questions may still frame RDS Proxy as MySQL/PostgreSQL-only, reflecting its June 2020 GA scope before MariaDB and SQL Server were added.

### Read-only endpoints for Aurora — not covered in the lesson

The diagram shows only a writer/standby Multi-AZ topology. Since <cite index="29-1">March 2021, RDS Proxy supports creating one or more read-only endpoints to connect applications to Aurora Replicas, pooling and sharing connections among them and automatically connecting to the next available replica on failure</cite>.

<cite index="31-1">The default proxy endpoint is read/write and routes all requests to the cluster's writer instance; you can create additional read/write or read-only endpoints for better traffic distribution</cite>.

📚 Why choose a **proxy read-only endpoint** over Aurora's native **reader endpoint**? Per the AWS Database Blog, the proxy version adds <cite index="30-1">multiplexing to reduce overall connection count, connection pooling that balances connections across readers, and graceful failover that actively monitors instances and routes around failures without DNS caching delay — keeping connections open and pausing new queries until the cluster is back online</cite>.

⚠️ Aurora-only. Reader endpoints are a **cluster** concept — for plain RDS instances with read replicas, <cite index="22-1">a proxy can only associate with the primary (writer) instance, not with read replicas</cite>.

### Session pinning — the hidden catch on multiplexing

Not mentioned in the lesson at all, but it's the main real-world gotcha. <cite index="25-1">When RDS Proxy detects a database request that makes it impractical to move your session to a different connection — or can't be certain multiplexing is safe — it turns off multiplexing for that connection for the remainder of the session. This is called pinning.</cite>

Pinning silently negates the connection-pooling benefit. Common triggers include session state changes, prepared statements, and <cite index="21-1">any statement with a text size greater than 16 KB</cite>.

### Authentication — Secrets Manager + IAM

The lesson mentions Lambda's IAM authentication in passing. The fuller picture: <cite index="21-1">You can enforce IAM authentication for clients connecting to the proxy, and the proxy itself connects to the database using either IAM database authentication or credentials stored in AWS Secrets Manager</cite>.

🎯 **Secrets Manager is a hard requirement** — a proxy must be configured with at least one secret containing DB credentials, even when clients authenticate via IAM. Exam answers pairing "RDS Proxy" with "Secrets Manager" and "no hard-coded credentials" are usually correct.

### Multi-AZ cluster mode

📚 Beyond the standby instance shown in the diagram, RDS Proxy also fronts Multi-AZ **cluster** deployments: <cite index="13-1">RDS Proxy supports Multi-AZ with two readable standbys for typically under 35 second failovers</cite>. Cross-link with your existing note on RDS Multi-AZ (instance vs. cluster mode).

---

## Exam Traps

|⚠️ Trap|Reality|
|---|---|
|"RDS Proxy is publicly accessible / expose it to internet clients"|**VPC-only.** No public internet access. Requires VPC or private VPC-connected network.|
|"RDS Proxy is a general-purpose proxy for any database"|RDS and Aurora only. **No Oracle.** No self-managed databases on EC2.|
|"You need to size/scale/patch the proxy"|**Fully managed, auto-scaling, HA by default.** Zero infrastructure admin — that _is_ the selling point vs. self-managed proxies.|
|"The application needs code changes to know about the proxy"|**Transparent.** The proxy endpoint looks like a normal DB endpoint. The app thinks it's talking to a database.|
|"Failover means the client reconnects/errors"|The **client stays connected and waits**. The proxy re-establishes the backend pool.|
|"Connection pooling means N client conns = N DB conns"|**Multiplexing** means fewer DB connections than client connections. That's the load reduction.|
|"Proxy fixes read scaling for RDS read replicas"|Read-only endpoints are an **Aurora cluster** feature. Plain RDS proxies target the primary only.|
|"Multiplexing always applies"|📚 **Session pinning** can disable it for a session.|
|"RDS Proxy eliminates failover impact entirely"|📚 Reduces it by up to 66% and hides it — but latency can rise and in-flight transactions may need retrying.|

### 🎯 Trigger Keywords

|If the question says...|Think...|
|---|---|
|"too many connections" error|**RDS Proxy**|
|t2 / t3 / small / burstable DB instance + connection exhaustion|**RDS Proxy**|
|Lambda + RDS/Aurora + connection overhead or scaling|**RDS Proxy**|
|"reduce failover time" / "transparent failover" / "without application changes"|**RDS Proxy**|
|"connection pooling" / "share connections" / "multiplexing"|**RDS Proxy**|
|SaaS / long-running app + "low latency" + database connections|**RDS Proxy**|
|"no hard-coded database credentials" + RDS|**RDS Proxy + Secrets Manager (+ IAM auth)**|
|"publicly accessible database proxy"|⚠️ **NOT RDS Proxy** — VPC-only|
|Oracle + database proxy|⚠️ **NOT RDS Proxy** — unsupported engine|
|Read scaling across Aurora Replicas + pooling|**RDS Proxy read-only endpoint**|

---

## Quick Reference

|Attribute|RDS Proxy|
|---|---|
|**Type**|Fully managed database proxy|
|**Works with**|RDS (MySQL, MariaDB, PostgreSQL, SQL Server) + Aurora (MySQL, PostgreSQL)|
|**Not supported**|RDS for Oracle; self-managed DBs on EC2|
|**Availability**|Auto-scaling, highly available by default; spans AZs|
|**Network access**|**VPC-only** — no public internet|
|**Access method**|Proxy endpoint (transparent to app)|
|**Core benefits**|Connection pooling, multiplexing, failover abstraction|
|**Failover improvement**|Up to ~66% faster (bypasses DNS caching)|
|**Security**|Can enforce SSL/TLS; IAM auth for clients; Secrets Manager for DB credentials|
|**Pricing model**|📚 Per vCPU of the underlying DB instance|
|**Key limitation**|📚 Session pinning can disable multiplexing|
|**Primary use case**|Serverless (Lambda), connection-constrained DBs, failover resilience|

---

## Open Questions for Roger

1. **Read-only endpoints** — worth a cross-link from your Aurora note? The proxy-vs-native-reader-endpoint comparison is exactly the kind of "which endpoint" distinction the exam likes, and it bridges two notes you already have.
2. **Session pinning** — do you want this expanded (full trigger list: `SET` statements, temp tables, prepared statements, >16KB statements)? It's arguably below SAA depth, but it's the #1 reason RDS Proxy underdelivers in practice.
3. **Multi-AZ cluster mode** — the "two readable standbys / <35s failover" figure ties into your existing Multi-AZ note. Cross-link, or leave it in the Multi-AZ note only?
4. **Lambda + VPC networking** — the diagram's ENI is doing quiet work here. Does your vault already have a Lambda VPC networking note to link to, or is that still ahead in the course?

---

## Source

- Adrian Cantrill, AWS Solutions Architect Associate (SAA-C03), _"Relational Database Service (RDS) - RDS Proxy"_ — [learn.cantrill.io](https://learn.cantrill.io/)
- Diagram: RDS - Proxy - Architecture (Cantrill course material)