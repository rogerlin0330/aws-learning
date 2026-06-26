---
title: Database Transaction Models — ACID, BASE & CAP
type: concept
tags: [database, acid, base, cap-theorem, consistency, availability, partition-tolerance, eventual-consistency, rds, dynamodb, nosql, relational, transactions]
certs: [aws-saa]
updated: 2026-06-25
---

# Database Transaction Models — ACID, BASE & CAP

**ACID** and **BASE** are two database **transaction models** — both acronyms. They define how transactions behave to and from a database, which in turn shapes how the database system itself is architected. This is theoretical but testable: expect a small number of exam questions keyed off the trigger terms, and use them to pick between [RDS](../services/rds.md) (relational, ACID) and NoSQL stores like DynamoDB (BASE).

## The CAP Theorem (the foundation)

A computer-science theorem underpinning both models. **CAP** = **C**onsistency, **A**vailability, **P**artition tolerance.

| Term | Meaning |
| --- | --- |
| **Consistency** | Every read receives the most recent write — or an error |
| **Availability** | Every request receives a non-error response — but with no guarantee it contains the most recent write |
| **Partition tolerance** | The system keeps operating even when messages between its nodes are dropped or error out |

**The core rule: a database can deliver a maximum of two of the three.**

Why? Picture a multi-node database on a network. If communication between nodes fails (a partition) and someone reads, you have two choices:

1. **Cancel** the operation — lower availability, preserve consistency.
2. **Proceed** with the operation — improve availability, risk consistency.

So once multiple nodes and a network are involved, the real choice collapses to **consistency vs availability**. ACID and BASE pick opposite sides:

> [!tip] The one-line summary
> **ACID focuses on consistency. BASE focuses on availability.**

## ACID

> [!tip] Exam default
> If you see **ACID**, assume an **[RDS](../services/rds.md)** / relational database. ACID's rigid rules **limit scalability** — which is exactly why it's the wrong fit for internet-scale, high-velocity workloads and the right fit for financial systems.

| Letter | Property | Meaning |
| --- | --- | --- |
| **A** | **Atomic** | Either *all* parts of a transaction succeed, or *none* do |
| **C** | **Consistent** | Transactions move the DB from one valid state to another — nothing in between |
| **I** | **Isolated** | Concurrent transactions don't interfere; parallel execution matches sequential results |
| **D** | **Durable** | Once committed, a transaction survives system/power failure or restart |

- **Atomic** — a $10 bank transfer is "remove $10 from A" + "add $10 to B". Both apply or neither does; you never want one half to land.
- **Consistent** — relational DBs have inter-table links and value-range rules; every transaction must respect them and leave a valid state.
- **Isolated** — many concurrent transactions (different apps/users) each execute as if alone.
- **Durable** — once the DB reports "committed", a failure or restart won't lose the data.

Most relational platforms use ACID because it implements a rigid, reliable form of data management — and that rigidity is precisely what caps scalability.

## BASE

> [!tip] Exam default
> If you see **BASE**, assume a **NoSQL**-style database (DynamoDB). BASE databases are **highly scalable** and **high performance** because they offload consistency concerns to the application.

| Letters | Property | Meaning |
| --- | --- | --- |
| **BA** | **Basically Available** | Read/write ops are available as much as possible, but with **no consistency guarantee**; availability comes from spreading and replicating across all nodes |
| **S** | **Soft state** | The DB does **not** enforce its own consistency — it delegates that to the **developer/application** |
| **E** | **Eventually consistent** | Consistency may happen *eventually*; wait long enough and a read matches a prior write — but not immediately by default |

- **Basically available** — instead of enforcing immediate consistency, BASE-modeled stores ensure availability by replicating across nodes; reads and writes are "best effort, no guarantee".
- **Soft state** — breaks from the self-enforcing-consistency idea. The application must be consistency-aware. If it needs to always see all prior writes, it must **explicitly request** a consistent read (and only if the DB supports it); otherwise it must tolerate stale reads.
- **Eventually consistent** — the default read behaviour. Many BASE databases *can* do both eventual and immediate reads — but the app has to know which it's asking for.

## AWS Mapping: DynamoDB

DynamoDB normally behaves in a **BASE-like** way, but it straddles both models:

- Offers **both** eventually consistent (default) and **strongly/immediately consistent** reads — the application must choose which it requests.
- Offers **DynamoDB Transactions**, which provide **ACID** guarantees on top of the otherwise-BASE store.

So DynamoDB is the standing exception to "NoSQL = BASE only" — see the trigger table below.

> [!tip] Exam Quick Reference
>
> | If you see… | Default assumption |
> | --- | --- |
> | **BASE** | NoSQL-style database |
> | **ACID** | [RDS](../services/rds.md) / relational database |
> | **NoSQL / DynamoDB** *together with* **ACID** or **transactions** | **DynamoDB Transactions** |
>
> | Model | Optimizes for | Scalability | Typical use |
> | --- | --- | --- | --- |
> | **ACID** | Consistency | Lower (rigid rules) | RDS, relational, financial |
> | **BASE** | Availability | High / high performance | NoSQL, DynamoDB |

> [!warning] Traps & Gotchas
>
> - **CAP = pick two of three**, but once a partition is involved the real choice is **consistency vs availability**: ACID → C, BASE → A.
> - **ACID's "C" ≠ CAP's "C".** ACID consistency = "moves between valid states per DB rules"; CAP consistency = "every read sees the latest write". The exam may blur these — read carefully.
> - **BASE ≠ "bad".** It sounds weak ("kinda available", "soft state") but it's deliberately trading guarantees for **scale and performance**.
> - **Eventual consistency is the BASE default, not the only option** — DynamoDB can do strongly consistent reads *if the app asks*.
> - **DynamoDB can be ACID too** via DynamoDB Transactions — don't reflexively map DynamoDB to BASE when the question mentions transactions/ACID.

## References

- Cantrill SAA-C03 study notes — *ACID vs. BASE — Database Transaction Models*
- Related: [RDS](../services/rds.md) (the canonical ACID/relational service)
