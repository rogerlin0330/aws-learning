---
title: RDS Proxy
type: service
tags: [rds-proxy, rds, aurora, database, connection-pooling, multiplexing, serverless, lambda, high-availability, failover, secrets-manager, iam, vpc, session-pinning, reader-endpoint]
certs: [aws-saa]
updated: 2026-07-16
---

# RDS Proxy

**RDS Proxy** is a **fully managed database proxy** that sits between your clients and your [RDS](rds.md) or [Aurora](aurora.md) database instances. Instead of every client opening its own connection straight to the database, clients connect to the proxy, and the proxy maintains a **long-lived pool of connections** to the database that it reuses and multiplexes across many client connections.

It matters for the exam in its own right and because it underpins many other RDS architectures — especially serverless ones built on [Lambda](lambda.md).

## Why It Exists — Three Problems

1. **Connection establishment is expensive.** Opening and closing a database connection takes time and consumes database memory and CPU. For small operations the connection overhead can dominate the actual work. This is worst with **serverless**: many Lambda invocations each open and close their own connection, and you are billed for that setup time as compute.
2. **Handling database failure in application code is hard.** How long do you wait, what does the app do while waiting, when do you declare failure, and how do you react to a [Multi-AZ](rds.md#multi-az-deployments) failover to the standby? Building all of this into the application adds significant overhead and risk.
3. **Running your own proxy is hard.** A database proxy solves problems 1 and 2, but you may have no proxy experience, and managing proxies *at scale* is its own burden. This is where the "fully managed" part adds the value.

## The Architectural Change

Without the proxy, each client opens a brand-new connection directly to the database every time it needs to do work. With the proxy in place, the client opens a **fast, cheap connection to the proxy**, and the proxy holds a **long-lived, multiplexed pool of connections** back to the database. Two consequences follow:

- **Connection pooling** — the proxy-to-database connections are created once and maintained long-term. They are *not* created and torn down per application request or per Lambda invocation.
- **Multiplexing** — the proxy services a **larger** number of client-to-proxy connections using a **smaller** number of proxy-to-database connections. This is especially valuable on small database instances where resources are scarce.

## Diagram Walkthrough

![RDS Proxy architecture inside VPC1 in us-east-1, spanning three AZs: a Multi-AZ primary in AZB replicating synchronously to a standby in AZC, one logical RDS Proxy spanning all three AZs, EC2 application servers and a Lambda ENI connecting to the proxy, with a large bundle of client connections funnelling into a small persistent database connection pool](../assets/rds-proxy-architecture.png)

The whole architecture lives **inside a VPC** — nothing here is public. Across three availability zones (AZA, AZB, AZC), each with a **private subnet**:

- **Database tier** — a **Primary** RDS instance in AZB with a **Standby** in AZC, joined by **Multi-AZ synchronous replication**.
- **Proxy tier** — an RDS Proxy icon appears in all three AZs, encircled by a single dotted box. That dotted box is the visual shorthand for *"this is one logical proxy spanning all three AZs"* — a managed, highly available, auto-scaling service, not three things you manage.
- **Application tier** — EC2 application servers in AZA and AZC each hold client connections to their local proxy, and a **Lambda ENI** in AZB is how the (regional, VPC-configured) Lambda functions reach into the VPC.

The point of the diagram is the two sides of the proxy:

- **Left of the proxy (client-to-proxy):** a *thick bundle* of connections — **many** short-lived client connections.
- **Right of the proxy (proxy-to-database):** the lines **pinch together** through a small ellipse labelled **"Long Term Connection Pool"** with a **"Multiplexing"** marker. That pinch *is* the multiplexing — a large number of client connections funnelling into a smaller, persistent pool of database connections.

**Reading the failover story off the diagram:** clients hold connections to the proxy endpoint. If the Primary in AZB fails, the proxy establishes new pool connections to the promoted Standby in AZC **in the background**. The client-side lines never break — clients stay connected to the same endpoint and simply wait. The application never knows a failover happened.

## Mechanics

### Connection flow

1. **Client to proxy.** Fast to establish, and places **no load on the database server** — the connection terminates at the proxy.
2. **Proxy to database (the pool).** Long-lived, created and maintained independently of client activity. Reused across clients and across Lambda invocations.
3. **Multiplexing.** N client connections onto M database connections, where M is less than N. This reduces database-side connection count, memory, and CPU.

### Failover behaviour

The critical mental model: **the client's endpoint never changes.**

Without the proxy, a failover means waiting for the **database CNAME to update** from primary to standby, and your application (or its DNS cache) has to notice, re-resolve, and reconnect.

With the proxy:

- Clients stay connected to the single **proxy endpoint**.
- The proxy detects the failure and reconnects its pool to the new primary in the background.
- Client connections **wait** rather than error.
- The failover becomes **transparent to the application** — the app doesn't even know it's talking to a proxy; it thinks it's talking to a database.

The reason the improvement is so large: the proxy **bypasses DNS caching** entirely, which is the mechanism behind the "up to 66%" figure. AWS reduces RDS and Aurora Multi-AZ failover times by up to **66%** with RDS Proxy. The failover is transparent but not *free* — during a failover the application may see increased latency, and in-flight transactions may need to be retried.

## When to Use RDS Proxy

This maps most directly to exam questions. Recognise these scenario types:

| Scenario | Why RDS Proxy |
| --- | --- |
| **"Too many connections" errors** | The proxy reduces the connection count hitting the database. Especially relevant on **small/burstable instances (t2, t3)** where connection limits are low. |
| **AWS Lambda** | Eliminates per-invocation connection setup/usage/teardown by reusing the long-running pool. Can also leverage the **IAM authentication** already available to the function via its **execution role**. |
| **Long-running applications (e.g. SaaS) where low latency matters** | No connection establishment per user interaction — reuse the existing pool. |
| **Resilience to database failure is a priority** | Failover time is reduced and made transparent to the application. |

## Key Facts

- **Fully managed** database proxy, usable with **RDS and Aurora**.
- **Auto-scaling and highly available by default.** You don't size, scale, or patch it. Much lower admin overhead than a self-managed proxy.
- **Connection pooling** significantly reduces database load, for two reasons: no constant open/close cycling stressing the database, and **multiplexing** (fewer proxy-to-database connections than client-to-proxy connections).
- **VPC-only.** Not accessible from the public internet — access must come from within a [VPC](vpc.md) or a private VPC-connected network (VPN / Direct Connect).
- **Accessed via a proxy endpoint** that behaves like a normal database endpoint, completely transparent to the application.
- **Can enforce SSL/TLS** connections.
- **Reduces failover time by up to ~66%** versus connecting to the database directly (by bypassing DNS caching).
- **Abstracts database failure away from the application** — clients wait at the proxy rather than erroring.

## Supported Engines

The course frames this simply as "usable with RDS and Aurora." The current list is broader than the "MySQL/PostgreSQL only" framing older material uses: RDS Proxy is available for **Aurora MySQL, Aurora PostgreSQL, RDS for MariaDB, RDS for MySQL, RDS for PostgreSQL, and RDS for SQL Server**. **RDS for Oracle is not supported.**

At the API/CLI level this is expressed as an **engine family**:

- **MYSQL** — Aurora MySQL, RDS for MariaDB, RDS for MySQL
- **POSTGRESQL** — Aurora PostgreSQL, RDS for PostgreSQL
- **SQLSERVER** — RDS for Microsoft SQL Server

> [!warning] Gotcha
> Older exam questions may still frame RDS Proxy as **MySQL/PostgreSQL-only**, reflecting its June 2020 GA scope before MariaDB and SQL Server were added. Read the answer options in that light, but know the current reality.

## Read-Only Endpoints (Aurora Only)

The diagram shows a writer/standby Multi-AZ topology, but RDS Proxy also supports **read-only endpoints** that connect applications to **Aurora Replicas**, pooling and sharing connections among them and automatically connecting to the next available replica on failure. The **default** proxy endpoint is **read/write** and routes all requests to the cluster's **writer**; you can create additional read/write or read-only endpoints for better traffic distribution.

Why choose a **proxy read-only endpoint** over Aurora's native [reader endpoint](aurora.md#endpoints)? The proxy version adds **multiplexing** to reduce overall connection count, **connection pooling** that balances connections across readers, and **graceful failover** that actively monitors instances and routes around failures without DNS-caching delay — keeping connections open and pausing new queries until the cluster is back online.

> [!warning] Gotcha
> Read-only endpoints are an **Aurora cluster** feature. For plain RDS instances with read replicas, a proxy can only associate with the **primary (writer)** instance, **not** with read replicas — so "use RDS Proxy to scale reads across RDS read replicas" is wrong.

## Session Pinning — The Catch on Multiplexing

Not mentioned in the lesson, but the main real-world gotcha. When RDS Proxy detects a request that makes it impractical to move your session to a different connection — or can't be certain multiplexing is safe — it turns off multiplexing for that connection for the rest of the session. This is called **pinning**, and it silently negates the connection-pooling benefit.

Common triggers include session state changes (such as `SET` statements), temporary tables, prepared statements, and **any statement with a text size greater than 16 KB**.

## Authentication — Secrets Manager + IAM

You can enforce **IAM authentication** for clients connecting to the proxy (see [Identity and Access](../concepts/identity-and-access.md)), and the proxy itself connects to the database using either IAM database authentication or credentials stored in **AWS Secrets Manager**.

> [!tip] Exam Tip
> **Secrets Manager is a hard requirement** — a proxy must be configured with at least one secret containing database credentials, even when clients authenticate via IAM. Exam answers pairing "RDS Proxy" with "Secrets Manager" and "no hard-coded credentials" are usually correct.

## Multi-AZ Cluster Mode

Beyond the single-standby topology in the diagram, RDS Proxy also fronts [Multi-AZ **cluster**](rds.md#multi-az-deployments) deployments — two readable standbys with typically under-35-second failovers.

## Exam Traps

| Trap | Reality |
| --- | --- |
| "RDS Proxy is publicly accessible / expose it to internet clients" | **VPC-only.** No public internet access. Requires a VPC or private VPC-connected network. |
| "RDS Proxy is a general-purpose proxy for any database" | RDS and Aurora only. **No Oracle.** No self-managed databases on EC2. |
| "You need to size/scale/patch the proxy" | **Fully managed, auto-scaling, HA by default.** Zero infrastructure admin — that *is* the selling point over self-managed proxies. |
| "The application needs code changes to know about the proxy" | **Transparent.** The proxy endpoint looks like a normal database endpoint. |
| "Failover means the client reconnects or errors" | The **client stays connected and waits**. The proxy re-establishes the backend pool. |
| "Connection pooling means N client connections equal N database connections" | **Multiplexing** means fewer database connections than client connections. That's the load reduction. |
| "Proxy fixes read scaling for RDS read replicas" | Read-only endpoints are an **Aurora cluster** feature. Plain RDS proxies target the primary only. |
| "Multiplexing always applies" | **Session pinning** can disable it for a session. |
| "RDS Proxy eliminates failover impact entirely" | Reduces it by up to 66% and hides it, but latency can rise and in-flight transactions may need retrying. |

### Trigger keywords

| If the question says... | Think... |
| --- | --- |
| "too many connections" error | **RDS Proxy** |
| t2 / t3 / small / burstable database instance + connection exhaustion | **RDS Proxy** |
| Lambda + RDS/Aurora + connection overhead or scaling | **RDS Proxy** |
| "reduce failover time" / "transparent failover" / "without application changes" | **RDS Proxy** |
| "connection pooling" / "share connections" / "multiplexing" | **RDS Proxy** |
| SaaS / long-running app + "low latency" + database connections | **RDS Proxy** |
| "no hard-coded database credentials" + RDS | **RDS Proxy + Secrets Manager (+ IAM auth)** |
| "publicly accessible database proxy" | **NOT RDS Proxy** — VPC-only |
| Oracle + database proxy | **NOT RDS Proxy** — unsupported engine |
| Read scaling across Aurora Replicas + pooling | **RDS Proxy read-only endpoint** |

## Quick Reference

| Attribute | RDS Proxy |
| --- | --- |
| **Type** | Fully managed database proxy |
| **Works with** | RDS (MySQL, MariaDB, PostgreSQL, SQL Server) + Aurora (MySQL, PostgreSQL) |
| **Not supported** | RDS for Oracle; self-managed databases on EC2 |
| **Availability** | Auto-scaling, highly available by default; spans AZs |
| **Network access** | **VPC-only** — no public internet |
| **Access method** | Proxy endpoint (transparent to the app) |
| **Core benefits** | Connection pooling, multiplexing, failover abstraction |
| **Failover improvement** | Up to ~66% faster (bypasses DNS caching) |
| **Security** | Can enforce SSL/TLS; IAM auth for clients; Secrets Manager for database credentials |
| **Pricing model** | Per vCPU-hour of the underlying database instances |
| **Key limitation** | Session pinning can disable multiplexing |
| **Primary use case** | Serverless (Lambda), connection-constrained databases, failover resilience |

## References

- Adrian Cantrill, AWS Solutions Architect Associate (SAA-C03), *"Relational Database Service (RDS) — RDS Proxy"* — [learn.cantrill.io](https://learn.cantrill.io/)
- [Using Amazon RDS Proxy — Amazon RDS User Guide](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-proxy.html) (supported engines, endpoints, session pinning, authentication)
- [Amazon RDS Proxy — product page](https://aws.amazon.com/rds/proxy/) (up-to-66% failover-time reduction, fully managed positioning)
- [Introducing RDS Proxy read-only endpoints — AWS Database Blog](https://aws.amazon.com/blogs/database/) (proxy reader vs Aurora native reader endpoint)
